# CI/CD — Troubleshooting

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

> Realistic CI/CD failure symptoms with exact error text, likely causes, diagnosis commands and fixes · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to CI/CD](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Jenkins Build and Agent Failures](#jenkins-build-and-agent-failures) — Q1–Q35
- [Jenkins Credentials, Plugins and Controller Issues](#jenkins-credentials-plugins-and-controller-issues) — Q36–Q60
- [GitHub Actions Failures](#github-actions-failures) — Q61–Q100
- [GitLab CI Failures](#gitlab-ci-failures) — Q101–Q125
- [Build Tools, Nexus, SonarQube and Test Failures](#build-tools-nexus-sonarqube-and-test-failures) — Q126–Q155
- [Deployment, Kubernetes and GitOps Failures](#deployment-kubernetes-and-gitops-failures) — Q156–Q180
- [Secrets, Security and Pipeline Performance](#secrets-security-and-pipeline-performance) — Q181–Q200

---

## Jenkins Build and Agent Failures

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. Jenkins console shows `sh: 1: mvn: not found` on one agent only. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Maven is not installed or not on `PATH` for that agent; the `tools {}` block is missing; the agent runs a non-login shell.

**Diagnose →**
```bash
ssh agent-b 'which mvn; echo $PATH; mvn -v'
```
Compare with a working agent, and check the node's "Tool Locations".

**Fix →** Declare `tools { maven 'M3' }` in the Jenkinsfile, or run the stage in a Maven container image.

**Prevent →** Build agents from one image or Packer template so tools are identical everywhere.

</details>

### Q2. `java.lang.NoSuchMethodError: No such DSL method 'xyz' found among steps`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The plugin that provides the step is not installed or disabled; a typo; the step is used in the wrong context; a shared library is not loaded.

**Diagnose →** Open `/pipeline-syntax/globals` on the controller to list available steps; check Manage Jenkins > Plugins.

**Fix →** Install the plugin (for example `pipeline-utility-steps` for `readJSON`) or fix the spelling/library import `@Library('lib') _`.

**Prevent →** Pin plugins in `plugins.txt` and test pipelines against the same image in staging.

</details>

### Q3. `WorkflowScript: 12: Expected a step @ line 12, column 5.` in a Declarative pipeline. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Plain Groovy (variable assignment, `if`) placed directly in `steps {}`; Declarative allows only steps there.

**Diagnose →** Read the line number in the error; lint with:
```bash
curl -X POST -u user:token -F "jenkinsfile=<Jenkinsfile" https://jenkins.example.com/pipeline-model-converter/validate
```

**Fix →** Wrap Groovy logic in `script { ... }`, or use `when` for conditions.

**Prevent →** Lint the Jenkinsfile in a pre-commit hook or PR check.

</details>

### Q4. `hudson.AbortException: script returned exit code 1` with nothing useful above it. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A shell command failed; output was hidden by `returnStdout`, `-q` flags or redirection.

**Diagnose →** Re-run the command locally; add `set -euxo pipefail` and print context before the failing command.

**Fix →** Start scripts with `set -euxo pipefail`, remove quiet flags, or capture output with `returnStatus`/`tee` into an artifact.

**Prevent →** Keep verbose logs in CI and move complex logic into scripts that log what they do.

</details>

### Q5. Build hangs at "Waiting for next available executor" even though agents look idle. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Label expression matches no online node; agent is offline; node usage restricted; executors set to 0; the label has a typo.

**Diagnose →** Check Manage Jenkins > Nodes for online status and labels; hover the queue item for "Why am I in the queue".

**Fix →** Fix the label, bring the agent online, or add capacity.

**Prevent →** Alert on queue items older than N minutes and on offline agents.

</details>

### Q6. `Cannot run program "nohup" (in directory ...): error=2, No such file or directory` on a Windows agent. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The pipeline uses `sh` on a Windows node, which has no POSIX shell.

**Diagnose →** Check the node OS label and which step is used.

**Fix →** Use `bat` or `powershell` on Windows, or choose by `isUnix()`; install Git Bash only if truly necessary.

**Prevent →** Label nodes by OS and select agents with `label 'linux'` for shell steps.

</details>

### Q7. Build fails with `java.io.NotSerializableException: java.util.regex.Matcher`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A non-serializable object was kept in a CPS-transformed variable across a pipeline step.

**Diagnose →** The stack trace names the class; find where a `Matcher`, `JsonSlurper` result or similar is stored.

**Fix →** Extract logic into a method annotated `@NonCPS` that returns plain Strings/Lists, or use `readJSON`/`sh` with `grep`.

**Prevent →** Keep parsing in helper scripts or `@NonCPS` functions that do not call pipeline steps.

</details>

### Q8. `Scripts not permitted to use method groovy.lang.GroovyObject invokeMethod`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Sandboxed pipeline code calls a method not on the allow-list.

**Diagnose →** Manage Jenkins > In-process Script Approval lists the pending signature.

**Fix →** Replace the call with a built-in step or a shell command; if you approve, review the signature carefully; or move helper code into a trusted global library.

**Prevent →** Restrict who can approve scripts and keep complex logic out of Jenkinsfiles.

</details>

### Q9. `ERROR: Couldn't find any revision to build. Verify the repository and branch configuration for this job.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Branch specifier wrong (`*/master` vs `main`); empty repo; wrong credentials; the branch was deleted.

**Diagnose →**
```bash
git ls-remote https://github.com/org/repo.git
```
Check the job's Branch Specifier.

**Fix →** Use `*/main` (or `refs/heads/main`), and fix repository URL or credentials.

**Prevent →** Use multibranch jobs, which discover branches automatically.

</details>

### Q10. `java.nio.file.AccessDeniedException` while Jenkins deletes the workspace. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A container build created root-owned files in the mounted workspace; open file handles on Windows.

**Diagnose →** `ls -ln workspace/` and compare to the Jenkins user's UID.

**Fix →** Run the container with `-u $(id -u):$(id -g)`, or delete the files via a container as root; on Windows stop locking processes.

**Prevent →** Always run build containers as the agent's UID, and use ephemeral agents.

</details>

### Q11. `Agent went offline during the build` / `ChannelClosedException`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Agent OOM-killed or rebooted; spot instance reclaimed; network drop; pod evicted.

**Diagnose →**
```bash
journalctl -u jenkins-agent --since "1 hour ago"
dmesg -T | grep -i 'killed process'
kubectl describe pod <agent-pod>   # look for OOMKilled / Evicted
```

**Fix →** Increase memory limits, use on-demand nodes for critical builds, retry on agent loss with `retry(conditions: [agent()])`.

**Prevent →** Set requests/limits sensibly and monitor node memory.

</details>

### Q12. `Remote call on JNLP4-connect connection from <ip> failed` and agent will not connect. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong secret; TCP agent port blocked; reverse proxy blocking the websocket; Jenkins URL misconfigured.

**Diagnose →**
```bash
curl -I https://jenkins.example.com/tcpSlaveAgentListener/
java -jar agent.jar -url https://jenkins.example.com -name agent-1 -secret <s> -webSocket
```

**Fix →** Use `-webSocket` behind an HTTPS proxy, correct the secret, and set the right Jenkins URL.

**Prevent →** Prefer websocket connections and test connectivity with a health check.

</details>

### Q13. `error: RPC failed; HTTP 408 curl 22 The requested URL returned error: 408` during checkout. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Large repository over slow network; proxy timeouts.

**Diagnose →** Re-run `git clone` on the agent with `GIT_TRACE=1 GIT_CURL_VERBOSE=1`.

**Fix →** Enable shallow clone and a reference repo in the Git extension (Advanced clone behaviours), increase the checkout timeout, set `http.postBuffer`.

**Prevent →** Use shallow clones, sparse checkout and a local mirror cache.

</details>

### Q14. Jenkinsfile change is not picked up; the old pipeline runs. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Job reads a different branch or script path; "Replay" used; webhook not firing; Lightweight checkout cache.

**Diagnose →** Check "Script Path" and branch; look at build log's checkout commit SHA.

**Fix →** Fix path/branch, rescan the multibranch job, trigger a build.

**Prevent →** Use multibranch jobs with webhooks and print `GIT_COMMIT` at pipeline start.

</details>

### Q15. Parallel stages: one fails but others keep running for 30 minutes. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `failFast` not set.

**Diagnose →** Check whether `failFast true` is defined on the parallel block or `parallelsAlwaysFailFast()` in options.

**Fix →** Add `failFast true` or `options { parallelsAlwaysFailFast() }`.

**Prevent →** Make fail-fast the default in the shared library template.

</details>

### Q16. `Timeout has been exceeded` after 10 minutes though the step normally takes 3. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Slow dependency download, test hang, resource contention, agent waiting for input (a command prompting interactively).

**Diagnose →** Use `jstack` or `ps -ef` on the agent during the hang; look at the log timestamps (Timestamper plugin).

**Fix →** Make commands non-interactive (`-y`, `--batch-mode`), fix the hang, tune the timeout.

**Prevent →** Add timestamps and per-stage metrics to detect slow creep.

</details>

### Q17. `docker: command not found` in a stage that uses `docker build`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Docker not installed on the agent, or the stage runs in a container without the Docker CLI.

**Diagnose →** `which docker` on the agent; check the stage `agent` definition.

**Fix →** Install Docker on the node, or use a daemonless builder such as BuildKit/Buildah/Kaniko in a pod.

**Prevent →** Label Docker-capable agents and select by label.

</details>

### Q18. `Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Jenkins agent user is not in the `docker` group, or group membership was not refreshed.

**Diagnose →** `id jenkins`, `ls -l /var/run/docker.sock`.

**Fix →** `sudo usermod -aG docker jenkins` and restart the agent service so the new group applies.

**Prevent →** Configure it in the image; avoid sharing the socket with untrusted pipelines.

</details>

### Q19. `fatal: detected dubious ownership in repository at '/var/jenkins_home/workspace/x'`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Workspace owned by a different user than the one running git (common with containers).

**Diagnose →** `ls -ld` on the directory and `id` in the build.

**Fix →** Run containers with the matching UID, or `git config --global --add safe.directory '*'` as a last resort in ephemeral build images.

**Prevent →** Align UID/GID between host and container agents.

</details>

### Q20. `ERROR: script returned exit code 137` during a Maven or Docker build. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Exit code 137 = SIGKILL, usually OOM kill.

**Diagnose →**
```bash
dmesg -T | grep -i oom
kubectl describe pod <pod> | grep -A3 'Last State'
```

**Fix →** Raise container memory limit, set `MAVEN_OPTS=-Xmx1g`, reduce parallel test forks.

**Prevent →** Size requests/limits from observed build memory, and monitor OOM events.

</details>

### Q21. Pipeline fails at start with `org.codehaus.groovy.control.MultipleCompilationErrorsException: unable to resolve class`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing `import`, shared library not loaded, library class path wrong.

**Diagnose →** Read the unresolved class name; check `@Library` line and that library has `src/` package structure.

**Fix →** Add `@Library('lib@v1') _` and the proper `import`, fix the package path.

**Prevent →** Unit test library code and keep a sample pipeline that loads it.

</details>

### Q22. Build passes locally but `npm ci` fails in Jenkins with `npm ERR! code ERESOLVE`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Different npm/Node versions; lockfile out of sync; peer dependency conflict.

**Diagnose →** Print `node -v && npm -v` in the pipeline; compare with local.

**Fix →** Pin the Node version, regenerate the lockfile with the same npm version, fix the peer conflict (avoid blanket `--legacy-peer-deps`).

**Prevent →** `.nvmrc`/`engines` and a shared CI image.

</details>

### Q23. `Unable to find image 'maven:3.9' locally` then `toomanyrequests: You have reached your pull rate limit`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Anonymous Docker Hub pulls from a shared NAT IP.

**Diagnose →** The error text itself; `docker pull` manually; check Hub rate-limit headers.

**Fix →** Authenticate pulls, or pull via a Nexus/Artifactory/ECR pull-through cache.

**Prevent →** Mirror all base images in an internal registry.

</details>

### Q24. Stage `post { always { junit ... } }` fails with `No test report files were found. Configuration error?` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tests never ran (build failed earlier), or the glob is wrong.

**Diagnose →** `find . -name 'TEST-*.xml'` in the workspace.

**Fix →** Correct the glob (`**/target/surefire-reports/*.xml`); ensure tests execute; use `allowEmptyResults` only deliberately.

**Prevent →** Fail when test count is zero.

</details>

### Q25. `Failed to connect to repository: Command "git ls-remote -h ..." returned status code 128: fatal: could not read Username` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing or wrong credentials on the SCM source.

**Diagnose →** Test the credential manually; check the credential ID selected on the job.

**Fix →** Choose the right credential, or use SSH with a deploy key; for GitHub use a GitHub App credential.

**Prevent →** Use folder-scoped, documented credential IDs.

</details>

### Q26. Multibranch job does not create jobs for new PR branches. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No webhook; wrong branch discovery behaviours; repository scan failing; no Jenkinsfile on the branch.

**Diagnose →** Open "Scan Repository Log" on the job.

**Fix →** Add discovery behaviours (branches, PRs from origin and forks), set the webhook, fix credentials.

**Prevent →** Alert on scan failures.

</details>

### Q27. `The node ... is offline` for ephemeral Kubernetes agents; pods stay `Pending`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No cluster capacity; unsatisfiable resource requests; node selector/taint mismatch; image pull failure.

**Diagnose →**
```bash
kubectl get pods -n jenkins
kubectl describe pod <agent> -n jenkins
```
Read the Events section.

**Fix →** Add nodes or autoscale; fix requests, tolerations or image.

**Prevent →** Cluster autoscaler plus a dedicated node pool for builds.

</details>

### Q28. Builds slowly degrade; `Started by timer` jobs pile up in the queue. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Jobs run longer than their interval; concurrency allowed; not enough executors.

**Diagnose →** Check duration trends and queue; look for `disableConcurrentBuilds`.

**Fix →** Disable concurrent builds, increase interval, add capacity.

**Prevent →** Alert when duration exceeds the cron interval.

</details>

### Q29. `ERROR: Maven JVM terminated unexpectedly with exit code 137`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Surefire forks plus JVM heap exceed container memory.

**Diagnose →** Check `dmesg`/pod `OOMKilled`; look at `-Xmx` versus limit.

**Fix →** Lower fork count (`-DforkCount=1`), set heap below the container limit, give more memory.

**Prevent →** JVM flag `-XX:MaxRAMPercentage=70`.

</details>

### Q30. After adding `input` the pipeline fails with `Scripts not permitted` or the approval page rejects the user. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `submitter` does not include the user's ID or group; case sensitivity; SSO username differs.

**Diagnose →** Compare `submitter` to the real username in the Jenkins user page.

**Fix →** Use the exact ID or group name as reported by the security realm.

**Prevent →** Use groups rather than usernames.

</details>

### Q31. `ERROR: Error cloning remote repo 'origin'` with `fatal: unable to access ... SSL certificate problem: unable to get local issuer certificate`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Corporate TLS proxy or private CA not trusted by the agent's Git/Java.

**Diagnose →** `openssl s_client -connect git.example.com:443 -showcerts`.

**Fix →** Install the CA into the OS trust store and into the Java keystore (`keytool -importcert`), set `GIT_SSL_CAINFO` if needed. Do not disable verification.

**Prevent →** Bake the CA bundle into agent images.

</details>

### Q32. A pipeline using `archiveArtifacts` fails with `ERROR: No artifacts found that match the file pattern`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Build output path differs; the build failed earlier; the file is outside the workspace.

**Diagnose →** `ls -R` in the workspace before archiving.

**Fix →** Correct the Ant-style pattern (`target/*.jar`).

**Prevent →** Print the artifact path as part of the build step.

</details>

### Q33. `cannot run program "docker-compose"` or `docker compose` not found in a build. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Compose v1 binary removed; only the `docker compose` plugin is available, or neither installed.

**Diagnose →** `docker compose version`, `docker-compose --version`.

**Fix →** Use `docker compose` (v2 plugin) and install `docker-compose-plugin`.

**Prevent →** Pin tools in the agent image.

</details>

### Q34. Build using `withEnv` and `PATH+EXTRA` finds the wrong binary version. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** PATH order; multiple installs; `tools {}` prepending a different version.

**Diagnose →** `sh 'type -a java; java -version; echo $PATH'`.

**Fix →** Set one source of truth for the tool version, remove conflicting entries.

**Prevent →** Print tool versions at the beginning of every build.

</details>

### Q35. A Windows agent build fails with `The filename or extension is too long` / path over 260 chars. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Deep node_modules or long job names produce paths over MAX_PATH.

**Diagnose →** Check the workspace path length in the log.

**Fix →** Shorten the workspace root (`ws('C:\\w\\x')`), enable long paths (`LongPathsEnabled`, `git config core.longpaths true`).

**Prevent →** Use short job names or Linux agents.

</details>

---

## Jenkins Credentials, Plugins and Controller Issues

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q36. `ERROR: Could not find credentials entry with ID 'docker-hub'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ID typo; credential lives in a different folder scope; the job's context cannot see it.

**Diagnose →** Check Credentials > Stores for the scope and the exact ID.

**Fix →** Fix the ID or create the credential in the correct folder/global store.

**Prevent →** Document credential IDs and use consistent naming.

</details>

### Q37. Jenkins UI shows `HTTP ERROR 403 No valid crumb was included in the request`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CSRF protection and a reverse proxy dropping the session or crumb headers; API call without a crumb.

**Diagnose →** Check proxy config and the request headers.

**Fix →** Use API tokens (which don't need crumbs when sent with basic auth), or fetch the crumb from `/crumbIssuer/api/json`; fix the proxy to preserve headers.

**Prevent →** Do not disable CSRF protection.

</details>

### Q38. After upgrade, Jenkins shows `Jenkins failed to start` and logs `Failed Loading plugin ... Plugin is missing: ...`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Plugin dependency mismatch; plugins built for a newer core.

**Diagnose →**
```bash
journalctl -u jenkins -n 200
ls $JENKINS_HOME/plugins/*.jpi
```

**Fix →** Install the missing dependency or roll back the plugin by replacing its `.jpi` with the previous version; rollback the core if needed.

**Prevent →** Upgrade in staging with the same plugin set.

</details>

### Q39. Jenkins is extremely slow; the UI times out; thread dump shows many blocked threads. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Heavy plugin, too many jobs, large build histories, GC pauses, slow disk, or a stuck SCM polling thread.

**Diagnose →**
```bash
jcmd <pid> Thread.print > threads.txt
jcmd <pid> GC.heap_info
```
Check the Monitoring plugin or `/threadDump`.

**Fix →** Remove the offending plugin, reduce polling, discard old builds, tune heap/GC and move `JENKINS_HOME` to fast storage.

**Prevent →** Track controller JVM metrics and executors usage.

</details>

### Q40. `java.lang.OutOfMemoryError: Java heap space` on the controller. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Heap too small for job count; memory leak in a plugin; massive pipeline logs.

**Diagnose →** Take a heap dump (`jcmd <pid> GC.heap_dump /tmp/h.hprof`) and inspect with Eclipse MAT; check `JAVA_OPTS`.

**Fix →** Increase `-Xmx`, remove the leaking plugin, reduce build retention.

**Prevent →** Alert on heap utilisation and old gen GC time.

</details>

### Q41. `No such property: env for class: groovy.lang.Binding` in a shared library. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Accessing pipeline globals from a class in `src/` without passing the script context.

**Diagnose →** Check where `env` is used.

**Fix →** Pass `this` (steps) into the class constructor and use `steps.env`, `steps.sh`.

**Prevent →** Keep step calls in `vars/` and pure logic in `src/`.

</details>

### Q42. `Unable to create live FilePath for agent` / `java.io.IOException: Remote file system ... is not writable`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Disk full or read-only agent root.

**Diagnose →** `df -h`, `df -i` on the agent; check mount flags.

**Fix →** Free space (`cleanWs`, docker prune), fix permissions.

**Prevent →** Disk-space node monitor with auto-offline and cleanup jobs.

</details>

### Q43. Credentials binding fails: `Bad substitution` or `$PASSWORD: unbound variable`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Variable names mismatched between `withCredentials` and the script; wrong quote type; running `sh` using `/bin/sh` with Bash-only syntax.

**Diagnose →** Read the binding names and the script.

**Fix →** Align names, use single quotes, add `#!/bin/bash` where bash features are used.

**Prevent →** Standardise variable names in library steps.

</details>

### Q44. The GitHub webhook shows `Last delivery was not successful` with 403 or timeouts. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Jenkins not reachable; CSRF/crumb on the wrong endpoint; proxy blocks `POST`; wrong URL (missing trailing slash).

**Diagnose →** GitHub Webhooks > Recent Deliveries shows the response body; test with `curl -I https://jenkins.example.com/github-webhook/`.

**Fix →** Use `/github-webhook/` with trailing slash, open ingress for GitHub IPs, correct TLS.

**Prevent →** Monitor webhook delivery failures.

</details>

### Q45. `java.lang.IllegalStateException: Jenkins.instance is missing` in a plugin or script. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Code runs on an agent JVM instead of the controller, or Jenkins is shutting down.

**Diagnose →** Check where the code executes (inside a `node` block vs controller).

**Fix →** Move controller-API calls into `@NonCPS` library code running on the controller, not inside an agent-side closure.

**Prevent →** Avoid direct `Jenkins.instance` in pipelines.

</details>

### Q46. Jenkins reports `Insufficient permission` for a user who was granted access via an LDAP group. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Group name case mismatch; LDAP group membership not resolved; role strategy using different group name format.

**Diagnose →** Check `/whoAmI/` to see authorities for the user.

**Fix →** Use exactly the group names displayed in `/whoAmI/` (for example `ROLE_...` or lowercase CN).

**Prevent →** Test access with a non-admin account after config changes.

</details>

### Q47. `Jenkins: Please wait while Jenkins is getting ready to work` hangs after restart. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Many jobs loading; corrupt job config; plugin initialisation hang; huge build history.

**Diagnose →** `tail -f` the Jenkins log; check which job or plugin is loading last.

**Fix →** Wait for loading, move away a corrupt job directory, disable the problematic plugin.

**Prevent →** Reduce history; backup before changes.

</details>

### Q48. JCasC reload fails with `ConfiguratorException: Invalid configuration elements for type ...`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A plugin is missing, version changed attributes, or YAML indentation is wrong.

**Diagnose →** Read the attribute name in the error and check the plugin docs at `/configuration-as-code/reference`.

**Fix →** Install the plugin or fix attribute names.

**Prevent →** Validate JCasC in staging on every plugin upgrade.

</details>

### Q49. All builds suddenly fail with `Cannot contact agent: java.io.IOException: Could not copy remoting.jar`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SSH agent disk full; permissions on the remote root; Java missing on agent.

**Diagnose →** SSH in manually, check `java -version`, `df -h`, ownership of the remote FS root.

**Fix →** Free disk, install Java, correct ownership.

**Prevent →** Automated agent provisioning.

</details>

### Q50. SSH agent fails: `Host key verification failed`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Host key changed (rebuilt VM) or the verification strategy rejects unknown keys.

**Diagnose →** Check the "Host Key Verification Strategy" in the node setup; `ssh-keyscan agent`.

**Fix →** Use "Manually trusted key verification" and update the key, or "Known hosts file".

**Prevent →** Use inbound/websocket or Kubernetes agents instead of SSH to pets.

</details>

### Q51. Build log is full of `[Pipeline] // withCredentials` but secrets show as `****` inside curl output; the API rejects them. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Token contains trailing newline or whitespace copied into the credential; wrong secret type.

**Diagnose →** `printf '%s' "$TOKEN" | wc -c` and `| od -c`.

**Fix →** Re-create the credential without trailing newline; use `Secret text` for tokens.

**Prevent →** Add a token validity check as the first step.

</details>

### Q52. `Error 403 You are not authorized to ... while loading 'config.xml'` after enabling SSO. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Authorization matrix missing the SSO user or group; admin locked out.

**Diagnose →** Check Jenkins log for the user ID; check `/whoAmI/`.

**Fix →** Temporarily disable security in `config.xml` (set `<useSecurity>false</useSecurity>`) on the controller, restart, repair the matrix, re-enable.

**Prevent →** Keep a break-glass local admin and JCasC-managed authorization.

</details>

### Q53. Plugin update leaves `Pipeline: Groovy` classes broken: `NoClassDefFoundError`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Mixed versions of tightly coupled pipeline plugins (workflow-cps, workflow-api).

**Diagnose →** Plugin Manager "Installed" shows versions; look at `Failed Loading plugin` in logs.

**Fix →** Update the entire plugin family together or revert to the known good set from backup.

**Prevent →** Lock a tested `plugins.txt` set and update as a group.

</details>

### Q54. The controller disk is 100 percent full and Jenkins stops. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Unbounded build history, artifacts and logs stored on the controller.

**Diagnose →**
```bash
du -xh --max-depth=2 $JENKINS_HOME | sort -h | tail -20
df -i $JENKINS_HOME
```

**Fix →** Delete old builds, set `buildDiscarder`, move artifacts to Nexus/S3, expand the volume.

**Prevent →** Retention defaults in templates and disk alerts at 80%.

</details>

### Q55. `FATAL: Remote call on agent failed` and `java.lang.UnsupportedClassVersionError: ... class file version 65.0`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Agent runs an older Java than the one required by the controller or agent.jar (class version 65 means Java 21).

**Diagnose →** `java -version` on the agent.

**Fix →** Install a supported JDK on the agent (Jenkins requires Java 17 or newer for current releases).

**Prevent →** Align agent runtime in the image.

</details>

### Q56. Pipeline replay works but the committed Jenkinsfile fails with syntax errors about `@Library`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Library version in `@Library` does not exist; default version not set; "Load implicitly" assumptions differ.

**Diagnose →** Look at "Loading library" lines in the log.

**Fix →** Use an existing tag or branch (`@Library('lib@main') _`).

**Prevent →** Pin library versions and release via tags.

</details>

### Q57. Jenkins shows `Unable to find a build for artifact copy from: upstream-job`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Upstream job has no successful build, wrong selector, or artifacts were discarded.

**Diagnose →** Check upstream job history and artifact retention.

**Fix →** Use a proper selector (`lastSuccessful()`), or fetch from Nexus using version.

**Prevent →** Use an artifact repository rather than copying artifacts between jobs.

</details>

### Q58. Jenkins agent pods leak: hundreds of completed `Error` pods remain in the namespace. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pods crash before cleanup; the controller restarted while pods were running.

**Diagnose →** `kubectl get pods -n jenkins --field-selector=status.phase=Failed`.

**Fix →** Set pod retention to Never/OnFailure sensibly; delete stale pods with a CronJob; set `activeDeadlineSeconds` on the template.

**Prevent →** Resource quotas on the build namespace.

</details>

### Q59. Pipelines occasionally fail with `java.net.SocketTimeoutException: Read timed out` talking to the Docker or Git server. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Network flakiness, rate limits, overloaded server.

**Diagnose →** Run `curl -w '%{time_total}'` from the agent repeatedly; check the server's load.

**Fix →** Add retries with backoff on idempotent steps, raise timeouts, use caches/mirrors.

**Prevent →** Local mirrors reduce external dependencies.

</details>

### Q60. `ERROR: Build failed: hudson.remoting.RequestAbortedException` when the controller restarts. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Controller restarted mid-build; non-durable steps lost.

**Diagnose →** Check controller restart time and the agent connection log.

**Fix →** Pipeline builds are resilient to controller restarts if steps are durable; set performance-optimised durability thoughtfully, and use `restart` safely via "Safe Restart".

**Prevent →** Use safe restart and avoid restarts during peak.

</details>

---

## GitHub Actions Failures

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q61. Workflow never triggers after pushing. No run appears in the Actions tab. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** File not under `.github/workflows/`; invalid YAML; `on:` filters (branches/paths) do not match; workflow exists only on a non-default branch for `schedule`/`workflow_dispatch`; `[skip ci]` in the commit; Actions disabled.

**Diagnose →**
```bash
gh workflow list
gh run list --workflow ci.yml
actionlint .github/workflows/*.yml
```

**Fix →** Fix path/YAML, adjust the filters, and push to the right branch.

**Prevent →** Run `actionlint` in pre-commit.

</details>

### Q62. `Error: Process completed with exit code 1.` with no further detail. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A script returned non-zero; the error text was above or hidden.

**Diagnose →** Expand earlier log lines; re-run with debug logging (set secret `ACTIONS_STEP_DEBUG=true`, or "Re-run with debug logging"); add `set -x`.

**Fix →** Fix the failing command; default shell for `run` is `bash -e` on Linux, so any failing command stops the step.

**Prevent →** Log inputs before commands; use `set -euo pipefail` in scripts.

</details>

### Q63. `Error: Resource not accessible by integration` (HTTP 403) when the workflow comments on a PR or pushes a tag. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `GITHUB_TOKEN` lacks the needed permission, or the event is a fork PR where the token is read-only.

**Diagnose →** Check the "Set up job" section, which lists `GITHUB_TOKEN Permissions`; look at the top-level `permissions:` block.

**Fix →**
```yaml
permissions:
  contents: write
  pull-requests: write
```
Grant only what is needed, at job level.

**Prevent →** Start from `permissions: contents: read` and add per job.

</details>

### Q64. `Context access might be invalid: SECRET_NAME` and the secret is empty inside the job. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Secret defined in a different repo/environment; name typo (case-sensitive); fork PR; the job lacks `environment:` for an environment secret; reusable workflow not passed the secret.

**Diagnose →** Check Settings > Secrets (repo, environment, organization visibility); echo a masked check such as `[ -n "$S" ] && echo set`.

**Fix →** Define the secret at the right scope, add `environment:`, pass `secrets:` or `secrets: inherit` to reusable workflows.

**Prevent →** Fail early with a validation step for required secrets.

</details>

### Q65. `Unable to resolve action actions/checkout@v5, unable to find version v5`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The tag does not exist, or a typo in the ref.

**Diagnose →** `git ls-remote --tags https://github.com/actions/checkout`.

**Fix →** Use an existing tag or a full SHA.

**Prevent →** Pin to SHAs and let Dependabot propose updates.

</details>

### Q66. `The workflow is not valid. .github/workflows/ci.yml (Line: 14, Col: 9): Unexpected value 'run'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Indentation error, `run` placed outside a step, tabs in YAML.

**Diagnose →** `actionlint` and `yamllint`; read the reported line and column.

**Fix →** Make each step a list item under `steps:` with correct indentation.

**Prevent →** Editor YAML schema validation for workflows.

</details>

### Q67. `fatal: could not read Username for 'https://github.com': No such device or address` when a job clones a private repo. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `GITHUB_TOKEN` only covers the current repo; submodules or another private repo are fetched without a token.

**Diagnose →** Look at which repo the failing `git` command targets.

**Fix →** Use `actions/checkout` with `token:` from a GitHub App token or deploy key; for submodules set `submodules: recursive` and a suitable `token`/`ssh-key`.

**Prevent →** Use GitHub App tokens scoped to required repos.

</details>

### Q68. OIDC to AWS fails: `Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Trust policy `sub` condition does not match (branch vs environment form); wrong `aud`; missing `id-token: write`; the OIDC provider missing in the account.

**Diagnose →** Print the `sub` claim your workflow sends (for example `repo:org/app:ref:refs/heads/main` or `repo:org/app:environment:prod`) and compare with the role's trust policy:
```bash
aws iam get-role --role-name gha-deploy --query 'Role.AssumeRolePolicyDocument'
```

**Fix →** Align the `StringLike`/`StringEquals` condition with the real `sub` and `aud: sts.amazonaws.com`; add `permissions: id-token: write`.

**Prevent →** Keep the trust policy in Terraform with tests per environment.

</details>

### Q69. `Error: Input required and not supplied: token` in an action. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The action needs a `with: token:` that was not provided or the secret expression evaluated empty.

**Diagnose →** Read the action's README; check the expression.

**Fix →** Provide the input: `with: token: ${{ secrets.MY_TOKEN }}`.

**Prevent →** Validate required inputs in a first step.

</details>

### Q70. Cache is never restored: `Cache not found for input keys`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Key changes every run (for example includes `github.sha`); different OS or path; PR branch scope cannot see another branch's cache; evicted due to the repo's cache size limit.

**Diagnose →** `gh cache list`; compare keys between runs.

**Fix →** Use a key based on the lockfile hash with `restore-keys` fallbacks, and make sure the main branch populates the cache.

**Prevent →** Monitor cache hit rate; cache package manager directories only.

</details>

### Q71. `Error: Unable to download artifact(s): Artifact not found for name: build`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Upload step skipped or failed; name mismatch; download in another workflow run; v3 vs v4 differences; artifact expired.

**Diagnose →** Check the upload step log and the run's Artifacts section; `gh run view <id>`.

**Fix →** Ensure `needs:` ordering, matching names, `if-no-files-found: error` on upload, and use `run-id` plus a token for cross-run downloads.

**Prevent →** Use unique artifact names per matrix element.

</details>

### Q72. `Error: Conflict: an artifact with this name already exists on the workflow run` in a matrix. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Artifact v4 requires unique names; matrix jobs upload the same name.

**Diagnose →** Check names in the upload step.

**Fix →** Use `name: logs-${{ matrix.os }}-${{ matrix.node }}`, and download all with `pattern: logs-*` and `merge-multiple: true`.

**Prevent →** Include matrix keys in artifact names.

</details>

### Q73. Job stays `Queued` for a long time with a self-hosted label. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No online runner matches the labels; runner group not allowed for the repo; runner busy.

**Diagnose →** Settings > Actions > Runners to see status and labels; `journalctl -u actions.runner.*` on the host.

**Fix →** Fix labels or runner group access, restart the service, add capacity.

**Prevent →** Alert on queue time and runner offline events.

</details>

### Q74. Runner log shows `The runner registration token has expired` / registration failing. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Registration tokens are valid for one hour and single use for registration.

**Diagnose →** Check when the token was created.

**Fix →** Generate a fresh token via API: `gh api -X POST repos/ORG/REPO/actions/runners/registration-token`, or use ARC with a GitHub App.

**Prevent →** Automate registration with just-in-time runners.

</details>

### Q75. `Error: The operation was canceled.` for jobs that appear to run fine. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `cancel-in-progress` concurrency cancelled it; timeout; manual cancel; runner lost communication.

**Diagnose →** Check the run summary for "cancelled" reason and the concurrency group.

**Fix →** Narrow the concurrency group (include ref), or set `cancel-in-progress: false` for deploys.

**Prevent →** Use `cancel-in-progress: ${{ github.event_name == 'pull_request' }}`.

</details>

### Q76. `if: github.ref == 'refs/heads/main'` never true on PR builds. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** On `pull_request` events `github.ref` is `refs/pull/<n>/merge`, not the branch.

**Diagnose →** Print `github.ref`, `github.head_ref`, `github.base_ref`.

**Fix →** Use `github.head_ref` for the PR branch, or condition on `github.event_name == 'push'`.

**Prevent →** Dump context once in debugging.

</details>

### Q77. `The job was not started because recent account payments have failed or your spending limit needs to be increased`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Billing issue or budget limit for private repos/minutes.

**Diagnose →** Organization Settings > Billing and plans.

**Fix →** Fix the payment method or raise the spending limit.

**Prevent →** Budget alerts and usage reports.

</details>

### Q78. `docker: Error response from daemon: Get "https://registry-1.docker.io/v2/": net/http: request canceled` or rate limit in Actions. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Docker Hub rate limiting or transient network issue.

**Diagnose →** Check response text for `toomanyrequests`.

**Fix →** Log in with `docker/login-action` using a Hub token, or pull from GHCR/ECR mirror.

**Prevent →** Mirror base images and cache layers.

</details>

### Q79. `Error: Bad credentials` from `gh` or an action using `GITHUB_TOKEN`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Token not passed as `GH_TOKEN`; expired PAT; used inside a container without env.

**Diagnose →** `gh auth status` in the step.

**Fix →** `env: GH_TOKEN: ${{ github.token }}` on the step.

**Prevent →** Prefer `github.token` and set env at job level.

</details>

### Q80. A step using `echo "::set-output name=v::1"` prints a deprecation warning and outputs are empty. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `set-output` and `save-state` commands were deprecated and later disabled.

**Diagnose →** Warning text in the log.

**Fix →** `echo "v=1" >> "$GITHUB_OUTPUT"`; similarly `$GITHUB_ENV` and `$GITHUB_PATH` instead of `set-env`/`add-path`.

**Prevent →** Update old third-party actions.

</details>

### Q81. Environment variable set in one step is missing in the next. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `export VAR=x` only affects its own step's shell.

**Diagnose →** Print `env` in the next step.

**Fix →** Append to the file: `echo "VAR=x" >> "$GITHUB_ENV"`.

**Prevent →** Use step `outputs` for typed data and `GITHUB_ENV` for env vars.

</details>

### Q82. `Error: Process completed with exit code 143` (or `Runner received shutdown signal`). 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Job cancelled or runner shut down; for self-hosted, VM/pod terminated or spot reclaimed.

**Diagnose →** Check host logs and whether another workflow cancelled it.

**Fix →** Use on-demand nodes for ARC runner pods or set graceful shutdown; fix the cancel logic.

**Prevent →** Keep runner infrastructure stable and observable.

</details>

### Q83. `No space left on device` on a GitHub-hosted runner. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Large Docker images, build caches or dependencies exceed free disk.

**Diagnose →** `df -h` and `docker system df` at start and end.

**Fix →** Remove unneeded preinstalled software (for example `/usr/share/dotnet`, `/opt/ghc`), prune Docker, or use a larger runner.

**Prevent →** Multi-stage builds, smaller images, and disk checks.

</details>

### Q84. Reusable workflow fails: `invalid value workflow reference: no version specified`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `uses:` for a reusable workflow in another repo requires a ref after `@`.

**Diagnose →** Look at the `uses:` line.

**Fix →** `uses: org/repo/.github/workflows/build.yml@v1`; for same-repo use `./.github/workflows/build.yml` (no ref).

**Prevent →** Pin to tags or SHAs.

</details>

### Q85. Caller of a reusable workflow gets `Secret X is required, but not provided while calling`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The called workflow declares `on.workflow_call.secrets.X.required: true` but the caller did not pass it.

**Diagnose →** Compare the `secrets:` blocks.

**Fix →** Pass `secrets: inherit` or `secrets: X: ${{ secrets.X }}`.

**Prevent →** Document required inputs and secrets in the callee.

</details>

### Q86. Pull request from a fork fails with `Resource not accessible by integration` on a commenting step. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Fork PRs get a read-only token and no secrets under `pull_request`.

**Diagnose →** Check the event and token permissions in "Set up job".

**Fix →** Upload results as an artifact, and post comments from a separate `workflow_run` workflow that treats the artifact as untrusted input.

**Prevent →** Design PR workflows with the fork limits in mind.

</details>

### Q87. Matrix job fails only for Node 18 and cancels all other jobs. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `fail-fast` defaults to true; one failure cancels in-progress matrix jobs.

**Diagnose →** Look at "Cancelled" statuses of siblings.

**Fix →** Set `strategy.fail-fast: false` while you debug; use `continue-on-error` for experimental entries.

**Prevent →** Decide deliberately: fail-fast for cost, no fail-fast for full information.

</details>

### Q88. A composite action step fails with `Required property is missing: shell`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `run` steps inside composite actions must declare `shell`.

**Diagnose →** Check `action.yml`.

**Fix →** Add `shell: bash` to each `run` step.

**Prevent →** Validate action metadata with actionlint.

</details>

### Q89. `Error: Cannot find module` or tests pass locally but fail on `ubuntu-latest` only after a runner image update. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `ubuntu-latest` moved to a new image with different tool versions; unpinned tool versions.

**Diagnose →** Compare "Runner Image" version in the Set up job section between a green and red run.

**Fix →** Pin the OS (`ubuntu-24.04`) and tool versions; fix the code for the new defaults.

**Prevent →** Pin runner images and upgrade deliberately with a canary workflow.

</details>

### Q90. `Error: HttpError: Not Found` when deploying to an environment with required reviewers. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Environment name mismatch or token missing `deployments` permission; branch not allowed by the environment's deployment branch rule.

**Diagnose →** Check Settings > Environments for the name and branch policy.

**Fix →** Match the name exactly, allow the branch, grant `deployments: write` where needed.

**Prevent →** Define environments in Terraform/IaC.

</details>

### Q91. `Waiting for approval` job never proceeds and is auto-cancelled after 30 days. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Reviewers not notified; reviewers list contains inactive users; self-review prevented.

**Diagnose →** Environment settings > required reviewers; "Prevent self-review" option.

**Fix →** Add active reviewers or a team; notify them (Slack bot).

**Prevent →** Notifications on pending deployments.

</details>

### Q92. A workflow triggered by `GITHUB_TOKEN` pushes does not trigger other workflows. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Events created by `GITHUB_TOKEN` (except `workflow_dispatch` and `repository_dispatch`) do not start new workflow runs, to avoid recursion.

**Diagnose →** Check which token pushed the commit/tag.

**Fix →** Push with a GitHub App token or PAT if the downstream workflow must run, or call it explicitly with `workflow_call`/`gh workflow run`.

**Prevent →** Design chains using `workflow_call` or `workflow_run`.

</details>

### Q93. `Error: buildx failed with: ERROR: failed to solve: failed to compute cache key: "/dist" not found`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `COPY` source missing from the build context: not built yet, excluded by `.dockerignore`, or wrong context path.

**Diagnose →** `ls` the context directory in a step before the build; read `.dockerignore`.

**Fix →** Build assets first, set `context:`, and fix `.dockerignore`.

**Prevent →** Build the app in a multi-stage Dockerfile.

</details>

### Q94. `docker/build-push-action` fails: `denied: permission_denied: write_package`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing `packages: write`, or the package is linked to another repo and not granted access.

**Diagnose →** Check `permissions:` and the package settings (Manage Actions access).

**Fix →** Add `packages: write`, and grant the repository access in the package settings.

**Prevent →** Use consistent image names tied to the repo.

</details>

### Q95. Self-hosted runner jobs fail with `Cannot connect to the Docker daemon at unix:///var/run/docker.sock`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Docker not running or the runner user lacks permission.

**Diagnose →** `systemctl status docker`, `id runner`.

**Fix →** Start Docker, add the runner user to the `docker` group and restart the runner service.

**Prevent →** Bake into the runner image.

</details>

### Q96. Step `uses: ./.github/actions/setup` fails with `Can't find 'action.yml', 'action.yaml' or 'Dockerfile'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Local actions need the repo checked out first.

**Diagnose →** Check step order.

**Fix →** Add `actions/checkout` before the local action step.

**Prevent →** Put checkout first in every job.

</details>

### Q97. `The 'set-env' command is disabled` or `Unable to process file command 'env' successfully. Invalid format`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Old syntax, or writing multiline values to `GITHUB_ENV` without a delimiter.

**Diagnose →** Read the failing line.

**Fix →**
```bash
{
  echo 'NOTES<<EOF'
  cat notes.txt
  echo 'EOF'
} >> "$GITHUB_ENV"
```

**Prevent →** Use a random delimiter for untrusted content.

</details>

### Q98. Scheduled workflow stopped running after months with no commits. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Scheduled workflows in public repositories are disabled automatically after 60 days without repo activity.

**Diagnose →** Actions tab shows a disabled banner.

**Fix →** Re-enable the workflow and push activity, or use an external trigger.

**Prevent →** Have a periodic keep-alive or use `workflow_dispatch` from an external scheduler.

</details>

### Q99. `Error: HttpError: Resource protected by organization SAML enforcement` in a workflow. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A PAT is not authorized for SSO.

**Diagnose →** Check the token's SSO authorization in GitHub.

**Fix →** Authorize the token for the org, or replace it with a GitHub App token.

**Prevent →** Avoid PATs in automation.

</details>

### Q100. A job fails: `Error: This request has been automatically failed because it uses a deprecated version of actions/upload-artifact: v3`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Artifact actions v1-v3 were deprecated and removed.

**Diagnose →** Search `uses: actions/(upload|download)-artifact@v3` in the repo.

**Fix →** Upgrade to v4 (and adjust unique names and `merge-multiple`).

**Prevent →** Dependabot for `github-actions` ecosystem.

</details>

---

## GitLab CI Failures

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. `This GitLab CI configuration is invalid: jobs:build config contains unknown keys: scripts`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Typo in a keyword.

**Diagnose →** CI Lint (Pipeline editor > Validate) or `glab ci lint`.

**Fix →** Correct to `script`.

**Prevent →** Lint in an MR check and use the editor with schema validation.

</details>

### Q102. Pipeline is not created: `Pipeline will not run for the selected trigger. The rules configuration prevented any jobs from being added`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Every job's `rules` evaluated false for this pipeline source, or `workflow:rules` excluded it.

**Diagnose →** Check `$CI_PIPELINE_SOURCE` and branch for the event; simulate with the pipeline editor.

**Fix →** Add a rule for the source (`merge_request_event`, `push`, `schedule`) or fix `workflow:rules`.

**Prevent →** A sensible `workflow:rules` block in every project template.

</details>

### Q103. `ERROR: Job failed: failed to pull image "x" with specified policies [always]: Error response from daemon: pull access denied`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Private image without registry credentials; typo; the project registry path.

**Diagnose →** Test `docker pull` on the runner host; check `DOCKER_AUTH_CONFIG`.

**Fix →** Set the `DOCKER_AUTH_CONFIG` CI variable with registry credentials or use dependency proxy.

**Prevent →** Use the GitLab Dependency Proxy for external images.

</details>

### Q104. `Cannot connect to the Docker daemon at tcp://docker:2375. Is the docker daemon running?` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** DinD service not started, TLS settings mismatch, or non-privileged runner.

**Diagnose →** Check the service logs in the job; runner config for `privileged = true`.

**Fix →**
```yaml
services: [docker:27-dind]
variables:
  DOCKER_HOST: tcp://docker:2376
  DOCKER_TLS_CERTDIR: "/certs"
```
Or use Kaniko/BuildKit.

**Prevent →** Standardise one image-build approach.

</details>

### Q105. `ERROR: Job failed (system failure): ... context deadline exceeded` on Kubernetes executor. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pod pending (no capacity), image pull slow, API server issues.

**Diagnose →** `kubectl get pods -n gitlab-runner`, `kubectl describe pod`.

**Fix →** Add capacity, raise `poll_timeout`, fix image pull problems.

**Prevent →** Autoscaling node pool and image pre-pull.

</details>

### Q106. `This job is stuck because the project doesn't have any runners online assigned to it.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No runner with matching tags; runners paused; shared runners disabled.

**Diagnose →** Settings > CI/CD > Runners; compare `tags:` with runner tags.

**Fix →** Fix tags, enable runners, start the runner service (`gitlab-runner status`).

**Prevent →** Alert on stuck jobs and runner offline.

</details>

### Q107. `fatal: unable to access 'https://gitlab.example.com/group/proj.git/': SSL certificate problem: self signed certificate`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Runner does not trust a private CA.

**Diagnose →** `openssl s_client -connect gitlab.example.com:443`.

**Fix →** Set `tls-ca-file` in the runner config (`/etc/gitlab-runner/certs/gitlab.example.com.crt`).

**Prevent →** Provision CA trust in runner images.

</details>

### Q108. Artifact not available in the next job: `ERROR: Downloading artifacts from coordinator... not found`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Artifacts expired; the producer job did not run (rules) or lacks `artifacts:`; `dependencies`/`needs` excludes it.

**Diagnose →** Check the producer job's artifact browser and `needs`.

**Fix →** Declare `artifacts: paths:` with sufficient `expire_in`, and list the producer in `needs: [job]`.

**Prevent →** Keep artifact retention aligned to pipeline length.

</details>

### Q109. `fatal: couldn't find remote ref refs/merge-requests/12/head` or detached head issues in MR pipelines. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Shallow clone depth, forks, or MR ref not fetched.

**Diagnose →** `git log -1`, `echo $CI_MERGE_REQUEST_REF_PATH`.

**Fix →** Increase `GIT_DEPTH`, set `GIT_STRATEGY: clone`, fetch the right ref.

**Prevent →** Use `CI_MERGE_REQUEST_DIFF_BASE_SHA` for diffs.

</details>

### Q110. Cache is not restored: `WARNING: node_modules/: no matching files`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Paths are relative to project dir and must be inside it; nothing produced the path in the earlier job; the key changes.

**Diagnose →** Read the cache section of the job log (key, paths, hit or miss).

**Fix →** Cache `.npm/` and set `npm_config_cache`, use `key: files:`.

**Prevent →** Use `policy: pull` where jobs only read.

</details>

### Q111. `Job's rule evaluation produced: variable $FOO is not defined` or rules `if` always false for a custom variable. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Variable is protected/masked for a different ref, or defined in a scope that is not available at pipeline creation time (job-level `variables` cannot be used in `rules:if` of workflow).

**Diagnose →** Check variable scope and the `rules` evaluation context.

**Fix →** Define the variable at the top-level `variables:` or in project settings with proper scope.

**Prevent →** Print `env` (non-secret) while debugging.

</details>

### Q112. `ERROR: Job failed: exit code 1` with `/bin/bash: line 1: npm: command not found`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong or default image without Node.

**Diagnose →** Check `image:` for the job and default.

**Fix →** Set `image: node:22`.

**Prevent →** Set `default: image:` and pin versions.

</details>

### Q113. Pipeline fails: `Insufficient permissions to ... CI_JOB_TOKEN` when pulling from another project. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The target project does not allow the calling project in its job token allowlist.

**Diagnose →** Target project > Settings > CI/CD > Job token permissions.

**Fix →** Add the calling project to the allowlist.

**Prevent →** Manage the allowlist as code or via API.

</details>

### Q114. Downstream pipeline fails to start: `downstream pipeline can not be created, the user is not authorized`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The triggering user lacks permission on the downstream project/branch (protected branch).

**Diagnose →** Check roles and protected branch rules downstream.

**Fix →** Grant developer role or allow the pipeline user to push/merge as needed.

**Prevent →** Use project access tokens scoped for triggers.

</details>

### Q115. `ERROR: Preparation failed: Error response from daemon: No such image: ...` using Docker executor. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `pull_policy` set to `never` or `if-not-present` while the image is missing, or a typo.

**Diagnose →** Check runner `config.toml` `pull_policy`.

**Fix →** Use `pull_policy = ["always", "if-not-present"]`.

**Prevent →** Consistent image naming and a mirror.

</details>

### Q116. Jobs get `ERROR: Job failed: execution took longer than 1h0m0s seconds`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Default project timeout of 60 minutes reached; a hung step.

**Diagnose →** Check the job trace for the last output.

**Fix →** Fix the hang, or increase `timeout:` per job and project/runner timeout.

**Prevent →** Per-job timeouts and duration alerts.

</details>

### Q117. Merge request pipeline shows `Detached merge request pipeline` and `$CI_COMMIT_BRANCH` is empty. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** MR pipelines do not set `CI_COMMIT_BRANCH`; they set `CI_MERGE_REQUEST_*` variables.

**Diagnose →** Print `CI_PIPELINE_SOURCE` and `CI_MERGE_REQUEST_SOURCE_BRANCH_NAME`.

**Fix →** Use `CI_COMMIT_REF_NAME` or MR variables in logic.

**Prevent →** Test rules in both branch and MR pipelines.

</details>

### Q118. `exec user process caused: exec format error` on a job. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Image architecture (arm64 vs amd64) does not match the runner.

**Diagnose →** `docker image inspect --format '{{.Architecture}}' image`; `uname -m` on runner.

**Fix →** Use a multi-arch image or tag runners by architecture and select via `tags:`.

**Prevent →** Build multi-arch images with `buildx`.

</details>

### Q119. `Skipping Git submodules setup` / submodule clone fails `Permission denied`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `GIT_SUBMODULE_STRATEGY` not set, or the job token lacks access to the submodule project.

**Diagnose →** Check job log "Updating/initializing submodules".

**Fix →** `variables: GIT_SUBMODULE_STRATEGY: recursive` and allow the job token on the submodule project.

**Prevent →** Use relative URLs in `.gitmodules`.

</details>

### Q120. Runner stops picking up jobs; `gitlab-runner verify` shows `is removed`/`authentication failed`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Runner deleted in UI, token rotated or expired.

**Diagnose →**
```bash
sudo gitlab-runner verify
sudo journalctl -u gitlab-runner -n 100
```

**Fix →** Create a new runner in the UI and re-register with `gitlab-runner register --token <glrt-...>`.

**Prevent →** Manage runners via Terraform and monitor offline runners.

</details>

### Q121. `rules:changes` job always runs on new branches. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** For branch pipelines without a base, `changes` evaluates true when there is no previous commit.

**Diagnose →** Check whether it is a first push to the branch.

**Fix →** Add `compare_to: refs/heads/main` under `rules:changes`, or use MR pipelines.

**Prevent →** Prefer MR pipelines for change-based rules.

</details>

### Q122. `yaml invalid: Unknown anchor` / `!reference` fails. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Anchors defined in a different included file (anchors do not cross files), or wrong `!reference` path.

**Diagnose →** CI Lint output.

**Fix →** Use `extends:` or `!reference [.template, script]` for cross-file reuse.

**Prevent →** Do not use anchors across includes.

</details>

### Q123. `ERROR: Uploading artifacts as "archive" to coordinator... too large` (413). 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Artifact exceeds the max artifact size limit.

**Diagnose →** `du -sh` on the path and check instance/project max artifact size.

**Fix →** Reduce what is archived, or raise the limit; store large outputs in a package registry or S3.

**Prevent →** Use `artifacts:exclude` and the package registry for binaries.

</details>

### Q124. Pipeline passes in MR but deploy job is missing on main. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Deploy `rules` only match the default branch, but other `rules` earlier return `when: never`.

**Diagnose →** Check rule order; first matching rule wins.

**Fix →** Reorder rules so the main-branch rule matches before exclusions.

**Prevent →** Keep rule blocks simple and documented.

</details>

### Q125. `ERROR: Job failed: pod "runner-..." status is "Failed"` with reason `OOMKilled` on Kubernetes runner. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Job container memory limit exceeded.

**Diagnose →** `kubectl describe pod` (Last State: OOMKilled).

**Fix →** Increase `KUBERNETES_MEMORY_LIMIT`/request via job variables or runner config; reduce parallelism.

**Prevent →** Tune per-job resource overrides.

</details>

---

## Build Tools, Nexus, SonarQube and Test Failures

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. `[ERROR] Failed to execute goal ... Could not resolve dependencies ... Could not transfer artifact ... from/to central ... Connect timed out`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No route to Maven Central; proxy settings missing; mirror misconfigured.

**Diagnose →**
```bash
mvn -B -X dependency:resolve | grep -i 'mirror\|proxy'
curl -I https://repo.maven.apache.org/maven2/
```

**Fix →** Point `settings.xml` to the Nexus mirror or set the `<proxy>` block.

**Prevent →** Always use an internal repository proxy.

</details>

### Q127. `Return code is: 401, ReasonPhrase: Unauthorized` on `mvn deploy` to Nexus. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Server `<id>` in `settings.xml` does not match `distributionManagement` repository id; wrong credentials; token expired.

**Diagnose →** Compare ids; test with `curl -u user:pass -I https://nexus/repository/maven-releases/`.

**Fix →** Align ids and update credentials.

**Prevent →** Standard `settings.xml` generated by CI from secrets.

</details>

### Q128. `Return code is: 400, ReasonPhrase: Repository does not allow updating assets: maven-releases`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Redeploy of an existing release version is disabled (correctly).

**Diagnose →** Check if the version already exists in Nexus.

**Fix →** Bump the version; never overwrite releases. Delete only in true emergencies by an admin.

**Prevent →** Automatic version computation per build.

</details>

### Q129. `Return code is: 413, ReasonPhrase: Request Entity Too Large` when uploading an artifact. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Reverse proxy (nginx) body size limit.

**Diagnose →** Check the proxy config and its error log.

**Fix →** Set `client_max_body_size 0;` (or a higher limit) for the Nexus location and reload nginx.

**Prevent →** Include proxy limits in the Nexus runbook.

</details>

### Q130. `npm ERR! code E401` / `Unable to authenticate, need: Basic realm="Sonatype Nexus Repository Manager"`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing or wrong `_auth`/`_authToken` in `.npmrc`; npm Bearer Token realm not enabled in Nexus.

**Diagnose →** `npm config list`, `npm whoami --registry <url>`.

**Fix →** Provide credentials in `.npmrc` from CI secrets and enable the "npm Bearer Token Realm" or use Basic auth correctly.

**Prevent →** Template `.npmrc` creation in the pipeline.

</details>

### Q131. SonarQube scan: `ERROR: Not authorized. Analyzing this project requires authentication.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing or invalid `SONAR_TOKEN`.

**Diagnose →** `curl -u $SONAR_TOKEN: https://sonar/api/authentication/validate`.

**Fix →** Generate a project analysis token and set `SONAR_TOKEN` as a secret.

**Prevent →** Use scoped analysis tokens per project.

</details>

### Q132. `ERROR: Quality gate failed` but the dashboard looks fine on the main branch. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The gate evaluates new code (since previous version or reference branch), and this PR adds uncovered or duplicated lines.

**Diagnose →** Open the PR analysis; look at "New Code" metrics.

**Fix →** Add tests, reduce duplication, fix new issues.

**Prevent →** Show Sonar results in PR decoration.

</details>

### Q133. `Shallow clone detected, no blame information will be provided` and new-code detection is wrong. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CI cloned with limited depth.

**Diagnose →** `git rev-parse --is-shallow-repository`.

**Fix →** `fetch-depth: 0` in `actions/checkout`, or `GIT_DEPTH: 0` in GitLab.

**Prevent →** Standard checkout settings for the scan job.

</details>

### Q134. `java.lang.OutOfMemoryError: Java heap space` in SonarScanner or Maven Sonar plugin. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Large project and default heap.

**Diagnose →** Check logs near the failure and file counts.

**Fix →** `SONAR_SCANNER_OPTS=-Xmx2048m` or `MAVEN_OPTS=-Xmx2g`; exclude generated sources with `sonar.exclusions`.

**Prevent →** Exclude vendor/generated directories.

</details>

### Q135. Unit tests pass locally but fail in CI with `Connection refused` to a database. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** DB service not ready or wrong host (localhost vs service name).

**Diagnose →** Check the service container logs and the host in the connection string.

**Fix →** Add health checks and a wait loop (`pg_isready`), use the correct hostname.

**Prevent →** Use Testcontainers, which handles readiness.

</details>

### Q136. Tests fail only in CI with `java.util.concurrent.TimeoutException` or intermittent timeouts. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CI runners are slower and have fewer CPUs; timeouts are tuned for laptops.

**Diagnose →** Compare test durations; check CPU limits and parallelism.

**Fix →** Use polling/awaits instead of fixed sleeps (Awaitility), reduce parallel forks, adjust timeouts.

**Prevent →** Avoid sleep-based tests.

</details>

### Q137. Tests pass in sequence but fail when run in parallel. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Shared global state, files, ports or database rows.

**Diagnose →** Run a single failing pair; randomise order.

**Fix →** Isolate state per test, use random ports/temp dirs.

**Prevent →** Run tests in random order in CI.

</details>

### Q138. `mvn verify` fails: `Tests run: 0` or surefire finds no tests after a JUnit upgrade. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Surefire version too old for JUnit 5, or mixed JUnit 4/5 engines without vintage.

**Diagnose →** `mvn dependency:tree | grep junit`; check the surefire version.

**Fix →** Use Surefire 3.x and add `junit-vintage-engine` for legacy tests.

**Prevent →** Pin plugin versions in the POM.

</details>

### Q139. `Gradle build fails: Could not resolve all files for configuration ':compileClasspath'`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Repository not declared, offline mirror lacks the artifact, or credentials missing.

**Diagnose →** `./gradlew dependencies --configuration compileClasspath --info`.

**Fix →** Add the repository (Nexus URL with credentials from env), refresh dependencies.

**Prevent →** Lock dependencies with `--write-locks`.

</details>

### Q140. `npm run build` fails in CI with `JavaScript heap out of memory`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Node default heap limit versus project size, or container memory limit.

**Diagnose →** Check the memory limit and the Node version.

**Fix →** `NODE_OPTIONS=--max-old-space-size=4096` and increase the runner memory.

**Prevent →** Track bundle and build memory trends.

</details>

### Q141. `Error: ENOSPC: System limit for number of file watchers reached` in CI. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** inotify limit reached by watch-mode tools.

**Diagnose →** `cat /proc/sys/fs/inotify/max_user_watches`.

**Fix →** Disable watch mode in CI (`CI=true`, `--watchAll=false`), or raise the limit on the host.

**Prevent →** Use non-watch test commands in pipelines.

</details>

### Q142. Python job: `ERROR: Could not find a version that satisfies the requirement foo==1.2` using a private index. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong `--index-url`, missing credentials, or package yanked/not proxied.

**Diagnose →** `pip install -v foo==1.2 --index-url ...`.

**Fix →** Configure `PIP_INDEX_URL` with token and check that the proxy repo includes the package.

**Prevent →** Pin with hashes and use a proxy.

</details>

### Q143. Build uses a cached layer and ships stale code. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `COPY . .` before dependency steps or a cache key not invalidating on source changes; stale remote cache.

**Diagnose →** `docker build --progress=plain` and look for `CACHED`.

**Fix →** Fix layer order, and use `--no-cache` or invalidate cache for the release build.

**Prevent →** Bake the commit SHA into the image label and verify at deploy.

</details>

### Q144. Docker build in CI: `failed to solve: process "/bin/sh -c npm ci" did not complete successfully: exit code: 1`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Lockfile mismatch, private registry token not available at build time, network blocked.

**Diagnose →** `docker build --progress=plain --no-cache .` and read npm's error.

**Fix →** Use BuildKit secrets (`RUN --mount=type=secret,id=npmrc ...`) for private registries.

**Prevent →** Never pass secrets with `ARG`.

</details>

### Q145. SonarQube analysis is stuck at `Waiting for the analysis report to be processed`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Compute Engine backlog or failure; Elasticsearch issue; DB problem.

**Diagnose →** Administration > Projects > Background Tasks; check `ce.log` and `es.log`.

**Fix →** Resolve the failed CE task, restart the service if ES is red, increase resources.

**Prevent →** Monitor CE queue length.

</details>

### Q146. `Could not find artifact com.example:lib:jar:1.2.0-SNAPSHOT` although it was just published. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Snapshot repo not in the group/mirror, local `~/.m2` caching a failed lookup (`lastUpdated` file), update policy interval.

**Diagnose →** Check Nexus for the artifact and `ls ~/.m2/repository/com/example/lib/`.

**Fix →** Add the snapshot repo, run `mvn -U`, remove `*.lastUpdated` files.

**Prevent →** Use release versions for cross-team dependencies.

</details>

### Q147. Nexus UI shows blob store at 100 percent and uploads fail. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No cleanup policy or compaction.

**Diagnose →** Check the blob store usage and tasks.

**Fix →** Delete old components, run "Compact blob store", expand the volume.

**Prevent →** Cleanup policies and disk alerts.

</details>

### Q148. `Error: ERESOLVE unable to resolve dependency tree` suddenly in CI after a new package release. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `npm install` pulled a new transitive version; no lockfile in use.

**Diagnose →** `npm ls`, compare lockfile in Git.

**Fix →** Use `npm ci` with a committed lockfile, pin the offending version via `overrides`.

**Prevent →** Lockfile plus Renovate.

</details>

### Q149. Test report parser: `Failed to parse JUnit XML: Premature end of file`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Test process killed (OOM/timeout) while writing the report.

**Diagnose →** Check the file size and exit code (137).

**Fix →** Address the killer (memory/timeout), re-run.

**Prevent →** Set limits and fail with a clear message.

</details>

### Q150. Cypress/Playwright tests fail in CI with `browserType.launch: Executable doesn't exist`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Browsers not installed in the CI image.

**Diagnose →** `npx playwright --version`; check the install step.

**Fix →** `npx playwright install --with-deps` or use the official Playwright image matching your version.

**Prevent →** Pin image version equal to the library version and cache browsers.

</details>

### Q151. Coverage differs between local and CI by 10 percent. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Different include/exclude patterns, test subset in CI, parallel shards not merged, path remapping.

**Diagnose →** Compare file lists in the coverage reports.

**Fix →** Merge sharded reports and align configs.

**Prevent →** One coverage config used by both.

</details>

### Q152. Maven `[ERROR] Source option 5 is no longer supported. Use 8 or later.` after a JDK change. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The POM relies on default compiler level; newer JDK dropped old defaults.

**Diagnose →** `mvn -v`, check `maven.compiler.*`.

**Fix →** Set `<maven.compiler.release>17</maven.compiler.release>`.

**Prevent →** Explicit compiler settings and toolchains.

</details>

### Q153. `JAVA_HOME is not defined correctly` in CI. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong path or JDK missing in the image.

**Diagnose →** `echo $JAVA_HOME; ls $JAVA_HOME/bin/java`.

**Fix →** Use `actions/setup-java@v4` or an image with the JDK; set `JAVA_HOME` accordingly.

**Prevent →** Use pinned images.

</details>

### Q154. Gradle daemon disappears: `Gradle build daemon disappeared unexpectedly (it may have been killed or may have crashed)`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Memory pressure or OOM kill.

**Diagnose →** `dmesg`, container limits, `org.gradle.jvmargs`.

**Fix →** Lower `-Xmx` below container limit, use `--no-daemon` in CI, reduce workers.

**Prevent →** Size runners for builds.

</details>

### Q155. Flaky integration tests fail with `Address already in use`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Fixed ports reused by parallel tests or leftover processes.

**Diagnose →** `ss -ltnp | grep :8080`.

**Fix →** Use random ports (port 0) and clean up processes.

**Prevent →** No fixed ports in tests.

</details>

---

## Deployment, Kubernetes and GitOps Failures

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q156. `kubectl rollout status` hangs: `Waiting for deployment "app" rollout to finish: 1 old replicas are pending termination...` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** New pods not Ready (failing readiness), PDB blocking, long termination grace.

**Diagnose →**
```bash
kubectl get pods -l app=app
kubectl describe pod <new-pod>
kubectl logs <new-pod> --previous
```

**Fix →** Fix the probe or app startup issue; adjust `terminationGracePeriodSeconds`.

**Prevent →** Set `progressDeadlineSeconds` so the pipeline fails fast.

</details>

### Q157. Pods in `CrashLoopBackOff` right after deploy. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** App crash on startup (missing env/config/secret), bad command, failing migrations.

**Diagnose →** `kubectl logs <pod> --previous`, `kubectl describe pod`.

**Fix →** Provide missing config, fix the entrypoint, roll back with `kubectl rollout undo`.

**Prevent →** Config validation in CI and canary rollouts.

</details>

### Q158. `ImagePullBackOff` with `manifest for registry/app:1.4.2 not found`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Deploy ran before push finished, tag typo, wrong registry.

**Diagnose →** `docker manifest inspect registry/app:1.4.2`.

**Fix →** Push the image, correct the tag.

**Prevent →** Deploy by digest from the build output.

</details>

### Q159. Helm: `Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A previous Helm run was killed, leaving the release in `pending-upgrade`.

**Diagnose →** `helm history app -n prod`.

**Fix →** `helm rollback app <last-good> -n prod`, or delete the pending release secret as a last resort.

**Prevent →** Use `--atomic --timeout` and serialize deploys with concurrency control.

</details>

### Q160. Helm: `Error: rendered manifests contain a resource that already exists. Unable to continue with install`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Resource created outside Helm or by another release.

**Diagnose →** `kubectl get <kind> <name> -o yaml | grep -A5 labels`.

**Fix →** Adopt it by adding the labels/annotations `app.kubernetes.io/managed-by=Helm`, `meta.helm.sh/release-name`, `meta.helm.sh/release-namespace`, or delete the resource.

**Prevent →** One owner per resource.

</details>

### Q161. Argo CD app stuck `Progressing` forever. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A Deployment not available, Ingress without an address, or a custom resource with no health check.

**Diagnose →** `argocd app get <app>`; find the resource with non-Healthy status.

**Fix →** Fix the failing workload, or add a custom health check in `argocd-cm`.

**Prevent →** Readiness probes and health customisation for CRDs.

</details>

### Q162. Argo CD: `ComparisonError: rpc error: ... failed to generate manifest ... helm template ... error`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Template error, missing values file, wrong chart version.

**Diagnose →** `argocd app manifests <app>`; run `helm template` locally with the same values.

**Fix →** Fix the chart/values and push.

**Prevent →** CI that runs `helm lint` and `helm template` on every change.

</details>

### Q163. Argo CD `Sync failed: one or more objects failed to apply ... is invalid`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Schema violation, immutable field change (for example Deployment selector, Job spec), or missing CRD.

**Diagnose →** Read the sync result message; `kubectl apply --dry-run=server -f`.

**Fix →** Delete and recreate for immutable fields (use `Replace=true` or `Force=true` carefully), install the CRD first with sync waves.

**Prevent →** `kubeconform` and server-side dry-run in CI.

</details>

### Q164. Flux: `kustomize build failed: accumulating resources: ... no such file or directory`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong `path` in the Kustomization or a missing file reference.

**Diagnose →** `flux get kustomizations`, `kubectl -n flux-system describe kustomization <name>`; `kustomize build <path>` locally.

**Fix →** Correct the path or the resource list.

**Prevent →** Run `kustomize build` in PR checks.

</details>

### Q165. Deployment goes green in CI but users see 502/503 errors for a minute. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Readiness probe passes too early, load balancer not yet registering pods, missing preStop drain.

**Diagnose →** Compare pod Ready timestamps to ALB/ingress target health; check ingress logs.

**Fix →** Fix readiness to reflect real readiness, add `preStop` sleep and `minReadySeconds`.

**Prevent →** Zero-downtime tests in staging with load.

</details>

### Q166. `kubectl apply` in the pipeline fails: `error: You must be logged in to the server (Unauthorized)`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Expired token or kubeconfig, EKS token not refreshed, wrong context.

**Diagnose →** `kubectl config current-context`, `aws eks get-token --cluster-name X`, `aws sts get-caller-identity`.

**Fix →** Refresh credentials (`aws eks update-kubeconfig`), check the identity mapping (access entries or aws-auth).

**Prevent →** Use short-lived OIDC credentials created per job.

</details>

### Q167. `Error from server (Forbidden): deployments.apps "app" is forbidden: User "system:serviceaccount:ci:deployer" cannot patch resource`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** RBAC Role lacks verbs for the resource, or bound in the wrong namespace.

**Diagnose →** `kubectl auth can-i patch deployments -n prod --as=system:serviceaccount:ci:deployer`.

**Fix →** Add the needed verbs to the Role and RoleBinding in the target namespace.

**Prevent →** Define CI RBAC in Git and test with `can-i`.

</details>

### Q168. Database migration job in the pipeline fails with `Lock wait timeout exceeded`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Long-running transaction or app holding locks on the table; migration needs an exclusive lock.

**Diagnose →** `SHOW PROCESSLIST` / `pg_stat_activity` and `pg_locks`.

**Fix →** Run during low traffic, use online schema change tools, set `lock_timeout` and retry, split the migration.

**Prevent →** Review migrations for locking behaviour; use expand/contract.

</details>

### Q169. Flyway: `Validate failed: Migrations have failed validation. Detected applied migration not resolved locally`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A migration applied in the DB is missing from the artifact, often by a branch mismatch.

**Diagnose →** `flyway info`.

**Fix →** Restore the missing migration file; do not edit applied ones; use `repair` only deliberately.

**Prevent →** Ship migrations in the same artifact as the code.

</details>

### Q170. Canary analysis fails with `no data` and the Rollout aborts. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Prometheus query returns an empty vector because of label mismatch or low traffic for the canary.

**Diagnose →** `kubectl argo rollouts get rollout app`, run the query in Prometheus.

**Fix →** Fix labels, add `failureCondition` handling of empty results, ensure test traffic reaches the canary.

**Prevent →** Synthetic traffic during analysis windows.

</details>

### Q171. Argo CD `Application` shows `Unknown` with `repository not accessible`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Expired deploy key or token, wrong URL, network policy.

**Diagnose →** `argocd repo list`, `argocd repo get <url>`.

**Fix →** Update credentials; prefer GitHub App credentials for repos.

**Prevent →** Alert on repo connection failures.

</details>

### Q172. `terraform apply` in CI: `Error acquiring the state lock`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A previous run crashed or is still running; two pipelines race.

**Diagnose →** Read the lock ID and `Who` in the error; check running pipelines.

**Fix →** After confirming nothing runs, `terraform force-unlock <ID>`.

**Prevent →** Serialize with `concurrency` groups and avoid cancelling apply.

</details>

### Q173. Terraform plan in CI shows unexpected diffs that did not appear locally. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Different provider/Terraform versions, missing variables, drift.

**Diagnose →** Compare `terraform version` and provider lock file.

**Fix →** Commit `.terraform.lock.hcl` and pin versions; use the same var files.

**Prevent →** `required_version` and pinned providers.

</details>

### Q174. Lambda deploy by SAM fails: `An error occurred (ResourceConflictException) ... The operation cannot be performed at this time. An update is in progress`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Concurrent update to the same function.

**Diagnose →** `aws lambda get-function --function-name X --query Configuration.LastUpdateStatus`.

**Fix →** Wait with `aws lambda wait function-updated`, serialize deploys.

**Prevent →** Concurrency control on deploy workflows.

</details>

### Q175. ECS service deployment keeps rolling back: `service ... was unable to place a task`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Insufficient CPU/memory in the cluster, no subnets or ENI capacity, failing health checks.

**Diagnose →** `aws ecs describe-services --services X --query 'services[0].events[:5]'`.

**Fix →** Add capacity, fix target group health check, fix image/secrets.

**Prevent →** Deployment circuit breaker with rollback enabled.

</details>

### Q176. Kubernetes `Evicted` pods after a deploy due to ephemeral storage. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Logs or temp data exceed ephemeral storage limits.

**Diagnose →** `kubectl describe pod` (The node was low on resource: ephemeral-storage).

**Fix →** Set ephemeral-storage requests/limits, rotate logs, write to volumes.

**Prevent →** Resource policy checks in admission.

</details>

### Q177. New deployment succeeds but still serves the old version. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Mutable tag (`latest`) with `imagePullPolicy: IfNotPresent`, CDN/cache, wrong service selector.

**Diagnose →** `kubectl get pod -o jsonpath='{.items[*].status.containerStatuses[*].imageID}'`.

**Fix →** Deploy by unique tag or digest, purge caches.

**Prevent →** Immutable tags only.

</details>

### Q178. Argo CD sync stuck on a hook: `waiting for completion of hook batch/Job/db-migrate`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Hook Job failing or never completing; old Job not deleted so name conflicts.

**Diagnose →** `kubectl logs job/db-migrate`, `kubectl describe job`.

**Fix →** Fix the job, add `argocd.argoproj.io/hook-delete-policy: BeforeHookCreation`.

**Prevent →** Idempotent migrations and `backoffLimit`/`activeDeadlineSeconds`.

</details>

### Q179. `Error: INSTALLATION FAILED: failed pre-install: timed out waiting for the condition` with Helm. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A pre-install hook Job did not complete; image pull or DB unreachable.

**Diagnose →** `kubectl get jobs,pods`; `kubectl logs` for the hook pod.

**Fix →** Resolve the hook problem, increase `--timeout` if justified.

**Prevent →** Keep hooks short and idempotent.

</details>

### Q180. Blue-green switch done, but traffic still hits the old stack. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** DNS TTL, client connection reuse, ALB listener rule not updated, Service selector unchanged.

**Diagnose →** `dig +short app.example.com`; check listener rules or `kubectl get svc -o yaml`.

**Fix →** Update the selector/target group weights, lower TTL ahead of time, drain old connections.

**Prevent →** Lower TTL before cutover.

</details>

---

## Secrets, Security and Pipeline Performance

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q181. A secret printed in logs appears as `***` in GitHub Actions but a base64 variant is visible. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Masking only applies to the exact registered value.

**Diagnose →** Search logs for transformed values.

**Fix →** Rotate the secret; register derived values with `::add-mask::`, and avoid printing them.

**Prevent →** Never echo secrets or derived values.

</details>

### Q182. `gitleaks` finds a secret in an old commit, but the file is gone now. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Secrets remain in Git history.

**Diagnose →** `gitleaks detect --log-opts="--all"`.

**Fix →** Rotate the secret, then rewrite history with `git filter-repo` if required.

**Prevent →** Pre-commit scanning and push protection.

</details>

### Q183. Trivy scan fails the pipeline: `FATAL ... failed to download vulnerability DB ... TOOMANYREQUESTS`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Rate limits fetching the vulnerability DB from the public registry.

**Diagnose →** Read the Trivy logs.

**Fix →** Mirror the DB (`--db-repository` to an internal registry) or cache `~/.cache/trivy`.

**Prevent →** Cache the DB and fall back to the Java DB mirror when needed.

</details>

### Q184. `cosign verify` fails: `no matching signatures: none of the expected identities matched`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Certificate identity or issuer values differ from the actual workflow identity (branch, tag, workflow path).

**Diagnose →**
```bash
cosign verify IMAGE --certificate-identity-regexp '.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com | jq '.[0].optional'
```

**Fix →** Use the correct `--certificate-identity` (e.g. `https://github.com/org/repo/.github/workflows/release.yml@refs/tags/v1.0.0`) or a regexp.

**Prevent →** Standardise the signing workflow and its identity.

</details>

### Q185. Kyverno `verifyImages` blocks all deploys after rollout: `image verification failed`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong keys/identity, images never signed, registry unreachable from admission controller.

**Diagnose →** `kubectl describe clusterpolicy`, Kyverno admission logs, `cosign verify` manually.

**Fix →** Switch policy to `Audit` to recover, then correct identity or sign images.

**Prevent →** Roll out in audit mode first and test signed images in staging.

</details>

### Q186. Dependabot/Renovate PRs fail CI with unrelated errors. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Dependabot PRs get a read-only token and no regular secrets; secrets must be added as Dependabot secrets.

**Diagnose →** Check the error for missing secret.

**Fix →** Add required secrets under "Dependabot secrets".

**Prevent →** Make workflows work without secrets for PR validation.

</details>

### Q187. OIDC to Azure/GCP fails: `AADSTS70021: No matching federated identity record found`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Federated credential subject does not match the workflow's `sub` (branch, environment, PR).

**Diagnose →** Compare the token subject with the federated credential's subject.

**Fix →** Create a federated credential for the right entity type (environment/branch).

**Prevent →** Use environments for consistent `sub` claims.

</details>

### Q188. CI pipeline is slow: checkout takes 4 minutes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Huge repo, full history, LFS objects, submodules.

**Diagnose →** Time the checkout step; `git count-objects -vH`.

**Fix →** Shallow clone, sparse checkout, partial clone (`--filter=blob:none`), skip LFS where unneeded, use a cached reference clone.

**Prevent →** Repository hygiene and `git gc` / large file policy.

</details>

### Q189. Queue times spike every morning at 9 AM. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Demand peak against fixed capacity.

**Diagnose →** Graph queue time by hour and by repo; check autoscaler limits.

**Fix →** Autoscale runners, increase max, pre-warm capacity before peak, cancel redundant runs.

**Prevent →** Alert on queue time SLO.

</details>

### Q190. Build time doubled after adding a security scan. How do you recover time? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Serial execution and uncached DB downloads.

**Diagnose →** Stage timing breakdown.

**Fix →** Run in parallel to unit tests, cache scanner DBs, scan only changed paths on PRs and the full scan nightly.

**Prevent →** Performance budget for the pipeline.

</details>

### Q191. Pipeline succeeds but the deployed artifact differs from what was tested. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Rebuild per environment, mutable tags, different base image on rebuild.

**Diagnose →** Compare image digests at each stage.

**Fix →** Build once, pass the digest, deploy by digest.

**Prevent →** Admission policy that requires digests and verified signatures.

</details>

### Q192. `Error: Unable to get ACTIONS_ID_TOKEN_REQUEST_URL env variable`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing `permissions: id-token: write` at job/workflow level.

**Diagnose →** Check `permissions:`; note that defining any `permissions` resets unspecified ones to `none`.

**Fix →** Add `id-token: write` and `contents: read`.

**Prevent →** Template permissions blocks for OIDC jobs.

</details>

### Q193. Secrets work on `main` but not in a reusable workflow called from a different org repo. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Secrets are not inherited across organizations; environment secrets not available.

**Diagnose →** Check the call site for `secrets:` and org visibility.

**Fix →** Pass secrets explicitly; `secrets: inherit` only works within the same org/enterprise.

**Prevent →** Prefer OIDC to avoid secret passing.

</details>

### Q194. Vault login in CI fails: `permission denied` or `claim "bound_claims" does not match`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** JWT/OIDC role bound claims (ref, repo, environment) differ from the token.

**Diagnose →** Decode the JWT payload (`cut -d. -f2 | base64 -d`), read the Vault role: `vault read auth/jwt/role/ci`.

**Fix →** Align `bound_claims` or `bound_subject` with the token claims.

**Prevent →** Keep Vault roles as code and test with real tokens.

</details>

### Q195. Self-hosted runner pods on Kubernetes (ARC) never start jobs: `Failed to create runner ... 401`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** GitHub App or PAT credentials wrong/expired, wrong installation ID, missing permissions.

**Diagnose →** `kubectl logs -n arc-systems deploy/arc-gha-rs-controller`; verify the secret contents.

**Fix →** Recreate the secret with correct app ID, installation ID and private key.

**Prevent →** Rotate and monitor credentials; use a GitHub App.

</details>

### Q196. `Error: HttpError: API rate limit exceeded for installation` in automation. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Too many API calls from workflows or bots sharing a token.

**Diagnose →** `gh api rate_limit`.

**Fix →** Use conditional requests/caching, backoff, spread across GitHub App installations, reduce polling.

**Prevent →** Cache API results and avoid per-commit polling.

</details>

### Q197. Pipeline blocked: `SonarQube quality gate status: ERROR` after upgrading Sonar, though code is unchanged. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** New rules in the upgraded version flag existing code as new issues, or a changed new-code period.

**Diagnose →** Check the rules profile changes and the new-code baseline.

**Fix →** Review rule changes, adjust the profile, and reset baseline if appropriate.

**Prevent →** Test Sonar upgrades on a copy and communicate rule changes.

</details>

### Q198. After rotating a registry credential, only some pipelines fail with `unauthorized: authentication required`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Old credentials cached in agent `~/.docker/config.json`, per-repo secrets not updated.

**Diagnose →** `cat ~/.docker/config.json | jq '.auths | keys'`, grep for the secret name across repos.

**Fix →** Update all copies, log out and in again on agents.

**Prevent →** One org-level secret or OIDC-based registry auth.

</details>

### Q199. Build cache poisoning concern: a PR build populated a cache later used on main. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Caches shared across trust levels.

**Diagnose →** Review cache scope rules and keys.

**Fix →** Scope caches by branch/trust; in GitHub, caches from PR merge refs are only readable by that PR; use read-only restore on release jobs and prefer clean builds for releases.

**Prevent →** Release builds must not restore untrusted caches.

</details>

### Q200. Production deploy failed halfway across five regions. How do you recover and prevent the mixed state? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No wave gating, non-atomic multi-region rollout, no health gates.

**Diagnose →** List the version per region (`kubectl --context region-x get deploy app -o jsonpath='{.spec.template.spec.containers[0].image}'`), check alarms.

**Fix →** Pause the rollout, then either roll forward the failed regions or roll back the updated ones to the last good digest.

**Prevent →** Wave-based rollout with automated bake time, health gates, and per-region rollback.

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
