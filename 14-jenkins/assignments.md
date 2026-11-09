# Jenkins — Assignments

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

> 25 practical assignments: 10 🟢 Beginner, 10 🟡 Intermediate, 5 🔴 Advanced. Do them on a real Jenkins (Docker or a kind cluster is enough); reading is not enough.

[⬅ Back to Jenkins](./README.md) · [🏠 Home](../README.md)

## Index

| # | Level | Assignment |
|---|-------|------------|
| 1 | 🟢 | Run Jenkins in Docker and map `JENKINS_HOME` |
| 2 | 🟢 | Your first declarative pipeline from SCM |
| 3 | 🟢 | Parameters, environment and `when` |
| 4 | 🟢 | Post conditions and build results |
| 5 | 🟢 | Credentials binding and masking |
| 6 | 🟢 | Add an inbound agent and empty the controller |
| 7 | 🟢 | JUnit reports and archived artifacts |
| 8 | 🟢 | Cron triggers, `H`, and the REST API |
| 9 | 🟢 | Stash, unstash and multiple agents |
| 10 | 🟢 | Jenkins CLI and the declarative linter |
| 11 | 🟡 | Parallel stages and fail-fast |
| 12 | 🟡 | Matrix builds with exclusions |
| 13 | 🟡 | Approval gates with `input` and no held agent |
| 14 | 🟡 | Multibranch pipeline with PR discovery and webhooks |
| 15 | 🟡 | Your first shared library |
| 16 | 🟡 | Controller image with pinned plugins |
| 17 | 🟡 | Configure a controller entirely with JCasC |
| 18 | 🟡 | Kubernetes agents with multi-container pods |
| 19 | 🟡 | SonarQube quality gate |
| 20 | 🟡 | Folder-scoped credentials and matrix authorization |
| 21 | 🔴 | Seed every job with Job DSL from JCasC |
| 22 | 🔴 | Unit test a shared library with JenkinsPipelineUnit |
| 23 | 🔴 | CPS deep dive: break it and fix it |
| 24 | 🔴 | Backup, disaster recovery and an LTS upgrade drill |
| 25 | 🔴 | Secure supply-chain pipeline: build, scan, sign, deploy by digest |

---

## 🟢 Beginner

### Assignment 1. Run Jenkins in Docker and map `JENKINS_HOME` 🟢

**Objective:** Get a controller running and understand where its state lives.

**Tasks**
1. Create a named volume and run `jenkins/jenkins:lts-jdk21` with ports 8080 and the volume mounted at `/var/jenkins_home`.
2. Read the initial admin password from the container and finish the setup wizard with the suggested plugins.
3. List the top level of `/var/jenkins_home` and write one line describing each of: `config.xml`, `jobs/`, `plugins/`, `secrets/`, `users/`, `nodes/`.
4. Create a Freestyle job that runs `echo hello`, build it, and find its `log` file on disk.
5. Delete and recreate the container with the same volume; confirm the job and its build survive.

**Expected outcome / acceptance criteria**
- Jenkins reachable at `http://localhost:8080` after the container is recreated, with the job and build history intact.
- A short note mapping each directory to its purpose, including why `secrets/` is the most sensitive.

<details>
<summary>Hints</summary>

`docker volume create jenkins_home`, `docker run -d --name jenkins -p 8080:8080 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts-jdk21`, `docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword`. Build logs are in `jobs/<job>/builds/<n>/log`.

</details>

**Bonus:** Find the job's `config.xml` and change the `echo` text directly in the file, then use **Manage Jenkins > Reload Configuration from Disk** and run it again.

### Assignment 2. Your first declarative pipeline from SCM 🟢

**Objective:** Move from UI-configured jobs to a Jenkinsfile in Git.

**Tasks**
1. Create a GitHub repo with a small app (any language) and a `Jenkinsfile` with three stages: Checkout info, Build, Test.
2. In the Checkout info stage, print `GIT_COMMIT`, `GIT_BRANCH`, `BUILD_NUMBER` and `WORKSPACE`.
3. Create a **Pipeline** job using "Pipeline script from SCM" pointing at the repo.
4. Break the Test stage on purpose (`exit 1`), observe the result, then fix it.
5. Open the Pipeline Syntax page and generate a `sh` step with `returnStdout`.

