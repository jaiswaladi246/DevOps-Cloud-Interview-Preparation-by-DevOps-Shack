# GitHub Actions — Troubleshooting Drills

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

> Real GitHub Actions errors and symptoms: likely causes, diagnosis, fix and prevention · 100 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to GitHub Actions](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Workflows Not Triggering & Invalid Files](#workflows-not-triggering--invalid-files) — Q1–Q18
- [Expressions, Outputs, Matrix & Conditions](#expressions-outputs-matrix--conditions) — Q19–Q34
- [Permissions, Tokens & Secrets](#permissions-tokens--secrets) — Q35–Q48
- [OIDC & Cloud Authentication](#oidc--cloud-authentication) — Q49–Q62
- [Caching, Artifacts, Builds & Scans](#caching-artifacts-builds--scans) — Q63–Q76
- [Runners, Self-hosted & ARC](#runners-self-hosted--arc) — Q77–Q90
- [Deployments, Environments & Releases](#deployments-environments--releases) — Q91–Q100

---

## Workflows Not Triggering & Invalid Files

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. You pushed a new workflow file and nothing appears in the Actions tab. 🟢

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

### Q2. `Invalid workflow file: .github/workflows/ci.yml#L14 You have an error in your yaml syntax on line 14` 🟢

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

### Q3. `The workflow is not valid. .github/workflows/ci.yml (Line: 9, Col: 5): Unexpected value 'steps'` 🟢

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

### Q4. The workflow runs on push to `main` but never on pull requests. 🟢

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

### Q5. The "Run workflow" button for `workflow_dispatch` does not appear. 🟢

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

### Q6. A scheduled workflow stopped running weeks ago, no errors. 🟢

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

### Q7. A cron set to `0 9 * * *` runs at 14:30 local time, and sometimes 15 minutes late. 🟢

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

### Q8. Pushing a release tag did not trigger the `on: push: tags: ['v*']` workflow. 🟡

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

### Q9. A workflow with `paths: ['src/**']` did not run for a PR that changed `src/app.js`. 🟡

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

### Q10. PR shows "Expected — Waiting for status to be reported" for a required check and can never merge. 🟡

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

### Q11. `No event triggers defined in "on"` 🟢

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

### Q12. `Error: Unable to resolve action 'actions/checkout@v5.9', unable to find version 'v5.9'` 🟢

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

### Q13. `Can't find 'action.yml', 'action.yaml' or 'Dockerfile' under '/home/runner/work/app/app/.github/actions/setup'. Did you forget to run actions/checkout before running your local action?` 🟢

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

### Q14. `error parsing called workflow ".github/workflows/deploy.yml": workflow is not reusable as it is missing a `on.workflow_call` trigger` 🟢

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

### Q15. `Invalid input, environment is not defined in the referenced workflow.` when calling a reusable workflow 🟡

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

### Q16. The workflow on a fork PR shows "This workflow is awaiting approval from a maintainer". 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Repository setting requires approval for fork PR workflows from first-time contributors or all outside collaborators.

**Diagnose →** Settings, Actions, General, "Approval for running fork pull request workflows from contributors".

**Fix →** Review the PR diff (especially workflow and build files), then "Approve and run".

**Prevent →** Keep approval on for public repos; it is a guard against crypto-mining and abuse.

</details>

### Q17. A workflow file change in a PR does not take effect; the old behaviour runs. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The trigger reads the workflow from the default branch: `pull_request_target`, `workflow_run`, `issue_comment`, `schedule`. Or you re-ran an old run, which reuses the original workflow file.

**Diagnose →** Check the trigger; check the run's commit and attempt number (`gh run view <id> --json headSha,attempt`).

**Fix →** Merge the change to the default branch for those triggers; push a new commit instead of re-running.

**Prevent →** Know which events use base vs head workflow definitions.

</details>

### Q18. `The workflow is not valid. ... (Line: 22, Col: 16): Unrecognized named-value: 'env'. Located at position 1 within expression: env.RUNNER` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `env` context used where it is not available, such as `runs-on`, job-level `if`, `concurrency` at workflow level, or `services`.

**Diagnose →** Check the context availability table for that key.

**Fix →** Use `vars.RUNNER` or `inputs.runner` for `runs-on`, move the condition to step level, or compute the value in an earlier job and pass via `needs`.

**Prevent →** actionlint knows context availability and catches this before pushing.

</details>

## Expressions, Outputs, Matrix & Conditions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q19. A downstream job reads `needs.build.outputs.digest` and gets an empty string. 🟡

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

### Q20. `Warning: Skip output 'image' since it may contain secret.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The value contains a registered secret or masked string, often the AWS account ID (part of the ECR registry) because someone stored the account ID as a secret, or `configure-aws-credentials` was told to mask the account ID (`mask-aws-account-id: true`).

**Diagnose →** Check which secrets or masks overlap with the value.

**Fix →** Store account IDs as variables, leave account-ID masking off on `configure-aws-credentials` if your policy allows, or output only the non-sensitive part (digest) and rebuild the full reference downstream.

**Prevent →** Do not register non-secret identifiers as secrets.

</details>

### Q21. `Error: Unable to process file command 'output' successfully. Error: Invalid format '  "items": ['` 🟡

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

### Q22. `Error when evaluating 'strategy' for job 'build'. .github/workflows/ci.yml (Line: 30, Col: 15): Matrix vector 'service' does not contain any values` 🟡

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

### Q23. `Error reading JToken from JsonReader. Path '', line 0, position 0.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `fromJSON('')`: the output feeding `fromJSON` is empty because the upstream job or step was skipped, failed or the output name is wrong.

**Diagnose →** Check upstream job status and whether the output is set.

**Fix →** Fix the output; guard with `if` on non-empty; or provide a default: `fromJSON(needs.plan.outputs.matrix || '{"include":[]}')`, plus an empty-matrix guard.

**Prevent →** Validate JSON in the producing step with `jq -e .`.

</details>

### Q24. `if: github.event.inputs.deploy == true` never runs the step, even when the box is ticked. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `github.event.inputs` values are strings; `'true' == true` is false after numeric coercion.

**Diagnose →** Dump `toJSON(github.event.inputs)` and `toJSON(inputs)`.

**Fix →** `if: inputs.deploy` (typed) or `if: github.event.inputs.deploy == 'true'`.

**Prevent →** Use the `inputs` context everywhere.

</details>

### Q25. Deploy job is always "skipped" even though build succeeded. 🟡

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

### Q26. A notification step with `if: always()` posted "failed" even when the run was cancelled by a newer commit. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `always()` runs on cancel too, and the script assumed non-success meant failure.

**Diagnose →** Check `job.status` in the step: `cancelled`.

**Fix →** Use `if: failure()` for failure alerts, or `if: ${{ !cancelled() }}` with status-specific messages.

**Prevent →** Reserve `always()` for cleanup.

</details>

### Q27. A matrix job's output used downstream has a different value on each run. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** All matrix legs write the same job output; the last leg to finish wins.

**Diagnose →** Compare leg finish times with the value.

**Fix →** Per-leg artifacts (`digest-${{ matrix.arch }}`) collected by a fan-in job, or separate non-matrix jobs when values must be distinct.

**Prevent →** Never consume job outputs from matrix jobs as if they were single values.

</details>

### Q28. `$GITHUB_ENV` variable set in one step is empty in the same step. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `GITHUB_ENV` writes take effect in subsequent steps only.

**Diagnose →** Check where the variable is read.

**Fix →** Also `export` it in the current step, or read it in the next step.

**Prevent →** Remember: env files apply to later steps.

</details>

### Q29. `$GITHUB_ENV` value set in the build job is empty in the deploy job. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `GITHUB_ENV` is per job; jobs run on different runners.

**Diagnose →** Check job boundaries.

**Fix →** Use job outputs and `needs`, or an artifact.

**Prevent →** Draw the job graph and mark how data crosses each edge.

</details>

### Q30. A step with `run: npm test | tee test.log` passes even though tests failed. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Default shell is `bash -e {0}` without `pipefail`, so the pipeline's exit code is `tee`'s (0).

**Diagnose →** Check the "Run" header in logs: `shell: /usr/bin/bash -e {0}`.

**Fix →** Set `shell: bash` (adds `-o pipefail`) on the step or in `defaults.run`, or `set -o pipefail` in the script.

**Prevent →** Workflow-level `defaults: run: shell: bash`.

</details>

### Q31. `if: contains(github.event.pull_request.labels, 'deploy')` never matches. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `labels` is an array of objects, not strings.

**Diagnose →** `echo '${{ toJSON(github.event.pull_request.labels) }}'` via env.

**Fix →** `contains(github.event.pull_request.labels.*.name, 'deploy')`.

**Prevent →** Use object filters for arrays of objects.

</details>

### Q32. A ternary `${{ inputs.replicas && inputs.replicas || 2 }}` returns 2 when replicas is 0. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `0` is falsy, so `cond && a || b` falls through to `b`.

**Diagnose →** Evaluate with sample values.

**Fix →** Use a default on the input itself (`default: 2`) and read `inputs.replicas` directly, or compute in a shell step.

**Prevent →** Keep the ternary idiom for string values that are never empty.

</details>

### Q33. `hashFiles('**/package-lock.json') couldn't finish within 120 seconds.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Huge workspace (large `node_modules`, checked-out submodules, build outputs) scanned by the glob on a slow self-hosted disk.

**Diagnose →** `find . -name package-lock.json | wc -l` and workspace size with `du -sh`.

**Fix →** Narrow the pattern (`package-lock.json`, `services/*/package-lock.json`), exclude `!**/node_modules/**`, or compute the key in a step with `sha256sum`.

**Prevent →** Ephemeral runners with clean workspaces.

</details>

### Q34. `Unrecognized named-value: 'secrets'. Located at position 1 within expression: secrets.NPM_TOKEN != ''` 🟡

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

## Permissions, Tokens & Secrets

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q35. `Error: Resource not accessible by integration` when a workflow tries to comment on a PR. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `GITHUB_TOKEN` lacks `pull-requests: write` (or `issues: write` for the issues comments API), a job-level `permissions` block replaced the workflow-level one, or the PR is from a fork so the token is read-only.

**Diagnose →** The "Set up job" log section lists "GITHUB_TOKEN Permissions". Check `github.event.pull_request.head.repo.fork`.

**Fix →** Add `permissions: pull-requests: write` to that job; for forks use the `workflow_run` two-workflow pattern.

**Prevent →** Declare permissions per job and test fork PRs in a sandbox repo.

</details>

### Q36. `remote: Permission to devopsshack/shack-api.git denied to github-actions[bot]. fatal: unable to access 'https://github.com/devopsshack/shack-api/': The requested URL returned error: 403` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Token has `contents: read` only (restricted default), the branch is protected and the bot is not allowed to push, or `persist-credentials: false` removed the credentials.

**Diagnose →** Check "GITHUB_TOKEN Permissions" in the log and branch protection or rulesets on the target branch.

**Fix →** `permissions: contents: write` for that job; push to an unprotected branch and open a PR; or use a GitHub App in the ruleset bypass list.

**Prevent →** Prefer PR-based automation over direct pushes to protected branches.

</details>

### Q37. `fatal: could not read Username for 'https://github.com': terminal prompts disabled` when cloning a second private repo. 🟢

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

### Q38. A secret is empty in the workflow even though it is set in Settings. 🟢

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

### Q39. `Error: Input required and not supplied: token` on a fork PR. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The step passes `token: ${{ secrets.SOME_PAT }}`, which is empty for forks.

**Diagnose →** Check whether the run is from a fork.

**Fix →** Skip the step for forks (`if: github.event.pull_request.head.repo.full_name == github.repository`) or use `github.token` where read access is enough.

**Prevent →** Design PR workflows to work without secrets.

</details>

### Q40. Releases created by a workflow do not trigger the `on: release` deploy workflow. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The release was created with `GITHUB_TOKEN`, and events from `GITHUB_TOKEN` do not start new runs.

**Diagnose →** Check the release author: `github-actions[bot]`.

**Fix →** Create releases with a GitHub App token, or call the deploy as a reusable workflow from the release workflow.

**Prevent →** Document the `GITHUB_TOKEN` no-trigger rule in your platform docs.

</details>

### Q41. `Error: HttpError: Resource not accessible by integration` when uploading SARIF to code scanning. 🟡

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

### Q42. `denied: permission_denied: write_package` when pushing to GHCR. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing `packages: write`, the package exists but is not linked to this repo (created by another repo or a user), or the image name has uppercase letters.

**Diagnose →** Package settings, "Manage Actions access" list; check image name casing.

**Fix →** Add `packages: write`, grant the repo write access in package settings, lowercase the name (`${GITHUB_REPOSITORY,,}`).

**Prevent →** Add `org.opencontainers.image.source` label so packages link to the repo automatically.

</details>

### Q43. A secret appears in plain text in the logs despite masking. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The printed form differs from the registered value: base64, JSON-escaped, URL-encoded, a substring, or with trailing newline differences; or the value was generated at runtime and never masked.

**Diagnose →** Find the step and the transformation.

**Fix →** Rotate immediately, delete the run logs, add `::add-mask::` for runtime and derived values, remove `set -x`.

**Prevent →** Never echo secrets; review debug output before enabling `ACTIONS_STEP_DEBUG` on sensitive workflows.

</details>

### Q44. `gh: To use GitHub CLI in a GitHub Actions workflow, set the GH_TOKEN environment variable.` 🟢

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

### Q45. `actions/checkout` fails with `remote: Repository not found.` on a private repo after adding a job-level `permissions` block. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The job block replaced the workflow block and did not include `contents: read`, so the token cannot read the repo.

**Diagnose →** "GITHUB_TOKEN Permissions" in the log shows `Contents: none`.

**Fix →** Add `contents: read` to the job's permissions.

**Prevent →** Remember job-level permissions replace, not merge.

</details>

### Q46. A workflow can no longer cancel or re-run other workflows: `Resource not accessible by integration` on the actions API. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing `actions: write` after switching to restricted default permissions.

**Diagnose →** Check the API call and permissions.

**Fix →** Add `permissions: actions: write` to that job only.

**Prevent →** Inventory which jobs use which APIs when tightening defaults.

</details>

### Q47. An org secret is visible to the repo in Settings but empty in a private repo's workflow. 🟡

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

### Q48. Workflows suddenly fail with permission errors across many repos after an org policy change. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Org default workflow permissions switched to read-only, and workflows relied on implicit write.

**Diagnose →** Search for workflows without `permissions:` that write (push, release, comment).

**Fix →** Add explicit `permissions` to affected workflows; do not revert the org default.

**Prevent →** Before flipping defaults, scan repos and open PRs adding explicit permissions.

</details>

## OIDC & Cloud Authentication

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q49. `Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity` 🔴

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

### Q50. `Error: Credentials could not be loaded, please check your action inputs: Could not load credentials from any providers` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The job lacks `id-token: write`, so `configure-aws-credentials` cannot get an OIDC token and falls back to looking for keys; or `role-to-assume` is empty (a variable not set in this environment).

**Diagnose →** Check permissions and that `vars.DEPLOY_ROLE_ARN` resolves (print it, it is not secret).

**Fix →** Add `permissions: id-token: write` (plus `contents: read`) and set the variable at the correct level.

**Prevent →** Fail fast: `test -n "$ROLE_ARN" || { echo "::error::DEPLOY_ROLE_ARN not set"; exit 1; }`.

</details>

### Q51. `Error: Unable to get ACTIONS_ID_TOKEN_REQUEST_URL env variable` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `id-token: write` missing for the job, or for reusable workflows, missing in the **caller** job's permissions (the called workflow cannot elevate); or a fork PR where OIDC is not available with write.

**Diagnose →** Check permissions at caller and callee.

**Fix →** Grant `id-token: write` on the calling job and in the called job.

**Prevent →** Standard permission snippet in the reusable workflow's README.

</details>

### Q52. OIDC works on `main` but fails on a tag-triggered release workflow. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Trust policy only allows `repo:ORG/REPO:ref:refs/heads/main`; tag runs have `sub` `repo:ORG/REPO:ref:refs/tags/v1.4.0`.

**Diagnose →** Decode the token claims in the failing run.

**Fix →** Add a `StringLike` condition for `repo:ORG/REPO:ref:refs/tags/v*`, or bind the release job to an environment and trust the environment subject.

**Prevent →** Use environments for every privileged job so `sub` is predictable.

</details>

### Q53. `An error occurred (AccessDenied) when calling the AssumeRoleWithWebIdentity operation: Incorrect token audience` 🟡

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

### Q54. `No OpenIDConnect provider found in your account for https://token.actions.githubusercontent.com` 🟢

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

### Q55. After assuming the role, `aws ecr get-login-password` works but push fails: `denied: User: arn:aws:sts::111122223333:assumed-role/gha-ecr-push/gha-123 is not authorized to perform: ecr:InitiateLayerUpload on resource: arn:aws:ecr:ap-south-1:111122223333:repository/shack-api` 🟡

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

### Q56. `error: You must be logged in to the server (Unauthorized)` from kubectl on EKS after OIDC login. 🔴

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

### Q57. kubectl now authenticates but returns `Error from server (Forbidden): deployments.apps "shack-api" is forbidden: User "arn:aws:sts::111122223333:assumed-role/gha-deploy-prod/gha-987" cannot patch resource "deployments" in API group "apps" in the namespace "shack-staging"` 🟡

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

### Q58. `AADSTS70021: No matching federated identity record found for presented assertion. Assertion Issuer: 'https://token.actions.githubusercontent.com'. Assertion Subject: 'repo:devopsshack/shack-api:environment:prod'.` 🟡

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

### Q59. GCP: `Permission 'iam.serviceAccounts.getAccessToken' denied on resource (or it may not exist).` 🔴

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

### Q60. Assumed-role credentials expire mid-job: `ExpiredToken: The security token included in the request is expired` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The job runs longer than the session duration (default 1 hour), for example long Terraform applies or big image builds.

**Diagnose →** Compare job duration with `role-duration-seconds`.

**Fix →** Set `role-duration-seconds: 7200` and raise the role's `MaxSessionDuration`, or re-run `configure-aws-credentials` before the late steps.

**Prevent →** Split long jobs; authenticate right before the cloud steps.

</details>

### Q61. A PR workflow intended to use the read-only role fails with AccessDenied, but `main` works. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** PR token `sub` is `repo:ORG/REPO:pull_request`; the trust allows only the `ref` form. Or the PR is from a fork where OIDC tokens are not issued with write.

**Diagnose →** Decode claims; check fork status.

**Fix →** Add a separate role trusting `repo:ORG/REPO:pull_request` with read-only permissions.

**Prevent →** Document the four `sub` forms in your platform guide.

</details>

### Q62. OIDC with a custom `sub` template stopped working for every repo after an org change. 🔴

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

## Caching, Artifacts, Builds & Scans

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q63. `Cache not found for input keys: npm-Linux-3f9a...` on every single run. 🟢

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

### Q64. `Failed to save: Unable to reserve cache with key npm-Linux-3f9a..., another job may be creating this cache.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Several matrix legs or parallel jobs try to save the same key; only the first succeeds. Harmless, but noisy.

**Diagnose →** Check which jobs share the key.

**Fix →** Include matrix values in the key when contents differ, or save from one leg only with `actions/cache/save` and an `if`.

**Prevent →** Design keys per OS, arch and toolchain version.

</details>

### Q65. A cache restores but the build is still slow and reinstalls everything. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cached the wrong path (`node_modules` with `npm ci`, which deletes it), path mismatch between OS (`~/.npm` vs `%AppData%`), restore-key hit an old cache with mostly stale content, or the tool uses a different cache directory (Yarn Berry, pnpm store).

**Diagnose →** `npm config get cache`, `pnpm store path`, check restored size in logs.

**Fix →** Cache the tool's real cache path, or use setup-node's `cache: pnpm`.

**Prevent →** Measure step durations with and without cache.

</details>

### Q66. `Error: Failed to CreateArtifact: Received non-retryable error: Failed request: (409) Conflict: an artifact with this name already exists on the workflow run` 🟢

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

### Q67. `Error: Unable to download artifact(s): Artifact not found for name: build` 🟢

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

### Q68. `Warning: No files were found with the provided path: coverage/. No artifacts will be uploaded.` but tests ran. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Coverage written to a different directory, `working-directory` used for tests but the upload path is relative to the workspace root, or the test step failed and coverage was not written.

**Diagnose →** Add `ls -R` before upload; check `defaults.run.working-directory`.

**Fix →** Use the correct path (`services/api/coverage/`), add `if: ${{ !cancelled() }}` to upload even after test failure.

**Prevent →** `if-no-files-found: error`.

</details>

### Q69. `ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref ...: "/package-lock.json": not found` 🟡

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

### Q70. `System.IO.IOException: No space left on device` on a hosted runner during a Docker build. 🟡

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

### Q71. `tag invalid: The image tag 'sha-a1b2c3d' already exists in the 'shack-api' repository and cannot be overwritten because the repository is immutable.` 🟡

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

### Q72. Trivy fails with `FATAL Fatal error init error: DB error: failed to download vulnerability DB ... TOOMANYREQUESTS: retry-after` 🟡

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

### Q73. Trivy blocks the pipeline on a CRITICAL CVE in a base image package with no fix available. 🟡

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

### Q74. `buildx failed with: ERROR: failed to solve: failed to push 111122223333.dkr.ecr.ap-south-1.amazonaws.com/shack-api:sha-a1b2c3d: unexpected status from HEAD request ... 401 Unauthorized` 🟡

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

### Q75. An image built on a hosted runner fails in EKS Graviton nodes with `exec format error`. 🟡

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

### Q76. Docker layer cache (`type=gha`) never hits on PR builds. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cache only written from PR refs (invisible to other PRs), cache evicted due to the 10 GB quota, `scope` differs between writers and readers, or the build context changes early layers each time (copying source before installing dependencies).

**Diagnose →** `gh cache list --key buildkit` and build logs showing `CACHED` vs rebuilt layers.

**Fix →** Write cache from `main`, keep scopes consistent, reorder Dockerfile, consider registry cache.

**Prevent →** Monitor cache size and hit rates.

</details>

## Runners, Self-hosted & ARC

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q77. Job stuck at `Waiting for a runner to pick up this job...` 🟢

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

### Q78. `This request was automatically failed because there were no enabled runners online to process the request for more than 1 days.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A job queued 24 hours with no matching runner, often an old label like `ubuntu-20.04` after retirement or a decommissioned self-hosted fleet.

**Diagnose →** Check `runs-on` value in the failed job.

**Fix →** Update to a current label; restore the runner fleet.

**Prevent →** Track GitHub runner image deprecation announcements and search org workflows for retired labels.

</details>

### Q79. `Must not run with sudo` when running `./config.sh`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The runner configuration script refuses root.

**Diagnose →** `whoami`.

**Fix →** Create a dedicated user (`useradd -m runner`), run `config.sh` as that user, then install the service with `sudo ./svc.sh install runner`. In containers, set `RUNNER_ALLOW_RUNASROOT=1` only if you understand the implications.

**Prevent →** Bake a non-root runner user into the runner image or AMI.

</details>

### Q80. `Http response code: NotFound from 'POST https://api.github.com/actions/runner-registration'` during `config.sh`. 🟢

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

### Q81. Self-hosted runner logs: `Runner version v2.311.0 is deprecated and cannot receive messages.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Auto-update disabled (`--disableupdate`) or blocked by network, and the runner fell below the minimum supported version.

**Diagnose →** `./run.sh --version`; check runner logs in `_diag/`.

**Fix →** Upgrade the runner binary or rebuild the runner image with the latest release.

**Prevent →** If auto-update is disabled, automate image rebuilds on new runner releases (Renovate on the runner version).

</details>

### Q82. `fatal: detected dubious ownership in repository at '/__w/app/app'` in a container job on a self-hosted runner. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Workspace owned by a different UID than the container user (root vs runner user), triggering git's safe.directory check.

**Diagnose →** `ls -ln` in the workspace and `id` in the container.

**Fix →** `git config --global --add safe.directory "$GITHUB_WORKSPACE"`, or run the container with the runner's UID (`options: --user 1001`).

**Prevent →** Ephemeral runners and consistent UIDs across images.

</details>

### Q83. Self-hosted runner jobs fail with `EACCES: permission denied, unlink '/home/runner/_work/app/app/dist/index.js'` at checkout. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A previous container job wrote root-owned files into the persistent workspace; the runner user cannot delete them.

**Diagnose →** `find _work -user root | head`.

**Fix →** Clean with a privileged cleanup (`docker run --rm -v $PWD:/w alpine rm -rf /w/dist`), then run containers as the runner UID.

**Prevent →** Ephemeral runners, or a job-completed hook that resets ownership.

</details>

### Q84. ARC listener pod logs `failed to get runner registration token ... 401 Bad credentials`. 🔴

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

### Q85. ARC runner pods stay `Pending`; jobs wait. 🔴

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

### Q86. ARC job fails: `Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?` 🟡

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

### Q87. Jobs on ARC fail at "Initialize containers" with `Error: Jobs without a job container are forbidden on this runner, please add a 'container:' to your job` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The scale set uses kubernetes container mode with `ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER=true`, which requires every job to declare `container:`.

**Diagnose →** Check the scale set's template env and container mode.

**Fix →** Add `container: image: ...` to the job, or set the variable to `false` if host execution is acceptable.

**Prevent →** Publish a runner usage guide per scale set.

</details>

### Q88. ARC scales to `maxRunners` and stays there with idle pods. 🔴

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

### Q89. Runner jobs on EKS can call AWS with the node's role, which they should not. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pods can reach the EC2 instance metadata service (IMDS) and inherit the node IAM role.

**Diagnose →** In a job: `curl -s -m 2 http://169.254.169.254/latest/meta-data/iam/security-credentials/`.

**Fix →** Enforce IMDSv2 with hop limit 1 on runner nodes (launch template or Karpenter `metadataOptions.httpPutResponseHopLimit: 1`), or block with a NetworkPolicy; use OIDC in workflows instead.

**Prevent →** Include the IMDS check in your runner security tests.

</details>

### Q90. `The runner has received a shutdown signal. This can happen when the runner service is stopped, or a manually started runner is canceled.` on hosted or ARC runners. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Self-hosted: the VM or pod was terminated (spot interruption, node scale-down, OOM-killed runner). Hosted: rare infrastructure issue or the job exhausted memory and the VM became unresponsive.

**Diagnose →** `kubectl get events -n arc-runners`, node termination events, Karpenter logs; on hosted, check memory usage patterns.

**Fix →** Add a PodDisruptionBudget or `karpenter.sh/do-not-disrupt: "true"` annotation on runner pods, use on-demand for long jobs, reduce memory use; re-run the job.

**Prevent →** Keep jobs idempotent and short so interruptions are cheap.

</details>

## Deployments, Environments & Releases

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q91. `Branch "feature/login" is not allowed to deploy to prod due to environment protection rules.` 🟢

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

### Q92. Deploy job shows `Waiting for review: prod needs approval to start deploying changes` but no reviewer got notified. 🟢

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

### Q93. `Canceling since a higher priority waiting request for 'deploy-prod' exists` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Concurrency group behaviour: a newer run joined the group, so the older pending run was cancelled (or the running one with `cancel-in-progress: true`).

**Diagnose →** Find the concurrency group definition and the competing run.

**Fix →** Expected for "deploy latest" semantics. If every version must deploy, remove `cancel-in-progress` and accept that only one pending run is kept, or redesign to queue via a different mechanism.

**Prevent →** Explain the one-running-one-pending rule in team docs.

</details>

### Q94. `kubectl rollout status` timed out: `error: deployment "shack-api" exceeded its progress deadline` 🟡

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

### Q95. `ErrImagePull ... failed to resolve reference "111122223333.dkr.ecr.ap-south-1.amazonaws.com/shack-api@sha256:...": not found` after a deploy. 🔴

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

### Q96. Release workflow `gh release create` fails with `HTTP 403: Resource not accessible by integration` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing `contents: write`.

**Diagnose →** "GITHUB_TOKEN Permissions" in the log.

**Fix →** Add `permissions: contents: write` to the release job only.

**Prevent →** Template release workflows with correct permissions.

</details>

### Q97. `gh release create v1.4.0 --verify-tag` fails with `tag v1.4.0 doesn't exist in the repo`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The workflow was dispatched manually with a version input but nobody pushed the tag, or the tag exists locally only.

**Diagnose →** `git ls-remote --tags origin v1.4.0`.

**Fix →** Push the tag, or drop `--verify-tag` and pass `--target "$GITHUB_SHA"` to create it (with the no-trigger caveat for `GITHUB_TOKEN`).

**Prevent →** Trigger releases from tag pushes, not from free-text version inputs.

</details>

### Q98. Release notes generated by `--generate-notes` include unrelated commits from months ago. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The previous tag is not found (shallow checkout irrelevant here, but tag naming changed, for example `release-1.3` vs `v1.3.0`), so GitHub compares against an older tag.

**Diagnose →** `gh release list` and compare tag patterns.

**Fix →** Pass `--notes-start-tag v1.3.0` explicitly.

**Prevent →** Keep a consistent tag scheme and configure `.github/release.yml` categories.

</details>

### Q99. release-please merged its PR but no release or tag was created. 🟡

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

### Q100. Prod deploy succeeded in GitHub, but the old version is still serving traffic. 🔴

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
