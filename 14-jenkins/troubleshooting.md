# Jenkins — Troubleshooting Drills

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

> Real Jenkins errors and symptoms: likely causes, diagnosis, fix and prevention · 100 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Jenkins](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Jenkinsfile syntax, Groovy & CPS errors](#jenkinsfile-syntax-groovy--cps-errors) — Q1–Q20
- [Agents, executors & the queue](#agents-executors--the-queue) — Q21–Q35
- [SCM, multibranch & webhooks](#scm-multibranch--webhooks) — Q36–Q50
- [Credentials, permissions & script approval](#credentials-permissions--script-approval) — Q51–Q62
- [Plugins, JCasC & Kubernetes agents](#plugins-jcasc--kubernetes-agents) — Q63–Q80
- [Controller health, disk & upgrades](#controller-health-disk--upgrades) — Q81–Q90
- [Integration failures: SonarQube, Nexus, Docker, Trivy, kubectl](#integration-failures-sonarqube-nexus-docker-trivy-kubectl) — Q91–Q100

---

## Jenkinsfile syntax, Groovy & CPS errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. `WorkflowScript: 1: Missing required section "agent" @ line 1, column 1.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The declarative `pipeline {}` block has no top-level `agent`, or it was put inside `stages`.

**Diagnose →**
```bash
curl -s -X POST -u alice:$TOKEN -F "jenkinsfile=<Jenkinsfile" https://jenkins/pipeline-model-converter/validate
```

**Fix →** Add `agent any`, `agent none` (with per-stage agents) or a specific agent directly under `pipeline {`.

**Prevent →** Lint Jenkinsfiles in a pre-commit hook or PR check with the validate endpoint.

</details>

### Q2. `Unknown stage section "step". Starting with version 0.5, steps in a stage must be in a ‘steps’ block.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Typo `step {}` instead of `steps {}`, or steps written directly in the `stage` block.

**Diagnose →** The message includes line and column; open the Jenkinsfile there.

**Fix →**
```groovy
stage('Build') { steps { sh 'make' } }
```

**Prevent →** Use the Declarative Directive Generator and an editor plugin with Jenkinsfile linting.

</details>

### Q3. `java.lang.NoSuchMethodError: No such DSL method 'withSonarQubeEnv' found among steps [archive, bat, build, ...]` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The plugin providing the step (SonarQube Scanner) is not installed or failed to load, or the step name is misspelled.

**Diagnose →**
```bash
curl -s -u admin:$TOKEN "https://jenkins/pluginManager/api/json?depth=1&tree=plugins[shortName,active,enabled]" | jq '.plugins[] | select(.shortName=="sonar")'
```
Check **Manage Jenkins > Plugins > Installed** and startup logs for load failures.

**Fix →** Add `sonar` to `plugins.txt`, rebuild the image, redeploy. Fix spelling if it is a typo.

**Prevent →** Keep a smoke pipeline that calls every step your library relies on, run after each image change.

</details>

### Q4. `java.io.NotSerializableException: java.util.regex.Matcher` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A `Matcher` from `=~` is held in a variable while a step (`sh`, `echo`) runs, and CPS tries to serialize it.

**Diagnose →** The stack trace shows the variable's location in `WorkflowScript` or the library class.

**Fix →**
```groovy
@NonCPS
String extractVersion(String s) {
  def m = (s =~ /version=(\S+)/)
  m ? m[0][1] : null
}
```
Or use `s.find(/version=(\S+)/) { full, v -> v }` and don't keep the matcher.

**Prevent →** Rule in code review: no `Matcher`, `LazyMap` or iterators in pipeline variables.

</details>

### Q5. `java.io.NotSerializableException: groovy.json.internal.LazyMap` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `new JsonSlurper().parseText(...)` returns a `LazyMap`, which is not serializable, and it is alive across a step.

**Diagnose →** Search the Jenkinsfile and library for `JsonSlurper`.

**Fix →** Use Pipeline Utility Steps: `def data = readJSON text: response` (returns serializable `JSONObject`), or `JsonSlurperClassic` inside a `@NonCPS` method returning plain maps.

**Prevent →** Standardize on `readJSON`/`readYaml` in the library.

</details>

### Q6. `org.jenkinsci.plugins.scriptsecurity.sandbox.RejectedAccessException: Scripts not permitted to use method groovy.lang.GroovyObject invokeMethod java.lang.String java.lang.Object` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Sandbox blocks a dynamic method call, often from calling a method on an untyped object in an untrusted library, or a misspelled method name that Groovy resolves dynamically.

**Diagnose →** Look at the line in the stack trace; check **In-process Script Approval** for the pending signature.

**Fix →** Fix the typo or type the variable. Do not approve `invokeMethod` generically; it allows calling any method. Move logic to a trusted library if it legitimately needs it.

**Prevent →** Unit test library code with JenkinsPipelineUnit and run test pipelines in the sandbox.

</details>

### Q7. `groovy.lang.MissingPropertyException: No such property: IMAGE_TAG for class: groovy.lang.Binding` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A variable referenced in Groovy that was never defined, a `def` variable declared in another `script {}` block (local scope), or an env var accessed without `env.`.

**Diagnose →** Find where `IMAGE_TAG` should be set; check whether it was a `def` inside a different block.

**Fix →** Set `env.IMAGE_TAG = ...` and read `env.IMAGE_TAG`, or declare the variable above the `pipeline {}` block.

**Prevent →** Prefer `env.` for cross-stage values; avoid relying on undeclared binding variables.

</details>

### Q8. `WorkflowScript: 14: Environment variable values must either be single quoted, double quoted, or function calls.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An unquoted value or expression in `environment {}`, e.g. `TAG = env.BRANCH_NAME + '-x'` or `COUNT = 3`.

**Diagnose →** Line 14 of the Jenkinsfile.

**Fix →** Use a GString: `TAG = "${env.BRANCH_NAME}-x"`, or compute in a `script {}` block with `env.TAG = ...`.

**Prevent →** Keep `environment` for static strings and `credentials()`; compute dynamic values in a dedicated stage.

</details>

### Q9. `WorkflowScript: 22: Invalid condition "sucess" - valid conditions are [always, changed, fixed, regression, aborted, success, unsuccessful, unstable, failure, notBuilt, cleanup]` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Typo in a `post` condition.

**Diagnose →** The error lists valid names; compare with line 22.

**Fix →** `post { success { ... } }`.

**Prevent →** Linting in PR checks catches it before merge.

</details>

### Q10. `General error during class generation: Method code too large!` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The declarative pipeline compiles to a method exceeding 64 KB of bytecode: too many stages, huge `matrix`, long inline scripts.

**Diagnose →** Count stages and inline script length; recent large additions are the trigger.

**Fix →** Move inline shell to files, extract stages into library steps, enable `SCRIPT_SPLITTING_TRANSFORMATION=true` system property, split into multiple jobs.

**Prevent →** Keep Jenkinsfiles short by policy (template plus parameters).

</details>

### Q11. `expected to call java.util.ArrayList.each but wound up catching org.jenkinsci.plugins.workflow.cps.CpsClosure2.call; see: https://jenkins.io/redirect/pipeline-cps-method-mismatches/` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A CPS-transformed closure is called from non-CPS code: e.g. a closure passed to a Java method, `toString()` override, constructor or `@NonCPS` method invoking a CPS closure.

**Diagnose →** Find the call site from the stack trace; check for closures passed to `sort`, `Comparator`, `inject`, or used inside `@NonCPS` methods.

**Fix →** Replace with a plain `for` loop in CPS code, or make the whole helper `@NonCPS` without calling steps inside.

**Prevent →** Keep `@NonCPS` helpers pure (no steps, no CPS closures).

</details>

### Q12. `hudson.AbortException: script returned exit code 127` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Command not found in the container or agent where `sh` ran: running in `jnlp` instead of the tool container, wrong image, missing `PATH`.

**Diagnose →**
```groovy
sh 'echo $PATH; which node || true; cat /etc/os-release'
```
Check whether the step is inside `container('node')` or a `defaultContainer` is set.

**Fix →** Wrap with `container('node') { }`, set `defaultContainer`, or use an image that has the tool.

**Prevent →** Standard pod templates per stack in the shared library.

</details>

### Q13. `Warning: A secret was passed to "sh" using Groovy String interpolation, which is insecure. Affected argument(s) used the following variable(s): [NEXUS_PSW]` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Double-quoted `sh "... ${NEXUS_PSW} ..."` interpolates the secret in Groovy.

**Diagnose →** The warning names the variable; search for it in double-quoted strings.

**Fix →**
```groovy
sh 'curl -u "$NEXUS_USR:$NEXUS_PSW" https://nexus/...'
```
If you also need Groovy variables, pass them through `withEnv` and keep single quotes.

**Prevent →** Lint rule in library CI: no `${...PSW}` or `${...TOKEN}` in double-quoted `sh`.

</details>

### Q14. A `when { branch 'main' }` stage is always skipped, even on main. The job is a plain Pipeline, not multibranch. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `branch` condition reads `BRANCH_NAME`, which only multibranch jobs set. Plain Pipeline jobs don't have it (`GIT_BRANCH` is `origin/main`).

**Diagnose →**
```groovy
sh 'env | sort | grep -E "BRANCH|GIT_"'
```

**Fix →** Convert to multibranch (recommended), or use `when { expression { env.GIT_BRANCH == 'origin/main' } }`.

**Prevent →** Use multibranch for any repo-driven pipeline.

</details>

### Q15. The build is aborted with `Cancelling nested steps due to timeout` during a test stage that normally takes 5 minutes. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Stage `timeout` hit because tests hung (waiting on a network resource, deadlock) or because the timeout includes waiting for the agent pod to start.

**Diagnose →** Check timestamps (`timestamps()`), when the agent became available, and the last log lines before the timeout; take a thread dump in the test container if reproducible.

**Fix →** Fix the hang (add test-level timeouts, mock external calls). Put `timeout` inside the stage steps (not stage options) if pod startup time should not count.

**Prevent →** Test framework timeouts plus pipeline timeouts as a backstop.

</details>

### Q16. `Scripts not permitted to use method org.jenkinsci.plugins.workflow.support.steps.build.RunWrapper getRawBuild` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pipeline calls `currentBuild.rawBuild` to access the internal `Run` object, which is blocked by the sandbox.

**Diagnose →** Search for `rawBuild` usage.

**Fix →** Use the safe `RunWrapper` API: `currentBuild.getBuildCauses()`, `currentBuild.changeSets`, `currentBuild.previousBuild.result`. Do not approve `getRawBuild`; it exposes the full internal API.

**Prevent →** Provide library steps for common needs (causes, changes, previous result).

</details>

### Q17. Steps after `input` never run, and the log shows `Aborted by anonymous` after a few minutes. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The surrounding `timeout` expired; timeouts abort as SYSTEM/anonymous. Sometimes a reverse proxy breaks the approval POST.

**Diagnose →** Check the timeout around the `input` and the time the build waited. Check browser dev tools if users report clicking "Proceed" with no effect.

**Fix →** Increase the timeout to match the approval SLA; notify approvers when the gate opens (Slack with the build URL).

**Prevent →** Explicit timeout and notification in the template's approval stage.

</details>

### Q18. `ERROR: Unable to find Jenkinsfile` / `Lightweight checkout support not available, falling back to full checkout.` followed by a failure 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The configured Script Path does not match the file location or case (`jenkinsfile` vs `Jenkinsfile`), or the branch spec contains variables.

**Diagnose →** Check job config **Script Path** and `git ls-files | grep -i jenkinsfile` on the branch.

**Fix →** Correct the path/case; use a literal branch spec or multibranch.

**Prevent →** Standard `Jenkinsfile` at repo root, or a documented path per monorepo service.

</details>

### Q19. `java.lang.IllegalArgumentException: Unsupported parameter type for sh` or `No signature of method: sh() is applicable for argument types: (java.lang.String, java.lang.Boolean)` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong step argument style: positional arguments mixed with named ones, e.g. `sh('ls', true)`.

**Diagnose →** Check the Snippet Generator for the step's parameter names.

**Fix →** Use named parameters: `sh(script: 'ls', returnStdout: true)`.

**Prevent →** Generate unfamiliar step syntax from the Snippet Generator.

</details>

### Q20. A library class call fails with `unable to resolve class com.devopsshack.ImageTag` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Library not loaded before the import, wrong package path (`src/com/devopsshack/ImageTag.groovy` must declare `package com.devopsshack`), library loaded with the dynamic `library` step (no compile-time import), or a typo.

**Diagnose →** Check the log for `Loading library shack-lib@v1` and browse the library tag for the file path.

**Fix →** Use `@Library('shack-lib@v1') import com.devopsshack.ImageTag` at the top, align package declaration and folders, or access via `library('shack-lib@v1').com.devopsshack.ImageTag`.

**Prevent →** JenkinsPipelineUnit tests that load classes from `src/`.

</details>

## Agents, executors & the queue

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q21. Build stuck: `Still waiting to schedule task. ‘Jenkins’ doesn’t have label ‘docker’` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No node or pod template has the label `docker`; the label is misspelled; the agent with that label was removed.

**Diagnose →**
```bash
curl -s -u admin:$TOKEN "https://jenkins/computer/api/json?tree=computer[displayName,offline,assignedLabels[name]]" | jq
```
Also check **Manage Jenkins > Clouds** pod template labels.

**Fix →** Add the label to the right node or pod template, or fix the Jenkinsfile label expression.

**Prevent →** Keep labels documented and managed in JCasC.

</details>

### Q22. `Waiting for next available executor on ‘linux’` for 20 minutes during peak hours 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** All executors on matching nodes are busy, or the cloud has hit its concurrency cap.

**Diagnose →** Check **Build Executor Status** and Load Statistics for the label; for clouds, check the cap versus running pods: `kubectl -n jenkins-agents get pods | wc -l`.

**Fix →** Add capacity (more agents or higher cloud cap and quota), reduce waste (`abortPrevious`, skip unchanged), stagger cron jobs.

**Prevent →** Alert on queue wait time and review capacity monthly.

</details>

### Q23. `All nodes of label ‘windows’ are offline` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Windows agent service stopped, VM down, agent secret changed, or network/proxy blocking the connection.

**Diagnose →** Node page shows the offline reason; on the VM check the service status and `<workDir>\remoting\logs\remoting.log.0`.

**Fix →** Restart the agent service; if the secret changed (node recreated), update it in the service config; open the connection path (WebSocket over 443).

**Prevent →** Monitor agent online status and auto-restart the service.

</details>

### Q24. `hudson.remoting.ChannelClosedException: Channel "hudson.remoting.Channel@6e4b:agent-07": Remote call on agent-07 failed. The channel is closing down or has closed down` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The agent process died (OOM-killed, pod evicted, VM rebooted), or the network connection dropped (idle timeout on a load balancer or proxy).

**Diagnose →** Check the agent host: `dmesg | grep -i kill`, `kubectl describe pod` (OOMKilled, Evicted), proxy timeouts; agent remoting logs.

**Fix →** Give the agent/pod more memory, fix eviction causes (ephemeral storage limits), raise LB idle timeouts or rely on WebSocket ping.

**Prevent →** `retry(conditions: [agent(), kubernetesAgent()])` around stages to survive occasional agent loss.

</details>

### Q25. Inbound agent fails: `SEVERE: http://jenkins:8080/tcpSlaveAgentListener/ is invalid: 404 Not Found` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The TCP agent port is disabled on the controller, and the agent was started without `-webSocket`, or the URL is wrong (missing context path).

**Diagnose →** Check `Manage Jenkins > Security > Agents` (TCP port) and the agent command line.

**Fix →** Start the agent with `-webSocket` (preferred), or enable the TCP port (fixed 50000) and open it.

**Prevent →** Standardize on WebSocket agents in all templates and docs.

</details>

### Q26. Agent connects but builds fail with `java.io.IOException: Cannot run program "nohup" (in directory "C:\jenkins\workspace\app"): CreateProcess error=2` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A `sh` step is running on a Windows agent.

**Diagnose →** Check which node ran the step (`NODE_NAME`) and the label used.

**Fix →** Use `bat` or `powershell` on Windows, or branch with `if (isUnix()) { sh ... } else { bat ... }`; fix label expressions so Linux steps go to Linux agents.

**Prevent →** Explicit OS labels on every agent and in every pipeline.

</details>

### Q27. `Building on the built-in node can be a security issue. You should set up distributed builds.` warning, and some jobs still run there 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The built-in node still has executors; jobs with `agent any` land on it.

**Diagnose →** **Nodes > Built-In Node > Configure** shows executors; check recent builds' `NODE_NAME`.

**Fix →** Set `numExecutors: 0` in JCasC, and ensure there are agents for jobs that used `agent any`.

**Prevent →** JCasC enforces it on every restart.

</details>

### Q28. Builds on a static agent fail with `No space left on device` in the workspace 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Workspaces from hundreds of branch jobs and `@tmp`/`@2` copies accumulate; Docker images and build caches fill the disk.

**Diagnose →**
```bash
df -h /home/jenkins
du -sh /home/jenkins/agent/workspace/* | sort -h | tail
docker system df
```

**Fix →** Delete old workspaces, `docker system prune -af --filter until=72h`, add `cleanWs()` in `post { cleanup {} }`.

**Prevent →** Disk space monitor threshold on the node (marks it offline), ephemeral agents.

</details>

### Q29. Node marked offline: `Disk space is below threshold of 1.00 GiB. Only 812.33 MiB out of 1.00 GiB left on /tmp.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The node monitor found low free space on the temp directory or workspace partition, and took the node offline to protect builds.

**Diagnose →** `df -h /tmp` on the agent; find large files with `du -xh /tmp | sort -h | tail`.

**Fix →** Clean up `/tmp` (stale build tool caches, test outputs), then **Bring this node back online**. Point `java.io.tmpdir` to a larger disk if needed.

**Prevent →** Cron cleanup on static agents; prefer ephemeral agents.

</details>

### Q30. A pipeline stage shows `Waiting for agent to connect (30/100): shack-app-main-12-x7k2p-9q8wz` and eventually fails 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The pod started but the `jnlp` container cannot reach the controller (wrong Jenkins URL, NetworkPolicy, DNS), or the pod is stuck Pending/ImagePullBackOff.

**Diagnose →**
```bash
kubectl -n jenkins-agents get pod shack-app-main-12-x7k2p-9q8wz -o wide
kubectl -n jenkins-agents describe pod shack-app-main-12-x7k2p-9q8wz
kubectl -n jenkins-agents logs shack-app-main-12-x7k2p-9q8wz -c jnlp
```

**Fix →** Correct `jenkinsUrl` (e.g. `http://jenkins.jenkins.svc.cluster.local:8080/`), enable WebSocket, allow egress to the Jenkins service in NetworkPolicy, fix image names.

**Prevent →** Smoke pipeline after any cloud or network change.

</details>

### Q31. Builds randomly fail with `Agent shack-app-pr-7-abc was deleted; cancelling node body` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The pod was deleted underneath the build: node scale-down by the cluster autoscaler, eviction due to memory/ephemeral storage, spot/preemptible node reclaim, or `activeDeadlineSeconds`.

**Diagnose →** `kubectl get events -n jenkins-agents --sort-by=.lastTimestamp`, autoscaler logs, node termination events.

**Fix →** Add `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` annotation to agent pods, set proper requests/limits, avoid spot nodes for long builds or add `retry(conditions: [kubernetesAgent()])`.

**Prevent →** Separate node pool for agents with tuned autoscaler settings.

</details>

### Q32. Builds are stuck in the queue with `There are no nodes with the label ‘maven’` but the pod template with label `maven` exists. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The cloud is at its concurrency limit, the template's instance cap is reached, the cloud's connection to the API server is failing, or the template is restricted to certain jobs/folders.

**Diagnose →** **Manage Jenkins > Clouds > kubernetes > Test Connection**; controller log for `KubernetesCloud` errors; count running pods vs caps.

**Fix →** Fix credentials/API connectivity, raise caps, remove unintended restrictions.

**Prevent →** Alert on Kubernetes plugin provisioning errors.

</details>

### Q33. A static agent takes builds then fails `java.lang.UnsupportedClassVersionError: hudson/remoting/Launcher has been compiled by a more recent version of the Java Runtime (class file version 61.0)` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The agent runs an older Java (11 = class file 55) while `agent.jar` requires Java 17 (61).

**Diagnose →** `java -version` on the agent and the Java path used by the launcher (SSH launcher's JavaPath setting).

**Fix →** Install JDK 17 or 21 on the agent and point the launcher at it; build tools can still use another JDK via `tools` or `JAVA_HOME` in the build.

**Prevent →** Bake agent images/VMs with the supported Java; check before core upgrades.

</details>

### Q34. SSH agent launch fails: `[SSH] Opening SSH connection to 10.0.4.12:22. ... Server rejected the 1 private key(s) for jenkins (credentialId:agent-ssh/method:publickey)` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Public key not in the agent user's `authorized_keys`, wrong user, bad file permissions, or key type disabled by the server.

**Diagnose →** On the agent: `ls -la ~jenkins/.ssh`, `journalctl -u ssh` for the rejection reason; test from the controller with the same key.

**Fix →** Add the key, fix permissions (`700 ~/.ssh`, `600 authorized_keys`), use an ed25519 or RSA key the server accepts.

**Prevent →** Provision agents with configuration management and keep the key in JCasC credentials.

</details>

### Q35. SSH agent: `Host key verification failed.` or `No entry currently exists in the Known Hosts file for this host` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Host key verification strategy is "Known hosts file" and the agent's key is not in the controller's `known_hosts`, or the agent was rebuilt with a new host key.

**Diagnose →** Check the node's Host Key Verification Strategy and `~jenkins/.ssh/known_hosts` on the controller.

**Fix →** Add the new key (`ssh-keyscan -t ed25519 10.0.4.12 >> known_hosts` after verifying it out of band) or use "Manually provided key" in the node config.

**Prevent →** Manage host keys as code; do not use "Non verifying" strategy.

</details>

## SCM, multibranch & webhooks

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q36. `ERROR: Couldn't find any revision to build. Verify the repository and branch configuration for this job.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The branch specifier does not match any branch (`*/master` on a repo that uses `main`), the refspec excludes it, or the credential cannot see the repo.

**Diagnose →** Run `git ls-remote https://github.com/org/repo.git` with the same credential; compare with the job's branch spec.

**Fix →** Set the branch spec to `*/main` (or use multibranch so branches are discovered).

**Prevent →** Avoid single-branch Pipeline jobs for repos; use multibranch.

</details>

### Q37. `stderr: fatal: could not read Username for 'https://github.com': No such device or address` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** HTTPS clone of a private repo without credentials, or the credential ID is not set/visible to the job.

**Diagnose →** Check the SCM config credential field and whether the credential is in a folder the job can access.

**Fix →** Select a valid credential (GitHub App or username/token) in the SCM or branch source config.

**Prevent →** Define branch sources with credentials in Job DSL so none are missing.

</details>

### Q38. Checkout fails with `No ED25519 host key is known for github.com and you have requested strict checking.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SSH clone and the Git Host Key Verification setting is "Known hosts file" but GitHub's key is not in `known_hosts` of the agent/controller.

**Diagnose →** **Manage Jenkins > Security > Git Host Key Verification Configuration**.

**Fix →** Use "Manually provided keys" with GitHub's published keys (from `https://api.github.com/meta`), or switch to HTTPS with a GitHub App.

**Prevent →** Set the strategy and keys in JCasC (`security.gitHostKeyVerificationConfiguration`).

</details>

### Q39. GitHub webhook deliveries show `302` or `404` and builds do not start 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing trailing slash (`/github-webhook` redirects), wrong path (`/github-webhook/` is for the GitHub plugin; generic webhooks use different endpoints), or the reverse proxy rewrites the request.

**Diagnose →** GitHub repo **Settings > Webhooks > Recent Deliveries**: response code and body.

**Fix →** Payload URL `https://jenkins.example.com/github-webhook/` with trailing slash, content type `application/json`; check the proxy forwards POST bodies.

**Prevent →** Manage webhooks via an org-level hook or GitHub App with a documented URL.

</details>

### Q40. Webhook deliveries return `200` but nothing builds 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Jenkins received the event but no job matches: repository URL in the job differs (`.git` suffix, SSH vs HTTPS, different case), the job is a plain Pipeline without "GitHub hook trigger for GITScm polling", or the event type is not handled.

**Diagnose →** **Manage Jenkins > System Log**, add a recorder for `org.jenkinsci.plugins.github` and `jenkins.scm.api` at FINE; check the multibranch **Events** log.

**Fix →** Make the job's repo URL match; enable the trigger for non-multibranch jobs; ensure GitHub server config in Jenkins matches the API URL.

**Prevent →** Prefer multibranch with GitHub Branch Source where event matching is automatic.

</details>

### Q41. Multibranch scan log: `‘Jenkinsfile’ not found` and `Does not meet criteria` for a branch that has a Jenkinsfile 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong case or path, the file was added after the last scan, the credential lacks access to file contents, or a branch filter excludes it.

**Diagnose →** Read **Scan Repository Log**; confirm path with `git ls-tree -r origin/<branch> --name-only | grep -i jenkinsfile`.

**Fix →** Correct the Script Path or file name; re-run **Scan Repository Now**.

**Prevent →** Webhooks for push events so new files trigger re-indexing.

</details>

### Q42. `GitHub API Usage: Current quota for Github API usage has 4 remaining (1 over budget). Next quota of 5000 in 38 min. Sleeping for 37 min.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Many multibranch scans with a personal token, frequent periodic scans, organization folders scanning hundreds of repos.

**Diagnose →** Scan logs show rate limit messages; check scan intervals across folders.

**Fix →** Switch to a GitHub App credential (higher, per-installation limits), reduce periodic scans to daily, rely on webhooks, choose the "Throttle at/near rate limit" API usage strategy.

**Prevent →** Credential choice and scan intervals standardized in Job DSL.

</details>

### Q43. PR builds show the wrong code: a fix pushed to the target branch is missing in PR merge builds 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The merge build uses the target revision recorded at indexing time; the PR was not re-indexed after the target changed, or "Discover pull requests" uses the head strategy.

**Diagnose →** Check build log for `Merging remotes/origin/main commit <sha>` and compare with the current `main` head.

**Fix →** Re-run the PR build after a scan; enable rebuilding PRs when the target branch changes (GitHub Branch Source can trigger on target changes if events are delivered) or rebase the PR.

**Prevent →** Merge queue or require branches to be up to date in branch protection.

</details>

### Q44. Branch jobs disappear unexpectedly after a scan 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The orphaned item strategy removed them because the scan temporarily failed to list branches (expired credential, API outage) or a filter change excluded them.

**Diagnose →** Scan log for errors and `Will remove ...` lines; check credential validity.

**Fix →** Fix the credential/filter and re-scan; restored branches recreate jobs, but build history of deleted jobs is gone unless restored from backup.

**Prevent →** Abort scans on errors (default behaviour for most sources), keep backups, monitor credential expiry.

</details>

### Q45. `ERROR: Could not find any definition of libraries [shack-lib]` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The library name in `@Library` does not match any configured global or folder library, or the job is outside the folder where the library is defined.

**Diagnose →** Check **Manage Jenkins > System > Global Trusted Pipeline Libraries** and the folder's library config.

**Fix →** Configure the library (JCasC `unclassified.globalLibraries`) or fix the name.

**Prevent →** Library configuration in JCasC; document the exact name.

</details>

### Q46. `ERROR: Version override not permitted for library shack-lib` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Jenkinsfile requests `@Library('shack-lib@feature-x')` but the library config has "Allow default version to be overridden" disabled.

**Diagnose →** Library config in Manage Jenkins or JCasC (`allowVersionOverride: false`).

**Fix →** Use the default version, or test the branch via a folder-level library or staging controller.

**Prevent →** Document how to test library branches safely.

</details>

### Q47. `ERROR: Library shack-lib expected to contain at least one of src or vars directories` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The retriever points to the wrong repo, branch or subdirectory; the library files are nested (e.g. under `jenkins/`) without setting the library path.

**Diagnose →** Browse the configured version in Git and check top-level folders.

**Fix →** Fix the repo URL/version, or set **Library Path** (`libraryPath: 'jenkins/'` in `modernSCM`) for libraries in a subfolder.

**Prevent →** Keep libraries in dedicated repos with `vars/` and `src/` at the root.

</details>

### Q48. Builds start twice for each push: once as `main` and once as a pull request, plus once from polling 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `pollSCM` trigger in the Jenkinsfile plus webhooks, and branch discovery set to "All branches" with PR discovery.

**Diagnose →** Build causes (`Started by an SCM change`, `Branch indexing`, `Push event to branch main`).

**Fix →** Remove `pollSCM` when webhooks work; set branch discovery to exclude branches filed as PRs.

**Prevent →** Template-level conventions and review for triggers.

</details>

### Q49. `hudson.plugins.git.GitException: Command "git fetch --tags --force --progress -- https://github.com/org/big-repo.git +refs/heads/*:refs/remotes/origin/*" returned status code 128: ... timeout after 10 minutes` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Huge repository (history, binaries) or slow network; default 10-minute clone/fetch timeout.

**Diagnose →** Repo size (`git count-objects -vH` on a local clone), network throughput from agents.

**Fix →** Use shallow clone (`cloneOption(shallow: true, depth: 1, noTags: true, timeout: 30)`), narrow refspec to the branch, reference repositories on static agents, Git LFS for binaries.

**Prevent →** Repo hygiene (no binaries in Git) and a mirror close to the agents.

</details>

### Q50. Commit statuses never appear on GitHub PRs, even though builds pass 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The credential lacks permission to write statuses (PAT without `repo:status`, GitHub App without "Commit statuses: write" or "Checks: write"), or the job uses a plain Git SCM instead of GitHub Branch Source.

**Diagnose →** Controller log for `GitHubCommitStatusSetter` or `Could not update commit status` errors.

**Fix →** Grant the permission to the App/token, or use GitHub Branch Source; add `githubNotify` explicitly if needed.

**Prevent →** Use a GitHub App with documented permissions for all orgs.

</details>

## Credentials, permissions & script approval

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. `ERROR: Could not find credentials entry with ID 'dockerhub'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ID is wrong, the credential is in a different folder, or it is System-scoped (not usable by jobs).

**Diagnose →** Check credentials at global and folder levels; verify scope is Global.

**Fix →** Move or recreate the credential in the job's folder (or a parent) with Global scope; correct the ID.

**Prevent →** Define credentials in JCasC with consistent IDs documented in the library.

</details>

### Q52. API call returns `HTTP ERROR 403 No valid crumb was included in the request` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** POST with username + password (session-based) without a crumb, or crumb obtained from a different session (no cookie jar), or a proxy changing client IP when crumbs are tied to IP.

**Diagnose →** Check whether the script uses an API token; inspect whether cookies are kept between the crumb and POST requests.

**Fix →** Use an API token (no crumb needed), or fetch the crumb with `-c cookies` and send with `-b cookies -H "Jenkins-Crumb: ..."`.

**Prevent →** Standardize automation on API tokens for service users.

</details>

### Q53. `Access Denied: alice is missing the Job/Build permission` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Authorization strategy does not grant the permission to Alice or her groups on that job/folder; group names from SSO do not match the configured names.

**Diagnose →** Check `https://jenkins/whoAmI/` while logged in as Alice (shows authorities/groups); compare with the matrix.

**Fix →** Grant the permission to the right group at the folder level, matching exact group names (case-sensitive for some realms).

**Prevent →** Manage authorization in JCasC with group names verified against the IdP.

</details>

### Q54. After enabling SSO, everyone sees `alice is missing the Overall/Read permission` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Permissions were granted to local users or to groups that the new realm does not provide; group claim not configured in the OIDC/SAML settings.

**Diagnose →** `/whoAmI/` shows the authorities from the new realm; check the IdP token claims.

**Fix →** Configure the groups claim (e.g. `groups`) in the realm; grant `Overall/Read` to `authenticated` and real permissions to IdP groups. Use the break-glass admin to fix if locked out.

**Prevent →** Test realm changes on staging with real accounts first.

</details>

### Q55. Locked out: SSO is down and nobody can log in to Jenkins to fix the realm 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No break-glass account; realm misconfiguration applied.

**Diagnose →** Confirm IdP status; check controller logs for realm errors.

**Fix →** If JCasC manages security, fix the YAML (or point to a temporary local realm) and restart/reload. Otherwise, stop Jenkins, edit `config.xml` setting `<useSecurity>` to a temporary local realm, start, fix, and revert. As a last resort, a Groovy script in `init.groovy.d/` can set a realm at startup.

**Prevent →** Break-glass local admin in a vault, JCasC-managed security, tested quarterly.

</details>

### Q56. A pipeline using `withCredentials` prints `****` in places where the secret is not expected, breaking output parsing 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The secret value is a short or common string (e.g. `admin`, `1234`) and masking replaces every occurrence in the log.

**Diagnose →** Check the secret's value length/content.

**Fix →** Use strong, unique secrets; parse command output from files instead of the console log.

**Prevent →** Credential policies requiring long random secrets.

</details>

### Q57. `org.jenkinsci.plugins.scriptsecurity.sandbox.RejectedAccessException: Scripts not permitted to use staticMethod org.codehaus.groovy.runtime.DefaultGroovyMethods execute java.lang.String` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pipeline code uses `"ls".execute()` to run a process; in pipelines this runs on the controller, not the agent.

**Diagnose →** Search for `.execute()`.

**Fix →** Replace with `sh(script: 'ls', returnStdout: true)`. Never approve this signature: it allows arbitrary commands on the controller.

**Prevent →** Educate pipeline authors that Groovy runs on the controller; review approvals list.

</details>

### Q58. A job using a credential works when triggered by an admin but fails with `CredentialsNotFound`-style errors when triggered by an upstream job or a normal user 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Authorize Project plugin runs builds as the triggering user, who lacks `Credentials/UseItem` on that credential; or the credential is user-scoped (stored in the admin's personal store).

**Diagnose →** Job's **Authorization** setting and the credential's store (user vs folder).

**Fix →** Move the credential to the folder store; grant `Credentials/UseItem` to the group that runs the job, or configure the job to run as a specific service identity.

**Prevent →** Never use personal credential stores for shared jobs.

</details>

### Q59. Users can no longer see job descriptions as formatted HTML after a security update 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Markup formatter changed to Plain text (or HTML sanitized by Safe HTML).

**Diagnose →** **Manage Jenkins > Security > Markup Formatter**.

**Fix →** Use "Safe HTML" (OWASP Markup Formatter) for basic formatting; do not revert to raw HTML.

**Prevent →** Set the formatter in JCasC (`jenkins.markupFormatter: owaspMarkupFormatter`).

</details>

### Q60. Replay button is missing for developers who used to have it 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `Run/Replay` permission removed (intentional hardening), or the build was from a pipeline type that does not support replay.

**Diagnose →** Check the authorization matrix for `Run/Replay` on the folder.

**Fix →** Explain the policy; offer a test folder with Replay allowed for experiments, or use PR branches.

**Prevent →** Document permission changes in the platform changelog.

</details>

### Q61. Agents fail to connect with `Rejected connection: ... is not authorized to connect` or the agent secret error `Incorrect secret` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The node was deleted and recreated (new secret), the agent uses a stale secret, or the node name differs from `-name`.

**Diagnose →** Compare the node name and secret on the node's page with the agent's start command.

**Fix →** Update the agent with the current secret and exact name; restart the agent.

**Prevent →** Store agent secrets in a secret store and automate agent registration.

</details>

### Q62. `java.lang.SecurityException: Rejecting unsandboxed script` or `UnapprovedUsageException: script not yet approved for use` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A pipeline configured with "Use Groovy Sandbox" unchecked (inline script) by a non-admin, so the whole script needs admin approval; or Job DSL scripts with script security enabled.

**Diagnose →** Job config sandbox checkbox; pending items in In-process Script Approval.

**Fix →** Enable the sandbox for the job, or have an admin review and approve the whole script (only for trusted content).

**Prevent →** Pipelines from SCM always use the sandbox; admin-only for unsandboxed scripts.

</details>

## Plugins, JCasC & Kubernetes agents

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q63. Startup log: `SEVERE: Failed Loading plugin Pipeline: Groovy v... (workflow-cps)` with `Update required: Pipeline: Supporting APIs (workflow-support) ... to be updated to ... or higher` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A plugin was updated but its dependency was not; plugins installed manually without dependency resolution.

**Diagnose →** Read the full list of failed plugins in the log and **Manage Jenkins** admin monitor.

**Fix →** Rebuild the image with `jenkins-plugin-cli` resolving dependencies (or update the named dependency); restart.

**Prevent →** Never install plugins via UI on production; use `plugins.txt` and CI-built images.

</details>

### Q64. Plugin fails with `Jenkins (2.426.3) or higher required` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The plugin version requires a newer core than the one running.

**Diagnose →** `jenkins-plugin-cli --jenkins-version <your core> --plugin-file plugins.txt --list`.

**Fix →** Pin a plugin version compatible with your core, or upgrade core first.

**Prevent →** Always pass `--jenkins-version` when generating plugin updates.

</details>

### Q65. JCasC fails at startup: `io.jenkins.plugins.casc.ConfiguratorException: Invalid configuration elements for type class jenkins.model.Jenkins : agentProtocols` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The YAML uses an attribute that no longer exists in this version (removed setting) or is misspelled/misplaced.

**Diagnose →** The message names the type and attribute; check the JCasC reference at `/configuration-as-code/reference` on the running version, or export the current config.

**Fix →** Remove or rename the attribute to match the current schema.

**Prevent →** CI that boots the controller image with the JCasC file and fails on `ConfiguratorException`.

</details>

### Q66. `io.jenkins.plugins.casc.UnknownConfiguratorException: No configurator for the following root elements: sonarGlobalConfiguration` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The element is at the wrong level (must be under `unclassified:`), or the plugin that provides it is not installed.

**Diagnose →** Check indentation and the plugin list.

**Fix →**
```yaml
unclassified:
  sonarGlobalConfiguration:
    installations:
      - name: sonarqube
        serverUrl: http://sonarqube.sonarqube.svc:9000
        credentialsId: sonar-token
```

**Prevent →** Use the exported YAML as a template and validate against the JCasC schema.

</details>

### Q67. JCasC credentials contain the literal text `${GITHUB_TOKEN}` instead of the value 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The environment variable or secret file was not available to the controller process, so resolution produced the raw string (JCasC logs a warning), or the value was escaped with `^${...}`.

**Diagnose →** `kubectl -n jenkins exec jenkins-0 -- printenv GITHUB_TOKEN`, check `/run/secrets/GITHUB_TOKEN`, controller log for `Configuration import: Found unresolved variable GITHUB_TOKEN`.

**Fix →** Provide the secret as env (`envFrom: secretRef`) or a file in the secrets dir; restart/reload.

**Prevent →** A startup check that fails if unresolved variables are logged.

</details>

### Q68. Kubernetes plugin: `Failure executing: POST at: https://10.96.0.1/api/v1/namespaces/jenkins-agents/pods. Message: pods is forbidden: User "system:serviceaccount:jenkins:jenkins" cannot create resource "pods" in API group "" in the namespace "jenkins-agents".` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing Role/RoleBinding for the controller's service account in the agent namespace.

**Diagnose →**
```bash
kubectl auth can-i create pods -n jenkins-agents --as system:serviceaccount:jenkins:jenkins
```

**Fix →** Create a Role with pods, pods/exec, pods/log, events permissions in `jenkins-agents` and bind it to `jenkins:jenkins`.

**Prevent →** RBAC in the same Helm chart/manifests as the controller.

</details>

### Q69. Agent pod events: `Failed to pull image "maven:3.9-eclipse-temurin21": manifest unknown` and the build log says `Unable to pull container image "maven:3.9-eclipse-temurin21". Check if image tag name is spelled correctly.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Typo in the image tag (`temurin21` instead of `temurin-21`), private image without pull secret, or registry rate limiting (`toomanyrequests`).

**Diagnose →** `kubectl -n jenkins-agents describe pod <pod>`; try `crane manifest maven:3.9-eclipse-temurin-21`.

**Fix →** Correct the tag; add `imagePullSecrets`; mirror images to an internal registry.

**Prevent →** Central pod templates in the library and a registry mirror.

</details>

### Q70. `process apparently never started in /home/jenkins/agent/workspace/shack-app_main@tmp/durable-3f2a1b9c` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** In a `container()` step, the tool container runs as a different UID than `jnlp` and cannot write to the `@tmp` control directory; the image has no `/bin/sh` or `nohup`; or the workspace path differs (`workingDir` mismatch).

**Diagnose →** Enable launch diagnostics for one run: `-Dorg.jenkinsci.plugins.durabletask.BourneShellScript.LAUNCH_DIAGNOSTICS=true` on the controller; check `id` in both containers.

**Fix →** Run containers with the same UID (pod `securityContext.runAsUser: 1000` and `fsGroup: 1000`), use images with a shell (`:debug` variants), keep the default working directory `/home/jenkins/agent`.

**Prevent →** Base pod template with a consistent security context.

</details>

### Q71. Build fails with `Container [maven] was terminated (Exit Code: 137, Reason: OOMKilled)` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The container's memory limit is lower than the JVM or test suite needs; JVM heap not bounded within the container limit.

**Diagnose →** `kubectl describe pod` (Last State: OOMKilled); check `MAVEN_OPTS` and container limits.

**Fix →** Raise the memory limit and set `MAVEN_OPTS=-XX:MaxRAMPercentage=75` so the JVM respects it; limit Surefire forks (`-DforkCount=1`).

**Prevent →** Right-size requests/limits per template from observed usage.

</details>

### Q72. Pods stay `Pending` with `0/6 nodes are available: 6 Insufficient cpu.` and builds wait 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Requests too high for available nodes, cluster autoscaler not scaling (max size reached, node group misconfigured), quota limits.

**Diagnose →** `kubectl -n jenkins-agents describe pod`, autoscaler logs, `kubectl describe resourcequota -n jenkins-agents`.

**Fix →** Lower requests to realistic values, increase node group max, fix autoscaler config.

**Prevent →** Capacity alerts and realistic requests in templates.

</details>

### Q73. Agent pod logs (`jnlp`): `java.net.UnknownHostException: jenkins.jenkins.svc.cluster.local` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Service name or namespace wrong, agents running in another cluster where that DNS name does not exist, or DNS blocked by NetworkPolicy (port 53 to kube-dns).

**Diagnose →** `kubectl -n jenkins get svc`; from a debug pod in the agent namespace: `nslookup jenkins.jenkins.svc.cluster.local`.

**Fix →** Correct `jenkinsUrl`; allow DNS egress; for remote clusters use the public HTTPS URL with WebSocket.

**Prevent →** Include DNS in the agents' NetworkPolicy baseline.

</details>

### Q74. A `container('kaniko')` step fails: `sh: can't access tty; job control turned off` then `/busybox/sh: kubectl: not found`, or `ERROR: Process exited immediately after creation` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The container's main process exits (no `sleep`/`cat` command), or a distroless image lacks a shell; the wrong container was targeted.

**Diagnose →** Check pod status (container terminated), the image entrypoint, and the container name in `container()`.

**Fix →** Use `gcr.io/kaniko-project/executor:debug` (includes busybox shell) with `command: ["sleep"]`, `args: ["infinity"]`; target the correct container for each tool.

**Prevent →** Reusable templates with a known-good command for each tool image.

</details>

### Q75. JCasC reload wiped out a manually added cloud and all builds are stuck 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** JCasC replaces the `clouds` list with what the YAML contains; the manual cloud was not in Git.

**Diagnose →** Compare JCasC YAML and the previous UI config (Job Configuration History for system config).

**Fix →** Add the cloud to JCasC and reload.

**Prevent →** No manual config on JCasC-managed controllers; announce this rule in the system message.

</details>

### Q76. After upgrading the kubernetes plugin, pod templates defined with `containerTemplate` fail with `ERROR: Invalid pod template` or fields are ignored 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Deprecated fields removed or behaviour changed between plugin versions; YAML merge strategy defaults changed.

**Diagnose →** Plugin changelog, controller logs at FINE for `org.csanchez.jenkins.plugins.kubernetes`, the generated pod YAML (printed at build start in the log).

**Fix →** Migrate to raw `yaml` pod specs (most stable interface), set `yamlMergeStrategy` explicitly.

**Prevent →** Upgrade plugins on staging with smoke pipelines covering each template.

</details>

### Q77. Job DSL in JCasC fails: `ERROR: (script, line 3) No signature of method: javaposse.jobdsl.dsl.helpers.workflow.BranchSourcesContext.github() is applicable` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Static DSL method not available; GitHub branch source is exposed only through the dynamic DSL (`branchSource { source { github { ... } } }`), or the plugin is missing.

**Diagnose →** Check the Job DSL API viewer on your controller (`/plugin/job-dsl/api-viewer/index.html`) for available methods.

**Fix →**
```groovy
branchSources { branchSource { source { github {
  id('shack-app'); repoOwner('devopsshack'); repository('shack-app')
  repositoryUrl('https://github.com/devopsshack/shack-app'); configuredByUrl(true)
  credentialsId('github-app')
} } } }
```

**Prevent →** Use the API viewer of your own controller as the reference, not internet examples.

</details>

### Q78. Agents connect but every `sh` step in `container('node')` hangs for minutes before running 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The exec API calls are slow or throttled (API server overloaded, webhook admission controllers on `pods/exec`), or a proxy between controller and API server buffers the exec stream.

**Diagnose →** Time `kubectl exec` manually from the controller pod; check API server latency metrics and admission webhooks.

**Fix →** Fix API server or webhook latency; as a workaround run steps in the `jnlp` container by building a custom agent image with the tools.

**Prevent →** Monitor API server latency; keep exec-heavy pipelines in mind when adding admission webhooks.

</details>

### Q79. The JCasC page shows `Configuration as Code: unable to read /var/jenkins_config/jenkins.yaml` after a ConfigMap change 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ConfigMap key name changed, mount path changed, or YAML syntax error making the file invalid.

**Diagnose →** `kubectl -n jenkins exec jenkins-0 -- ls -la /var/jenkins_config/`; `yamllint jenkins.yaml`.

**Fix →** Align `CASC_JENKINS_CONFIG` with the mounted file; fix the YAML.

**Prevent →** Lint YAML in CI and keep the path in one place in the manifests.

</details>

### Q80. Builds fail with `java.lang.InterruptedException` at random steps and the controller log shows `Pod ... was terminated: Reason: Evicted, The node was low on resource: ephemeral-storage` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The workspace (emptyDir) and container layers fill the node's ephemeral storage; Kubelet evicts pods.

**Diagnose →** `kubectl get events` for evictions; node disk usage.

**Fix →** Set `resources.requests/limits.ephemeral-storage` on build containers, use a PVC or larger node disks for heavy builds, clean caches during the build.

**Prevent →** Size node disks for build workloads; monitor ephemeral storage.

</details>

## Controller health, disk & upgrades

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q81. Controller crashes with `java.lang.OutOfMemoryError: Java heap space` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Heap too small for the workload, too many retained builds, huge pipelines (many steps), giant logs, or a plugin leak.

**Diagnose →** Heap dump from `-XX:+HeapDumpOnOutOfMemoryError` analysed with Eclipse MAT; GC logs; recent plugin changes.

**Fix →** Short term: restart with more heap (`-XX:MaxRAMPercentage=75` and a larger container limit). Then fix the cause: discard old builds, reduce step counts, upgrade/remove the leaking plugin.

**Prevent →** Alert on heap usage after GC above 80%.

</details>

### Q82. Controller pod restarts repeatedly: liveness probe failed: `Get "http://10.0.3.7:8080/login": context deadline exceeded` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Startup takes longer than the probe allows (many jobs, slow disk), or long GC pauses make `/login` unresponsive.

**Diagnose →** `kubectl -n jenkins describe pod jenkins-0` events; controller logs around startup; GC logs.

**Fix →** Add a `startupProbe` with a generous `failureThreshold` (e.g. 60 × 10 s), keep liveness lenient; fix GC pressure.

**Prevent →** Measure startup time after each change; keep probes aligned.

</details>

### Q83. Jenkins UI shows `Jenkins is going to shut down` and new builds are not starting 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Someone (or an upgrade script) triggered Prepare for Shutdown / `quiet-down`.

**Diagnose →** Banner in the UI; audit trail for `/quietDown`.

**Fix →** **Manage Jenkins > Cancel Shutdown** or `curl -X POST -u admin:$TOKEN https://jenkins/cancelQuietDown`.

**Prevent →** Use quiet-down only in maintenance windows with an announcement.

</details>

### Q84. `JENKINS_HOME` volume is 98% full 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Build records and artifacts without retention, orphaned branch jobs, workspaces on the built-in node, heap dumps, logs.

**Diagnose →**
```bash
du -h --max-depth=2 /var/jenkins_home | sort -h | tail -20
find /var/jenkins_home/jobs -name archive -type d -exec du -sh {} + | sort -h | tail
```

**Fix →** Delete old builds via `buildDiscarder` and a one-off cleanup, remove orphaned jobs, delete stale heap dumps; move artifacts to Nexus/S3.

**Prevent →** Retention by default in templates; disk alerts at 80%.

</details>

### Q85. After an upgrade: `Running with Java 11, which is older than the minimum required version (Java 17).` and Jenkins won't start 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** New core requires Java 17+; the host still runs Java 11.

**Diagnose →** `java -version` for the Java used by the service (`systemctl cat jenkins` for `JAVA_HOME`).

**Fix →** Install Java 17 or 21 and point the service at it (`JAVA_HOME` in the systemd override), or use a `jdk21` image tag.

**Prevent →** Read the LTS upgrade guide; upgrade Java first as a separate change.

</details>

### Q86. UI is slow and the thread dump shows dozens of threads `BLOCKED` on `hudson.model.Queue` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Queue maintenance is slow due to a huge queue, expensive label expressions, or a plugin with slow `QueueTaskDispatcher`/`canTake` logic (e.g. throttling plugins evaluating many items).

**Diagnose →** Several thread dumps a few seconds apart; find the thread holding the queue lock and its stack (plugin package names).

**Fix →** Reduce queue size (cancel redundant builds), simplify label logic, update or remove the offending plugin.

**Prevent →** Keep queue-affecting plugins minimal; monitor queue size.

</details>

### Q87. `java.io.IOException: Too many open files` in the controller log 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Low file descriptor limit for the Jenkins process, or a leak from a plugin keeping files/sockets open.

**Diagnose →** `cat /proc/$(pgrep -f jenkins.war)/limits | grep 'open files'` and `ls /proc/<pid>/fd | wc -l`; `lsof -p <pid> | awk '{print $9}' | sort | uniq -c | sort -n | tail`.

**Fix →** Raise limits (`LimitNOFILE=65536` in systemd, container runtime ulimits); identify leaking files from `lsof` and update the plugin.

**Prevent →** Monitor open FDs.

</details>

### Q88. Builds started before a controller restart show `Resuming build at ...` then fail with `Pausing (Preparing for shutdown)` loops or `Could not find any running agent` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The agents those builds used (Kubernetes pods) were deleted or did not reconnect after the restart, so the resumed builds cannot continue.

**Diagnose →** Build log after resume; `kubectl get pods` in agents namespace; whether quiet-down was used before the restart.

**Fix →** Abort the stuck builds and rerun. Use quiet-down before planned restarts.

**Prevent →** Graceful maintenance, `retry(conditions: [kubernetesAgent(), nonresumable()])` for idempotent stages.

</details>

### Q89. After restoring from backup, credentials appear but jobs fail with `hudson.util.Secret` decryption errors and empty passwords 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `secrets/master.key` and `secrets/hudson.util.Secret` were not restored (or came from a different instance), so stored secrets can't be decrypted.

**Diagnose →** Compare `secrets/` with the original backup; controller log for `Failed to decrypt` warnings.

**Fix →** Restore the matching `secrets/` directory and restart. If lost, recreate every credential.

**Prevent →** Back up `secrets/` with the rest of `JENKINS_HOME`, test restores, keep secrets in an external store.

</details>

### Q90. `It appears that your reverse proxy set up is broken.` appears in Manage Jenkins and some links point to `http://localhost:8080` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Jenkins URL not set or wrong; proxy not forwarding `Host`/`X-Forwarded-*` headers.

**Diagnose →** **Manage Jenkins > System > Jenkins URL**; proxy config; `curl -I https://jenkins.example.com/administrativeMonitor/hudson.diagnosis.ReverseProxySetupMonitor/test`.

**Fix →** Set `unclassified.location.url` to the external URL; forward `X-Forwarded-Proto`, `X-Forwarded-Host`, `X-Forwarded-Port`.

**Prevent →** Configure the URL in JCasC and test via the monitor after proxy changes.

</details>

## Integration failures: SonarQube, Nexus, Docker, Trivy, kubectl

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q91. `java.lang.IllegalStateException: Unable to get SonarQube task id and/or server name. Please use the 'withSonarQubeEnv' wrapper to run your analysis.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `waitForQualityGate` called in a build where the analysis did not run inside `withSonarQubeEnv`, the analysis ran on another agent and `report-task.txt` is not found, or analysis failed earlier.

**Diagnose →** Confirm `withSonarQubeEnv('sonarqube')` wraps the scanner; look for `.scannerwork/report-task.txt` (or `target/sonar/report-task.txt` for Maven).

**Fix →** Wrap the scanner call with `withSonarQubeEnv`; run gate in the same build after analysis.

**Prevent →** Library step that does analysis and gate together.

</details>

### Q92. `waitForQualityGate` fails with `Timeout has been exceeded` though SonarQube shows the analysis as complete 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SonarQube webhook to Jenkins missing, wrong URL (needs `/sonarqube-webhook/`), blocked by network, or secret mismatch.

**Diagnose →** SonarQube **Administration > Configuration > Webhooks** delivery history (last response code).

**Fix →** Add the webhook to `https://jenkins.example.com/sonarqube-webhook/`; align the webhook secret with Jenkins config; open network path.

**Prevent →** Include the webhook in SonarQube's configuration as code or provisioning scripts.

</details>

### Q93. Sonar scanner: `ERROR: Not authorized. Analyzing this project requires authentication. Please check the user token in the property 'sonar.token' or the environment variable 'SONAR_TOKEN'.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Token missing, expired or revoked; the Sonar server config in Jenkins points to a wrong credential; token lacks "Execute Analysis" permission for the project.

**Diagnose →** Check the credential ID in the SonarQube server config; test the token with `curl -u $TOKEN: https://sonar/api/authentication/validate`.

**Fix →** Generate a new token with the right permissions, update the Jenkins credential.

**Prevent →** Use a project analysis token or global analysis token owned by a service account, with rotation.

</details>

### Q94. Maven deploy to Nexus: `Failed to deploy artifacts: Could not transfer artifact com.shack:app:jar:1.4.0 from/to maven-releases: status code: 400, reason phrase: Repository does not allow updating assets: maven-releases (400)` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Re-deploying an existing release version; the releases repository disallows redeploy.

**Diagnose →** Check Nexus for the existing artifact; check whether the build re-ran for the same tag.

**Fix →** Bump the version; never overwrite releases. For reruns, skip deploy if the artifact already exists.

**Prevent →** Version from tags, deploy releases only from tag builds, snapshots for branches.

</details>

### Q95. Nexus returns `status code: 401, reason phrase: Unauthorized (401)` during `mvn deploy` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `settings.xml` `<server><id>` does not match the `<repository><id>` in `distributionManagement`, credentials not injected, or wrong password.

**Diagnose →** `mvn -B help:effective-settings -s ci/settings.xml` (passwords masked); compare IDs.

**Fix →** Align IDs; inject credentials via `withCredentials` or Config File Provider with credential mapping.

**Prevent →** Library step that generates `settings.xml` from credentials consistently.

</details>

### Q96. `docker: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?` inside a Kubernetes agent pod 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No Docker daemon exists in the pod (and nodes run containerd), so `docker build` cannot work.

**Diagnose →** `ls -l /var/run/docker.sock` in the container; node runtime via `kubectl get nodes -o wide`.

**Fix →** Use Kaniko or BuildKit for builds in pods; avoid mounting a host socket. If you must use Docker, use a dedicated VM agent with Docker installed.

**Prevent →** Standard image build step in the library based on a daemonless builder.

</details>

### Q97. Kaniko: `error checking push permissions -- make sure you entered the correct tag name, and that you are authenticated correctly, and try again: checking push permission for "docker.io/devopsshack/shack-app:main-1a2b3c4": POST https://index.docker.io/v2/devopsshack/shack-app/blobs/uploads/: UNAUTHORIZED` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing or malformed `/kaniko/.docker/config.json`, wrong registry key (`https://index.docker.io/v1/` for Docker Hub), wrong repo name, or token lacks write scope.

**Diagnose →** `cat /kaniko/.docker/config.json` (in a debug run, never print in normal builds); verify the token can push with `crane auth login` and `crane push`.

**Fix →** Generate config.json with the correct auth:
```bash
AUTH=$(printf '%s:%s' "$REG_USER" "$REG_PASS" | base64 | tr -d '\n')
printf '{"auths":{"https://index.docker.io/v1/":{"auth":"%s"}}}' "$AUTH" > /kaniko/.docker/config.json
```

**Prevent →** Library step that writes the config and validates the repository name.

</details>

### Q98. Trivy: `FATAL Fatal error init error: DB error: failed to download vulnerability DB ... TOOMANYREQUESTS` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Many ephemeral pods downloading the DB from the public registry hit rate limits; no network egress.

**Diagnose →** Error text mentions the DB repository; check egress rules.

**Fix →** Mirror the DB to an internal registry and use `--db-repository registry.example.com/trivy-db` (and `--java-db-repository`), or cache the DB on a PVC with `--cache-dir` and `--skip-db-update` on most builds.

**Prevent →** A scheduled job that refreshes the internal DB mirror.

</details>

### Q99. Deploy stage: `error: You must be logged in to the server (Unauthorized)` from `kubectl` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Expired token in the kubeconfig credential, wrong kubeconfig, or (on EKS) the IAM identity is not mapped in access entries/aws-auth.

**Diagnose →** `kubectl config view --minify`, `kubectl auth whoami` (1.28+), `aws sts get-caller-identity` for EKS.

**Fix →** Refresh the credential; for EKS add an access entry for the role; prefer in-cluster service accounts for deploy pods.

**Prevent →** Avoid static kubeconfigs; use service accounts or workload identity.

</details>

### Q100. `kubectl rollout status` fails: `error: deployment "shack-app" exceeded its progress deadline` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** New pods never become ready: image pull failure (wrong tag pushed), crash loop, failing readiness probe, insufficient resources.

**Diagnose →**
```bash
kubectl -n shack-dev get pods -l app=shack-app
kubectl -n shack-dev describe pod <new-pod>
kubectl -n shack-dev logs <new-pod> --previous
```

**Fix →** Fix the root cause; roll back with `kubectl -n shack-dev rollout undo deploy/shack-app` (automate it in `post { failure {} }`).

**Prevent →** Smoke test images in CI, realistic probes, and automated rollback in the deploy step.

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
