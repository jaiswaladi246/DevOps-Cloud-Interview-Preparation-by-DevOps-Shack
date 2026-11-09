# Jenkins — Scenario-Based Questions

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

> Real-world Jenkins situations and how to reason through them in an interview · 100 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Jenkins](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Pipeline design & Jenkinsfile decisions](#pipeline-design--jenkinsfile-decisions) — Q1–Q20
- [Shared libraries & multibranch](#shared-libraries--multibranch) — Q21–Q38
- [Platform: plugins, JCasC & Kubernetes agents](#platform-plugins-jcasc--kubernetes-agents) — Q39–Q56
- [Credentials & security](#credentials--security) — Q57–Q74
- [Operations: scale, HA, upgrades & performance](#operations-scale-ha-upgrades--performance) — Q75–Q90
- [Integrations & delivery](#integrations--delivery) — Q91–Q100

---

## Pipeline design & Jenkinsfile decisions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. A team's PR builds pile up: every push to an open PR starts a new 25-minute build while the old one keeps running. What do you change? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Outdated builds waste executors and pods; only the newest commit matters for a PR.

**What I'd do →**
1. Add to the Jenkinsfile:
   ```groovy
   options { disableConcurrentBuilds(abortPrevious: true) }
   ```
2. Confirm the branch source discovers PRs and excludes branches filed as PRs, so each push builds once.
3. Keep `main` builds unaborted if every commit must be built: `disableConcurrentBuilds(abortPrevious: env.BRANCH_NAME != 'main')`.

**Why →** `abortPrevious` cancels the running build of the same job when a new one starts, freeing capacity immediately.

**Interview tip →** Mention that each PR is its own child job in multibranch, so this only affects builds of the same PR.

</details>

### Q2. A deploy pipeline waits on `input` for approval, and on Monday you find 14 pods sitting idle since Friday. Fix it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The `input` step is inside a stage that holds an agent, so each waiting build keeps a pod (and its resource requests) alive.

**What I'd do →**
1. Use `agent none` at the top and give build stages their own agent.
2. Move the approval to a stage-level `input` directive with no agent and a timeout:
   ```groovy
   stage('Approve prod') {
     options { timeout(time: 8, unit: 'HOURS') }
     input { message 'Deploy to prod?'; submitter 'release-managers' }
     steps { echo 'Approved' }
   }
   ```
3. Run the actual deploy in the next stage with its own deploy pod.
4. Add `milestone()` or `abortPrevious` so an older build cannot be approved after a newer one.

**Why →** Waiting on the controller costs nothing; waiting inside an agent costs a pod per build.

**Interview tip →** Always say "timeout on every input"; unbounded approvals are a classic Jenkins smell.

</details>

### Q3. Unit tests, lint, dependency audit and SAST run sequentially and take 22 minutes. How do you restructure the pipeline? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** These checks are independent and can run concurrently.

**What I'd do →**
1. Put them under one `parallel` stage with `failFast true`.
2. If they share one pod, run them as parallel branches using different containers; if heavy, give each branch its own agent.
3. Run the fastest check (lint, under 1 minute) also as a separate first stage so obvious failures stop the build before pods spin up.
   ```groovy
   stage('Checks') { failFast true; parallel {
     stage('Unit') { steps { container('node') { sh 'npm test' } } }
     stage('Audit') { steps { container('node') { sh 'npm audit --audit-level=high' } } }
     stage('SAST') { steps { container('semgrep') { sh 'semgrep ci' } } }
   } }
   ```

**Why →** Wall time becomes the slowest branch instead of the sum.

**Interview tip →** Quote numbers: "22 minutes down to 8, the longest branch".

</details>

### Q4. Developers want to rerun only the deploy stage when it fails due to a flaky cluster API, without rebuilding the image. What do you offer? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rebuilding is slow and produces a different artifact; we want to retry with the same image.

**What I'd do →**
1. Ensure the deploy stage is a top-level declarative stage so **Restart from Stage** is available.
2. Pass the image tag through `env` computed from the commit (deterministic), not from a stash.
3. Add automatic retry only for infrastructure errors around the API calls: `retry(count: 2, conditions: [kubernetesAgent(), nonresumable()])`, plus a `retry(2)` around `kubectl apply` with backoff.
4. Enable `preserveStashes(buildCount: 5)` if the stage needs stashed manifests.

**Why →** Restart from Stage reuses earlier stage results and redeploys the exact same artifact.

**Interview tip →** Connect to "build once, deploy many".

</details>

### Q5. A pipeline must run integration tests against a shared staging database that can only handle one test run at a time across four different repos. How do you coordinate? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `disableConcurrentBuilds` only works per job; we need a cross-job mutex.

**What I'd do →**
1. Install Lockable Resources and define a resource `staging-db` (via JCasC under `unclassified.lockableResourcesManager`).
2. Wrap only the integration stage:
   ```groovy
   stage('Integration') {
     options { lock(resource: 'staging-db') }
     steps { sh './run-it.sh' }
   }
   ```
3. Put a `timeout` on the stage so a hung test releases the lock.
4. Long term, give each run an ephemeral database (a Testcontainers Postgres in the pod) and remove the lock.

**Why →** `lock` queues builds across jobs and releases automatically when the block exits, even on failure.

**Interview tip →** Say that locks hide a scalability problem; ephemeral environments are the real fix.

</details>

### Q6. A library must be tested on Node 18, 20 and 22, on Linux and Windows, but Node 18 on Windows is not supported. Design the stage. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic matrix with an exclusion.

**What I'd do →**
```groovy
stage('Test') {
  matrix {
    axes {
      axis { name 'NODE_VERSION'; values '18', '20', '22' }
      axis { name 'PLATFORM'; values 'linux', 'windows' }
    }
    excludes {
      exclude {
        axis { name 'NODE_VERSION'; values '18' }
        axis { name 'PLATFORM'; values 'windows' }
      }
    }
    agent { label "${PLATFORM}" }
    stages { stage('npm test') { steps { script {
      if (isUnix()) { sh 'npm ci && npm test' } else { bat 'npm ci && npm test' }
    } } } }
  }
}
```
1. Select Node per cell with nvm/fnm or a container image tag per version on Linux.
2. Publish JUnit per cell so failures show which combination broke.

**Why →** `matrix` generates five parallel cells with readable names and no copy-pasted stages.

**Interview tip →** Mention `isUnix()` for cross-platform steps.

</details>

### Q7. Your 700-line Jenkinsfile now fails with `Method code too large!`. You need a fix today and a plan for next month. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The compiled pipeline exceeds the JVM's 64 KB method limit.

**What I'd do →**
1. Today: move the longest inline `sh '''...'''` blocks into script files in the repo (`ci/deploy.sh`), which shrinks bytecode dramatically.
2. Try enabling script splitting via the system property `-Dorg.jenkinsci.plugins.pipeline.modeldefinition.parser.RuntimeASTTransformer.SCRIPT_SPLITTING_TRANSFORMATION=true`.
3. Next month: extract repeated stage logic into shared library steps, replace copy-pasted per-environment stages with a loop or `matrix`, and split release and CI into separate jobs.

**Why →** The limit is on generated bytecode, so less inline code and fewer stages per method are the only real fixes.

**Interview tip →** Use this to show you know the Jenkinsfile is compiled Groovy, not YAML.

</details>

### Q8. A nightly job should run only on `main` in a multibranch project, but every feature branch also fires at 2 AM. What is wrong and how do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `triggers { cron('H 2 * * *') }` in the Jenkinsfile registers the trigger on every branch job that has the file.

**What I'd do →**
1. Make the cron conditional:
   ```groovy
   triggers { cron(env.BRANCH_NAME == 'main' ? 'H 2 * * *' : '') }
   ```
2. Note that the trigger registers after the next build of each branch, so trigger one build per branch or wait for the next push.
3. Alternatively, move nightly work to a separate pipeline job targeting `main`.

**Why →** Triggers are job properties set by running the Jenkinsfile; an empty cron string clears them.

**Interview tip →** Mention that properties in a Jenkinsfile only apply after the build runs once.

</details>

### Q9. Product asks for a "deploy any version to any environment" button. How do you design the job? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Separate the promotion/deploy job from the CI pipeline, deploying existing artifacts only.

**What I'd do →**
1. Create a parameterized pipeline `deploy/shack-app` with `choice(name: 'ENV', choices: ['dev','staging','prod'])` and `string(name: 'TAG')`.
2. Validate the tag exists in the registry (`crane digest $IMAGE:$TAG`) and was built from `main`.
3. Gate prod with a stage `input` restricted to `release-managers`, and use folder permissions so only certain groups can build the job.
4. Record `currentBuild.displayName = "${params.ENV} ${params.TAG}"` for an audit-friendly history.

**Why →** Deploying artifacts by tag never rebuilds and gives a clear deployment log per environment.

**Interview tip →** Say that with GitOps, this job would open a PR to the config repo instead of calling `kubectl`.

</details>

### Q10. A stage should run only when files under `infra/` changed, but on new branches it never runs. Why, and what do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** `when { changeset 'infra/**' }` compares against changes since the previous build of that branch job. The first build of a new branch has no recorded changesets, so the condition is false.

**What I'd do →**
1. Compute the diff against the target branch yourself:
   ```groovy
   when { expression { sh(script: 'git diff --name-only origin/main...HEAD | grep -q "^infra/"', returnStatus: true) == 0 } }
   ```
2. Ensure the clone fetches `main` (not a shallow single-branch clone).
3. For PRs, use `CHANGE_TARGET` instead of hardcoding `main`.

**Why →** Diffing against the merge base is deterministic regardless of build history.

**Interview tip →** Knowing the first-build changeset gotcha signals real Jenkins experience.

</details>

### Q11. A team insists on scripted pipelines because "declarative is too restrictive". How do you respond? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Usually they hit one limitation (dynamic stages, complex conditions) and abandoned declarative entirely.

**What I'd do →**
1. Ask for the concrete limitation; most are solved with `script {}`, `matrix`, `when { expression {} }` or a library step.
2. Show dynamic `parallel` inside a `script` block for runtime-generated stages.
3. Point out what they lose with scripted: syntax validation, Restart from Stage, `post` conditions, consistent UI.
4. Agree on a rule: declarative skeleton, scripted logic only in reviewed library steps.

**Why →** Consistency across hundreds of repos matters more than per-team freedom.

**Interview tip →** Frame it as a standard with escape hatches, not a ban.

</details>

### Q12. Builds are green even though the test step printed failures, because the test command's exit code is swallowed by a pipe. How do you make the pipeline honest? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `sh 'npm test | tee test.log'` returns the exit code of `tee`, not `npm test`.

**What I'd do →**
1. Use `set -o pipefail` with bash:
   ```groovy
   sh '''#!/bin/bash
   set -euo pipefail
   npm test | tee test.log
   '''
   ```
2. Publish JUnit results with `allowEmptyResults: false`, so missing reports also fail.
3. Review other `sh` steps for `|| true` and pipes.

**Why →** `/bin/sh` on many images is `dash`, which lacks `pipefail`; the shebang forces bash.

**Interview tip →** Mention both the shell issue and making test reports the source of truth.

</details>

### Q13. You need the build to be UNSTABLE when coverage drops below 80% but still produce an artifact. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A soft gate: visible signal, no blocked delivery for this branch.

**What I'd do →**
1. Parse coverage in a step and call `unstable`:
   ```groovy
   script {
     def pct = sh(script: "jq '.total.lines.pct' coverage/coverage-summary.json", returnStdout: true).trim() as Double
     if (pct < 80) { unstable("Line coverage ${pct}% is below 80%") }
   }
   ```
2. Or use the Coverage plugin's `recordCoverage` with quality gates set to unstable.
3. Make `main` deploy stages require `currentBuild.currentResult == 'SUCCESS'` if unstable builds must not ship.

**Why →** UNSTABLE is Jenkins' built-in "passed with warnings" state.

**Interview tip →** Explain the difference between `result` and `currentResult`.

</details>

### Q14. Your pipeline uses `stash` to move a 3 GB build output between stages on different agents and the controller struggles. What do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Stashes are tarred through the controller and stored in `JENKINS_HOME`.

**What I'd do →**
1. Keep stages that need the output on one agent (nested stages under one parent `agent`).
2. If you must cross agents, push to an artifact store (Nexus raw repo, S3) and pull by build ID.
3. Install Artifact Manager on S3 so stash and archive go directly from agent to S3.
4. Stash only what is needed (`includes: 'dist/**'`), not the whole workspace.

**Why →** The controller is a scheduler, not a file server; large stashes cause heap and disk pressure.

**Interview tip →** Say "stash for kilobytes to a few MB, artifact store for anything big".

</details>

### Q15. A release pipeline needs two approvals: QA lead, then a release manager, each from a different group. How do you model it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Sequential gates with different submitters and audit.

**What I'd do →**
1. Two stages with stage-level `input` and no agent:
   ```groovy
   stage('QA sign-off') { options { timeout(time: 1, unit: 'DAYS') }
     input { message 'QA approved?'; submitter 'qa-leads'; submitterParameter 'QA_APPROVER' }
     steps { echo "QA: ${QA_APPROVER}" } }
   stage('Release approval') { options { timeout(time: 1, unit: 'DAYS') }
     input { message 'Release to prod?'; submitter 'release-managers'; submitterParameter 'RM_APPROVER' }
     steps { echo "RM: ${RM_APPROVER}" } }
   ```
2. Write approver names into `currentBuild.description` and the change ticket.
3. Ensure the same person cannot approve both (check `QA_APPROVER != RM_APPROVER` and fail).

**Why →** Group-restricted submitters plus recorded identities give separation of duties.

**Interview tip →** Mention that admins can always approve, so keep the admin group small.

</details>

### Q16. Some tests are flaky and fail one build in ten. Product wants green builds now. What do you do in Jenkins, and what do you refuse to do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Retrying the whole stage hides real failures; we need targeted mitigation and tracking.

**What I'd do →**
1. Retry at the test-framework level for known flaky tests only (`jest --retryTimes` scoped by tag, Maven Surefire `rerunFailingTestsCount=2`); Surefire reports flakes in XML.
2. Track flaky tests with JUnit history and quarantine them into a separate non-blocking stage using `catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE')`.
3. Use `retry` with `conditions: [agent(), kubernetesAgent()]` only for infra failures.
4. Refuse a blanket `retry(3)` around the test stage.

**Why →** Blind retries turn real intermittent bugs into invisible production incidents.

**Interview tip →** Separate infra flakiness (retry) from test flakiness (fix or quarantine).

</details>

### Q17. A Jenkinsfile must work for both PR builds (no push, no deploy) and `main` builds (push and deploy). Show the structure. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** One file, branch-aware stages.

**What I'd do →**
```groovy
stage('Push image') {
  when { branch 'main' }
  steps { pushImage() }
}
stage('Deploy dev') {
  when { allOf { branch 'main'; not { changeRequest() } } }
  steps { deploy('dev') }
}
```
1. All PR builds run build, test, scan.
2. Push and deploy stages run only on `main`.
3. Deploy credentials are scoped so PR jobs cannot use them even if someone edits the `when`.

**Why →** One Jenkinsfile avoids drift between PR and main pipelines.

**Interview tip →** Always pair `when` conditions with credential scoping; `when` alone is not a security control.

</details>

### Q18. Your pipeline should automatically roll back the Kubernetes deployment if the post-deploy smoke test fails. Design it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Deploy, verify, and undo on failure within the same stage.

**What I'd do →**
```groovy
stage('Deploy prod') {
  steps {
    container('kubectl') {
      sh 'kubectl -n shack-prod set image deploy/shack-app app=$IMAGE:$TAG'
      sh 'kubectl -n shack-prod rollout status deploy/shack-app --timeout=180s'
      sh './ci/smoke.sh https://shack.example.com'
    }
  }
  post {
    failure { container('kubectl') { sh 'kubectl -n shack-prod rollout undo deploy/shack-app' } }
  }
}
```
1. Annotate the change cause (`kubernetes.io/change-cause`) with the build URL.
2. Alert the channel on rollback.

**Why →** `rollout status` catches failed pods, the smoke test catches broken behaviour, and `post failure` handles both.

**Interview tip →** Mention progressive delivery (Argo Rollouts) as the next step beyond scripted rollback.

</details>

### Q19. Two teams copy the same 150-line Jenkinsfile with small differences. Each fix must be applied in 40 repos. What do you propose? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic case for a shared library.

**What I'd do →**
1. Identify the common steps (build, test, scan, push, deploy) and create library steps with sane defaults.
2. Offer a pipeline template step (`nodeServicePipeline(...)`) for standard services, plus building blocks for special ones.
3. Version the library (`v1`), migrate repos with automated PRs, and keep each Jenkinsfile under 20 lines.
4. Track adoption by logging library version per build.

**Why →** One fix lands everywhere when the moving tag is updated, with a controlled rollout.

**Interview tip →** Mention the trade-off: templates limit flexibility, so leave extension hooks.

</details>

### Q20. A pipeline runs a 3-hour performance test. If the controller restarts mid-run, the build is lost. How do you make long builds resilient? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Durability, agent survival and step design all matter.

**What I'd do →**
1. Use `durabilityHint('MAX_SURVIVABILITY')` for this pipeline.
2. Run the test on a static agent or a pod that survives controller restarts (pods are not deleted when the controller restarts; they reconnect if the controller returns within the agent's timeout).
3. Use plain `sh` steps (durable tasks) which keep running and writing output to the agent during the restart.
4. Do maintenance with quiet-down so new long builds do not start before a planned restart.
5. Better: trigger the perf test in an external system and have Jenkins wait on it, so Jenkins restarts do not matter.

**Why →** Pipeline resume needs persisted CPS state plus a surviving agent and a durable step.

**Interview tip →** Explain that `sh` is durable because output goes to files on the agent, not the channel.

</details>

## Shared libraries & multibranch

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q21. A shared library change merged to `main` broke 200 pipelines within minutes. How do you prevent a repeat? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pipelines loaded the library's default version `main`, so every merge was an instant global rollout.

**What I'd do →**
1. Set the global library's default version to a tag (`v1`), and have Jenkinsfiles pin `@Library('shack-lib@v1')`.
2. Release flow: PR with JenkinsPipelineUnit tests, merge, tag `v1.7.0`, run a canary set of pipelines on a staging controller, then move the `v1` tag.
3. Disable "Allow default version to be overridden" on production, or provide a separate `shack-lib-dev` config for testing branches.
4. Roll back by moving `v1` back to the previous tag.

**Why →** Tags turn library releases into deliberate, reversible events.

**Interview tip →** Say "the library is production code that runs unsandboxed, so it gets a release process".

</details>

### Q22. You want to test a library branch against a real app pipeline without affecting anyone. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need an isolated way to load an unreleased library version.

**What I'd do →**
1. On a staging controller or in a test folder, define a folder-level library `shack-lib` pointing to the same repo with version override allowed.
2. Use **Replay** on a recent app build and change the annotation to `@Library('shack-lib@feature/kaniko-cache') _`.
3. Or create a test branch of the app repo with that annotation.
4. Run JenkinsPipelineUnit tests in the library's own CI first.

**Why →** Folder-level libraries are sandboxed and scoped, so experiments cannot leak into production pipelines.

**Interview tip →** Mention Replay as the fastest feedback loop.

</details>

### Q23. Every app repo has its own pod YAML for builds, and they all drift. How do you standardize? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pod templates are platform concerns and belong in the library or cloud config.

**What I'd do →**
1. Put standard pod definitions in `resources/pods/node.yaml`, `maven.yaml` in the shared library.
2. Expose a step that wraps them:
   ```groovy
   // vars/withBuildPod.groovy
   def call(String type, Closure body) {
     podTemplate(yaml: libraryResource("pods/${type}.yaml")) { node(POD_LABEL) { body() } }
   }
   ```
3. Or define named templates in JCasC with a common base (`inheritFrom`) and let apps reference labels.
4. Allow overrides via a merge (`yamlMergeStrategy: merge()`) for resources only.

**Why →** One place to update images, security contexts and resource requests.

**Interview tip →** Mention that image bumps then happen via library releases, not 100 PRs.

</details>

### Q24. Security wants every pipeline to run a secret scan, even if teams forget to add it. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Enforcement must not rely on each Jenkinsfile.

**What I'd do →**
1. Make the standard pipeline template include the scan; teams using the template get it automatically.
2. For custom pipelines, use an implicitly loaded library with a `vars/` step called from the template, or enforce at org folder level with the **Pipeline: Multibranch with defaults** or **Templating Engine (JTE)** approach where the pipeline skeleton is defined centrally.
3. Add a branch protection required check that comes from a separate, centrally owned job (`security/secret-scan`), not from the team's Jenkinsfile.
4. Report pipelines missing the stage via build metadata.

**Why →** A required check produced outside the team's control cannot be removed by editing their Jenkinsfile.

**Interview tip →** Name the Jenkins Templating Engine as the "governance" option.

</details>

### Q25. A multibranch project has 600 stale branch jobs and the folder page takes 30 seconds to load. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Orphaned branches are kept forever, and old builds are not discarded.

**What I'd do →**
1. Set the orphaned item strategy to discard old items: days to keep 7, max 20.
2. Add `buildDiscarder(logRotator(numToKeepStr: '10'))` in the Jenkinsfile.
3. Filter discovery to relevant heads (`main`, `release/*`, PRs) with a wildcard filter.
4. Run **Scan Repository Now** to apply the cleanup.

**Why →** Each job and build directory is loaded into memory; fewer items means faster UI and smaller heap.

**Interview tip →** Mention coordinating with teams before deleting release branch history.

</details>

### Q26. A new org with 150 repositories wants Jenkins builds for every repo with a Jenkinsfile, without tickets to the platform team. What do you set up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Organization folder use case.

**What I'd do →**
1. Create a GitHub Organization folder with a GitHub App credential, defined in JCasC/Job DSL (`organizationFolder('acme') { organizations { github { repoOwner('acme') ... } } }`).
2. Discover repos with a `Jenkinsfile`, branches excluding PR branches, PRs merged with target, forks trusted only from collaborators.
3. Configure an org-level webhook to `/github-webhook/`.
4. Apply folder-level credentials and permissions for the org.
5. Set orphaned item strategy and a periodic scan (1 day) as backup.

**Why →** New repos are picked up automatically and builds start on the first push.

**Interview tip →** Mention GitHub App rate limits as the reason not to use a personal token at this scale.

</details>

### Q27. PRs from forks must be built, but the security team is worried about secrets. How do you configure it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fork PRs run untrusted code; the Jenkinsfile and build scripts can be attacker-controlled.

**What I'd do →**
1. Set fork PR trust to "From users with Admin or Write permission" (or Nobody), so untrusted forks run the target branch's Jenkinsfile.
2. Remember the target Jenkinsfile still runs the fork's code (tests, build scripts), so do not bind any secrets in PR stages.
3. Run PR builds on a separate, isolated agent pool (different namespace or cluster, no network to internal systems).
4. Require a maintainer comment or label to trigger builds for first-time contributors.

**Why →** Trust settings protect the pipeline definition, not the code it executes; isolation protects the rest.

**Interview tip →** This "trusted Jenkinsfile, untrusted code" distinction is what interviewers want to hear.

</details>

### Q28. You need a release pipeline that runs when a tag like `v1.4.0` is pushed. Tag jobs appear in multibranch but never build. Make it work. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tags are discovered but not built automatically by default.

**What I'd do →**
1. Add the "Discover tags" trait.
2. Install Basic Branch Build Strategies and add "Tags" with "Ignore tags older than 1 day" plus "Regular branches" and "Change requests" (adding any strategy replaces the default, so list all you need).
3. In the Jenkinsfile, gate release stages with `when { buildingTag() }` or `when { tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP' }`.
4. Use `TAG_NAME` as the version.

**Why →** The age limit prevents a flood of builds for historical tags on first indexing.

**Interview tip →** Mention that adding build strategies overrides the defaults, a common trap.

</details>

### Q29. A library step needs a 200-line bash script, and escaping it inside Groovy strings is causing bugs. How do you structure it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Groovy string escaping (`$`, `\`, quotes) is error-prone.

**What I'd do →**
1. Put the script in `resources/scripts/deploy.sh` in the library.
2. Load and run it:
   ```groovy
   writeFile file: '.ci/deploy.sh', text: libraryResource('scripts/deploy.sh')
   sh 'bash .ci/deploy.sh'
   ```
3. Pass inputs as environment variables with `withEnv`, not interpolation.
4. Lint the script with ShellCheck in the library's CI.

**Why →** The script stays a real shell file: testable, lintable, no Groovy escaping.

**Interview tip →** Mention injection safety as a bonus of env-var passing.

</details>

### Q30. A team's multibranch job builds every push to every branch, including `dependabot/*` branches that also have PRs. Builds double. Fix. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Branch discovery is set to "All branches".

**What I'd do →**
1. Set branch discovery to "Exclude branches that are also filed as PRs".
2. Keep PR discovery with "merge with target".
3. Optionally filter out `dependabot/*` branch builds with a name filter and rely on their PR builds.

**Why →** Each change is built once in the form that matters (the PR merge result).

**Interview tip →** Explain why merge strategy tests what will land on `main`.

</details>

### Q31. How would you add a common post-build step (send build metrics) to every pipeline on the controller? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cross-cutting concern; editing every Jenkinsfile does not scale.

**What I'd do →**
1. Prefer platform-level collection: the OpenTelemetry or Prometheus plugin captures build durations and results for all jobs without pipeline changes.
2. For custom data, add it to the shared pipeline template's `post { always { } }`.
3. For pipelines not using the template, an implicitly loaded library step can be called from a pipeline-level hook only if included, so combine with adoption tracking.

**Why →** Plugins observe all builds centrally; templates add custom logic where needed.

**Interview tip →** Mention OpenTelemetry tracing pipelines in Jaeger or Grafana Tempo.

</details>

### Q32. A library uses a class that stores the build's config in a static field. Builds started at the same time see each other's values. Why and how do you fix it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Library classes are loaded once per library version per controller classloader context and static fields are shared across concurrent builds.

**What I'd do →**
1. Remove static mutable state; pass config as method arguments or instance fields.
2. Create instances per build (`new Config(this, params)`) and keep them `Serializable`.
3. Add a JenkinsPipelineUnit test that simulates two configs.

**Why →** Static fields behave like global variables across builds, and CPS resume makes them even less predictable.

**Interview tip →** Say "library code must be stateless across builds".

</details>

### Q33. You must migrate 80 Freestyle jobs to multibranch pipelines. Plan it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Large migration; reduce risk and repetition.

**What I'd do →**
1. Inventory jobs with the API (`/api/json?tree=jobs[name,_class]`) and group by pattern (Maven app, Node app, cron script).
2. Build library steps or templates for each pattern.
3. Add Jenkinsfiles via automated PRs, generating the parameters from each job's `config.xml`.
4. Create multibranch jobs via Job DSL in JCasC; run both old and new in parallel for a week, comparing results.
5. Disable Freestyle jobs, then delete after a retention period.

**Why →** Pattern-based templates make the 80 jobs mostly mechanical, and parallel running catches gaps.

**Interview tip →** Mention the cron/admin jobs might stay as simple pipelines, not every job needs multibranch.

</details>

### Q34. A global library is configured as "Load implicitly", and a developer asks why their pipeline got a new behaviour without any change on their side. How do you handle it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Implicit libraries load the default version into every pipeline automatically.

**What I'd do →**
1. Explain the implicit load and show the build log line `Loading library shack-lib@v1`.
2. Check the library changelog between the last good and current builds.
3. Advise pinning a specific tag in their Jenkinsfile if they need stability (`@Library('shack-lib@v1.6.2') _`).
4. Review whether implicit loading is justified; restrict it to small, stable utilities.

**Why →** Implicit loading trades explicitness for convenience; teams must know it exists.

**Interview tip →** Recommend explicit `@Library` in all pipelines for readability.

</details>

### Q35. You want branch builds and PR builds to report separate statuses to GitHub and block merges when either fails. Configure it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** GitHub branch protection needs the right context names.

**What I'd do →**
1. Note Jenkins contexts: `continuous-integration/jenkins/branch` and `continuous-integration/jenkins/pr-merge`.
2. In branch protection for `main`, require `continuous-integration/jenkins/pr-merge` (PRs are what merge).
3. Use the GitHub Checks plugin with a GitHub App to publish per-stage checks if you want "SonarQube" and "Trivy" as separate required checks via `publishChecks`.
4. Verify the credential can write statuses.

**Why →** Requiring the PR-merge context guarantees the merged result was built.

**Interview tip →** Mention that status names must match exactly or the PR waits forever.

</details>

### Q36. A team needs the same pipeline for 25 microservices in one monorepo, but only changed services should build. Design it in Jenkins. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** One multibranch job, dynamic stages based on the diff.

**What I'd do →**
1. Compute changed services:
   ```groovy
   def base = env.CHANGE_TARGET ? "origin/${env.CHANGE_TARGET}" : 'HEAD~1'
   def changed = sh(script: "git diff --name-only ${base}...HEAD | cut -d/ -f2 | sort -u", returnStdout: true).trim().split('\n')
   ```
2. Build a `parallel` map with one branch per changed service, each calling a library step.
3. Force all services to build on `main` nightly or when shared code (`libs/`) changes.
4. Cache dependencies per service via a repository manager.

**Why →** Dynamic parallel keeps one pipeline while avoiding 25 full builds per commit.

**Interview tip →** Mention that for very large monorepos a graph-aware build tool beats shell diffs.

</details>

### Q37. You need to share configuration (registry URL, Sonar server name, cluster names) across all pipelines without hardcoding. Where do you put it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Non-secret global settings.

**What I'd do →**
1. Global environment variables set via JCasC (`jenkins.globalNodeProperties.envVars`), or folder-level properties for per-team values.
2. Or a library `resources/config/defaults.yaml` read via `readYaml text: libraryResource(...)`.
3. Keep secrets in credentials, never in these.

**Why →** Values change in one place and are versioned with the platform code.

**Interview tip →** Folder properties allow team-specific overrides with the same variable names.

</details>

### Q38. A team asks to write a `vars/` step that calls Jenkins internal APIs (`Jenkins.instance.getItem`) to list jobs. Should it live in the trusted library? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Trusted library code runs unsandboxed; any step there becomes a capability every pipeline gets.

**What I'd do →**
1. Ask what the real need is; often the REST API or a plugin step answers it.
2. If needed, implement a narrow step (`listJobsInFolder(folder)`) that returns names only, validates input, and checks the caller's folder.
3. Review it like privileged code; never expose generic "run this Groovy" helpers.
4. Prefer doing such admin tasks in a separate admin-only job.

**Why →** A trusted library is an admin API surface for all Jenkinsfile authors.

**Interview tip →** Tie it to the principle of least privilege.

</details>

## Platform: plugins, JCasC & Kubernetes agents

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q39. You inherit a controller with 230 plugins, half unused, on a version two LTS lines behind. Plan the cleanup and upgrade. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Large surface, upgrade risk compounds with plugin count.

**What I'd do →**
1. Export the plugin list and current config (`/configuration-as-code/viewExport`), back up `JENKINS_HOME`.
2. Find usage with the Plugin Usage plugin and by grepping `jobs/*/config.xml` and Jenkinsfiles for steps; mark unused plugins (dependencies of used ones stay).
3. Clone production to staging, remove unused plugins there, check for "old data" and broken jobs.
4. Build an image with the target LTS and `jenkins-plugin-cli --jenkins-version <target> --available-updates`.
5. Upgrade staging, run smoke pipelines, then production with quiet-down and a rollback image.
6. Move everything into `plugins.txt` and JCasC so this never accumulates again.

**Why →** Removing plugins before upgrading reduces conflicts and testing scope.

**Interview tip →** Mention the Java 17 requirement as a classic upgrade blocker.

</details>

### Q40. An admin changes a cloud setting in the UI, and it reverts after the next controller restart. Developers are confused. What is your policy? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** JCasC owns that section and reapplies it at startup.

**What I'd do →**
1. Explain JCasC as the source of truth and make the change via PR to `jenkins.yaml`.
2. Add a system message: "Configured by JCasC, UI changes will be lost".
3. Restrict `Overall/Administer` to the platform team.
4. Use the JCasC export to diff UI state against Git periodically.

**Why →** Mixed ownership leads to silent reverts and outages.

**Interview tip →** Say "the UI is read-only for configuration on our controllers".

</details>

### Q41. You must stand up a new Jenkins controller for a team in under an hour, identical to the existing ones. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Needs image + JCasC + Job DSL + secrets.

**What I'd do →**
1. Reuse the platform controller image (pinned core and plugins).
2. Create a values/overlay file for the team: URL, SSO group mapping, folder names, credentials references.
3. `helm upgrade --install jenkins-payments jenkins/jenkins -f base.yaml -f payments.yaml -n jenkins-payments`.
4. Job DSL in JCasC creates the team's org folder; indexing creates all jobs.
5. Verify with a smoke pipeline.

**Why →** Everything is parameterized code, so a controller is a deployment, not a project.

**Interview tip →** Mention the same mechanism doubles as disaster recovery.

</details>

### Q42. Builds wait 2–3 minutes before starting because new agent pods are slow to schedule. How do you speed up agent startup? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Time goes into node scale-up, image pulls and the agent connecting.

**What I'd do →**
1. Check pod events: `kubectl -n jenkins-agents describe pod <pod>` for `Pulling image` duration and `FailedScheduling`.
2. Use smaller, pinned images and pre-pull them on build nodes (a DaemonSet that pulls them, or node image baking).
3. Keep a small warm buffer: overprovisioning pause pods with low priority so the autoscaler keeps spare nodes.
4. Set `imagePullPolicy: IfNotPresent` with immutable tags.
5. Consider `idleMinutes` for very frequent short jobs.

**Why →** Pod startup is mostly image pulls and node provisioning, both fixable at the cluster level.

**Interview tip →** Quote the breakdown: scheduling, pull, container start, agent connect.

</details>

### Q43. Developers need Docker image builds in Kubernetes agents, but the cluster runs containerd and security forbids privileged pods. What do you choose? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** No Docker socket, no privileged containers.

**What I'd do →**
1. Evaluate Kaniko (runs as root inside the container but not privileged) or BuildKit rootless (needs `seccomp`/`apparmor` unconfined but not privileged).
2. Run builders on a dedicated tainted node pool with no access to production networks.
3. Push to the registry with short-lived credentials; use a registry cache repo for layer caching.
4. If policy forbids root entirely, use a remote BuildKit service run by the platform team or a managed builder.

**Why →** Daemonless builders avoid the host socket and fit containerd clusters, with isolation compensating for their remaining privileges.

**Interview tip →** Mention Kaniko's archived upstream status and that you would track a maintained fork or BuildKit.

</details>

### Q44. The controller needs to provision agents in three Kubernetes clusters in different regions. How do you configure it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multiple clouds in the kubernetes plugin, external API access.

**What I'd do →**
1. Define three `kubernetes` clouds in JCasC, each with `serverUrl`, `serverCertificate`, `namespace` and a credential (service account token or kubeconfig as a Secret file credential).
2. Set `jenkinsUrl` to the public HTTPS URL and `webSocket: true` so remote pods connect through the ingress.
3. Give each cloud distinct pod template labels (`k8s-eu`, `k8s-us`) and set capacity limits.
4. Create RBAC in each cluster for the Jenkins service account in the agents namespace.

**Why →** WebSocket over HTTPS avoids exposing TCP 50000 across regions.

**Interview tip →** Mention that one controller across regions still has one point of failure; consider regional controllers.

</details>

### Q45. You want to roll out a new version of the `jnlp` inbound-agent image to all pod templates. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The `jnlp` image is set per template or defaulted by the plugin.

**What I'd do →**
1. Define the `jnlp` container once in a base template in JCasC (`name: base`, container `jnlp` with `image: jenkins/inbound-agent:<version>-jdk21`).
2. Make other templates `inheritFrom: base`.
3. Bump the version in JCasC via PR, reload, run smoke builds.

**Why →** Inheritance means one change propagates; agents with old remoting versions can also trigger compatibility warnings.

**Interview tip →** Keep the remoting version close to the controller's for compatibility.

</details>

### Q46. A Job DSL seed in JCasC should create a folder per team from a YAML list. How do you structure it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Data-driven job generation.

**What I'd do →**
1. Store `teams.yaml` in the platform repo, mounted into the controller.
2. Job DSL script:
   ```groovy
   def teams = new org.yaml.snakeyaml.Yaml().load(readFileFromWorkspace('teams.yaml'))
   teams.each { t ->
     folder(t.name) { displayName(t.display) }
     organizationFolder("${t.name}/repos") { /* github org config */ }
   }
   ```
   (When run from JCasC, read the file with plain Groovy `new File('/var/jenkins_config/teams.yaml').text`.)
3. Apply folder permissions and credentials per team in the same script.

**Why →** Adding a team is a one-line YAML change reviewed in Git.

**Interview tip →** Mention Job DSL's `removedJobAction` for cleaning up deleted teams.

</details>

### Q47. How do you prevent one team's builds from using all agent capacity in a shared Kubernetes cluster? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fairness and isolation across teams.

**What I'd do →**
1. Separate agent namespaces per team with `ResourceQuota` and `LimitRange`.
2. A kubernetes cloud per namespace (or per team controller) with its own concurrency limit.
3. Enforce resource requests in pod templates so scheduling is predictable.
4. Monitor queue time per team.

**Why →** Quotas cap usage at the cluster level, regardless of what Jenkins requests.

**Interview tip →** Mention priority classes for release builds over PR builds.

</details>

### Q48. Your controller runs on Kubernetes. A node drain for an upgrade kills the controller pod mid-day. How do you make maintenance safe? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Single-replica StatefulSet is disrupted by drains.

**What I'd do →**
1. Add a PodDisruptionBudget with `maxUnavailable: 0` so drains block until an operator acts.
2. Schedule maintenance windows: quiet-down via API, wait for builds, then drain.
3. Set a long `terminationGracePeriodSeconds` and a `preStop` hook that calls `/safeExit` if desired.
4. Put the controller on a dedicated node pool upgraded separately.

**Why →** The controller is a stateful singleton; it needs explicit handling during cluster operations.

**Interview tip →** Mention that build pods also need PDB thinking or tolerance to eviction.

</details>

### Q49. A plugin you rely on has been abandoned and has an unfixed security advisory. What are your options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Risk accepted, mitigated or removed.

**What I'd do →**
1. Read the advisory: what permission is needed to exploit it? Maybe only admins can trigger it.
2. Look for a replacement plugin or replace with `sh` steps calling a CLI.
3. If it must stay short term, reduce exposure (restrict permissions, disable affected features) and document the risk acceptance with an expiry.
4. Consider adopting the plugin (the Jenkins project has an "adopt a plugin" process).

**Why →** Abandoned plugins only get worse with every core upgrade.

**Interview tip →** Prefer CLI tools in containers over niche plugins in general.

</details>

### Q50. A team needs GPU agents only for model training jobs, which run twice a week. How do you provide them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Expensive, rare resource; must not idle.

**What I'd do →**
1. A pod template labelled `gpu` with `nodeSelector` for the GPU node pool, `tolerations` for its taint, and `resources.limits: nvidia.com/gpu: 1`.
2. Cluster autoscaler scales the GPU pool from zero when a pod is pending.
3. Restrict the template to the ML folder (separate cloud with folder restriction or Job Restrictions).
4. Set a pipeline timeout so jobs do not hold GPUs indefinitely.

**Why →** Ephemeral pods on scale-to-zero nodes mean GPUs cost money only during training.

**Interview tip →** Mention `instanceCap` to limit concurrent GPU pods.

</details>

### Q51. You want every controller config change reviewed, but people still need to test changes quickly. Design the workflow. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** GitOps for Jenkins itself.

**What I'd do →**
1. Platform repo with `Dockerfile`, `plugins.txt`, `casc/*.yaml`, `jobs/*.groovy`.
2. PR pipeline: build the image, start it in a container with test secrets, fail on `ConfiguratorException` in logs, run a smoke Jenkinsfile through the API.
3. Ephemeral preview controller per PR in a sandbox namespace for manual checks.
4. Merge deploys to staging, then production via Helm with an approval.

**Why →** Fast feedback plus review without anyone touching production UI.

**Interview tip →** "We test Jenkins with Jenkins" usually lands well.

</details>

### Q52. Pods for builds must pull images from a private registry. How do you configure it cleanly? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Kubernetes pull secrets for agent pods.

**What I'd do →**
1. Create a `docker-registry` Secret in the agents namespace:
   ```bash
   kubectl -n jenkins-agents create secret docker-registry regcred \
     --docker-server=registry.example.com --docker-username=ci --docker-password="$PASS"
   ```
2. Add `imagePullSecrets: [{name: regcred}]` to the base pod template, or attach it to the agents' service account.
3. Prefer node-level registry auth (ECR credential helper, workload identity) in cloud clusters.

**Why →** Pull secrets are a cluster concern; Jenkins credentials are not involved in image pulls.

**Interview tip →** Distinguish pulling build images (cluster) from pushing built images (pipeline credentials).

</details>

### Q53. Two plugins require different versions of a shared dependency, and `jenkins-plugin-cli` fails the image build. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dependency conflict, typically one plugin is too old for the other's requirement.

**What I'd do →**
1. Read the error; it names the plugin and required version.
2. Update the older plugin to a version compatible with the newer dependency (check its changelog and required core).
3. If one plugin is abandoned and blocks the upgrade, replace or remove it.
4. Let `jenkins-plugin-cli` resolve transitive dependencies; pin only top-level plugins plus any you explicitly need.

**Why →** Over-pinning transitive dependencies causes many of these conflicts.

**Interview tip →** Show you know the difference between top-level and transitive pins.

</details>

### Q54. An auditor asks you to prove the Jenkins configuration in production equals what is in Git. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Drift detection.

**What I'd do →**
1. A scheduled job exports `/configuration-as-code/viewExport` (admin token) and diffs it against the rendered YAML in Git, ignoring generated fields.
2. Compare installed plugins (`pluginManager/api/json`) with `plugins.txt`.
3. Use Job Configuration History to show any UI changes with authors.
4. Alert on drift and enforce read-only by restricting admin rights.

**Why →** Continuous drift detection is the evidence auditors want.

**Interview tip →** Mention the image digest in the running pod as evidence for core and plugin versions.

</details>

### Q55. Builds in Kubernetes need a Maven cache, and downloads take 6 minutes per build. What do you implement? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ephemeral pods start with an empty `~/.m2`.

**What I'd do →**
1. Point Maven at a Nexus/Artifactory proxy inside the cluster via `settings.xml` mirror; this alone usually drops time to under a minute.
2. Optionally mount a shared PVC for `~/.m2/repository` (ReadWriteMany), accepting concurrency risks, or a per-node cache.
3. Use `mvn -B -T 1C` and dependency:go-offline in a cached image layer for stable dependencies.

**Why →** A nearby proxy turns internet downloads into LAN transfers and adds resilience.

**Interview tip →** Mention that baking dependencies into the build image is another option for slow-changing deps.

</details>

### Q56. Agents on Windows VMs must run alongside Kubernetes pods. How do you manage them as code? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Static agents need definitions and a connection method.

**What I'd do →**
1. Define the nodes in JCasC (`jenkins.nodes: - permanent: name, labelString, remoteFS, launcher: inbound with webSocket`).
2. Install the agent as a Windows service using the Jenkins agent service wrapper (WinSW) with `-webSocket` and the secret from a secure store.
3. Build the VM image with Packer (JDK 21, build tools) and replace VMs instead of patching.
4. Label them `windows`, `EXCLUSIVE` mode.

**Why →** Both node definition and machine setup become reproducible.

**Interview tip →** Mention the EC2 or Azure VM cloud plugins for ephemeral Windows agents.

</details>

## Credentials & security

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q57. A security review finds AWS access keys stored as Jenkins credentials used by 60 pipelines. The goal is no long-lived keys. Plan it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Replace static keys with workload identity or short-lived tokens.

**What I'd do →**
1. For Kubernetes agents on EKS, use IRSA or EKS Pod Identity: deploy pod templates with a service account mapped to an IAM role; the AWS SDK picks up credentials automatically.
2. For agents outside AWS, use OIDC federation (e.g. the controller issues tokens via a plugin) or Vault's AWS secrets engine issuing 15-minute credentials per build.
3. Migrate pipelines folder by folder, then delete the keys and deactivate them in IAM.
4. Scope roles per environment; prod role only usable by the prod deploy template.

**Why →** Short-lived credentials limit blast radius when a build or log leaks them.

**Interview tip →** Name IRSA/Pod Identity explicitly; it is the expected answer for EKS.

</details>

### Q58. A developer printed a credential with `echo $TOKEN | base64` and it appeared unmasked in the log. What do you do now and later? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Masking only matches the exact value; the secret is exposed.

**What I'd do →**
1. Rotate the credential immediately; treat it as compromised.
2. Delete the affected build logs (or the builds) and check who viewed them.
3. Tighten who can modify pipelines that bind that credential: folder-scoped credentials, PR review for Jenkinsfiles, no Replay for non-admins.
4. Add a log scan (e.g. gitleaks over archived logs) for leaked patterns.

**Why →** Anyone who can run code with a secret can exfiltrate it; masking is not a control.

**Interview tip →** Say "rotate first, investigate second".

</details>

### Q59. Team A can see Team B's production kubeconfig in the credential dropdown. Fix the model. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Credentials are global; access is not segmented.

**What I'd do →**
1. Create per-team folders and move each team's credentials into their folder's store.
2. Grant `Credentials/View` and `Job/Configure` only within each team's folder (project matrix or role strategy).
3. Remove the global copies after migration.

**Why →** Folder credentials are only resolvable by jobs inside the folder.

**Interview tip →** Mention that `Job/Configure` in a folder implies use of its credentials.

</details>

### Q60. Jenkins must be reachable by GitHub webhooks, but the UI must stay internal. Design the exposure. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Split public and private paths.

**What I'd do →**
1. Internal ingress/load balancer for the full UI, behind VPN or SSO proxy.
2. A separate public ingress that routes only `/github-webhook/` (and `/sonarqube-webhook/` if Sonar is external) to the Jenkins service; return 404 for everything else.
3. Restrict the public path to GitHub's hook IP ranges and require the webhook secret.
4. Alternatively use a relay (a small service in a DMZ, or GitHub's webhook forwarding via a queue) so Jenkins has no inbound exposure.

**Why →** Minimal public surface while keeping instant builds.

**Interview tip →** Mention webhook HMAC verification as a second layer.

</details>

### Q61. Your script approval list contains `method hudson.model.Run getLog` and `staticMethod jenkins.model.Jenkins getInstance`. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** These approvals give every Jenkinsfile author access to Jenkins internals: `Jenkins.getInstance` leads to credentials and configuration.

**What I'd do →**
1. Find which pipelines use them (search Jenkinsfiles and recent `RejectedAccessException` history).
2. Replace with safe alternatives: library steps that do the specific job, or plugin steps.
3. Revoke the dangerous approvals in `In-process Script Approval`.
4. Audit recent builds for abuse; rotate high-value credentials if exploitation is plausible.

**Why →** Approving internal API access effectively makes all pipeline authors admins.

**Interview tip →** Demonstrate that you know which signatures are dangerous, not just "approve carefully".

</details>

### Q62. A contractor needs to trigger and view builds for one project for three months. How do you grant access? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Least privilege with an expiry.

**What I'd do →**
1. Add the contractor to an IdP group with an expiry date (e.g. `ext-shack-builders`).
2. Grant that group `Overall/Read` and on the project folder `Job/Read`, `Job/Build`, `Job/Cancel` only.
3. No `Job/Configure`, `Run/Replay` or credential permissions.
4. Review access at expiry.

**Why →** Group-based grants in the IdP give central revocation and audit.

**Interview tip →** Avoid local Jenkins users for humans.

</details>

### Q63. You need to give a release bot a token to call the Jenkins API. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Machine identity with minimal permissions.

**What I'd do →**
1. Create a service account user in the IdP or a local Jenkins user `svc-release-bot`.
2. Grant only the permissions it needs on specific folders (`Job/Build`, `Job/Read`).
3. Generate a named API token, store it in the bot's secret store, rotate periodically.
4. Monitor token last-use and revoke unused ones.

**Why →** Separate identity makes audit clear and limits damage if leaked.

**Interview tip →** Mention that API tokens do not need CSRF crumbs.

</details>

### Q64. A pentest shows Jenkins is behind a reverse proxy but the "It appears that your reverse proxy set up is broken" warning is shown and some redirects go to `http://`. What do you fix? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The proxy does not forward the right headers, or the Jenkins URL is wrong.

**What I'd do →**
1. Set `unclassified.location.url: https://jenkins.example.com/` in JCasC.
2. Configure the proxy to pass `Host`, `X-Forwarded-Proto: https`, `X-Forwarded-Port: 443` and `X-Forwarded-For`, and to proxy WebSocket upgrades.
3. For nginx: `proxy_set_header X-Forwarded-Proto $scheme;` and `proxy_redirect` correct; for ingress-nginx the defaults usually work with the URL set.
4. Enable HSTS at the proxy.

**Why →** Jenkins builds absolute URLs from its configured URL and forwarded headers.

**Interview tip →** Mention WebSocket agents also need upgrade headers through the proxy.

</details>

### Q65. Jenkinsfiles in a repo deploy to production, and any developer with write access could edit the deploy stage in a branch. How do you prevent misuse? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pipeline code in a branch runs with whatever credentials it can reach.

**What I'd do →**
1. Prod credentials live in a separate folder used by a separate deploy job whose Jenkinsfile comes from a protected platform repo, not the app repo.
2. The app pipeline only publishes an artifact and triggers the deploy job with the tag.
3. The deploy job uses the Authorize Project plugin and an `input` restricted to release managers.
4. Branch protection and CODEOWNERS on `Jenkinsfile` changes in app repos.

**Why →** Separating "who writes the pipeline" from "who can deploy" is the only real fix.

**Interview tip →** Or mention GitOps: Jenkins has no prod credentials at all.

</details>

### Q66. You suspect a malicious build exfiltrated data. What Jenkins evidence do you gather? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Incident response on CI.

**What I'd do →**
1. Preserve the build directory (`jobs/<job>/builds/<n>/`): log, `build.xml` (cause, parameters), flow nodes.
2. Collect Audit Trail logs, Job Configuration History, and reverse proxy logs for the period.
3. Check which credentials the job could bind and when the Jenkinsfile last changed (git history).
4. Check agent pod/node logs and network egress logs (NetworkPolicy/firewall flow logs).
5. Rotate every credential the job could access; restrict the job and review similar pipelines.

**Why →** Jenkins data plus network logs reconstruct who ran what with which secrets.

**Interview tip →** Mention egress restrictions on build agents as a preventive control.

</details>

### Q67. A team wants to SSH into production servers from pipelines. How do you implement it safely? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** SSH keys in CI are high value.

**What I'd do →**
1. Store the key as an SSH credential in the team's prod folder only.
2. Use `sshagent(['prod-deploy-key']) { sh 'ssh -o StrictHostKeyChecking=yes deploy@host ...' }` with known hosts pinned in the repo or image.
3. Limit the key on the server side (`command=` in `authorized_keys`, non-root user, `from=` agent IPs).
4. Prefer SSH certificates with short validity issued by Vault per build.

**Why →** Narrow key capabilities and short lifetimes reduce impact of leaks.

**Interview tip →** Never disable host key checking in pipelines.

</details>

### Q68. Your org requires that only signed and scanned images reach production. Where does Jenkins fit? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Jenkins produces evidence; the cluster enforces it.

**What I'd do →**
1. Pipeline: build to tarball, Trivy scan with gate, push, sign with cosign (keyless via OIDC or a KMS key), attach SBOM and scan attestations.
2. Cluster: admission policy (Kyverno `verifyImages` or Sigstore policy-controller) that rejects unsigned or unattested images in prod namespaces.
3. Jenkins never holds a long-lived signing key in plain credentials; use KMS.

**Why →** Enforcement at admission means bypassing Jenkins does not bypass the policy.

**Interview tip →** "Jenkins signs, the cluster verifies".

</details>

### Q69. A user reports they can see the whole job configuration including masked passwords in XML at `/job/x/config.xml`. Is this a leak? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Users with `Job/ExtendedRead` or `Job/Configure` can read `config.xml`. Secrets in it are encrypted (`{AQAAABAAAA...}`), but anyone with script console access could decrypt them.

**What I'd do →**
1. Check who has `Job/Configure`/`ExtendedRead` on that job.
2. Move secrets out of job parameters/configs into the credentials store and reference by ID.
3. Use password parameters sparingly; they are stored encrypted but visible to configurers in encrypted form.

**Why →** Encrypted-at-rest secrets are still only as safe as the admin boundary.

**Interview tip →** Know that the `{AQAAAB...}` format is a Jenkins `Secret`.

</details>

### Q70. You must enforce MFA for Jenkins access. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Jenkins has no native MFA; delegate to the IdP.

**What I'd do →**
1. Configure the OpenID Connect or SAML security realm against Okta/Entra/Keycloak with MFA policies.
2. Disable local accounts except one break-glass admin stored in a vault.
3. API tokens remain for automation, tied to service users with minimal permissions.

**Why →** SSO centralizes MFA, joiner/leaver processes and audit.

**Interview tip →** Mention the break-glass account and testing it.

</details>

### Q71. The security team wants all build agents to have no internet access except to approved registries. How do you implement it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Egress control for CI.

**What I'd do →**
1. Kubernetes NetworkPolicy in the agents namespace: default-deny egress, allow DNS, the Jenkins service, Nexus/Artifactory, the container registry and SonarQube.
2. Route all dependency downloads through the repository manager (which has internet access and scans packages).
3. For cloud APIs, use VPC endpoints.
4. Monitor denied flows to find breakage.

**Why →** Blocks exfiltration and dependency confusion attacks while keeping builds working.

**Interview tip →** Mention that this also improves reproducibility because everything is proxied.

</details>

### Q72. Somebody added `Overall/Administer` to the `authenticated` group "temporarily". How do you detect and prevent this? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** UI change to security config.

**What I'd do →**
1. Revert immediately via JCasC reload, which restores the defined authorization.
2. Find the change in Audit Trail or Job Configuration History (system config).
3. Make authorization JCasC-only and alert on drift.
4. Reduce admins, require SSO+MFA.

**Why →** Codified security config makes unauthorized changes temporary and visible.

**Interview tip →** Mention periodic JCasC re-apply as a self-healing control.

</details>

### Q73. Pipelines need database passwords at deploy time from AWS Secrets Manager. Compare two ways to integrate. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Credentials provider vs runtime fetch.

**What I'd do →**
1. **Credentials provider plugin**: secrets tagged in Secrets Manager appear as Jenkins credentials; pipelines use `withCredentials` normally; Jenkins needs IAM read on those secrets.
2. **Runtime fetch**: the deploy pod uses IRSA and runs `aws secretsmanager get-secret-value`; Jenkins never sees the secret store.
3. Even better, let the application read secrets at runtime (External Secrets Operator), so the pipeline never handles them.

**Why →** Each step away from Jenkins reduces how many places hold the secret.

**Interview tip →** Mention least privilege on the IAM policy per environment.

</details>

### Q74. Your Jenkins exposes the TCP agent port 50000 to the internet for remote agents. What do you change? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unnecessary exposed port.

**What I'd do →**
1. Switch agents to WebSocket (`-webSocket`) so they connect over HTTPS 443.
2. Disable the TCP agent port (`jenkins.slaveAgentPort: -1` in JCasC).
3. Close the firewall rule.

**Why →** One HTTPS endpoint behind the proxy with TLS is simpler to secure.

**Interview tip →** Mention agent secrets still authenticate the connection either way.

</details>

## Operations: scale, HA, upgrades & performance

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q75. Jenkins is a single VM with local disk and no backups. The business wants an RTO of 1 hour. Design recovery. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Make the controller reproducible and protect what is not.

**What I'd do →**
1. Codify: controller image with plugins, JCasC, Job DSL for jobs.
2. Externalize: credentials to a secret store, artifacts to Nexus/S3.
3. Back up `JENKINS_HOME` (minus workspaces) nightly to versioned S3; keep `secrets/` encrypted.
4. Runbook: provision a new controller (Terraform/Helm), restore backup or accept history loss, update DNS, verify agents.
5. Run a recovery drill quarterly and time it.

**Why →** With config as code, recovery is deployment plus a restore, comfortably under an hour.

**Interview tip →** State RPO too: nightly backup means up to 24 h of build history loss.

</details>

### Q76. Queue length spikes to 80 every morning at 9:30 and builds wait 15 minutes. How do you analyse and fix? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Capacity vs demand mismatch, or a provisioning bottleneck.

**What I'd do →**
1. Check `/queue/api/json` why-reasons: no executors, cloud capacity reached, or pods pending.
2. Look at metrics: executor utilisation, cloud `containerCap`, cluster autoscaler events.
3. Raise cloud caps and namespace quotas if the cluster has room; add node pool headroom before 9:30 (scheduled scale-up).
4. Reduce demand: `abortPrevious` on PRs, skip unchanged services, avoid cron jobs at 9:30.

**Why →** Most queue spikes are caps or autoscaler lag, not Jenkins itself.

**Interview tip →** Always ask "what is the why-reason in the queue" first.

</details>

### Q77. The controller has 12 GB heap and still does full GCs every few minutes. What do you investigate? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Memory pressure from data volume or a leak.

**What I'd do →**
1. GC logs and `jstat -gcutil` to confirm old-gen growth after GC.
2. Heap histogram (`jcmd <pid> GC.class_histogram`) or a heap dump analysed with Eclipse MAT: look for `WorkflowRun`, `FlowNode`, huge `String`s, a specific plugin's classes.
3. Common causes: thousands of retained builds loaded in memory, pipelines with tens of thousands of steps (huge flow graphs), massive console logs, a leaking plugin.
4. Fix: `buildDiscarder`, reduce step counts (fewer `echo` in loops), upgrade/remove the leaking plugin; only then add heap.

**Why →** More heap delays the problem; data volume and leaks are usually the root.

**Interview tip →** Mention that step count matters: each step creates flow nodes held in memory.

</details>

### Q78. You need to upgrade from Jenkins 2.426 LTS (Java 11) to the current LTS. What is your sequence? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two big changes: Java runtime and core/plugins.

**What I'd do →**
1. First switch the existing version to Java 17 (2.426 supports 17), controller and agents; validate.
2. Then upgrade core in LTS steps if the gap is large, reading each upgrade guide.
3. Update plugins with `jenkins-plugin-cli --jenkins-version <target> --available-updates` in staging.
4. Test on a restored copy of production data, then upgrade production with a backup and rollback image.

**Why →** Separating Java and Jenkins upgrades isolates failures.

**Interview tip →** Remember agents also need Java 17+ for current remoting.

</details>

### Q79. Build history is needed for 2 years for compliance, but the controller disk is filling. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Compliance retention conflicts with controller performance.

**What I'd do →**
1. Keep only recent builds in Jenkins (e.g. 30 days).
2. Export build metadata and logs to long-term storage before deletion: a `post { always { } }` step that uploads the log and a JSON summary to S3 with object lock, or a log shipping plugin.
3. Store release artifacts in the repository manager with retention policies.
4. Document the evidence trail (commit, build URL, artifact digest, approver).

**Why →** Jenkins is not an archive; compliance data belongs in immutable storage.

**Interview tip →** Mention object lock (WORM) for audit evidence.

</details>

### Q80. A plugin update caused the controller to fail on startup in production. Recover. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need fast rollback.

**What I'd do →**
1. Roll back to the previous controller image (with the old `plugins.txt`); with Kubernetes, `helm rollback` or revert the image tag.
2. If plugins were installed into `JENKINS_HOME` manually, remove the bad `.jpi` from `plugins/` (or rename to `.jpi.disabled`) and restart.
3. Check logs for the root cause (`SEVERE: Failed Loading plugin`).
4. Reproduce in staging before retrying.

**Why →** Image-based plugins make rollback a redeploy.

**Interview tip →** Mention that plugin data migrations can make rollback partial; always back up before upgrades.

</details>

### Q81. Your controller serves 2,000 developers and 4,000 jobs. When do you split, and how? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scale and blast radius.

**What I'd do →**
1. Metrics to decide: startup time, heap, UI latency, how often a restart affects everyone, plugin conflicts.
2. Split by business domain into 4–6 controllers, each generated from the same image and JCasC base with team overlays.
3. Shared agent clusters (separate namespaces per controller), shared libraries, central SSO.
4. A small portal or naming scheme so developers know which controller is theirs.

**Why →** Smaller controllers are faster, safer to upgrade, and isolate failures.

**Interview tip →** Mention CloudBees CI as the commercial path if central management is needed.

</details>

### Q82. Console pages of long builds take 40 seconds to load and freeze browsers. How do you fix it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Huge logs.

**What I'd do →**
1. Reduce noise: `mvn -B -q` or `-ntp` (no transfer progress), `npm ci --no-progress`, avoid `set -x` in loops.
2. Write verbose output to files and archive them compressed.
3. Use the Pipeline Graph View to jump to the failing step's log only.
4. Discard old builds.

**Why →** Logs are stored and rendered by the controller; less output means faster everything.

**Interview tip →** `-ntp` for Maven is a quick win many forget.

</details>

### Q83. How do you know if Jenkins is healthy? Define SLIs and alerts. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Treat Jenkins as a product with SLOs.

**What I'd do →**
1. SLIs: queue wait time p95, build start success (pods connect), controller availability (`/login` 200), UI latency, webhook-to-start latency.
2. Saturation: heap usage, executor utilisation, disk free, agent pods pending.
3. Alerts: queue wait p95 > 5 min for 15 min, heap > 85%, disk < 15%, controller down 2 min.
4. Dashboards from the Prometheus plugin in Grafana.

**Why →** Users feel queue time and failures; those are the right SLIs.

**Interview tip →** Mention measuring "infrastructure failure rate" separate from test failures.

</details>

### Q84. A storage outage corrupted some `build.xml` files and jobs show errors on load. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Partial data corruption.

**What I'd do →**
1. Check logs for `Failed to load` with paths; Jenkins lists unreadable data under Manage Old Data.
2. For corrupted build directories, move them out of `jobs/*/builds/` (or restore from backup) and reload configuration from disk.
3. If `config.xml` files are corrupted, restore from backup or regenerate jobs via Job DSL.
4. Verify storage health and move `JENKINS_HOME` to reliable block storage.

**Why →** Corrupted build records can be dropped; configs come back from code.

**Interview tip →** Mention "Reload Configuration from Disk" and its effect.

</details>

### Q85. Can you run two Jenkins controller replicas behind a load balancer on a shared NFS volume for HA? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A common misconception.

**What I'd do →**
1. Explain no: each controller keeps state in memory and writes files independently; two writers corrupt data and both run the same scheduled builds.
2. Offer real options: single replica StatefulSet with fast rescheduling, warm standby with replication, multiple independent controllers, or CloudBees HA.
3. Also mention NFS latency hurts Jenkins performance; prefer block storage.

**Why →** Open-source Jenkins is single-writer by design.

**Interview tip →** Saying "no, and here is why" earns more credit than a creative but unsafe design.

</details>

### Q86. Builds slowed down after moving `JENKINS_HOME` to EFS/NFS. What happened and what do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Jenkins does many small file operations (build records, flow nodes, logs); network file systems add latency to each.

**What I'd do →**
1. Measure with `iostat`/EFS metrics; check burst credits on EFS.
2. Move to block storage (EBS gp3 with provisioned IOPS, or a fast CSI volume).
3. Use `PERFORMANCE_OPTIMIZED` durability to reduce writes.
4. Keep workspaces off the controller entirely.

**Why →** Block storage gives low-latency I/O that Jenkins' write pattern needs.

**Interview tip →** Mention EFS throughput modes if forced to stay on it.

</details>

### Q87. Developers complain Jenkins is "down" during upgrades every month. How do you reduce disruption? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Planned downtime is noticeable because of running builds and slow startup.

**What I'd do →**
1. Announce maintenance windows via system message (JCasC `systemMessage`).
2. Quiet-down ahead of time, then restart when idle.
3. Speed up startup: fewer plugins, fewer retained builds, faster disk.
4. Upgrade quarterly with the LTS cadence, not ad hoc.

**Why →** Predictable, short windows feel much better than surprise restarts.

**Interview tip →** Measure startup time and make it an explicit metric.

</details>

### Q88. A rogue job with `while true` Groovy loop is consuming controller CPU. Fix now and prevent. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** CPS code runs on the controller.

**What I'd do →**
1. Abort the build; if it ignores aborts, use `/term` then `/kill`.
2. Find it with a thread dump (`/threadDump`) showing the CPS VM thread for that job.
3. Prevent: pipeline-level `timeout` in the standard template, code review on Jenkinsfiles, keep logic in `sh`.

**Why →** Groovy loops do not touch agents; only timeouts and aborts stop them.

**Interview tip →** Explain CPS VM threads per build.

</details>

### Q89. You need to move 300 jobs from an old controller to a new Kubernetes-based one with minimal disruption. Plan it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Migration with parallel run.

**What I'd do →**
1. Build the new controller from code (image, JCasC, Job DSL); recreate jobs as multibranch where possible.
2. Credentials: recreate in the new store or external provider; do not copy `secrets/` unless you migrate `JENKINS_HOME` wholesale.
3. Migrate team by team: switch their webhooks and branch protection checks to the new controller, keep the old one read-only for history.
4. Decommission after a retention period, with history exported.

**Why →** Team-by-team cutover limits risk and lets you fix issues incrementally.

**Interview tip →** Mention status check names may change, requiring branch protection updates.

</details>

### Q90. The controller restart takes 25 minutes. How do you bring it down to under 5? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Startup loads every job and build record and initializes plugins.

**What I'd do →**
1. Measure with startup logs (`Started initialization`, `Loaded all jobs`, `Completed initialization`) to see which phase is slow.
2. Discard old builds and orphaned branch jobs; job loading is usually dominant.
3. Remove unused plugins; check for plugins doing heavy work on start.
4. Fast storage, enough heap, and avoid `init.groovy.d` scripts doing slow API calls.

**Why →** Startup time scales with items, builds and plugins.

**Interview tip →** Quote "Loaded all jobs" timing from logs as your diagnostic.

</details>

## Integrations & delivery

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q91. The SonarQube quality gate step waits until timeout on every build, even though analysis completes. Design the correct integration. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** SonarQube is not calling back Jenkins.

**What I'd do →**
1. Configure a SonarQube webhook to `https://jenkins.example.com/sonarqube-webhook/` (trailing slash), with a secret matching the Jenkins Sonar server config.
2. Ensure network connectivity from SonarQube to Jenkins.
3. Keep `waitForQualityGate` in the same build after `withSonarQubeEnv`, with `timeout(time: 10, unit: 'MINUTES')`.
4. Run the gate step outside the agent pod if possible to avoid holding resources.

**Why →** The step is event-driven; without the webhook it only times out.

**Interview tip →** Mention `report-task.txt` linking the analysis to the build.

</details>

### Q92. You need to publish Java artifacts: snapshots on every `main` build, releases on tags. Design with Nexus. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two repositories with different policies.

**What I'd do →**
1. `pom.xml` `distributionManagement` with `maven-snapshots` and `maven-releases` URLs.
2. Pipeline: on `main`, `mvn -B deploy` with `-SNAPSHOT` version; on tag `v1.4.0`, set version from `TAG_NAME` (`mvn versions:set -DnewVersion=1.4.0`) and deploy to releases.
3. Nexus: releases disallow redeploy, snapshots have cleanup policies.
4. Credentials via settings.xml from Config File Provider with credential substitution.

**Why →** Immutable releases plus disposable snapshots is the standard Maven model.

**Interview tip →** Mention that redeploying a release returning 400 is a feature.

</details>

### Q93. Developers want to deploy to Kubernetes from Jenkins, but security does not want cluster-admin kubeconfigs in Jenkins. What do you propose? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reduce or remove cluster credentials in CI.

**What I'd do →**
1. Preferred: GitOps. Jenkins builds, scans, pushes and then commits the new tag to a config repo (or opens a PR for prod); Argo CD in the cluster pulls and applies.
2. If push-based is required: per-namespace service accounts with `edit` RoleBindings only in target namespaces, used by deploy pod templates (in-cluster) or short-lived tokens (`kubectl create token deployer --duration=15m` issued by an automation).
3. No cluster-admin anywhere in Jenkins.

**Why →** GitOps keeps cluster credentials inside the cluster and adds drift correction.

**Interview tip →** Explain the Jenkins role becomes "CI and promotion", Argo CD does "CD".

</details>

### Q94. Trivy blocks a release because of a HIGH CVE in a base image package with no fix available. Business wants to ship. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Risk decision, not a pipeline bug.

**What I'd do →**
1. Check exploitability: is the package used at runtime, is it reachable?
2. If no fix exists, `--ignore-unfixed` already skips it; if it is flagged anyway, consider a different base (distroless, newer minor).
3. If accepting, add the CVE to `.trivyignore` with a justification and expiry comment, approved by security.
4. Track it and remove the exception when a fix ships.

**Why →** Gates should block by default and allow documented, time-boxed exceptions.

**Interview tip →** Mention VEX documents as the formal way to state non-exploitability.

</details>

### Q95. Image builds take 9 minutes in Kubernetes pods because nothing is cached. How do you add caching with Kaniko? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ephemeral pods lose the layer cache.

**What I'd do →**
1. Enable Kaniko's registry cache:
   ```bash
   /kaniko/executor --context dir://$WORKSPACE --dockerfile Dockerfile \
     --destination $IMAGE:$TAG --cache=true --cache-repo=$REGISTRY/cache/shack-app
   ```
2. Order the Dockerfile so dependency install layers come before source copy.
3. Use a base image mirror in the internal registry.
4. For BuildKit, use `--export-cache type=registry` and `--import-cache`.

**Why →** A registry-backed cache survives pod deletion.

**Interview tip →** Mention cache repos need their own retention policy.

</details>

### Q96. A pipeline should promote the exact image that passed staging to production, not a rebuild. How do you guarantee it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Immutable artifact promotion.

**What I'd do →**
1. Build once and record the digest: `crane digest $IMAGE:$TAG` and store it in `env.IMAGE_DIGEST` and the build description.
2. Deploy to staging and prod by digest (`image@sha256:...`).
3. Optionally copy/tag the image into a prod registry with `crane copy` without rebuilding.
4. Prod deploy job takes the digest as a parameter, verifies the signature.

**Why →** Digests are immutable; tags can be overwritten.

**Interview tip →** Say "same digest in every environment" explicitly.

</details>

### Q97. Your SonarQube analysis must decorate PRs with comments and fail PR builds on new issues only. How do you configure the pipeline? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** PR analysis needs PR parameters and an edition that supports it.

**What I'd do →**
1. Use SonarQube Developer Edition or above (or SonarQube Cloud); Community Build lacks branch/PR analysis natively.
2. Pass PR parameters on PR builds:
   ```bash
   sonar-scanner -Dsonar.pullrequest.key=$CHANGE_ID \
     -Dsonar.pullrequest.branch=$CHANGE_BRANCH -Dsonar.pullrequest.base=$CHANGE_TARGET
   ```
3. Configure the GitHub integration in SonarQube for decoration.
4. Quality gate on "new code" conditions; `waitForQualityGate abortPipeline: true`.

**Why →** New-code gates stop regressions without forcing teams to fix legacy debt first.

**Interview tip →** Many Sonar plugins auto-detect PR params in Jenkins multibranch; still know the properties.

</details>

### Q98. You need to run end-to-end tests against an ephemeral environment per PR and clean it up reliably. Design it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Preview environments tied to PR lifecycle.

**What I'd do →**
1. On PR build: create namespace `pr-${CHANGE_ID}`, deploy with Helm using the PR image tag, run E2E tests.
2. In `post { cleanup { } }`: optional teardown; better to keep the environment while the PR is open.
3. A scheduled cleanup job deletes namespaces whose PR is closed (query GitHub API) or older than N days; label namespaces with `pr`, `created-at`.
4. ResourceQuota per preview namespace.

**Why →** `post cleanup` is not enough (aborted builds, controller restarts), so a reaper job is essential.

**Interview tip →** Mention cost controls on preview environments.

</details>

### Q99. Developers want Jenkins to post the Trivy and Sonar summary on the PR. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Feedback in the PR improves response.

**What I'd do →**
1. Use the GitHub Checks plugin `publishChecks name: 'Trivy', title: 'Image scan', summary: readFile('trivy-summary.md'), conclusion: 'FAILURE'`.
2. Or `pullRequest.comment()` via the Pipeline GitHub plugin, or call the GitHub API with `curl` and the GitHub App token.
3. Keep the full report as an archived artifact and link it.

**Why →** Developers act on feedback they see without opening Jenkins.

**Interview tip →** Mention avoiding comment spam by updating one comment.

</details>

### Q100. A Helm-based deploy stage occasionally fails with a stuck release and blocks all later deploys. Design the stage to be robust. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Helm leaves the release in `pending-upgrade` if interrupted.

**What I'd do →**
1. Use `helm upgrade --install --atomic --timeout 5m --wait` so failed upgrades roll back automatically.
2. Use `lock(resource: "deploy-${ENV}")` so two builds never upgrade the same release concurrently.
3. Put a stage timeout slightly longer than Helm's timeout so Jenkins does not kill Helm mid-operation.
4. Document recovery: `helm history`, `helm rollback <release> <rev>`.

**Why →** Atomic upgrades plus locks remove the two common causes of stuck releases.

**Interview tip →** Mention `--atomic` (renamed `--rollback-on-failure` in Helm 4) so you sound current.

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