**Expected outcome / acceptance criteria**
- A green build with three stages visible in the stage view or Pipeline Graph View.
- A red build showing exactly which stage failed and the exit code.

<details>
<summary>Hints</summary>

```groovy
pipeline {
  agent any
  stages {
    stage('Checkout info') { steps { sh 'echo "$GIT_COMMIT $GIT_BRANCH $BUILD_NUMBER $WORKSPACE"' } }
    stage('Build') { steps { sh 'echo building' } }
    stage('Test') { steps { sh 'echo testing' } }
  }
}
```

</details>

**Bonus:** Add `options { timestamps(); timeout(time: 10, unit: 'MINUTES') }` and `buildDiscarder(logRotator(numToKeepStr: '10'))`.

### Assignment 3. Parameters, environment and `when` 🟢

**Objective:** Make a pipeline configurable and conditional.

**Tasks**
1. Add parameters: `choice` `ENV` (dev, staging, prod), `booleanParam` `RUN_SMOKE`, `string` `VERSION`.
2. Add a pipeline-level `environment` with `APP_NAME` and a stage-level `environment` that overrides it.
3. Add a stage that runs only when `params.RUN_SMOKE` is true and one that runs only when `ENV == 'prod'`.
4. Run the job with three different parameter combinations.
5. Print `params.RUN_SMOKE` and `env.RUN_SMOKE` and explain the type difference.

**Expected outcome / acceptance criteria**
- Skipped stages are shown as skipped, not failed.
- You can explain why `if (env.RUN_SMOKE)` is truthy even when the value is `'false'`.

<details>
<summary>Hints</summary>

`when { expression { params.RUN_SMOKE } }`, `when { environment name: 'ENV', value: 'prod' }`. The first run after adding `parameters` registers them; the "Build with Parameters" button appears afterwards.

</details>

**Bonus:** Add `when { beforeAgent true; ... }` to a stage with its own agent and confirm no agent is allocated when it is skipped.

### Assignment 4. Post conditions and build results 🟢

**Objective:** Control what happens after success, failure and everything in between.

**Tasks**
1. Add a pipeline-level `post` with `always`, `success`, `failure`, `unstable`, `fixed`, `regression` and `cleanup`, each printing its name.
2. Produce each result: SUCCESS, FAILURE (`error()`), UNSTABLE (`unstable()`), ABORTED (click Abort during a `sleep 60`).
3. Run FAILURE then SUCCESS to trigger `fixed`, and SUCCESS then FAILURE to trigger `regression`.
4. Use `catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE')` on one stage and observe the colours.
5. Record the order in which `post` conditions fired.

**Expected outcome / acceptance criteria**
- A table of the five results with the post blocks that fired for each.
- `cleanup` fires last every time.

<details>
<summary>Hints</summary>

Order is `always`, `changed`, `fixed`, `regression`, `aborted`, `failure`, `success`, `unstable`, `unsuccessful`, `cleanup`.

</details>

**Bonus:** Add a stage-level `post` and show it runs before the pipeline-level one.

### Assignment 5. Credentials binding and masking 🟢

**Objective:** Use secrets without leaking them.

**Tasks**
1. Create credentials: secret text `demo-token`, username/password `demo-user`, secret file `demo-file`.
2. Bind `demo-user` with `environment { CREDS = credentials('demo-user') }` and print `CREDS_USR` and the length of `CREDS_PSW`.
3. Bind all three with `withCredentials` in one stage.
4. Write one `sh` step with double quotes interpolating the secret and read the warning Jenkins prints; rewrite it with single quotes.
5. Echo the token base64-encoded and observe that masking does not catch it. Then delete that build.

**Expected outcome / acceptance criteria**
- The plain secret is shown as `****` in the log.
- You can explain why masking is a convenience, not a security control.

<details>
<summary>Hints</summary>

```groovy
withCredentials([string(credentialsId: 'demo-token', variable: 'TOKEN'),
                 file(credentialsId: 'demo-file', variable: 'CFG')]) {
  sh 'echo "token length: ${#TOKEN}"; wc -c "$CFG"'
}
```

</details>

**Bonus:** Move the credentials into a folder and prove a job outside the folder gets `Could not find credentials entry with ID`.

### Assignment 6. Add an inbound agent and empty the controller 🟢

**Objective:** Run builds on an agent, never on the controller.

