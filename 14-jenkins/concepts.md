# Jenkins — Core Concepts

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

> Architecture, Jenkinsfile syntax, CPS, shared libraries, multibranch, plugins, JCasC, Kubernetes agents, security, operations, integrations and rapid-fire commands · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Jenkins](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Architecture, executors & JENKINS_HOME](#architecture-executors--jenkins_home) — Q1–Q25
- [Declarative pipeline syntax](#declarative-pipeline-syntax) — Q26–Q60
- [Scripted pipeline, Groovy & CPS](#scripted-pipeline-groovy--cps) — Q61–Q80
- [Shared libraries](#shared-libraries) — Q81–Q100
- [Multibranch, organization folders & webhooks](#multibranch-organization-folders--webhooks) — Q101–Q120
- [Plugins, JCasC & Jenkins on Kubernetes](#plugins-jcasc--jenkins-on-kubernetes) — Q121–Q145
- [Credentials & security](#credentials--security) — Q146–Q165
- [Backups, HA, upgrades & performance](#backups-ha-upgrades--performance) — Q166–Q180
- [Integrations: SonarQube, Nexus, Artifactory, Docker, Trivy](#integrations-sonarqube-nexus-artifactory-docker-trivy) — Q181–Q190
- [Best practices & rapid-fire commands](#best-practices--rapid-fire-commands) — Q191–Q200

---

## Architecture, executors & JENKINS_HOME

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. What are the main components of a Jenkins installation? 🟢

<details>
<summary><b>Answer</b></summary>

- **Controller**: a Java web app (Jetty inside `jenkins.war`) that serves the UI and REST API, stores configuration and build records, holds credentials, runs the build queue and executes pipeline Groovy (CPS).
- **Agents**: machines, VMs, containers or pods that run the actual build steps.
- **Executors**: slots on a node; one executor runs one build or one `node {}` block.
- **Plugins**: nearly every feature, including Pipeline, Git and credentials.
- **`JENKINS_HOME`**: the directory where all state lives as files.

</details>

### Q2. What is the difference between a node, an agent and an executor? 🟢

<details>
<summary><b>Answer</b></summary>

A **node** is any machine Jenkins can run builds on, including the controller itself (the "built-in node"). An **agent** is a node other than the controller. An **executor** is a concurrency slot on a node: an agent with 4 executors can run 4 builds at the same time. In pipeline terms, each `node('linux') { }` or declarative `agent` block occupies one executor for its duration.

</details>

### Q3. Why should the built-in node have 0 executors? 🟢

<details>
<summary><b>Answer</b></summary>

Anything that runs on the built-in node runs as the Jenkins process user with read access to `JENKINS_HOME`, including `secrets/master.key`, `credentials.xml` and every job's config. A malicious or careless `sh 'cat $JENKINS_HOME/...'` can exfiltrate all credentials. It also competes with the controller for CPU, memory and disk. Set it in JCasC:

```yaml
jenkins:
  numExecutors: 0
  mode: EXCLUSIVE
```

</details>

### Q4. What is `JENKINS_HOME` and what are its most important contents? 🟢

<details>
<summary><b>Answer</b></summary>

It is the single directory holding all Jenkins state (default `/var/jenkins_home` in the official image, `/var/lib/jenkins` for packages).

| Path | Contents |
|------|----------|
| `config.xml` | Global configuration |
| `jobs/<job>/config.xml` | Job definition |
| `jobs/<job>/builds/<n>/` | Build record, `log`, `build.xml`, test results |
| `plugins/` | `.jpi` files and exploded plugins |
| `nodes/` | Agent definitions |
| `users/` | User records and API tokens |
| `credentials.xml` | Global credentials (encrypted) |
| `secrets/` | `master.key`, `hudson.util.Secret`, `initialAdminPassword` |
| `workspace/` | Workspaces of builds that ran on the built-in node |

</details>

### Q5. What happens if you lose the `secrets/` directory but keep `credentials.xml`? 🔴

<details>
<summary><b>Answer</b></summary>

All stored secrets become unreadable. Credentials are encrypted with `hudson.util.Secret`, which is itself encrypted with `secrets/master.key`. Without those two files Jenkins cannot decrypt anything: credentials appear empty, API tokens break, and secret fields in job configs fail. Always back up `secrets/` together with the rest of `JENKINS_HOME`, but store the backup with stricter access because `master.key` plus `credentials.xml` is everything an attacker needs.

</details>

### Q6. How does an agent communicate with the controller? 🟡

<details>
<summary><b>Answer</b></summary>

Through the **remoting** library, a bidirectional channel that serializes commands and streams. Connection modes:

- **SSH launcher**: controller connects out to the agent over SSH and starts `agent.jar`.
- **Inbound (formerly JNLP)**: the agent starts `agent.jar` and connects to the controller, either over **WebSocket** on the HTTP port (recommended, works through ingress and proxies) or over the TCP agent port (default 50000).
- **Cloud plugins** (kubernetes, EC2) provision a machine or pod which then connects inbound.

```bash
java -jar agent.jar -url https://jenkins.example.com/ \
  -secret @secret-file -name linux-01 -webSocket -workDir /home/jenkins/agent
```

</details>

### Q7. What is the build queue and what can keep an item waiting in it? 🟡

<details>
<summary><b>Answer</b></summary>

The queue holds tasks waiting to start. An item waits because of: the **quiet period** (default 5 s), no online node matches the label, all matching executors are busy, a cloud is still provisioning a node, `disableConcurrentBuilds()` with a running build, a `lock` resource held elsewhere, or a blocked upstream/downstream relationship. Hover the item in the queue or read `/queue/api/json` to see the `why` field, for example `Waiting for next available executor on ‘linux’`.

</details>

### Q8. What are labels and label expressions? 🟢

<details>
<summary><b>Answer</b></summary>

Labels are tags on nodes describing capabilities: OS, tools, size. Jobs request a label expression and the scheduler picks any online node with a free executor that matches.

```groovy
agent { label 'linux && docker && !arm64' }
```

Operators: `&&`, `||`, `!`, `->` (implies), `<->`, and parentheses. Use capability labels (`docker`, `jdk21`) rather than hostnames, so you can replace machines without editing pipelines.

</details>

### Q9. What do the node usage modes "Use this node as much as possible" and "Only build jobs with label expressions matching this node" mean? 🟡

<details>
<summary><b>Answer</b></summary>

`NORMAL` mode lets any job without a label (or with a matching one) run on the node. `EXCLUSIVE` mode only accepts builds whose label expression explicitly matches the node. Use `EXCLUSIVE` for expensive or special nodes (GPU, macOS signing, prod-deploy agents) so random jobs do not land there, and for the built-in node.

</details>

### Q10. What is a workspace and where is it stored? 🟢

<details>
<summary><b>Answer</b></summary>

The workspace is the directory on the agent where the build checks out code and runs commands: `<agent workDir>/workspace/<job-path>`. It persists between builds on static agents (which speeds incremental builds but leaks state), and disappears with the pod on Kubernetes agents. Concurrent builds of the same job get suffixed workspaces like `myjob@2`. Temporary script files live in `myjob@tmp`.

</details>

### Q11. What is the difference between permanent agents and cloud agents? 🟡

<details>
<summary><b>Answer</b></summary>

**Permanent (static) agents** are defined once and stay connected: good for special hardware or licensed tools, but they drift, accumulate state and sit idle. **Cloud agents** are provisioned on demand by a cloud plugin (kubernetes, EC2, Azure VM, Docker) when the queue needs them and removed after use or idle timeout. In 2025–2026 most teams run ephemeral Kubernetes pods for Linux builds and keep a few static agents for macOS, Windows or hardware-in-the-loop.

</details>

### Q12. What Java version does current Jenkins require? 🟡

<details>
<summary><b>Answer</b></summary>

Jenkins LTS releases since 2.463.1 require **Java 17 or 21** on the controller and on agents (remoting runs on the agent JVM too). Java 11 support ended in 2024. The official images come as `jenkins/jenkins:lts-jdk17` and `lts-jdk21`; we use JDK 21. Your build tools can use a different JDK, installed in the build container or through the `tools` directive, because the agent JVM and the build JDK are independent.

</details>

### Q13. LTS vs weekly releases: which one do you run and how does the version numbering work? 🟢

<details>
<summary><b>Answer</b></summary>

Weekly releases (`2.4xx`, `2.5xx`) ship every week with new features. LTS is a baseline weekly picked roughly every 12 weeks and stabilised with backports, numbered `2.<base>.<patch>` such as `2.479.3`. Production runs LTS, upgraded within a few weeks of each new LTS line and immediately for security advisories.

</details>

### Q14. What is stored in a build record? 🟢

<details>
<summary><b>Answer</b></summary>

`jobs/<job>/builds/<number>/` contains `build.xml` (result, duration, causes, parameters, actions), `log` (console output), `changelog*.xml` (SCM changes), `junitResult.xml`, `archive/` (archived artifacts unless an external artifact manager is used), and for pipelines `workflow/` or `workflow-completed/` with flow node data. Old records are removed by `buildDiscarder`. Build records are the part of Jenkins that cannot be recreated from Git, so they are what backups really protect.

</details>

### Q15. What is the "quiet period"? 🟢

<details>
<summary><b>Answer</b></summary>

A delay (default 5 seconds globally, overridable per job with `options { quietPeriod(30) }`) between scheduling a build and starting it. Triggers that arrive during the quiet period are merged into the same queued build. It is useful when several commits land in quick succession and you do not want a build per commit.

</details>

### Q16. How does Jenkins decide where to run a build when several nodes match? 🔴

<details>
<summary><b>Answer</b></summary>

The default `LoadBalancer` uses consistent hashing on the job name to prefer the node the job last ran on (to reuse the workspace and caches), falling back to other matching nodes with free executors. If none are free, the item waits; cloud plugins observe the queue through `NodeProvisioner` and provision new nodes based on load statistics. Plugins like Least Load or Throttle Concurrent Builds change the strategy. With Kubernetes agents the hashing barely matters because every build gets a fresh pod.

</details>

### Q17. What are flyweight executors? 🔴

<details>
<summary><b>Answer</b></summary>

A pipeline build itself needs a lightweight "flyweight" task on the controller that runs the CPS program. It does not consume one of the configured executors, which is why a pipeline appears to run even when the built-in node has 0 executors. Only `node {}` or declarative `agent` blocks take a real ("heavyweight") executor on an agent. With `agent none` at the top and `input` steps outside any agent, a waiting build holds no executor at all.

</details>

### Q18. Freestyle job vs Pipeline job: why is Freestyle considered legacy? 🟢

<details>
<summary><b>Answer</b></summary>

Freestyle jobs are configured in the UI: build steps, post-build actions, stored only in `config.xml`. They cannot express complex flows, are not code-reviewed, and do not survive controller restarts mid-build. Pipeline jobs define the process in a `Jenkinsfile` stored with the code, support stages, parallelism, approvals, resume after restart, and shared libraries. Freestyle is still fine for trivial admin tasks, but new delivery pipelines should be Pipeline or Multibranch.

</details>

### Q19. What are the main job (item) types in Jenkins? 🟢

<details>
<summary><b>Answer</b></summary>

- **Freestyle project**: UI-configured steps.
- **Pipeline**: one Jenkinsfile, inline or from SCM, single branch.
- **Multibranch Pipeline**: one child job per branch, PR and tag containing a Jenkinsfile.
- **Organization Folder**: scans an entire GitHub org or Bitbucket project and creates a multibranch job per repo.
- **Folder**: groups items and scopes credentials, libraries and permissions.
- **Multi-configuration (matrix) project**: legacy axis-based job, replaced by the declarative `matrix` directive.

</details>

### Q20. What happens to running pipeline builds when the controller restarts? 🔴

<details>
<summary><b>Answer</b></summary>

Pipelines are designed to survive a restart. The CPS program state is persisted to `build.xml`/`program.dat`, and on startup Jenkins resumes each build. Running `sh` steps on agents keep going because durable tasks write output to files on the agent and Jenkins reattaches. This fails if the agent also died (Kubernetes pods are often gone), if the pipeline uses `PERFORMANCE_OPTIMIZED` durability, or if the build had `disableResume()`. Use **Prepare for Shutdown** (`/safeExit` or `/quietDown`) to let builds finish before maintenance.

</details>

### Q21. What is the difference between `/restart`, `/safeRestart`, `/quietDown` and `/safeExit`? 🟡

<details>
<summary><b>Answer</b></summary>

| Endpoint | Behaviour |
|----------|-----------|
| `/restart` | Restarts immediately, interrupting builds (pipelines try to resume) |
| `/safeRestart` | Stops starting new builds, waits for running ones, then restarts |
| `/quietDown` | Prepare for shutdown: no new builds start; `/cancelQuietDown` reverts |
| `/exit` and `/safeExit` | Shut down now, or after builds finish |

On Kubernetes, prefer `safeExit` before deleting the controller pod, or rely on graceful termination with a long `terminationGracePeriodSeconds`.

</details>

### Q22. What is the agent work directory and why does `-workDir` matter? 🟢

<details>
<summary><b>Answer</b></summary>

`-workDir` tells `agent.jar` where to keep its remoting cache, logs (`remoting/logs`) and workspaces. Without it, remoting uses the current directory and writes logs to stderr, making connection problems hard to debug and jar caches end up in the user's home. Always set it, for example `-workDir /home/jenkins/agent`, and put it on a disk with enough space for workspaces.

</details>

### Q23. What does the `jnlp` container do in a Kubernetes agent pod? 🟡

<details>
<summary><b>Answer</b></summary>

It runs `jenkins/inbound-agent`, which starts `agent.jar` and connects back to the controller. It is the remoting endpoint for the pod; other containers (maven, node, kaniko) are just tools that share the workspace volume. Steps run in `jnlp` by default unless you wrap them in `container('name')` or set `defaultContainer`. If the `jnlp` container cannot reach the controller URL, the pod never becomes an agent and the build waits with `Waiting for agent to connect`.

</details>

### Q24. What is the Jenkins CLI and how do you authenticate it? 🟢

<details>
<summary><b>Answer</b></summary>

`jenkins-cli.jar` (downloadable from `/jnlpJars/jenkins-cli.jar`) issues admin commands over HTTP or SSH. Authenticate with a user and API token, never a password.

```bash
java -jar jenkins-cli.jar -s https://jenkins.example.com/ -auth alice:$TOKEN -webSocket who-am-i
java -jar jenkins-cli.jar -s https://jenkins.example.com/ -auth alice:$TOKEN build myjob -p ENV=dev -f -v
```

`-f` waits for completion and `-v` streams the console log.

</details>

### Q25. How do you call the Jenkins REST API? 🟢

<details>
<summary><b>Answer</b></summary>

Append `/api/json` (or `/api/xml`) to almost any URL, and use the `tree` parameter to limit fields. POST requests need an API token (which bypasses the CSRF crumb) or a crumb with a session.

```bash
curl -s -u alice:$TOKEN "https://jenkins/job/app/job/main/lastBuild/api/json?tree=result,number,duration"
curl -X POST -u alice:$TOKEN "https://jenkins/job/deploy/buildWithParameters?ENV=dev"
curl -s -u alice:$TOKEN "https://jenkins/job/app/job/main/42/consoleText"
```

</details>

---

## Declarative pipeline syntax

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. What is the minimum valid declarative pipeline? 🟢

<details>
<summary><b>Answer</b></summary>

A `pipeline` block with an `agent`, a `stages` block containing at least one `stage`, and each stage containing `steps` (or nested `stages`, `parallel` or `matrix`).

```groovy
pipeline {
  agent any
  stages {
    stage('Build') {
      steps { sh 'echo hello' }
    }
  }
}
```

Missing `agent` fails with `Missing required section "agent"`.

</details>

### Q27. What agent types exist in declarative pipelines? 🟡

<details>
<summary><b>Answer</b></summary>

| Type | Meaning |
|------|---------|
| `agent any` | Any available executor |
| `agent none` | No global agent; each stage declares its own |
| `agent { label 'linux && docker' }` | Node matching the expression |
| `agent { node { label 'linux'; customWorkspace '/data/ws/app' } }` | Label plus options |
| `agent { docker { image 'maven:3.9-eclipse-temurin-21'; args '-v m2:/root/.m2' } }` | Run steps inside a container on a Docker-capable node |
| `agent { dockerfile { filename 'ci/Dockerfile' } }` | Build and use an image from the repo |
| `agent { kubernetes { yamlFile 'ci/pod.yaml'; defaultContainer 'maven' } }` | Pod via the kubernetes plugin |

</details>

### Q28. When do you use `agent none` at the top level? 🟡

<details>
<summary><b>Answer</b></summary>

When stages need different agents (Linux build, Windows test, deploy pod with a special service account) or when the pipeline has an `input` step that may wait for hours. With `agent none`, the approval stage holds no executor and no pod. The trade-off: each stage with its own agent gets a separate workspace, so pass files with `stash`/`unstash` or artifacts, or group stages under a parent stage that owns the agent.

</details>

### Q29. How do nested sequential stages work and why are they useful? 🟡

<details>
<summary><b>Answer</b></summary>

A stage can contain a `stages` block instead of `steps`. The child stages run in order and share the parent's agent and workspace, while still appearing individually in the UI.

```groovy
stage('CI') {
  agent { kubernetes { yamlFile 'ci/build-pod.yaml' } }
  stages {
    stage('Test')  { steps { sh 'npm test' } }
    stage('Build') { steps { sh 'npm run build' } }
  }
}
```

This is how we combine `agent none` at the top with one pod for all CI steps.

</details>

### Q30. List the `post` conditions and when each fires. 🟡

<details>
<summary><b>Answer</b></summary>

| Condition | Fires when |
|-----------|-----------|
| `always` | Every time |
| `changed` | Result differs from the previous build |
| `fixed` | Now SUCCESS, previous was FAILURE or UNSTABLE |
| `regression` | Now FAILURE, UNSTABLE or ABORTED, previous was SUCCESS |
| `aborted` | ABORTED |
| `failure` | FAILURE |
| `success` | SUCCESS |
| `unstable` | UNSTABLE |
| `unsuccessful` | Anything other than SUCCESS |
| `cleanup` | After all others, regardless of result |

`post` can sit at pipeline level or stage level.

</details>

### Q31. What are the most useful `when` conditions? 🟡

<details>
<summary><b>Answer</b></summary>

```groovy
when { branch 'main' }                       // multibranch only, uses BRANCH_NAME
when { branch pattern: 'release/\\d+', comparator: 'REGEXP' }
when { changeRequest target: 'main' }        // PR builds
when { buildingTag() }  /  when { tag 'v*' }
when { changeset 'services/api/**' }
when { environment name: 'DEPLOY', value: 'true' }
when { expression { params.RUN_E2E } }
when { triggeredBy 'TimerTrigger' }
when { allOf { branch 'main'; not { changeRequest() } } }
when { anyOf { branch 'main'; branch 'hotfix/*' } }
```

</details>

### Q32. What do `beforeAgent`, `beforeInput` and `beforeOptions` do inside `when`? 🔴

<details>
<summary><b>Answer</b></summary>

By default a stage's `when` is evaluated after entering its `agent`, after its `input` and after its `options` are applied. That wastes a pod on a skipped stage, or asks for approval on a branch that will skip anyway.

```groovy
stage('Deploy prod') {
  agent { kubernetes { yamlFile 'ci/deploy-pod.yaml' } }
  when { beforeAgent true; beforeInput true; branch 'main' }
  input { message 'Deploy to prod?'; submitter 'release-managers' }
  steps { sh './deploy.sh prod' }
}
```

`beforeAgent true` skips pod creation; `beforeInput true` skips the prompt on non-main branches.

</details>

### Q33. How does `parallel` work in declarative, and what is `failFast`? 🟡

<details>
<summary><b>Answer</b></summary>

A stage containing `parallel` runs its child stages concurrently, each optionally with its own agent. `failFast true` (or `options { parallelsAlwaysFailFast() }`) aborts the other branches as soon as one fails.

```groovy
stage('Checks') {
  failFast true
  parallel {
    stage('Unit') { steps { sh 'npm test' } }
    stage('Lint') { steps { sh 'npm run lint' } }
    stage('Audit') { steps { sh 'npm audit --audit-level=high' } }
  }
}
```

A stage cannot have both `steps` and `parallel`.

</details>

### Q34. How does the `matrix` directive work? 🟡

<details>
<summary><b>Answer</b></summary>

`matrix` generates one parallel cell per combination of `axes`, with optional `excludes`. Each cell runs the inner `stages`, and axis values are available as environment variables.

```groovy
stage('Test matrix') {
  matrix {
    axes {
      axis { name 'NODE'; values '20', '22' }
      axis { name 'OS';   values 'linux', 'windows' }
    }
    excludes { exclude { axis { name 'OS'; values 'windows' }; axis { name 'NODE'; values '20' } } }
    agent { label "${OS}" }
    stages { stage('Test') { steps { echo "Node ${NODE} on ${OS}" } } }
  }
}
```

Watch the cell count: axes multiply, and large matrices can hit the JVM `Method code too large` limit.

</details>

### Q35. Name the `options` you use in almost every production pipeline. 🟡

<details>
<summary><b>Answer</b></summary>

```groovy
options {
  timeout(time: 45, unit: 'MINUTES')
  buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '5'))
  disableConcurrentBuilds(abortPrevious: true)
  timestamps()                 // timestamper plugin; or ansiColor('xterm')
  skipDefaultCheckout()        // when you checkout explicitly
  durabilityHint('PERFORMANCE_OPTIMIZED')
  retry(1)                     // at stage level, for flaky infra
}
```

`disableConcurrentBuilds(abortPrevious: true)` is very useful for PR builds: a new push cancels the outdated build.

</details>

### Q36. What is the difference between pipeline-level and stage-level `options`? 🟡

<details>
<summary><b>Answer</b></summary>

Pipeline-level options apply to the whole run (`buildDiscarder`, `disableConcurrentBuilds`, `durabilityHint` are pipeline-only). Stage-level options support a subset: `timeout`, `retry`, `timestamps`, `skipDefaultCheckout`, `lock`. A stage-level `timeout` includes time spent waiting for the agent and for `input`, so put a timeout on approval stages to avoid builds waiting forever.

</details>

### Q37. How does the `environment` directive work, and what values can it hold? 🟡

<details>
<summary><b>Answer</b></summary>

`environment` sets environment variables for the whole pipeline or a single stage. Values must be single-quoted or double-quoted strings, or function calls such as `credentials()`.

```groovy
environment {
  APP = 'shack-api'
  IMAGE = "docker.io/devopsshack/${APP}"
  SONAR = credentials('sonar-token')
}
```

To compute a value with logic, use `script { env.VERSION = sh(script: 'git describe --tags', returnStdout: true).trim() }` in a step. Stage-level environment overrides pipeline-level for that stage.

</details>

### Q38. What does `credentials('id')` produce for each credential type in `environment`? 🟡

<details>
<summary><b>Answer</b></summary>

| Credential type | Variables created for `X = credentials('id')` |
|-----------------|-----------------------------------------------|
| Secret text | `X` = the secret |
| Username with password | `X` = `user:pass`, `X_USR`, `X_PSW` |
| Secret file | `X` = path to a temp file |
| SSH username with private key | `X` = path to key file, `X_USR` (and `X_PSW` for passphrase) |

All values are masked in the console log and the temp files are deleted when the block ends.

</details>

### Q39. How does `withCredentials` differ from `credentials()` in `environment`? 🟡

<details>
<summary><b>Answer</b></summary>

`withCredentials` binds secrets only for the enclosed block, supports more binding types and lets you choose variable names.

```groovy
withCredentials([
  usernamePassword(credentialsId: 'nexus', usernameVariable: 'NX_USER', passwordVariable: 'NX_PASS'),
  file(credentialsId: 'kubeconfig-prod', variable: 'KUBECONFIG'),
  sshUserPrivateKey(credentialsId: 'deploy-key', keyFileVariable: 'KEY'),
  string(credentialsId: 'slack-webhook', variable: 'SLACK_URL')
]) {
  sh 'kubectl --kubeconfig "$KUBECONFIG" get ns'
}
```

Narrow scope is safer: a secret bound at pipeline level is visible to every step and every script.

</details>

### Q40. Why must secrets be used with single quotes in `sh`? 🟡

<details>
<summary><b>Answer</b></summary>

With double quotes Groovy interpolates the secret into the command string before it reaches the shell, so the plain value is passed as the script argument, may appear in `ps` on the agent and in the step's arguments in the build record. Jenkins warns: `A secret was passed to "sh" using Groovy String interpolation, which is insecure.`

```groovy
sh "curl -u $NX_USER:$NX_PASS ..."   // bad: Groovy interpolation
sh 'curl -u "$NX_USER:$NX_PASS" ...' // good: shell expands env vars
```

</details>

### Q41. How does the declarative `input` directive work? 🟡

<details>
<summary><b>Answer</b></summary>

A stage-level `input` pauses before the stage runs and asks a user to proceed or abort. `submitter` restricts who can approve (users or groups, comma-separated), and `parameters` collects values that become environment variables in the stage.

```groovy
stage('Deploy prod') {
  options { timeout(time: 4, unit: 'HOURS') }
  input {
    message 'Promote to production?'
    ok 'Deploy'
    submitter 'release-managers,alice'
    submitterParameter 'APPROVER'
    parameters { string(name: 'CHANGE_ID', description: 'Change ticket') }
  }
  steps { sh './deploy.sh prod "$CHANGE_ID"'; echo "Approved by ${APPROVER}" }
}
```

</details>

### Q42. Stage-level `input` directive vs the `input` step: what is the difference? 🔴

<details>
<summary><b>Answer</b></summary>

The **directive** is evaluated before the stage's `agent` is allocated, so no executor or pod is held while waiting. The **step** (`input message: '...'` inside `steps` or `script`) runs wherever it is called; inside an agent block it keeps the executor busy (and the pod alive) for the whole wait. The step returns a value you can assign in `script {}`; the directive exposes parameters as env vars. For approvals, prefer the directive plus `agent none` at the top, and always a `timeout`.

</details>

### Q43. What parameter types are available in the `parameters` directive? 🟢

<details>
<summary><b>Answer</b></summary>

```groovy
parameters {
  string(name: 'VERSION', defaultValue: '1.0.0', description: 'Release version')
  text(name: 'NOTES', defaultValue: '')
  booleanParam(name: 'RUN_E2E', defaultValue: false)
  choice(name: 'ENV', choices: ['dev', 'staging', 'prod'])
  password(name: 'TEMP_PASS', defaultValue: '')
}
```

Access with `params.VERSION`. Parameters defined in a Jenkinsfile only appear in the UI after the first run, because Jenkins must execute the file once to learn them.

</details>

### Q44. What triggers can a declarative pipeline declare? 🟡

<details>
<summary><b>Answer</b></summary>

```groovy
triggers {
  cron('H 2 * * 1-5')                 // nightly on weekdays
  pollSCM('H/15 * * * *')             // polling, avoid when webhooks work
  upstream(upstreamProjects: 'lib-build', threshold: hudson.model.Result.SUCCESS)
}
```

In multibranch jobs, pushes are handled by branch source events, so `triggers` is mainly used for cron. Note that `triggers` in a branch Jenkinsfile applies to every branch; guard with `cron(env.BRANCH_NAME == 'main' ? 'H 2 * * *' : '')`.

</details>

### Q45. What does `H` mean in a Jenkins cron expression? 🟢

<details>
<summary><b>Answer</b></summary>

`H` is a hash of the job name, giving a stable but spread-out value within the range. `H 2 * * *` runs every day at 02:xx with a minute fixed per job. It prevents hundreds of jobs from firing at 02:00:00 and spiking the controller. `H/15` means every 15 minutes with a per-job offset; `H(0-29)` limits the range.

</details>

### Q46. What does the `tools` directive do? 🟢

<details>
<summary><b>Answer</b></summary>

It installs (if needed) and puts on `PATH` a tool configured globally under Tools, such as a JDK, Maven or NodeJS installation.

```groovy
tools { jdk 'temurin-21'; maven 'maven-3.9' }
```

The name must match the global tool configuration (often set via JCasC). With container-based agents we usually skip `tools` and pick an image that already contains the right versions, which is faster and reproducible.

</details>

### Q47. `stash`/`unstash` vs `archiveArtifacts`: when do you use each? 🟡

<details>
<summary><b>Answer</b></summary>

`stash` saves files to the controller for use later in the **same build**, typically to move build output between agents; stashes are discarded at build end. `archiveArtifacts` keeps files with the **build record** for download after the build. Both pass data through the controller, so neither suits large binaries: push packages to Nexus/Artifactory and images to a registry instead.

```groovy
stash name: 'dist', includes: 'dist/**'
unstash 'dist'
archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
```

</details>

### Q48. What does `skipDefaultCheckout` do, and when do you need it? 🟡

<details>
<summary><b>Answer</b></summary>

By default a declarative pipeline from SCM checks out the repo automatically when an agent is allocated (at pipeline level and in each stage with its own agent). `options { skipDefaultCheckout() }` disables that. Use it when stages do not need the source (deploy pods, notification stages), when you need custom checkout options (shallow clone, sparse checkout, submodules), or to avoid repeated clones of a large repo.

```groovy
checkout scmGit(branches: [[name: env.BRANCH_NAME]], extensions: [cloneOption(shallow: true, depth: 1, noTags: true)], userRemoteConfigs: scm.userRemoteConfigs)
```

</details>

### Q49. What are the common built-in environment variables in a pipeline? 🟢

<details>
<summary><b>Answer</b></summary>

`BUILD_NUMBER`, `BUILD_ID`, `BUILD_URL`, `JOB_NAME`, `JOB_BASE_NAME`, `WORKSPACE`, `NODE_NAME`, `EXECUTOR_NUMBER`, `JENKINS_URL`. In multibranch: `BRANCH_NAME`, `CHANGE_ID`, `CHANGE_TARGET`, `CHANGE_BRANCH`, `CHANGE_AUTHOR`, `TAG_NAME`. From the Git plugin after checkout: `GIT_COMMIT`, `GIT_BRANCH`, `GIT_URL`, `GIT_PREVIOUS_SUCCESSFUL_COMMIT`. The full list is at `${JENKINS_URL}pipeline-syntax/globals`.

</details>

### Q50. How do you set the build result without failing the step? 🟡

<details>
<summary><b>Answer</b></summary>

```groovy
unstable(message: 'Coverage below 80%')          // UNSTABLE, continue
catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
  sh './integration-tests.sh'
}
warnError('Lint failed') { sh 'npm run lint' }    // stage and build UNSTABLE
error('Missing release notes')                    // fail now
```

`catchError` lets the pipeline continue and marks the stage red but the build yellow, which is useful for non-blocking checks.

</details>

### Q51. What do `retry`, `timeout` and `waitUntil` do as steps? 🔴

<details>
<summary><b>Answer</b></summary>

```groovy
retry(count: 3, conditions: [agent(), kubernetesAgent(), nonresumable()]) {
  sh 'mvn -B deploy'
}
timeout(time: 10, unit: 'MINUTES') { sh './e2e.sh' }
timeout(time: 5, unit: 'MINUTES') {
  waitUntil(initialRecurrencePeriod: 5000) {
    sh(script: 'curl -fs https://app/healthz', returnStatus: true) == 0
  }
}
```

`retry` with `conditions` retries only on infrastructure failures (agent lost, pod deleted), not on real test failures, which is what you want in production.

</details>

### Q52. How do you capture a command's output or exit code with `sh`? 🟢

<details>
<summary><b>Answer</b></summary>

```groovy
script {
  def sha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
  def rc  = sh(script: 'grep -q TODO src/*.js', returnStatus: true)
  if (rc == 0) { unstable('TODOs found') }
  env.SHORT_SHA = sha
}
```

`returnStdout` includes the trailing newline, so always `.trim()`. `returnStatus` prevents a non-zero exit from failing the build.

</details>

### Q53. What does `lock` do and which plugin provides it? 🟡

<details>
<summary><b>Answer</b></summary>

The Lockable Resources plugin provides `lock`, which serialises access to a named resource across all jobs: a shared test environment, a database, a device.

```groovy
stage('Integration') {
  options { lock(resource: 'staging-db') }
  steps { sh './run-integration.sh' }
}
```

`lock(label: 'test-env', quantity: 1, variable: 'ENV_NAME')` picks one free resource from a pool and exposes its name. `disableConcurrentBuilds` only limits one job; `lock` works across jobs.

</details>

### Q54. What does `milestone` do? 🔴

<details>
<summary><b>Answer</b></summary>

`milestone(ordinal)` ensures builds pass a point in order: when a newer build passes milestone N, older builds that have not yet reached N are aborted. Combined with `input` it prevents an old build from being approved and deploying over a newer one.

```groovy
milestone(1)
input 'Deploy?'
milestone(2)
```

In declarative pipelines, `disableConcurrentBuilds(abortPrevious: true)` covers the common case more simply.

</details>

### Q55. How do you trigger another job from a pipeline and pass parameters? 🟢

<details>
<summary><b>Answer</b></summary>

```groovy
def r = build job: 'deploy/shack-api', wait: true, propagate: false,
  parameters: [string(name: 'IMAGE_TAG', value: env.TAG), booleanParam(name: 'DRY_RUN', value: false)]
echo "Downstream result: ${r.result}, build ${r.number}"
```

Use relative paths like `../deploy` inside folders. `propagate: false` keeps the upstream build from failing automatically so you can decide. `wait: false` fires and forgets.

</details>

### Q56. What do `junit` and `recordIssues` do? 🟢

<details>
<summary><b>Answer</b></summary>

`junit 'reports/**/*.xml'` parses JUnit XML, shows test trends, and marks the build UNSTABLE if any test failed. `recordIssues` (Warnings NG plugin) collects static analysis results such as ESLint, Checkstyle or SpotBugs with quality gates.

```groovy
post { always { junit allowEmptyResults: false, testResults: 'reports/junit.xml' } }
recordIssues tools: [esLint(pattern: 'eslint.xml')], qualityGates: [[threshold: 1, type: 'TOTAL_ERROR', unstable: false]]
```

</details>

### Q57. What are the `dir`, `ws` and `deleteDir` steps? 🟢

<details>
<summary><b>Answer</b></summary>

`dir('frontend') { sh 'npm ci' }` changes directory for the block (creating it if needed). `ws('/data/custom') { }` allocates a different workspace. `deleteDir()` deletes the current directory recursively, often used as `dir('tmp') { deleteDir() }`. For full cleanup use `cleanWs()` from the Workspace Cleanup plugin in `post { cleanup { } }`.

</details>

### Q58. How do you load configuration files from the repo in a pipeline? 🟡

<details>
<summary><b>Answer</b></summary>

With the Pipeline Utility Steps plugin:

```groovy
script {
  def cfg  = readYaml file: 'deploy/config.yaml'
  def pkg  = readJSON file: 'package.json'
  def props = readProperties file: 'build.properties'
  env.APP_VERSION = pkg.version
  writeYaml file: 'out.yaml', data: cfg, overwrite: true
}
```

These run on the agent workspace but parse on the controller. Avoid `readFile` + `JsonSlurper` in pipeline code: `JsonSlurper` returns a non-serializable `LazyMap`.

</details>

### Q59. What is the Declarative Directive Generator and Snippet Generator? 🟢

<details>
<summary><b>Answer</b></summary>

Both live under `<job>/pipeline-syntax`. The **Snippet Generator** produces correct step syntax (`checkout`, `withCredentials`, `junit`) for the plugins actually installed on your controller. The **Directive Generator** produces declarative directives (`agent`, `options`, `triggers`, `when`). The same page links to **Global Variables Reference** listing `env`, `params`, `currentBuild`, `scm` and library-defined globals. It is the fastest way to get a plugin's parameter names right.

</details>

### Q60. How do you validate a Jenkinsfile without running it? 🟡

<details>
<summary><b>Answer</b></summary>

Use the declarative linter via CLI or REST, and the "Replay" feature for quick experiments.

```bash
curl -s -X POST -u alice:$TOKEN -F "jenkinsfile=<Jenkinsfile" https://jenkins/pipeline-model-converter/validate
java -jar jenkins-cli.jar -s https://jenkins/ -auth alice:$TOKEN declarative-linter < Jenkinsfile
```

The linter checks syntax and structure only, not whether steps or credentials exist. Editors such as VS Code have Jenkinsfile extensions that call the same endpoint.

</details>

---

## Scripted pipeline, Groovy & CPS

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q61. What does a scripted pipeline look like and when would you still write one? 🟡

<details>
<summary><b>Answer</b></summary>

```groovy
node('linux') {
  stage('Checkout') { checkout scm }
  stage('Build') {
    try {
      sh 'make build'
    } catch (err) {
      currentBuild.result = 'FAILURE'
      throw err
    } finally {
      junit 'reports/*.xml'
    }
  }
}
```

We still use scripted style inside shared library steps, for dynamically generated stages (one per service discovered at runtime), and for legacy jobs. For application Jenkinsfiles, declarative is the default.

</details>

### Q62. What is CPS and why does Jenkins transform pipeline Groovy? 🔴

<details>
<summary><b>Answer</b></summary>

CPS (continuation-passing style) is a program transformation done by the `workflow-cps` plugin (using `groovy-cps`). Each pipeline method is rewritten so that its execution state (call stack, local variables, program counter) is an explicit object that can be serialized to disk. That makes pipelines **durable**: Jenkins can persist the program after each step and resume it after a controller restart. The cost: slower execution, restrictions on Groovy features, and the requirement that all live variables be `Serializable`.

</details>

### Q63. Why does `java.io.NotSerializableException` happen in pipelines? 🔴

<details>
<summary><b>Answer</b></summary>

At each step boundary, CPS serializes the program state. If a local variable holds an object that does not implement `Serializable` (such as `java.util.regex.Matcher`, `groovy.json.internal.LazyMap`, an `Iterator`, an HTTP connection), serialization fails. Fixes:

- Do not keep such objects across a step: compute, extract plain values, and null the variable.
- Move the logic into a `@NonCPS` method that returns plain `String`/`List`/`Map`.
- Use serializable alternatives: `readJSON` instead of `JsonSlurper`, `findAll` instead of `matcher` loops.

</details>

### Q64. What does `@NonCPS` do and what are its rules? 🔴

<details>
<summary><b>Answer</b></summary>

It marks a method to run as ordinary compiled Groovy, outside CPS. It is faster and can use non-serializable objects internally. Rules:

- It must not call pipeline steps (`sh`, `echo`, `readFile`); behaviour is undefined (usually the step runs once and the method returns oddly).
- It cannot be resumed mid-method after a restart.
- Its return value goes back into CPS code, so it must be serializable.

```groovy
@NonCPS
List<String> extractVersions(String text) {
  (text =~ /v(\d+\.\d+\.\d+)/).collect { it[1] }
}
```

</details>

### Q65. Why do some Groovy closures behave unexpectedly in pipelines, for example `list.each { }` or `sort`? 🔴

<details>
<summary><b>Answer</b></summary>

CPS has its own implementations of many collection methods, and historically `each`, `collect`, `sort` with closures, and similar methods behaved differently (only the first element processed, or `expected to call ... but wound up catching ...` warnings). Modern `workflow-cps` versions support most of them, but `sort` with a comparator closure and some `Comparator`-based APIs still misbehave in CPS. Plain `for (x in list)` loops are always safe; put complex collection manipulation in `@NonCPS` helpers.

</details>

### Q66. What is the `script` block in declarative, and what are its limits? 🟢

<details>
<summary><b>Answer</b></summary>

`script { }` embeds scripted Groovy inside a declarative `steps` block: variables, `if`, loops, `try/catch`, calling methods. It is the escape hatch for logic declarative cannot express. Keep it short; if a `script` block grows beyond ~10 lines or appears in several pipelines, move it into a shared library step. Variables declared with `def` inside `script` are local to that block; use `env.X` to share values across stages.

</details>

### Q67. How do you share a variable between stages in declarative? 🟡

<details>
<summary><b>Answer</b></summary>

Options, from cleanest to messiest:

1. `env.IMAGE_TAG = "..."` inside `script {}`; env vars are strings only.
2. A variable declared outside the `pipeline {}` block (`def tag`) and assigned in `script {}`; works but is global mutable state.
3. Return values from a shared library step.

```groovy
def releaseTag
pipeline {
  agent any
  stages {
    stage('Version') { steps { script { releaseTag = sh(script: 'git describe --tags', returnStdout: true).trim() } } }
    stage('Show')    { steps { echo "Tag: ${releaseTag}" } }
  }
}
```

</details>

### Q68. How do you generate parallel stages dynamically? 🔴

<details>
<summary><b>Answer</b></summary>

Build a map of branch names to closures and pass it to the `parallel` step inside `script {}`.

```groovy
script {
  def services = sh(script: 'ls services', returnStdout: true).trim().split('\n')
  def branches = [:]
  for (s in services) {
    def svc = s   // capture per iteration
    branches[svc] = { stage(svc) { sh "make -C services/${svc} test" } }
  }
  branches.failFast = true
  parallel branches
}
```

The `def svc = s` copy matters: otherwise every closure captures the same loop variable.

</details>

### Q69. What is the `currentBuild` global and what is useful on it? 🟡

<details>
<summary><b>Answer</b></summary>

`currentBuild` is a `RunWrapper` for the running build: `result`, `currentResult`, `number`, `displayName`, `description`, `durationString`, `previousBuild`, `changeSets`, `getBuildCauses()`, `buildVariables`. We set `currentBuild.displayName = "#${BUILD_NUMBER} ${TAG}"` and `currentBuild.description` for readability. `currentBuild.rawBuild` exposes the internal `Run` object but needs script approval and should be avoided.

</details>

### Q70. How do you find out what triggered a build? 🟡

<details>
<summary><b>Answer</b></summary>

```groovy
script {
  def userCause = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')
  if (userCause) { echo "Started by ${userCause[0].userId}" }
  def timer = currentBuild.getBuildCauses('hudson.triggers.TimerTrigger$TimerTriggerCause')
}
```

In declarative, `when { triggeredBy 'UserIdCause' }` or `triggeredBy cause: 'UserIdCause', detail: 'alice'` handles the common cases without code.

</details>

### Q71. Why is heavy Groovy in a Jenkinsfile bad for the controller? 🟡

<details>
<summary><b>Answer</b></summary>

All pipeline Groovy (loops, string processing, JSON parsing, `readFile` content handling) runs on the controller JVM under CPS, which is much slower than normal Groovy and allocates heavily. One pipeline parsing a 50 MB JSON log can stall the controller for everyone. Push work into `sh` steps on the agent (`jq`, Python, a CLI tool) and keep Groovy as orchestration only.

</details>

### Q72. What is the `load` step and why do shared libraries replace it? 🟡

<details>
<summary><b>Answer</b></summary>

`load 'ci/helpers.groovy'` evaluates a Groovy file from the workspace and returns its object, so you can call `helpers.deploy()`. It requires a workspace (an agent), is not versioned independently, cannot hold classes cleanly, and every repo copies the same file. Shared libraries are versioned, loaded before the pipeline starts, support classes and resources, and are reused across repos.

</details>

### Q73. What does `evaluate` or `GroovyShell` inside a pipeline mean for security? 🔴

<details>
<summary><b>Answer</b></summary>

Arbitrary Groovy evaluation in an unsandboxed context (a trusted library, or an admin-approved script) runs with full controller privileges: it can read `Jenkins.instance`, decrypt credentials, and modify configuration. In a sandboxed Jenkinsfile, such calls are blocked and require script approval. Admins should never approve signatures like `staticMethod org.codehaus.groovy.runtime.DefaultGroovyMethods execute java.lang.String` or `new groovy.lang.GroovyShell`, because they turn any Jenkinsfile author into a controller admin.

</details>

### Q74. What is the difference between `env.X`, `params.X` and plain variables? 🟢

<details>
<summary><b>Answer</b></summary>

`params.X` is a read-only map of build parameters with their real types (boolean stays boolean). `env.X` is the environment of the build; values are strings, so `env.RUN_E2E == 'true'` not `true`. Parameters are also copied into `env`. Plain Groovy variables live only in the script and are not visible to `sh` unless you pass them in. A common bug is `if (env.DEBUG)` being true for the string `'false'`.

</details>

### Q75. What is pipeline durability and what are the three durability hints? 🔴

<details>
<summary><b>Answer</b></summary>

Durability controls how often the pipeline state is written to disk.

| Hint | Behaviour |
|------|-----------|
| `MAX_SURVIVABILITY` | Writes after every step; safest, most disk I/O |
| `SURVIVABLE_NONATOMIC` | Writes every step but non-atomically |
| `PERFORMANCE_OPTIMIZED` | Writes only at certain points; much less I/O, but a crash can lose the running build |

Set per pipeline with `options { durabilityHint('PERFORMANCE_OPTIMIZED') }` or globally. We use performance-optimized for most CI pipelines (rerun is cheap) and max survivability for long release pipelines.

</details>

### Q76. What is "Restart from Stage" and when does it not work? 🟡

<details>
<summary><b>Answer</b></summary>

For completed declarative builds, you can rerun from a chosen top-level stage, reusing the earlier stages' results. It does not work for scripted pipelines, for stages nested inside `parallel` or `matrix` (you restart the parent), and the workspace is not preserved; stashes are only available if `options { preserveStashes(buildCount: 5) }` was set. It is good for retrying a failed deploy without rebuilding.

</details>

### Q77. What is "Replay" and why is it powerful and dangerous? 🟡

<details>
<summary><b>Answer</b></summary>

Replay reruns a completed build with an edited Jenkinsfile (and edited untrusted library code) without committing. It is great for debugging pipeline syntax quickly. It is dangerous because anyone with Run/Replay permission can run arbitrary pipeline code using the job's credentials. In production we restrict Replay to admins via the `Run/Replay` permission and keep real changes in pull requests.

</details>

### Q78. How do you handle errors in scripted pipeline correctly? 🟡

<details>
<summary><b>Answer</b></summary>

Use `try/catch/finally`, rethrow after handling, and never swallow `FlowInterruptedException` (user abort or timeout), or the build will keep running after someone clicked Abort.

```groovy
try {
  sh './deploy.sh'
} catch (org.jenkinsci.plugins.workflow.steps.FlowInterruptedException e) {
  throw e
} catch (err) {
  currentBuild.result = 'FAILURE'
  slackSend channel: '#deploys', message: "Deploy failed: ${err}"
  throw err
} finally {
  sh 'rm -f kubeconfig'
}
```

</details>

### Q79. Why is `sleep` or `waitUntil` inside an agent block wasteful? 🟢

<details>
<summary><b>Answer</b></summary>

While a build sleeps inside `node {}` or a stage with an agent, it holds the executor and, on Kubernetes, keeps the pod with its CPU and memory requests reserved. Waiting for an external system (a deployment, an approval, a long test run elsewhere) should happen outside agent blocks when possible, or use webhooks such as `waitForQualityGate` that pause cheaply on the controller.

</details>

### Q80. What does the `Method code too large!` error mean and how do you avoid it? 🔴

<details>
<summary><b>Answer</b></summary>

The JVM limits a single method's bytecode to 64 KB. A declarative pipeline is compiled into one large method, so very long Jenkinsfiles with many stages, big `matrix` blocks or long inline scripts exceed the limit and fail at compile time. Fixes: move logic into shared library steps, replace repeated stages with a `matrix` or a generated `parallel`, move long `sh` scripts into files in the repo, and split one giant pipeline into several jobs. Enabling `org.jenkinsci.plugins.pipeline.modeldefinition.parser.RuntimeASTTransformer.SCRIPT_SPLITTING_TRANSFORMATION=true` helps in many cases.

</details>

---

## Shared libraries

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q81. What is the directory structure of a shared library? 🟢

<details>
<summary><b>Answer</b></summary>

```
shack-lib/
├── vars/
│   ├── kanikoBuild.groovy      # global step: kanikoBuild(...)
│   └── kanikoBuild.txt         # help text shown in Global Variables Reference
├── src/
│   └── com/devopsshack/
│       └── ImageTag.groovy     # class: com.devopsshack.ImageTag
├── resources/
│   └── pods/node-build.yaml    # loaded with libraryResource
└── test/                       # JenkinsPipelineUnit tests
```

At least one of `vars/` or `src/` must exist.

</details>

### Q82. How does a `vars/` global step work? 🟢

<details>
<summary><b>Answer</b></summary>

Each `vars/<name>.groovy` file becomes a global variable named `<name>`. If it defines `call(...)`, it can be invoked like a step.

```groovy
// vars/trivyScan.groovy
def call(Map cfg = [:]) {
  String severity = cfg.get('severity', 'HIGH,CRITICAL')
  sh "trivy image --exit-code 1 --ignore-unfixed --severity ${severity} --input ${cfg.input}"
}
```

```groovy
trivyScan(input: 'image.tar')
```

Other methods in the same file can be called as `trivyScan.helper()`.

</details>

### Q83. What goes in `src/` and how is it used? 🟡

<details>
<summary><b>Answer</b></summary>

`src/` holds Groovy classes on a standard package path, imported from pipelines or `vars/` files.

```groovy
// src/com/devopsshack/ImageTag.groovy
package com.devopsshack
class ImageTag implements Serializable {
  static String forBranch(String branch, String sha) {
    "${branch.replaceAll('[^a-zA-Z0-9_.-]', '-')}-${sha.take(7)}"
  }
}
```

```groovy
@Library('shack-lib@v1') import com.devopsshack.ImageTag
def tag = ImageTag.forBranch(env.BRANCH_NAME, env.GIT_COMMIT)
```

Classes that need to call steps must receive the script object (`this`) and call `script.sh(...)`.

</details>

### Q84. How do classes in `src/` call pipeline steps? 🟡

<details>
<summary><b>Answer</b></summary>

Steps exist on the pipeline script object, not inside your class. Pass the script in and keep the class serializable.

```groovy
package com.devopsshack
class Deployer implements Serializable {
  def steps
  Deployer(steps) { this.steps = steps }
  void rollout(String ns, String deploy) {
    steps.sh "kubectl -n ${ns} rollout status deploy/${deploy} --timeout=120s"
  }
}
```

```groovy
new com.devopsshack.Deployer(this).rollout('shack-dev', 'shack-app')
```

</details>

### Q85. What is `libraryResource` used for? 🟡

<details>
<summary><b>Answer</b></summary>

It reads a file from the library's `resources/` folder as a string: pod templates, shell scripts, config templates, sonar properties. Combine with `writeFile` to use it in the workspace.

```groovy
def script = libraryResource 'scripts/notify.sh'
writeFile file: 'notify.sh', text: script
sh 'bash notify.sh'

podTemplate(yaml: libraryResource('pods/maven.yaml')) { node(POD_LABEL) { ... } }
```

This keeps scripts out of Groovy strings, where quoting is painful.

</details>

### Q86. What are the ways to load a library in a Jenkinsfile? 🟡

<details>
<summary><b>Answer</b></summary>

```groovy
@Library('shack-lib') _                       // default version
@Library('shack-lib@v1.4.0') _               // tag
@Library(['shack-lib@v1', 'sec-lib@main']) _  // several
@Library('shack-lib@v1') import com.devopsshack.ImageTag
library 'shack-lib@v1'                        // dynamic step, at runtime
library identifier: 'adhoc@main', retriever: modernSCM([$class: 'GitSCMSource', remote: 'https://github.com/org/adhoc-lib.git'])
```

The `_` is needed because an annotation must annotate something. Libraries marked **implicit** load automatically without any annotation.

</details>

### Q87. Global vs folder-level vs ad hoc libraries: what is the security difference? 🔴

<details>
<summary><b>Answer</b></summary>

- **Global libraries** (Manage Jenkins > System) are **trusted**: they run outside the Groovy sandbox with full controller access. Only admins can configure them, and whoever can push to the library repo effectively has admin.
- **Folder-level libraries** are **untrusted**: they run in the sandbox, like Jenkinsfiles, so script approval applies.
- **Ad hoc** libraries loaded with `library identifier:` from arbitrary repos are untrusted too.

Protect trusted library repos with branch protection, required reviews and tags.

</details>

### Q88. How do you version a shared library and why should apps pin it? 🟡

<details>
<summary><b>Answer</b></summary>

Tag releases with semver (`v1.4.0`) and maintain a moving major tag or branch (`v1`) for compatible updates. Apps reference `@Library('shack-lib@v1')` to get fixes but not breaking changes. Set `allowVersionOverride` deliberately: if true, any Jenkinsfile can load a branch with untested code; if the library is trusted, that branch runs unsandboxed. Many teams allow overrides only for a `-dev` library configuration used in testing.

</details>

### Q89. What do the "Load implicitly" and "Allow default version to be overridden" options do? 🟡

<details>
<summary><b>Answer</b></summary>

**Load implicitly** makes the library available in every pipeline without `@Library`, at its default version. It is convenient for company-wide steps but means every build depends on it and changes affect everyone instantly. **Allow default version to be overridden** lets a Jenkinsfile request `@Library('lib@feature-x')`. Disable it on production controllers or the library's trust guarantees depend on every branch of the repo being safe.

</details>

### Q90. How do you write a "pipeline template" step in a shared library? 🔴

<details>
<summary><b>Answer</b></summary>

A `vars/` step can contain an entire declarative pipeline, so app repos only pass parameters.

```groovy
// vars/nodeServicePipeline.groovy
def call(Map cfg) {
  pipeline {
    agent { kubernetes { yaml libraryResource('pods/node-build.yaml'); defaultContainer 'node' } }
    options { timeout(time: 30, unit: 'MINUTES') }
    stages {
      stage('Test')  { steps { sh 'npm ci && npm test' } }
      stage('Image') { steps { kanikoBuild(image: cfg.image) } }
    }
  }
}
```

```groovy
@Library('shack-lib@v1') _
nodeServicePipeline(image: 'docker.io/devopsshack/shack-api')
```

Only one `pipeline {}` may run per build. It standardizes teams but reduces flexibility, so expose hooks for extra stages.

</details>

### Q91. How do you unit test a shared library? 🔴

<details>
<summary><b>Answer</b></summary>

Use **JenkinsPipelineUnit** (Groovy, run with Gradle or Maven). It loads library scripts, mocks steps (`helper.registerAllowedMethod('sh', [Map]) { ... }`) and records the call stack so you can assert what commands would run.

```groovy
class TrivyScanTest extends BasePipelineTest {
  @Test void failsOnHighByDefault() {
    def script = loadScript('vars/trivyScan.groovy')
    script.call(input: 'image.tar')
    assert helper.callStack.find { it.methodName == 'sh' }.argsToString().contains('--severity HIGH,CRITICAL')
  }
}
```

Integration test new versions with a test Jenkinsfile on a staging controller before moving the `v1` tag.

</details>

### Q92. What is a `.txt` file in `vars/` for? 🟢

<details>
<summary><b>Answer</b></summary>

`vars/kanikoBuild.txt` (HTML or Markdown depending on the markup formatter) is the documentation shown in **Pipeline Syntax > Global Variables Reference** for jobs that load the library. Document required arguments, defaults and an example. It is the closest thing to API docs your developers will read.

</details>

### Q93. How should a global step accept arguments? 🟡

<details>
<summary><b>Answer</b></summary>

Use a `Map` with defaults and validate required keys, which keeps calls readable and backward compatible.

```groovy
def call(Map args = [:]) {
  def cfg = [namespace: 'default', timeout: '120s'] + args
  if (!cfg.deployment) { error "k8sDeploy: 'deployment' is required" }
  sh "kubectl -n ${cfg.namespace} rollout status deploy/${cfg.deployment} --timeout=${cfg.timeout}"
}
```

A closure argument (`def call(Map cfg, Closure body)`) lets you write wrapper steps such as `withVault(path: 'x') { ... }`.

</details>

### Q94. Why must classes in `src/` implement `Serializable`? 🟡

<details>
<summary><b>Answer</b></summary>

Library classes run under CPS like the Jenkinsfile. If an instance is alive when a step executes (for example a `Deployer` object held in a variable while `sh` runs), CPS serializes it. A class without `Serializable` causes `NotSerializableException`. Fields that hold non-serializable objects must be `transient` or avoided.

</details>

### Q95. How does library caching work and why might it matter? 🔴

<details>
<summary><b>Answer</b></summary>

By default Jenkins fetches the library from SCM on every build. Large orgs with hundreds of builds per hour can hammer Git and slow build start. The global library configuration offers **Cache fetched versions on controller for quick retrieval** with a refresh time in minutes and an exclusion list (exclude moving branches like `main`). Cache tags, not branches, or you will run stale code.

</details>

### Q96. What is the difference between a step and a global variable with properties? 🟡

<details>
<summary><b>Answer</b></summary>

A `vars/` file without `call()` acts as a namespace object: `vars/cfg.groovy` with `def registry() { 'docker.io/devopsshack' }` is used as `cfg.registry()`. Global variables cannot hold state reliably between calls, because each access may get a new instance; do not use fields to store build data. Use `env` or return values instead.

</details>

### Q97. How do you keep `sh` commands safe from injection inside library steps? 🔴

<details>
<summary><b>Answer</b></summary>

Arguments interpolated into `sh` strings can come from branch names, PR titles or parameters, which attackers control. Pass values through the environment instead of interpolating, and quote them in the shell.

```groovy
withEnv(["IMAGE_REF=${cfg.image}"]) {
  sh 'crane push image.tar "$IMAGE_REF"'
}
```

Validate inputs with a regex (`cfg.image ==~ /^[a-z0-9.\/:_-]+$/`) before use.

</details>

### Q98. What does `resolve` or `library` returning an object let you do? 🔴

<details>
<summary><b>Answer</b></summary>

The `library` step returns an object through which you can reach `src/` classes dynamically without imports:

```groovy
def lib = library('shack-lib@v1')
def tag = lib.com.devopsshack.ImageTag.forBranch(env.BRANCH_NAME, env.GIT_COMMIT)
```

This is useful when the library version is chosen at runtime (`library "shack-lib@${params.LIB_VERSION}"`), since `@Library` annotations are resolved at compile time and cannot use variables.

</details>

### Q99. How do you roll out a breaking change in a shared library used by 300 pipelines? 🔴

<details>
<summary><b>Answer</b></summary>

1. Ship the new behaviour behind a new step name or an opt-in argument, keeping the old path working.
2. Release as a new major (`v2.0.0`, moving `v2` tag) while `v1` stays frozen except for fixes.
3. Add a deprecation `echo` or `unstable` in `v1` for the old usage.
4. Migrate pipelines in batches through automated PRs (Renovate can bump `@Library` versions).
5. Track usage by logging library version per build, then retire `v1`.

</details>

### Q100. What is `@Grab` and should you use it in libraries? 🔴

<details>
<summary><b>Answer</b></summary>

`@Grab` downloads Maven dependencies into the controller's classpath at runtime. It only works in trusted libraries, runs on the controller, pulls arbitrary jars from the internet and can conflict with plugin classes. Avoid it. If you need a tool (a YAML library, an HTTP client), run it in the build container with `sh` or use an existing plugin step such as `readYaml` or `httpRequest`.

</details>

---

## Multibranch, organization folders & webhooks

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. How does a multibranch pipeline work? 🟢

<details>
<summary><b>Answer</b></summary>

A multibranch job points at one repository through a **branch source** (GitHub, Bitbucket, GitLab, plain Git). During **branch indexing** it lists branches, PRs and tags, checks each for the configured script path (default `Jenkinsfile`), and creates a child job for every head that has one. New commits trigger builds of the matching child. When a branch is deleted, its job is removed according to the **orphaned item strategy**.

</details>

### Q102. What is branch indexing and what triggers it? 🟢

<details>
<summary><b>Answer</b></summary>

Indexing is the scan of the repository's heads. It runs when you click **Scan Repository Now**, on the periodic interval (`Scan Multibranch Pipeline Triggers`, e.g. 1 day), and on SCM events delivered by webhooks. The log is under **Scan Repository Log**. Indexing of an organization folder (**Scan Organization**) finds repositories and creates or removes multibranch jobs.

</details>

### Q103. What are the discovery traits in the GitHub branch source? 🟡

<details>
<summary><b>Answer</b></summary>

| Trait | Options |
|-------|---------|
| Discover branches | Exclude branches also filed as PRs (avoids double builds), only branches filed as PRs, all branches |
| Discover pull requests from origin | Merge with target (tests the merge result), the PR head, or both |
| Discover pull requests from forks | Same strategies plus a trust policy: Nobody, Collaborators, Everyone, From users with Admin or Write permission |
| Discover tags | Build tags (tag jobs are not built automatically unless a build strategy says so) |

Plus filters (`Filter by name (with wildcards)` or regex), clone options and status check behaviour.

</details>

### Q104. What does the fork PR trust setting protect against? 🔴

<details>
<summary><b>Answer</b></summary>

For PRs from forks, the **Jenkinsfile itself** can be modified by an outsider. If untrusted, Jenkins builds the fork's code but uses the **target branch's Jenkinsfile**, so an attacker cannot change the pipeline to dump credentials. The default `From users with Admin or Write permission` trusts collaborators only. Additionally, never expose deploy credentials to PR builds: scope them so only `main` can use them, or gate with `when { not { changeRequest() } }`.

</details>

### Q105. What environment variables are set for PR builds? 🟢

<details>
<summary><b>Answer</b></summary>

`BRANCH_NAME` is `PR-123`. `CHANGE_ID=123`, `CHANGE_URL`, `CHANGE_TITLE`, `CHANGE_AUTHOR`, `CHANGE_AUTHOR_EMAIL`, `CHANGE_TARGET` (e.g. `main`), `CHANGE_BRANCH` (source branch), `CHANGE_FORK` (if from a fork). Use `when { changeRequest() }` or `when { changeRequest target: 'main' }` to run PR-only stages.

</details>

### Q106. What is an organization folder? 🟡

<details>
<summary><b>Answer</b></summary>

A folder bound to a GitHub organization, Bitbucket team/project or GitLab group. It scans every repository, and for each one containing a `Jenkinsfile` (or a configured marker) it creates a multibranch job automatically. New repos appear without any Jenkins change, and one org-level webhook covers all repos. Use repo name filters to limit scope, and folder-scoped credentials for checkout.

</details>

### Q107. How does a GitHub webhook reach Jenkins and what URL do you configure? 🟢

<details>
<summary><b>Answer</b></summary>

With the GitHub plugin and GitHub Branch Source, configure the webhook in the repo or org as:

- Payload URL: `https://jenkins.example.com/github-webhook/` (the trailing slash matters)
- Content type: `application/json`
- Events: push, pull requests (and optionally "Send me everything")
- Secret: shared secret configured in Jenkins under the GitHub server's **Shared secrets** for HMAC validation

Jenkins receives the event, matches it to multibranch jobs for that repo, re-indexes the affected head and triggers the build. GitHub's **Recent Deliveries** shows the response code.

</details>

### Q108. GitHub App vs personal access token for GitHub Branch Source: which and why? 🟡

<details>
<summary><b>Answer</b></summary>

Use a **GitHub App** credential. It authenticates as the app, not a person, so builds do not break when an employee leaves; permissions are fine-grained per repository; rate limits are higher (scale with repos, up to 15,000 requests/hour for Enterprise Cloud orgs vs 5,000 for a PAT) and short-lived installation tokens are generated per use. Configure it as a "GitHub App" credential with the App ID and private key (PKCS#8). A PAT is acceptable only for small setups.

</details>

### Q109. What are build strategies in multibranch and why are tags not built by default? 🔴

<details>
<summary><b>Answer</b></summary>

When a multibranch job is first created or re-indexed, it would otherwise build every discovered head, including years of old tags. So tags are discovered but not built automatically. The **Basic Branch Build Strategies** plugin adds strategies: build regular branches, build change requests, build tags created in the last N days, skip initial build on first indexing. A common setup: build branches and PRs, and build tags created within the last 1 day, so pushing `v1.2.0` triggers a release build.

</details>

### Q110. What is the orphaned item strategy? 🟢

<details>
<summary><b>Answer</b></summary>

It decides what happens to child jobs whose branch or PR disappeared. Options: discard old items (with days to keep and max number to keep), or keep forever. Without it, deleted branches pile up as disabled jobs with build history consuming disk. A typical setting: discard after 7 days, keep max 20. Remember that discarding deletes the job's build records, including artifacts for that branch.

</details>

### Q111. How do you use a non-default Jenkinsfile path in a multibranch job? 🟢

<details>
<summary><b>Answer</b></summary>

Set **Build Configuration > Mode: by Jenkinsfile > Script Path** to, for example, `ci/Jenkinsfile`. For monorepos with several pipelines, create one multibranch job per service pointing to `services/api/Jenkinsfile`, and filter builds with `when { changeset 'services/api/**' }`, or use a plugin like "Multibranch Pipeline Inline Definition" to define the pipeline centrally.

</details>

### Q112. What is "lightweight checkout" in pipeline-from-SCM jobs? 🟡

<details>
<summary><b>Answer</b></summary>

To read the Jenkinsfile, Jenkins can fetch just that one file via the SCM API (GitHub contents API, `git archive`) instead of cloning the whole repo onto the controller. It is faster and keeps the controller clean. If unsupported (for example, parameters like `${BRANCH}` in the branch spec), you see `Lightweight checkout support not available, falling back to full checkout.` and a full clone happens in the controller's `workspace@script` directory.

</details>

### Q113. How does Jenkins report build status back to GitHub? 🟡

<details>
<summary><b>Answer</b></summary>

GitHub Branch Source automatically sets a commit status (or a Checks run with the GitHub App and the GitHub Checks plugin) named `continuous-integration/jenkins/branch` or `.../pr-merge`, updated when the build starts and finishes. Branch protection then requires that check. You can add custom statuses with `githubNotify` or publish per-stage checks with `publishChecks`. Statuses fail silently if the credential lacks `repo:status` permission or the App lacks "Commit statuses: write".

</details>

### Q114. How do you avoid building the same commit twice (branch and PR)? 🟢

<details>
<summary><b>Answer</b></summary>

In the branch discovery trait choose **Exclude branches that are also filed as PRs**. Then a feature branch with an open PR is built only as `PR-123`, while `main` and branches without PRs build as branches. Choose PR strategy **Merging the pull request with the current target branch revision** so you test what will actually land.

</details>

### Q115. What is the difference between SCM polling and webhooks in multibranch? 🟢

<details>
<summary><b>Answer</b></summary>

Polling makes Jenkins ask the Git server periodically; with many repos it burns API quota and adds latency. Webhooks push events to Jenkins instantly. In multibranch, keep a periodic scan (e.g. daily) as a safety net for missed webhooks, and rely on webhooks for real-time builds. If Jenkins is not reachable from the internet, use a relay (Smee, a webhook proxy) or an internal Git server.

</details>

### Q116. How do you skip CI for a commit in a multibranch pipeline? 🟡

<details>
<summary><b>Answer</b></summary>

Jenkins has no built-in `[skip ci]`. Options: the **SCM Skip** plugin (`scmSkip(deleteBuild: true, skipPattern: '.*\\[skip ci\\].*')`), the GitHub Branch Source trait from the "SCM Filter Commit Message" plugin, or a manual check of `git log -1 --pretty=%B` that calls `currentBuild.result = 'NOT_BUILT'` and returns. Be careful: allowing skip on `main` can let unbuilt code reach deploy pipelines.

</details>

### Q117. What does `checkout scm` actually check out in a multibranch job? 🔴

<details>
<summary><b>Answer</b></summary>

`scm` is the SCM object for the exact revision that triggered the build, as resolved during indexing. For a branch, it is the branch head commit; for a PR with merge strategy, Jenkins fetches the PR head and merges it with the target branch locally. This guarantees the build uses the same revision whose Jenkinsfile was loaded, even if new commits arrive meanwhile.

</details>

### Q118. How do you handle a monorepo where each service should build only when it changes? 🔴

<details>
<summary><b>Answer</b></summary>

Options:

1. One multibranch job per service with its own script path and `when { changeset 'services/api/**' }` on every stage (simple, but every push indexes every job).
2. One pipeline that computes changed paths (`git diff --name-only origin/main...HEAD`) and builds a dynamic `parallel` map for affected services.
3. A build tool that understands the graph (Nx, Bazel, Gradle) invoked once.

Note that `changeset` compares against the previous build of the branch, so the first build of a new branch sees no changeset and may skip everything; handle that case explicitly.

</details>

### Q119. What happens when a Jenkinsfile is removed from a branch? 🟢

<details>
<summary><b>Answer</b></summary>

On the next indexing, the branch no longer meets the criteria (log line: `‘Jenkinsfile’ not found` and `Does not meet criteria`), so the branch job is treated as orphaned and removed or disabled according to the orphaned item strategy. If you only want to stop builds temporarily, disable the branch job or add a condition rather than deleting the file.

</details>

### Q120. How do you secure the webhook endpoint? 🔴

<details>
<summary><b>Answer</b></summary>

1. Configure a shared secret so Jenkins validates the `X-Hub-Signature-256` HMAC on each delivery.
2. Expose only `/github-webhook/` publicly through the reverse proxy or a dedicated ingress path; keep the UI behind VPN/SSO.
3. Allowlist GitHub's webhook IP ranges (from `https://api.github.com/meta`) at the load balancer.
4. Remember the webhook only triggers indexing; Jenkins then fetches from GitHub with its own credentials, so a forged payload cannot inject code.

</details>

---

## Plugins, JCasC & Jenkins on Kubernetes

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q121. How do you manage plugins in a reproducible way? 🟡

<details>
<summary><b>Answer</b></summary>

Pin plugins in `plugins.txt` and install them at image build time with `jenkins-plugin-cli`; never click "Install" on production.

```text
# one plugin per line, name:version (exact versions come from the update center)
configuration-as-code:<pinned-version>
kubernetes:<pinned-version>
workflow-aggregator:<pinned-version>
```

```dockerfile
FROM jenkins/jenkins:2.479.3-lts-jdk21
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt
```

Use `jenkins-plugin-cli --available-updates --output txt` to produce an updated list for a PR.

</details>

### Q122. What does `jenkins-plugin-cli` resolve for you, and what flags matter? 🟡

<details>
<summary><b>Answer</b></summary>

It downloads plugins and their **transitive dependencies**, checks minimum core versions, and fails on conflicts. Useful flags:

```bash
jenkins-plugin-cli --plugin-file plugins.txt --verbose
jenkins-plugin-cli --plugins kubernetes:latest git:latest --latest false
jenkins-plugin-cli --list --plugin-file plugins.txt
jenkins-plugin-cli --available-updates --output txt --plugin-file plugins.txt
jenkins-plugin-cli --view-security-warnings --plugin-file plugins.txt
```

`--jenkins-version` checks compatibility against a specific core version without a running instance.

</details>

### Q123. Why should you keep the number of plugins low? 🟢

<details>
<summary><b>Answer</b></summary>

Each plugin adds startup time, memory, upgrade risk (dependency conflicts after core upgrades) and attack surface (most Jenkins security advisories are in plugins). Abandoned plugins stop receiving fixes. We audit plugins yearly: remove unused ones (check usage with the Plugin Usage plugin), replace UI-heavy plugins with plain `sh` steps, and prefer plugins with recent releases and many installs.

</details>

### Q124. What is JCasC and how is it loaded? 🟢

<details>
<summary><b>Answer</b></summary>

The Configuration as Code plugin applies Jenkins configuration from YAML. It reads the path, directory or URL in `CASC_JENKINS_CONFIG` (default `$JENKINS_HOME/jenkins.yaml`) at startup. You can re-apply later from **Manage Jenkins > Configuration as Code > Reload existing configuration**, or with a POST to `/configuration-as-code/reload`. **View Configuration** exports the current config as YAML, a good starting point.

</details>

### Q125. What are the top-level root elements in a JCasC file? 🟡

<details>
<summary><b>Answer</b></summary>

| Root | Contains |
|------|----------|
| `jenkins` | Security realm, authorization, nodes, clouds, `numExecutors`, system message, global properties |
| `credentials` | System credentials by domain |
| `security` | Script approval, global job DSL security, `apiToken` settings, SSHD |
| `unclassified` | Most plugin globals: `location` (URL, admin email), `globalLibraries`, `sonarGlobalConfiguration`, GitHub servers |
| `tool` | JDK, Maven, Git, NodeJS installations |
| `jobs` | Job DSL scripts that seed jobs |

</details>

### Q126. How do you inject secrets into JCasC without committing them? 🔴

<details>
<summary><b>Answer</b></summary>

JCasC resolves `${VAR}` placeholders from, in order, secret sources: environment variables, files in `/run/secrets` (Docker/Kubernetes secrets, filename = variable name), and plugins for Vault, AWS Secrets Manager or Kubernetes. `${VAR:-default}` provides a default.

```yaml
credentials:
  system:
    domainCredentials:
      - credentials:
          - string:
              id: sonar-token
              scope: GLOBAL
              secret: "${SONAR_TOKEN}"
```

Use `^${VAR}` to escape a literal `${...}` that must not be resolved.

</details>

### Q127. How do you configure authorization with JCasC? 🟡

<details>
<summary><b>Answer</b></summary>

With matrix-auth 3.x:

```yaml
jenkins:
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: admin
          password: "${JENKINS_ADMIN_PASSWORD}"
  authorizationStrategy:
    globalMatrix:
      entries:
        - user:
            name: admin
            permissions: ["Overall/Administer"]
        - group:
            name: authenticated
            permissions: ["Overall/Read", "Job/Read", "Job/Build", "Job/Cancel"]
```

In real organizations the realm is LDAP, SAML or OIDC, with groups mapped from the identity provider.

</details>

### Q128. How do you create jobs from JCasC? 🟡

<details>
<summary><b>Answer</b></summary>

Install the Job DSL plugin and put DSL scripts under `jobs:`.

```yaml
jobs:
  - script: >
      folder('payments')
  - file: /var/jenkins_config/jobs/multibranch.groovy
```

Job DSL creates or updates folders, pipelines and multibranch jobs idempotently. Combined with JCasC, a brand-new controller comes up with every folder and multibranch job defined, and indexing recreates the branch jobs.

</details>

### Q129. Should you manage everything through JCasC, including jobs and credentials? 🔴

<details>
<summary><b>Answer</b></summary>

Configuration: yes. Jobs: yes for folders and multibranch/org folder definitions via Job DSL; branch jobs are generated by indexing anyway. Credentials: define them in JCasC but resolve values from a secret store, or better, use a credentials provider plugin (Kubernetes credentials provider, AWS Secrets Manager credentials provider, HashiCorp Vault) so secrets never touch the YAML. Avoid hybrid setups where some admins click in the UI, because the next restart or reload silently overwrites their changes.

</details>

### Q130. What does the Kubernetes plugin need to create agent pods? 🔴

<details>
<summary><b>Answer</b></summary>

1. A Kubernetes cloud configuration: API server URL (blank when the controller runs in-cluster), namespace, credentials (in-cluster service account by default), the Jenkins URL agents use to connect back, and either WebSocket mode or a tunnel to port 50000.
2. RBAC for the controller's service account in the agent namespace: `pods` (create, delete, get, list, watch), `pods/exec` (create, get), `pods/log` (get), `events` (get, list, watch), and `persistentvolumeclaims` if used.
3. Pod templates, defined in the cloud config or inline in pipelines.

</details>

### Q131. How do you define a pod template inline in a declarative pipeline? 🟡

<details>
<summary><b>Answer</b></summary>

```groovy
agent {
  kubernetes {
    defaultContainer 'maven'
    yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: maven
      image: maven:3.9-eclipse-temurin-21
      command: ["sleep"]
      args: ["infinity"]
      resources:
        requests: { cpu: "1", memory: "2Gi" }
        limits: { memory: "2Gi" }
'''
  }
}
```

Tool containers must stay alive (`sleep infinity` or `cat` with `tty: true`); the plugin adds the `jnlp` container automatically. `yamlFile 'ci/pod.yaml'` reads the template from the repo instead.

</details>

### Q132. How does the `container()` step work? 🟡

<details>
<summary><b>Answer</b></summary>

All containers in an agent pod share the workspace volume (`/home/jenkins/agent`). `container('kaniko') { sh '...' }` runs the shell step inside that container through the Kubernetes exec API, with the same working directory. So one stage can run `npm test` in `node`, then the build in `kaniko`, then the scan in `trivy`, all on the same files without stashing. The container image needs a shell (`/bin/sh`) for `sh` steps; distroless images do not have one, which is why tools ship `:debug` variants.

</details>

### Q133. What are pod template inheritance and `POD_LABEL`? 🔴

<details>
<summary><b>Answer</b></summary>

Templates can inherit from others with `inheritFrom 'base'`, merging containers, volumes and YAML (`yamlMergeStrategy merge()`), so a common base defines the `jnlp` image, node selector and tolerations. In scripted pipelines, `podTemplate { node(POD_LABEL) { } }` uses the automatically generated unique label for the template just defined:

```groovy
podTemplate(inheritFrom: 'base', containers: [containerTemplate(name: 'go', image: 'golang:1.23', command: 'sleep', args: 'infinity')]) {
  node(POD_LABEL) { container('go') { sh 'go test ./...' } }
}
```

</details>

### Q134. WebSocket vs TCP port 50000 for Kubernetes agents: which do you use? 🟡

<details>
<summary><b>Answer</b></summary>

WebSocket (`webSocket: true` in the cloud config, or `-webSocket` for `agent.jar`). Agents connect over the same HTTP(S) port as the UI, so you need only one Service port and it works through ingress controllers, load balancers and proxies. TCP 50000 requires an extra Service port and a separate path through firewalls. Inside one cluster both work; for agents in other clusters or networks WebSocket is much simpler.

</details>

### Q135. How do you cache dependencies with ephemeral Kubernetes agents? 🔴

<details>
<summary><b>Answer</b></summary>

Every pod starts empty, so `~/.m2` or `node_modules` downloads repeat. Options:

1. A repository manager proxy (Nexus/Artifactory) close to the cluster: the biggest win, and it also protects against upstream outages.
2. A shared ReadWriteMany PVC mounted at the cache path (watch concurrency and corruption).
3. Per-node `hostPath` caches (fast but node-specific and a security concern).
4. Tool-level remote caches: Gradle build cache, BuildKit/Kaniko registry cache (`--cache=true --cache-repo`).

</details>

### Q136. How do Kubernetes agent pods get permissions to deploy? 🟡

<details>
<summary><b>Answer</b></summary>

Set `serviceAccountName` in the deploy pod template and bind that service account with a RoleBinding in the target namespaces. `kubectl` in the pod uses the mounted token automatically. Use a separate service account per environment (`deployer-dev`, `deployer-prod`) and only use the prod template in stages gated by `when { branch 'main' }` and `input`. For cloud APIs use workload identity (IRSA, EKS Pod Identity, GKE Workload Identity) instead of static keys.

</details>

### Q137. Where should agent pods run relative to the controller? 🔴

<details>
<summary><b>Answer</b></summary>

In a separate namespace (e.g. `jenkins-agents`) with its own ResourceQuota, LimitRange and NetworkPolicies, ideally on a dedicated node pool (taints and tolerations) so builds cannot starve the controller or production workloads. For strong isolation (untrusted PR builds, privileged builders), use a separate cluster. Never run build pods in the same namespace as production applications, because a compromised build could reach their secrets.

</details>

### Q138. What does `podRetention` and `idleMinutes` control? 🟡

<details>
<summary><b>Answer</b></summary>

`podRetention` decides whether pods are deleted after the build: `never` (default, delete), `onFailure` (keep failed pods for debugging), or `always`. `idleMinutes` keeps a pod alive for reuse by later builds with the same label, trading isolation for speed. In production we delete pods and only enable `onFailure` temporarily while debugging, because retained pods consume quota.

</details>

### Q139. How do you limit the number of concurrent agent pods? 🟢

<details>
<summary><b>Answer</b></summary>

Set **Concurrency Limit** on the cloud (`containerCapStr` in JCasC) and per pod template (`instanceCap`). Back it with a Kubernetes `ResourceQuota` on the agent namespace so the cluster enforces limits even if Jenkins misbehaves. Excess builds wait in the Jenkins queue rather than flooding the scheduler with pending pods.

</details>

### Q140. How do you build container images inside Kubernetes agents without Docker? 🔴

<details>
<summary><b>Answer</b></summary>

Mounting the node's Docker socket gives every build root on the node and does not work on containerd-only clusters. Daemonless options:

- **Kaniko** (`executor:debug` image): builds from a Dockerfile in user space; the original Google project was archived in 2025, maintained forks exist.
- **BuildKit** rootless (`moby/buildkit:rootless` with `buildctl-daemonless.sh`), or a shared BuildKit deployment.
- **Buildah** in rootless mode.

Each still runs as a privileged-ish workload in some form, so isolate builder pods on dedicated nodes.

</details>

### Q141. What is the Docker Pipeline plugin and how is `docker.image().inside()` used? 🟡

<details>
<summary><b>Answer</b></summary>

On agents with a Docker daemon, the Docker Pipeline plugin provides a Groovy API:

```groovy
docker.image('node:22-alpine').inside('-v npm-cache:/root/.npm') {
  sh 'npm ci && npm test'
}
docker.withRegistry('https://registry.example.com', 'registry-creds') {
  def img = docker.build("shack-api:${env.GIT_COMMIT.take(7)}")
  img.push()
  img.push('latest')
}
```

`inside` mounts the workspace and runs the container with the agent's UID. It needs a Docker daemon on the agent, so it does not fit containerd-only Kubernetes agents.

</details>

### Q142. What are Configuration Slicing and other JCasC alternatives you might meet in old setups? 🟢

<details>
<summary><b>Answer</b></summary>

Older controllers often use: Groovy init scripts in `init.groovy.d/` (run at every startup, imperative, hard to review), the Configuration Slicing plugin, or hand-edited XML copied between servers. JCasC replaced these. Init scripts remain useful for the rare settings JCasC cannot express, but each one should be idempotent and in Git.

</details>

### Q143. How do you test a JCasC change before applying it to production? 🔴

<details>
<summary><b>Answer</b></summary>

1. Build the controller image with the change and start it locally or in a staging namespace with the same JCasC and dummy secrets.
2. Check startup logs for `ConfiguratorException` and unknown attributes.
3. Use the JCasC **schema** (`/configuration-as-code/schema`) for editor validation.
4. Run a smoke pipeline that exercises agents, credentials and libraries.
5. Merge, then roll the production controller (restart or reload).

Some teams run a CI job that starts Jenkins in a container with the new YAML and fails if the log contains `ConfiguratorException`.

</details>

### Q144. What does reload do to configuration not described in YAML? 🔴

<details>
<summary><b>Answer</b></summary>

JCasC applies what the YAML describes. For lists it typically replaces the whole list (for example, all clouds or all global libraries), so a cloud added manually in the UI disappears on reload. Settings not mentioned at all are left as they are. This partial ownership is why mixed manual and code configuration causes surprises; treat the YAML as the full truth for each section it touches.

</details>

### Q145. How do you install Jenkins on Kubernetes in production? 🟡

<details>
<summary><b>Answer</b></summary>

The official **Helm chart** (`jenkins/jenkins` from `https://charts.jenkins.io`) or the Jenkins Operator, or your own manifests. Essentials: a StatefulSet with one replica, a PersistentVolumeClaim for `JENKINS_HOME`, a custom controller image with pinned plugins, JCasC from a ConfigMap, secrets from a Secret or external store, a ServiceAccount with RBAC for agent pods, an Ingress with TLS, resource requests and limits, liveness and readiness probes on `/login`, and a backup job for the PVC.

```bash
helm repo add jenkins https://charts.jenkins.io && helm repo update
helm upgrade --install jenkins jenkins/jenkins -n jenkins --create-namespace -f values.yaml
```

</details>

---

## Credentials & security

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q146. What credential types does Jenkins support out of the box? 🟢

<details>
<summary><b>Answer</b></summary>

Username with password, Secret text, Secret file, SSH username with private key, Certificate (PKCS#12), plus plugin types: GitHub App, AWS credentials, Docker host certificate, Kubernetes service account, Vault tokens and more. Each has an **ID** that pipelines reference, a **scope** and a description. Pick meaningful IDs (`nexus-deployer`, `github-app-shack`) because they become part of every Jenkinsfile.

</details>

### Q147. What is the difference between Global and System credential scope? 🟡

<details>
<summary><b>Answer</b></summary>

**Global** credentials are available to jobs (pipelines can bind them) and to Jenkins itself. **System** credentials are only usable by Jenkins internals, such as the agent SSH launcher or the email server, and are never exposed to jobs. Use System scope for infrastructure credentials so no pipeline can read them.

</details>

### Q148. How do folder-scoped credentials help multi-team controllers? 🟡

<details>
<summary><b>Answer</b></summary>

Credentials stored in a folder's credential store are visible only to jobs inside that folder (and subfolders). Team Payments keeps `prod-deploy` in `/payments`, team Search cannot reference it even if they guess the ID. Combine with folder-level authorization so each team manages its own folder. This is the main multi-tenancy mechanism in a shared controller.

</details>

### Q149. How does Jenkins mask secrets in logs and what are its limits? 🔴

<details>
<summary><b>Answer</b></summary>

When a credential is bound, Jenkins replaces occurrences of its exact value in the console output with `****`. It does not mask transformed values: base64-encoded, URL-encoded, split across lines, or printed character by character. A malicious pipeline author can always exfiltrate a secret they can bind (for example by sending it to an external host). Masking prevents accidents, not attacks; access control on who can modify pipelines that use a credential is the real protection.

</details>

### Q150. What authorization strategies are available? 🟡

<details>
<summary><b>Answer</b></summary>

| Strategy | Use |
|----------|-----|
| Anyone can do anything | Never |
| Legacy mode | Never |
| Logged-in users can do anything | Labs only |
| Matrix-based security | Global permission grid per user/group |
| Project-based Matrix | Matrix plus per-folder/per-job grants |
| Role-Based Strategy (role-strategy plugin) | Global, item (regex on job names) and agent roles |

Production uses project-based matrix or role-based, with groups from SSO.

</details>

### Q151. What are the key permissions to understand in matrix auth? 🟡

<details>
<summary><b>Answer</b></summary>

`Overall/Administer` (everything, including running script console code), `Overall/Read` (needed to see anything), `Job/Read`, `Job/Build`, `Job/Cancel`, `Job/Configure` (can change the pipeline, so effectively can use its credentials), `Job/Workspace`, `Run/Replay` (run modified pipeline code), `Run/Update`, `Credentials/View`, `Credentials/Create`, `Agent/Connect`, `SCM/Tag`. Treat `Job/Configure`, `Run/Replay` and agent connect permissions as sensitive.

</details>

### Q152. What is the Groovy sandbox? 🟡

<details>
<summary><b>Answer</b></summary>

Pipeline scripts from Jenkinsfiles and untrusted libraries run in the Script Security sandbox, which allows only allowlisted method calls. Anything else throws `RejectedAccessException: Scripts not permitted to use method ...` and queues a signature for admin approval under **Manage Jenkins > In-process Script Approval**. The sandbox stops Jenkinsfile authors from calling `Jenkins.instance`, reading files on the controller or executing processes on it.

</details>

### Q153. How should admins handle script approval requests? 🔴

<details>
<summary><b>Answer</b></summary>

Approve only signatures that cannot escalate privileges: harmless string, collection or date methods. Never approve signatures that touch `jenkins.model.Jenkins`, `hudson.model.*` internals, `getRawBuild`, `execute`, file I/O on the controller, reflection or class loading. The right fix is usually to move the logic into a reviewed trusted shared library step, which does not need approvals. Periodically review the approved list (it is stored in `scriptApproval.xml`) and clear unnecessary entries.

</details>

### Q154. What is the CSRF crumb and when do you need it? 🟡

<details>
<summary><b>Answer</b></summary>

Jenkins protects state-changing POST requests with a crumb tied to the user session. Browsers handle it automatically. Scripts using **username + API token** do not need a crumb (since 2.176.2). Scripts using username + password and a session must first fetch a crumb:

```bash
CRUMB=$(curl -s -c jar -u alice:pass 'https://jenkins/crumbIssuer/api/json' | jq -r '.crumbRequestField + ":" + .crumb')
curl -s -b jar -u alice:pass -H "$CRUMB" -X POST https://jenkins/job/app/build
```

The error without it is `HTTP ERROR 403 No valid crumb was included in the request`. Never disable CSRF protection.

</details>

### Q155. What are API tokens and how do you manage them? 🟢

<details>
<summary><b>Answer</b></summary>

Per-user tokens generated in **User > Security > API Token**, used instead of passwords for CLI, REST and scripts. They inherit the user's permissions, can be named and revoked individually, and their last use is tracked. Disable legacy tokens (`security: apiToken: creationOfLegacyTokenEnabled: false`). For automation, create a dedicated service user with minimal permissions rather than using an admin's token.

</details>

### Q156. What is the script console and why is it the most dangerous page in Jenkins? 🟡

<details>
<summary><b>Answer</b></summary>

`/script` runs arbitrary Groovy on the controller with full privileges. It can decrypt every credential (`hudson.util.Secret.decrypt`), change config, delete jobs and run OS commands. It is restricted to `Overall/Administer`. Keep the number of admins tiny, require SSO with MFA, and audit with the Audit Trail plugin. Agents also have a script console (`/computer/<name>/script`).

</details>

### Q157. How do you integrate Jenkins with HashiCorp Vault or a cloud secret manager? 🔴

<details>
<summary><b>Answer</b></summary>

Two patterns:

1. **Credentials provider**: plugins such as AWS Secrets Manager Credentials Provider or Kubernetes Credentials Provider expose external secrets as normal Jenkins credentials (read-only, never stored in `JENKINS_HOME`).
2. **Runtime fetch**: the HashiCorp Vault plugin's `withVault` step reads secrets during the build, authenticating with AppRole or Kubernetes auth.

```groovy
withVault(configuration: [vaultUrl: 'https://vault:8200', vaultCredentialId: 'vault-approle'],
          vaultSecrets: [[path: 'secret/shack/prod', secretValues: [[envVar: 'DB_PASS', vaultKey: 'db_password']]]]) {
  sh './migrate.sh'
}
```

</details>

### Q158. How do you authenticate users through SSO? 🟡

<details>
<summary><b>Answer</b></summary>

Use a security realm plugin: **OpenID Connect Authentication** (Okta, Entra ID, Keycloak, Google), **SAML**, **LDAP/Active Directory**, or **GitHub Authentication**. Map IdP groups to Jenkins groups and grant permissions to groups only. Keep one local break-glass admin account with a vaulted password for when SSO is down, and test that it works.

</details>

### Q159. What is agent-to-controller security? 🔴

<details>
<summary><b>Answer</b></summary>

Agents are considered less trusted than the controller. Since Jenkins 2.326 the agent-to-controller access control is always on: an agent can only invoke allowlisted callables on the controller and cannot read or write arbitrary files in `JENKINS_HOME`. This means a compromised build agent cannot directly take over the controller. It is another reason to run zero builds on the built-in node, where this boundary does not exist.

</details>

### Q160. Why is running builds as root, or with the Docker socket mounted, a Jenkins security problem? 🟡

<details>
<summary><b>Answer</b></summary>

Builds run untrusted-ish code: dependencies, test suites, PR changes. With the host Docker socket, any build can start a privileged container that mounts the host filesystem, read other builds' workspaces and credentials, and pivot to the node. On Kubernetes, prefer rootless builders, `runAsNonRoot` tool containers, dedicated builder nodes, NetworkPolicies, and separate agents for trusted (main branch, deploy) vs untrusted (PRs) builds.

</details>

### Q161. What does the Markup Formatter setting protect? 🟢

<details>
<summary><b>Answer</b></summary>

Job descriptions, build descriptions and parameter descriptions are rendered as HTML. With **Plain text** or **Safe HTML** (OWASP Markup Formatter), users cannot inject `<script>` that would run in an admin's browser (stored XSS leading to admin takeover). Never select "Raw HTML" on a controller where non-admins can edit descriptions.

</details>

### Q162. How do you audit who did what in Jenkins? 🟡

<details>
<summary><b>Answer</b></summary>

- **Audit Trail** plugin: logs configuration changes, builds started and logins to a file or syslog.
- **Job Configuration History** plugin: diffs of job and system config changes.
- Build causes (`Started by user alice`) and `input` approver names (`submitterParameter`).
- Reverse proxy and SSO logs for access.

Ship these logs to your central logging system and alert on admin actions and script console use.

</details>

### Q163. How do you restrict which jobs can run on which agents? 🔴

<details>
<summary><b>Answer</b></summary>

Labels plus `EXCLUSIVE` node mode prevent accidental placement, but a pipeline author can always request any label. To enforce it, use the **Job Restrictions** plugin or **Authorize Project** plugin with agent permissions (`Agent/Build`), so that only jobs in `/payments/prod-deploy` can run on the agent holding prod network access. With Kubernetes, enforce at the cluster level: deploy templates use service accounts that only exist in a restricted namespace.

</details>

### Q164. What does the Authorize Project plugin change? 🔴

<details>
<summary><b>Answer</b></summary>

By default builds run as the internal `SYSTEM` user, which can access everything. Authorize Project makes builds run as a specific identity: the user who triggered it, the user who last configured the job, or an anonymous identity. Permission checks (triggering downstream jobs, using agents, accessing credentials with `Credentials/UseItem`) then apply to builds too, closing gaps where a low-privileged user's job could trigger privileged jobs.

</details>

### Q165. How do you rotate a credential used by many pipelines? 🟢

<details>
<summary><b>Answer</b></summary>

Update the value in place (same ID) so pipelines pick it up on their next run; running builds keep the old value until they finish. With JCasC, change the secret source and reload. With an external provider, rotate in Vault or Secrets Manager and Jenkins reads the new value on next lookup. Never create a new ID per rotation unless you also need to support both values during a migration.

</details>

---

## Backups, HA, upgrades & performance

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q166. What do you back up in Jenkins and what do you exclude? 🟡

<details>
<summary><b>Answer</b></summary>

Back up all of `JENKINS_HOME` except caches and rebuildable data:

| Include | Exclude |
|---------|---------|
| `config.xml`, `*.xml` at root | `workspace/` |
| `jobs/` (config and `builds/`) | `caches/` |
| `secrets/`, `credentials.xml` | `plugins/*/` exploded dirs (keep `.jpi` or rebuild from image) |
| `users/`, `nodes/` | `war/`, `logs/` optional |
| `userContent/`, `fingerprints/` | `.cache/`, `tools/` |

With config in Git (image + JCasC), the backup's real value is build history, credentials not in an external store, and `secrets/`.

</details>

### Q167. How do you back up Jenkins running on Kubernetes? 🟡

<details>
<summary><b>Answer</b></summary>

Snapshot the `JENKINS_HOME` PersistentVolume with VolumeSnapshots (CSI) or Velero on a schedule, and/or run a CronJob that `tar`s the important paths (excluding workspaces) and uploads to S3 with versioning and object lock. Practise restores: restore the snapshot into a new PVC, start a controller from the same image and JCasC, check credentials decrypt and build history appears.

</details>

### Q168. Does Jenkins support active-active high availability? 🔴

<details>
<summary><b>Answer</b></summary>

Open-source Jenkins does not: two controllers cannot share one `JENKINS_HOME` safely because state is held in memory and written to files. Practical approaches:

- **Fast recovery**: controller in a Kubernetes StatefulSet with a persistent volume; if the node dies, the pod reschedules in minutes. Pipelines resume if agents survived.
- **Warm standby**: replicate `JENKINS_HOME` to a second site and fail over with DNS.
- **Horizontal split**: multiple controllers by team or domain, reducing blast radius.
- CloudBees CI offers commercial HA (active-active) and controller management.

</details>

### Q169. How do you plan a Jenkins LTS upgrade? 🟡

<details>
<summary><b>Answer</b></summary>

1. Read the LTS upgrade guide for every version you cross (Java requirement changes, removed features).
2. Build a new controller image with the new core and updated `plugins.txt` (`jenkins-plugin-cli --jenkins-version <new> --available-updates`).
3. Restore a copy of production `JENKINS_HOME` into staging and boot the new image; check logs for plugin load failures and "old data" warnings.
4. Run smoke pipelines (agents, libraries, credentials, deploys).
5. Back up production, put it in quiet-down, swap the image, verify.
6. Keep the old image and the backup for rollback; plugin data migrations may not be reversible.

</details>

### Q170. What does "Manage Old Data" mean after an upgrade? 🟢

<details>
<summary><b>Answer</b></summary>

When a plugin is removed or its data format changes, configuration files may contain unreadable fields. Jenkins lists them under **Manage Jenkins > Manage Old Data** (also flagged in the admin monitor). Review them; if the plugin is intentionally gone, discard the old data so it does not cause warnings on every startup.

</details>

### Q171. How do you size and tune the controller JVM? 🔴

<details>
<summary><b>Answer</b></summary>

Set heap explicitly (`-Xms` = `-Xmx`, or `-XX:MaxRAMPercentage=75` in containers), use G1GC (default on JDK 17+), enable GC logging and heap dumps on OOM:

```bash
JAVA_OPTS="-XX:MaxRAMPercentage=75 -XX:+UseG1GC -XX:+UseStringDeduplication \
  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/jenkins_home/heapdumps \
  -Xlog:gc*:file=/var/jenkins_home/logs/gc.log:time,uptime:filecount=5,filesize=20m"
```

Typical controllers need 4–16 GB heap depending on job count, build retention and concurrent pipelines. Growing beyond that usually signals too much on one controller.

</details>

### Q172. What are the most common causes of a slow Jenkins controller? 🟡

<details>
<summary><b>Answer</b></summary>

- Too many builds retained (no `buildDiscarder`), making job loading and UI slow
- Heavy Groovy in pipelines (CPS on the controller), especially JSON parsing and big loops
- Builds running on the built-in node
- Slow `JENKINS_HOME` storage (NFS with high latency, burstable EBS volumes out of credits)
- Large console logs streamed through the controller
- Too many plugins, or a misbehaving plugin
- Polling hundreds of repos instead of webhooks
- Undersized heap causing constant GC

</details>

### Q173. How do you diagnose a hung or slow controller? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
# thread dump from inside the container
jcmd $(pgrep -f jenkins.war) Thread.print > /tmp/threads.txt
# or in the UI: /threadDump
# GC behaviour
jstat -gcutil $(pgrep -f jenkins.war) 5s
# heap histogram
jcmd $(pgrep -f jenkins.war) GC.class_histogram | head -30
```

Look for many threads blocked on the same lock, `Executor` threads stuck in SCM calls, long GC pauses. The **Support Core** plugin bundles thread dumps, logs, config and metrics in one zip for analysis. Also check disk latency (`iostat -x`) on the `JENKINS_HOME` volume.

</details>

### Q174. How do you monitor Jenkins? 🟡

<details>
<summary><b>Answer</b></summary>

Install the **Prometheus metrics** plugin (exposes `/prometheus/`) or OpenTelemetry plugin. Key metrics: queue size and wait time, busy vs idle executors, number of online agents, build duration and failure rate per job, JVM heap and GC time, HTTP request latency, disk free on `JENKINS_HOME`. Alert on queue length above threshold for 10 minutes, heap above 85%, disk below 15%, and agents offline. The OpenTelemetry plugin can also trace pipelines as spans.

</details>

### Q175. What is the impact of log size and how do you control it? 🟢

<details>
<summary><b>Answer</b></summary>

Console logs are stored on the controller and streamed through it. A test suite printing 2 GB of debug output slows the controller and fills the disk. Control it with quiet build tool flags (`mvn -B -q`, `--silent`), writing verbose output to files that you archive compressed, `buildDiscarder` to delete old logs, and external log storage via plugins (e.g. pipeline-cloudwatch-logs, or the OpenTelemetry plugin with a log backend).

</details>

### Q176. When should you split one controller into several? 🔴

<details>
<summary><b>Answer</b></summary>

Signals: thousands of jobs, controller heap above 16 GB, restarts affecting many unrelated teams, plugin requirements that conflict between teams, or security boundaries (regulated workloads). Split by business domain or team, each controller fully defined by image + JCasC, sharing the same libraries and agent clusters. Operation tools: Helm with per-controller values, or CloudBees Operations Center commercially.

</details>

### Q177. How do you move Jenkins to a new server or cluster? 🟡

<details>
<summary><b>Answer</b></summary>

1. Put the old controller in quiet-down and wait for builds.
2. Stop it and copy `JENKINS_HOME` (rsync, excluding workspaces) or restore from backup into the new volume.
3. Start the new controller with the same core and plugin versions (or the planned upgrade).
4. Update the Jenkins URL in `unclassified.location.url`, DNS, webhooks and agent connection URLs.
5. Verify credentials decrypt (this proves `secrets/` came along), agents connect, and a test pipeline runs.

</details>

### Q178. What is the impact of `buildDiscarder` on disk and performance? 🟢

<details>
<summary><b>Answer</b></summary>

Each retained build keeps a directory with its log, metadata, test results and artifacts. Jobs with 10,000 builds take longer to load and fill the disk. `buildDiscarder(logRotator(numToKeepStr: '30', daysToKeepStr: '30', artifactNumToKeepStr: '5'))` keeps history manageable. Set a global default with the Discard Old Build plugin or enforce it in your shared library's pipeline template.

</details>

### Q179. How do you keep the controller stateless enough to recreate quickly? 🔴

<details>
<summary><b>Answer</b></summary>

- Controller image with pinned core and plugins
- JCasC for all system config; Job DSL for folders and multibranch jobs
- Credentials in an external secret store via a credentials provider
- Artifacts in Nexus/Artifactory or S3 (Artifact Manager on S3), not in `JENKINS_HOME`
- Logs shipped externally
- Short build retention

Then recovery means "deploy a new controller and scan repositories"; only build history is lost if the volume is gone, which many teams accept.

</details>

### Q180. What is the Artifact Manager on S3 plugin for? 🟡

<details>
<summary><b>Answer</b></summary>

It makes `archiveArtifacts` and `stash` store files in S3 instead of `JENKINS_HOME`, uploading directly from the agent and serving downloads through presigned URLs. This removes large binaries from the controller disk and network path and makes the controller easier to back up and recreate.

</details>

---

## Integrations: SonarQube, Nexus, Artifactory, Docker, Trivy

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q181. How do you integrate SonarQube with a Jenkins pipeline? 🟡

<details>
<summary><b>Answer</b></summary>

Install the **SonarQube Scanner** plugin, configure the server (name, URL, token credential) under System or via JCasC (`unclassified.sonarGlobalConfiguration`), then:

```groovy
stage('SonarQube') {
  steps {
    withSonarQubeEnv('sonarqube') {
      sh 'sonar-scanner -Dsonar.projectKey=shack-app -Dsonar.sources=src -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info'
    }
  }
}
```

`withSonarQubeEnv` injects `SONAR_HOST_URL`, `SONAR_AUTH_TOKEN` and `SONARQUBE_SCANNER_PARAMS`, which the scanner reads automatically. For Maven use `mvn -B sonar:sonar`.

</details>

### Q182. How does `waitForQualityGate` work? 🟡

<details>
<summary><b>Answer</b></summary>

After analysis, the scanner writes `report-task.txt` with the task ID. `waitForQualityGate` reads it and pauses (without holding an executor if called outside an agent) until SonarQube calls back Jenkins' webhook `https://jenkins/sonarqube-webhook/` with the quality gate result.

```groovy
stage('Quality gate') {
  steps { timeout(time: 10, unit: 'MINUTES') { waitForQualityGate abortPipeline: true } }
}
```

Without the webhook configured in SonarQube (Administration > Configuration > Webhooks), the step waits until timeout. It must run in the same build after `withSonarQubeEnv`.

</details>

### Q183. How do you deploy Maven artifacts to Nexus from Jenkins? 🟡

<details>
<summary><b>Answer</b></summary>

Define repositories in `pom.xml` `distributionManagement` (releases and snapshots), and provide credentials in a `settings.xml` with matching `<server><id>`. Supply `settings.xml` from the Config File Provider plugin or generate it from credentials:

```groovy
withCredentials([usernamePassword(credentialsId: 'nexus-deployer', usernameVariable: 'NX_USER', passwordVariable: 'NX_PASS')]) {
  sh 'mvn -B -s ci/settings.xml deploy -DskipTests'
}
```

where `ci/settings.xml` uses `<username>${env.NX_USER}</username>` and `<password>${env.NX_PASS}</password>`. Releases repositories typically disallow redeploy, so re-running a release build fails by design.

</details>

### Q184. How do you publish npm packages or pull dependencies through Nexus or Artifactory? 🟢

<details>
<summary><b>Answer</b></summary>

Point npm at a group/virtual repository that proxies npmjs and hosts internal packages, and authenticate with a token from credentials:

```groovy
withCredentials([string(credentialsId: 'npm-token', variable: 'NPM_TOKEN')]) {
  sh '''
    echo "registry=https://nexus.example.com/repository/npm-group/" > .npmrc
    echo "//nexus.example.com/repository/npm-group/:_authToken=${NPM_TOKEN}" >> .npmrc
    npm ci
  '''
}
```

The proxy caches packages close to the agents, speeding ephemeral builds and surviving upstream outages.

</details>

### Q185. How do you use JFrog Artifactory from Jenkins? 🟡

<details>
<summary><b>Answer</b></summary>

The current approach is the **JFrog plugin**, which installs the JFrog CLI and exposes a `jf` step authenticated with a configured server:

```groovy
jf 'rt upload "target/*.jar" libs-release-local/shack-app/${BUILD_NUMBER}/'
jf 'rt build-publish'
jf 'docker push registry.example.com/shack-app:1.4.0'
```

Build info publishing links artifacts to the Jenkins build for traceability and Xray scanning. The older Artifactory plugin's Groovy DSL (`rtUpload`, `rtServer`) is deprecated in favour of `jf`.

</details>

### Q186. How do you build and push a Docker image from Jenkins on a VM agent? 🟢

<details>
<summary><b>Answer</b></summary>

```groovy
stage('Image') {
  steps {
    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'U', passwordVariable: 'P')]) {
      sh '''
        echo "$P" | docker login -u "$U" --password-stdin
        docker build -t devopsshack/shack-api:${GIT_COMMIT:0:7} .
        docker push devopsshack/shack-api:${GIT_COMMIT:0:7}
        docker logout
      '''
    }
  }
}
```

`${GIT_COMMIT:0:7}` is bash syntax; Jenkins' `sh` runs `/bin/sh`, which may be `dash`. Add `#!/bin/bash` as the first line of the script or compute the short SHA in Groovy.

</details>

### Q187. How do you scan an image with Trivy in a pipeline and gate on results? 🟡

<details>
<summary><b>Answer</b></summary>

```groovy
container('trivy') {
  sh '''
    trivy image --input image.tar \
      --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 \
      --format template --template "@contrib/junit.tpl" -o trivy-junit.xml
  '''
}
post { always { junit allowEmptyResults: true, testResults: 'trivy-junit.xml' } }
```

`--exit-code 1` fails the step when findings match. Cache the vulnerability DB (`--cache-dir` on a volume, or a mirrored `--db-repository`) so every build does not download it. Use `.trivyignore` with expiry comments for accepted risks.

</details>

### Q188. Why scan the image before pushing it? 🔴

<details>
<summary><b>Answer</b></summary>

If you push first and scan afterwards, a vulnerable image already exists in the registry with a valid tag, and anything watching the tag (Argo CD image updater, a developer running `pull`) may deploy it. Building to a local tarball (Kaniko `--no-push --tar-path`, `docker save`, BuildKit `type=oci`), scanning it, and only then pushing guarantees that every image in the registry passed the gate.

</details>

### Q189. How do you deploy to Kubernetes from Jenkins? 🟡

<details>
<summary><b>Answer</b></summary>

Common options:

- `kubectl`/`helm` in a deploy pod whose service account has a RoleBinding in the target namespace.
- `kubectl` with a kubeconfig stored as a Secret file credential (`withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG')])`) for external clusters.
- GitOps: Jenkins updates the image tag in a config repo and Argo CD or Flux applies it. This removes cluster credentials from Jenkins entirely, which is what we prefer for production.

Always follow `kubectl apply` with `kubectl rollout status --timeout` so the build fails when the rollout fails.

</details>

### Q190. How do you notify Slack or email from Jenkins? 🟢

<details>
<summary><b>Answer</b></summary>

```groovy
post {
  failure { slackSend channel: '#ci-alerts', color: 'danger', message: "FAILED ${env.JOB_NAME} #${env.BUILD_NUMBER} ${env.BUILD_URL}" }
  fixed   { slackSend channel: '#ci-alerts', color: 'good', message: "FIXED ${env.JOB_NAME}" }
  unsuccessful { emailext to: '$DEFAULT_RECIPIENTS', subject: '$PROJECT_NAME - $BUILD_STATUS', body: '${BUILD_URL}', recipientProviders: [culprits(), requestor()] }
}
```

Notify on `failure` and `fixed`, not on every success, or people mute the channel.

</details>

---

## Best practices & rapid-fire commands

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q191. What are the top Jenkinsfile best practices you follow? 🟡

<details>
<summary><b>Answer</b></summary>

- Declarative syntax, logic in shared libraries, small `script` blocks
- `agent none` at top when stages need different agents or approvals exist
- `timeout` on the pipeline and on every `input`
- `buildDiscarder`, `disableConcurrentBuilds(abortPrevious: true)` for PRs
- Single-quoted `sh` with credentials bound narrowly
- Heavy work in `sh` on agents, not in Groovy
- Fail fast: cheap checks first, `failFast` on parallel
- Build once, promote the same artifact; never rebuild per environment
- `post { always { junit ... } cleanup { cleanWs() } }`

</details>

### Q192. What are the top controller best practices? 🟡

<details>
<summary><b>Answer</b></summary>

0 executors on the built-in node; LTS only; image with pinned plugins; JCasC + Job DSL; SSO with group-based matrix or role auth; folder-scoped credentials; CSRF on; markup formatter safe; TLS behind a reverse proxy with correct `X-Forwarded-*` headers; WebSocket agents; Prometheus monitoring; backups with tested restores; short build retention; staging controller for upgrades; security advisories subscription.

</details>

### Q193. Rapid-fire: how do you get the initial admin password? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
kubectl -n jenkins exec jenkins-0 -- cat /var/jenkins_home/secrets/initialAdminPassword
sudo cat /var/lib/jenkins/secrets/initialAdminPassword   # package install
```

Skip the wizard entirely in automated setups with `-Djenkins.install.runSetupWizard=false` and JCasC.

</details>

### Q194. Rapid-fire: how do you list installed plugins and their versions? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
curl -s -u admin:$TOKEN "https://jenkins/pluginManager/api/json?depth=1&tree=plugins[shortName,version]" \
  | jq -r '.plugins[] | "\(.shortName):\(.version)"' | sort
java -jar jenkins-cli.jar -s https://jenkins/ -auth admin:$TOKEN list-plugins
```

The first command's output is a valid `plugins.txt`.

</details>

### Q195. Rapid-fire: how do you trigger a parameterized build with curl and follow it? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
curl -s -X POST -u alice:$TOKEN "https://jenkins/job/deploy/buildWithParameters?ENV=dev&TAG=main-abc1234" -D - | grep -i location
# Location: https://jenkins/queue/item/812/
curl -s -u alice:$TOKEN "https://jenkins/queue/item/812/api/json?tree=executable[number,url]"
curl -s -u alice:$TOKEN "https://jenkins/job/deploy/57/logText/progressiveText?start=0"
```

</details>

### Q196. Rapid-fire: how do you run the controller in Docker for a quick test? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
docker volume create jenkins_home
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts-jdk21
docker logs -f jenkins
```

Never mount the host Docker socket into this container for convenience on anything but a throwaway laptop setup.

</details>

### Q197. Rapid-fire: how do you connect an inbound agent with Docker? 🟢

<details>
<summary><b>Answer</b></summary>

Create the node in Jenkins (launch method: inbound), copy the secret, then:

```bash
docker run -d --name agent-1 --init jenkins/inbound-agent:latest-jdk21 \
  -url https://jenkins.example.com/ -webSocket \
  -secret 3f2a... -name agent-1 -workDir /home/jenkins/agent
```

With WebSocket only port 443/8080 is needed. Pin the inbound-agent tag in production.

</details>

### Q198. Rapid-fire: how do you reload JCasC and safely restart from the command line? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
curl -X POST -u admin:$TOKEN https://jenkins/configuration-as-code/reload
java -jar jenkins-cli.jar -s https://jenkins/ -auth admin:$TOKEN reload-jcasc-configuration
java -jar jenkins-cli.jar -s https://jenkins/ -auth admin:$TOKEN safe-restart
java -jar jenkins-cli.jar -s https://jenkins/ -auth admin:$TOKEN quiet-down
```

</details>

### Q199. Rapid-fire: how do you abort a stuck build that ignores "Abort"? 🟡

<details>
<summary><b>Answer</b></summary>

Escalate step by step: click Abort; then use the build URL `/term` (terminate), then `/kill` (hard kill, only offered after term). From the script console as a last resort:

```groovy
Jenkins.instance.getItemByFullName('shack-app/main').getBuildByNumber(42)
  .doKill()
```

Stuck builds are usually blocked on a dead agent channel or a swallowed `InterruptedException`. Fix the root cause after.

</details>

### Q200. Rapid-fire: where are the controller logs and how do you increase logging for one component? 🟢

<details>
<summary><b>Answer</b></summary>

Container: `docker logs jenkins` or `kubectl -n jenkins logs jenkins-0 -c jenkins`. Package install: `journalctl -u jenkins`. In the UI, **Manage Jenkins > System Log** shows recent entries; **Add new log recorder** with a logger name such as `org.csanchez.jenkins.plugins.kubernetes` at `FINE` captures detailed logs for one plugin without flooding everything. Agent remoting logs are in `<workDir>/remoting/logs`.

</details>

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
