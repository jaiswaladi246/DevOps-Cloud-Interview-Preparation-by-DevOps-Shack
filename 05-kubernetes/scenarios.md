# Kubernetes — Scenario-Based Questions

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

> Real-world Kubernetes situations: what you would do, why, and how to say it in an interview · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Kubernetes](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Deployments and rollouts](#deployments-and-rollouts) — Q1–Q25
- [Services, Ingress and DNS](#services-ingress-and-dns) — Q26–Q50
- [Configuration, Secrets and storage](#configuration-secrets-and-storage) — Q51–Q73
- [Resources, autoscaling and scheduling](#resources-autoscaling-and-scheduling) — Q74–Q96
- [Security, RBAC and policy](#security-rbac-and-policy) — Q97–Q119
- [Helm, operators and GitOps](#helm-operators-and-gitops) — Q120–Q142
- [EKS and cluster operations](#eks-and-cluster-operations) — Q143–Q168
- [Upgrades, etcd, disaster recovery and incidents](#upgrades-etcd-disaster-recovery-and-incidents) — Q169–Q200

---

## Deployments and rollouts

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. You applied a new image tag and the rollout seems stuck at "1 out of 3 new replicas have been updated". What do you check first? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The Deployment controller created one new pod and is waiting for it to become Ready before it continues. The problem is almost always that new pod.

**What I'd do →**

1. `kubectl rollout status deploy/web` to confirm it is really stuck
2. `kubectl get pods -l app=web` and find the pod with the newest age
3. `kubectl describe pod <new-pod>` and read Events at the bottom (ImagePullBackOff, Unschedulable, failed probe)
4. `kubectl logs <new-pod> --previous` if it is restarting
5. Fix the cause, or roll back with `kubectl rollout undo deploy/web`

**Why →** The rolling update honours maxUnavailable and maxSurge, so a broken new pod blocks the rollout while the old pods keep serving. Nothing is down, which buys you time to debug.

**Interview tip →** Say that the old ReplicaSet is still serving traffic, so a stuck rollout is not an outage.

</details>

### Q2. How do you roll back a bad release and verify you are back on the previous version? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Production is showing errors right after a deploy and you need the last known-good version now.

**What I'd do →**

1. `kubectl rollout history deploy/web` to see revisions
2. `kubectl rollout undo deploy/web` (or `--to-revision=3`)
3. `kubectl rollout status deploy/web`
4. `kubectl get deploy web -o jsonpath='{.spec.template.spec.containers[0].image}'` to confirm the image
5. Annotate the revision with the reason using `kubectl annotate deploy/web kubernetes.io/change-cause="rollback to v1.4.2"`

**Why →** Rollback re-applies the pod template of an older ReplicaSet, so it is fast and uses the same rolling strategy. Only the pod template is rolled back, not replicas or other fields.

**Interview tip →** Mention revisionHistoryLimit: if it is 0 there is nothing to roll back to.

</details>

### Q3. A rollout must never reduce capacity below current replicas. How do you configure the strategy? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A latency-sensitive API with 10 replicas that cannot lose capacity during deploys.

**What I'd do →**

1. Set `strategy.rollingUpdate.maxUnavailable: 0` and `maxSurge: 25%` (or a number)
2. Make sure the cluster has headroom for the surge pods or the rollout will hang Pending
3. Add a readinessProbe so a new pod only counts as available once it can serve
4. Set `minReadySeconds: 10` to avoid counting a pod that flaps right after start
5. Verify with `kubectl rollout status` and watch `kubectl get rs -w`

**Why →** With maxUnavailable 0 the controller must bring up a new Ready pod before it terminates an old one, so serving capacity never drops.

**Interview tip →** Point out the trade-off: you need spare capacity for surge, otherwise the rollout stalls.

</details>

### Q4. You need a blue/green deployment without a service mesh. How do you do it with plain Kubernetes? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The team wants instant cutover and instant rollback for a payments service.

**What I'd do →**

1. Run two Deployments, `app-blue` and `app-green`, each with a `version` label
2. Point the Service selector at `version: blue`
3. Deploy and test green through a separate Service or port-forward
4. Cut over with `kubectl patch svc app -p '{"spec":{"selector":{"app":"app","version":"green"}}}'`
5. Keep blue running until you are confident, then scale it down

**Why →** A Service selector is evaluated continuously, so changing it re-points endpoints within seconds and the change is trivially reversible.

**Interview tip →** Mention the cost: you run double capacity during the window, and DB schema changes must be backward compatible.

</details>

### Q5. How would you do a canary release with only Deployments and a Service? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want 10% of traffic on the new version without Argo Rollouts or a mesh.

**What I'd do →**

1. Create `app-stable` with 9 replicas and `app-canary` with 1 replica, both carrying `app: web`
2. The Service selects only `app: web`, so traffic splits roughly by pod count
3. Watch error rate and latency of the canary pods in Prometheus
4. Scale the canary up and the stable down in steps, or delete the canary on failure
5. Finish by updating the stable image and removing the canary

**Why →** kube-proxy balances across all ready endpoints, so replica ratio approximates traffic ratio. It is coarse, but needs no extra tooling.

**Interview tip →** Say that for exact percentages or header routing you would use Argo Rollouts, Flagger, or Gateway API weights.

</details>

### Q6. What is the difference between Recreate and RollingUpdate strategies and when would you pick Recreate? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A legacy app holds an exclusive lock on a ReadWriteOnce volume and cannot run two versions together.

**What I'd do →**

1. Use `strategy.type: Recreate` for that Deployment
2. Expect downtime: all old pods are terminated before new ones start
3. Schedule the change in a maintenance window
4. Confirm with `kubectl get pods -w` that the old pods are gone before the new ones appear

**Why →** RollingUpdate overlaps old and new pods. Apps that cannot tolerate two versions, or share a single-attach volume, need Recreate.

**Interview tip →** Mention that a StatefulSet or a proper RWX volume is often the better long-term fix.

</details>

### Q7. After a deploy, pods are Ready but users get 502s for a few seconds. What is happening and how do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Old pods are removed from endpoints and killed while the ingress or load balancer still sends traffic to them.

**What I'd do →**

1. Add a `preStop` hook with `sleep 10` (or use `lifecycle.preStop.sleep.seconds` on recent versions)
2. Make the app handle SIGTERM: stop accepting new connections, finish in-flight requests
3. Set `terminationGracePeriodSeconds` larger than preStop plus drain time
4. Check ingress controller and ALB deregistration delay are not longer than the grace period
5. Test with `hey` or `k6` during a rollout

**Why →** Endpoint removal and SIGTERM happen in parallel. Proxies and load balancers update asynchronously, so the pod must keep serving briefly after it is told to stop.

**Interview tip →** This is a classic question; the key phrase is "endpoint propagation races SIGTERM".

</details>

### Q8. You must deploy a database migration that runs once before the new app version starts. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each release needs a schema migration, and running it from every app pod would race.

**What I'd do →**

1. Run the migration as a Job (or a Helm `pre-upgrade` hook) with `backoffLimit` and `activeDeadlineSeconds`
2. Use an expand-and-contract migration so the old version still works with the new schema
3. Gate the Deployment update on Job completion in CI (`kubectl wait --for=condition=complete job/migrate --timeout=300s`)
4. Avoid init containers for migrations, since every replica would run them
5. Keep the migration idempotent and take a backup first

**Why →** A Job runs to completion exactly once, and backward-compatible migrations make rollbacks safe.

**Interview tip →** Interviewers like hearing "expand, deploy, contract" for zero-downtime schema changes.

</details>

### Q9. How do you scale a Deployment and what happens if an HPA also manages it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Someone ran `kubectl scale deploy web --replicas=2` but replicas keep returning to 6.

**What I'd do →**

1. `kubectl get hpa` to see whether an HPA targets the Deployment
2. `kubectl describe hpa web` to read min/max and current metrics
3. Change `minReplicas`/`maxReplicas` on the HPA instead of the Deployment
4. Remove `replicas:` from the manifest in Git so GitOps does not fight the HPA

**Why →** The HPA controller owns `spec.replicas` once attached and overwrites manual changes on its next sync.

**Interview tip →** Mention that leaving `replicas` in the manifest causes a reset on every apply.

</details>

### Q10. A Deployment rollout hits `ProgressDeadlineExceeded`. What does it mean and what do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The Deployment condition Progressing=False with reason ProgressDeadlineExceeded after 600s by default.

**What I'd do →**

1. `kubectl describe deploy web` and read Conditions
2. Find the blocking pods and read their events and logs
3. Remember the controller does not roll back by itself; run `kubectl rollout undo` if needed
4. Tune `progressDeadlineSeconds` if the app genuinely needs longer to start
5. In CI use `kubectl rollout status --timeout=5m` to fail the pipeline

**Why →** The deadline only marks the rollout as failed for tooling; Kubernetes keeps trying and leaves the half-rolled state in place.

**Interview tip →** Distinguish "marked failed" from "rolled back": Kubernetes does neither automatically.

</details>

### Q11. How do you pause a rollout, make multiple changes, and resume it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need to change the image, env vars and resources together without triggering three rollouts.

**What I'd do →**

1. `kubectl rollout pause deploy/web`
2. Apply `kubectl set image`, `kubectl set env`, `kubectl set resources`
3. Confirm no new ReplicaSet was created with `kubectl get rs`
4. `kubectl rollout resume deploy/web`
5. Watch `kubectl rollout status deploy/web`

**Why →** Pausing stops the controller from reconciling template changes, so all edits land in a single revision.

**Interview tip →** In GitOps you would simply commit one change, but pause is useful during manual incident work.

</details>

### Q12. What is a ReplicaSet and why do you rarely create one directly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A new teammate asks why `kubectl get rs` shows several ReplicaSets for one Deployment.

**What I'd do →**

1. `kubectl get rs -l app=web` to list them
2. Note that old ReplicaSets are scaled to 0 and kept for rollback
3. `kubectl describe rs <name>` shows the owner Deployment
4. Limit the clutter with `revisionHistoryLimit: 5`

**Why →** A Deployment owns one ReplicaSet per pod-template revision. Rolling updates are just scaling one up and another down.

**Interview tip →** Say you always create Deployments, which add rollout and rollback on top of ReplicaSets.

</details>

### Q13. You want a rolling update to wait until a new pod has been healthy for 30 seconds. What setting do you use? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pods pass readiness quickly but crash 15 seconds later because of a lazy-loaded dependency.

**What I'd do →**

1. Set `minReadySeconds: 30` on the Deployment
2. Keep a realistic readinessProbe that checks dependencies the app needs to serve
3. Consider a `startupProbe` for slow boots
4. Watch `kubectl rollout status` to see that availability is delayed

**Why →** minReadySeconds tells the controller how long a pod must be Ready, without any container crashing, before it counts as available and the rollout can advance.

**Interview tip →** Contrast with readiness: readiness says "can serve now", minReadySeconds says "has stayed healthy".

</details>

### Q14. Deploys by `kubectl apply` in CI sometimes revert a hot-fix someone made with `kubectl edit`. How do you stop this drift? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Manual edits in prod are lost at the next pipeline run, and nobody knows why.

**What I'd do →**

1. Detect drift with `kubectl diff -f manifests/` in CI
2. Adopt GitOps with Argo CD or Flux and enable self-heal/prune so Git is the source of truth
3. Restrict `edit` and `patch` for humans through RBAC, with break-glass access that is audited
4. Use server-side apply with field managers so ownership is explicit
5. Alert when the live state differs from Git

**Why →** Client-side apply only merges the fields it knows about from the last-applied annotation, so out-of-band changes either persist silently or get overwritten.

**Interview tip →** Say "Git is the source of truth, and humans get read-only in prod".

</details>

### Q15. How do you make a deployment of 200 pods safe against a bad image without waiting for full rollout? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A bad version has taken down services before because you only noticed after the rollout finished.

**What I'd do →**

1. Use a progressive delivery tool such as Argo Rollouts with canary steps (5%, 25%, 50%, 100%) and pauses
2. Define an AnalysisTemplate querying Prometheus for error rate and p99 latency
3. Auto-abort and roll back when the analysis fails
4. Keep `maxUnavailable: 0` and PodDisruptionBudgets in place
5. Run smoke tests as a pre-promotion analysis step

**Why →** Automated analysis between steps limits the blast radius to a small share of traffic and removes the human from the critical path of detection.

**Interview tip →** Mention SLO-based gates: promote only if the error budget burn is acceptable.

</details>

### Q16. How do you ensure a rolling update does not violate availability during node drains at the same time? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The platform team is upgrading nodes while an app team deploys, and the service dipped.

**What I'd do →**

1. Create a PodDisruptionBudget, for example `minAvailable: 80%` or `maxUnavailable: 1`
2. Run at least 3 replicas spread over zones with `topologySpreadConstraints`
3. Remember PDBs apply to evictions (drain), not to rolling-update deletions
4. Coordinate by using `maxUnavailable: 0` on the Deployment
5. Check with `kubectl get pdb` that `ALLOWED DISRUPTIONS` is above 0

**Why →** A PDB limits voluntary evictions from drains, while the Deployment strategy limits its own rollout. You need both to be safe in parallel.

**Interview tip →** Many candidates think PDB protects against rolling updates; it does not.

</details>

### Q17. What does `kubectl rollout restart` do and when do you use it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A ConfigMap changed and the app only reads it at startup.

**What I'd do →**

1. `kubectl rollout restart deploy/web`
2. `kubectl rollout status deploy/web`
3. Verify new pods have newer AGE with `kubectl get pods`
4. Prefer a checksum annotation in the pod template (Helm `sha256sum`) so config changes trigger rollouts automatically

**Why →** The command adds a `kubectl.kubernetes.io/restartedAt` annotation to the pod template, which creates a new ReplicaSet and a normal rolling update.

**Interview tip →** Note it is zero-downtime when probes and strategy are correct, unlike deleting pods by hand.

</details>

### Q18. You have to run a one-off task and a nightly job. Which workload objects do you use and what settings matter? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ops wants a nightly report and a one-time data fix in the cluster.

**What I'd do →**

1. Use a Job for one-off work and a CronJob for scheduled work
2. Set `backoffLimit`, `activeDeadlineSeconds` and `ttlSecondsAfterFinished`
3. On CronJob set `concurrencyPolicy: Forbid` to avoid overlapping runs and `successfulJobsHistoryLimit`
4. Use `restartPolicy: OnFailure` or `Never`, never `Always`
5. Test by `kubectl create job --from=cronjob/report manual-1`

**Why →** Jobs track completions, retries and cleanup, which Deployments do not.

**Interview tip →** Mention `timeZone` field on CronJob (stable since 1.27) to avoid UTC surprises.

</details>

### Q19. A StatefulSet update does not progress. Pod `db-2` is stuck and `db-1` and `db-0` are old. What is going on? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** StatefulSets update in reverse ordinal order and wait for each pod to be Running and Ready.

**What I'd do →**

1. `kubectl get pods -l app=db` to see which ordinal is blocking
2. `kubectl describe pod db-2` for events
3. Check `updateStrategy` (RollingUpdate vs OnDelete) and `partition`
4. Fix the pod (probe, image, PVC) because the controller will not skip it
5. If the pod is broken by the new template, revert the template and delete the stuck pod

**Why →** With the default `OrderedReady` policy, a StatefulSet halts when one pod is not Ready to protect quorum-based systems.

**Interview tip →** Mention `partition` for staged canary updates of StatefulSets.

</details>

### Q20. Your team needs one pod on every node for log collection. Which object and what do you watch for? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You must ship node logs, including from tainted nodes.

**What I'd do →**

1. Use a DaemonSet
2. Add tolerations for control-plane and other taints the agent must run on
3. Set resource requests and a `priorityClassName: system-node-critical` where justified
4. Use `updateStrategy.rollingUpdate.maxUnavailable` to control rollout speed
5. Verify `kubectl get ds` shows DESIRED equals READY

**Why →** DaemonSet pods are scheduled one per matching node, and taints silently exclude nodes unless tolerated.

**Interview tip →** Mention that agents without requests can get evicted first under pressure.

</details>

### Q21. A Deployment shows READY 0/3 but the image is fine. What are the quick checks? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The pods are Running but not Ready.

**What I'd do →**

1. `kubectl describe pod` and look for `Readiness probe failed` events
2. `kubectl logs` to see whether the app started and on what port
3. Check the probe path and port against the container `containerPort`
4. `kubectl exec` and `curl localhost:8080/healthz` from inside
5. Fix the probe or the app, then re-apply

**Why →** Running means the container process is alive; Ready means the readiness probe passes. Services only send traffic to Ready pods.

**Interview tip →** Describe the difference between liveness, readiness and startup probes clearly.

</details>

### Q22. You need to move a stateless service from one cluster to another with zero downtime. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A cluster is being retired and traffic must shift gradually.

**What I'd do →**

1. Deploy the same manifests to the new cluster through GitOps and verify with smoke tests
2. Put both clusters behind a shared entry point such as Route 53 weighted records, a global load balancer or a CDN
3. Start at 5% to the new cluster and watch error rates
4. Increase in steps, keeping DNS TTL low (30-60s)
5. Decommission the old cluster only after traffic is zero and connections drain

**Why →** Weighted DNS or global load balancing decouples cutover from the clusters themselves and makes rollback a weight change.

**Interview tip →** Mention external dependencies: secrets, databases, allowlists, and egress IPs.

</details>

### Q23. How do you prevent a Deployment from being scheduled all on one node or one zone? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** All replicas of a critical service landed on a single node and an outage took them out together.

**What I'd do →**

1. Add `topologySpreadConstraints` with `topologyKey: topology.kubernetes.io/zone` and `maxSkew: 1`
2. Add a second constraint on `kubernetes.io/hostname`
3. Choose `whenUnsatisfiable: ScheduleAnyway` for soft, `DoNotSchedule` for hard
4. Verify with `kubectl get pods -o wide`
5. Alternative is `podAntiAffinity`

**Why →** Spread constraints give even distribution across failure domains and scale better than anti-affinity for large replica counts.

**Interview tip →** Mention that spread is evaluated at scheduling time only; a descheduler can rebalance later.

</details>

### Q24. What is the difference between `kubectl apply` and `kubectl create`, and which do you use in pipelines? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A pipeline step failed with "AlreadyExists".

**What I'd do →**

1. Use `kubectl apply -f` (declarative) in pipelines
2. Use `create` only for imperative one-offs, which errors if the object exists
3. Use `kubectl diff` before apply to preview
4. Prefer server-side apply with `--server-side` for large objects such as CRDs

**Why →** apply stores the desired state and merges changes, so it is idempotent. create is a single imperative action.

**Interview tip →** Mention annotation size limits (262144 bytes) that break client-side apply on large CRDs.

</details>

### Q25. A rolling update leaves the Deployment with two ReplicaSets both having pods, and it never converges. Why? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Old pods keep getting recreated while new pods fail readiness, and an HPA is also scaling.

**What I'd do →**

1. `kubectl get rs -o wide` and compare DESIRED, CURRENT, READY per ReplicaSet
2. `kubectl describe deploy` to read events and the scaling messages
3. Check that the new pods fail readiness and that HPA is not fighting the controller during surge
4. Check ResourceQuota for pod count or CPU that blocks the new pods
5. Resolve the readiness or quota issue, or roll back

**Why →** The Deployment controller will not scale the old RS down below what keeps availability, so a new RS that never becomes Ready freezes the rollout in a mixed state.

**Interview tip →** Mention `kubectl get events --sort-by=.lastTimestamp` as your first move.

</details>

## Services, Ingress and DNS

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. A Service has no endpoints. How do you find out why? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `kubectl get endpoints web` shows `<none>` and clients get connection refused or timeouts.

**What I'd do →**

1. `kubectl get svc web -o yaml` and note `spec.selector`
2. `kubectl get pods --show-labels` and compare labels exactly
3. `kubectl get pods -l app=web` with the selector to see if any match
4. Check the pods are Ready, since only Ready pods become endpoints
5. Fix the selector or labels, then `kubectl get endpointslices -l kubernetes.io/service-name=web`

**Why →** Endpoints are built from pods that match the selector and pass readiness. A typo in one label is the most common cause.

**Interview tip →** Mention the Service `targetPort` mismatch as the second most common cause.

</details>

### Q27. When do you use ClusterIP, NodePort, LoadBalancer and ExternalName? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A developer asks how to expose a new internal API and a public website.

**What I'd do →**

1. ClusterIP for in-cluster traffic, the default
2. NodePort mostly for debugging or when a load balancer sits in front
3. LoadBalancer to get a cloud LB per Service, such as an NLB on EKS
4. Ingress or Gateway API in front of many HTTP services to share one LB
5. ExternalName to alias an external DNS name through a cluster DNS CNAME

**Why →** Each type builds on the previous: LoadBalancer includes NodePort, which includes ClusterIP.

**Interview tip →** Say that you rarely give every service its own LB in production because of cost.

</details>

### Q28. Traffic to a Service works from some pods but times out from others. How do you debug it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Only pods on certain nodes fail to reach the ClusterIP.

**What I'd do →**

1. Run `kubectl run tmp --rm -it --image=nicolaka/netshoot -- bash` on the failing node using `nodeName`
2. `curl -v <clusterIP>:<port>` and `curl <podIP>:<targetPort>` to separate Service from pod networking
3. Check kube-proxy on that node with `kubectl -n kube-system logs ds/kube-proxy`
4. On the node inspect `iptables-save | grep <svc>` or `ipvsadm -Ln`
5. Check the CNI pod health and node security groups or firewall

**Why →** If pod IP works but ClusterIP does not, the issue is kube-proxy rules. If pod IP fails too, it is the CNI or network policy.

**Interview tip →** Separate the layers: pod to pod, pod to Service, external to Service.

</details>

### Q29. You want to expose three HTTP services under one hostname with different paths. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `/api`, `/web` and `/admin` must share `app.example.com` and one load balancer.

**What I'd do →**

1. Install an ingress controller such as ingress-nginx or the AWS Load Balancer Controller
2. Create an Ingress with `rules.host` and three `paths` with `pathType: Prefix`
3. Add TLS using a cert-manager Certificate and `tls.secretName`
4. Use annotations only for controller-specific behaviour
5. Test with `curl -H 'Host: app.example.com' http://<lb>/api`

**Why →** One L7 entry point routes by host and path, saving LBs and centralising TLS.

**Interview tip →** Mention Gateway API (GA) as the successor with HTTPRoute and role separation.

</details>

### Q30. ingress-nginx returns 404 for a new host. What do you check? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The Ingress object exists and has an address, but requests return the default backend 404.

**What I'd do →**

1. `kubectl describe ingress app` and check Rules and Events
2. Verify `ingressClassName` matches the controller class (`kubectl get ingressclass`)
3. Check the backend Service name, port and that it has endpoints
4. Curl with the correct Host header, because a 404 usually means no matching host rule
5. `kubectl -n ingress-nginx logs deploy/ingress-nginx-controller` for config reload errors

**Why →** An Ingress with the wrong or missing class is simply ignored by the controller, and a Host mismatch falls through to the default backend.

**Interview tip →** Mention the removal of the old `kubernetes.io/ingress.class` annotation habit in favour of `ingressClassName`.

</details>

### Q31. ingress-nginx returns 502 or 503 intermittently. What are the likely reasons? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Errors appear during deploys and under load.

**What I'd do →**

1. 503 means no ready endpoints for the backend, 502 means the upstream closed or refused
2. `kubectl get endpoints` to see readiness gaps
3. Check pod restarts and OOMKills with `kubectl get pods`
4. Raise `proxy-read-timeout` or fix slow upstreams if you see 504
5. Add preStop sleep and graceful shutdown for deploy-time errors

**Why →** Ingress sends traffic to pod IPs directly, so any pod that is shutting down or unready can cause 5xx.

**Interview tip →** Memorise the mapping: 502 bad upstream, 503 no upstream, 504 upstream timeout.

</details>

### Q32. A pod cannot resolve `my-svc` but can reach the pod IP. What do you test first? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** DNS lookups fail inside the cluster.

**What I'd do →**

1. `kubectl exec -it app -- nslookup my-svc` and `cat /etc/resolv.conf`
2. `kubectl get pods -n kube-system -l k8s-app=kube-dns`
3. `kubectl logs -n kube-system -l k8s-app=kube-dns`
4. Try the FQDN `my-svc.default.svc.cluster.local`
5. Test from a different namespace, since short names only resolve within the same namespace

**Why →** The resolv.conf search path includes the pod namespace, so cross-namespace calls need `svc.namespace`.

**Interview tip →** Mention ndots:5 and its effect on lookup volume.

</details>

### Q33. CoreDNS is slow and the cluster makes thousands of external lookups. How do you reduce the load? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Latency spikes and CoreDNS CPU is high, and many queries are for external names with long search-path expansions.

**What I'd do →**

1. Lower `ndots` in pod `dnsConfig` to 2 for apps that call external hosts, or use trailing-dot FQDNs
2. Deploy NodeLocal DNSCache as a DaemonSet
3. Scale CoreDNS with the cluster-proportional-autoscaler
4. Enable the `cache` plugin with a sensible TTL in the Corefile
5. Watch `coredns_dns_request_duration_seconds` in Prometheus

**Why →** With ndots:5, an external name like `api.stripe.com` is tried against four search domains first, multiplying queries.

**Interview tip →** Also mention conntrack races on UDP DNS that NodeLocal DNSCache avoids.

</details>

### Q34. You need a Service reachable from inside a VPC but not the internet on EKS. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** An internal gRPC service needs a private load balancer.

**What I'd do →**

1. Set annotation `service.beta.kubernetes.io/aws-load-balancer-scheme: internal`
2. Use `aws-load-balancer-type: external` with `nlb-target-type: ip` and the AWS Load Balancer Controller
3. Ensure private subnets are tagged `kubernetes.io/role/internal-elb=1`
4. Restrict with security groups
5. Verify with `kubectl get svc` that the hostname resolves to private IPs

**Why →** The controller reads the subnet tags and annotations to place the LB in the right subnets and scheme.

**Interview tip →** Mention that instance target type goes via NodePort, IP mode goes straight to pods.

</details>

### Q35. How do you route traffic to an external database through a Kubernetes name? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Apps use `db.prod.svc` and you want to switch between an RDS endpoint and an in-cluster DB without code changes.

**What I'd do →**

1. Create a Service of `type: ExternalName` with `externalName: mydb.abc.ap-south-1.rds.amazonaws.com`
2. Apps keep using `db.prod.svc.cluster.local`
3. For IPs, use a Service without selector plus a manual EndpointSlice
4. Remember ExternalName does not work well with TLS SNI hostname checks
5. Change only the Service to cut over

**Why →** The cluster DNS returns a CNAME, which decouples app configuration from the backend location.

**Interview tip →** Mention that ExternalName does no proxying and no health checks.

</details>

### Q36. Session stickiness is required for a legacy app. What are the options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A stateful web app breaks when requests bounce between pods.

**What I'd do →**

1. Use `sessionAffinity: ClientIP` on the Service with `sessionAffinityConfig.clientIP.timeoutSeconds`
2. Prefer ingress cookie affinity such as `nginx.ingress.kubernetes.io/affinity: cookie`
3. With ALB use target group stickiness annotations
4. Plan to externalise the session to Redis, which is the real fix
5. Test with several clients behind NAT

**Why →** ClientIP affinity is weak behind NAT or proxies, because many users share one source IP.

**Interview tip →** Say you treat stickiness as a stop-gap and push for stateless sessions.

</details>

### Q37. You must restrict an app so that only the frontend can call it. Services alone do not do that. What do you add? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Any pod in the cluster can call the backend today.

**What I'd do →**

1. Confirm the CNI enforces NetworkPolicy (Calico, Cilium, or VPC CNI network policy mode)
2. Create a default-deny ingress policy in the namespace
3. Add an allow policy selecting backend pods and the frontend pod labels (plus namespaceSelector if cross-namespace)
4. Allow DNS egress to kube-dns if you also restrict egress
5. Test with `kubectl exec` from allowed and denied pods

**Why →** Services only provide discovery and load balancing. Isolation requires NetworkPolicy enforced by the CNI.

**Interview tip →** Mention that NetworkPolicy objects are silently ignored if the CNI does not implement them.

</details>

### Q38. What is a headless Service and where do you use it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A StatefulSet needs stable DNS names for each pod.

**What I'd do →**

1. Set `clusterIP: None` on the Service
2. Reference it as `serviceName` in the StatefulSet
3. Each pod gets `pod-0.svc.ns.svc.cluster.local`
4. Verify with `nslookup svc` which returns pod IPs rather than a virtual IP
5. Use for databases, Kafka, and clients that do their own load balancing

**Why →** A headless Service skips the virtual IP and returns DNS A records of the pods, giving stable identities.

**Interview tip →** Mention that gRPC clients need headless or a mesh because ClusterIP balances per connection.

</details>

### Q39. gRPC traffic goes to only one pod behind a ClusterIP Service. Why and how do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Long-lived HTTP/2 connections get pinned to one backend.

**What I'd do →**

1. Explain that kube-proxy balances per TCP connection, not per request
2. Use a headless Service with client-side load balancing in the gRPC client
3. Or put an L7 proxy such as Envoy or a mesh (Linkerd, Istio) in front
4. Or use the ALB or ingress with gRPC support
5. Set max connection age on servers to force reconnects

**Why →** HTTP/2 multiplexes many requests on one connection, so connection-level balancing yields skew.

**Interview tip →** Name the concept: L4 vs L7 load balancing.

</details>

### Q40. You need to expose a non-HTTP TCP service such as Postgres or Kafka externally. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ingress only handles HTTP, but a partner needs TCP access.

**What I'd do →**

1. Create a `type: LoadBalancer` Service (NLB on AWS) on the TCP port
2. Restrict with `loadBalancerSourceRanges` and security groups
3. Use TLS at the app layer
4. Or use ingress-nginx TCP services ConfigMap, or a Gateway API TCPRoute
5. Prefer PrivateLink or VPN for partners rather than the open internet

**Why →** L4 passes bytes through, and needs one LB or one port per service.

**Interview tip →** Mention the security angle: never expose a database publicly.

</details>

### Q41. A new Ingress with TLS serves the controller's default certificate. What is wrong? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Browsers show "Kubernetes Ingress Controller Fake Certificate".

**What I'd do →**

1. `kubectl describe ingress` to see the TLS section and secret name
2. Check the secret exists in the same namespace as the Ingress (`kubectl get secret app-tls`) and has `tls.crt` and `tls.key`
3. Check the `hosts` under `tls` match the rule host
4. If using cert-manager, `kubectl describe certificate` and `kubectl describe challenge`
5. Check controller logs for secret load errors

**Why →** The controller falls back to a self-signed default when it cannot find or parse the referenced secret or SNI host.

**Interview tip →** Mention that secrets must be in the Ingress namespace.

</details>

### Q42. How do you test a Service quickly without exposing it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A developer wants to hit an internal service from a laptop.

**What I'd do →**

1. `kubectl port-forward svc/web 8080:80`
2. `curl localhost:8080`
3. For a pod use `kubectl port-forward pod/web-abc 8080:8080`
4. Use `kubectl exec` to curl from inside when the issue is in-cluster only
5. Stop with Ctrl+C

**Why →** port-forward tunnels through the API server to a pod, so no Service or LB changes are needed.

**Interview tip →** Mention that port-forward to a Service picks one pod and does not load balance.

</details>

### Q43. NodePort traffic works on one node but not others. What could be wrong? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Service of type NodePort answers on node A's IP but times out on node B.

**What I'd do →**

1. Check `externalTrafficPolicy` (Local only answers on nodes that host a pod)
2. Check the node security group or firewall allows the 30000-32767 range
3. `kubectl get pods -o wide` to see which nodes host pods
4. Check kube-proxy health on node B
5. Use `externalTrafficPolicy: Cluster` if you want any node to answer

**Why →** With `Local`, nodes without a local endpoint drop traffic to preserve the client IP.

**Interview tip →** Mention health checks used by cloud LBs rely on this behaviour.

</details>

### Q44. A cloud LB Service is stuck in `<pending>` for EXTERNAL-IP. What do you check? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A LoadBalancer Service never receives an address on EKS.

**What I'd do →**

1. `kubectl describe svc` and read Events for controller errors
2. Check the AWS Load Balancer Controller logs: `kubectl -n kube-system logs deploy/aws-load-balancer-controller`
3. Verify IAM permissions via IRSA or Pod Identity
4. Verify subnet tags (`kubernetes.io/role/elb`) and cluster tag
5. Check service quotas for NLBs or Elastic IPs

**Why →** Something outside Kubernetes (IAM, subnets, quota) is almost always the blocker, and the event on the Service tells you which.

**Interview tip →** Mention testing on bare metal: you need MetalLB for LoadBalancer.

</details>

### Q45. How do you do path-based rewrite for an app that expects to be served from `/`? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You expose `/shop` but the app serves assets from `/`.

**What I'd do →**

1. With ingress-nginx use `nginx.ingress.kubernetes.io/rewrite-target: /$2` and path `/shop(/|$)(.*)` with `pathType: ImplementationSpecific`
2. Better, configure the app base path
3. Check redirects and absolute asset URLs
4. Test with `curl -I`
5. Consider a subdomain instead of paths

**Why →** Rewriting handles incoming paths but not links generated by the app, so base-path support in the app is cleaner.

**Interview tip →** Say you prefer subdomains to avoid path rewriting complexity.

</details>

### Q46. Explain how you would migrate from ingress-nginx to Gateway API with no downtime. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The community ingress-nginx project is being retired and the platform team wants to move on.

**What I'd do →**

1. Install a Gateway API implementation (Envoy Gateway, Cilium, or a cloud controller) alongside ingress-nginx
2. Translate Ingress resources to Gateway and HTTPRoute objects (tools like ingress2gateway help)
3. Create the new Gateway with its own LB and test with Host headers
4. Shift DNS weight gradually from the old LB to the new one
5. Remove ingress-nginx after traffic is drained and rollback window passes

**Why →** Running both in parallel with separate load balancers turns the cutover into a DNS change that can be reversed.

**Interview tip →** Show awareness that annotations do not translate one-to-one and need review.

</details>

### Q47. You want to rate limit and add basic auth at the ingress. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A staging site must be protected and API abuse limited.

**What I'd do →**

1. For ingress-nginx use annotations `nginx.ingress.kubernetes.io/limit-rps` and `auth-type: basic` with `auth-secret`
2. Create the htpasswd secret with `htpasswd -c auth user` and `kubectl create secret generic basic-auth --from-file=auth`
3. Use OAuth2 Proxy for real SSO
4. Apply WAF at the CDN or ALB layer for serious protection
5. Verify with `curl -u user:pass`

**Why →** Cheap controls at the edge protect the backend before traffic reaches the app.

**Interview tip →** Say basic auth is for staging, SSO for production.

</details>

### Q48. Why does a pod's IP change and how do apps cope? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A developer hard-coded a pod IP in config.

**What I'd do →**

1. Explain pods are ephemeral and get a new IP on recreation
2. Use a Service DNS name instead
3. For StatefulSets use the stable pod DNS name from the headless Service
4. Check with `kubectl get pods -o wide` before and after deleting a pod

**Why →** Services provide a stable virtual IP and name over a changing set of pod IPs.

**Interview tip →** Short answer: "never address pods directly".

</details>

### Q49. Two namespaces need to call each other's services but you must keep them isolated otherwise. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `team-a` needs the API in `team-b`.

**What I'd do →**

1. Use the FQDN `api.team-b.svc.cluster.local`
2. Add a NetworkPolicy in team-b that allows ingress from namespaces with label `team: a` using `namespaceSelector`
3. Ensure the default deny stays in place
4. Allow DNS egress
5. Test using a debug pod in team-a

**Why →** DNS is global across the cluster, so isolation must be enforced by policy, not naming.

**Interview tip →** Mention that namespace selectors rely on the automatic label `kubernetes.io/metadata.name`.

</details>

### Q50. Clients see the real source IP as a node IP, not the client IP. How do you preserve it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** An allowlist in the app breaks because every request appears to come from inside the cluster.

**What I'd do →**

1. For a LoadBalancer or NodePort Service set `externalTrafficPolicy: Local`
2. Or use NLB with proxy protocol and configure the app or ingress to read it
3. For ALB read `X-Forwarded-For` and trust only the LB ranges
4. Make sure the pods are spread so every LB target node has a pod
5. Verify with an echo service that prints headers

**Why →** `Cluster` policy SNATs traffic to balance across nodes, which hides the original address.

**Interview tip →** Mention the trade-off: Local can create uneven load.

</details>

## Configuration, Secrets and storage

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. How do you inject configuration into pods and what are the options? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A team asks how to pass settings to an app without baking them into the image.

**What I'd do →**

1. Create a ConfigMap with `kubectl create configmap app-config --from-file=app.properties`
2. Consume it as env vars with `envFrom` or `valueFrom.configMapKeyRef`
3. Or mount it as files under `volumes` and `volumeMounts`
4. Remember env vars need a pod restart, while mounted files update eventually
5. Keep secrets in Secret objects, not ConfigMaps

**Why →** ConfigMaps decouple config from images, so one image runs in every environment.

**Interview tip →** Mention that subPath mounts do not receive updates.

</details>

### Q52. A ConfigMap was updated but the pod still shows old values. Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The change was applied but the app behaves as before.

**What I'd do →**

1. Check how it is consumed: env vars are fixed at container start
2. For mounted files wait for the kubelet sync (up to about a minute) and check with `kubectl exec -- cat /etc/app/config`
3. Check whether `subPath` is used, which never updates
4. Check the app reloads files, since many only read at startup
5. Trigger `kubectl rollout restart` or use a config checksum annotation

**Why →** Kubernetes updates the projected files, but it cannot make a process re-read them.

**Interview tip →** Say Reloader or a checksum annotation makes it automatic.

</details>

### Q53. How do you store a database password safely in Kubernetes? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A repo contains a plain-text password in a manifest.

**What I'd do →**

1. Rotate the exposed password immediately
2. Store it with `kubectl create secret generic db --from-literal=password=...` or better through an external manager
3. Use External Secrets Operator or the Secrets Store CSI driver with AWS Secrets Manager
4. Mount as a file or env var in the pod
5. Enable encryption at rest for etcd and restrict Secret RBAC

**Why →** Secrets are base64 encoded, not encrypted, so the real protection is encryption at rest, RBAC, and keeping values out of Git.

**Interview tip →** Say "base64 is not encryption" early.

</details>

### Q54. How do you manage secrets in GitOps without committing them in plain text? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Argo CD syncs from Git but the secrets must not be in the repo.

**What I'd do →**

1. Use External Secrets Operator: a `SecretStore` points to AWS Secrets Manager and an `ExternalSecret` creates the Secret
2. Or use Sealed Secrets: encrypt with `kubeseal` and commit the SealedSecret
3. Or SOPS with KMS and the Argo CD or Flux integration
4. Restrict who can read the resulting Secret
5. Plan rotation: ESO refreshes on `refreshInterval`

**Why →** References or encrypted payloads are safe in Git, and the cluster resolves them at runtime.

**Interview tip →** Mention the trade-off: ESO needs access to the secret store, Sealed Secrets keeps the key in the cluster.

</details>

### Q55. A pod fails with `CreateContainerConfigError` and a missing key. What happened? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Event says `couldn't find key DB_HOST in ConfigMap default/app-config`.

**What I'd do →**

1. `kubectl describe pod` and read the message
2. `kubectl get cm app-config -o yaml` to see actual keys
3. Fix the key name or add the key
4. Use `optional: true` only if the value is truly optional
5. Re-create the pod or let the Deployment retry

**Why →** The kubelet cannot build the container environment when a referenced key or object is missing.

**Interview tip →** Same error appears for missing Secrets.

</details>

### Q56. What is the difference between emptyDir, hostPath, PV/PVC and when do you use each? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A developer wants scratch space and persistent storage.

**What I'd do →**

1. emptyDir for scratch data shared across containers in a pod, lost with the pod
2. hostPath mounts a node directory; avoid it except for node agents
3. PVC through a StorageClass for durable data
4. For memory-backed scratch use `emptyDir.medium: Memory` with a `sizeLimit`
5. Choose by durability and portability needs

**Why →** Storage lifetime differs: pod, node, or independent of both.

**Interview tip →** Mention hostPath as a security risk and a PSA baseline violation.

</details>

### Q57. A StatefulSet's PVC stays Pending. How do you debug? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `kubectl get pvc` shows Pending for `data-db-0`.

**What I'd do →**

1. `kubectl describe pvc data-db-0` and read Events
2. `kubectl get storageclass` to confirm a default class or the named class exists
3. Check `volumeBindingMode: WaitForFirstConsumer`, which keeps PVCs Pending until a pod is scheduled
4. Check the CSI driver pods and IAM permissions on EKS
5. Check zone and capacity constraints

**Why →** A PVC is bound only when a PV exists or can be provisioned, and the binding mode may delay that on purpose.

**Interview tip →** Mention WaitForFirstConsumer prevents zone mismatch on EBS.

</details>

### Q58. A pod on EKS cannot attach an EBS volume: `Multi-Attach error` or volume in a different zone. Explain. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A rescheduled pod is stuck ContainerCreating with a volume error.

**What I'd do →**

1. `kubectl describe pod` and read the AttachVolume events
2. EBS is zonal and ReadWriteOnce, so the pod must be in the same AZ as the volume
3. Multi-Attach means the old node still holds the volume, so wait for detach or check the old pod is gone
4. Use `topologySpreadConstraints` or nodeAffinity for the zone
5. Consider EFS for shared RWX

**Why →** EBS volumes attach to one instance in one AZ, so scheduling must respect that.

**Interview tip →** Mention `WaitForFirstConsumer` and zone-aware node groups.

</details>

### Q59. You need to resize a PVC. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A database volume is 90% full.

**What I'd do →**

1. Verify the StorageClass has `allowVolumeExpansion: true`
2. `kubectl patch pvc data -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'`
3. Watch `kubectl get pvc` and the condition `FileSystemResizePending`
4. Online expansion works for most CSI drivers without restarting the pod
5. Remember you cannot shrink a PVC

**Why →** The CSI driver expands the backing volume and the kubelet grows the filesystem.

**Interview tip →** Mention EBS limit: one modification per volume every 6 hours.

</details>

### Q60. How do you take backups and restore stateful workloads on Kubernetes? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The business needs RPO below 1 hour for a database on Kubernetes.

**What I'd do →**

1. Use database-native backups (pg_dump, pgBackRest, WAL archiving to S3) as the primary method
2. Use CSI VolumeSnapshots for crash-consistent snapshots with a `VolumeSnapshotClass`
3. Use Velero for namespace-level backups of objects and PV snapshots
4. Test restores regularly into a scratch cluster
5. Store backups in a separate account and region

**Why →** Snapshots alone can be inconsistent for databases, and untested backups are not backups.

**Interview tip →** Mention an operator such as CloudNativePG which automates this.

</details>

### Q61. A pod needs a read-only config file and a writable temp dir, with `readOnlyRootFilesystem: true`. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The security team requires an immutable container filesystem.

**What I'd do →**

1. Set `securityContext.readOnlyRootFilesystem: true`
2. Mount ConfigMap files with `readOnly: true`
3. Add `emptyDir` volumes for `/tmp` and any cache directories the app writes
4. Check logs for "Read-only file system" errors and add paths accordingly
5. Verify with `kubectl exec -- touch /x`

**Why →** Immutable roots limit what an attacker can persist, and emptyDir gives the app the few writable paths it needs.

**Interview tip →** Good place to mention `runAsNonRoot` and dropping capabilities too.

</details>

### Q62. What is the difference between a PV, a PVC, and a StorageClass? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A newcomer is confused by three storage objects.

**What I'd do →**

1. PV is the actual storage resource
2. PVC is a request for storage by a user
3. StorageClass defines how to dynamically provision PVs (provisioner, parameters, reclaim policy)
4. Check with `kubectl get pv,pvc,sc`
5. In practice you create PVCs and dynamic provisioning creates PVs

**Why →** The split separates admin concerns from developer requests.

**Interview tip →** Draw the binding relationship if there is a whiteboard.

</details>

### Q63. You deleted a StatefulSet and its PVCs remain. Is this expected, and how do you control it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Storage costs are growing from orphaned volumes.

**What I'd do →**

1. Yes, StatefulSet PVCs are retained by default
2. Use `persistentVolumeClaimRetentionPolicy` with `whenDeleted: Delete` and `whenScaled: Retain` where appropriate
3. Check PV `reclaimPolicy` (Delete or Retain) on the StorageClass
4. List orphaned volumes with `kubectl get pvc -A` and cloud tags
5. Clean up deliberately

**Why →** The default favours data safety over cost.

**Interview tip →** Say you tag volumes by cluster to find orphans.

</details>

### Q64. How do you share a volume between multiple pods across nodes? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Several web pods need the same uploads directory.

**What I'd do →**

1. Use a ReadWriteMany capable backend such as EFS, NFS or CephFS
2. On EKS install the EFS CSI driver and create a StorageClass with `provisioningMode: efs-ap`
3. Create a PVC with `accessModes: ReadWriteMany`
4. Better, move uploads to S3 object storage
5. Test concurrent writes and locking behaviour

**Why →** EBS is RWO only, so cross-node sharing needs a network filesystem or object storage.

**Interview tip →** Mention that object storage is usually the cleaner design.

</details>

### Q65. A node is lost and a StatefulSet pod with an EBS volume does not reschedule for 6 minutes or more. Why? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Data pod is stuck Terminating on a NotReady node and the replacement pod waits.

**What I'd do →**

1. The node controller marks the pod for deletion but the kubelet cannot confirm termination
2. StatefulSets guarantee at-most-one pod with that identity, so they will not start the replacement until the old one is gone
3. Check the volume attachment with `kubectl get volumeattachment`
4. If the node is confirmed dead, `kubectl delete pod --force --grace-period=0` or use the out-of-service taint `node.kubernetes.io/out-of-service=nodeshutdown:NoExecute`
5. Fix node health detection and use shorter tolerations

**Why →** At-most-one semantics protect against two writers on the same data, so Kubernetes waits for proof the old one is dead.

**Interview tip →** Explain the out-of-service taint, it shows depth.

</details>

### Q66. Your app needs files from S3 mounted like a filesystem. What are the options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A legacy tool expects local paths for large datasets.

**What I'd do →**

1. Use the Mountpoint for Amazon S3 CSI driver for read-heavy workloads
2. Or sync with an init container using `aws s3 sync`
3. Use IRSA or Pod Identity for access, not static keys
4. Know that S3 is not POSIX: no random writes
5. Prefer SDK access in new applications

**Why →** Mountpoint gives file-like reads without copying, but semantics differ from a real filesystem.

**Interview tip →** Call out the POSIX limitation.

</details>

### Q67. How can you see which pods use a PVC before deleting it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A PVC stuck in Terminating after delete.

**What I'd do →**

1. `kubectl describe pvc data` shows `Used By`
2. `kubectl get pods -A -o json | jq` for claimName matching
3. Delete or scale down the pods using it
4. The `kubernetes.io/pvc-protection` finalizer clears once no pod uses it
5. Verify with `kubectl get pvc`

**Why →** Storage protection finalizers stop in-use claims from being deleted.

**Interview tip →** Do not patch finalizers off before checking for users.

</details>

### Q68. How do you inject a Secret as a file with strict permissions? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** An app needs a TLS key file readable only by its user.

**What I'd do →**

1. Mount the Secret as a volume with `defaultMode: 0400`
2. Set `securityContext.fsGroup` and `runAsUser` so the file is readable by the process
3. Prefer files over env vars, since env vars leak in `/proc` and crash dumps
4. Verify with `kubectl exec -- ls -l /etc/tls`
5. Rotate by updating the Secret; mounted files refresh

**Why →** File mounts have permissions and are updated automatically, env vars have neither.

**Interview tip →** Mention that mode in YAML is octal in YAML 1.1 but decimal in JSON.

</details>

### Q69. You want immutable ConfigMaps and Secrets. Why and how? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Accidental config edits keep causing incidents and kubelet watch load is high.

**What I'd do →**

1. Set `immutable: true` on the ConfigMap or Secret
2. Version names like `app-config-v7` and update the Deployment reference
3. Roll out and roll back through Deployment revisions
4. Helm and Kustomize `configMapGenerator` add hashes automatically
5. Delete old versions later

**Why →** Immutability prevents accidental change and reduces API server watches.

**Interview tip →** Mention Kustomize hash suffix as the easy route.

</details>

### Q70. How do you encrypt Secrets at rest in etcd and rotate the key? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The audit requires encrypted Secret data in etcd.

**What I'd do →**

1. Create an `EncryptionConfiguration` with a `kms` provider (AWS KMS v2) first and `identity` last
2. Pass it with `--encryption-provider-config` on kube-apiserver (on EKS enable envelope encryption in the cluster)
3. Re-encrypt existing data: `kubectl get secrets -A -o json | kubectl replace -f -`
4. Rotate by adding the new key first, restart apiservers, re-encrypt, then remove the old key
5. Verify with `etcdctl get` showing the `k8s:enc:` prefix

**Why →** The order of providers decides which encrypts, and all providers listed can decrypt, which makes rotation safe.

**Interview tip →** Say that on EKS this is a cluster setting backed by a KMS key.

</details>

### Q71. How do you pass a different config per environment with one set of manifests? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dev, staging and prod differ only in replicas, hostnames and limits.

**What I'd do →**

1. Use Kustomize with `base/` and `overlays/dev|staging|prod`
2. Patch values with `patches` and `configMapGenerator`
3. Or Helm with one `values-<env>.yaml` per environment
4. Render with `kustomize build overlays/prod` or `helm template` in CI
5. Validate with `kubeconform`

**Why →** Base plus overlays keeps duplication low and differences reviewable.

**Interview tip →** Say you never copy-paste manifests per environment.

</details>

### Q72. What happens to data in an emptyDir when a container crashes versus when the pod is deleted? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A teammate expects emptyDir data to survive restarts.

**What I'd do →**

1. Container restart: data survives because the volume belongs to the pod
2. Pod deleted or rescheduled: data is gone
3. Verify with `kubectl exec` writing a file, killing the process, and reading again
4. Use a PVC for durable data

**Why →** emptyDir lifetime equals pod lifetime.

**Interview tip →** Short and precise answers score well here.

</details>

### Q73. A pod mounting an NFS or EFS volume hangs on startup in ContainerCreating. What do you check? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `MountVolume.SetUp failed ... mount.nfs: Connection timed out`.

**What I'd do →**

1. Check the security group on the EFS mount targets allows port 2049 from the nodes
2. Check mount targets exist in each AZ used by nodes
3. Check the CSI driver pods and the node has `nfs-utils` or the driver
4. Check DNS resolution of the filesystem ID from the node
5. Test `nc -zv <mount-target-ip> 2049` from the node

**Why →** A mount timeout is nearly always network reachability on port 2049.

**Interview tip →** Mention that stuck NFS mounts can also hang node shutdown.

</details>

## Resources, autoscaling and scheduling

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q74. How do you choose requests and limits for a new service? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A new API goes live next week and nobody knows its footprint.

**What I'd do →**

1. Load test in staging and record usage with `kubectl top pod` and Prometheus
2. Set requests near p90 usage and memory limit with 20-30% headroom
3. Be cautious with CPU limits, since they cause throttling
4. Use the Vertical Pod Autoscaler in recommendation mode to refine
5. Review after two weeks of production data

**Why →** Requests drive scheduling and QoS, limits cap usage. Wrong values cause either waste or evictions.

**Interview tip →** Say you start from measurements, not guesses.

</details>

### Q75. Pods are Pending with `0/6 nodes are available: 6 Insufficient cpu`. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A new Deployment cannot schedule though nodes look idle in the dashboard.

**What I'd do →**

1. `kubectl describe pod` to read the scheduling message
2. `kubectl describe nodes | grep -A8 "Allocated resources"` to see requested versus allocatable
3. Compare to actual use with `kubectl top nodes`
4. Right-size inflated requests, or add nodes (Cluster Autoscaler or Karpenter)
5. Check that quotas and LimitRanges are not injecting large defaults

**Why →** The scheduler works on requests, not live usage, so a cluster can look idle yet be full.

**Interview tip →** This distinction is asked constantly.

</details>

### Q76. Explain QoS classes and which pods get evicted first. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Under node memory pressure, important pods got killed before batch jobs.

**What I'd do →**

1. Guaranteed: requests equal limits for CPU and memory in every container
2. Burstable: at least one request or limit set
3. BestEffort: none set
4. Eviction order is BestEffort, then Burstable exceeding requests, then Guaranteed last
5. Check with `kubectl get pod -o jsonpath='{.status.qosClass}'`

**Why →** The kubelet ranks pods by QoS and usage over requests, and OOM scores follow the same ordering.

**Interview tip →** Mention `system-node-critical` priority for agents.

</details>

### Q77. CPU limits cause latency spikes though average usage is low. Why and what is the fix? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** p99 latency is bad while CPU usage graphs look fine.

**What I'd do →**

1. Check `container_cpu_cfs_throttled_periods_total` in Prometheus
2. CFS quota throttles in 100ms periods, so bursts hit the limit even at low average
3. Remove or raise CPU limits and keep accurate requests
4. Keep memory limits
5. Re-measure p99 after the change

**Why →** Throttling is per period, not per average, so spiky services are hurt.

**Interview tip →** Say many teams run CPU requests without limits.

</details>

### Q78. How do you configure an HPA on CPU? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Traffic varies during the day and you want automatic scaling.

**What I'd do →**

1. Ensure metrics-server runs: `kubectl top pods`
2. Set CPU requests on the Deployment, since utilisation is relative to requests
3. `kubectl autoscale deploy web --cpu-percent=70 --min=3 --max=20`
4. Check with `kubectl get hpa` and `kubectl describe hpa web`
5. Load test and watch scaling events

**Why →** HPA computes desired replicas from current metric divided by target.

**Interview tip →** Remember: no requests means HPA shows `<unknown>`.

</details>

### Q79. HPA shows `<unknown>/70%` for CPU. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The HPA never scales and TARGETS is unknown.

**What I'd do →**

1. `kubectl describe hpa` and read the conditions (`FailedGetResourceMetric`)
2. `kubectl top pods` to test metrics-server
3. Confirm the pods define `resources.requests.cpu`
4. Check `kubectl get apiservice v1beta1.metrics.k8s.io` is Available
5. On EKS check that metrics-server is installed and security groups allow its port

**Why →** HPA needs both a working Metrics API and requests to compute utilisation.

**Interview tip →** Name the three failure points: metrics-server, requests, APIService.

</details>

### Q80. You need to scale on queue length or requests per second, not CPU. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A worker Deployment should scale on SQS depth.

**What I'd do →**

1. Install KEDA
2. Create a `ScaledObject` with an SQS trigger and `queueLength`
3. Use IRSA for access
4. Allow scale to zero with `minReplicaCount: 0` where acceptable
5. Check `kubectl get hpa` since KEDA creates an HPA behind the scenes

**Why →** External metrics reflect real load for async workers better than CPU.

**Interview tip →** Mention Prometheus Adapter as the other route for custom metrics.

</details>

### Q81. Your HPA flaps between 5 and 20 replicas every few minutes. How do you stabilise it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Oscillating replicas cause cold starts and cost.

**What I'd do →**

1. Set `behavior.scaleDown.stabilizationWindowSeconds: 300`
2. Limit rate with `policies` (for example 2 pods per 60s on scale down)
3. Use a more stable metric, for example averaged requests per second
4. Fix slow startup with startup probes so new pods count correctly
5. Review target utilisation

**Why →** Stabilisation windows make the HPA use the highest recommendation in the window, so it scales down slowly.

**Interview tip →** Mention `autoscaling/v2` behavior field.

</details>

### Q82. Pods are Pending, the Cluster Autoscaler does not add nodes. Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The ASG has capacity but no scale-up happens.

**What I'd do →**

1. `kubectl -n kube-system logs deploy/cluster-autoscaler` and look for "didn't trigger scale-up" reasons
2. Check the pod can fit on any node group (nodeSelector, taints, zone or PVC constraints)
3. Check max size of the node group and service quotas
4. Check the ASG tags `k8s.io/cluster-autoscaler/enabled`
5. Check IAM permissions

**Why →** The autoscaler simulates scheduling against node group templates, and any mismatch stops scale-up.

**Interview tip →** Mention Karpenter's different model of picking instance types directly.

</details>

### Q83. Why choose Karpenter over Cluster Autoscaler and what must you configure? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Node costs are high and scale-up takes minutes with fixed instance types.

**What I'd do →**

1. Install Karpenter with IAM roles and an interruption queue
2. Define a `NodePool` with requirements (capacity types spot and on-demand, instance families, zones) and limits
3. Define an `EC2NodeClass` for AMI, subnets and security groups
4. Set `disruption.consolidationPolicy` and budgets
5. Observe with `kubectl get nodeclaims`

**Why →** Karpenter provisions right-sized instances directly for pending pods and consolidates underused ones.

**Interview tip →** Say you set NodePool limits to cap runaway cost.

</details>

### Q84. How do you run workloads on spot instances safely? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Compute cost must drop by 50% for stateless services.

**What I'd do →**

1. Run multiple instance types and AZs in the node group or NodePool
2. Taint spot nodes and tolerate them only in interruption-tolerant workloads
3. Handle SIGTERM with a graceful shutdown (spot gives two minutes)
4. Use PDBs and enough replicas
5. Keep a baseline on on-demand capacity

**Why →** Spot is reclaimed with short notice, so the design must tolerate loss.

**Interview tip →** Mention the AWS Node Termination Handler or Karpenter's interruption handling.

</details>

### Q85. How do you force a pod onto GPU nodes only? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** ML pods keep landing on regular nodes.

**What I'd do →**

1. Label and taint GPU nodes, for example `kubectl taint nodes gpu-1 nvidia.com/gpu=present:NoSchedule`
2. Request `resources.limits.nvidia.com/gpu: 1`
3. Add a toleration and a nodeSelector or affinity
4. Install the NVIDIA device plugin
5. Check with `kubectl describe node | grep nvidia`

**Why →** Taints repel normal pods, tolerations plus selectors attract the intended ones.

**Interview tip →** Remember a toleration alone does not attract pods.

</details>

### Q86. Explain node affinity, pod affinity and anti-affinity with a real use. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A cache should sit close to its app, while replicas must not share a node.

**What I'd do →**

1. Use `nodeAffinity` to target node labels such as instance type
2. Use `podAffinity` with `topologyKey: kubernetes.io/hostname` to co-locate app and cache
3. Use `podAntiAffinity` to spread replicas
4. Prefer `preferredDuringSchedulingIgnoredDuringExecution` for soft rules
5. Verify with `kubectl get pods -o wide`

**Why →** Required rules can block scheduling, preferred rules only bias it.

**Interview tip →** Note the cost of anti-affinity at large scale.

</details>

### Q87. A namespace hits its ResourceQuota and new pods fail. What do you see and do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Deployments cannot create pods, with `exceeded quota` errors.

**What I'd do →**

1. `kubectl describe quota -n team-a` to see used versus hard
2. `kubectl describe rs <name>` events show `forbidden: exceeded quota`
3. Right-size workloads or raise the quota through the platform process
4. Note that quotas on requests require every pod to declare requests
5. Add a LimitRange for defaults

**Why →** Quota failures appear on the ReplicaSet, not on the Deployment, which confuses people.

**Interview tip →** Mention that quota also limits object counts, such as services of type LoadBalancer.

</details>

### Q88. What does a LimitRange do and why combine it with ResourceQuota? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A namespace has pods without requests and the quota rejects them.

**What I'd do →**

1. Create a `LimitRange` with `default`, `defaultRequest`, `min` and `max`
2. Apply with `kubectl apply -f` in the namespace
3. Check an unconfigured pod gets defaults via `kubectl get pod -o yaml`
4. Keep ResourceQuota for the namespace total
5. Document the defaults for developers

**Why →** LimitRange sets per-pod defaults and bounds, quota sets the total.

**Interview tip →** Say it prevents both unbounded pods and huge single pods.

</details>

### Q89. Explain pod priority and preemption. When did you use it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** An important payment service could not schedule because batch jobs filled the cluster.

**What I'd do →**

1. Create `PriorityClass` objects, for example `critical` at 100000 and `batch` at 100
2. Reference with `priorityClassName`
3. The scheduler preempts lower-priority pods to fit higher ones
4. Set `preemptionPolicy: Never` for pods that should queue but not evict
5. Protect with PDBs, although preemption tries but may violate them

**Why →** Priority gives deterministic behaviour under scarcity.

**Interview tip →** Warn that overusing high priority makes everything critical.

</details>

### Q90. A DaemonSet pod stays Pending on one node. Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** One node has no log agent.

**What I'd do →**

1. `kubectl describe pod` and read the scheduling reason
2. Check node taints versus tolerations
3. Check node allocatable resources, since DaemonSet pods still need room
4. Check the nodeSelector against node labels
5. Free space or add the toleration

**Why →** DaemonSet pods use normal scheduling constraints through node affinity.

**Interview tip →** Mention priority classes for agents.

</details>

### Q91. Memory usage of a Java service in a pod is much higher than the heap. How do you size it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The pod is OOMKilled although `-Xmx` is 2g and the limit is 2.5g.

**What I'd do →**

1. Remember JVM memory includes metaspace, thread stacks, direct buffers and code cache
2. Set `-XX:MaxRAMPercentage=70` instead of a fixed Xmx
3. Set the limit with 30% headroom
4. Check usage with `kubectl top` and JVM NMT
5. Use `kubectl describe pod` to confirm `Reason: OOMKilled`, exit code 137

**Why →** Container limits include all process memory, not just heap.

**Interview tip →** Mention container-aware JVMs since version 10.

</details>

### Q92. How would you right-size a cluster that runs at 15% real CPU utilisation? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Finance asks why the cluster is expensive.

**What I'd do →**

1. Compare requests to usage per workload with Prometheus or Kubecost
2. Apply VPA recommendations or reduce requests
3. Enable bin-packing with Karpenter consolidation or Cluster Autoscaler scale-down
4. Move batch to spot
5. Review quotas and idle namespaces

**Why →** Waste comes mostly from over-requested resources, not from missing nodes.

**Interview tip →** Quote a saving you achieved and how you measured it.

</details>

### Q93. What is the Vertical Pod Autoscaler and when do you not use it with HPA? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A team wants automatic resource tuning.

**What I'd do →**

1. Install VPA and create a `VerticalPodAutoscaler` with `updateMode: "Off"` to get recommendations
2. Read them with `kubectl describe vpa`
3. Use `Auto` only for workloads that tolerate restarts
4. Do not let VPA and HPA both act on CPU for the same Deployment
5. Consider in-place resize in recent Kubernetes versions

**Why →** VPA changes requests, which would feed back into HPA's utilisation maths.

**Interview tip →** Start with recommendation mode.

</details>

### Q94. How do you drain a node safely for maintenance? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A node needs kernel patching.

**What I'd do →**

1. `kubectl cordon node-1`
2. `kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data`
3. Watch for PDB-blocked evictions and fix them
4. Patch and reboot
5. `kubectl uncordon node-1`

**Why →** Drain evicts pods through the Eviction API, honouring PDBs and grace periods.

**Interview tip →** Mention that bare pods are not rescheduled.

</details>

### Q95. A drain hangs forever on `Cannot evict pod as it would violate the pod's disruption budget`. What now? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A PDB with minAvailable equal to replicas blocks every eviction.

**What I'd do →**

1. `kubectl get pdb -A` and check ALLOWED DISRUPTIONS equals 0
2. Check the workload has too few replicas or unhealthy pods
3. Fix the PDB (use `maxUnavailable: 1`) or scale up first
4. Never force-delete as a habit
5. As a last resort use `--disable-eviction` knowing it bypasses PDBs

**Why →** A PDB that allows zero disruptions makes the workload undrainable.

**Interview tip →** Say PDBs need to be reviewed in platform onboarding.

</details>

### Q96. Pods land unevenly: some nodes are overloaded, others idle. What can you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The scheduler placed pods well initially but nodes drifted.

**What I'd do →**

1. Check requests vs real usage per node
2. Add `topologySpreadConstraints`
3. Run the Descheduler to evict pods violating policies
4. Ensure every pod has realistic requests
5. Consider the scheduler profile `MostAllocated` or `LeastAllocated` depending on goals

**Why →** The scheduler decides only at scheduling time and never moves running pods.

**Interview tip →** Mention descheduler as the rebalancer.

</details>

## Security, RBAC and policy

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q97. A developer needs read-only access to one namespace. How do you grant it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A new joiner should view pods and logs in `staging` only.

**What I'd do →**

1. Use the built-in `view` ClusterRole bound with a RoleBinding: `kubectl create rolebinding dev-view --clusterrole=view --user=asha@example.com -n staging`
2. Bind groups rather than users when using SSO
3. Test with `kubectl auth can-i list pods -n staging --as asha@example.com`
4. Confirm secrets are not readable: `can-i get secrets`
5. On EKS map IAM identities with access entries

**Why →** A RoleBinding scopes a ClusterRole to a single namespace.

**Interview tip →** Say least privilege and group-based bindings.

</details>

### Q98. Explain the difference between Role, ClusterRole, RoleBinding and ClusterRoleBinding. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** An audit finds a ClusterRoleBinding where a RoleBinding was intended.

**What I'd do →**

1. Role grants permissions inside a namespace
2. ClusterRole grants cluster-wide or can be reused across namespaces
3. RoleBinding grants a Role or ClusterRole within one namespace
4. ClusterRoleBinding grants cluster-wide
5. Audit with `kubectl get clusterrolebindings -o wide` and `kubectl who-can`/`rbac-tool`

**Why →** Binding scope decides how far the permissions reach, not the role kind.

**Interview tip →** Mention aggregated ClusterRoles.

</details>

### Q99. A CI pipeline needs to deploy to one namespace. What identity do you give it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The pipeline currently uses a cluster-admin kubeconfig.

**What I'd do →**

1. Create a ServiceAccount `ci-deployer` in the namespace
2. Grant a Role limited to deployments, services, configmaps with verbs get, list, create, update, patch
3. Use short-lived tokens through OIDC federation (GitHub Actions to IAM to EKS) rather than long-lived secrets
4. Verify with `kubectl auth can-i --list --as system:serviceaccount:app:ci-deployer -n app`
5. Rotate and audit

**Why →** Short-lived, scoped credentials limit damage when a pipeline is compromised.

**Interview tip →** Say never cluster-admin in CI.

</details>

### Q100. What does a ServiceAccount do and why should you disable the token automount? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pods do not call the API but carry a token.

**What I'd do →**

1. Set `automountServiceAccountToken: false` on pods or ServiceAccounts that do not need the API
2. Create dedicated ServiceAccounts per workload
3. Check with `kubectl exec -- ls /var/run/secrets/kubernetes.io/serviceaccount`
4. Use bound, projected tokens (default now)
5. Do not use the `default` ServiceAccount for anything

**Why →** A token in a compromised pod is a foothold into the API server.

**Interview tip →** Mention bound tokens expire.

</details>

### Q101. How do you let pods on EKS access S3 without static keys? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Developers put AWS keys in Secrets.

**What I'd do →**

1. Use IRSA: associate an OIDC provider with the cluster and create an IAM role trusted by the ServiceAccount
2. Annotate the ServiceAccount with `eks.amazonaws.com/role-arn`
3. Or use EKS Pod Identity with `aws eks create-pod-identity-association`
4. Scope the IAM policy to the bucket
5. Verify with `aws sts get-caller-identity` inside the pod

**Why →** The pod receives temporary credentials via web identity, no secrets to rotate.

**Interview tip →** Pod Identity is simpler across clusters.

</details>

### Q102. A pod has access to the node's IAM role through IMDS. How do you prevent credential theft? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A compromised pod can query `169.254.169.254` and use node permissions.

**What I'd do →**

1. Enforce IMDSv2 with hop limit 1 on node launch templates
2. Use IRSA or Pod Identity for workload permissions
3. Block `169.254.169.254` with a NetworkPolicy for pods that do not need it
4. Keep the node role minimal
5. Detect with GuardDuty EKS protection

**Why →** Hop limit 1 stops containers from reaching IMDS through the extra network hop.

**Interview tip →** Credential theft via IMDS is a classic cloud breach path.

</details>

### Q103. How do you enforce a default-deny network posture in a namespace? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Compliance requires that pods only talk to what they need.

**What I'd do →**

1. Apply a policy with `podSelector: {}` and `policyTypes: [Ingress, Egress]`
2. Add explicit allows for DNS (UDP/TCP 53 to kube-dns)
3. Add allows per app (frontend to backend, backend to DB)
4. Test with `kubectl exec ... -- nc -zv`
5. Use Cilium or Calico flow logs to find missing rules

**Why →** Default deny turns the model from allow-all to allow-listed.

**Interview tip →** Say DNS egress is the thing everyone forgets.

</details>

### Q104. How do you stop privileged containers and hostPath mounts cluster-wide? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A team tried to run a privileged pod in prod.

**What I'd do →**

1. Use Pod Security Admission by labelling namespaces: `kubectl label ns prod pod-security.kubernetes.io/enforce=restricted`
2. Roll out with `warn` and `audit` modes first
3. For finer rules use Kyverno or OPA Gatekeeper
4. Exempt only system namespaces
5. Test with a dry run: `kubectl label --dry-run=server --overwrite ns prod pod-security.kubernetes.io/enforce=baseline`

**Why →** PSA gives three profiles (privileged, baseline, restricted) that replace PodSecurityPolicy.

**Interview tip →** Know that PSP was removed in 1.25.

</details>

### Q105. What is the difference between PSA and Kyverno or Gatekeeper? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The security team asks whether PSA is enough.

**What I'd do →**

1. PSA is built in with fixed profiles per namespace
2. Kyverno and Gatekeeper add custom policies such as required labels, allowed registries, mandatory limits
3. Kyverno also mutates and generates resources
4. Use ValidatingAdmissionPolicy (CEL) for lightweight in-tree rules
5. Run policies in audit mode before enforce

**Why →** Different layers: PSA for baseline pod hardening, policy engines for organisation rules.

**Interview tip →** Mention ValidatingAdmissionPolicy GA.

</details>

### Q106. You must allow only images from your private registry and signed by your pipeline. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Developers can pull anything from the internet today.

**What I'd do →**

1. Use a Kyverno `verifyImages` policy with cosign keys or keyless OIDC identities
2. Or use Sigstore policy-controller
3. Restrict registries with a validating policy on `spec.containers[*].image`
4. Sign in CI with `cosign sign`
5. Reference digests rather than tags

**Why →** Admission-time verification makes the supply-chain rule enforceable.

**Interview tip →** Mention SBOMs and attestations.

</details>

### Q107. How do you check who can do what in a cluster? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Someone asks whether a ServiceAccount can delete pods.

**What I'd do →**

1. `kubectl auth can-i delete pods -n app --as system:serviceaccount:app:builder`
2. `kubectl auth can-i --list -n app`
3. `kubectl get rolebindings,clusterrolebindings -A -o wide`
4. Use `kubectl-who-can` or `rbac-lookup` plugins
5. Review regularly

**Why →** can-i evaluates the real authorizer, including aggregated roles.

**Interview tip →** Mention impersonation permissions.

</details>

### Q108. A pod can create other pods in its namespace. Why is that dangerous? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reviewing RBAC shows `create pods` for a CI ServiceAccount.

**What I'd do →**

1. Explain that creating pods allows mounting any Secret or ServiceAccount in the namespace and running arbitrary images
2. Treat `create pods`, `exec`, `get secrets` and `impersonate` as privilege escalation paths
3. Constrain with PSA restricted and admission policy
4. Split CI per namespace
5. Audit with `kubectl auth can-i --list`

**Why →** Pod creation lets the actor assume the identities and data available in the namespace.

**Interview tip →** Mention RBAC escalation prevention (bind and escalate verbs).

</details>

### Q109. How do you limit which namespaces teams can see in a multi-tenant cluster? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Teams share one EKS cluster.

**What I'd do →**

1. One namespace per team with RoleBindings to team groups
2. ResourceQuota and LimitRange per namespace
3. Default-deny NetworkPolicy
4. PSA restricted
5. Separate node pools via taints for sensitive tenants

**Why →** Namespaces are a soft boundary and need quota, network and RBAC controls.

**Interview tip →** Say hard isolation needs separate clusters.

</details>

### Q110. What would you do after finding a compromised pod in production? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A crypto miner is running in a container.

**What I'd do →**

1. Isolate: apply a deny-all NetworkPolicy or label to quarantine the pod
2. Keep evidence: `kubectl logs`, `kubectl get pod -o yaml`, node-level forensics, and snapshot the volume
3. Cordon the node and review other pods on it
4. Rotate Secrets, ServiceAccount tokens and IAM roles it could reach
5. Find the entry point (image, exposed dashboard, RBAC), fix, then rebuild from clean images

**Why →** Containing first preserves evidence and stops lateral movement.

**Interview tip →** Say "isolate, preserve, eradicate, rotate".

</details>

### Q111. How do you secure access to the API server on EKS? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The cluster endpoint is open to the world.

**What I'd do →**

1. Set the endpoint to private or restrict `publicAccessCidrs`
2. Use IAM authentication with access entries and least-privilege groups
3. Enable control plane logging (api, audit, authenticator)
4. Remove the creator admin where possible
5. Use SSO and short-lived tokens for humans

**Why →** Reducing exposure and authenticating through IAM covers most risk.

**Interview tip →** Mention `aws eks update-cluster-config`.

</details>

### Q112. How do you scan for vulnerabilities in a running cluster and in CI? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Leadership wants continuous scanning.

**What I'd do →**

1. In CI scan images with Trivy or Grype and fail on critical issues
2. In cluster run the Trivy Operator for continuous reports
3. Scan manifests with kubescape, kube-linter or Checkov
4. Check CIS benchmarks with kube-bench
5. Track fixes by SLA

**Why →** Shift left to catch issues early and scan runtime for newly published CVEs.

**Interview tip →** Mention that results need triage, not just reports.

</details>

### Q113. What is a securityContext and what do you set by default? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A security review lists the defaults to enforce.

**What I'd do →**

1. `runAsNonRoot: true` and a numeric `runAsUser`
2. `allowPrivilegeEscalation: false`
3. `capabilities.drop: ["ALL"]`
4. `readOnlyRootFilesystem: true`
5. `seccompProfile.type: RuntimeDefault`

**Why →** These settings satisfy the restricted PSA profile.

**Interview tip →** Recite the five settings fast.

</details>

### Q114. How do you handle multi-cluster RBAC and audit at scale? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Twelve clusters, each with hand-made bindings.

**What I'd do →**

1. Manage RBAC as code through GitOps and a shared base
2. Bind IdP groups instead of users
3. Ship audit logs to a central store
4. Use policy to forbid ad hoc ClusterRoleBindings
5. Review access quarterly with automated reports

**Why →** Consistency across clusters removes drift and makes audits cheap.

**Interview tip →** Say you treat RBAC as code reviewed in PRs.

</details>

### Q115. How do you rotate a leaked ServiceAccount token? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A token appeared in a public log.

**What I'd do →**

1. Delete the affected Secret (legacy) or the ServiceAccount and recreate it
2. Bound tokens expire, and you can restart pods to get new ones
3. Check audit logs for use of the token
4. Reduce its permissions
5. Move to short-lived tokens with TokenRequest

**Why →** Revocation means invalidating the identity, not just removing the file.

**Interview tip →** Mention audit log queries by `user.username`.

</details>

### Q116. How do you prevent a pod from reaching the internet but allow internal calls? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Compliance forbids egress to the public internet for a batch job.

**What I'd do →**

1. Create an egress NetworkPolicy allowing DNS and in-cluster CIDRs (`ipBlock`)
2. Or use a private subnet without NAT route for those nodes
3. Use VPC endpoints for AWS services
4. Test with `curl` to a public IP
5. Use FQDN policies with Cilium if needed

**Why →** Without egress policy every pod can reach anything its node can.

**Interview tip →** Standard NetworkPolicy cannot filter by hostname.

</details>

### Q117. You find that `kubectl exec` is used in production by many people. How do you control it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Auditors flagged uncontrolled shell access to production pods.

**What I'd do →**

1. Restrict `pods/exec` and `pods/attach` in RBAC
2. Offer break-glass roles with approval and time limit
3. Enable audit policy at RequestResponse level for exec
4. Use ephemeral debug containers (`kubectl debug`) under controlled roles
5. Alert on exec events in prod namespaces

**Why →** Exec bypasses application controls and leaves little trace without audit logs.

**Interview tip →** Say exec in prod should be exceptional and logged.

</details>

### Q118. What is OPA Gatekeeper's ConstraintTemplate and how does it differ from Kyverno policy? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The team must choose a policy engine.

**What I'd do →**

1. Gatekeeper uses Rego in a ConstraintTemplate plus Constraint instances
2. Kyverno policies are YAML with match and validate blocks, no new language
3. Kyverno supports mutate, generate, verifyImages natively
4. Gatekeeper suits teams already using OPA
5. Pilot both in audit mode

**Why →** The choice is mostly about language and capabilities.

**Interview tip →** Say you pick by team skills and use case.

</details>

### Q119. Why is running containers as root a problem? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A scan flags many containers running as UID 0.

**What I'd do →**

1. Root in the container is root on the shared kernel if an escape occurs
2. Set `runAsNonRoot` and build images with a non-root `USER`
3. Fix file ownership with `fsGroup`
4. Verify with `kubectl exec -- id`
5. Enforce with PSA

**Why →** Least privilege reduces the effect of a container escape.

**Interview tip →** Mention user namespaces as an extra layer.

</details>

## Helm, operators and GitOps

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q120. A `helm upgrade` failed halfway. How do you see state and recover? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The release shows `failed` status after a timeout.

**What I'd do →**

1. `helm status web -n app` and `helm history web -n app`
2. `kubectl get pods` to see what is broken
3. `helm rollback web 3 -n app --wait` to the last good revision
4. Fix the chart or values and retry with `helm upgrade --install --atomic --timeout 5m`
5. Use `helm get manifest` to compare

**Why →** Helm keeps revisions as Secrets, so rollback is a re-apply of an older manifest.

**Interview tip →** Mention `--atomic` auto-rolls back on failure.

</details>

### Q121. Helm says `cannot re-use a name that is still in use` or `another operation is in progress`. What now? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The pipeline is blocked on a pending release state.

**What I'd do →**

1. `helm list -a -n app` to see pending-install or pending-upgrade
2. Check no helm process is still running
3. `helm rollback` to the last deployed revision, or `helm uninstall` for a failed first install
4. As a last resort delete the stuck release Secret `sh.helm.release.v1.web.vN`
5. Add timeouts to CI jobs

**Why →** A killed Helm client leaves the release in a pending state that blocks new operations.

**Interview tip →** Say you avoid deleting release Secrets unless necessary.

</details>

### Q122. How do you review what a Helm upgrade will change before running it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Production upgrades need a change review.

**What I'd do →**

1. `helm diff upgrade web ./chart -f values-prod.yaml` using the helm-diff plugin
2. `helm template` and `kubeconform` in CI
3. `helm upgrade --dry-run --debug`
4. Post the diff into the pull request
5. Gate on approval

**Why →** A diff shows the real manifest changes rather than only values changes.

**Interview tip →** Mention that values changes can hide big rendered diffs.

</details>

### Q123. Where do you keep chart values for multiple environments? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Teams copy whole charts per environment.

**What I'd do →**

1. Keep one chart and `values.yaml` defaults
2. Add `values-dev.yaml`, `values-prod.yaml`
3. Pass with multiple `-f` flags, later ones override
4. Keep secrets out of values files
5. Pin chart versions with `--version`

**Why →** One chart plus small overrides keeps environments consistent.

**Interview tip →** Say you pin chart versions in production.

</details>

### Q124. What does `helm template` do and how is it different from `helm install`? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Someone wants to see manifests without touching the cluster.

**What I'd do →**

1. `helm template web ./chart -f values.yaml` renders manifests locally
2. Pipe to `kubectl apply --dry-run=server -f -` for server-side validation
3. `helm install` renders and applies, and tracks a release
4. Use template in GitOps tools such as Argo CD which render themselves

**Why →** template is rendering only, no release is created.

**Interview tip →** Mention `--validate` against a live cluster.

</details>

### Q125. A Helm chart upgrade changed an immutable field and failed. How do you proceed? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Error: `field is immutable` for a Deployment selector or a Job template.

**What I'd do →**

1. Identify the field in the error (`spec.selector`, `spec.template` of Job, PVC size, Service clusterIP)
2. For Jobs delete the old Job first or use `ttlSecondsAfterFinished`
3. For selectors, plan a migration: create a new Deployment name and shift traffic
4. Avoid changing selector labels in chart updates
5. Use hooks to delete and recreate where acceptable

**Why →** Kubernetes forbids changing certain fields, so the object must be recreated.

**Interview tip →** Mention that selector labels should never include chart version.

</details>

### Q126. How would you structure Helm charts for 30 microservices? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each team copies a similar chart.

**What I'd do →**

1. Build a common library chart (`type: library`) with shared templates
2. Each service chart contains a small `values.yaml` and depends on the library
3. Publish charts to an OCI registry (ECR)
4. Version and test with `helm lint` and `ct lint-and-install`
5. Or use one generic chart parametrised by values

**Why →** Shared templates reduce drift and let platform changes roll out uniformly.

**Interview tip →** Say you keep chart logic simple.

</details>

### Q127. What is an operator and when would you use one? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A team wants to run PostgreSQL on Kubernetes.

**What I'd do →**

1. An operator is a controller plus CRDs that encodes operational knowledge
2. Install the operator (for example CloudNativePG) with Helm
3. Create a `Cluster` custom resource declaring instances and backups
4. Check with `kubectl get cluster` and operator logs
5. Weigh managed RDS versus self-run

**Why →** Operators automate day-2 tasks such as failover, backups and upgrades.

**Interview tip →** Say you evaluate operator maturity before trusting it.

</details>

### Q128. A CRD was deleted and all custom resources disappeared. How do you prevent that? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Someone uninstalled an operator chart and the data objects vanished.

**What I'd do →**

1. Explain that deleting a CRD deletes all its objects
2. Helm does not manage CRDs in `crds/` on upgrade or uninstall
3. Install CRDs separately and protect with RBAC
4. Back up with Velero or GitOps
5. Add admission policy blocking CRD deletion

**Why →** CRD deletion is cascading and immediate.

**Interview tip →** Memorise this as a favourite gotcha.

</details>

### Q129. How would you design a GitOps repo layout and promotion flow for dev, staging and prod? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Several teams deploy to three environments through Argo CD.

**What I'd do →**

1. Separate app source repos from a config repo
2. Use Kustomize bases with env overlays, or ApplicationSets with a cluster generator
3. Promote by pull request changing the image tag in the next environment
4. Automate dev, require approval for prod
5. Protect branches and use CODEOWNERS

**Why →** Git history becomes the audit trail for every change.

**Interview tip →** Say promotion is a PR, not a pipeline push.

</details>

### Q130. Argo CD shows an app as `OutOfSync` constantly though nothing changed. Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A Deployment is always out of sync with Git.

**What I'd do →**

1. `argocd app diff web` to see the differing fields
2. Typical causes: HPA changing replicas, mutating webhooks adding fields, controllers adding defaults
3. Add `ignoreDifferences` with jsonPointers
4. Remove `replicas` from Git when using HPA
5. Use `ServerSideApply=true` sync option

**Why →** Fields written by other controllers show as drift unless ignored.

**Interview tip →** Mention managedFieldsManagers.

</details>

### Q131. Argo CD sync fails with `one or more objects failed to apply`. How do you debug? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A sync runs, then fails with a validation error.

**What I'd do →**

1. `argocd app get web` and `argocd app sync web --dry-run`
2. Read the resource-level message in the UI or CLI
3. Common causes: CRD not yet installed, namespace missing, immutable field changes, admission denial
4. Use sync waves and `CreateNamespace=true`
5. Check controller logs: `kubectl -n argocd logs deploy/argocd-application-controller`

**Why →** Ordering and validation errors are the usual reasons, and waves control order.

**Interview tip →** Mention `SkipDryRunOnMissingResource`.

</details>

### Q132. How do you handle Helm chart or operator upgrades safely in GitOps across many clusters? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A new operator version needs a CRD change.

**What I'd do →**

1. Read release notes for breaking changes and CRD updates
2. Upgrade CRDs first as a separate Argo Application with sync wave 0
3. Roll the operator to a canary cluster, validate, then widen
4. Keep the previous version ready for rollback
5. Test on a copy of production data

**Why →** CRDs are cluster-wide and not rolled back by Helm, so they get special care.

**Interview tip →** Say you stagger by cluster ring.

</details>

### Q133. Explain init containers and sidecars with a real use. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** An app must wait for a DB, and logs must be shipped.

**What I'd do →**

1. Init containers run to completion in order before the app, for waiting or migrations
2. Sidecars run alongside, for log shipping or proxies
3. Native sidecars (init containers with `restartPolicy: Always`) start before and stop after the app
4. Check `kubectl get pod` for `Init:0/1`
5. Read init logs with `kubectl logs pod -c init`

**Why →** Native sidecars fix ordering problems, such as Jobs that never finish because of a sidecar.

**Interview tip →** Mention native sidecars GA in 1.33.

</details>

### Q134. How do you view and edit a running resource, and why is that risky? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A developer edited a Deployment live.

**What I'd do →**

1. `kubectl edit deploy web` opens the live object
2. The change is not in Git and will be reverted or lost
3. Use `kubectl diff` to compare
4. Make the change in Git
5. Use RBAC to limit edits in production

**Why →** Live edits create drift.

**Interview tip →** Mention ArgoCD self-heal.

</details>

### Q135. How do you write a minimal controller or choose between operator frameworks? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The team needs a custom operator for an internal resource.

**What I'd do →**

1. Consider if a Helm chart or a CronJob is enough first
2. Use Kubebuilder or Operator SDK (Go) for full control, or Kopf (Python) for simple needs
3. Define the CRD schema with validation and a status subresource
4. Make reconciliation idempotent
5. Add finalizers for cleanup, metrics and leader election

**Why →** Reconcile loops must be level-triggered and idempotent.

**Interview tip →** Mention status conditions.

</details>

### Q136. A custom resource is stuck in Terminating. Why? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** `kubectl delete` hangs on a custom resource.

**What I'd do →**

1. `kubectl get cr x -o yaml` and inspect `metadata.finalizers` and `deletionTimestamp`
2. The operator is expected to remove the finalizer after cleanup
3. Check whether the operator is running: `kubectl get pods -n operators`
4. Fix the operator and let it finish cleanup
5. Last resort: `kubectl patch cr x --type=merge -p '{"metadata":{"finalizers":[]}}'` knowing external resources may leak

**Why →** A finalizer holds the object until its controller completes cleanup.

**Interview tip →** Say you patch finalizers only after confirming what cleanup is skipped.

</details>

### Q137. How do you template Kubernetes manifests without Helm? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A team dislikes Go templates.

**What I'd do →**

1. Use Kustomize overlays with patches
2. Use `cdk8s`, Jsonnet or CUE for generating manifests
3. Check output with `kustomize build`
4. Keep logic out of manifests when possible
5. Pick one tool per repo

**Why →** Different tools suit different teams, consistency matters most.

**Interview tip →** Show you know trade-offs, not just one tool.

</details>

### Q138. What are Helm hooks and give an example. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A DB migration must run before upgrade.

**What I'd do →**

1. Add annotation `"helm.sh/hook": pre-upgrade` to a Job
2. Add `"helm.sh/hook-delete-policy": before-hook-creation`
3. Set weights for ordering
4. Watch the hook with `kubectl get jobs`
5. Remember hooks are not part of the release manifest

**Why →** Hooks run at lifecycle points like pre-install and post-upgrade.

**Interview tip →** Mention test hooks with `helm test`.

</details>

### Q139. How would you migrate workloads from `kubectl apply` scripts to Argo CD without downtime? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Many resources already exist, deployed manually.

**What I'd do →**

1. Commit current live manifests to Git (`kubectl get -o yaml` cleaned)
2. Create the Argo Application with automated sync off
3. Use `argocd app diff` until it is clean
4. Enable sync without prune first
5. Then enable prune and self-heal

**Why →** Adoption is safe when Argo sees no difference before taking control.

**Interview tip →** Say you do the first sync manually.

</details>

### Q140. How do you version and promote container images? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Teams use `latest` tags.

**What I'd do →**

1. Tag images with the Git SHA and a semantic version
2. Never deploy `latest`
3. Prefer digests in production manifests
4. Scan before promotion
5. Automate bumps with Argo CD Image Updater or Renovate

**Why →** Immutable references make deployments reproducible and rollbacks real.

**Interview tip →** Mention `imagePullPolicy` defaults.

</details>

### Q141. What is Kustomize `configMapGenerator` and why is it useful? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Config changes must trigger rollouts.

**What I'd do →**

1. Define `configMapGenerator` in kustomization.yaml
2. Kustomize appends a hash to the name
3. References in Deployments are rewritten
4. A content change creates a new name, so pods roll
5. Old ConfigMaps can be pruned

**Why →** Hash suffixes tie pod revisions to config versions.

**Interview tip →** Cheap and neat approach for rollbacks.

</details>

### Q142. You want to uninstall a Helm release but keep the PVCs. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Data must survive an uninstall.

**What I'd do →**

1. Check how PVCs were created (chart template versus StatefulSet volumeClaimTemplates)
2. StatefulSet PVCs survive `helm uninstall`
3. Annotate chart-managed PVCs with `helm.sh/resource-policy: keep`
4. Verify with `kubectl get pvc` after uninstall
5. Back up first

**Why →** The keep policy tells Helm not to delete a resource on uninstall.

**Interview tip →** Always back up before uninstall.

</details>

## EKS and cluster operations

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q143. How do you create a production-ready EKS cluster and what do you decide up front? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You are asked to stand up the first EKS cluster for a team.

**What I'd do →**

1. Decide VPC layout: private subnets for nodes across three AZs, public only for load balancers
2. Create the cluster with Terraform or `eksctl create cluster -f cluster.yaml`
3. Choose managed node groups, Karpenter or Fargate
4. Enable control plane logs, secrets encryption and OIDC
5. Install add-ons: VPC CNI, CoreDNS, kube-proxy, EBS CSI, metrics-server

**Why →** Cluster shape is hard to change later, so network and identity choices come first.

**Interview tip →** Say infrastructure is code, never console-built.

</details>

### Q144. Pods on EKS run out of IP addresses. What happened and how do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** New pods fail with `failed to assign an IP address to container`.

**What I'd do →**

1. Understand the VPC CNI gives each pod a VPC IP from the node's ENIs, limited by instance type
2. `kubectl -n kube-system logs ds/aws-node` and check subnet free IPs
3. Enable prefix delegation (`ENABLE_PREFIX_DELEGATION=true`)
4. Add secondary CIDR with custom networking
5. Use larger instances or more subnets

**Why →** Pod density equals ENIs times IPs per ENI, and subnet size is the second ceiling.

**Interview tip →** Mention `/16` VPC planning.

</details>

### Q145. How do you give a new engineer access to an EKS cluster? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Old method used the `aws-auth` ConfigMap.

**What I'd do →**

1. Use EKS access entries: `aws eks create-access-entry --cluster-name prod --principal-arn <role-arn>`
2. Associate an access policy such as `AmazonEKSViewPolicy` with scope namespace or cluster
3. Or map to a Kubernetes group and use RBAC
4. Test with `aws eks update-kubeconfig` then `kubectl auth whoami`
5. Remove the ConfigMap path over time

**Why →** Access entries are API-managed and avoid lockouts caused by editing aws-auth.

**Interview tip →** Mention the aws-auth lockout risk.

</details>

### Q146. Compare managed node groups, self-managed nodes, Fargate and Karpenter. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The team needs to choose a compute model.

**What I'd do →**

1. Managed node groups: AWS handles AMI updates and draining, simple and common
2. Self-managed: full control, more work
3. Fargate: no nodes, per-pod pricing, no DaemonSets or privileged pods
4. Karpenter: dynamic, cost-aware, fast scale
5. Mix: a small managed group for system add-ons plus Karpenter for workloads

**Why →** Trade control against operational effort.

**Interview tip →** Say you run system pods on a stable group.

</details>

### Q147. How do you upgrade an EKS cluster from 1.31 to 1.32 with minimal risk? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Quarterly upgrade of a production cluster.

**What I'd do →**

1. Read release notes and check deprecated APIs with `kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis` or Pluto/kubent
2. Check EKS upgrade insights
3. Test in staging
4. Upgrade the control plane (`aws eks update-cluster-version`)
5. Upgrade add-ons, then node groups (one minor at a time, nodes within skew), verifying PDBs and workloads

**Why →** The control plane moves first and nodes must stay within the supported version skew.

**Interview tip →** Say you never skip minors.

</details>

### Q148. A node group upgrade is slow or stuck on EKS. What do you look at? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rolling update has been running for an hour.

**What I'd do →**

1. `aws eks describe-update` for errors
2. `kubectl get nodes` for NotReady or SchedulingDisabled
3. Look for PDBs blocking drains
4. Adjust `updateConfig.maxUnavailable`
5. Check that new nodes can join (IAM, security groups, subnets, AMI)

**Why →** The most common blocker is a PDB with zero allowed disruptions.

**Interview tip →** Mention surge upgrades.

</details>

### Q149. How do you connect to a node for debugging without SSH? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Nodes have no SSH keys.

**What I'd do →**

1. `kubectl debug node/ip-10-0-1-5 -it --image=ubuntu` which mounts the host at `/host`
2. Or use AWS SSM Session Manager
3. Inspect with `chroot /host`
4. Use journalctl for kubelet
5. Delete the debug pod after

**Why →** Debug pods avoid opening SSH and leave an audit trail.

**Interview tip →** Mention SSM as the standard.

</details>

### Q150. What add-ons do you install on EKS first and why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A fresh cluster has only the defaults.

**What I'd do →**

1. VPC CNI, CoreDNS, kube-proxy as managed add-ons
2. EBS CSI driver for volumes
3. AWS Load Balancer Controller for ingress and NLB
4. metrics-server, then Prometheus stack
5. Karpenter or Cluster Autoscaler, and external-dns plus cert-manager

**Why →** These cover networking, storage, traffic, observability and scaling.

**Interview tip →** Manage add-ons through IaC.

</details>

### Q151. How do you design multi-AZ resilience for a service on EKS? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The business wants to survive an AZ outage.

**What I'd do →**

1. Run nodes in at least three AZs
2. Use `topologySpreadConstraints` on zone with `maxSkew: 1`
3. Set PDBs and enough replicas to absorb one AZ loss
4. Use a regional database (Aurora Multi-AZ)
5. Test with a zone drain or AWS FIS experiment

**Why →** Spreading plus headroom means the loss of a zone leaves capacity.

**Interview tip →** Mention cross-AZ data transfer cost.

</details>

### Q152. Your EKS cluster endpoint is private. How does CI reach it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** GitHub-hosted runners cannot connect.

**What I'd do →**

1. Use self-hosted runners inside the VPC
2. Or run CI in AWS CodeBuild with VPC config
3. Or GitOps pull model, so the cluster pulls from Git
4. Or temporarily allow specific CIDRs
5. Verify with `kubectl get ns` from the runner

**Why →** Pull-based deployment removes the need for inbound access.

**Interview tip →** Say GitOps solves this elegantly.

</details>

### Q153. Explain EKS Pod Identity vs IRSA. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Choosing between workload IAM methods.

**What I'd do →**

1. IRSA uses the cluster OIDC provider and a role trust policy per cluster
2. Pod Identity uses an agent and `CreatePodIdentityAssociation` with no OIDC setup
3. Pod Identity roles can be reused across clusters
4. Both give temporary credentials
5. Check SDK versions for support

**Why →** Pod Identity removes per-cluster trust policy edits.

**Interview tip →** Know both.

</details>

### Q154. How do you control cost on a large EKS estate? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** EKS spend doubled in a quarter.

**What I'd do →**

1. Tag and attribute cost with Kubecost or OpenCost
2. Right-size requests
3. Use Karpenter consolidation and Spot for stateless
4. Use Graviton instances
5. Reduce cross-AZ traffic and idle environments (scale dev to zero at night)

**Why →** Most savings come from requests, instance choice and idle capacity.

**Interview tip →** Quote percentages.

</details>

### Q155. What do you do when a node is NotReady? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A node flipped to NotReady.

**What I'd do →**

1. `kubectl describe node` and read Conditions
2. Check kubelet and container runtime on the node
3. Check disk, memory, PID pressure
4. Check network to the API server
5. Cordon, drain if possible, replace the node

**Why →** NotReady means the kubelet stopped reporting healthy status.

**Interview tip →** Cattle not pets: replace nodes.

</details>

### Q156. How do you view logs and metrics on EKS? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** No observability yet.

**What I'd do →**

1. Enable control plane logs to CloudWatch
2. Run Fluent Bit as DaemonSet or use CloudWatch Observability add-on
3. Install kube-prometheus-stack or Amazon Managed Prometheus
4. Grafana dashboards
5. Alerts for restarts, pending pods, node pressure

**Why →** Visibility into control plane, nodes and workloads is required.

**Interview tip →** Say you alert on symptoms.

</details>

### Q157. How do you restrict an EKS node group to private subnets but pull images from ECR? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Nodes have no internet path.

**What I'd do →**

1. Create VPC endpoints for ECR API, ECR DKR and S3 gateway endpoint
2. Add endpoints for STS, EC2 and logs as needed
3. Check endpoint security groups allow 443
4. Verify with `crictl pull`
5. Use pull-through cache for public images

**Why →** Private endpoints keep traffic inside AWS.

**Interview tip →** Don't forget the S3 gateway endpoint.

</details>

### Q158. How do you handle a bad AMI or add-on rollout across 40 nodes? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Nodes boot but pods fail.

**What I'd do →**

1. Pause the node group update
2. Roll back launch template version
3. Cordon bad nodes
4. Move workloads using drain
5. Test AMI in canary node group first

**Why →** Canary node groups limit blast radius.

**Interview tip →** Say AMIs are versioned and canaried.

</details>

### Q159. Explain how DNS and certificates are automated on EKS. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Teams request hostnames by hand.

**What I'd do →**

1. Install external-dns with IAM for Route 53
2. Add `external-dns.alpha.kubernetes.io/hostname` annotations or use Ingress hosts
3. Install cert-manager with a ClusterIssuer (Let's Encrypt DNS-01 via Route 53)
4. Request certificates through Ingress annotations or Certificate objects
5. Verify with `kubectl get certificate`

**Why →** Automation removes manual tickets and expiring certificates.

**Interview tip →** Know DNS-01 vs HTTP-01.

</details>

### Q160. You need to run Windows or ARM workloads. What changes? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** New Graviton nodes are added.

**What I'd do →**

1. Build multi-arch images with `docker buildx --platform linux/amd64,linux/arm64`
2. Label and taint ARM nodes
3. Use nodeSelector `kubernetes.io/arch`
4. Check add-ons support ARM
5. Test performance

**Why →** Image architecture must match the node.

**Interview tip →** Name `exec format error`.

</details>

### Q161. What is the control plane responsible for and who manages it on EKS? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A manager asks what AWS handles.

**What I'd do →**

1. AWS manages API servers, etcd, scheduler and controllers
2. You manage nodes, add-ons, workloads and IAM
3. Control plane logging is optional
4. Versions need upgrades by you
5. Pay per cluster hour

**Why →** It is a shared responsibility model.

**Interview tip →** Mention extended support costs.

</details>

### Q162. How would you set up multi-cluster management for 10 EKS clusters? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Many clusters drift.

**What I'd do →**

1. Define clusters with Terraform modules
2. Use Argo CD ApplicationSets targeting a cluster list
3. Standardise add-ons via a platform repo
4. Centralise logs, metrics and policy
5. Roll changes by rings

**Why →** Fleet consistency needs code and staged rollouts.

**Interview tip →** Say rings: dev, staging, prod-1, prod-rest.

</details>

### Q163. kubectl on a laptop shows `You must be logged in to the server (Unauthorized)`. What do you do on EKS? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Access broke after a role change.

**What I'd do →**

1. `aws sts get-caller-identity` to check identity
2. `aws eks update-kubeconfig --name prod`
3. Check the identity has an access entry or aws-auth mapping
4. Check token expiry and clock skew
5. Check the assumed role ARN versus the mapped one

**Why →** Authentication maps an IAM principal to a Kubernetes user.

**Interview tip →** Role ARNs in mappings must not include the path.

</details>

### Q164. How do you run batch jobs at scale cheaply? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Nightly ETL with 500 pods.

**What I'd do →**

1. Use Jobs with `parallelism` and `completions`
2. Run on Spot via Karpenter NodePool
3. Use indexed Jobs
4. Set resource requests accurately
5. Use `ttlSecondsAfterFinished`

**Why →** Batch tolerates interruption, so spot saves cost.

**Interview tip →** Consider Argo Workflows or Volcano.

</details>

### Q165. How do you check which Kubernetes version a cluster and its nodes run? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Version skew question.

**What I'd do →**

1. `kubectl version`
2. `kubectl get nodes -o wide`
3. `aws eks describe-cluster --name prod --query cluster.version`
4. Compare kubelet versions
5. Keep nodes within skew

**Why →** Skew policy limits kubelet to the control plane version minus a few minors.

**Interview tip →** Say supported skew is N-3 for kubelet.

</details>

### Q166. Describe your approach when API server latency rises in EKS. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** kubectl is slow and controllers time out.

**What I'd do →**

1. Check API server metrics `apiserver_request_duration_seconds`
2. Find heavy clients: list calls without limits in audit logs
3. Look for webhook latency
4. Check etcd size and object counts
5. Enable API Priority and Fairness tuning and fix noisy controllers

**Why →** Slow admission webhooks and expensive lists are frequent causes.

**Interview tip →** Say you look at webhook latency first.

</details>

### Q167. How do you roll out a cluster-wide change such as new log agent config safely? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A DaemonSet config affects every node.

**What I'd do →**

1. Test on a staging cluster
2. Use `updateStrategy.rollingUpdate.maxUnavailable`
3. Canary on one node group using nodeSelector
4. Watch resource usage
5. Roll back by reverting Git

**Why →** Cluster-wide agents are high blast radius.

**Interview tip →** Mention canary nodes.

</details>

### Q168. What is a namespace and how do you organise them? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Teams mix everything in default.

**What I'd do →**

1. One namespace per team or app and environment
2. Apply quotas, RBAC, network policies per namespace
3. Never use `default`
4. Label namespaces
5. Use `kubectl config set-context --current --namespace=app`

**Why →** Namespaces scope names and policies.

**Interview tip →** Namespaces are not security boundaries alone.

</details>

## Upgrades, etcd, disaster recovery and incidents

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q169. How do you back up etcd on a self-managed cluster? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A kubeadm cluster has no backup.

**What I'd do →**

1. `ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key`
2. Verify with `etcdutl snapshot status /backup/etcd.db`
3. Copy offsite to S3
4. Schedule with a CronJob or systemd timer
5. Test restore regularly

**Why →** A snapshot captures all cluster state, and restoring is the only proof of a good backup.

**Interview tip →** Say on EKS you do not own etcd, use Velero or GitOps.

</details>

### Q170. How do you restore a cluster from an etcd snapshot? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Control plane data is corrupted.

**What I'd do →**

1. Stop kube-apiserver and etcd static pods by moving manifests out of `/etc/kubernetes/manifests`
2. `etcdutl snapshot restore /backup/etcd.db --data-dir /var/lib/etcd-restore`
3. Point the etcd manifest data dir to the restored directory
4. For multi-node, restore each member with `--initial-cluster` flags
5. Start components and verify with `kubectl get nodes`

**Why →** Restore creates a new cluster identity, so all members must be restored consistently.

**Interview tip →** Rehearse this at least once.

</details>

### Q171. etcd database space exceeded: `mvcc: database space exceeded`. What now? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** API writes fail.

**What I'd do →**

1. `etcdctl endpoint status -w table` to see DB size
2. `etcdctl alarm list`
3. Compact: get revision and `etcdctl compact <rev>`
4. `etcdctl defrag` one member at a time
5. `etcdctl alarm disarm`, then find noisy writers (events, CRDs) and raise `--quota-backend-bytes` if justified

**Why →** Compaction removes old revisions, defrag returns space to disk.

**Interview tip →** Mention default 2 GiB quota.

</details>

### Q172. Walk through a kubeadm minor version upgrade. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Move from 1.31 to 1.32 on a self-managed cluster.

**What I'd do →**

1. Upgrade kubeadm on the first control plane node and `kubeadm upgrade plan`
2. `kubeadm upgrade apply v1.32.x`
3. Upgrade kubelet and kubectl, restart kubelet
4. On other control plane nodes `kubeadm upgrade node`
5. For workers cordon and drain, `kubeadm upgrade node`, upgrade kubelet, uncordon

**Why →** Control plane first, then workers one by one.

**Interview tip →** Say take an etcd snapshot before.

</details>

### Q173. How do you find deprecated or removed APIs before an upgrade? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Manifests use `policy/v1beta1`.

**What I'd do →**

1. Run Pluto: `pluto detect-files -d manifests/` and `pluto detect-helm`
2. Or kubent against the cluster
3. Check API server metric `apiserver_requested_deprecated_apis`
4. Update manifests and Helm charts
5. Test in staging

**Why →** Removed APIs break applies and controllers after upgrade.

**Interview tip →** Show you check before, not after.

</details>

### Q174. A cluster upgrade broke an operator because its CRD API version was removed. How do you recover? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** After the upgrade, custom resources cannot be served.

**What I'd do →**

1. Check `kubectl get crd x -o yaml` for `storedVersions`
2. Re-install the operator version that supports the cluster
3. Migrate stored objects to the new version (storage version migration)
4. Update manifests to the served version
5. Roll forward rather than back, since control plane downgrade is unsupported

**Why →** Stored versions must be migrated before an old version is dropped.

**Interview tip →** Control plane rollback is not supported.

</details>

### Q175. What are the main control plane components and what does each do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Interview warm-up.

**What I'd do →**

1. kube-apiserver: front door and validation
2. etcd: stores state
3. kube-scheduler: assigns pods to nodes
4. kube-controller-manager: runs control loops
5. cloud-controller-manager: talks to cloud APIs

**Why →** Everything flows through the API server.

**Interview tip →** Add kubelet and kube-proxy on nodes.

</details>

### Q176. What happens when you run `kubectl apply -f deploy.yaml`? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Walk through the flow.

**What I'd do →**

1. kubectl sends the object to the API server after authentication and authorization
2. Admission controllers mutate and validate
3. The object is persisted in etcd
4. Deployment controller creates a ReplicaSet, which creates pods
5. Scheduler binds pods, kubelet starts containers via CRI

**Why →** It is a chain of watch-driven controllers.

**Interview tip →** Mention the watch mechanism.

</details>

### Q177. The whole control plane is down on a self-managed cluster. Do running workloads keep working? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** API servers are unreachable.

**What I'd do →**

1. Pods already running keep running since the kubelet acts independently
2. Services and kube-proxy rules stay in place
3. No new scheduling, scaling or rollouts
4. Node failures are not detected
5. Restore the control plane from static pod manifests or snapshot

**Why →** The data plane is decoupled from the control plane for steady state.

**Interview tip →** Highlights why control plane HA matters.

</details>

### Q178. How do you set up a highly available control plane? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Single master is a risk.

**What I'd do →**

1. Run three or five control plane nodes across AZs
2. Put a load balancer in front of API servers
3. Use stacked or external etcd with odd members
4. Back up etcd
5. Use `kubeadm init --control-plane-endpoint`

**Why →** etcd needs quorum, so odd members and spread matter.

**Interview tip →** Know quorum maths: 3 tolerates 1, 5 tolerates 2.

</details>

### Q179. You get paged: API latency high and pods failing. Describe your incident approach. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A general production incident.

**What I'd do →**

1. Declare the incident and assign roles
2. Check recent changes (deploys, upgrades) and roll back if correlated
3. Check cluster health: nodes, control plane, DNS, ingress
4. Use `kubectl get events -A --sort-by=.lastTimestamp`
5. Communicate status, then write a blameless postmortem

**Why →** Stabilise first, understand later.

**Interview tip →** Say you roll back before root-causing.

</details>

### Q180. Velero restore into a new cluster: what do you verify? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** DR drill.

**What I'd do →**

1. Install Velero with the same bucket
2. `velero restore create --from-backup daily-1`
3. Check namespaces, PVs, secrets
4. Check storage class mapping
5. Run smoke tests

**Why →** Restores are complete only when workloads function.

**Interview tip →** Mention restore time objectives.

</details>

### Q181. Define RTO and RPO and how they shape a Kubernetes DR plan. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Business asks for a DR design.

**What I'd do →**

1. RPO is how much data you can lose, RTO is how long you can be down
2. Low RPO needs continuous data replication
3. Low RTO needs warm standby cluster
4. Keep manifests in Git for quick rebuild
5. Practise failover quarterly

**Why →** Targets determine cost and architecture.

**Interview tip →** Name active-active versus pilot light.

</details>

### Q182. How do you check the health of cluster components? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Quick triage.

**What I'd do →**

1. `kubectl get nodes`
2. `kubectl get pods -n kube-system`
3. `kubectl get --raw='/readyz?verbose'`
4. `kubectl get events -A`
5. `kubectl top nodes`

**Why →** These cover the main failure areas.

**Interview tip →** Mention `/livez` and `/readyz`.

</details>

### Q183. Certificates on a kubeadm cluster expire. What happens and how do you fix it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** kubectl returns `x509: certificate has expired`.

**What I'd do →**

1. `kubeadm certs check-expiration`
2. `kubeadm certs renew all`
3. Restart control plane static pods by moving manifests out and back
4. Update admin kubeconfig
5. Monitor expiry going forward

**Why →** kubeadm certs last one year unless renewed by upgrade.

**Interview tip →** Upgrading renews certs.

</details>

### Q184. How do you debug a pod that has no shell in its image? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Distroless images.

**What I'd do →**

1. `kubectl debug -it pod/app --image=busybox --target=app`
2. Use `--copy-to` to clone the pod with a debug image
3. Inspect shared process namespace
4. Check network from the same netns
5. Remove debug pods

**Why →** Ephemeral containers attach tooling without rebuilding the image.

**Interview tip →** Say you do not bake debug tools into prod images.

</details>

### Q185. How would you perform a zero-downtime migration of a stateful database from a VM into Kubernetes? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Move PostgreSQL to a CloudNativePG cluster.

**What I'd do →**

1. Set up the operator and a replica cluster
2. Configure logical or physical replication from the VM
3. Verify lag and consistency
4. Cut over during a short write freeze: promote and switch app connection strings
5. Keep the VM as fallback until validated

**Why →** Replication allows a small controlled cutover window.

**Interview tip →** Say you rehearse and have a rollback plan.

</details>

### Q186. How do you test that your workloads survive failures? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Leadership wants resilience proof.

**What I'd do →**

1. Use chaos tools such as Chaos Mesh, LitmusChaos or AWS FIS
2. Start with pod kills, then node drains, then AZ failure
3. Define steady-state metrics
4. Run in staging first, then in prod with guardrails
5. Fix findings and repeat

**Why →** Failure drills expose missing PDBs, probes, and capacity.

**Interview tip →** Mention game days.

</details>

### Q187. What is the role of kubelet and what happens if it stops? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Node basics.

**What I'd do →**

1. Kubelet registers the node, runs pods through the container runtime and reports status
2. If it stops the node becomes NotReady after about 40 seconds
3. Running containers keep running
4. Pods are evicted after the toleration period (300s default)
5. Restart with `systemctl restart kubelet`

**Why →** Kubelet is the node agent.

**Interview tip →** Mention `node-monitor-grace-period`.

</details>

### Q188. Explain how you would upgrade to a new CNI or switch from one CNI to another. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Moving from VPC CNI to Cilium.

**What I'd do →**

1. Test in a new cluster first
2. Prefer building a new node group or cluster
3. Plan IP ranges and network policies mapping
4. Migrate workloads gradually
5. Have rollback of node group

**Why →** CNI swaps in place are risky since pod networking breaks.

**Interview tip →** Prefer blue/green clusters.

</details>

### Q189. How do you handle rapid growth to 5000 pods in a cluster? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Capacity planning.

**What I'd do →**

1. Check API server and etcd limits
2. Scale CoreDNS and monitoring
3. Use IPAM prefix delegation
4. Watch scheduler throughput and Events
5. Consider splitting into multiple clusters

**Why →** Control plane scaling limits appear before node limits.

**Interview tip →** Mention cell-based architecture.

</details>

### Q190. A namespace is stuck Terminating. What are the causes and the proper fix? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** `kubectl delete ns app` never finishes.

**What I'd do →**

1. `kubectl get ns app -o yaml` and check `status.conditions` for what remains
2. `kubectl api-resources --verbs=list --namespaced -o name | xargs -n1 kubectl get -n app` to find leftover objects
3. Common blockers: finalizers on custom resources, unavailable aggregated API like metrics
4. Fix the owner (operator or APIService)
5. Only then remove finalizers

**Why →** Namespaces cannot finish deletion while any object or APIService still blocks.

**Interview tip →** Do not use the raw finalize API without checking.

</details>

### Q191. How do you migrate workloads between namespaces? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reorganising.

**What I'd do →**

1. Export YAML and clean namespace and uid fields
2. Re-apply in the new namespace
3. Copy Secrets and ConfigMaps
4. Move data with a snapshot-restored PVC
5. Shift traffic and delete old

**Why →** There is no rename, only recreate.

**Interview tip →** Mention Velero namespace mapping.

</details>

### Q192. How do you read a Kubernetes event and why do events disappear? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Events are missing.

**What I'd do →**

1. `kubectl get events --sort-by=.lastTimestamp`
2. `kubectl describe` includes events for that object
3. Events live about one hour by default
4. Ship events to logs with an exporter
5. Use `-A` for all namespaces

**Why →** Events are short-lived objects in etcd.

**Interview tip →** Export them for history.

</details>

### Q193. Explain leader election and why controllers use it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multiple replicas of a controller.

**What I'd do →**

1. Replicas compete for a Lease object
2. One is leader, others standby
3. Leader renews the lease regularly
4. On failure another takes over
5. Check `kubectl get lease -n kube-system`

**Why →** Only one active reconciler prevents conflicting actions.

**Interview tip →** Mention scheduler and controller-manager use leases.

</details>

### Q194. How do you troubleshoot slow `kubectl` commands? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Every command takes 10 seconds.

**What I'd do →**

1. `kubectl get pods -v=8` to see request timing
2. Check the kubeconfig exec plugin (aws token) latency
3. Check API server latency and discovery cache
4. Check network and VPN
5. Clear `~/.kube/cache`

**Why →** Often the credential plugin or discovery is slow, not the cluster.

**Interview tip →** Use `-v` verbosity.

</details>

### Q195. What is the difference between a Deployment, StatefulSet and DaemonSet? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fundamentals.

**What I'd do →**

1. Deployment: stateless, interchangeable replicas
2. StatefulSet: stable identity and storage, ordered operations
3. DaemonSet: one pod per node
4. Pick by state and placement needs
5. Verify with `kubectl get deploy,sts,ds`

**Why →** Each controller encodes a different guarantee.

**Interview tip →** Give an example for each.

</details>

### Q196. Your team asks for in-place pod resize and sidecar features. How do you handle feature gates and version adoption? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A new Kubernetes feature is attractive.

**What I'd do →**

1. Check the feature stage (alpha, beta, GA) in the release notes
2. Alpha is off by default and unsupported on EKS
3. Pilot on a test cluster
4. Adopt GA features in production
5. Track the deprecation policy

**Why →** Production should rely on GA or well-proven beta features.

**Interview tip →** Say alpha features do not go to prod.

</details>

### Q197. How do you list everything running in a namespace and see what is unhealthy? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You are handed an unfamiliar namespace during an incident.

**What I'd do →**

1. `kubectl get all -n app` for the main workload objects
2. `kubectl get pods -n app -o wide` to see nodes and restarts
3. `kubectl get events -n app --sort-by=.lastTimestamp`
4. `kubectl top pods -n app` for resource usage
5. `kubectl describe` on anything not Running or Ready

**Why →** A few commands give you state, placement, events and usage within a minute.

**Interview tip →** Note that `get all` does not include ConfigMaps, Secrets, Ingress or PVCs.

</details>

### Q198. How do you stop and start a workload for the night without deleting it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A dev environment costs money overnight.

**What I'd do →**

1. `kubectl scale deploy --all --replicas=0 -n dev`
2. Record the original counts first (`kubectl get deploy -n dev -o custom-columns=NAME:.metadata.name,REPLICAS:.spec.replicas`)
3. Scale back up in the morning
4. Automate with a CronJob and a scoped ServiceAccount, or KEDA cron scaler
5. Keep PVCs, which are not deleted by scaling

**Why →** Scaling to zero keeps all objects and data while releasing compute.

**Interview tip →** Mention that GitOps will undo manual scaling unless paused.

</details>

### Q199. How do you copy files in and out of a pod? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need to grab a heap dump from a pod.

**What I'd do →**

1. `kubectl cp app-pod:/tmp/heap.hprof ./heap.hprof -c app -n prod`
2. For large files use `kubectl exec ... -- tar cf - /tmp | tar xf -`
3. Check the image has `tar` since cp relies on it
4. Clean up the file in the pod afterwards
5. Prefer shipping dumps to S3 automatically for production

**Why →** kubectl cp streams a tar archive through the API server.

**Interview tip →** Say distroless images need an ephemeral debug container instead.

</details>

### Q200. What labels and annotations should every workload carry, and why? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The platform team wants consistent metadata for cost and ownership.

**What I'd do →**

1. Use recommended labels such as `app.kubernetes.io/name`, `app.kubernetes.io/instance`, `app.kubernetes.io/version`
2. Add `team`, `env` and `cost-center` labels
3. Use annotations for non-identifying metadata such as runbook links
4. Enforce with Kyverno or ValidatingAdmissionPolicy
5. Query with `kubectl get pods -l team=payments`

**Why →** Labels drive selectors, cost allocation and policy, so they must be consistent.

**Interview tip →** Remember labels select, annotations describe.

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