**Tasks**
1. Create a permanent node `docker-agent-1`, label `linux docker`, launch method "Launch agent by connecting it to the controller", remote root `/home/jenkins/agent`.
2. Start `jenkins/inbound-agent:latest-jdk21` with `-webSocket`, the URL, secret and name.
3. Set the built-in node to 0 executors.
4. Change your pipeline to `agent { label 'linux' }` and confirm `NODE_NAME`.
5. Stop the agent container while a build runs and read the error.

**Expected outcome / acceptance criteria**
- Builds run on `docker-agent-1`; the built-in node shows 0 executors.
- You saved the error text from the killed agent and can explain it.

<details>
<summary>Hints</summary>

`docker run -d --name agent1 --network <jenkins-network> jenkins/inbound-agent:latest-jdk21 -url http://jenkins:8080/ -webSocket -secret <secret> -name docker-agent-1 -workDir /home/jenkins/agent`. Put both containers on the same user-defined Docker network so `jenkins` resolves.

</details>

**Bonus:** Set the node to `EXCLUSIVE` mode and show that a job with `agent any` no longer runs there.

### Assignment 7. JUnit reports and archived artifacts 🟢

**Objective:** Publish test results and keep build outputs.

**Tasks**
1. Use a Node or Maven project that produces JUnit XML (`jest-junit` or Surefire).
2. Publish results in `post { always { junit ... } }`.
3. Make one test fail and observe UNSTABLE vs FAILURE depending on whether the test command fails the step.
4. Archive the build output with `archiveArtifacts artifacts: 'dist/**', fingerprint: true`.
5. Configure `buildDiscarder` to keep artifacts for only the last 3 builds.

**Expected outcome / acceptance criteria**
- A test result trend graph after 3+ builds.
- Artifacts downloadable from the build page and removed for older builds.

<details>
<summary>Hints</summary>

`buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '3'))`. If `npm test` fails the step, the build is FAILURE; to get UNSTABLE, run tests with `returnStatus: true` and let `junit` set the result.

</details>

**Bonus:** Add `allowEmptyResults: false` and show the build fails when the report path is wrong.

### Assignment 8. Cron triggers, `H`, and the REST API 🟢

**Objective:** Schedule builds and control Jenkins from scripts.

**Tasks**
1. Add `triggers { cron('H/5 * * * *') }` and observe the actual minutes it runs.
2. Generate an API token for your user.
3. Trigger a parameterized build with `curl` and capture the queue item URL from the `Location` header.
4. Poll the queue item until it has a build number, then fetch `consoleText` of that build.
5. Fetch the last 5 build results of the job with a `tree` query.

**Expected outcome / acceptance criteria**
- A shell script that triggers a build and prints its final result.
- You can explain why `H` spreads load.

<details>
<summary>Hints</summary>

`curl -s -X POST -u user:$TOKEN "$URL/job/app/buildWithParameters?ENV=dev" -D - | grep -i location`, then `$QUEUE/api/json?tree=executable[number,url]`, then `$URL/job/app/<n>/api/json?tree=result,building`.

</details>

**Bonus:** Use `/job/app/api/json?tree=builds[number,result,duration]{0,5}` to get only the latest 5.

### Assignment 9. Stash, unstash and multiple agents 🟢

**Objective:** Move small files between stages on different agents.

**Tasks**
1. Use `agent none` and two stages on different agents (two labels, or two Docker agent images).
2. Stage 1 builds a file (`dist/app.txt` with the build number) and stashes it.
3. Stage 2 unstashes it and prints it.
4. Try to read the file in stage 2 without `unstash` and record the error.
5. Explain where stashes are stored and why large stashes are a problem.

**Expected outcome / acceptance criteria**
- Stage 2 prints the content created by stage 1 on another agent.

<details>
<summary>Hints</summary>

`stash name: 'dist', includes: 'dist/**'` and `unstash 'dist'`. Stashes are stored on the controller and deleted at the end of the build unless `preserveStashes` is set.

</details>

**Bonus:** Replace the two agents with nested stages under one parent agent and remove the stash entirely.

### Assignment 10. Jenkins CLI and the declarative linter 🟢

**Objective:** Administer Jenkins and validate pipelines from the terminal.

