# Kubernetes — Production & Best Practices

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

> Real-world Kubernetes design, security, scale, cost and operations, plus rapid-fire commands · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Kubernetes](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Cluster Design & Multi-Tenancy](#cluster-design--multi-tenancy) — Q1–Q25
- [Reliability & Rollouts](#reliability--rollouts) — Q26–Q51
- [Autoscaling, Capacity & Cost](#autoscaling-capacity--cost) — Q52–Q76
- [Security Hardening & Compliance](#security-hardening--compliance) — Q77–Q101
- [Observability & Day-2 Operations](#observability--day-2-operations) — Q102–Q126
- [EKS in Production](#eks-in-production) — Q127–Q151
- [Upgrades, Backup & Disaster Recovery](#upgrades-backup--disaster-recovery) — Q152–Q175
- [Rapid-Fire Commands](#rapid-fire-commands) — Q176–Q200

---

## Cluster Design & Multi-Tenancy

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. How many clusters should a company run, and how do you decide? 🟢

<details>
<summary><b>Answer</b></summary>

There is no magic number; it is a blast-radius and ownership trade-off. We usually split by environment (at least prod vs non-prod), and by compliance or region boundary. Many small clusters cost more in control-plane fees and operations; one giant shared cluster increases blast radius and upgrade risk. A common middle ground: one prod cluster per region, shared non-prod clusters, dedicated clusters for regulated workloads.

</details>

### Q2. Namespace per environment or per team: which do you pick? 🟢

<details>
<summary><b>Answer</b></summary>

Separate environments belong in separate clusters (or at least separate node pools and accounts), not just namespaces. Inside a cluster, we use a namespace per team or per application, and attach RBAC, ResourceQuota, LimitRange and default-deny NetworkPolicy to each namespace through a template or GitOps.

</details>

### Q3. What namespace conventions and labels do you standardize on? 🟢

<details>
<summary><b>Answer</b></summary>

Namespaces like `team-app-env`, and mandatory labels on workloads: `app.kubernetes.io/name`, `app.kubernetes.io/version`, `app.kubernetes.io/part-of`, plus `owner`, `cost-center` and `env`. Consistent labels power cost allocation, alerting routing, NetworkPolicy and policy engines.
```yaml
labels:
  app.kubernetes.io/name: payments
  owner: team-payments
  cost-center: cc-1042
```

</details>

### Q4. Why should you never run workloads on the control plane nodes? 🟢

<details>
<summary><b>Answer</b></summary>

Control-plane nodes are tainted (`node-role.kubernetes.io/control-plane:NoSchedule`) so that noisy apps cannot starve the API server or etcd. On managed Kubernetes this is moot because you cannot reach them. On self-managed clusters keep the taint and dedicate worker nodes.

</details>

### Q5. What is the minimum HA setup for a self-managed control plane? 🟢

<details>
<summary><b>Answer</b></summary>

Three control-plane nodes across three failure domains with a stable load-balanced API endpoint, and etcd either stacked on those nodes or external (3 members). Two nodes give no fault tolerance for etcd. Worker nodes should span at least three zones too.

</details>

### Q6. What goes in a "cluster baseline" before apps are deployed? 🟢

<details>
<summary><b>Answer</b></summary>

CNI with NetworkPolicy enforcement, CoreDNS tuned, metrics-server, ingress/Gateway controller, cert-manager, ExternalDNS, secrets integration, logging and metrics agents, policy engine, autoscaler, default StorageClass, PriorityClasses, namespace templates and RBAC groups mapped from SSO. We install all of it through GitOps so every cluster is identical.

</details>

### Q7. Why use separate node pools? 🟢

<details>
<summary><b>Answer</b></summary>

To isolate workload types with different needs: system add-ons, general apps, memory-heavy jobs, GPU, Spot batch, or untrusted tenants. You taint and label pools, and use tolerations plus node affinity so each workload lands where it belongs, and you can size, upgrade and scale pools independently.

</details>

### Q8. What do you put on dedicated "system" nodes? 🟢

<details>
<summary><b>Answer</b></summary>

Cluster-critical add-ons (CoreDNS, ingress controllers, CNI helpers, metrics, autoscaler, Karpenter itself) on a small on-demand node group tainted `CriticalAddonsOnly`. That keeps them stable and away from noisy tenants, and prevents the autoscaler from being scheduled onto nodes it manages.

</details>

### Q9. How do you design multi-tenancy: soft vs hard isolation? 🟡

<details>
<summary><b>Answer</b></summary>

Soft (trusted teams): namespaces + RBAC + quotas + NetworkPolicy + PSA + policy engine. Harder: dedicated node pools per tenant, `RuntimeClass` sandboxes (gVisor/Kata), separate ingress and observability scopes. Hard (untrusted or regulated tenants): separate clusters or virtual clusters (vCluster) with their own control plane. Namespaces alone never stop kernel-level escapes.

</details>

### Q10. How do you enforce guardrails with ResourceQuota, LimitRange and policy together? 🟡

<details>
<summary><b>Answer</b></summary>

Quota caps a team's total, LimitRange fills in defaults so nothing runs without requests, and policy (Kyverno/VAP) rejects Pods with no limits-or-requests, `latest` tags, missing probes or missing labels. Quota is how we make costs a team-level decision rather than a platform-level surprise.
```yaml
apiVersion: v1
kind: LimitRange
metadata: { name: defaults }
spec:
  limits:
  - type: Container
    default: { memory: 512Mi }
    defaultRequest: { cpu: 100m, memory: 128Mi }
```

</details>

### Q11. How would you structure Git repositories and environments for GitOps? 🟡

<details>
<summary><b>Answer</b></summary>

Separate app source repos from an environment/config repo. In the config repo, a base plus overlays per environment (Kustomize or Helm values), promoted by pull request. Argo CD or Flux in each cluster pulls and reconciles, with `prune` and `selfHeal` enabled. Production changes need review, and drift from manual edits is reverted automatically.

</details>

### Q12. Argo CD vs Flux: how do you choose? 🟡

<details>
<summary><b>Answer</b></summary>

Argo CD gives a rich UI, ApplicationSets and a straightforward multi-cluster model with RBAC and SSO, popular for platform teams wanting visibility. Flux is a set of lean controllers (Source, Kustomize, Helm, Notification), composes well with Kubernetes-native tooling, and fits Git-only workflows. Both reconcile from Git; choose by team preference and multi-tenancy needs.

</details>

### Q13. Control plane per cluster costs money. How do virtual clusters or hosted control planes help? 🟡

<details>
<summary><b>Answer</b></summary>

vCluster (and Cluster API with hosted control planes like Kamaji or HyperShift) run a tenant's API server and etcd as Pods inside a host cluster, sharing the worker nodes. Tenants get cluster-admin on their virtual cluster for CRDs and namespaces without touching the host, at a fraction of the cost of real clusters.

</details>

### Q14. How do you decide between managed (EKS/GKE/AKS) and self-managed Kubernetes? 🟡

<details>
<summary><b>Answer</b></summary>

Managed removes control plane operations, patching and etcd backup, and integrates with IAM and load balancers. Self-managed makes sense for on-prem, air-gapped, extreme customization (API server flags, custom schedulers) or special hardware. Cost of engineers' time usually outweighs control-plane fees, so we default to managed unless there is a hard constraint.

</details>

### Q15. How do you onboard a new team to the platform in a repeatable way? 🟡

<details>
<summary><b>Answer</b></summary>

A self-service request (PR or portal) generates a namespace with labels, quotas, RBAC groups tied to SSO, default NetworkPolicies, a ServiceAccount for CI with scoped rights, and an Argo CD AppProject restricting repos, clusters and namespaces. Everything is code, reviewed and reversible; no one gets cluster-admin by default.

</details>

### Q16. What is the platform-engineering view of Kubernetes: golden paths and abstractions? 🟡

<details>
<summary><b>Answer</b></summary>

Developers should not hand-write 400 lines of YAML. We provide a golden path: a Helm chart or Crossplane/Score abstraction with sensible defaults (probes, PDB, HPA, security context), plus templates in an internal developer portal (Backstage). Teams may deviate, but the paved road is the supported, secure, cheapest option.

</details>

### Q17. How do you manage configuration per environment without duplicating manifests? 🟡

<details>
<summary><b>Answer</b></summary>

Keep one base and layer differences: Kustomize overlays (replicas, resources, ingress hosts), or one Helm chart with `values-dev.yaml`/`values-prod.yaml`. Secrets never go in those files in plaintext; reference them through External Secrets or sealed secrets. Promote the same image digest across environments, changing only config.

</details>

### Q18. What is cluster-API (CAPI) and when is it worthwhile? 🟡

<details>
<summary><b>Answer</b></summary>

Cluster API manages clusters declaratively as Kubernetes objects (`Cluster`, `MachineDeployment`) via providers for AWS, Azure, vSphere, etc., from a management cluster. It pays off when you operate many clusters or on-prem fleets and want consistent lifecycle and upgrades. For a handful of EKS clusters, Terraform plus GitOps is usually simpler.

</details>

### Q19. How do you handle shared services like ingress, DNS and certs across tenants? 🟡

<details>
<summary><b>Answer</b></summary>

Run them centrally in platform-owned namespaces. A shared ingress/Gateway with per-team `HTTPRoute`s (restricted by `allowedRoutes`), ExternalDNS creating records from annotations, and cert-manager issuing certificates via a `ClusterIssuer`. Teams own routes, the platform owns the load balancer and certificate issuance policy.

</details>

### Q20. Design a Kubernetes platform for 50 microservice teams on AWS. What are your key decisions? 🔴

<details>
<summary><b>Answer</b></summary>

Multi-account AWS (prod, non-prod, shared services), one EKS cluster per env per region, private endpoints, Karpenter for compute with system node group, Cilium or VPC CNI with NetworkPolicy, Gateway API for ingress, Argo CD with ApplicationSets, Kyverno guardrails, IRSA or Pod Identity per workload, Prometheus/Grafana/Loki or managed equivalents, quotas per namespace, Backstage templates, and a quarterly upgrade cadence. Decisions are ordered by blast radius and by what is hard to change later (networking, account structure, IAM model).

</details>

### Q21. How do you plan IP address space for large clusters, especially on AWS? 🔴

<details>
<summary><b>Answer</b></summary>

Reserve large non-overlapping CIDRs early; Pod IPs on VPC CNI consume subnet addresses. We use /19 or /18 private subnets per AZ for nodes and Pods, or a secondary CIDR (100.64.0.0/10) with custom networking, or prefix delegation, or IPv6. Check the service CIDR and peering/Transit Gateway overlap. Running out of IPs is a painful, disruptive fix after launch.

</details>

### Q22. What are the trade-offs of overlay networking vs native VPC routing? 🔴

<details>
<summary><b>Answer</b></summary>

Overlay (VXLAN/Geneve via Calico/Flannel/Cilium) works anywhere and conserves VPC IPs but adds encapsulation overhead (MTU, CPU) and makes Pods unreachable from the VPC without gateways. Native VPC IPs (AWS VPC CNI) give direct reachability, security-group integration and lower latency, at the price of IP consumption and ENI limits.

</details>

### Q23. How would you isolate noisy-neighbor and untrusted workloads on shared nodes? 🔴

<details>
<summary><b>Answer</b></summary>

Combine hard limits on memory, PIDs and ephemeral storage; Guaranteed QoS for sensitive services; `PriorityClass` and PDB; dedicated, tainted node pools per tier; `RuntimeClass` with gVisor/Kata for untrusted code; and per-namespace quotas. For CPU-sensitive workloads use the static CPU manager policy to pin cores.

</details>

### Q24. When would you choose a service mesh and what do you pay for it? 🔴

<details>
<summary><b>Answer</b></summary>

When you need uniform mTLS, fine-grained traffic policy, retries/timeouts, progressive delivery and consistent L7 telemetry across many services and teams. The cost: added latency and memory per Pod (sidecar) or per node (ambient), operational complexity, upgrade coordination and debugging layers. For fewer than a dozen services, an ingress, NetworkPolicy and app-level TLS may suffice.

</details>

### Q25. How do you design for multi-region Kubernetes? 🔴

<details>
<summary><b>Answer</b></summary>

Run independent clusters per region with identical GitOps config, global traffic steering (Route 53 latency/failover, Global Accelerator), and data replicated by the data layer (Aurora Global, DynamoDB global tables) since Kubernetes does not replicate state. Avoid stretching one cluster across regions: etcd and API latency make it fragile. Test failover with game days.

</details>

## Reliability & Rollouts

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. What are the must-have settings for a production Deployment? 🟢

<details>
<summary><b>Answer</b></summary>

At least 2 replicas with topology spread, readiness and liveness (and startup if slow) probes, CPU/memory requests and a memory limit, a PDB, `terminationGracePeriodSeconds` tuned to the app, a non-root `securityContext`, an immutable image tag or digest, and `maxUnavailable: 0` or a low value for rollouts. Missing any of these shows up as an outage eventually.

</details>

### Q27. Why use image digests or immutable tags instead of `latest`? 🟢

<details>
<summary><b>Answer</b></summary>

`latest` is mutable: a node that pulls later may run different code than one that pulled earlier, rollbacks are ambiguous, and `imagePullPolicy: Always` adds registry dependency. Pin `repo/app:1.8.3` or better `repo/app@sha256:...`, and let CI update the tag in Git so every deploy is traceable.

</details>

### Q28. What does a PodDisruptionBudget protect and how do you size it? 🟢

<details>
<summary><b>Answer</b></summary>

It limits voluntary evictions (drains, autoscaler, Karpenter). For a 3-replica service we typically use `maxUnavailable: 1`, or `minAvailable: 2`. Avoid `minAvailable` equal to replicas because it blocks drains forever; for single replicas a PDB is pointless without more replicas.
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: web }
spec:
  maxUnavailable: 1
  selector: { matchLabels: { app: web } }
```

</details>

### Q29. How do you set graceful shutdown right? 🟢

<details>
<summary><b>Answer</b></summary>

Handle SIGTERM in the app (stop accepting, finish in-flight requests), set `terminationGracePeriodSeconds` longer than the longest request, and add a `preStop` sleep of 5-15 s so endpoint removal propagates to load balancers before shutdown starts.
```yaml
lifecycle:
  preStop:
    exec: { command: ["sleep", "10"] }
terminationGracePeriodSeconds: 45
```

</details>

### Q30. Why do rollouts sometimes cause brief 502/503 errors? 🟢

<details>
<summary><b>Answer</b></summary>

New Pods can receive traffic before they are truly ready (weak readiness probe), and terminating Pods can still get requests because kube-proxy, the Ingress/ALB and the Pod shutdown race. Fix with a correct readiness probe, `preStop` delay, connection draining on the LB, and `maxUnavailable: 0`.

</details>

### Q31. How many replicas should you run and how do you spread them? 🟢

<details>
<summary><b>Answer</b></summary>

Minimum 2, usually 3 for critical services, spread across zones and nodes using topology spread constraints (`maxSkew: 1` on zone and hostname). Replica count depends on capacity needed to survive losing one zone: with 3 zones, size so two zones can carry the load (N+1 or N+zone).

</details>

### Q32. What should a readiness probe check? 🟢

<details>
<summary><b>Answer</b></summary>

That this instance can serve traffic now: its own initialization finished, caches warmed, and critical local dependencies reachable. Keep it fast and cheap. It should not call downstream services in a chain, or one slow dependency will drain every replica at once.

</details>

### Q33. What does `imagePullPolicy` default to? 🟢

<details>
<summary><b>Answer</b></summary>

`Always` if the tag is `latest` or omitted, otherwise `IfNotPresent`. With digests it makes no difference. We keep `IfNotPresent` for fixed tags to avoid registry round trips on every restart, and host a pull-through cache or private registry near the cluster.

</details>

### Q34. Compare rolling, blue-green and canary deployments on Kubernetes. 🟡

<details>
<summary><b>Answer</b></summary>

Rolling is built in and simple, but rollback is slow and both versions overlap. Blue-green runs two full stacks and flips a Service selector or route, giving instant rollback at double capacity. Canary sends a small share of traffic to the new version and watches metrics; it needs weighted routing (Gateway API, ingress annotations, mesh, Argo Rollouts, Flagger).

</details>

### Q35. How does Argo Rollouts or Flagger automate progressive delivery? 🟡

<details>
<summary><b>Answer</b></summary>

They replace Deployments with a `Rollout` (or wrap them) and shift traffic in steps (5%, 25%, 50%, 100%) via ingress, mesh or Gateway API, running **analysis** against Prometheus or Datadog metrics (error rate, p99 latency) between steps. If analysis fails, they abort and roll back automatically.
```yaml
strategy:
  canary:
    steps:
    - setWeight: 10
    - pause: { duration: 5m }
    - setWeight: 50
```

</details>

### Q36. How do you ensure database migrations don't break rolling deployments? 🟡

<details>
<summary><b>Answer</b></summary>

Use expand/contract: ship backwards-compatible schema changes first (add columns, no drops), deploy code that works with old and new schema, then remove old structures in a later release. Run migrations as a pre-deploy Job or Helm hook rather than in every Pod's startup, and make them idempotent with locking.

</details>

### Q37. How would you handle zero-downtime for long-lived connections (WebSockets, gRPC)? 🟡

<details>
<summary><b>Answer</b></summary>

Long connections do not rebalance on their own: set a generous `terminationGracePeriodSeconds`, send a close/GOAWAY on SIGTERM so clients reconnect, use connection max-age on the proxy, and use client-side load balancing or a mesh for gRPC (an L4 Service pins connections to one Pod). Rollouts use `maxSurge` for headroom.

</details>

### Q38. What are the common causes of CrashLoopBackOff in production and how do you triage quickly? 🟡

<details>
<summary><b>Answer</b></summary>

Application error or bad config (check `kubectl logs --previous`), missing Secret/ConfigMap, OOMKilled (exit 137), failing liveness probe, permissions on mounted volumes, wrong command/entrypoint (exit 126/127), or dependency unavailable at start. Triage order: events, previous logs, exit code, probes, resources.
```bash
kubectl describe pod web-7c9d | sed -n '/Last State/,/Ready/p'
kubectl logs web-7c9d --previous --tail=100
```

</details>

### Q39. How do you tune probes for a slow-starting JVM app? 🟡

<details>
<summary><b>Answer</b></summary>

Use a `startupProbe` with a generous budget (`failureThreshold: 30`, `periodSeconds: 10` gives 5 minutes) so liveness does not kill it mid-boot, then a tight liveness and readiness. Also give enough CPU request because JVM startup is CPU-bound, and consider CDS/AOT or lower JVM heap flags (`-XX:MaxRAMPercentage`).

</details>

### Q40. Why does a Pod get OOMKilled even though the app heap looks small? 🟡

<details>
<summary><b>Answer</b></summary>

The container limit counts all memory: heap plus metaspace, thread stacks, native buffers, page cache attributable to the cgroup, and `emptyDir: Memory`. Set JVM `-XX:MaxRAMPercentage=70`, leave headroom, and watch `container_memory_working_set_bytes` (what the OOM killer and kubelet use), not RSS alone.

</details>

### Q41. How do you run a stateful database on Kubernetes responsibly? 🟡

<details>
<summary><b>Answer</b></summary>

Use a mature operator (CloudNativePG, Percona, Strimzi) rather than hand-rolled StatefulSets, with PVCs on fast zonal storage, anti-affinity across zones, PDBs, tested backups to object storage with PITR, resource Guaranteed QoS and monitoring. Many teams still prefer RDS/Aurora for primary OLTP because managed services carry the backup and failover burden.

</details>

### Q42. What do you do about Pods stuck in Terminating? 🟡

<details>
<summary><b>Answer</b></summary>

Check finalizers, volume detach problems and a hung process or unreachable node. First `kubectl describe` and the node's state; fix the underlying cause. As a last resort force delete, understanding it only removes the API object, not necessarily the container.
```bash
kubectl delete pod web-0 --grace-period=0 --force
```
For StatefulSets, force deletion risks two Pods with the same identity if the node is merely partitioned.

</details>

### Q43. How do you make Jobs and CronJobs reliable? 🟡

<details>
<summary><b>Answer</b></summary>

Set `backoffLimit`, `activeDeadlineSeconds`, `ttlSecondsAfterFinished`, `concurrencyPolicy: Forbid` for non-reentrant tasks, `startingDeadlineSeconds`, and `successfulJobsHistoryLimit`/`failedJobsHistoryLimit`. Make jobs idempotent, emit metrics, and alert when a CronJob has not succeeded within its expected window (`kube_cronjob_status_last_successful_time`).

</details>

### Q44. How do init containers and readiness gates help ordering and external dependencies? 🟡

<details>
<summary><b>Answer</b></summary>

Init containers can wait for DNS/DB (`until nc -z db 5432; do sleep 2; done`), but a better pattern is apps retrying with backoff. **Readiness gates** (`readinessGates` with custom conditions) hold a Pod out of Ready until an external controller approves, as the AWS Load Balancer Controller does to wait for target registration.

</details>

### Q45. How do you achieve zero-downtime upgrades with the AWS ALB and `ip` targets? 🔴

<details>
<summary><b>Answer</b></summary>

Enable the controller's Pod readiness gate (`elbv2.k8s.aws/pod-readiness-gate-inject: enabled` on the namespace), so a Pod is Ready only when the target is healthy in the target group. Add a `preStop` sleep greater than the deregistration delay interplay (set the target group `deregistration_delay.timeout_seconds` to match), and `maxUnavailable: 0`.

</details>

### Q46. Describe how to protect a cluster from a bad deployment taking down shared infrastructure. 🔴

<details>
<summary><b>Answer</b></summary>

Per-namespace quotas and LimitRanges, PriorityClasses so critical add-ons preempt apps, PDBs, API Priority and Fairness so controllers cannot flood the API server, admission policies blocking dangerous settings, progressive delivery with automatic rollback, and rate limits on CI/GitOps sync. Also separate failure domains by node pool and keep system add-ons on dedicated nodes.

</details>

### Q47. How do you design SLO-based alerting for Kubernetes services? 🔴

<details>
<summary><b>Answer</b></summary>

Define SLIs (availability = non-5xx / total, latency p99) and SLOs (99.9%), then alert on **burn rate** over multiple windows (for example 14.4x over 1h and 5m, 6x over 6h and 30m) instead of raw CPU or restarts. Resource and Pod-level alerts become tickets or dashboards; only user-visible symptoms page.

</details>

### Q48. What failure modes does a webhook-based admission controller introduce, and how do you mitigate them? 🔴

<details>
<summary><b>Answer</b></summary>

If a `failurePolicy: Fail` webhook is down or slow, every matching create/update blocks, which can deadlock the cluster (even the webhook's own Pods). Mitigate: run several replicas across zones with PDB, exclude `kube-system` and the webhook namespace via `namespaceSelector`, set short `timeoutSeconds`, `failurePolicy: Ignore` for non-critical rules, and prefer in-process ValidatingAdmissionPolicy where possible.

</details>

### Q49. How do you test resilience (chaos engineering) on Kubernetes? 🔴

<details>
<summary><b>Answer</b></summary>

Run game days with tools such as Chaos Mesh, Litmus or AWS FIS: kill Pods, drain nodes, inject network latency, fail a zone, throttle DNS. Hypothesis first (SLO holds when a zone is lost), blast radius limited, run in staging then in production off-peak with abort conditions. Findings feed PDBs, replica counts and runbooks.

</details>

### Q50. How do you prevent a single bad Pod from being sent traffic after it becomes unhealthy at runtime? 🟡

<details>
<summary><b>Answer</b></summary>

Readiness probes keep running after startup: when they fail, the Pod is removed from endpoints until it recovers. Pair that with outlier detection at the proxy (Envoy/mesh) and client timeouts plus retries with budgets, so one slow replica does not drag p99 for everyone.

</details>

### Q51. How do you design retries, timeouts and circuit breaking to avoid retry storms in a microservice mesh? 🔴

<details>
<summary><b>Answer</b></summary>

Set per-hop timeouts shorter than the caller's timeout, cap retries (1-2) with jittered backoff and a retry budget (for example 20% of requests), retry only idempotent calls and only on connect failure/5xx, and add circuit breakers or concurrency limits. Without budgets, retries multiply load by the call depth when a dependency degrades.

</details>

## Autoscaling, Capacity & Cost

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q52. How do you right-size requests and limits? 🟢

<details>
<summary><b>Answer</b></summary>

Measure real usage for a couple of weeks with Prometheus (p95 CPU, p99 memory working set), set requests near p90-p95 usage plus headroom, set memory limit modestly above the request, and use VPA in recommendation mode or tools like Goldilocks to see suggestions. Over-requesting wastes money, under-requesting causes throttling and evictions.

</details>

### Q53. Why does HPA need accurate CPU requests? 🟢

<details>
<summary><b>Answer</b></summary>

HPA utilization is computed against **requests**: 70% of a 1-core request means 700m. An inflated request makes scaling happen too late; a tiny request makes it scale up aggressively. Fix requests first before tuning HPA thresholds.

</details>

### Q54. What are Spot instances good for on Kubernetes? 🟢

<details>
<summary><b>Answer</b></summary>

Stateless, fault-tolerant workloads and batch jobs, at up to about 60-90% discount. Handle two-minute interruption notices (Karpenter and the AWS Node Termination Handler drain nodes), diversify over many instance types and zones, keep PDBs and enough replicas, and keep critical or stateful workloads on On-Demand.

</details>

### Q55. How do you reduce Kubernetes cost quickly? 🟢

<details>
<summary><b>Answer</b></summary>

Right-size requests, delete idle workloads and orphaned PVCs/load balancers, scale non-prod to zero at night, move suitable workloads to Spot, adopt Graviton (arm64) nodes, use bin-packing or Karpenter consolidation, reduce cross-AZ traffic, and set log retention. Visibility first: OpenCost or Kubecost per namespace and label.

</details>

### Q56. What is metrics-server and what does it not do? 🟢

<details>
<summary><b>Answer</b></summary>

It collects CPU/memory usage from kubelets and serves `metrics.k8s.io` for `kubectl top` and HPA. It keeps only current values, not history, so it cannot answer "what was usage yesterday" and is not a monitoring system; use Prometheus for that.

</details>

### Q57. What is a good default for the node size and why? 🟢

<details>
<summary><b>Answer</b></summary>

Mid-size nodes (for example 4-16 vCPU) balance bin-packing and blast radius. Tiny nodes waste capacity to DaemonSets and system reservations and hit IP/Pod limits; huge nodes mean a node loss drops many Pods and increases drain time. Mix a few instance families so autoscalers find capacity.

</details>

### Q58. What labels are needed for cost allocation? 🟢

<details>
<summary><b>Answer</b></summary>

At minimum `team`, `app`, `env` and `cost-center` on workloads and namespaces, with matching AWS tags on nodes/volumes. Cost tools join Pod labels with node cost, and Pods without labels end up in an "unallocated" bucket that nobody owns.

</details>

### Q59. How do you configure Karpenter NodePools for production? 🟡

<details>
<summary><b>Answer</b></summary>

Define `NodePool` requirements (instance categories, generations, `karpenter.sh/capacity-type` of spot and on-demand, architectures), `limits` for total CPU/memory, `disruption` with `consolidationPolicy: WhenEmptyOrUnderutilized`, budgets restricting disruption windows, and `expireAfter` for node refresh. Use separate NodePools for system, general and GPU/batch, each with taints.
```yaml
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  budgets: [{ nodes: "10%" }]
```

</details>

### Q60. What are the risks of Karpenter consolidation and how do you control them? 🟡

<details>
<summary><b>Answer</b></summary>

It evicts Pods to replace or remove nodes, which hits workloads without PDBs, with long startup, or with `karpenter.sh/do-not-disrupt: "true"` annotations missing for sensitive jobs. Control with disruption budgets (including time windows), PDBs, `do-not-disrupt` on batch, and graceful shutdown handling so evictions are invisible.

</details>

### Q61. Cluster Autoscaler vs Karpenter on EKS: when do you pick each? 🟡

<details>
<summary><b>Answer</b></summary>

Cluster Autoscaler is mature, works with fixed node groups and many clouds, but is slower and bound to pre-defined instance shapes. Karpenter is faster (about 1 minute), picks right-sized instances across many types, handles Spot diversification and consolidation, but is AWS/Azure-focused and needs a controller node group. New EKS builds usually choose Karpenter or Auto Mode.

</details>

### Q62. How do you tune HPA to avoid flapping and to react to spikes? 🟡

<details>
<summary><b>Answer</b></summary>

Use `behavior`: fast scale-up policies (for example 100% or +4 Pods per 15 s), slower scale-down with `stabilizationWindowSeconds: 300+`, a sensible target (60-70% CPU), and correct `minReplicas` to survive a zone loss. For spiky traffic, scale on request rate or queue length via KEDA instead of lagging CPU.

</details>

### Q63. Why do Pods sometimes stay Pending and what does the autoscaler do about it? 🟡

<details>
<summary><b>Answer</b></summary>

`kubectl describe pod` events show `FailedScheduling: 0/12 nodes are available: 3 Insufficient cpu, 9 node(s) had untolerated taint...`. If requests simply do not fit, the autoscaler adds nodes; if the cause is an unsatisfiable selector, taint, affinity, or a PVC in another zone, adding nodes will not help.
```bash
kubectl get events --field-selector reason=FailedScheduling -A
```

</details>

### Q64. How would you size cluster headroom and avoid scale-up delays? 🟡

<details>
<summary><b>Answer</b></summary>

Keep a buffer using low-priority "pause" Pods (overprovisioning) that real Pods preempt, so new capacity is already warm and the autoscaler refills the buffer in the background. Size the buffer to your fastest expected burst and node provisioning time, and use PriorityClass `-10` for the placeholders.

</details>

### Q65. What is KEDA and when would you use it instead of HPA on CPU? 🟡

<details>
<summary><b>Answer</b></summary>

KEDA is an event-driven autoscaler that feeds external metrics (SQS length, Kafka lag, Prometheus queries, cron) to an HPA and can scale to zero. Use it for queue consumers and scheduled load where CPU does not reflect work waiting. It creates and manages the HPA for you through a `ScaledObject`.

</details>

### Q66. How do you cut cross-AZ network costs? 🟡

<details>
<summary><b>Answer</b></summary>

Prefer zone-aware routing (`trafficDistribution: PreferClose`), keep chatty services topologically close, use zonal caches, use VPC endpoints for AWS APIs, and put NAT gateways per AZ. Cross-AZ costs about $0.01/GB each direction on AWS, which becomes large for chatty microservices and replication.

</details>

### Q67. How do you reduce image pull times and registry cost? 🟡

<details>
<summary><b>Answer</b></summary>

Slim images (distroless or Alpine), multi-stage builds, layer ordering, pre-pulled/golden AMIs for large images, ECR pull-through cache, `IfNotPresent`, Spegel or Kraken peer-to-peer distribution, and image streaming (such as SOCI on EKS) for big images. Faster pulls shorten autoscaling reaction time as well as deploys.

</details>

### Q68. How would you build a showback/chargeback system for shared clusters? 🔴

<details>
<summary><b>Answer</b></summary>

Collect Pod CPU/memory requests (and usage) per namespace/label via OpenCost/Kubecost, allocate node cost proportionally by requests (with idle cost distributed), add PV, load balancer and network costs, and join with AWS CUR for accuracy. Publish per-team dashboards and monthly reports; chargeback only after teams trust the numbers.

</details>

### Q69. How do you decide between CPU limits on or off? 🔴

<details>
<summary><b>Answer</b></summary>

Without limits, Pods use idle CPU and avoid CFS throttling, with requests protecting others under contention, which suits most latency-sensitive apps. With limits you get predictable ceilings, needed for multi-tenant isolation or Guaranteed QoS. Our default: set CPU requests, omit CPU limits for trusted services, always set memory limits, and enforce limits for untrusted tenants.

</details>

### Q70. Explain bin-packing vs spreading for cost and reliability. 🔴

<details>
<summary><b>Answer</b></summary>

Bin-packing (MostAllocated scoring, Karpenter consolidation) fills nodes to cut cost but concentrates failures; spreading (LeastAllocated, topology spread) improves resilience but wastes capacity. We spread critical stateless services across zones and nodes with spread constraints, while letting consolidation pack the rest, and keep PDBs as the safety net.

</details>

### Q71. How do you plan capacity for a scheduled big-event traffic spike? 🔴

<details>
<summary><b>Answer</b></summary>

Load test to find per-Pod throughput and dependency limits (DB connections, rate limits), compute replicas for the forecast plus 30-50% headroom, pre-scale `minReplicas`, pre-warm nodes with overprovisioning or raised Karpenter limits, raise quotas (EC2 vCPU limits, ALB LCUs), warm load balancers, and set a rollback freeze. Autoscaling reacts; known events need proactive scaling.

</details>

### Q72. How do Graviton and Spot diversification change a Kubernetes fleet design? 🔴

<details>
<summary><b>Answer</b></summary>

Build multi-arch images (`docker buildx --platform linux/amd64,linux/arm64`), allow both architectures in NodePool requirements, and let Karpenter pick the cheapest. Graviton typically gives 20-40% better price/performance. Add many instance families and sizes for Spot to reduce interruption correlation, and test arm64 for native dependencies.

</details>

### Q73. What causes the cluster autoscaler to thrash and how do you stop it? 🔴

<details>
<summary><b>Answer</b></summary>

Aggressive scale-down thresholds, Pods that cannot move (PDBs, local storage, no controller) causing repeated failed attempts, mismatched node group zones with zonal volumes, and HPA flapping. Tune `scale-down-unneeded-time`, use `--balance-similar-node-groups`, one ASG per AZ for zonal storage, and stabilize HPA behavior.

</details>

### Q74. How do you scale a workload based on SQS queue depth on EKS? 🟡

<details>
<summary><b>Answer</b></summary>

Use KEDA's `aws-sqs-queue` scaler with a `ScaledObject`, authenticating through Pod Identity or IRSA. Target messages per replica from measured throughput, set `minReplicaCount` (0 if latency allows), `cooldownPeriod`, and `maxReplicaCount` bounded by downstream capacity so a backlog does not overwhelm the database.
```yaml
triggers:
- type: aws-sqs-queue
  metadata: { queueURL: https://sqs.ap-south-1.amazonaws.com/123456789012/jobs, queueLength: "20", awsRegion: ap-south-1 }
```

</details>

### Q75. How do you stop non-production clusters from burning money overnight? 🟡

<details>
<summary><b>Answer</b></summary>

Schedule scale-down: KEDA cron scaler or a CronJob scaling Deployments to zero, Karpenter consolidating empty nodes, or tools like kube-downscaler with namespace annotations. Shut non-prod node pools or whole clusters off-hours, delete preview environments after PR close with TTL controllers, and report savings.

</details>

### Q76. How would you measure and attack the idle/waste share of a cluster's bill? 🔴

<details>
<summary><b>Answer</b></summary>

Compare requested vs used resources per namespace (OpenCost allocation, `kube_pod_container_resource_requests` vs `container_cpu_usage_seconds_total`), find node utilization (allocatable vs requested) and unattached volumes/LBs. Fix in order: remove unused, right-size requests via VPA recommendations, raise bin-packing through consolidation, then Spot/Savings Plans for the stable baseline.

</details>

## Security Hardening & Compliance

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q77. What is the baseline security checklist for a production cluster? 🟢

<details>
<summary><b>Answer</b></summary>

Private API endpoint or restricted CIDRs, SSO/OIDC authentication with group-based RBAC, no cluster-admin for humans day-to-day, PSA `restricted` on app namespaces, default-deny NetworkPolicies, encrypted Secrets at rest, image scanning and signing, audit logging, no `latest` tags, workload identity instead of static keys, and patched nodes. It is layered; no single control is enough.

</details>

### Q78. Why should containers run as non-root? 🟢

<details>
<summary><b>Answer</b></summary>

If an attacker gets code execution as root in a container, a runtime or kernel bug gives them root on the node more easily, and writes to volumes happen as root. Set `runAsNonRoot: true`, a numeric `runAsUser`, and build images with a dedicated user. Restricted PSA enforces this.
```dockerfile
RUN addgroup -S app && adduser -S app -G app
USER 10001
```

</details>

### Q79. How should human access to the cluster be managed? 🟢

<details>
<summary><b>Answer</b></summary>

Authenticate via corporate SSO (OIDC or IAM Identity Center), map IdP groups to RBAC roles (view, edit per namespace), use short-lived credentials, grant elevated access just-in-time with approval and expiry, and log everything. Avoid shared kubeconfigs and long-lived client certificates.

</details>

### Q80. What is the purpose of image scanning in CI and in the cluster? 🟢

<details>
<summary><b>Answer</b></summary>

CI scanning (Trivy, Grype) blocks known critical CVEs before deploy; registry scanning (ECR enhanced scanning with Inspector) catches newly disclosed CVEs in stored images; runtime scanning flags what is running. Combine with minimal base images and regular rebuilds so fixes flow without manual work.
```bash
trivy image --severity CRITICAL,HIGH --exit-code 1 repo/app:1.8.3
```

</details>

### Q81. Why not store secrets in Git or in ConfigMaps? 🟢

<details>
<summary><b>Answer</b></summary>

Git history is forever and widely readable, and ConfigMaps have no secret handling (no encryption options, shown in plaintext by describe-level access). Use a secret manager (AWS Secrets Manager, Vault) with External Secrets Operator or the Secrets Store CSI driver, or SOPS/Sealed Secrets if you must keep encrypted values in Git.

</details>

### Q82. What is a default-deny NetworkPolicy baseline? 🟢

<details>
<summary><b>Answer</b></summary>

Each application namespace starts with deny-all ingress and egress, plus explicit allows: DNS to CoreDNS, ingress from the ingress controller namespace, and specific service-to-service flows. It limits lateral movement after a compromise and documents the real dependency graph.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

</details>

### Q83. Why should nodes be treated as immutable? 🟢

<details>
<summary><b>Answer</b></summary>

Patching or SSH-ing into nodes causes drift and unrepeatable fixes. Instead build hardened AMIs (Bottlerocket or AL2023), roll nodes regularly (Karpenter `expireAfter`, node group refresh), disable SSH in favour of SSM Session Manager, and replace rather than repair. Reduced attack surface and a regular refresh keep CVEs short-lived.

</details>

### Q84. How do you secure the supply chain of container images? 🟡

<details>
<summary><b>Answer</b></summary>

Build in trusted CI from pinned base images, generate an SBOM (Syft), scan, sign images with Cosign (keyless via OIDC) and attach provenance (SLSA attestations), store in a private registry, and verify signatures at admission with Kyverno `verifyImages` or Sigstore policy-controller. Only deploy by digest from approved registries.

</details>

### Q85. How would you enforce "only signed images from our registry" in the cluster? 🟡

<details>
<summary><b>Answer</b></summary>

A Kyverno `ClusterPolicy` with `verifyImages` checks the Cosign signature against a trusted identity (OIDC issuer and subject of your CI) and rejects others, together with a rule restricting registries to `123456789012.dkr.ecr.*.amazonaws.com/*`. Start in `Audit` mode, review violations, then switch to `Enforce`.
```yaml
validationFailureAction: Enforce
rules:
- name: verify-signature
  verifyImages:
  - imageReferences: ["123456789012.dkr.ecr.ap-south-1.amazonaws.com/*"]
```

</details>

### Q86. How do you apply least privilege to ServiceAccounts and CI/CD deploys? 🟡

<details>
<summary><b>Answer</b></summary>

One ServiceAccount per workload, `automountServiceAccountToken: false` when unused, Roles limited to specific resources and namespaces, and cloud permissions through IRSA/Pod Identity with narrow IAM policies. For CI, use OIDC federation to assume a role with an EKS access entry limited to a namespace, not a long-lived admin kubeconfig.

</details>

### Q87. What are the main attack paths in a Kubernetes cluster and how do you close them? 🟡

<details>
<summary><b>Answer</b></summary>

Exposed API server or dashboard, vulnerable apps leading to RCE, over-privileged ServiceAccounts, privileged containers/hostPath, exposed kubelet, node IAM role reachable via IMDS, secrets in env vars and images, and flat networking. Close with private endpoints, PSA, RBAC audits, IMDSv2 hop limit 1, NetworkPolicies, secret managers and runtime detection (Falco).

</details>

### Q88. Why set IMDSv2 hop limit to 1 on nodes? 🟡

<details>
<summary><b>Answer</b></summary>

Pods can reach the EC2 metadata service and steal the node's IAM role credentials, which often have broad rights. Requiring IMDSv2 and setting `HttpPutResponseHopLimit` to 1 stops containers in non-host-network Pods from fetching tokens (the extra network hop), forcing them to use IRSA/Pod Identity instead.
```bash
aws ec2 modify-instance-metadata-options --instance-id i-0abc --http-tokens required --http-put-response-hop-limit 1
```

</details>

### Q89. How do you run runtime threat detection? 🟡

<details>
<summary><b>Answer</b></summary>

Deploy Falco (or Tetragon/Sysdig/GuardDuty EKS Runtime Monitoring) as a DaemonSet using eBPF to watch syscalls, flagging shell spawn in a container, writes to `/etc`, reading service account tokens, or unexpected outbound connections. Route alerts to the SIEM/Slack and tune rules to avoid fatigue.

</details>

### Q90. How do you rotate and manage TLS certificates automatically? 🟡

<details>
<summary><b>Answer</b></summary>

cert-manager with a `ClusterIssuer` (Let's Encrypt via ACME DNS-01, AWS Private CA, or Vault) creates `Certificate` resources that populate Secrets and renew them before expiry (default at two-thirds of lifetime). Alert on `certmanager_certificate_expiration_timestamp_seconds`. For ALB use ACM certificates instead.

</details>

### Q91. How do you secure the kubelet and nodes? 🟡

<details>
<summary><b>Answer</b></summary>

Disable the anonymous kubelet auth (`--anonymous-auth=false`), use webhook authorization, keep the read-only port (10255) off, enable `protectKernelDefaults`, restrict the node IAM role, and use the NodeRestriction admission plugin so a node can only modify its own objects. CIS benchmarks (kube-bench) check these settings.
```bash
kube-bench run --targets node
```

</details>

### Q92. How do you protect etcd and secrets inside the cluster? 🟡

<details>
<summary><b>Answer</b></summary>

Encrypt Secrets at rest with KMS, allow only the API servers to reach etcd over mutual TLS, isolate etcd on separate security groups, restrict `get/list secrets` RBAC (note that `list` reveals values), enable audit logging, and avoid exposing Secrets as environment variables since they leak into logs and crash dumps.

</details>

### Q93. Design a zero-trust network model for Kubernetes. 🔴

<details>
<summary><b>Answer</b></summary>

Identity-based, deny-by-default: default-deny NetworkPolicies (or Cilium identity-based L3-L7 policies), mTLS between all workloads using a mesh or SPIFFE/SPIRE identities, authorization policies per service, egress control via egress gateway or FQDN policies, and short-lived credentials. Network location never grants trust; every call authenticates and authorizes.

</details>

### Q94. How do you handle compliance (PCI, SOC 2, HIPAA) on Kubernetes? 🔴

<details>
<summary><b>Answer</b></summary>

Map controls to evidence: audit logs retained and immutable, RBAC reviews, encryption at rest and in transit, network segmentation by NetworkPolicy and dedicated node pools or clusters for in-scope workloads, change control through Git PRs, vulnerability management SLAs, and policy-as-code reports (Kyverno policy reports, Config/Security Hub). Automate evidence collection.

</details>

### Q95. What is your response plan when a container is suspected compromised? 🔴

<details>
<summary><b>Answer</b></summary>

Isolate first: label the Pod to match a quarantine NetworkPolicy and cordon the node, but do not delete the Pod yet; capture evidence (`kubectl debug` with a forensic image, node memory/disk snapshot, logs, audit events), rotate secrets and ServiceAccount tokens the Pod could read, check lateral movement via audit logs, then rebuild from clean images and fix the root cause.

</details>

### Q96. How do you secure multi-tenant clusters against privilege escalation through Pod creation rights? 🔴

<details>
<summary><b>Answer</b></summary>

A user who can create Pods can mount any Secret or use any ServiceAccount in that namespace. Restrict via PSA `restricted`, policy disallowing hostPath, privileged, hostPID, and capabilities beyond a safe set, namespace-scoped RBAC with no cross-namespace Secret access, and scoped ServiceAccount usage via admission rules (a Pod may only use its own SA).

</details>

### Q97. What do you monitor in the Kubernetes audit log for security? 🔴

<details>
<summary><b>Answer</b></summary>

Creation of ClusterRoleBindings to cluster-admin, `exec`/`attach` into Pods, Secret reads outside expected identities, `impersonate` usage, changes to admission webhooks or PSA labels, anonymous requests that succeeded, `system:masters` users, and new privileged Pods. Send to the SIEM with detections, and keep logs off-cluster.

</details>

### Q98. How do OIDC authentication, groups claims and RBAC combine for enterprise SSO? 🔴

<details>
<summary><b>Answer</b></summary>

Configure the API server (or EKS OIDC identity provider config, or access entries with IAM Identity Center) with `--oidc-issuer-url`, `--oidc-client-id`, `--oidc-username-claim` and `--oidc-groups-claim`. RoleBindings reference IdP **groups** (`Group: platform-admins`), so access follows IdP membership and revocation is immediate on token expiry. Use `kubelogin` (`kubectl oidc-login`) on client side.

</details>

### Q99. Describe how to harden a cluster to the CIS Kubernetes benchmark in practice. 🔴

<details>
<summary><b>Answer</b></summary>

Run kube-bench for control-plane and node findings; on managed services some items belong to the provider (see the provider's CIS benchmark). Fix node config via AMI/user-data (kubelet flags, file permissions), API settings (audit, admission, RBAC), and workload settings via policy (PSA, no privileged, resource limits). Track as code and re-scan in CI and on a schedule.

</details>

### Q100. What is Pod-to-Pod encryption in transit and how do you get it without a full mesh? 🟡

<details>
<summary><b>Answer</b></summary>

Options: mTLS via a service mesh, Cilium transparent encryption (WireGuard or IPsec) between nodes, or application-level TLS with cert-manager-issued certificates. WireGuard in Cilium encrypts all node-to-node Pod traffic with no app changes, while a mesh adds identity-aware authorization on top.

</details>

### Q101. How do you handle secrets rotation without restarting every workload? 🔴

<details>
<summary><b>Answer</b></summary>

Mount secrets as files (not env vars) from the Secrets Store CSI driver with rotation enabled, or use Reloader (stakater) to trigger rolling restarts on Secret/ConfigMap change. Prefer dynamic short-lived credentials (Vault database engine, IAM auth for RDS) so rotation is continuous and a leaked secret expires quickly.

</details>

## Observability & Day-2 Operations

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q102. What are the three pillars of observability and the standard tools on Kubernetes? 🟢

<details>
<summary><b>Answer</b></summary>

Metrics (Prometheus + Grafana, kube-state-metrics, node-exporter), logs (Fluent Bit or Vector to Loki, OpenSearch or CloudWatch), and traces (OpenTelemetry Collector to Tempo, Jaeger or X-Ray). kube-prometheus-stack bundles Prometheus Operator, Alertmanager, Grafana and standard dashboards.

</details>

### Q103. What is kube-state-metrics and how does it differ from metrics-server? 🟢

<details>
<summary><b>Answer</b></summary>

kube-state-metrics turns object state into Prometheus metrics (Deployment replicas desired vs available, Pod phases, restarts, PVC status), useful for alerting on cluster state. metrics-server only provides current CPU/memory usage for `kubectl top` and HPA. Both are needed.

</details>

### Q104. Which alerts do you set up first on a cluster? 🟢

<details>
<summary><b>Answer</b></summary>

Node NotReady, Pod CrashLoopBackOff or restarts, Deployments with unavailable replicas, Pending Pods for 10+ minutes, PVC almost full, node disk/memory pressure, API server error rate/latency, etcd leader changes, certificate expiry, and CronJob failures. Pair each with a runbook link.
```promql
kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} > 0
```

</details>

### Q105. Where do container logs live and how do you ship them? 🟢

<details>
<summary><b>Answer</b></summary>

The container runtime writes stdout/stderr to files under `/var/log/pods` and `/var/log/containers` on the node; `kubectl logs` reads from there and rotation is limited by kubelet settings. A DaemonSet agent (Fluent Bit) tails those files, enriches with Pod metadata and forwards to a central store. Apps should log to stdout in JSON.

</details>

### Q106. What is the difference between `kubectl logs --previous` and `kubectl logs -f`? 🟢

<details>
<summary><b>Answer</b></summary>

`--previous` prints logs from the last terminated instance of the container (essential for CrashLoopBackOff), `-f` streams the current instance. Use `--since`, `--tail`, `-c` for containers, and `-l app=web --prefix` to read many Pods.
```bash
kubectl logs -l app=web --prefix --since=15m --max-log-requests=20
```

</details>

### Q107. What does a good runbook for a Kubernetes alert include? 🟢

<details>
<summary><b>Answer</b></summary>

Impact and severity, quick checks (exact kubectl commands and dashboards), likely causes ordered by frequency, mitigation steps (scale, rollback, cordon), escalation contacts, and how to verify recovery. Review runbooks after every incident so they improve.

</details>

### Q108. How do you collect cluster events and why? 🟢

<details>
<summary><b>Answer</b></summary>

Events vanish after about an hour, so run an event exporter (Kubernetes Event Exporter, or the CloudWatch Observability add-on) to ship them to logs/alerting. They reveal scheduling failures, image pull errors, probe failures and evictions that metrics miss.

</details>

### Q109. How do you scale Prometheus for a large Kubernetes estate? 🟡

<details>
<summary><b>Answer</b></summary>

Limit cardinality (drop high-cardinality labels, `metric_relabel_configs`), shard with the Prometheus Operator, use remote write to Mimir, Thanos, VictoriaMetrics or Amazon Managed Prometheus for long-term and global views, and shorten local retention. Use recording rules for expensive queries and watch `prometheus_tsdb_head_series`.

</details>

### Q110. What is the USE/RED approach for Kubernetes dashboards? 🟡

<details>
<summary><b>Answer</b></summary>

USE (Utilization, Saturation, Errors) for resources like nodes and disks; RED (Rate, Errors, Duration) for services. A node dashboard shows CPU saturation and memory pressure; a service dashboard shows request rate, error ratio and p50/p95/p99 latency, with Pod restarts and HPA state overlaid.

</details>

### Q111. How would you debug a Service that returns "connection refused" or times out? 🟡

<details>
<summary><b>Answer</b></summary>

Walk the path: Pod Ready? `kubectl get endpointslices` populated? Selector matches labels? `targetPort` equals the container port? App binds to `0.0.0.0` and not localhost? NetworkPolicy blocking? kube-proxy/CNI healthy on the node? Test from a debug Pod to the Pod IP, then to the ClusterIP, then DNS.
```bash
kubectl get endpointslices -l kubernetes.io/service-name=web
kubectl run t --rm -it --image=nicolaka/netshoot -- curl -sv web.shop.svc:80
```

</details>

### Q112. How do you debug DNS problems in a cluster? 🟡

<details>
<summary><b>Answer</b></summary>

Check CoreDNS Pods and logs, test resolution from a debug Pod, look for NXDOMAIN due to `ndots`, throttled upstream (conntrack races, UDP limits), and the `forward` target. Scale CoreDNS with the cluster-proportional autoscaler, add NodeLocal DNSCache, and monitor `coredns_dns_request_duration_seconds` and error counts.
```bash
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=50
kubectl run -it --rm d --image=busybox:1.36 -- nslookup kubernetes.default
```

</details>

### Q113. How do you investigate a node in NotReady? 🟡

<details>
<summary><b>Answer</b></summary>

`kubectl describe node` for conditions (MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable); check kubelet and container runtime logs (`journalctl -u kubelet`, `-u containerd`), disk space and inodes, CNI Pod status, and connectivity to the API server. On cloud, check instance status checks, then cordon and replace rather than nurse it.

</details>

### Q114. What is the best way to troubleshoot a Pod that has no shell or tools? 🟡

<details>
<summary><b>Answer</b></summary>

Use an ephemeral debug container sharing the target's process namespace: `kubectl debug -it web-0 --image=nicolaka/netshoot --target=app`. For node-level issues `kubectl debug node/ip-10-0-1-5 -it --image=ubuntu` mounts the host filesystem at `/host`. Distroless images are safe yet debuggable this way.

</details>

### Q115. How do you trace latency across microservices? 🟡

<details>
<summary><b>Answer</b></summary>

Instrument with OpenTelemetry SDKs, propagate W3C `traceparent` headers, run an OpenTelemetry Collector (DaemonSet or gateway) with tail-based sampling, and store in Tempo/Jaeger/X-Ray. Link traces to logs via trace IDs and to metrics via exemplars, so one slow request leads to its logs and spans.

</details>

### Q116. How do you manage log volume and cost? 🟡

<details>
<summary><b>Answer</b></summary>

Log at the right level, drop health-check noise and debug logs in the agent, set per-namespace retention tiers, sample verbose logs, store cheap object-storage backends (Loki + S3), and use structured logs so you query by field instead of grepping. Alert on log volume anomalies.

</details>

### Q117. What are golden signals for the control plane? 🟡

<details>
<summary><b>Answer</b></summary>

API server request latency and error rate (`apiserver_request_duration_seconds`, 5xx), in-flight requests and APF rejections, etcd fsync and DB size, controller workqueue depth and retries, scheduler pending Pods and scheduling latency, and watch cache lag. On EKS the CloudWatch control plane metrics and API server `/metrics` give similar views.

</details>

### Q118. How do you keep Helm and GitOps deployments observable and auditable? 🟡

<details>
<summary><b>Answer</b></summary>

Argo CD or Flux emit sync status, health and drift metrics and send notifications to Slack; each deploy equals a Git commit with author and review; tag Grafana dashboards with deployment annotations to correlate incidents with changes; keep `helm history` and Git as audit trail.

</details>

### Q119. How would you design a centralized multi-cluster observability stack? 🔴

<details>
<summary><b>Answer</b></summary>

Per-cluster agents (Prometheus in agent mode or OTel Collector, Fluent Bit) remote-write to a central backend (Mimir/Thanos/AMP for metrics, Loki/OpenSearch for logs, Tempo for traces) with a `cluster` external label, tenant IDs for isolation, unified Grafana, and alerting rules evaluated centrally and locally for resilience. Plan network egress costs and per-tenant limits.

</details>

### Q120. Describe a systematic approach to an incident: "latency is up and Pods are restarting." 🔴

<details>
<summary><b>Answer</b></summary>

Declare the incident and assign roles; check recent changes (deploys, config, node pool updates); see golden signals and saturation (CPU throttling, memory near limit, HPA maxed, node pressure); read restart reasons (OOMKilled, probe failure); mitigate first (rollback, scale up, shift traffic), then root cause using traces and logs, and write a blameless post-mortem with action items.

</details>

### Q121. How do you detect and alert on CPU throttling and memory pressure before users notice? 🔴

<details>
<summary><b>Answer</b></summary>

Alert on `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m]) > 0.25` for latency-sensitive services, memory working set over 85% of limit, restart count increases with OOM reason, node `MemoryPressure` conditions and eviction events. Pair with dashboards that show request vs limit vs usage.

</details>

### Q122. How would you troubleshoot a slow API server in a large cluster? 🔴

<details>
<summary><b>Answer</b></summary>

Check `apiserver_request_duration_seconds` by verb/resource, APF queue wait times, etcd latency, and large LIST calls from misbehaving controllers (clients without watches, `kubectl get -A` loops, huge CRD lists). Use audit logs to find top callers, enable pagination/watch caches, fix noisy operators, defrag/compact etcd, and raise APF limits only after fixing the cause.

</details>

### Q123. What are the dangers of high-cardinality metrics in Kubernetes and how do you avoid them? 🔴

<details>
<summary><b>Answer</b></summary>

Labels such as Pod name, user ID or URL path multiply series, blowing up memory, ingestion cost and query times, and can OOM Prometheus. Avoid unbounded labels, drop `pod`-level labels for short-lived Jobs in aggregation, apply relabeling limits (`sample_limit`, `label_limit`), and track `topk(10, count by (__name__)({__name__=~".+"}))`.

</details>

### Q124. How do you build an SLO dashboard and error budget policy into the delivery process? 🔴

<details>
<summary><b>Answer</b></summary>

Compute SLIs from request metrics (Sloth/Pyrra generate recording rules), show remaining error budget and burn rate per service, and tie policy to it: when the budget is exhausted, freeze risky releases and prioritize reliability work. Rollouts using Argo analysis can consult the same SLI queries to gate promotion.

</details>

### Q125. How do you alert on a Deployment that is "stuck" mid-rollout? 🟡

<details>
<summary><b>Answer</b></summary>

Alert when `kube_deployment_status_replicas_unavailable > 0` or `kube_deployment_status_observed_generation != kube_deployment_metadata_generation` for more than 10-15 minutes, and when the `Progressing` condition has reason `ProgressDeadlineExceeded`. Argo CD health status `Degraded` is another signal.

</details>

### Q126. How would you build a Kubernetes cost and reliability review that runs every quarter? 🔴

<details>
<summary><b>Answer</b></summary>

Gather per-namespace utilization vs requests, top restarts and OOMs, PDB and replica coverage, version and API deprecation status, and incident themes. Score services against a production-readiness checklist (probes, limits, PDB, SLOs, runbooks), publish findings to owners, and track fixes as tickets with due dates.

</details>

## EKS in Production

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q127. How do you create a production EKS cluster in a repeatable way? 🟢

<details>
<summary><b>Answer</b></summary>

Use infrastructure as code: Terraform (`terraform-aws-modules/eks`) or eksctl with a reviewed config, in a VPC with private subnets across 3 AZs, private/restricted endpoint, control plane logging, KMS secrets encryption, managed add-ons and an OIDC provider or Pod Identity. Then bootstrap platform components with GitOps. Never click-ops production.
```bash
eksctl create cluster -f cluster.yaml
```

</details>

### Q128. Which subnets and AZs should an EKS cluster use? 🟢

<details>
<summary><b>Answer</b></summary>

At least two, normally three AZs, with private subnets for nodes and Pods (tagged `kubernetes.io/role/internal-elb=1`) and public subnets for internet-facing load balancers (`kubernetes.io/role/elb=1`). Use NAT gateways per AZ or VPC endpoints (ECR, S3, STS, EC2, logs) to cut NAT cost and dependency.

</details>

### Q129. How do you give an EKS Pod access to S3 safely? 🟢

<details>
<summary><b>Answer</b></summary>

Use IRSA or EKS Pod Identity with a dedicated ServiceAccount and an IAM role scoped to the exact bucket and prefix; never node role permissions or static access keys. Pod Identity avoids creating an OIDC provider per cluster and simplifies role trust.
```bash
aws eks create-pod-identity-association --cluster-name prod --namespace shop --service-account s3-reader --role-arn arn:aws:iam::123456789012:role/s3-reader
```

</details>

### Q130. Which EKS control-plane logs do you enable and why? 🟢

<details>
<summary><b>Answer</b></summary>

`audit` and `authenticator` are the most valuable for security and forensics; `api` for request debugging; `controllerManager` and `scheduler` for troubleshooting. They cost CloudWatch ingestion, so set retention (30-90 days) and ship audit to long-term S3.

</details>

### Q131. What are the EKS service limits that most often bite? 🟢

<details>
<summary><b>Answer</b></summary>

Pods per node (ENI/IP limits), subnet IP exhaustion, EC2 vCPU quotas for Spot/On-Demand, ELB and target group quotas, Route 53/API rate limits, security groups per ENI, and ECR pull rate. Request quota increases before launches and monitor with Service Quotas alarms.

</details>

### Q132. How should the cluster be exposed publicly? 🟢

<details>
<summary><b>Answer</b></summary>

Put an ALB (via AWS Load Balancer Controller) or NLB in public subnets with AWS WAF, TLS from ACM, and Shield as needed; keep Pods in private subnets; route to Pods by `ip` target type. The API server stays private or limited by CIDR; developers reach it via VPN/SSM.

</details>

### Q133. What is the recommended default StorageClass on EKS? 🟢

<details>
<summary><b>Answer</b></summary>

`gp3` through the EBS CSI driver (decoupled IOPS/throughput, cheaper than gp2) with `volumeBindingMode: WaitForFirstConsumer` and `allowVolumeExpansion: true`, encrypted with a KMS key. Use EFS for shared RWX and FSx for Lustre/ONTAP for HPC or special needs.
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: gp3 }
provisioner: ebs.csi.aws.com
parameters: { type: gp3, encrypted: "true" }
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

</details>

### Q134. How do you choose among managed node groups, Karpenter, Fargate and Auto Mode? 🟡

<details>
<summary><b>Answer</b></summary>

Managed node groups: simple and stable, good for system nodes. Karpenter: best cost/speed for dynamic workloads. Fargate: isolation per Pod and no node ops but limited features and higher per-unit cost. Auto Mode: least operations, opinionated. A common pattern is a small managed node group for add-ons plus Karpenter for everything else.

</details>

### Q135. How do you design EKS networking with the VPC CNI to avoid IP exhaustion? 🟡

<details>
<summary><b>Answer</b></summary>

Use /19+ subnets, enable prefix delegation, tune `WARM_PREFIX_TARGET`/`WARM_IP_TARGET` and `MINIMUM_IP_TARGET` to avoid hoarding IPs, use a secondary CIDR with custom networking for Pods, or build IPv6 clusters where each Pod gets a routable address and no NAT. Security groups for Pods are available with supported instance types.

</details>

### Q136. How do you secure access to the EKS API for humans and CI? 🟡

<details>
<summary><b>Answer</b></summary>

Humans: SSO via IAM Identity Center permission sets mapped through access entries to namespace-scoped policies; no static IAM users. CI: GitHub OIDC to assume an IAM role, mapped via access entry to a limited Kubernetes group. Restrict public endpoint CIDRs or go private with VPN, and audit via CloudTrail and audit logs.
```bash
aws eks create-access-entry --cluster-name prod --principal-arn arn:aws:iam::123456789012:role/ci-deploy
aws eks associate-access-policy --cluster-name prod --principal-arn arn:aws:iam::123456789012:role/ci-deploy --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy --access-scope type=namespace,namespaces=shop
```

</details>

### Q137. What are the differences between AL2023, Bottlerocket and custom AMIs for nodes? 🟡

<details>
<summary><b>Answer</b></summary>

Amazon Linux 2023 is the current general-purpose EKS AMI (AL2 is end-of-life for EKS AMIs), uses nodeadm for bootstrap. Bottlerocket is a minimal, container-optimized, immutable OS with API-driven updates and a smaller attack surface. Custom AMIs allow baked-in agents/images, but you own patching and compatibility.

</details>

### Q138. How do you handle EKS ingress at scale: ALB per Ingress or shared? 🟡

<details>
<summary><b>Answer</b></summary>

One ALB per Ingress becomes costly and hits quotas. Use `alb.ingress.kubernetes.io/group.name` to share an ALB across Ingresses (rule limits apply), or a shared NGINX/Envoy Gateway behind one NLB for many routes, or the Gateway API with the AWS Load Balancer Controller. Weigh cost, WAF per app, and blast radius.

</details>

### Q139. How do you monitor and alert on EKS control plane health? 🟡

<details>
<summary><b>Answer</b></summary>

Use CloudWatch metrics/logs, the API server `/metrics` endpoint (`apiserver_request_total` 5xx, latency), EKS Health Insights, and AWS Health events for planned maintenance. Alert on authentication failures, API throttling (429), webhook latency, and upcoming end-of-support dates for the cluster's Kubernetes version.

</details>

### Q140. How do you roll out an EKS-managed add-on update safely? 🟡

<details>
<summary><b>Answer</b></summary>

Check `describe-addon-versions` for compatibility with the cluster version, update non-production first, pick `--resolve-conflicts` carefully (`NONE` keeps your custom config conflicts visible, `OVERWRITE` replaces it), and upgrade one add-on at a time with rollback ready. The VPC CNI and CoreDNS are the risky ones; run them after the control plane.
```bash
aws eks update-addon --cluster-name prod --addon-name coredns --addon-version v1.11.4-eksbuild.2 --resolve-conflicts PRESERVE
```

</details>

### Q141. What do you do when a node group scales but nodes never join the cluster? 🟡

<details>
<summary><b>Answer</b></summary>

Check instance system logs and `/var/log/cloud-init-output.log`, node IAM role has `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy` and `AmazonEC2ContainerRegistryReadOnly` (or ECR pull rights), an access entry of type EC2_LINUX (or aws-auth mapping) exists, security groups allow node-to-control-plane 443, subnets have route to the endpoint/VPC endpoints, and the bootstrap script has the right cluster name and CA.

</details>

### Q142. How do you use EKS with multiple accounts and shared networking? 🟡

<details>
<summary><b>Answer</b></summary>

Hub-and-spoke AWS Organization: workload accounts per env with their own EKS clusters, a shared-services account for ECR, Transit Gateway and DNS, centralized logging/security accounts. Cross-account ECR pulls via repository policies, Route 53 private zones shared via RAM, and IAM roles per workload via Pod Identity cross-account role chaining.

</details>

### Q143. Design a cost-optimized yet resilient EKS production architecture for an e-commerce site. 🔴

<details>
<summary><b>Answer</b></summary>

3 AZs, private subnets, system managed node group (On-Demand, 2-3 small nodes, tainted), Karpenter NodePools: general (mixed Spot+On-Demand, Graviton), critical (On-Demand), batch (Spot only). ALB with WAF fronting Gateway API, HPA/KEDA on RPS and queue length, topology spread + PDBs, RDS/Aurora outside the cluster, ElastiCache, VPC endpoints, Kubecost showback, Velero + GitOps for recovery, and Pod Identity for AWS access.

</details>

### Q144. How do you migrate workloads from one EKS cluster to a new one with minimal downtime? 🔴

<details>
<summary><b>Answer</b></summary>

Build the new cluster with identical GitOps config, replicate secrets and data (Velero or native data sync), run both clusters behind Route 53 weighted records or a shared ALB/NLB, shift traffic gradually (1%, 10%, 50%, 100%) while watching SLOs, keep the old cluster warm for rollback, then decommission. Stateful migration is the hard part: plan data replication cutover explicitly.

</details>

### Q145. How do you debug "Pods unable to pull from ECR" in EKS? 🔴

<details>
<summary><b>Answer</b></summary>

Look at `kubectl describe pod` for `ErrImagePull`: `no basic auth credentials` (node role lacks ECR permissions), `i/o timeout` (no route to ECR/S3 endpoints: NAT or VPC endpoints for `ecr.api`, `ecr.dkr`, and S3 gateway endpoint), `denied` (cross-account repo policy), or `manifest unknown` (wrong tag/arch). Check the node's role and run `aws ecr get-login-password` from a debug context.

</details>

### Q146. What is your approach to EKS security with GuardDuty, Inspector and Security Hub? 🔴

<details>
<summary><b>Answer</b></summary>

Enable GuardDuty EKS Audit Log Monitoring and Runtime Monitoring (agent as add-on), Inspector for ECR and node CVE scanning, Security Hub for CIS EKS benchmark findings and aggregated alerts, Config rules for drift in cluster settings (public endpoint, logging, encryption), and route findings to the SIEM/on-call with defined severities.

</details>

### Q147. How does EKS Pod Identity differ from IRSA and when would you still use IRSA? 🔴

<details>
<summary><b>Answer</b></summary>

Pod Identity uses the Pod Identity Agent (DaemonSet) to deliver credentials; associations are managed via EKS API, roles trust `pods.eks.amazonaws.com`, supports session tags, works for cross-account with role chaining and removes OIDC provider limits (100 per account). IRSA still fits Fargate (Pod Identity is not supported), non-EKS clusters and existing setups relying on OIDC claims.

</details>

### Q148. How do you restrict which Pods can reach AWS services over the network on EKS? 🟡

<details>
<summary><b>Answer</b></summary>

Use security groups for Pods (branch ENIs, `SecurityGroupPolicy`) on supported instance types so a Pod gets its own SG, VPC endpoints with endpoint policies restricting accounts/resources, and NetworkPolicy egress rules (Cilium FQDN rules or VPC CNI network policy support) so only designated workloads reach databases and the internet.

</details>

### Q149. How do you reduce NAT gateway costs for an EKS cluster? 🟡

<details>
<summary><b>Answer</b></summary>

Add gateway endpoints for S3 and DynamoDB (free), interface endpoints for ECR, STS, CloudWatch Logs, EC2, pull images through an ECR pull-through cache, keep traffic in-zone, and review top talkers with VPC Flow Logs. Large image pulls and log shipping through NAT are the usual surprises.

</details>

### Q150. How do you decide the right EC2 instance family for EKS nodes? 🟡

<details>
<summary><b>Answer</b></summary>

Check workload profile: compute-optimized (c7g/c7i) for CPU-bound, memory-optimized (r7g) for caches and JVM heavy, general (m7g) for mixed, GPU (g5/p4) for ML. Prefer Graviton when images are multi-arch, ensure ENI/IP limits fit Pod density (use prefix delegation), and let Karpenter pick among families with `instance-category` requirements.

</details>

### Q151. How do you design EKS Pod density and node sizing when each node can only run 29 Pods on small types? 🔴

<details>
<summary><b>Answer</b></summary>

Compute max Pods from ENI limits (`/etc/eks/eni-max-pods.txt` or the `max-pods-calculator.sh`), enable prefix delegation to raise it (up to 110 or 250 on larger nodes), and set kubelet `maxPods` accordingly. Choose larger instance types to reduce DaemonSet overhead per Pod, and verify subnets have enough /28 contiguous blocks, since fragmentation prevents prefix allocation.

</details>

## Upgrades, Backup & Disaster Recovery

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q152. What is your cluster upgrade cadence and why? 🟢

<details>
<summary><b>Answer</b></summary>

Upgrade at least every 3-4 months to stay inside standard support (EKS standard support is 14 months) and avoid multi-version jumps. Small, frequent upgrades are easier because there are fewer deprecated APIs per step, and the team stays practiced. Test in staging, then production during low traffic with a rollback plan.

</details>

### Q153. What do you check before upgrading a cluster? 🟢

<details>
<summary><b>Answer</b></summary>

Release notes and deprecations, compatibility of add-ons/CNI/CSI/ingress/operators with the target version, API usage via `kubent`/`pluto`, PDBs that could block drains, node and control-plane skew, free IPs in subnets, recent backups, and capacity for rolling nodes. Notify teams and freeze other changes.

</details>

### Q154. What is Velero and what does it back up? 🟢

<details>
<summary><b>Answer</b></summary>

Velero backs up Kubernetes API objects to S3-compatible storage and can snapshot PVs (CSI snapshots) or copy volume data with Kopia. You schedule backups, set TTLs and restore into the same or another cluster, which also supports cluster migration.
```bash
velero backup create full-$(date +%F) --include-namespaces shop --snapshot-move-data
velero schedule create daily --schedule="0 2 * * *" --ttl 168h
```

</details>

### Q155. Which things are NOT recoverable by just restoring etcd or manifests? 🟢

<details>
<summary><b>Answer</b></summary>

External state: databases, object storage, DNS records, cloud load balancers and IAM, secrets stored in external managers, and PV data (unless snapshotted). The recovery plan must cover them, otherwise recreating manifests yields an empty application.

</details>

### Q156. What are RPO and RTO and how do they drive Kubernetes DR design? 🟢

<details>
<summary><b>Answer</b></summary>

RPO is how much data you can lose; RTO is how long you can be down. Tight RPO needs continuous data replication (cross-region DB replicas); tight RTO needs warm standby or active-active clusters with GitOps; loose targets allow backup-and-restore. Define them per service, then choose a pattern and test it.

</details>

### Q157. What does a production-grade etcd backup strategy look like (frequency, off-cluster storage, restore drills)? 🟢

<details>
<summary><b>Answer</b></summary>

A scheduled snapshot with `etcdctl snapshot save` (CronJob on control plane or systemd timer), copied to off-node, encrypted object storage, keep multiple generations, and periodically test a restore. Include the PKI directory (`/etc/kubernetes/pki`) in backups, since a restore needs the CA.

</details>

### Q158. How do you do a safe node maintenance (patching) operation? 🟢

<details>
<summary><b>Answer</b></summary>

Cordon the node, drain with `--ignore-daemonsets --delete-emptydir-data --timeout=5m`, patch/replace, uncordon, and watch rescheduling. Do it one node (or one batch limited by PDBs) at a time; automate by replacing nodes (immutable) instead of patching in place.

</details>

### Q159. How do you upgrade a cluster with zero downtime step by step on EKS? 🟡

<details>
<summary><b>Answer</b></summary>

(1) Run upgrade insights and fix deprecated APIs. (2) Update the control plane (`aws eks update-cluster-version`), which takes 20-40 minutes and keeps the API available. (3) Update add-ons (kube-proxy, vpc-cni, coredns, CSI). (4) Roll node groups or let Karpenter drift-replace nodes. (5) Validate workloads, run smoke tests.
```bash
aws eks update-cluster-version --name prod --kubernetes-version 1.34
aws eks wait cluster-active --name prod
```

</details>

### Q160. How do you restore a single namespace or accidentally deleted resource? 🟡

<details>
<summary><b>Answer</b></summary>

With GitOps, revert the Git commit and let the controller recreate it; for PVs restore from snapshot. With Velero, `velero restore create --from-backup daily-2026-11-08 --include-namespaces shop`, mapping to a new namespace first (`--namespace-mappings`) to compare before replacing. Check that finalizers or owner references do not recreate stale state.

</details>

### Q161. What is a good DR strategy for Kubernetes: pilot light vs warm standby vs active-active? 🟡

<details>
<summary><b>Answer</b></summary>

Pilot light: second region with IaC and data replicas but minimal compute, scaled up on failover; cheapest, RTO tens of minutes. Warm standby: scaled-down but running copy, RTO minutes. Active-active: both serve traffic, RTO near zero but needs conflict-aware data and double cost. Pick by RTO/RPO and budget, and rehearse.

</details>

### Q162. How do you validate backups actually work? 🟡

<details>
<summary><b>Answer</b></summary>

Automate restore tests: nightly or weekly restore of the latest backup into an isolated cluster/namespace, run smoke tests, measure restore time against RTO and alert on failure. A backup that has never been restored is a hope, not a plan. Also check Velero `BackupStorageLocation` is `Available` and backups are `Completed` not `PartiallyFailed`.
```bash
velero backup get
velero restore create --from-backup daily-20261108 --namespace-mappings shop:shop-restore
```

</details>

### Q163. How do you handle CRD upgrades safely (cert-manager, Prometheus Operator)? 🟡

<details>
<summary><b>Answer</b></summary>

Helm will not update CRDs in `crds/`, so we apply CRDs from the new chart version separately (`kubectl apply --server-side -f crds/`) before upgrading the release, following vendor notes. Back up CRs first, avoid removing old stored versions until migrated, and check for conversion webhooks.

</details>

### Q164. How do you plan for etcd growth and defragmentation? 🟡

<details>
<summary><b>Answer</b></summary>

Watch `etcd_mvcc_db_total_size_in_bytes` against the quota (2 GiB default; raise up to 8 GiB), enable auto-compaction, and defragment members one at a time (defrag blocks the member) when fragmentation is high. If you hit "mvcc: database space exceeded", the cluster goes read-only until you compact, defrag and disarm the alarm.
```bash
etcdctl endpoint status -w table
etcdctl defrag --endpoints=https://10.0.1.10:2379
etcdctl alarm disarm
```

</details>

### Q165. What is your plan if the control plane (API server) becomes unavailable? 🟡

<details>
<summary><b>Answer</b></summary>

Running workloads keep running (the data plane does not need the API server for existing Pods), but no scheduling, scaling or deploys occur, and kubelet cannot update status. On self-managed clusters, restore API servers/etcd quorum; on EKS, open a support case and check Health. Keep break-glass access and runbooks, and avoid control-loop dependencies on the API for traffic serving.

</details>

### Q166. How do you coordinate application teams during a cluster upgrade? 🟡

<details>
<summary><b>Answer</b></summary>

Publish a schedule and deprecation scan results early, ensure every service has replicas >= 2 and a PDB, ask teams to fix deprecated APIs, run canary clusters or a non-prod rehearsal, and monitor SLO dashboards during the window with the platform team on call. Provide a communication channel and a post-upgrade report.

</details>

### Q167. How would you handle a cluster upgrade where a CNI or CSI version is incompatible with the target? 🔴

<details>
<summary><b>Answer</b></summary>

Do not upgrade the control plane until a compatible add-on release exists. Upgrade the add-on first if it supports both versions (the usual pattern); if not, use blue-green: build a new cluster at the target version with the new add-on, migrate workloads gradually. Always test the add-on upgrade path in staging, since CNI failures break all Pod networking.

</details>

### Q168. Design a backup and DR plan for a stateful workload (PostgreSQL) on EKS. 🔴

<details>
<summary><b>Answer</b></summary>

Use an operator (CloudNativePG) with 3 instances across AZs, synchronous replica for zero RPO inside the region, continuous WAL archiving plus base backups to S3 (cross-region replicated) for PITR, a cross-region replica cluster for DR, EBS snapshots as a secondary layer, tested restores monthly, runbooks for promote/failover, and alerts on replication lag and backup age.

</details>

### Q169. How does a restore of etcd on a multi-node cluster differ from single-node, and what are the pitfalls? 🔴

<details>
<summary><b>Answer</b></summary>

All members must restore from the **same** snapshot to a fresh data dir with new cluster tokens and consistent `--initial-cluster`; starting one with old data splits history. Stop all API servers first so none writes during the restore, restart etcd, then API servers, and expect to resync kubelets. Objects created after the snapshot are lost, and node leases/controller states converge afterward.

</details>

### Q170. How would you handle an emergency Kubernetes security patch (for example a container runtime or kernel CVE) across 100 nodes? 🔴

<details>
<summary><b>Answer</b></summary>

Assess exposure; if mitigations exist (policy, network block), apply them immediately; build a patched AMI, test on a canary pool, then roll the fleet using Karpenter drift or node group refresh with PDB-safe pacing, prioritizing internet-facing and multi-tenant pools. Track completion with node-version metrics and keep a documented fast-path for emergencies.

</details>

### Q171. What are the considerations for upgrading across the 1.xx boundary with removed features (e.g., removed APIs, cgroup v1 deprecation, dockershim-style changes)? 🔴

<details>
<summary><b>Answer</b></summary>

Read the deprecation guide for each version: removed APIs (like `policy/v1beta1` PSP, `flowcontrol v1beta2/v1beta3`), cgroup v1 to v2 shifts (kubelet in maintenance mode for v1), runtime versions (containerd 2.x with CRI changes), and feature gates graduating. Validate with staging clusters running real workloads, and use `kubectl convert` or tooling to migrate manifests ahead of time.

</details>

### Q172. How do you keep upgrade-safe Helm charts and manifests over time? 🟡

<details>
<summary><b>Answer</b></summary>

Pin chart and image versions, render with `helm template` in CI and validate with `kubeconform` against the target Kubernetes version's schemas, run `pluto` for deprecated APIs, use Renovate/Dependabot for controlled bumps, and keep a staging cluster one version ahead that runs the same GitOps repo.

</details>

### Q173. How do you test an upgrade without risking production? 🟡

<details>
<summary><b>Answer</b></summary>

Maintain a staging cluster with production-like add-ons and representative workloads, upgrade it first and soak for a few days, run conformance/smoke tests and synthetic traffic, and use ephemeral clusters (kind, vCluster, EKS test cluster from IaC) in CI to validate manifests and operators against the new version before touching prod.

</details>

### Q174. How do you roll back a failed control plane upgrade? 🟡

<details>
<summary><b>Answer</b></summary>

You generally cannot downgrade a Kubernetes control plane: EKS offers no rollback, and etcd data may have been migrated to new storage versions. Mitigation is prevention: staging soak, backups, and blue-green clusters for risky jumps. If something breaks, fix forward or fail over to a standby cluster, and restore workloads from GitOps.

</details>

### Q175. How would you perform a blue-green cluster upgrade and what makes it hard? 🔴

<details>
<summary><b>Answer</b></summary>

Create a new cluster at the target version from the same IaC and GitOps repo, validate with synthetic tests, then shift traffic through DNS weights or a shared load balancer. Hard parts: stateful data and PVs (replicate or snapshot-restore), external dependencies bound to cluster identity (IAM, security groups, allow-lists), webhooks and certificates, and cost during the overlap. It is the safest approach for major risk, and heavier than in-place upgrades.

</details>

## Rapid-Fire Commands

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. How do you list all Pods across namespaces with node and IP? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get pods -A -o wide
```
Add `--field-selector=status.phase!=Running` to see unhealthy ones and `--sort-by=.status.startTime` for ordering.

</details>

### Q177. How do you find why a Pod is Pending or not starting? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl describe pod <pod> -n <ns>      # read the Events section at the bottom
kubectl get events -n <ns> --sort-by=.lastTimestamp
```
Common reasons: `FailedScheduling`, `ImagePullBackOff`, unbound PVC, missing Secret/ConfigMap.

</details>

### Q178. How do you scale a Deployment or StatefulSet imperatively? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl scale deploy/web --replicas=5
kubectl scale sts/db --replicas=3
```
With an HPA present, the HPA will override manual scaling.

</details>

### Q179. How do you get a shell into a running container and run a one-off Pod? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl exec -it deploy/web -- sh
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- sh
```

</details>

### Q180. How do you see all resources and their short names? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl api-resources
kubectl api-resources --namespaced=true -o name
```
Short names: `po`, `svc`, `deploy`, `rs`, `sts`, `ds`, `cm`, `pvc`, `ns`, `sa`, `hpa`, `netpol`.

</details>

### Q181. How do you generate YAML for a resource without creating it? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl create deploy web --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml > deploy.yaml
kubectl expose deploy web --port=80 --dry-run=client -o yaml
```

</details>

### Q182. How do you copy files to and from a Pod? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl cp ./config.yaml shop/web-0:/tmp/config.yaml
kubectl cp shop/web-0:/var/log/app.log ./app.log -c app
```
It needs `tar` in the container image; for distroless use `kubectl debug` instead.

</details>

### Q183. How do you check the current context, switch namespace and list contexts? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl config current-context
kubectl config set-context --current --namespace=shop
kubectl config get-contexts
```

</details>

### Q184. How do you find the top CPU and memory consuming Pods and nodes? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl top pods -A --sort-by=cpu | head -15
kubectl top nodes
kubectl top pod web-0 --containers
```
Requires metrics-server.

</details>

### Q185. How do you patch a resource quickly and what are the patch types? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl patch deploy web -p '{"spec":{"replicas":4}}'                          # strategic merge
kubectl patch deploy web --type=json -p='[{"op":"replace","path":"/spec/replicas","value":4}]'
kubectl patch deploy web --type=merge -p '{"spec":{"paused":true}}'
```
Strategic merge is default for built-ins; CRDs only support `json` and `merge`.

</details>

### Q186. How do you extract a Secret's decoded value and list images in use? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get secret db -n shop -o jsonpath='{.data.password}' | base64 -d
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | tr ' ' '\n' | sort | uniq -c | sort -rn
```

</details>

### Q187. How do you cordon, drain and taint a node, and undo them? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl cordon n1
kubectl drain n1 --ignore-daemonsets --delete-emptydir-data --timeout=300s
kubectl taint nodes n1 maintenance=true:NoSchedule
kubectl taint nodes n1 maintenance=true:NoSchedule-     # trailing minus removes it
kubectl uncordon n1
```

</details>

### Q188. How do you roll back, pause and resume a rollout? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl rollout pause deploy/web
kubectl set image deploy/web web=nginx:1.27.2
kubectl rollout resume deploy/web
kubectl rollout undo deploy/web
kubectl rollout history deploy/web --revision=2
```

</details>

### Q189. How do you test RBAC and impersonate another user or service account? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl auth can-i delete pods -n shop --as=jane@example.com
kubectl auth can-i '*' '*' --as=system:serviceaccount:shop:ci
kubectl get pods -n shop --as=system:serviceaccount:shop:ci
kubectl auth whoami
```

</details>

### Q190. How do you find which Pods use a given ServiceAccount, PVC, or are not Ready? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.serviceAccountName=="ci") | .metadata.namespace+"/"+.metadata.name'
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="data") | .metadata.name'
kubectl get pods -A --no-headers | awk '$3!="Running" && $3!="Completed"'
```

</details>

### Q191. How do you stream logs from many Pods and from a crashed container? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl logs -l app=web --all-containers --prefix -f --max-log-requests=10
kubectl logs web-7c9d --previous
stern web -n shop --since 10m      # third-party multi-pod tailer
```

</details>

### Q192. How do you debug a node or a running Pod with ephemeral containers? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl debug -it web-0 --image=nicolaka/netshoot --target=app
kubectl debug web-0 -it --copy-to=web-debug --container=app --image=busybox:1.36
kubectl debug node/ip-10-0-1-5.ap-south-1.compute.internal -it --image=ubuntu
```

</details>

### Q193. How do you diff and server-side validate manifests before applying? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl diff -f manifests/
kubectl apply -f manifests/ --dry-run=server
kubectl apply --server-side -f manifests/
kubeconform -strict -summary manifests/     # offline schema validation
```

</details>

### Q194. How do you work with Helm in daily operations (inspect, diff, rollback)? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
helm list -A
helm get values web -n shop --all
helm get manifest web -n shop | kubectl diff -f -
helm diff upgrade web ./chart -f prod.yaml     # helm-diff plugin
helm rollback web 3 -n shop
```

</details>

### Q195. How do you list and cleanup failed or evicted Pods and completed Jobs? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get pods -A --field-selector=status.phase=Failed
kubectl delete pods -A --field-selector=status.phase=Failed
kubectl delete job -n shop --field-selector=status.successful=1
```

</details>

### Q196. Write the commands to find why a PVC is stuck Pending on EKS. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
kubectl describe pvc data-db-0 -n shop            # events: waiting for first consumer, provisioning failed
kubectl get sc                                    # does the class exist? provisioner ebs.csi.aws.com?
kubectl -n kube-system get pods -l app=ebs-csi-controller
kubectl -n kube-system logs deploy/ebs-csi-controller -c csi-provisioner --tail=50
```
Typical causes: missing IAM role for the driver, wrong StorageClass name, `WaitForFirstConsumer` with no Pod yet, or zone mismatch.

</details>

### Q197. Write the commands to check API server health, deprecated API usage and etcd from kubeadm. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get --raw='/readyz?verbose'
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
kubectl get componentstatuses            # deprecated; prefer /readyz
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key endpoint health
```

</details>

### Q198. How do you inspect effective scheduling decisions and why a Pod landed on a node? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get pod web-0 -o wide
kubectl get events --field-selector involvedObject.name=web-0
kubectl describe node <node> | sed -n '/Taints/,/Allocated resources/p'
kubectl get pod web-0 -o jsonpath='{.spec.affinity}{"\n"}{.spec.tolerations}'
```
For deep dives raise scheduler verbosity (`-v=10`) or use the scheduler simulator.

</details>

### Q199. How do you extract and verify RBAC for a ServiceAccount and find who can do what? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get rolebinding,clusterrolebinding -A -o json | jq -r '.items[] | select(.subjects[]? | .kind=="ServiceAccount" and .name=="ci") | .metadata.namespace+"/"+.metadata.name+" -> "+.roleRef.name'
kubectl auth can-i --list --as=system:serviceaccount:shop:ci -n shop
kubectl who-can delete secrets -n shop     # krew plugin
```

</details>

### Q200. How do you force-fix common stuck states: namespace stuck Terminating, PV Released, stuck finalizers? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get ns stuck -o json | jq '.spec.finalizers=[]' | kubectl replace --raw /api/v1/namespaces/stuck/finalize -f -
kubectl patch pv pv-1 --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'
kubectl patch crd foos.example.com --type=merge -p '{"metadata":{"finalizers":[]}}'
```
Only after confirming the controller is gone; these skip cleanup and can orphan cloud resources.

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
