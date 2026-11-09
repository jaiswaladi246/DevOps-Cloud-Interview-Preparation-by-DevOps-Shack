# Real-World Interview — Night-Before Cheatsheet

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

> The most-asked facts and commands across every tool in this repo, the numbers interviewers expect you to know cold, four answer frameworks, questions to ask, and a day-of checklist. Read it once the night before and once on the morning of.

[⬅ Back to Real-World Interview](./README.md) · [🏠 Home](../README.md)

## Table of contents

- [Linux](#linux)
- [Networking](#networking)
- [Git](#git)
- [Docker](#docker)
- [Kubernetes](#kubernetes)
- [CI/CD](#cicd)
- [Terraform](#terraform)
- [Ansible](#ansible)
- [AWS](#aws)
- [Monitoring and observability](#monitoring-and-observability)
- [DevSecOps](#devsecops)
- [Numbers worth knowing](#numbers-worth-knowing)
- [Answer frameworks](#answer-frameworks)
- [Questions to ask the interviewer](#questions-to-ask-the-interviewer)
- [Day-of checklist](#day-of-checklist)
- [Top 15 interview one-liners](#top-15-interview-one-liners)

---

## Linux

| Task | Command |
|------|---------|
| Disk usage per filesystem / inodes | `df -h` / `df -i` |
| Biggest directories on one filesystem | `du -xh --max-depth=1 / \| sort -h \| tail` |
| Deleted files still held open | `lsof +L1` |
| Top CPU or memory processes | `top`, `htop`, `ps aux --sort=-%mem \| head` |
| Load, CPU steal, I/O wait | `uptime`, `vmstat 1 5`, `iostat -x 1 3`, `mpstat -P ALL 1` |
| Memory, including available | `free -h` (look at `available`, not `free`) |
| OOM kills | `dmesg -T \| grep -i -E 'killed process\|oom'`, `journalctl -k` |
| Service status and logs | `systemctl status nginx`, `journalctl -u nginx --since "10 min ago" -f` |
| Listening ports and owning process | `ss -tulpn` |
| Open files and sockets of a process | `lsof -p <pid>`, `ls -l /proc/<pid>/fd` |
| What a process is doing | `strace -f -p <pid> -e trace=network,file` |
| Find large or recent files | `find /var -xdev -type f -size +500M`, `find . -mmin -10` |
| Permissions | `chmod 640 f` (rw-r-----), `chown app:app f`, `umask 022` |
| Text processing | `grep -E`, `awk '{print $1}'`, `sed -i 's/a/b/g'`, `sort \| uniq -c \| sort -rn` |
| Scheduled jobs | `crontab -l`, `systemctl list-timers` |

**Must know:** a process vs a thread; what load average measures (runnable plus uninterruptible tasks, not CPU percent); zombie vs orphan processes; hard vs soft links (inode vs path); `SIGTERM` (15, polite) vs `SIGKILL` (9, cannot be caught); `/proc` and `/sys`; why `df` and `du` disagree (deleted-but-open files).

## Networking

| Task | Command |
|------|---------|
| DNS lookup, short / full trace | `dig +short api.example.com`, `dig +trace example.com` |
| Query a specific resolver | `dig @8.8.8.8 example.com A` |
| HTTP with timings and TLS detail | `curl -v https://x`, `curl -s -o /dev/null -w '%{http_code} %{time_total}\n' https://x` |
| Force a host to an IP (bypass DNS) | `curl --resolve api.example.com:443:203.0.113.10 https://api.example.com` |
| TCP port reachability | `nc -zv db.internal 5432` |
| Certificate expiry | `echo \| openssl s_client -connect x:443 -servername x 2>/dev/null \| openssl x509 -noout -dates` |
| Path and per-hop loss | `traceroute -T -p 443 x`, `mtr -rwc 50 x` |
| Capture packets | `tcpdump -i any -nn port 53`, `tcpdump -i eth0 host 10.0.1.5 and port 443 -w cap.pcap` |
| Routes and interfaces | `ip route`, `ip addr`, `ip neigh` |

**Must know:** OSI and TCP/IP layers; TCP three-way handshake and `TIME_WAIT`; TCP vs UDP; how DNS resolution works (stub → recursive → root → TLD → authoritative, TTL caching); A, AAAA, CNAME, MX, TXT, NS, SRV, alias; L4 vs L7 load balancing; NAT and why private subnets need it for outbound; CIDR math (a /24 has 256 addresses); TLS handshake and SNI; forward vs reverse proxy.

## Git

| Task | Command |
|------|---------|
| Undo last commit, keep changes | `git reset --soft HEAD~1` |
| Undo a pushed commit safely | `git revert <sha>` |
| Recover "lost" commits | `git reflog`, then `git branch rescue <sha>` |
| Interactive cleanup before PR | `git rebase -i origin/main` |
| Safe force push | `git push --force-with-lease` |
| Who changed this line | `git blame -L 40,60 app.py`, `git log -S 'func_name' --oneline` |
| Find the commit that broke it | `git bisect start; git bisect bad; git bisect good <sha>` |
| Stash including untracked | `git stash -u`, `git stash pop` |
| Cherry-pick a fix to a release branch | `git cherry-pick -x <sha>` |
| Purge a file from history | `git filter-repo --path secrets.env --invert-paths` (and rotate the secret) |

**Must know:** merge vs rebase and when each is right; never rewrite shared history casually; `fetch` vs `pull`; trunk-based vs GitFlow; branch protection and required checks; a leaked secret is fixed by rotation, not by deleting the commit.

## Docker

| Task | Command |
|------|---------|
| Build with a tag / target stage | `docker build -t app:1.4.2 .`, `docker build --target test .` |
| Run detached with port and env | `docker run -d -p 8080:8080 -e ENV=prod --name app app:1.4.2` |
| Shell into a running container | `docker exec -it app sh` |
| Logs | `docker logs -f --tail 100 app` |
| Inspect config, IP, mounts, exit code | `docker inspect app --format '{{.State.ExitCode}}'` |
| Image layers and sizes | `docker history app:1.4.2`, `docker images` |
| Live resource usage | `docker stats` |
| Clean up | `docker system df`, `docker system prune -af --volumes` (careful) |
| Multi-platform build and push | `docker buildx build --platform linux/amd64,linux/arm64 -t r/app:1 --push .` |

**Must know:** image vs container; layers and cache order (dependencies before source); multi-stage builds; `CMD` vs `ENTRYPOINT`; exec form vs shell form (PID 1 and signals); `COPY` vs `ADD`; non-root `USER`; namespaces (isolation) and cgroups (limits); bind mount vs named volume; bridge vs host networking; `.dockerignore`; why `:latest` is not a version.

## Kubernetes

| Task | Command |
|------|---------|
| What is wrong with this pod | `kubectl describe pod <p>` (Events at the bottom) |
| Logs of the crashed container | `kubectl logs <p> --previous`, `-c <container>` for multi-container |
| Events, newest last | `kubectl get events --sort-by=.lastTimestamp` |
| Is the Service wired to pods | `kubectl get endpoints <svc>`, `kubectl get endpointslices -l kubernetes.io/service-name=<svc>` |
| Rollout status, history, undo | `kubectl rollout status deploy/x`, `rollout history`, `rollout undo deploy/x --to-revision=3` |
| Restart all pods of a Deployment | `kubectl rollout restart deploy/x` |
| Resource usage | `kubectl top pod -A --sort-by=memory`, `kubectl top node` |
| Debug a running pod | `kubectl debug -it <p> --image=busybox:1.36 --target=<container>` |
| Throwaway test pod | `kubectl run t --rm -it --restart=Never --image=curlimages/curl:8.10.1 -- sh` |
| Can I / can they | `kubectl auth can-i create deploy -n prod --as=system:serviceaccount:ci:deployer` |
| Port-forward | `kubectl port-forward svc/x 8080:80` |
| Drain and return a node | `kubectl drain <n> --ignore-daemonsets --delete-emptydir-data`, `kubectl uncordon <n>` |
| Explain a field | `kubectl explain deploy.spec.strategy.rollingUpdate` |

**Pod status cheat codes**

| Status | Usual cause | First check |
|--------|-------------|-------------|
| `Pending` | No node fits: resources, taints, affinity, unbound PVC | `describe` events: `0/3 nodes are available: ...` |
| `ImagePullBackOff` / `ErrImagePull` | Wrong name or tag, private registry without credentials | `describe`: `manifest unknown` or `401 Unauthorized` |
| `CrashLoopBackOff` | App exits repeatedly: config, dependency, bad command | `logs --previous`, exit code in `describe` |
| `OOMKilled` (exit 137) | Container exceeded memory limit | `describe` last state, memory metrics, raise limit or fix leak |
| `CreateContainerConfigError` | Missing ConfigMap or Secret key | `describe`: `secret "x" not found` |
| Running but not Ready | Readiness probe failing | `describe` probe failures, curl the probe path inside the pod |
| `Terminating` forever | Finalizer or unreachable node | `kubectl get pod -o yaml \| grep finalizers`, node status |
| `Evicted` | Node pressure (memory, disk) | `describe` node conditions, `kubectl get pod -o wide` |

**Must know:** control plane components (API server, etcd, scheduler, controller manager) and node components (kubelet, kube-proxy, container runtime); Deployment vs StatefulSet vs DaemonSet vs Job; Service types (ClusterIP, NodePort, LoadBalancer, ExternalName, headless); Ingress vs Gateway API; liveness vs readiness vs startup probes; requests vs limits and QoS classes (Guaranteed, Burstable, BestEffort); HPA vs VPA vs Cluster Autoscaler or Karpenter; PDBs only stop voluntary disruptions (evictions), not `kubectl delete pod` or node crashes; RBAC (Role, ClusterRole, bindings, ServiceAccounts); ConfigMap vs Secret (base64 is not encryption); NetworkPolicy needs a CNI that enforces it.

## CI/CD

| Concept | One-liner |
|---------|-----------|
| CI vs CD vs CD | Integrate and test every change / always releasable / automatically released |
| Pipeline order | Lint → unit test → build → scan → push artifact → deploy staging → test → promote → deploy prod → verify |
| Immutable artifacts | Build once, tag with the commit SHA, promote the same artifact through environments |
| GitOps | Desired state in Git, an in-cluster agent (ArgoCD, Flux) pulls and reconciles, drift is corrected |
| Deployment strategies | Rolling, blue/green (instant switch, double capacity), canary (gradual with metrics), feature flags (decouple deploy from release) |
| Secrets in pipelines | OIDC federation to the cloud, short-lived tokens, never long-lived keys in variables |
| Speed | Cache dependencies and Docker layers, parallelise tests, run only what changed |

```yaml
# GitHub Actions essentials
on: { pull_request: {}, push: { branches: [main] } }
permissions: { contents: read, id-token: write }   # least privilege + OIDC
concurrency: { group: deploy-prod, cancel-in-progress: false }
jobs:
  test:  { runs-on: ubuntu-latest, steps: [ { uses: actions/checkout@v4 }, { run: make test } ] }
  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    environment: production          # required reviewers, protected secrets
    runs-on: ubuntu-latest
    steps: [ { run: ./deploy.sh } ]
```

**Jenkins in one breath:** controller schedules, agents execute; declarative `Jenkinsfile` with `pipeline { agent; stages; post }`; credentials via `withCredentials`; shared libraries for reuse; ephemeral Kubernetes agents instead of static VMs.

## Terraform

| Task | Command |
|------|---------|
| Init, format, validate | `terraform init`, `terraform fmt -recursive`, `terraform validate` |
| Plan to a file, apply that plan | `terraform plan -out tfplan`, `terraform apply tfplan` |
| Target one resource (emergency only) | `terraform apply -target=aws_instance.web` |
| Inspect state | `terraform state list`, `terraform state show aws_s3_bucket.logs` |
| Rename or move in state | `moved {}` block, or `terraform state mv a b` |
| Adopt an existing resource | `import {}` block (1.5+), or `terraform import aws_s3_bucket.logs my-bucket` |
| Detect drift | `terraform plan -refresh-only`, `terraform plan -detailed-exitcode` (exit 2 = changes) |
| Force recreate | `terraform apply -replace=aws_instance.web` |
| Stuck lock | `terraform force-unlock <LOCK_ID>` (only when sure nothing is running) |
| Output for scripts | `terraform output -raw cluster_name`, `terraform show -json tfplan` |

**Must know:** state is the map between config and real resources, keep it remote, encrypted, versioned and locked (S3 with `use_lockfile = true` on 1.10+, or the older DynamoDB lock table); `count` vs `for_each` (for_each keys are stable); modules and versioning; workspaces vs separate state per environment; `lifecycle` (`prevent_destroy`, `create_before_destroy`, `ignore_changes`); data sources; "forces replacement" in a plan means destroy and recreate; never commit `.tfstate` or secrets in `.tfvars`.

## Ansible

| Task | Command |
|------|---------|
| Connectivity test | `ansible all -i inventory.ini -m ping` |
| Ad-hoc command | `ansible web -i inventory.ini -b -a "systemctl status nginx"` |
| Dry run with diff | `ansible-playbook site.yml --check --diff` |
| Limit hosts / tags | `--limit web1`, `--tags nginx`, `--skip-tags slow` |
| Debug verbosity | `-vvv` |
| List what would run | `--list-hosts`, `--list-tasks` |
| Encrypt a secret | `ansible-vault encrypt_string 's3cr3t' --name db_password` |
| Lint | `ansible-lint site.yml` |

**Must know:** agentless over SSH (WinRM for Windows); idempotency (modules describe desired state; `command` and `shell` need `creates`, `removes` or `changed_when`); handlers run once at the end of a play when notified; roles structure (`tasks`, `handlers`, `templates`, `defaults`, `vars`); variable precedence (extra vars `-e` win); facts; `serial` for rolling updates; Ansible (configure machines) vs Terraform (provision infrastructure).

## AWS

| Task | Command |
|------|---------|
| Who am I | `aws sts get-caller-identity` |
| Instances with state | `aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,State.Name,PrivateIpAddress]' --output table` |
| Shell without SSH | `aws ssm start-session --target i-0abc123` |
| Kubeconfig for EKS | `aws eks update-kubeconfig --name prod --region ap-south-1` |
| Target health behind an ALB | `aws elbv2 describe-target-health --target-group-arn <arn>` |
| Who did that | CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteBucket` |
| Why AccessDenied | `aws iam simulate-principal-policy --policy-source-arn <role> --action-names s3:GetObject` |
| Tail logs | `aws logs tail /aws/lambda/fn --follow --since 10m` |

**Must know:** Region vs AZ; VPC, public vs private subnet (route to IGW or not), NAT gateway; security groups (stateful, allow only) vs NACLs (stateless, allow and deny); IAM users vs roles, trust policy vs permission policy, explicit deny wins; EC2 instance profiles, IRSA or EKS Pod Identity for pods; ALB (L7) vs NLB (L4); RDS Multi-AZ (HA) vs read replica (scale, DR); S3 storage classes, versioning, Block Public Access; CloudWatch vs CloudTrail vs Config; the usual cost surprises (NAT data processing, cross-AZ traffic, idle load balancers, unattached EBS).

## Monitoring and observability

```promql
# Requests per second
sum(rate(http_requests_total[5m]))
# Error ratio
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
# p95 latency from a histogram
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
# Container memory vs limit
sum by (pod) (container_memory_working_set_bytes{container!=""})
  / sum by (pod) (kube_pod_container_resource_limits{resource="memory"})
# Disk will fill in 4 hours
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 4 * 3600) < 0
# Target down
up == 0
```

```logql
{namespace="prod", app="checkout"} |= "error" | json | status >= 500
sum by (pod) (count_over_time({namespace="prod"} |= "OOM" [5m]))
```

**Must know:** metrics vs logs vs traces; counters (use `rate`) vs gauges vs histograms; pull (Prometheus) vs push; RED (rate, errors, duration) for services and USE (utilisation, saturation, errors) for resources; four golden signals (latency, traffic, errors, saturation); SLI, SLO, SLA, error budget; alert on symptoms users feel, not every cause; cardinality (never put user IDs in labels); Alertmanager grouping, inhibition, silences; OpenTelemetry as the vendor-neutral instrumentation standard.

## DevSecOps

| Gate | Tool examples | Where |
|------|---------------|-------|
| Secrets in code | gitleaks, Trivy secret scanner, GitHub push protection | Pre-commit and CI |
| SAST | Semgrep, CodeQL, SonarQube | CI on PR |
| Dependencies (SCA) | Trivy, Grype, Dependabot, Snyk | CI and scheduled |
| IaC misconfiguration | Trivy config, Checkov, tfsec rules in Trivy | CI on PR |
| Image vulnerabilities | Trivy image, Grype, registry scanning (ECR) | CI before push, registry continuously |
| Signing and provenance | cosign, SLSA provenance, SBOM (Syft, Trivy) | CI after build |
| Admission policy | Kyverno, OPA Gatekeeper, Pod Security Admission | Cluster |
| Runtime detection | Falco, GuardDuty EKS runtime monitoring | Cluster and cloud |
| DAST | OWASP ZAP | Against staging |

```bash
trivy fs --scanners vuln,secret,misconfig --severity HIGH,CRITICAL --exit-code 1 .
trivy image --ignore-unfixed --severity HIGH,CRITICAL --exit-code 1 ghcr.io/org/app:sha
trivy config infra/terraform
cosign sign --yes ghcr.io/org/app@sha256:<digest>
```

**Must know:** shift left but also shield right; least privilege everywhere (IAM, RBAC, CI tokens); no long-lived credentials (OIDC, instance roles, Pod Identity); container hardening (non-root, read-only rootfs, drop capabilities, no privileged, seccomp `RuntimeDefault`); Pod Security Standards (privileged, baseline, restricted); secrets managers (Vault, AWS Secrets Manager with External Secrets Operator); supply-chain attacks (pin actions and images by digest, verify signatures).

---

## Numbers worth knowing

### HTTP status codes

| Code | Meaning | Interview angle |
|------|---------|-----------------|
| 200 / 201 / 204 | OK / Created / No Content | |
| 301 / 302 / 304 | Permanent redirect / temporary redirect / not modified (cache) | HTTP to HTTPS redirect is a 301 |
| 400 | Bad request | Client sent something malformed |
| 401 | Unauthenticated | No or bad credentials |
| 403 | Forbidden | Authenticated but not allowed (IAM, RBAC, WAF, S3 policy) |
| 404 | Not found | Wrong path, or ingress rule not matching |
| 405 | Method not allowed | |
| 408 / 413 / 429 | Request timeout / payload too large / too many requests | 413 from NGINX ingress: `proxy-body-size`; 429 is rate limiting |
| 500 | Internal server error | The app threw |
| 502 | Bad gateway | Proxy got an invalid response or connection refused or reset from upstream: crashed pod, wrong port |
| 503 | Service unavailable | No healthy upstream (no endpoints, all targets unhealthy), overload, maintenance |
| 504 | Gateway timeout | Upstream too slow; proxy or LB timeout hit |

### Ports

| Port | Service | Port | Service |
|------|---------|------|---------|
| 22 | SSH | 6443 | Kubernetes API server |
| 25 / 587 | SMTP / submission | 2379-2380 | etcd client / peer |
| 53 | DNS (UDP and TCP) | 10250 | kubelet API |
| 80 / 443 | HTTP / HTTPS | 30000-32767 | NodePort range |
| 123 | NTP (UDP) | 9090 | Prometheus |
| 389 / 636 | LDAP / LDAPS | 9093 | Alertmanager |
| 3306 | MySQL | 9100 | node_exporter |
| 5432 | PostgreSQL | 3000 | Grafana |
| 6379 | Redis | 3100 | Loki |
| 27017 | MongoDB | 9200 / 5601 | Elasticsearch / Kibana |
| 9092 | Kafka | 4317 / 4318 | OTLP gRPC / HTTP |
| 8080 | Jenkins (default), common app port | 8200 | Vault |
| 2375 / 2376 | Docker daemon plain / TLS (never expose 2375) | 4789 / 8472 | VXLAN (standard / Flannel) |

### SLO math

| Availability | Downtime per 30 days | Per year |
|--------------|----------------------|----------|
| 99% | 7 h 12 min | 3.65 days |
| 99.5% | 3 h 36 min | 1.83 days |
| 99.9% | 43.2 min | 8.76 h |
| 99.95% | 21.6 min | 4.38 h |
| 99.99% | 4.32 min | 52.6 min |
| 99.999% | 26 s | 5.26 min |

- **Error budget** = 1 − SLO. 99.9 percent over 30 days = 43.2 minutes, or 1 failed request in 1,000.
- **Burn rate** = actual error rate ÷ budgeted error rate. Burn rate 1 spends the budget exactly in the window.
- **Classic multi-window alerts (30-day SLO):** 14.4x over 1 h (2 percent of budget) pages; 6x over 6 h (5 percent) pages; 1x over 3 days (10 percent) opens a ticket. Each paired with a short window (5 min, 30 min, 6 h) so it resolves fast.
- **Serial dependencies multiply:** two services at 99.9 percent in series give about 99.8 percent. Redundant copies in parallel raise it: two independent 99 percent replicas give 99.99 percent.

### Kubernetes defaults

| Setting | Default |
|---------|---------|
| `terminationGracePeriodSeconds` | 30 s |
| Rolling update `maxSurge` / `maxUnavailable` | 25% / 25% |
| `revisionHistoryLimit` | 10 |
| Probe `periodSeconds` / `timeoutSeconds` / `failureThreshold` | 10 s / 1 s / 3 |
| Probe `initialDelaySeconds` / `successThreshold` | 0 / 1 |
| CrashLoopBackOff restart delay | 10 s, doubling, capped at 5 min; resets after 10 min running |
| Max pods per node (kubelet) | 110 (EKS with VPC CNI: depends on instance ENIs, e.g. 17 on t3.medium) |
| Tolerations for `not-ready` / `unreachable` | 300 s before pods are evicted from a failed node |
| Service type | ClusterIP |
| `imagePullPolicy` | `Always` for `:latest` or no tag, otherwise `IfNotPresent` |
| Pod DNS | `<svc>.<ns>.svc.cluster.local`, `ndots:5` |
| Job `backoffLimit` | 6 |
| HPA scale-down stabilisation | 300 s |
| Projected ServiceAccount token lifetime | 1 h (auto-rotated by kubelet) |
| kube-proxy mode | iptables |
| CPU units | `1` = 1 vCPU = `1000m` |

### Exit codes and signals

| Exit code | Meaning |
|-----------|---------|
| 0 | Success |
| 1 | General error (app-defined) |
| 2 | Misuse of shell builtin, bad arguments |
| 126 | Command found but not executable (permissions) |
| 127 | Command not found (`exec: "x": executable file not found in $PATH`) |
| 128 + n | Killed by signal n |
| 130 | SIGINT (Ctrl-C) |
| 137 | SIGKILL: OOM killer or a forced kill after grace period |
| 139 | SIGSEGV: segfault |
| 143 | SIGTERM: normal graceful stop |

Signals: `1 HUP` (reload), `2 INT`, `9 KILL` (cannot be caught), `15 TERM` (default for `kill` and Kubernetes stop), `19 STOP`.

### AWS numbers

| Fact | Value |
|------|-------|
| Reserved IPs per subnet | 5 (first four and last) |
| VPC CIDR size | /16 to /28 |
| Instance metadata endpoint | `169.254.169.254` (require IMDSv2) |
| ALB idle timeout default | 60 s |
| Target group deregistration delay default | 300 s |
| RDS Multi-AZ failover | typically 60 to 120 s, same endpoint |
| S3 max object / max single PUT | 5 TB / 5 GB (multipart above that) |
| S3 durability | 99.999999999% (11 nines) |
| Lambda max timeout / memory | 15 min / 10,240 MB |
| gp3 baseline | 3,000 IOPS and 125 MB/s, independent of size |
| EKS control plane price | 0.10 USD per hour per cluster (standard support) |

---

## Answer frameworks

### STAR for behavioral questions

| Part | Share of time | Contains |
|------|---------------|----------|
| **S**ituation | 15% | Team, system, scale, why it mattered |
| **T**ask | 10% | Your specific responsibility |
| **A**ction | 55% | What **you** did, in order, with tools and decisions and why |
| **R**esult | 20% | Numbers, outcome, and the lesson or systemic change |

Aim for 2 to 3 minutes. Prepare 8 stories that cover: an incident you handled, a mistake, a conflict, influencing without authority, a tight deadline, improving something nobody asked you to, mentoring or helping a teammate, and a failure you learned from. Most behavioral questions map to one of them.

### Scenario questions: Clarify → Mitigate → Diagnose → Fix → Prevent

1. **Clarify** (30 s): scope, impact, environment, what changed, timeline.
2. **State assumptions** out loud when the interviewer will not give details.
3. **Mitigate** first if users are affected: roll back, fail over, scale, disable a flag, shed load.
4. **Diagnose** with hypotheses and exact commands; say what each result would tell you.
5. **Fix** the root cause properly and **verify** with metrics, not hope.
6. **Prevent**: test, alert, guardrail, runbook, postmortem.

### Troubleshooting: follow the path, check what changed

- **What changed?** Deploys, config, infra, traffic, certificates, dependencies, cron jobs. Most incidents follow a change.
- **Follow the request path** and test each hop: client → DNS → CDN or LB → ingress → Service → pod → app → dependency (DB, cache, external API).
- **Bottom up when nothing makes sense:** is it running → can it reach the network → does name resolution work → is the port open → does the app respond → is the response right.
- **Split the problem in half:** one pod or all, one AZ or all, one endpoint or all, since when.
- **Use the right lens:** RED for services, USE for resources, logs for the "why", traces for "where the time goes".

### Live hands-on task approach

1. **Restate the task** in your own words and confirm the expected output.
2. **Ask about edge cases and constraints:** input size, empty input, error handling, which tools are allowed, target OS.
3. **Say your plan** in two or three sentences before typing.
4. **Build the simplest working version first,** run it, and show the output.
5. **Harden it:** errors, timeouts, quoting, `set -euo pipefail`, idempotency, exit codes.
6. **Test before you say done:** a happy path, a failure case, a second run for idempotent tasks.
7. **Narrate throughout,** and if stuck, say what you are trying and why; interviewers give hints to people who talk.
8. **Close with trade-offs:** what you would add with more time (tests, logging, packaging, CI).

---

## Questions to ask the interviewer

Pick two or three. Good questions show you think like someone who will be on call there.

**About the work**
- What does a typical week look like for this role: project work vs operational work vs interruptions?
- What is the biggest reliability or delivery problem the team is working on this quarter?
- How often do you deploy to production, and what does a rollback look like?
- What would you expect me to have achieved in the first 90 days?

**About on-call and operations**
- What does on-call look like: rotation size, pages per week, and is it compensated?
- When did you last have a major incident, and what changed afterwards?
- Do you write postmortems, and are they blameless in practice?

**About the team and growth**
- How are decisions about tooling made, and could I propose changes?
- How do engineers here learn: conference budget, certifications, internal talks?
- What do people who do well in this team have in common?

**Avoid** asking about things the job description already answers, and leave salary and leave policy for the HR round.

---

## Day-of checklist

**The night before**
- [ ] Re-read the job description and map each requirement to one story or project
- [ ] Re-read your own resume line by line; you can explain every tool and number on it
- [ ] Rehearse the 2-minute introduction out loud twice
- [ ] Skim this cheatsheet, then stop and sleep; cramming at 2 a.m. costs more than it gains

**One hour before**
- [ ] Laptop charged, charger plugged in, notifications and chat apps off
- [ ] Camera at eye level, face lit from the front, plain background, headphones with a mic tested
- [ ] Meeting link opened and tested; backup phone hotspot ready
- [ ] For live rounds: terminal font enlarged, a clean shell prompt, editor ready, `kubectl` and `docker` working if they will be needed, a sandbox (kind or a VM) up
- [ ] Water nearby, a notepad and pen for drawing and jotting the interviewer's names
- [ ] Your capstone repo and architecture diagram open in a tab, ready to share

**During**
- [ ] Ask one clarifying question before answering scenarios
- [ ] Lead with the answer, then the why; stop and offer more instead of rambling
- [ ] Say "I don't know, but here is how I'd find out" instead of bluffing
- [ ] Think out loud in live rounds; never go silent for more than 30 seconds
- [ ] Use "I" for your work, numbers wherever you have them
- [ ] Leave 5 minutes for your questions

**After**
- [ ] Write down every question you were asked within an hour, while you still remember them
- [ ] Note the two answers you would improve, and look them up in this repo
- [ ] Send a short thank-you note to the recruiter if that is the norm for the company

---

## Top 15 interview one-liners

1. **"Mitigate first, investigate second."** If users are affected, roll back or fail over before hunting for the root cause.
2. **"Most incidents follow a change, so my first question is what changed."**
3. **"A 502 means the upstream answered badly or refused; a 503 means there was no healthy upstream; a 504 means it was too slow."**
4. **"Readiness decides traffic, liveness decides restarts; never make liveness depend on a shared dependency."**
5. **"A PDB protects against evictions, not against `kubectl delete pod` or a node dying."**
6. **"Build once, tag with the commit SHA, promote the same artifact through every environment."**
7. **"In GitOps, Git is the source of truth and the cluster pulls; rollback is a revert."**
8. **"Terraform state is the map between code and reality, so it lives remote, encrypted, versioned and locked."**
9. **"'Forces replacement' on a stateful resource means data loss until proven otherwise."**
10. **"Idempotent means running it twice changes nothing the second time."**
11. **"Requests are for scheduling, limits are for protection; I set memory limits and usually skip CPU limits for latency-sensitive services."**
12. **"Alert on symptoms users feel and on error-budget burn, not on every CPU spike."**
13. **"99.9 percent over 30 days is 43 minutes of budget; spend it on shipping, not on surprises."**
14. **"No long-lived credentials anywhere: OIDC for CI, roles for machines, Pod Identity or IRSA for pods."**
15. **"A leaked secret is fixed by rotating it, not by deleting the commit."**

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