**Tasks**
1. Download `jenkins-cli.jar` from your controller.
2. Run `who-am-i`, `list-jobs`, `list-plugins` with `-auth` and `-webSocket`.
3. Validate a Jenkinsfile with `declarative-linter`, then introduce `step {}` instead of `steps {}` and validate again.
4. Validate the same file via `curl` to `/pipeline-model-converter/validate`.
5. Trigger a build with `build -f -v` and watch the log stream.

**Expected outcome / acceptance criteria**
- A pre-commit hook or Makefile target that lints the Jenkinsfile before pushing.

<details>
<summary>Hints</summary>

`java -jar jenkins-cli.jar -s http://localhost:8080/ -auth user:$TOKEN -webSocket declarative-linter < Jenkinsfile`.

</details>

**Bonus:** Run `safe-restart` from the CLI while a build is running and observe that Jenkins waits.

---

## 🟡 Intermediate

### Assignment 11. Parallel stages and fail-fast 🟡

**Objective:** Cut pipeline wall time with parallelism.

**Tasks**
1. Create four check stages that sleep 30, 45, 60 and 20 seconds; run them sequentially and record the total time.
2. Put them under one `parallel` block and record the time again.
3. Make the 20-second one fail; compare behaviour with and without `failFast true`.
4. Give one parallel branch its own agent and explain the workspace implications.
5. Generate a dynamic `parallel` map in `script {}` from a list of service names.

**Expected outcome / acceptance criteria**
- Wall time drops to roughly the slowest branch.
- With `failFast`, the remaining branches show as aborted.

<details>
<summary>Hints</summary>

In loops, copy the loop variable before capturing it in the closure: `def svc = s; branches[svc] = { ... }`.

</details>

**Bonus:** Use `options { parallelsAlwaysFailFast() }` at pipeline level instead of per stage.

### Assignment 12. Matrix builds with exclusions 🟡

**Objective:** Test across versions without copy-pasting stages.

**Tasks**
1. Create a `matrix` with axes `NODE_VERSION` (18, 20, 22) and `OS` (linux, alpine) using container images per combination.
2. Exclude `18` on `alpine`.
3. Publish JUnit results per cell with a unique report name.
4. Make one cell fail and confirm the others still finish (no fail-fast).
5. Count cells and explain how the matrix grows.

**Expected outcome / acceptance criteria**
- Five cells, named by axis values in the UI.

<details>
<summary>Hints</summary>

In the cell, choose an image with `agent { docker { image "node:${NODE_VERSION}${OS == 'alpine' ? '-alpine' : ''}" } }` on a Docker agent, or a Kubernetes pod template with the image interpolated.

</details>

**Bonus:** Add a third axis and observe when the pipeline becomes unwieldy; explain the `Method code too large` risk.

### Assignment 13. Approval gates with `input` and no held agent 🟡

**Objective:** Implement production approvals the right way.

**Tasks**
1. Write a pipeline with `agent none`, a Build stage with an agent, an Approve stage using the `input` directive, and a Deploy stage with an agent.
2. Restrict `submitter` to a user or group and capture `submitterParameter`.
3. Add a stage-level `timeout` of 15 minutes on the approval.
4. While the build waits, confirm no executor is busy (or no agent pod exists on Kubernetes).
5. Repeat with the `input` *step* inside the Build stage's agent and compare executor usage.

**Expected outcome / acceptance criteria**
- With the directive, 0 executors are used during the wait; with the step, 1 is.
- An unauthorised user cannot approve.

<details>
<summary>Hints</summary>

`input { message 'Deploy?'; ok 'Ship it'; submitter 'release-managers'; submitterParameter 'APPROVER' }`. Check **Build Executor Status** while waiting.

</details>

**Bonus:** Add `milestone()` before and after the approval and show that approving an older build is impossible once a newer one passed.

### Assignment 14. Multibranch pipeline with PR discovery and webhooks 🟡

**Objective:** Build every branch and PR automatically.

**Tasks**
1. Create a Multibranch Pipeline with the GitHub branch source.
2. Configure discovery: branches excluding PR branches, PRs merged with target.
3. Configure a webhook (ngrok or a public Jenkins) to `/github-webhook/`.
4. Open a PR and confirm a `PR-<n>` job builds within seconds; check the commit status on GitHub.
5. Add `when { branch 'main' }` to a deploy stage and confirm PR builds skip it.
6. Delete the branch and observe the orphaned item strategy.

