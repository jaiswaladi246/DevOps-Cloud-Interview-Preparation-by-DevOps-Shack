# Jenkins — Cheatsheet

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

> Dense reference for daily use and interview prep. Jenkins LTS 2.4xx+ on JDK 17/21, Pipeline (declarative + scripted), kubernetes plugin, Configuration as Code, Job DSL, matrix-auth 3.x.

[⬅ Back to Jenkins](./README.md) · [🏠 Home](../README.md)

## Contents

[Run Jenkins](#run-jenkins) · [Declarative skeleton](#declarative-pipeline-skeleton) · [agent](#agent) · [options](#options) · [triggers](#triggers) · [parameters](#parameters) · [environment and tools](#environment-and-tools) · [when](#when) · [parallel and matrix](#parallel-and-matrix) · [input](#input) · [post](#post) · [Common steps](#common-steps) · [Error handling](#error-handling-and-results) · [Credentials binding](#credentials-binding) · [Shared libraries](#shared-library-layout) · [Kubernetes agents](#kubernetes-agents) · [jenkins-cli](#jenkins-cli) · [REST API](#rest-api) · [JCasC](#jcasc-snippets) · [Job DSL](#job-dsl-seed-snippets) · [Plugins](#plugin-management) · [Script console](#script-console-snippets) · [JENKINS_HOME](#jenkins_home-layout) · [Errors](#common-errors) · [Top 15](#top-15-interview-one-liners)

---

## Run Jenkins

| Task | Command |
|------|---------|
| Run LTS in Docker | `docker run -d --name jenkins -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts-jdk21` |
| Initial admin password | `docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword` |
| Skip setup wizard (JCasC builds) | `-e JAVA_OPTS="-Djenkins.install.runSetupWizard=false"` |
| Point at a JCasC file | `-e CASC_JENKINS_CONFIG=/var/jenkins_home/casc/jenkins.yaml` |
| Install on Kubernetes (Helm) | `helm repo add jenkins https://charts.jenkins.io && helm upgrade --install jenkins jenkins/jenkins -n jenkins --create-namespace -f values.yaml` |
| Admin password (Helm chart) | `kubectl -n jenkins get secret jenkins -o jsonpath='{.data.jenkins-admin-password}' \| base64 -d` |
| Health check | `curl -s -o /dev/null -w '%{http_code}' https://jenkins.example.com/login` |
| Version | `curl -sI https://jenkins.example.com/ \| grep -i x-jenkins:` |

---

## Declarative pipeline skeleton

```groovy
@Library('shack-lib@v1.4.0') _

pipeline {
  agent none                                   // pick agents per stage
  options {
    timeout(time: 30, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '5'))
    disableConcurrentBuilds(abortPrevious: true)
    timestamps()
  }
  parameters {
    choice(name: 'ENV', choices: ['dev', 'staging', 'prod'], description: 'Target')
    booleanParam(name: 'SKIP_TESTS', defaultValue: false)
  }
  environment {
    APP      = 'shack-app'
    REGISTRY = 'registry.example.com/shack'
  }
  triggers { cron('H 2 * * 1-5') }

  stages {
    stage('Test') {
      agent { label 'linux && docker' }
      when { not { expression { params.SKIP_TESTS } } }
      steps {
        sh 'npm ci && npm test'
      }
      post { always { junit 'reports/**/*.xml' } }
    }
    stage('Build') {
      agent { label 'linux && docker' }
      steps {
        script { env.TAG = "${env.BRANCH_NAME}-${env.GIT_COMMIT.take(7)}" }
        sh 'docker build -t "$REGISTRY/$APP:$TAG" .'
      }
    }
    stage('Approve prod') {
      when { allOf { branch 'main'; expression { params.ENV == 'prod' } } }
      input { message 'Deploy to prod?'; submitter 'release-managers' }
      agent { label 'deploy' }
      steps { sh './deploy.sh "$ENV" "$TAG"' }
    }
  }
  post {
    failure { echo "Failed: ${env.BUILD_URL}" }
    cleanup { echo 'done' }
  }
}
```

Directive order inside `pipeline` and `stage` does not matter to the parser, but the convention above (agent, options, parameters, environment, triggers, stages, post) keeps files readable.

| Block | Where allowed | Notes |
|-------|---------------|-------|
| `agent` | pipeline (required), stage | `agent none` at top forces each stage to choose |
| `stages` / `stage` / `steps` | pipeline, nested in stage | A stage has exactly one of `steps`, `stages`, `parallel`, `matrix` |
| `options` | pipeline, stage | Stage options are a subset (`timeout`, `retry`, `skipDefaultCheckout`, `timestamps`) |
| `parameters` | pipeline only | First run registers them; values appear on the next run |
| `triggers` | pipeline only | Not used for multibranch push builds (webhooks do that) |
| `environment` | pipeline, stage | Supports `credentials('id')` |
| `tools` | pipeline, stage | Names must match Global Tool Configuration |
| `when` | stage | Evaluated after `agent` unless `beforeAgent true` |
| `input` | stage | Pauses before the stage's agent is allocated |
| `post` | pipeline, stage | Condition blocks, see [post](#post) |
| `script` | inside `steps` | Escape hatch to scripted Groovy; keep it small |

---

## agent

| Form | Meaning |
|------|---------|
| `agent any` | Any available executor (avoid in production; can land on the wrong node) |
| `agent none` | No global agent; each stage declares its own |
| `agent { label 'linux && docker' }` | Node matching the label expression (`&&`, `\|\|`, `!`) |
| `agent { node { label 'linux'; customWorkspace '/data/ws/app' } }` | Label plus custom workspace |
| `agent { docker { image 'node:22-alpine'; args '-v $HOME/.npm:/root/.npm'; reuseNode true } }` | Run the stage inside a container on a Docker-capable agent |
| `agent { dockerfile { filename 'ci/Dockerfile'; additionalBuildArgs '--pull' } }` | Build the image from the repo, then run inside it |
| `agent { kubernetes { yamlFile 'ci/pod.yaml'; defaultContainer 'node' } }` | Pod per build from the kubernetes plugin |

`reuseNode true` runs the container on the node and workspace already allocated by the parent `agent`, instead of grabbing a new executor.

---

## options

| Option | Effect |
|--------|--------|
| `timeout(time: 30, unit: 'MINUTES')` | Abort the whole run (or stage) after the limit |
| `retry(2)` | Retry the pipeline or stage on failure |
| `buildDiscarder(logRotator(numToKeepStr: '30', daysToKeepStr: '14'))` | Keep the controller disk under control |
| `disableConcurrentBuilds()` | One build at a time; `abortPrevious: true` cancels the older one |
| `skipDefaultCheckout()` | No automatic `checkout scm` per agent |
| `skipStagesAfterUnstable()` | Stop running stages once the build is UNSTABLE |
| `timestamps()` | Timestamp every log line |
| `quietPeriod(10)` | Wait N seconds before starting, merging rapid triggers |
| `preserveStashes(buildCount: 5)` | Keep stashes for restart-from-stage |
| `parallelsAlwaysFailFast()` | Every `parallel` block fails fast |
| `checkoutToSubdirectory('src')` | Default checkout goes into a subfolder |
| `durabilityHint('PERFORMANCE_OPTIMIZED')` | Fewer disk writes; may lose state on a hard controller crash |
| `newContainerPerStage()` | With a top-level `docker`/`dockerfile` agent, a fresh container per stage |
| `overrideIndexTriggers(false)` | Branch indexing does not trigger builds for this branch |

---

## triggers

| Trigger | Example | Notes |
|---------|---------|-------|
| Cron | `cron('H 2 * * 1-5')` | `H` hashes the job name to spread load |
| Poll SCM | `pollSCM('H/5 * * * *')` | Fallback only; webhooks are better |
| Upstream | `upstream(upstreamProjects: 'lib-build', threshold: hudson.model.Result.SUCCESS)` | Run after another job |
| GitHub push (freestyle-style jobs) | `githubPush()` | Multibranch jobs use the branch source webhook instead |

Cron aliases: `@hourly`, `@daily`, `@midnight`, `@weekly`, `@monthly` (all hashed). `H(0-29)/10 * * * *` = every 10 min within the first half hour. Time zone per job: `TZ=Asia/Kolkata` on the first line of the cron spec.

---

## parameters

| Type | Example |
|------|---------|
| String | `string(name: 'VERSION', defaultValue: '1.0.0', trim: true)` |
| Multi-line text | `text(name: 'NOTES', defaultValue: '')` |
| Boolean | `booleanParam(name: 'DRY_RUN', defaultValue: true)` |
| Choice | `choice(name: 'ENV', choices: ['dev', 'staging', 'prod'])` |
| Password | `password(name: 'TEMP_PASS', defaultValue: '')` |
| Credentials picker | `credentials(name: 'DEPLOY_CRED', credentialType: 'com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl', required: true)` |

Read them as `params.ENV` (typed: boolean stays boolean). In shell they are also environment variables: `sh 'echo "$ENV"'`.

---

## environment and tools

```groovy
environment {
  AWS_REGION   = 'ap-south-1'
  SONAR_TOKEN  = credentials('sonar-token')        // secret text -> masked
  NEXUS        = credentials('nexus-creds')        // user/pass -> NEXUS, NEXUS_USR, NEXUS_PSW
  GIT_SHORT    = "${env.GIT_COMMIT?.take(7)}"
}
tools {
  jdk   'temurin-21'
  maven 'maven-3.9'
  nodejs 'node-22'                                 // NodeJS plugin
}
```

| Built-in variable | Value |
|-------------------|-------|
| `BUILD_NUMBER`, `BUILD_ID` | Build number |
| `BUILD_URL`, `JOB_NAME`, `JOB_BASE_NAME` | Links and names (`JOB_NAME` includes folders) |
| `WORKSPACE` | Absolute workspace path on the agent |
| `NODE_NAME`, `NODE_LABELS`, `EXECUTOR_NUMBER` | Where the build runs |
| `BRANCH_NAME`, `TAG_NAME`, `CHANGE_ID`, `CHANGE_TARGET`, `CHANGE_BRANCH` | Multibranch only |
| `GIT_COMMIT`, `GIT_BRANCH`, `GIT_URL`, `GIT_PREVIOUS_SUCCESSFUL_COMMIT` | Set after `checkout scm` (git plugin) |
| `JENKINS_URL` | Controller URL from Location config |

`currentBuild` gives `result`, `currentResult`, `number`, `displayName`, `description`, `durationString`, `previousBuild`, `changeSets`, `getBuildCauses()`. Full list: `<job-url>/pipeline-syntax/globals`.

---

## when

| Condition | Example |
|-----------|---------|
| Branch (multibranch) | `when { branch 'main' }`, `branch pattern: 'release/.*', comparator: 'REGEXP'` |
| Tag build | `when { buildingTag() }`, `when { tag 'v*' }` |
| Pull request | `when { changeRequest target: 'main' }` |
| Files changed | `when { changeset 'charts/**' }` |
| Env var | `when { environment name: 'ENV', value: 'prod' }` |
| Groovy expression | `when { expression { params.ENV == 'prod' } }` |
| Combinators | `allOf { ... }`, `anyOf { ... }`, `not { ... }` |
| Cause | `when { triggeredBy 'TimerTrigger' }`, `triggeredBy cause: 'UserIdCause'` |
| Evaluate before agent | `when { beforeAgent true; branch 'main' }` |
| Evaluate before input / options | `beforeInput true`, `beforeOptions true` |

Always use `beforeAgent true` when the condition does not need the workspace, so skipped stages do not spin up a pod.

---

## parallel and matrix

```groovy
stage('Checks') {
  failFast true
  parallel {
    stage('Unit')  { agent { label 'linux' } steps { sh 'npm test' } }
    stage('Lint')  { agent { label 'linux' } steps { sh 'npm run lint' } }
    stage('Trivy fs') { agent { label 'linux' } steps { sh 'trivy fs --exit-code 1 --severity HIGH,CRITICAL .' } }
  }
}

stage('Cross-platform') {
  matrix {
    axes {
      axis { name 'OS';   values 'linux', 'windows' }
      axis { name 'NODE'; values '20', '22' }
    }
    excludes {
      exclude {
        axis { name 'OS';   values 'windows' }
        axis { name 'NODE'; values '20' }
      }
    }
    agent { label "${OS}" }
    stages {
      stage('Test') { steps { echo "OS=${OS} NODE=${NODE}" } }
    }
  }
}
```

Scripted parallel: `parallel(unit: { sh 'npm test' }, lint: { sh 'npm run lint' }, failFast: true)`.

---

## input

```groovy
stage('Approve') {
  agent none                                    // do not hold an executor while waiting
  options { timeout(time: 4, unit: 'HOURS') }
  input {
    message 'Promote to production?'
    ok 'Deploy'
    submitter 'release-managers,alice'          // users or groups
    submitterParameter 'APPROVER'
    parameters { string(name: 'CHANGE_TICKET', defaultValue: '') }
  }
  steps { echo "Approved by ${APPROVER}, ticket ${CHANGE_TICKET}" }
}
```

Step form inside `script`: `def r = input message: 'Go?', parameters: [choice(name: 'ENV', choices: ['staging','prod'])]`. Pair with `milestone()` so an older build cannot deploy after a newer one.

---

## post

| Condition | Runs when |
|-----------|-----------|
| `always` | Every time |
| `changed` | Result differs from the previous build |
| `fixed` | Now SUCCESS, previous was FAILURE or UNSTABLE |
| `regression` | Now FAILURE/UNSTABLE/ABORTED, previous was SUCCESS |
| `aborted` | Manually aborted or timed out |
| `failure` | FAILURE |
| `success` | SUCCESS |
| `unstable` | UNSTABLE (test failures, `unstable()`) |
| `unsuccessful` | Anything but SUCCESS |
| `cleanup` | Last, after all others |

Execution order is the table order. Put notifications in `failure`/`fixed`, `cleanWs()` in `cleanup`.

---

## Common steps

| Step | Example | Notes |
|------|---------|-------|
| Shell | `sh 'make test'` | Fails the build on non-zero exit |
| Shell with output | `def v = sh(script: 'git describe --tags', returnStdout: true).trim()` | Always `.trim()` |
| Shell exit code | `def rc = sh(script: 'grep -q x f', returnStatus: true)` | Does not fail the build |
| Labelled shell | `sh label: 'Unit tests', script: 'npm test'` | Nicer stage view |
| Windows | `bat 'build.cmd'`, `powershell 'Get-ChildItem'`, `pwsh '...'` | |
| Checkout | `checkout scm` | Uses the job's SCM (multibranch) |
| Git | `git url: 'https://github.com/org/app.git', branch: 'main', credentialsId: 'github-app'` | Simple clone |
| Change dir | `dir('frontend') { sh 'npm ci' }` | |
| Clean workspace | `deleteDir()`, `cleanWs()` | `cleanWs` needs Workspace Cleanup plugin |
| Stash / unstash | `stash name: 'dist', includes: 'dist/**'`, `unstash 'dist'` | Moves files between agents via the controller; keep small |
| Archive | `archiveArtifacts artifacts: 'dist/*.tgz', fingerprint: true, allowEmptyArchive: false` | Stored on the controller |
| Tests | `junit testResults: 'reports/*.xml', allowEmptyResults: false` | Failures mark UNSTABLE |
| Files | `readFile 'VERSION'`, `writeFile file: 'out.txt', text: 'x'`, `fileExists 'pom.xml'` | |
| JSON / YAML | `readJSON file: 'package.json'`, `readYaml file: 'values.yaml'`, `writeJSON file: 'o.json', json: m` | Pipeline Utility Steps plugin |
| Find files | `findFiles(glob: '**/target/*.jar')` | Pipeline Utility Steps |
| Env for a block | `withEnv(['NODE_ENV=production']) { sh 'npm run build' }` | |
| Downstream job | `build job: 'deploy', parameters: [string(name: 'TAG', value: env.TAG)], wait: true, propagate: true` | |
| Lock | `lock(resource: 'staging-env') { sh './deploy.sh staging' }` | Lockable Resources plugin |
| Ordering | `milestone()` | Older builds abort when a newer one passes |
| Wait | `sleep time: 30, unit: 'SECONDS'`, `waitUntil { sh(script: './ready.sh', returnStatus: true) == 0 }` | |
| Tool path | `def mvn = tool 'maven-3.9'` | Scripted use |
| Build info | `currentBuild.displayName = "#${BUILD_NUMBER} ${env.TAG}"` | |
| Email | `emailext to: 'team@example.com', subject: "${JOB_NAME} failed", body: "${BUILD_URL}"` | Email Extension plugin |
| Slack | `slackSend channel: '#deploys', color: 'danger', message: "Failed: ${BUILD_URL}"` | Slack plugin |
| SonarQube | `withSonarQubeEnv('sonarqube') { sh 'sonar-scanner' }` then `timeout(10) { waitForQualityGate abortPipeline: true }` | Needs the SonarQube webhook to Jenkins |

Snippet generator for any installed step: `<job-url>/pipeline-syntax`. Declarative directive generator: `<job-url>/directive-generator`.

## Error handling and results

| Need | Code |
|------|------|
| Fail now | `error 'Version file missing'` |
| Mark unstable, continue | `unstable 'Coverage below 80%'` |
| Catch, mark stage failed, keep going | `catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') { sh './flaky-e2e.sh' }` |
| Warn only | `warnError('lint failed') { sh 'npm run lint' }` |
| Retry a step | `retry(3) { sh './push.sh' }`; retry only on lost agents/pods: `retry(count: 2, conditions: [agent(), kubernetesAgent()]) { ... }` |
| Timeout a step | `timeout(time: 5, unit: 'MINUTES') { sh './smoke.sh' }` |
| Groovy try/catch | `script { try { sh 'x' } catch (err) { echo "${err}"; throw err } finally { sh 'cleanup' } }` |

Never swallow `org.jenkinsci.plugins.workflow.steps.FlowInterruptedException` in a `catch`; it is how aborts and timeouts propagate.

---

## Credentials binding

### Credential types and how they bind

| Credential type | `environment { X = credentials('id') }` gives | `withCredentials` binding |
|-----------------|-----------------------------------------------|---------------------------|
| Secret text | `X` = secret | `string(credentialsId: 'id', variable: 'TOKEN')` |
| Username + password | `X` = `user:pass`, `X_USR`, `X_PSW` | `usernamePassword(credentialsId: 'id', usernameVariable: 'U', passwordVariable: 'P')` |
| Secret file | `X` = path to a temp file | `file(credentialsId: 'id', variable: 'KUBECONFIG')` |
| SSH private key | `X` = key file path, `X_USR`, `X_PSW` (passphrase) | `sshUserPrivateKey(credentialsId: 'id', keyFileVariable: 'KEY', usernameVariable: 'U')` |
| Certificate (PKCS#12) | not supported | `certificate(credentialsId: 'id', keystoreVariable: 'KS', passwordVariable: 'KSPASS')` |
| User:pass as one string | — | `usernameColonPassword(credentialsId: 'id', variable: 'AUTH')` |
| AWS keys (AWS Credentials plugin) | — | `aws(credentialsId: 'aws-ci', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')` |

```groovy
// Use single quotes: the shell expands the secret, Groovy never sees it
withCredentials([usernamePassword(credentialsId: 'nexus-creds',
                                  usernameVariable: 'NEXUS_USER',
                                  passwordVariable: 'NEXUS_PASS')]) {
  sh 'echo "$NEXUS_PASS" | docker login nexus.example.com -u "$NEXUS_USER" --password-stdin'
}

withCredentials([file(credentialsId: 'kubeconfig-dev', variable: 'KUBECONFIG')]) {
  sh 'kubectl rollout status deploy/shack-app -n dev --timeout=120s'
}

sshagent(credentials: ['git-ssh']) {                 // SSH Agent plugin
  sh 'git push origin HEAD:refs/tags/v1.4.0'
}

docker.withRegistry('https://registry.example.com', 'registry-creds') {   // Docker Pipeline plugin
  docker.image("shack/app:${env.TAG}").push()
}
```

| Rule | Why |
|------|-----|
| `sh 'curl -u "$U:$P" ...'` not `sh "curl -u ${U}:${P} ..."` | Groovy interpolation puts the secret in the process args and triggers the "insecure interpolation of sensitive variables" warning |
| Never `echo` or `set -x` around secrets | Masking only matches the exact string; base64 or substrings leak |
| Scope credentials to folders | Team A's jobs cannot reference team B's credential IDs |
| Prefer short-lived creds | OIDC/IRSA on agents, Vault or AWS Secrets Manager credential providers |
| Credential IDs are an API | Name them `team-purpose-env` (`payments-nexus-prod`) and keep them stable |

---

## Shared library layout

```text
shack-lib/
├── vars/
│   ├── kanikoBuild.groovy        # global step: kanikoBuild(image: ..., tag: ...)
│   ├── kanikoBuild.txt           # help text shown in pipeline-syntax/globals
│   └── standardPipeline.groovy   # a whole declarative pipeline as one step
├── src/
│   └── com/devopsshack/ci/
│       └── Semver.groovy         # classes: import com.devopsshack.ci.Semver
├── resources/
│   └── com/devopsshack/ci/
│       └── pod-node.yaml         # libraryResource 'com/devopsshack/ci/pod-node.yaml'
└── test/
    └── groovy/KanikoBuildTest.groovy   # JenkinsPipelineUnit tests
```

```groovy
// vars/kanikoBuild.groovy
def call(Map cfg = [:]) {
  String image = cfg.image ?: error('kanikoBuild: image is required')
  String tag   = cfg.tag ?: env.GIT_COMMIT.take(7)
  container('kaniko') {
    sh """
      /kaniko/executor --context "\$WORKSPACE" --dockerfile Dockerfile \
        --destination "${image}:${tag}" --cache=true
    """
  }
}
```

```groovy
// vars/standardPipeline.groovy: one-line Jenkinsfiles for every repo
def call(Map cfg) {
  pipeline {
    agent { kubernetes { yaml libraryResource('com/devopsshack/ci/pod-node.yaml'); defaultContainer 'node' } }
    stages {
      stage('Test')  { steps { sh 'npm ci && npm test' } }
      stage('Build') { steps { kanikoBuild(image: cfg.image) } }
    }
  }
}
```

| Loading form | Example |
|--------------|---------|
| Annotation, pinned tag | `@Library('shack-lib@v1.4.0') _` |
| Several libraries | `@Library(['shack-lib@v1.4.0', 'security-lib@v2']) _` |
| Import classes too | `@Library('shack-lib@v1.4.0') import com.devopsshack.ci.Semver` |
| Dynamic (runtime) | `library 'shack-lib@main'` |
| Ad hoc from SCM (untrusted, sandboxed) | `library identifier: 'shack-lib@v1.4.0', retriever: modernSCM([$class: 'GitSCMSource', remote: 'https://github.com/org/shack-lib.git', credentialsId: 'github-app'])` |
| Implicit | Set `implicit: true` in the global config; no annotation needed |

| Fact | Detail |
|------|--------|
| Trusted | Global (Manage Jenkins) libraries run outside the sandbox; protect the repo like prod code |
| Untrusted | Folder-level or ad hoc libraries run in the sandbox |
| Version override | `allowVersionOverride: true` lets a Jenkinsfile pick `@branch`; disable it for prod libraries |
| `@NonCPS` | For pure Groovy helpers (regex, JSON parsing); cannot call steps |
| Steps from `src/` classes | Pass the script object in: `new Deployer(this).run()` and call `script.sh(...)` |
| Changes take effect | Next build that loads the library; `vars/` changes do not need a restart |

---

## Kubernetes agents

```groovy
pipeline {
  agent {
    kubernetes {
      defaultContainer 'node'
      yaml '''
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins-agent
  containers:
    - name: node
      image: node:22-alpine
      command: ["sleep"]
      args: ["infinity"]
      resources:
        requests: { cpu: "500m", memory: "1Gi" }
        limits:   { memory: "2Gi" }
    - name: kaniko
      image: gcr.io/kaniko-project/executor:debug
      command: ["sleep"]
      args: ["infinity"]
    - name: trivy
      image: aquasec/trivy:latest
      command: ["sleep"]
      args: ["infinity"]
'''
    }
  }
  stages {
    stage('Test')  { steps { sh 'npm ci && npm test' } }
    stage('Image') { steps { container('kaniko') { sh '/kaniko/executor --no-push --tar-path image.tar --destination app:ci' } } }
    stage('Scan')  { steps { container('trivy')  { sh 'trivy image --input image.tar --exit-code 1 --severity HIGH,CRITICAL --ignore-unfixed' } } }
  }
}
```

| Setting | Use |
|---------|-----|
| `yamlFile 'ci/pod.yaml'` | Keep the pod spec in the repo |
| `inheritFrom 'base'` | Start from a pod template defined in the cloud config |
| `defaultContainer 'node'` | Steps run there unless wrapped in `container()` |
| `podRetention onFailure()` | Keep failed pods for debugging (clean up after) |
| `idleMinutes 10` | Reuse the pod for the next build (trades isolation for speed) |
| `retries 2` / `retry(count: 2, conditions: [kubernetesAgent()])` | Survive spot node loss |
| `jnlp` container | Injected automatically (`jenkins/inbound-agent`); override only to pin its image |
| Scripted form | `podTemplate(yaml: y) { node(POD_LABEL) { container('node') { sh 'npm test' } } }` |

Agent pods connect back with WebSocket (`webSocket: true` in the cloud) or to port 50000; the controller service must be reachable from the agent namespace.

---

## jenkins-cli

Download from your controller and authenticate with an API token (User > Security > API Token).

```bash
curl -sO https://jenkins.example.com/jnlpJars/jenkins-cli.jar
alias jcli='java -jar jenkins-cli.jar -s https://jenkins.example.com/ -webSocket -auth alice:$JENKINS_TOKEN'
```

| Task | Command |
|------|---------|
| Who am I / list commands | `jcli who-am-i`, `jcli help` |
| List jobs (in a folder) | `jcli list-jobs`, `jcli list-jobs team-a` |
| Build, wait, stream log | `jcli build team-a/app/main -p ENV=dev -s -v -f` |
| Follow a build log | `jcli console team-a/app/main 42 -f` |
| Stop builds | `jcli stop-builds team-a/app/main` |
| Export job XML | `jcli get-job team-a/legacy-job > legacy-job.xml` |
| Create / update job | `jcli create-job new-job < job.xml`, `jcli update-job new-job < job.xml` |
| Copy / delete / enable / disable | `jcli copy-job src dst`, `jcli delete-job x`, `jcli enable-job x`, `jcli disable-job x` |
| Delete a build range | `jcli delete-builds team-a/app/main 1-100` |
| Keep a build forever | `jcli keep-build team-a/app/main 42` |
| Validate a Jenkinsfile | `jcli declarative-linter < Jenkinsfile` |
| Run Groovy | `jcli groovy = < script.groovy` |
| Nodes | `jcli offline-node agent-1 -m "patching"`, `jcli online-node agent-1`, `jcli get-node agent-1`, `jcli create-node agent-2 < node.xml` |
| Plugins | `jcli list-plugins`, `jcli install-plugin git -deploy` |
| Restart / maintenance | `jcli safe-restart`, `jcli quiet-down`, `jcli cancel-quiet-down`, `jcli reload-configuration` |
| JCasC | `jcli check-configuration < jenkins.yaml`, `jcli reload-jcasc-configuration`, `jcli export-configuration > current.yaml` |
| Credentials (credentials plugin) | `jcli list-credentials system::system::jenkins`, `jcli create-credentials-by-xml system::system::jenkins _ < cred.xml` |

`-webSocket` avoids needing the TCP agent port; the old `-remoting` mode is removed.

---

## REST API

Every page has `/api/json` (also `/api/xml`, `/api/python`). Authenticate with `user:api-token`; with a token no CSRF crumb is needed.

```bash
J=https://jenkins.example.com
AUTH="alice:$JENKINS_TOKEN"
```

| Task | Command |
|------|---------|
| Who am I | `curl -su $AUTH $J/whoAmI/api/json` |
| Job info, filtered | `curl -su $AUTH "$J/job/team-a/job/app/job/main/api/json?tree=lastBuild[number,result],healthReport[score]"` |
| Last 5 builds | `curl -su $AUTH "$J/job/app/api/json?tree=builds[number,result,duration,timestamp]{0,5}"` |
| Trigger build | `curl -su $AUTH -X POST $J/job/app/build` |
| Trigger with parameters | `curl -su $AUTH -X POST "$J/job/app/buildWithParameters?ENV=dev&TAG=abc1234"` |
| Find the build a trigger created | Read `Location:` header (queue item), then `curl -su $AUTH $J/queue/item/<id>/api/json` and take `executable.number` |
| Console log | `curl -su $AUTH $J/job/app/42/consoleText` |
| Streaming log | `curl -su $AUTH "$J/job/app/42/logText/progressiveText?start=0"` (next offset in `X-Text-Size`) |
| Stage view data | `curl -su $AUTH $J/job/app/42/wfapi/describe` |
| Stop a build | `curl -su $AUTH -X POST $J/job/app/42/stop` |
| Download an artifact | `curl -su $AUTH -O $J/job/app/lastSuccessfulBuild/artifact/dist/app.tgz` |
| Get / update job config | `curl -su $AUTH $J/job/app/config.xml > app.xml`; `curl -su $AUTH -X POST -H 'Content-Type: application/xml' --data-binary @app.xml $J/job/app/config.xml` |
| Create job | `curl -su $AUTH -X POST -H 'Content-Type: application/xml' --data-binary @app.xml "$J/createItem?name=app"` |
| Enable / disable / delete | `-X POST $J/job/app/enable`, `.../disable`, `.../doDelete` |
| Scan a multibranch job | `curl -su $AUTH -X POST $J/job/team-a/job/app/build?delay=0` |
| Queue / cancel | `curl -su $AUTH $J/queue/api/json`; `curl -su $AUTH -X POST "$J/queue/cancelItem?id=123"` |
| Nodes and executors | `curl -su $AUTH "$J/computer/api/json?tree=computer[displayName,offline,numExecutors]"` |
| Installed plugins | `curl -su $AUTH "$J/pluginManager/api/json?depth=1&tree=plugins[shortName,version]"` |
| Lint a Jenkinsfile | `curl -su $AUTH -X POST -F "jenkinsfile=<Jenkinsfile" $J/pipeline-model-converter/validate` |
| Run Groovy (admin) | `curl -su $AUTH --data-urlencode "script=println Jenkins.VERSION" $J/scriptText` |
| Safe restart | `curl -su $AUTH -X POST $J/safeRestart` |
| Crumb (password auth only) | `curl -su alice:pass -c jar $J/crumbIssuer/api/json`, then send `Jenkins-Crumb: <value>` with `-b jar` |

Folder paths nest as `/job/<folder>/job/<name>`. Branch names with `/` are double-encoded in URLs: `feature/login` becomes `feature%252Flogin`.

---

## JCasC snippets

Loaded from `CASC_JENKINS_CONFIG` (a file, a folder of YAML files, or a URL). Secrets come from env vars or mounted files via `${VAR}`; never commit values. Export the running config from **Manage Jenkins > Configuration as Code > View/Download** to learn the exact keys for any plugin.

```yaml
jenkins:
  systemMessage: "Managed by JCasC: changes in the UI will be overwritten"
  numExecutors: 0                       # nothing runs on the controller
  mode: EXCLUSIVE
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: admin
          password: "${JENKINS_ADMIN_PASSWORD}"
  authorizationStrategy:
    globalMatrix:                       # matrix-auth 3.x syntax
      entries:
        - user:
            name: admin
            permissions:
              - "Overall/Administer"
        - group:
            name: authenticated
            permissions:
              - "Overall/Read"
              - "Job/Read"
              - "Job/Build"
  crumbIssuer:
    standard:
      excludeClientIPFromCrumb: false
  clouds:
    - kubernetes:
        name: kubernetes
        namespace: jenkins-agents
        jenkinsUrl: "http://jenkins.jenkins.svc.cluster.local:8080/"
        webSocket: true
        containerCapStr: "50"
        podRetention: never
        templates:
          - name: base
            label: k8s
            serviceAccount: jenkins-agent
            containers:
              - name: jnlp
                image: jenkins/inbound-agent:latest-jdk21
                resourceRequestCpu: "200m"
                resourceRequestMemory: "256Mi"

unclassified:
  location:
    url: "https://jenkins.example.com/"
    adminAddress: "jenkins@example.com"
  globalLibraries:
    libraries:
      - name: shack-lib
        defaultVersion: v1.4.0
        implicit: false
        allowVersionOverride: false
        retriever:
          modernSCM:
            scm:
              git:
                remote: "https://github.com/org/shack-lib.git"
                credentialsId: github-app
  sonarGlobalConfiguration:
    installations:
      - name: sonarqube
        serverUrl: "https://sonar.example.com"
        credentialsId: sonar-token

credentials:
  system:
    domainCredentials:
      - credentials:
          - string:
              scope: GLOBAL
              id: sonar-token
              secret: "${SONAR_TOKEN}"
          - usernamePassword:
              scope: GLOBAL
              id: nexus-creds
              username: deployer
              password: "${NEXUS_PASSWORD}"
          - basicSSHUserPrivateKey:
              scope: GLOBAL
              id: git-ssh
              username: git
              privateKeySource:
                directEntry:
                  privateKey: "${readFile:/run/secrets/git-ssh-key}"
          - file:
              scope: GLOBAL
              id: kubeconfig-dev
              fileName: kubeconfig
              secretBytes: "${readFileBase64:/run/secrets/kubeconfig-dev}"

tool:
  maven:
    installations:
      - name: maven-3.9
        properties:
          - installSource:
              installers:
                - maven:
                    id: "3.9.9"

security:
  globalJobDslSecurityConfiguration:
    useScriptSecurity: true
```

| Task | How |
|------|-----|
| Variable with default | `"${SLACK_CHANNEL:-#ci}"` |
| Read a file / base64 of a file | `"${readFile:/path}"`, `"${readFileBase64:/path}"` |
| Escape a literal `${` | `^${NOT_A_VAR}` |
| Validate before applying | `jcli check-configuration < jenkins.yaml` |
| Reload without restart | `jcli reload-jcasc-configuration` or `POST $J/configuration-as-code/reload` |
| Schema for your plugin set | `$J/configuration-as-code/schema` |
| Secrets from Vault / AWS | Mount via CSI or set env vars; secret source plugins exist for Vault and AWS |

---

## Job DSL seed snippets

```yaml
# In JCasC: seed jobs at startup (job-dsl plugin)
jobs:
  - file: /var/jenkins_home/casc/jobs/teams.groovy
  - script: |
      folder('platform') { description('Platform team jobs') }
```

```groovy
// teams.groovy
['payments', 'search'].each { team ->
  folder(team)
  multibranchPipelineJob("${team}/app") {
    branchSources {
      branchSource {
        source {
          github {
            id("${team}-app")
            repoOwner('org')
            repository("${team}-app")
            repositoryUrl("https://github.com/org/${team}-app")
            configuredByUrl(true)
            credentialsId('github-app')
          }
        }
      }
    }
    orphanedItemStrategy {
      discardOldItems { numToKeep(10) }
    }
    triggers {
      periodicFolderTrigger { interval('1d') }    // safety net; webhooks do the real work
    }
  }
}
```

API reference for your exact plugin set: `$J/plugin/job-dsl/api-viewer/index.html`.

---

## Plugin management

```text
# plugins.txt: one plugin per line, <short-name>:<version>
# Replace "latest" with the exact versions exported from your tested staging controller
configuration-as-code:latest
workflow-aggregator:latest
git:latest
kubernetes:latest
job-dsl:latest
matrix-auth:latest
```

`latest` only shows the format. In production every line carries an exact version, exported from a staging controller with the [script console snippet](#script-console-snippets), so image builds are reproducible.

```dockerfile
FROM jenkins/jenkins:lts-jdk21
ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"
COPY --chown=jenkins:jenkins plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt
COPY --chown=jenkins:jenkins casc/ /var/jenkins_home/casc/
ENV CASC_JENKINS_CONFIG=/var/jenkins_home/casc
```

| Task | Command |
|------|---------|
| Install from file | `jenkins-plugin-cli --plugin-file plugins.txt` |
| Install specific plugins | `jenkins-plugin-cli --plugins git:5.7.0 kubernetes` |
| See what would be installed | `jenkins-plugin-cli --plugin-file plugins.txt --list` |
| Show available updates | `jenkins-plugin-cli --plugin-file plugins.txt --available-updates --output txt` |
| Show security warnings | `jenkins-plugin-cli --plugin-file plugins.txt --view-security-warnings` |
| Target a specific core version | `jenkins-plugin-cli --jenkins-version 2.492.3 --plugin-file plugins.txt` |
| Force upgrade on existing home | `-e PLUGINS_FORCE_UPGRADE=true` (ref plugins overwrite `JENKINS_HOME/plugins`) |
| Disable a broken plugin offline | `touch $JENKINS_HOME/plugins/<name>.jpi.disabled`, restart |
| Remove a plugin offline | delete `<name>.jpi` and the `<name>/` folder, restart |
| Export installed list | See [script console](#script-console-snippets) or `jcli list-plugins` |

Rules: pin versions, upgrade plugins and core on a staging controller first, read `Manage Jenkins > Security warnings`, and remove unused plugins. The plugin BOM (`io.jenkins.tools.bom`) is the source of tested-together versions for plugin developers and library tests.

---

## Script console snippets

`Manage Jenkins > Script Console` (`$J/script`) runs Groovy on the controller with full privileges. Admin only; every snippet below can damage a controller if mistyped.

```groovy
// Jenkins and Java version
println "Jenkins ${Jenkins.VERSION}, Java ${System.getProperty('java.version')}"

// Installed plugins in plugins.txt format
Jenkins.instance.pluginManager.plugins
  .sort { it.shortName }
  .each { println "${it.shortName}:${it.version}" }

// Running builds everywhere
Jenkins.instance.getAllItems(Job).each { job ->
  job.builds.findAll { it.isBuilding() }.each { println "${job.fullName} #${it.number}" }
}

// Force-kill a stuck pipeline build (try the UI abort and doTerm() first)
def run = Jenkins.instance.getItemByFullName('team-a/app/main').getBuildByNumber(42)
run.doKill()

// Clear the build queue
Jenkins.instance.queue.clear()

// Offline agents and their reason
Jenkins.instance.computers.findAll { it.offline }.each {
  println "${it.name}: ${it.offlineCause}"
}

// Delete builds older than #100 for one job
def job = Jenkins.instance.getItemByFullName('team-a/legacy-job')
job.builds.findAll { it.number < 100 }.each { it.delete() }

// Disable every job in a folder (multibranch children excluded)
Jenkins.instance.getItemByFullName('team-a').getAllJobs()
  .findAll { it instanceof jenkins.model.ParameterizedJobMixIn.ParameterizedJob }
  .each { it.makeDisabled(true) }

// List credential IDs and types at the system store (not values)
com.cloudbees.plugins.credentials.CredentialsProvider
  .lookupCredentials(com.cloudbees.plugins.credentials.common.StandardCredentials, Jenkins.instance, null, null)
  .each { println "${it.id}  ${it.class.simpleName}" }

// Pending script approvals (review before approving anything)
def sa = org.jenkinsci.plugins.scriptsecurity.scripts.ScriptApproval.get()
sa.pendingSignatures.each { println it.signature }

// Reload JCasC
io.jenkins.plugins.casc.ConfigurationAsCode.get().configure()

// Prepare for maintenance: stop accepting new builds
Jenkins.instance.doQuietDown()
```

Run the same scripts non-interactively with `jcli groovy = < script.groovy` or the `/scriptText` endpoint. For repeatable startup changes, put `.groovy` files in `JENKINS_HOME/init.groovy.d/`, but prefer JCasC.

---

## JENKINS_HOME layout

| Path | Contents | Back up? |
|------|----------|----------|
| `config.xml` | Global config (security, views, clouds when not JCasC) | Yes |
| `jobs/<folder>/jobs/<job>/config.xml` | Job definitions | Yes |
| `jobs/.../builds/<n>/` | `build.xml`, `log`, test results, archived artifacts | Yes (or accept loss) |
| `credentials.xml` | System-scope credentials (encrypted) | Yes |
| `secrets/master.key`, `secrets/hudson.util.Secret` | Keys that decrypt all credentials | Yes, separately and securely |
| `plugins/` | `.jpi` files and exploded folders | No if the image pins plugins |
| `nodes/` | Static agent definitions | Yes |
| `users/` | Local users, API tokens | Yes |
| `workspace/` | Workspaces of builds that ran on the controller | No |
| `caches/` | Git and library caches | No |
| `init.groovy.d/` | Startup scripts | In Git |
| `logs/`, `*.log` | Controller and task logs | Optional |

```bash
tar --exclude='./workspace' --exclude='./caches' --exclude='./plugins/*/' \
    -czf jenkins-home-$(date +%F).tgz -C /var/jenkins_home .
```

Useful URLs: `/manage`, `/systemInfo`, `/log/all`, `/threadDump`, `/load-statistics`, `/computer/`, `/script`, `/prometheus` (Prometheus plugin), `/safeRestart`, `/restart`.

---

## Common errors

| Error | Fix |
|-------|-----|
| `java.io.NotSerializableException: java.util.regex.Matcher` | Do not keep non-serializable objects in pipeline variables; move logic into a `@NonCPS` method returning plain types |
| `Scripts not permitted to use method ... (RejectedAccessException)` | Move code into a trusted shared library, or have an admin review it in In-process Script Approval |
| `WorkflowScript: 12: Expected a step @ line 12` | Groovy code directly in `steps`; wrap it in `script { }` |
| `No such DSL method 'readJSON' found among steps` | Plugin missing (Pipeline Utility Steps) or typo |
| `Method code too large!` | Jenkinsfile exceeds the JVM 64 KB method limit; move stages into shared library steps |
| `There are no nodes with the label 'docker'` | Label typo or no online agent with that label |
| `Waiting for next available executor on 'linux'` | All executors busy or agents offline; check `/computer/` and cloud caps |
| `Agent went offline during the build` / `ChannelClosedException` | Pod evicted or node lost (spot, OOM); use `retry(conditions: [kubernetesAgent()])` and right-size requests |
| `ERROR: Couldn't find any revision to build` | Branch spec does not match any branch; check refspec and branch name |
| `Host key verification failed` (git over SSH) | Configure Git Host Key Verification in Security settings (Known hosts file or manually provided keys) |
| `No valid crumb was included in the request` (403) | Use an API token instead of a password, or send the crumb with the session cookie |
| `java.lang.OutOfMemoryError: Java heap space` on the controller | Raise `-Xmx`, discard old builds, remove heavy plugins, check for huge logs/archives |
| `Pipeline ... Resuming build ... Waiting for reconnection of <agent>` after restart | Agent pod is gone; abort the build, then use durable task settings and `retry` |
| `Kubernetes ... pod ... is in phase Pending` / `ImagePullBackOff` | Image name, pull secret, node capacity or quotas in the agent namespace |
| `waitForQualityGate` hangs | SonarQube webhook to `$J/sonarqube-webhook/` missing or unreachable; always wrap in `timeout` |

---

## Top 15 interview one-liners

1. **Controller vs agent:** the controller schedules, stores config, credentials and build records; agents run builds. Production controllers have 0 executors.
2. **Declarative vs scripted:** declarative is a validated structure (`pipeline`, `stages`, `post`, `when`) with restart-from-stage; scripted is free Groovy. Write declarative, push logic into shared libraries.
3. **Where the Jenkinsfile runs:** the Groovy executes on the controller in the CPS engine; only steps like `sh` run on the agent.
4. **CPS:** pipeline code is transformed so its state can be serialized and resumed after a restart, which is why variables must be `Serializable` and why `@NonCPS` exists.
5. **`@NonCPS`:** plain Groovy for pure functions; it cannot call steps and must return serializable values.
6. **Sandbox and script approval:** untrusted pipeline code runs in a Groovy sandbox; global shared libraries are trusted and bypass it, so treat them like production code.
7. **Shared library layout:** `vars/` for global steps with `call()`, `src/` for classes, `resources/` for files via `libraryResource`; load with `@Library('lib@tag') _`.
8. **Credentials:** bind with `credentials()` or `withCredentials`, single-quote shell strings, scope to folders, and prefer short-lived identities over stored keys.
9. **Multibranch:** scans a repo and creates one job per branch/PR/tag with a Jenkinsfile; webhooks trigger indexing, periodic scans are the safety net.
10. **`input` without wasting executors:** put `input` at stage level with `agent none` so nothing is held while waiting, and wrap it in a `timeout`.
11. **`stash` vs `archiveArtifacts`:** stash moves small files between agents within one build; archive keeps outputs on the controller after the build. Big artifacts go to Nexus/Artifactory/S3.
12. **JCasC:** the controller's configuration as YAML in Git, applied at startup; with a plugin-pinned image and Job DSL seeds the controller becomes disposable.
13. **Jenkins on Kubernetes:** the kubernetes plugin creates a pod per build from a pod template, the `jnlp` container connects back, `container('x')` picks the tool container, and the pod is deleted after.
14. **HA:** core Jenkins has no active-active HA; use a fast-recovering single controller on a persistent volume, backups of `JENKINS_HOME` (with `secrets/`), and split load across multiple controllers.
15. **Upgrades:** stay on LTS, pin plugins in `plugins.txt`, test core and plugin upgrades on staging with the same jobs, back up first, and keep the previous image to roll back.

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