**Expected outcome / acceptance criteria**
- Webhook delivery shows HTTP 200; build cause is a push or PR event, not indexing.
- PR status check visible on GitHub.

<details>
<summary>Hints</summary>

Trailing slash on `/github-webhook/` matters. Use **Scan Repository Log** and **Multibranch Pipeline Events** to debug.

</details>

**Bonus:** Add tag discovery with a build strategy for tags newer than 1 day and release on `v*` tags.

### Assignment 15. Your first shared library 🟡

**Objective:** Extract reusable pipeline logic into a versioned library.

**Tasks**
1. Create a repo with `vars/greet.groovy` (a `call(Map)` step), `src/org/demo/Semver.groovy` (a class that bumps versions) and `resources/hello.sh`.
2. Configure it as a global library named `demo-lib` with default version `v1`.
3. Use all three from a Jenkinsfile: the step, the class via `import`, and the script via `libraryResource`.
4. Add `vars/greet.txt` and find it in the Global Variables Reference.
5. Release `v1.1.0`, move `v1`, and confirm pipelines pick up the change without edits.

**Expected outcome / acceptance criteria**
- Build log shows `Loading library demo-lib@v1`.
- Rollback by moving the tag back works.

<details>
<summary>Hints</summary>

`@Library('demo-lib@v1') import org.demo.Semver` then `greet(name: 'Shack')`, `Semver.bumpMinor('1.4.2')`, `writeFile file: 'hello.sh', text: libraryResource('hello.sh')`.

</details>

**Bonus:** Configure the same repo as a folder-level library and show that a method needing approval is blocked there but works in the global (trusted) one.

### Assignment 16. Controller image with pinned plugins 🟡

**Objective:** Make plugin installation reproducible.

**Tasks**
1. Export your installed plugins as `name:version` into `plugins.txt` using the REST API and `jq`.
2. Write a Dockerfile `FROM jenkins/jenkins:<pinned-lts>-jdk21` that installs them with `jenkins-plugin-cli`.
3. Build the image and start it with an empty volume; confirm the same plugin set.
4. Run `jenkins-plugin-cli --available-updates --output txt` to produce an update diff.
5. Remove one plugin that others depend on and read the error.

**Expected outcome / acceptance criteria**
- Two controllers built from the same image have identical plugin lists.
- A written plugin update procedure (staging, smoke test, rollback image).

<details>
<summary>Hints</summary>

`curl -s -u admin:$T "$URL/pluginManager/api/json?depth=1&tree=plugins[shortName,version]" | jq -r '.plugins[] | "\(.shortName):\(.version)"' | sort > plugins.txt`.

</details>

**Bonus:** Add `--view-security-warnings` to the build and fail the image build if any warnings are reported.

### Assignment 17. Configure a controller entirely with JCasC 🟡

**Objective:** Treat Jenkins configuration as code.

**Tasks**
1. Add `configuration-as-code` to your image and set `CASC_JENKINS_CONFIG`.
2. Write `jenkins.yaml` with: system message, 0 executors, local security realm with two users, matrix authorization, one secret text credential from an env var, a global library, and `unclassified.location.url`.
3. Start with `-Djenkins.install.runSetupWizard=false` and verify every setting.
4. Change a setting in the UI, reload JCasC, and observe the revert.
5. Introduce a typo in an attribute name and capture the `ConfiguratorException`.

**Expected outcome / acceptance criteria**
- A fresh container comes up fully configured with no manual steps.
- No secret values exist in Git.

<details>
<summary>Hints</summary>

Use **Manage Jenkins > Configuration as Code > View Configuration** on a hand-configured instance as a starting point. Secrets via `${VAR}` resolve from environment variables or `/run/secrets/VAR`.

</details>

**Bonus:** Add a CI job that boots the image with the YAML and fails if the log contains `ConfiguratorException`.

### Assignment 18. Kubernetes agents with multi-container pods 🟡

**Objective:** Run builds in ephemeral pods.

**Tasks**
1. Install Jenkins on kind (Helm chart or manifests) and configure the kubernetes cloud with WebSocket.
2. Create an agent namespace with RBAC for the controller's service account.
3. Write a pod template with `maven` and `node` containers (`sleep infinity`) and run steps in each with `container()`.
4. Show that a file written in one container is visible in the other.
5. Set `podRetention` to `onFailure`, fail a build, inspect the retained pod, then revert.

**Expected outcome / acceptance criteria**
- A new pod per build, deleted after success.
- `kubectl -n <agents-ns> get pods -w` shows the lifecycle.

<details>
<summary>Hints</summary>

`agent { kubernetes { yamlFile 'ci/pod.yaml'; defaultContainer 'maven' } }`. If the build hangs at `Waiting for agent to connect`, check the `jnlp` container logs and the `jenkinsUrl`.

</details>

**Bonus:** Add a `ResourceQuota` to the agent namespace and observe what happens when builds exceed it.

### Assignment 19. SonarQube quality gate 🟡

**Objective:** Block merges on code quality.

**Tasks**
1. Run SonarQube (container or in cluster) and generate an analysis token.
2. Configure the SonarQube server in Jenkins (UI or JCasC) with the token credential.
3. Add `withSonarQubeEnv` analysis and a `waitForQualityGate abortPipeline: true` stage with a timeout.
4. Configure the SonarQube webhook to `/sonarqube-webhook/`.
5. Create a custom gate that fails, run the pipeline, then restore the default gate.

**Expected outcome / acceptance criteria**
- The quality gate stage passes quickly (seconds after analysis) with the webhook, and times out without it.
- A failing gate aborts the build.

<details>
<summary>Hints</summary>

Without the webhook, `waitForQualityGate` waits until the timeout. Coverage must be generated before analysis (`lcov.info` for JS, JaCoCo XML for Java).

</details>

**Bonus:** Run the gate step outside any agent and explain the saving.

### Assignment 20. Folder-scoped credentials and matrix authorization 🟡

**Objective:** Make one controller safe for two teams.

**Tasks**
1. Create folders `team-a` and `team-b`, each with a job and a credential named `deploy-key`.
2. Use project-based matrix authorization: `team-a-devs` can build and configure in `team-a` only; same for team B.
3. Prove a team A job cannot bind team B's credential, and a team A user cannot see team B's folder.
4. Remove `Run/Replay` from developers and explain why.
5. Check effective permissions with `/whoAmI/` for a test user.

**Expected outcome / acceptance criteria**
- A permission matrix document and screenshots or logs of the denied attempts.

<details>
<summary>Hints</summary>

Use the Matrix Authorization Strategy plugin's "Project-based Matrix Authorization Strategy", then enable "Enable project-based security" on each folder.

</details>

**Bonus:** Express the whole setup in JCasC and Job DSL (`authorization { permission(...) }` on folders).

---

## 🔴 Advanced

### Assignment 21. Seed every job with Job DSL from JCasC 🔴

**Objective:** Make the controller fully disposable.

**Tasks**
1. Create a `teams.yaml` listing three teams with their GitHub repos.
2. Write a Job DSL script that creates a folder per team and a multibranch pipeline per repo with GitHub branch source, PR discovery and an orphaned item strategy.
3. Load the script from JCasC `jobs:`.
4. Destroy the controller volume, redeploy, and confirm all jobs return and re-index on their own.
5. Remove a team from the YAML, reload, and handle the removed jobs deliberately.

**Expected outcome / acceptance criteria**
- Recreating the controller from scratch restores every folder and job in under 10 minutes.
- No job is created by hand.

<details>
<summary>Hints</summary>

Use your controller's Job DSL API viewer (`/plugin/job-dsl/api-viewer/index.html`) for the exact dynamic DSL of `github` branch sources. Variables like `${GITHUB_ORG}` are substituted by JCasC before the DSL runs.

</details>

**Bonus:** Run the same Job DSL in a seed job triggered by changes to the platform repo, with `removedJobAction('DELETE')`.

### Assignment 22. Unit test a shared library with JenkinsPipelineUnit 🔴

**Objective:** Treat the library as production code with tests and CI.

**Tasks**
1. Add a Gradle build to your library with JenkinsPipelineUnit and JUnit 5.
2. Write tests for a `vars/` step: mock `sh`, `withCredentials` and `container`, assert the exact command and that a missing argument calls `error`.
3. Write tests for a `src/` class without mocks.
4. Add a Jenkinsfile to the library repo that runs the tests on every PR.
5. Add a staging integration test: a real pipeline on a test folder that loads the PR branch of the library.

**Expected outcome / acceptance criteria**
- `./gradlew test` passes locally and in Jenkins; a broken step fails the PR.
- The release procedure (tag, move `v1`) is documented in the library README.

<details>
<summary>Hints</summary>

Extend `BasePipelineTest`, call `helper.registerAllowedMethod('sh', [Map]) { m -> commands << m.script }`, load with `loadScript('vars/trivyScan.groovy')`. Print `printCallStack()` while writing tests.

</details>

**Bonus:** Generate the `vars/*.txt` docs from code comments during the library build.

### Assignment 23. CPS deep dive: break it and fix it 🔴

**Objective:** Understand how pipeline Groovy really executes.

**Tasks**
1. Write a pipeline that triggers `NotSerializableException` with a regex `Matcher` held across an `echo`; fix it with `@NonCPS`.
2. Trigger the same exception with `JsonSlurper` and fix it with `readJSON`.
3. Write a `@NonCPS` method that calls `echo` and document the actual behaviour.
4. Run a Groovy loop that builds a 50,000-element list and observe controller CPU; reimplement the work in `sh` with `jq` or `awk` and compare.
5. Restart the controller mid-build (with a long `sh` step running) and observe resume behaviour with `MAX_SURVIVABILITY` vs `PERFORMANCE_OPTIMIZED`.

**Expected outcome / acceptance criteria**
- A write-up explaining CPS, serialization, `@NonCPS` rules and durability with your own logs as evidence.

<details>
<summary>Hints</summary>

`options { durabilityHint('PERFORMANCE_OPTIMIZED') }`. Use `/threadDump` during the heavy Groovy loop to find the CPS VM thread for the build.

</details>

**Bonus:** Trigger `Method code too large!` with a generated Jenkinsfile and fix it with library extraction.

### Assignment 24. Backup, disaster recovery and an LTS upgrade drill 🔴

**Objective:** Prove you can recover and upgrade Jenkins without drama.

**Tasks**
1. Write a backup script (or Kubernetes CronJob) that archives `JENKINS_HOME` excluding `workspace/`, `caches/` and exploded plugins, and uploads it to object storage.
2. Destroy the controller and its volume; restore into a new instance and verify credentials decrypt and build history is present. Time it.
3. Repeat the restore without `secrets/` and record what breaks.
4. Upgrade the restored instance to the next LTS: update the base image, regenerate plugin versions with `--jenkins-version`, run smoke pipelines.
5. Roll back to the previous image and backup, and document what is and is not reversible.

**Expected outcome / acceptance criteria**
- A runbook with measured RTO and RPO, tested end to end.

<details>
<summary>Hints</summary>

`tar --exclude='./workspace' --exclude='./caches' --exclude='./plugins/*/' -czf jenkins-home.tgz -C /var/jenkins_home .`. Credentials failing to decrypt after restore means `secrets/master.key` or `secrets/hudson.util.Secret` is missing.

</details>

**Bonus:** Use Velero with CSI snapshots for the same drill and compare speed.

### Assignment 25. Secure supply-chain pipeline: build, scan, sign, deploy by digest 🔴

**Objective:** Combine everything into a pipeline you would defend in a security review.

**Tasks**
1. Multibranch pipeline on Kubernetes agents, loading your shared library by tag.
2. Build the image daemonless (Kaniko or BuildKit) into a tarball; scan with Trivy (fail on fixable HIGH/CRITICAL) and generate an SBOM (`trivy image --format cyclonedx`).
3. Push only from `main`; capture the digest; sign it with cosign using a key from a KMS or Vault, and attach the SBOM as an attestation.
4. Deploy to dev by digest with a dedicated service account; require a group-restricted `input` for prod; enforce signatures in prod with Kyverno `verifyImages`.
5. Show that an unsigned image applied manually to prod is rejected by the cluster.

**Expected outcome / acceptance criteria**
- Every prod pod runs a signed image digest built and scanned by the pipeline.
- Bypassing Jenkins does not bypass the policy.

<details>
<summary>Hints</summary>

`cosign sign --key awskms:///alias/cosign <repo>@<digest>`, `cosign attest --predicate sbom.json --type cyclonedx --key ... <repo>@<digest>`. Run cosign in its own container in the pod template.

</details>

**Bonus:** Replace the `kubectl` prod deploy with a commit to a GitOps repo watched by Argo CD, so Jenkins holds no prod cluster credentials.

---

[⬅ Back to Jenkins](./README.md) · [🏠 Home](../README.md)

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
