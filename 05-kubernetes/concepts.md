# Kubernetes — Core Concepts

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

> Core Kubernetes concepts, internals, comparisons and "what happens when…" questions · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Kubernetes](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Architecture & Control Plane](#architecture--control-plane) — Q1–Q27
- [Pods & Workloads](#pods--workloads) — Q28–Q56
- [Services, Networking, Ingress & Gateway API](#services-networking-ingress--gateway-api) — Q57–Q83
- [Configuration & Storage](#configuration--storage) — Q84–Q107
- [Scheduling, Resources & Autoscaling](#scheduling-resources--autoscaling) — Q108–Q131
- [Security: RBAC, ServiceAccounts & Pod Security](#security-rbac-serviceaccounts--pod-security) — Q132–Q151
- [Extending Kubernetes: Helm, CRDs, Operators & kubectl](#extending-kubernetes-helm-crds-operators--kubectl) — Q152–Q170
- [EKS, Upgrades & etcd Backup](#eks-upgrades--etcd-backup) — Q171–Q200

---

## Architecture & Control Plane

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. What are the main components of a Kubernetes cluster? 🟢

<details>
<summary><b>Answer</b></summary>

A cluster has a **control plane** and **worker nodes**. The control plane runs `kube-apiserver`, `etcd`, `kube-scheduler` and `kube-controller-manager` (plus `cloud-controller-manager` on cloud providers). Each worker node runs a `kubelet`, a container runtime (containerd or CRI-O) and `kube-proxy` (or a CNI that replaces it). Everything talks to the API server; nothing talks to etcd directly except the API server.

</details>

### Q2. What does the kube-apiserver do? 🟢

<details>
<summary><b>Answer</b></summary>

It is the front door of the cluster: a REST API that authenticates, authorizes, runs admission control, validates objects and persists them to etcd. It is stateless, so you can run several behind a load balancer. `kubectl`, the kubelet, controllers and the scheduler are all just API clients.
```bash
kubectl get --raw /readyz?verbose | head
```

</details>

### Q3. What is etcd and why does Kubernetes need it? 🟢

<details>
<summary><b>Answer</b></summary>

etcd is a strongly consistent, distributed key-value store that holds all cluster state (every object you create). It uses the Raft consensus protocol, so it needs a quorum of members (3 or 5 in production). Lose etcd without a backup and you lose the cluster's desired state, even though running containers keep running for a while.

</details>

### Q4. What does the kube-scheduler do? 🟢

<details>
<summary><b>Answer</b></summary>

It watches for Pods with an empty `spec.nodeName` and picks a node for each. It **filters** nodes that cannot run the Pod (resources, taints, affinity, volume constraints) and then **scores** the survivors, binding the Pod to the highest score. It only writes the binding; the kubelet on that node starts the containers.

</details>

### Q5. What does the kube-controller-manager do? 🟢

<details>
<summary><b>Answer</b></summary>

It runs the built-in control loops in one binary: Deployment, ReplicaSet, Node, Job, EndpointSlice, ServiceAccount, Namespace, PV controllers and more. Each loop watches the API for actual state, compares it to desired state in `spec`, and acts to close the gap. This reconcile pattern is the heart of Kubernetes.

</details>

### Q6. What does the kubelet do? 🟢

<details>
<summary><b>Answer</b></summary>

The kubelet is the node agent. It watches the API server for Pods bound to its node, tells the container runtime (via CRI) to pull images and start containers, mounts volumes, runs probes, and reports Pod and node status back. It also manages static Pods from a local manifest directory.
```bash
systemctl status kubelet
journalctl -u kubelet -f
```

</details>

### Q7. What does kube-proxy do? 🟢

<details>
<summary><b>Answer</b></summary>

It programs the node's packet-forwarding rules (iptables, IPVS or nftables mode) so that traffic to a Service ClusterIP is load-balanced to one of the backing Pod IPs. It watches Services and EndpointSlices. Some CNIs, such as Cilium, can replace it entirely with eBPF.

</details>

### Q8. What is a namespace and what does it isolate? 🟢

<details>
<summary><b>Answer</b></summary>

A namespace is a logical partition of namespaced objects (Pods, Services, Secrets, ConfigMaps, etc.) within one cluster. It scopes names, RBAC, ResourceQuotas and NetworkPolicies. It is **not** a security or network boundary by default; Pods in different namespaces can still talk unless a NetworkPolicy blocks them.
```bash
kubectl get ns
kubectl api-resources --namespaced=false   # cluster-scoped kinds
```

</details>

### Q9. What is the difference between imperative and declarative management? 🟢

<details>
<summary><b>Answer</b></summary>

Imperative means telling the cluster what to do (`kubectl run`, `kubectl scale`, `kubectl create`). Declarative means describing desired state in YAML and applying it (`kubectl apply -f`), letting Kubernetes reconcile. In production we use declarative files in Git, ideally applied by GitOps tooling, because they are reviewable and repeatable.

</details>

### Q10. Explain the reconcile loop (level-triggered control) model. 🟡

<details>
<summary><b>Answer</b></summary>

A controller watches objects, reads current state, compares it with `spec`, and makes the smallest change to converge. It is **level-triggered**: it acts on the current state, not on individual events, so a missed event or a restart does not cause drift; the next resync fixes it. This is why controllers must be idempotent.

</details>

### Q11. What are the API request stages inside the API server? 🟡

<details>
<summary><b>Answer</b></summary>

Authentication, then authorization (RBAC, Node), then **mutating admission**, then schema validation, then **validating admission**, and finally persistence to etcd. Mutating webhooks can change the object (inject sidecars, defaults); validating ones can only accept or reject. A request rejected at any stage never reaches etcd.

</details>

### Q12. What happens, step by step, when you run `kubectl apply -f deploy.yaml`? 🟡

<details>
<summary><b>Answer</b></summary>

`kubectl` reads kubeconfig, sends the object to the API server, which authenticates, authorizes and admits it, then stores the Deployment in etcd. The Deployment controller creates a ReplicaSet; the ReplicaSet controller creates Pods; the scheduler binds each Pod to a node; the kubelet on that node pulls the image and starts the containers via CRI; and endpoints are updated once the Pod is Ready.

</details>

### Q13. How do watches work and why are they important? 🟡

<details>
<summary><b>Answer</b></summary>

Clients open a long-lived HTTP stream (`?watch=true`) with a `resourceVersion` and receive ADDED/MODIFIED/DELETED events. Informers in client-go cache objects locally and use watches to stay current, so controllers do not poll the API server. If the watch falls too far behind (410 Gone), the client relists.

</details>

### Q14. What is `resourceVersion` and how does optimistic concurrency work? 🟡

<details>
<summary><b>Answer</b></summary>

Every object carries a `resourceVersion` derived from etcd's revision. An update must include the version it was based on; if the object changed in between, the API server returns `409 Conflict` ("the object has been modified; please apply your changes to the latest version"). Clients then re-read and retry instead of overwriting blindly.

</details>

### Q15. What is the difference between `kubectl apply` client-side and server-side apply? 🟡

<details>
<summary><b>Answer</b></summary>

Client-side apply stores the last-applied config in an annotation and computes a three-way patch locally. Server-side apply (`--server-side`) tracks **field ownership** (`managedFields`) on the server, so multiple managers can own different fields and conflicts are detected explicitly.
```bash
kubectl apply --server-side --field-manager=ci -f app.yaml
kubectl apply --server-side --force-conflicts -f app.yaml
```

</details>

### Q16. Where do the control plane components usually run on a kubeadm cluster, and how? 🟡

<details>
<summary><b>Answer</b></summary>

On kubeadm clusters the API server, scheduler, controller manager and etcd run as **static Pods**: the kubelet reads manifests from `/etc/kubernetes/manifests` and runs them without the API server. Edit a manifest there and the kubelet restarts the component. Managed services (EKS, GKE, AKS) hide the control plane entirely.

</details>

### Q17. What is the role of the cloud-controller-manager? 🟡

<details>
<summary><b>Answer</b></summary>

It splits cloud-specific logic out of the core controllers. It runs the node controller (detect deleted cloud instances), the route controller, and the service controller that provisions cloud load balancers for `type: LoadBalancer` Services. On EKS the AWS Load Balancer Controller handles much of this separately.

</details>

### Q18. How does a node join and register with the cluster? 🟡

<details>
<summary><b>Answer</b></summary>

The kubelet authenticates to the API server using a bootstrap token or client certificate (TLS bootstrapping), creates a CertificateSigningRequest, gets a signed client cert, then registers a `Node` object. Cloud setups usually use a managed bootstrap path (kubeadm join, EKS bootstrap via the AMI and IAM role mapped through access entries or aws-auth).

</details>

### Q19. How do nodes report health, and what are Leases for? 🟡

<details>
<summary><b>Answer</b></summary>

Each kubelet updates `status` conditions periodically and renews a small `Lease` object in `kube-node-lease` as a cheap heartbeat. The node controller marks a node `NotReady`/Unknown when heartbeats stop (`node-monitor-grace-period`, default 50s in recent versions) and later evicts its Pods via taints.
```bash
kubectl get lease -n kube-node-lease
```

</details>

### Q20. What is a CRI, CNI and CSI? 🟡

<details>
<summary><b>Answer</b></summary>

They are the three plug-in interfaces: **CRI** (Container Runtime Interface) lets the kubelet drive runtimes like containerd; **CNI** (Container Network Interface) provisions Pod networking and IPs; **CSI** (Container Storage Interface) lets storage vendors provide volumes without in-tree code. Swapping an implementation does not require changing Kubernetes itself.

</details>

### Q21. How does leader election work for the scheduler and controller manager? 🔴

<details>
<summary><b>Answer</b></summary>

You can run several replicas, but only one is active. They compete for a `Lease` object in `kube-system` (`kube-scheduler`, `kube-controller-manager`); the holder renews it every few seconds, and if it stops, another replica takes over after the lease duration. This gives HA without split-brain actions.
```bash
kubectl -n kube-system get lease kube-scheduler -o yaml
```

</details>

### Q22. How does Raft quorum affect etcd cluster sizing? 🔴

<details>
<summary><b>Answer</b></summary>

A write needs a majority: 2 of 3, 3 of 5. A 3-member cluster tolerates 1 failure, 5 tolerates 2; even sizes add no tolerance (4 members still tolerate 1). More members make writes slower because every write needs more acknowledgements, so 3 is the norm, 5 for large or high-availability-critical clusters across 3 zones.

</details>

### Q23. Why is etcd disk latency so critical, and how do you check it? 🔴

<details>
<summary><b>Answer</b></summary>

Raft commits require an fsync to the WAL; slow disks raise commit latency, trigger leader elections and make the API server time out. We run etcd on dedicated low-latency SSD/NVMe. Watch `etcd_disk_wal_fsync_duration_seconds` (p99 should stay under about 10 ms) and `etcd_server_leader_changes_seen_total`.
```bash
etcdctl endpoint status -w table
etcdctl check perf
```

</details>

### Q24. What is API Priority and Fairness (APF)? 🔴

<details>
<summary><b>Answer</b></summary>

APF replaced the old max-in-flight limits with a flow-control system: requests are classified by `FlowSchema` into `PriorityLevelConfiguration` queues, so one noisy client cannot starve critical traffic (such as node heartbeats or leader election). When overloaded the API server returns `429` with `Retry-After`. Metrics: `apiserver_flowcontrol_*`.

</details>

### Q25. How do API aggregation and the extension API server differ from CRDs? 🔴

<details>
<summary><b>Answer</b></summary>

A CRD stores custom resources in etcd and the main API server serves them with generic logic. The **aggregation layer** proxies an API group to your own server (like metrics-server's `metrics.k8s.io`), which lets you implement custom storage, subresources or non-CRUD semantics. CRDs are simpler and cover most needs.

</details>

### Q26. Describe the scheduler framework and its extension points. 🔴

<details>
<summary><b>Answer</b></summary>

The scheduler runs a pipeline of plugins: QueueSort, PreFilter, Filter, PostFilter (preemption), PreScore, Score, Reserve, Permit, PreBind, Bind and PostBind. Built-in features (NodeResourcesFit, TaintToleration, PodTopologySpread, VolumeBinding) are plugins you can enable, disable or weight through a `KubeSchedulerConfiguration` and multiple scheduling profiles.

</details>

### Q27. What is the difference between the `admission` and `authorization` phases, and where do ValidatingAdmissionPolicy and webhooks fit? 🔴

<details>
<summary><b>Answer</b></summary>

Authorization asks "may this identity perform this verb on this resource?" Admission asks "is this specific object acceptable or does it need defaults?" Webhooks call external servers (latency and availability risk, so set `failurePolicy` and `timeoutSeconds` carefully). `ValidatingAdmissionPolicy` (GA in 1.30) evaluates CEL expressions in-process with no webhook server.

</details>

## Pods & Workloads

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q28. What is a Pod? 🟢

<details>
<summary><b>Answer</b></summary>

The smallest deployable unit in Kubernetes: one or more containers that share a network namespace (one IP, shared localhost) and can share volumes. Containers in a Pod are always scheduled together on the same node. Pods are ephemeral; you rarely create them directly.
```bash
kubectl run web --image=nginx:1.27 --port=80
```

</details>

### Q29. Why would you run more than one container in a Pod? 🟢

<details>
<summary><b>Answer</b></summary>

When containers are tightly coupled and must share lifecycle, network or files: a log shipper sidecar, a service-mesh proxy, a config reloader. Do not put a web app and its database in one Pod, because they scale and fail independently.

</details>

### Q30. What is a ReplicaSet? 🟢

<details>
<summary><b>Answer</b></summary>

A controller object that keeps a stated number of identical Pod replicas running, using a label selector and a Pod template. If a Pod dies, it creates another; if there are too many, it deletes extras. You normally use Deployments, which manage ReplicaSets for you.

</details>

### Q31. What is a Deployment and what does it add over a ReplicaSet? 🟢

<details>
<summary><b>Answer</b></summary>

A Deployment manages ReplicaSets to provide declarative **rolling updates and rollbacks** for stateless apps. Changing the Pod template creates a new ReplicaSet and gradually shifts replicas over; the old ReplicaSets are kept (per `revisionHistoryLimit`) so you can roll back.
```bash
kubectl set image deploy/web web=nginx:1.27.1
kubectl rollout status deploy/web
kubectl rollout undo deploy/web
```

</details>

### Q32. What is a DaemonSet and when do you use one? 🟢

<details>
<summary><b>Answer</b></summary>

A DaemonSet ensures one Pod runs on every node (or a selected subset of nodes). Typical uses: log collectors (Fluent Bit), node monitoring agents, CNI and CSI node plugins, kube-proxy. New nodes automatically get the Pod; removed nodes' Pods are garbage-collected.

</details>

### Q33. What is the difference between a Job and a CronJob? 🟢

<details>
<summary><b>Answer</b></summary>

A Job runs Pods to completion (one-off batch work, with `completions`, `parallelism` and `backoffLimit`). A CronJob creates Jobs on a cron schedule. Use `restartPolicy: OnFailure` or `Never` for Job Pods, never `Always`.
```bash
kubectl create cronjob nightly --image=busybox --schedule="0 2 * * *" -- /bin/sh -c "date"
```

</details>

### Q34. What are labels, selectors and annotations? 🟢

<details>
<summary><b>Answer</b></summary>

Labels are key/value pairs used to identify and group objects; selectors query them (`app=web,tier=frontend`) and Services, ReplicaSets and NetworkPolicies depend on them. Annotations hold non-identifying metadata (build info, tool config) and are not used for selection.
```bash
kubectl get pods -l app=web,tier=frontend --show-labels
```

</details>

### Q35. Describe the Pod lifecycle phases and container states. 🟡

<details>
<summary><b>Answer</b></summary>

Pod phases are Pending, Running, Succeeded, Failed and Unknown. Containers have states Waiting (with reasons such as `ContainerCreating`, `ImagePullBackOff`, `CrashLoopBackOff`), Running and Terminated (with exit code and reason). Pod `conditions` (PodScheduled, Initialized, ContainersReady, Ready) tell you finer detail.
```bash
kubectl describe pod web-0 | sed -n '/Conditions/,/Volumes/p'
```

</details>

### Q36. What are init containers and how do they differ from sidecars? 🟡

<details>
<summary><b>Answer</b></summary>

Init containers run sequentially to completion before app containers start, useful for waiting on dependencies, running migrations or fetching config. Native **sidecar containers** (GA in 1.33) are init containers with `restartPolicy: Always`: they start before the app, keep running, and are shut down after the app, which fixes ordering problems for Jobs and mesh proxies.
```yaml
initContainers:
- name: proxy
  image: envoy:v1.31
  restartPolicy: Always
```

</details>

### Q37. What is `restartPolicy` and how does CrashLoopBackOff happen? 🟡

<details>
<summary><b>Answer</b></summary>

`restartPolicy` (Always, OnFailure, Never) applies to all containers in the Pod, and the kubelet restarts them locally with exponential back-off (10s, 20s, 40s ... capped at 5 minutes). `CrashLoopBackOff` means the container keeps exiting and the kubelet is waiting before the next attempt. Check `kubectl logs --previous` and the exit code.

</details>

### Q38. Explain Deployment rolling update parameters `maxSurge` and `maxUnavailable`. 🟡

<details>
<summary><b>Answer</b></summary>

`maxSurge` is how many extra Pods above `replicas` may exist during a rollout, `maxUnavailable` is how many may be unavailable. Both accept numbers or percentages; defaults are 25% each. With `maxSurge: 1, maxUnavailable: 0` you never drop capacity but need headroom for one extra Pod.
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
```

</details>

### Q39. What is the difference between `RollingUpdate` and `Recreate` strategies? 🟡

<details>
<summary><b>Answer</b></summary>

`RollingUpdate` replaces Pods gradually, keeping the app available. `Recreate` kills all old Pods first, then starts new ones, causing downtime. Use `Recreate` when two versions cannot run at once, for example a singleton app with an exclusive ReadWriteOnce volume or an incompatible schema change.

</details>

### Q40. How does a rollback work in a Deployment? 🟡

<details>
<summary><b>Answer</b></summary>

`kubectl rollout undo` sets the Pod template back to the previous ReplicaSet's template, which scales up again while the current one scales down. History comes from `revisionHistoryLimit` (default 10). Only template changes create revisions; scaling does not. Use `--to-revision=N` for a specific one.
```bash
kubectl rollout history deploy/web
kubectl rollout undo deploy/web --to-revision=3
```

</details>

### Q41. When does a Deployment rollout get stuck and what is `progressDeadlineSeconds`? 🟡

<details>
<summary><b>Answer</b></summary>

If new Pods never become Ready (bad image, failing readiness probe, quota, no capacity), the rollout stalls. After `progressDeadlineSeconds` (default 600) the Deployment gets condition `Progressing=False, reason=ProgressDeadlineExceeded`. Kubernetes does **not** roll back automatically; `kubectl rollout status` exits non-zero so CI can undo.

</details>

### Q42. How do StatefulSets differ from Deployments? 🟡

<details>
<summary><b>Answer</b></summary>

StatefulSets give each Pod a stable name (`db-0`, `db-1`), stable network identity via a headless Service, and its own PVC from `volumeClaimTemplates`. They create and terminate Pods in order (by default) and keep volumes when scaled down. Use them for databases, Kafka, ZooKeeper and other clustered stateful systems.

</details>

### Q43. What does `podManagementPolicy: Parallel` change in a StatefulSet? 🟡

<details>
<summary><b>Answer</b></summary>

The default `OrderedReady` starts Pod N only when N-1 is Running and Ready, and terminates in reverse order. `Parallel` launches or deletes all Pods at once, ignoring ordering. It speeds up scale operations for systems that do not need ordered startup, but it does not change the update strategy.

</details>

### Q44. How do StatefulSet updates work (`RollingUpdate` with `partition`)? 🟡

<details>
<summary><b>Answer</b></summary>

Pods are updated in reverse ordinal order, one at a time, waiting for each to be Ready. `rollingUpdate.partition: N` updates only Pods with ordinal >= N, which lets you canary on `db-2` and then lower the partition. `OnDelete` updates a Pod only when you delete it yourself.

</details>

### Q45. How do you control DaemonSet updates and run it on only some nodes? 🟡

<details>
<summary><b>Answer</b></summary>

Use `updateStrategy.rollingUpdate.maxUnavailable` to limit disruption (default 1), and restrict nodes with `nodeSelector` or node affinity in the Pod template. To run on tainted nodes (control plane, GPU), add tolerations because DaemonSet Pods do not ignore taints except for a few built-in ones.

</details>

### Q46. Explain Job parameters: `completions`, `parallelism`, `backoffLimit`, `activeDeadlineSeconds`, `ttlSecondsAfterFinished`. 🟡

<details>
<summary><b>Answer</b></summary>

`completions` is how many successful Pods are needed; `parallelism` is how many run at once; `backoffLimit` caps retries (default 6) before the Job fails; `activeDeadlineSeconds` is a hard time limit; `ttlSecondsAfterFinished` auto-deletes the finished Job and its Pods. Indexed Jobs (`completionMode: Indexed`) give each Pod a `JOB_COMPLETION_INDEX`.

</details>

### Q47. Explain CronJob `concurrencyPolicy`, `startingDeadlineSeconds` and `timeZone`. 🟡

<details>
<summary><b>Answer</b></summary>

`concurrencyPolicy` is Allow (default), Forbid (skip if the previous run is still going) or Replace. `startingDeadlineSeconds` is how late a missed run may still start; if more than 100 consecutive schedules are missed the controller stops scheduling. `spec.timeZone` (GA 1.27) takes an IANA name; without it the controller-manager's zone, usually UTC, is used.

</details>

### Q48. What are ownerReferences and how does garbage collection use them? 🟡

<details>
<summary><b>Answer</b></summary>

Dependent objects (Pods, ReplicaSets) carry `ownerReferences` to their owner. When the owner is deleted, the garbage collector deletes dependents according to the propagation policy: `background` (default), `foreground` or `orphan`. This is why deleting a Deployment removes its ReplicaSets and Pods.
```bash
kubectl delete deploy web --cascade=orphan
```

</details>

### Q49. What are finalizers and why can an object get stuck in Terminating? 🟡

<details>
<summary><b>Answer</b></summary>

Finalizers are keys in `metadata.finalizers` that block actual deletion until a controller finishes cleanup and removes its key. If the controller is gone or broken, the object stays `Terminating` forever (common with namespaces, PVCs and CRs). Fix the controller first; only as a last resort patch out the finalizer.
```bash
kubectl patch pvc data -p '{"metadata":{"finalizers":null}}' --type=merge
```

</details>

### Q50. What exactly happens when a Pod is deleted (graceful termination sequence)? 🔴

<details>
<summary><b>Answer</b></summary>

The API server sets `deletionTimestamp`; the Pod goes Terminating and is removed from Service endpoints. The kubelet runs the `preStop` hook, sends SIGTERM to PID 1 of each container, waits `terminationGracePeriodSeconds` (default 30), then SIGKILL. Endpoint removal and SIGTERM race, which is why apps often need a short `preStop` sleep.

</details>

### Q51. How does the ReplicaSet controller decide which Pod to delete when scaling down? 🔴

<details>
<summary><b>Answer</b></summary>

It ranks Pods: unscheduled before scheduled, Pending before Running, not-Ready before Ready, Pods on nodes with more replicas of the same set first, newer Ready times first, higher restart counts first. This favors removing the least useful Pods and spreading survivors. You can influence it with the `controller.kubernetes.io/pod-deletion-cost` annotation.

</details>

### Q52. Why does changing a Deployment's selector fail, and how are Pods adopted by ReplicaSets? 🔴

<details>
<summary><b>Answer</b></summary>

`spec.selector` is immutable in `apps/v1`. A ReplicaSet adopts any Pod matching its selector that has no controller ownerReference, which is dangerous with overlapping selectors. The Deployment adds a `pod-template-hash` label to separate its ReplicaSets and avoid overlap.

</details>

### Q53. What are in-place Pod resize and ephemeral containers? 🔴

<details>
<summary><b>Answer</b></summary>

Ephemeral containers (GA 1.25) let you inject a debug container into a running Pod without restarting it: `kubectl debug -it pod/web --image=busybox --target=app`. In-place Pod resize (beta in 1.33) lets you change CPU and memory requests/limits on a running Pod via the `resize` subresource, with a `resizePolicy` per container saying whether a restart is needed.

</details>

### Q54. How do PodDisruptionBudgets interact with voluntary evictions, and what do they not protect against? 🔴

<details>
<summary><b>Answer</b></summary>

A PDB (`minAvailable` or `maxUnavailable`) limits **voluntary** disruptions via the Eviction API: node drains, cluster autoscaler scale-down, Karpenter consolidation. It does not protect against node crashes, OOM kills or `kubectl delete pod`. A PDB that allows zero disruptions blocks drains indefinitely.
```bash
kubectl get pdb
```

</details>

### Q55. How do StatefulSet PVC retention (`persistentVolumeClaimRetentionPolicy`) and `minReadySeconds` change behavior? 🔴

<details>
<summary><b>Answer</b></summary>

By default PVCs from `volumeClaimTemplates` are kept on scale-down and delete (`Retain`). Setting `whenDeleted` and `whenScaled` to `Delete` (GA 1.32) removes them automatically, which is convenient for caches but risky for databases. `minReadySeconds` makes a Pod count as available only after being Ready that long, slowing rollouts safely.

</details>

### Q56. How do the Pod `activeDeadlineSeconds`, Job `podFailurePolicy` and `backoffLimitPerIndex` help batch reliability? 🔴

<details>
<summary><b>Answer</b></summary>

`podFailurePolicy` lets a Job treat specific exit codes or conditions as non-retriable (`FailJob`) or ignorable (`Ignore`, for example node disruptions), instead of burning `backoffLimit`. `backoffLimitPerIndex` in Indexed Jobs retries each index independently, so one bad shard does not fail the whole Job.

</details>

## Services, Networking, Ingress & Gateway API

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q57. What is a Kubernetes Service and why do we need one? 🟢

<details>
<summary><b>Answer</b></summary>

Pod IPs change every time a Pod is replaced, so clients need a stable virtual IP and DNS name. A Service selects Pods by label and gives them a stable ClusterIP; kube-proxy (or eBPF) load-balances connections across ready endpoints.
```bash
kubectl expose deploy web --port=80 --target-port=8080
kubectl get endpointslices -l kubernetes.io/service-name=web
```

</details>

### Q58. What are the Service types? 🟢

<details>
<summary><b>Answer</b></summary>

`ClusterIP` (default, internal only), `NodePort` (opens a port 30000-32767 on every node), `LoadBalancer` (asks the cloud for an external LB, built on NodePort), and `ExternalName` (a DNS CNAME to an outside name). A headless Service (`clusterIP: None`) is a special form that returns Pod IPs directly.

</details>

### Q59. What is the difference between `port`, `targetPort` and `nodePort`? 🟢

<details>
<summary><b>Answer</b></summary>

`port` is the port the Service listens on; `targetPort` is the container port traffic is forwarded to (number or named port); `nodePort` is the port opened on each node for NodePort/LoadBalancer Services. Mixing up `targetPort` with the app's real port is the classic cause of "connection refused".

</details>

### Q60. How does DNS work inside a cluster? 🟢

<details>
<summary><b>Answer</b></summary>

CoreDNS runs as a Deployment in `kube-system` and serves the `cluster.local` zone. A Service `web` in namespace `shop` resolves as `web.shop.svc.cluster.local`; within the same namespace plain `web` works thanks to the search list in the Pod's `/etc/resolv.conf`. Pods get `ndots:5` by default.
```bash
kubectl exec -it deploy/web -- cat /etc/resolv.conf
kubectl run -it --rm dnsutils --image=registry.k8s.io/e2e-test-images/agnhost:2.47 -- nslookup kubernetes.default
```

</details>

### Q61. What is a headless Service and when do we use it? 🟢

<details>
<summary><b>Answer</b></summary>

A Service with `clusterIP: None`: no virtual IP, and DNS returns the individual Pod IPs (A records). StatefulSets use it to give each Pod a stable DNS name such as `db-0.db.prod.svc`. Clients that need to pick a specific replica (databases, Kafka brokers) use it.

</details>

### Q62. What is an Ingress? 🟢

<details>
<summary><b>Answer</b></summary>

An API object that defines HTTP/HTTPS routing rules (host and path) to Services. It does nothing on its own: an **Ingress controller** (NGINX, AWS Load Balancer Controller, Traefik, HAProxy) watches Ingresses and configures a proxy or cloud load balancer. TLS is configured with a Secret referenced in `spec.tls`.

</details>

### Q63. What is the difference between Ingress and Service type LoadBalancer? 🟢

<details>
<summary><b>Answer</b></summary>

A `LoadBalancer` Service creates one L4 cloud load balancer per Service, which becomes expensive and has no path/host routing. One Ingress (or Gateway) with one load balancer can route many hostnames and paths to many Services at L7, with TLS termination in one place.

</details>

### Q64. What does `kubectl port-forward` do? 🟢

<details>
<summary><b>Answer</b></summary>

It opens a tunnel from your laptop through the API server to a Pod or Service port, for debugging without exposing anything. It is not for production traffic.
```bash
kubectl port-forward svc/web 8080:80
curl localhost:8080
```

</details>

### Q65. Explain how kube-proxy implements a ClusterIP in iptables mode. 🟡

<details>
<summary><b>Answer</b></summary>

kube-proxy writes NAT rules: a `KUBE-SERVICES` chain matches the ClusterIP:port and jumps to a per-Service chain that randomly picks a per-endpoint DNAT rule using `statistic` probabilities. It is connection-based (conntrack keeps flows pinned), not per-request. Rule count grows linearly with Services, so very large clusters prefer IPVS, nftables or eBPF.
```bash
sudo iptables -t nat -L KUBE-SERVICES -n | head
```

</details>

### Q66. What are EndpointSlices and why replaced Endpoints? 🟡

<details>
<summary><b>Answer</b></summary>

A single Endpoints object listing every Pod IP became huge and was resent in full on every change. EndpointSlices shard endpoints (100 per slice by default) with per-endpoint conditions (ready, serving, terminating) and topology hints, drastically reducing API and kube-proxy load in big Services.

</details>

### Q67. What is the Kubernetes network model? 🟡

<details>
<summary><b>Answer</b></summary>

Every Pod gets its own IP; all Pods can reach all other Pods without NAT; nodes can reach Pods without NAT. The CNI plugin implements this, via overlay (VXLAN/Geneve), routed (BGP), or VPC-native IPs (AWS VPC CNI, which assigns real VPC addresses from ENIs).

</details>

### Q68. What is `externalTrafficPolicy: Local` and what are the trade-offs? 🟡

<details>
<summary><b>Answer</b></summary>

With `Cluster` (default) traffic may hop to a Pod on another node and be SNATed, hiding the client IP. With `Local`, only Pods on the receiving node serve it, so the source IP is preserved and there is no extra hop, but load can be uneven and nodes without Pods fail the LB health check. `internalTrafficPolicy: Local` does the same for in-cluster traffic.

</details>

### Q69. How does session affinity work on a Service? 🟡

<details>
<summary><b>Answer</b></summary>

`sessionAffinity: ClientIP` pins a client IP to one Pod for `timeoutSeconds` (default 10800). It works at L4 only and breaks behind a NAT where many users share one IP. For real stickiness use cookie-based affinity at the Ingress or Gateway layer.

</details>

### Q70. What are Ingress `pathType` values and how does matching work? 🟡

<details>
<summary><b>Answer</b></summary>

`Exact` matches the path exactly; `Prefix` matches by path segments (`/api` matches `/api/v1` but not `/apiv1`); `ImplementationSpecific` leaves it to the controller. `ingressClassName` selects the controller; the old `kubernetes.io/ingress.class` annotation is deprecated.
```yaml
spec:
  ingressClassName: nginx
  rules:
  - host: shop.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend: { service: { name: api, port: { number: 80 } } }
```

</details>

### Q71. What is the Gateway API and how does it improve on Ingress? 🟡

<details>
<summary><b>Answer</b></summary>

Gateway API (GA v1 since 1.0 for core resources) splits roles: `GatewayClass` (infrastructure provider), `Gateway` (listeners, owned by platform team) and `HTTPRoute`/`GRPCRoute`/`TLSRoute` (owned by app teams). It supports header matching, weighted traffic splitting, redirects, cross-namespace references with `ReferenceGrant`, and is portable, without controller-specific annotations.

</details>

### Q72. Show a Gateway API HTTPRoute with a weighted canary. 🟡

<details>
<summary><b>Answer</b></summary>

Two backendRefs with weights split traffic by ratio; this is native, no annotations needed.
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: web }
spec:
  parentRefs: [{ name: public-gw }]
  hostnames: ["shop.example.com"]
  rules:
  - backendRefs:
    - { name: web-v1, port: 80, weight: 90 }
    - { name: web-v2, port: 80, weight: 10 }
```

</details>

### Q73. How does a NetworkPolicy work and what does it require? 🟡

<details>
<summary><b>Answer</b></summary>

A NetworkPolicy selects Pods (`podSelector`) and lists allowed ingress/egress peers by Pod, namespace or CIDR. Pods are open until selected by at least one policy for that direction, after which only allowed traffic passes (allow-lists are additive). It is enforced by the CNI (Calico, Cilium, Antrea); plain Flannel ignores it silently.

</details>

### Q74. What is the difference between the Pod CIDR, Service CIDR and node network? 🟡

<details>
<summary><b>Answer</b></summary>

Pod CIDR is the address range for Pod IPs (often a /24 per node on overlay CNIs); Service CIDR is the virtual range for ClusterIPs (`--service-cluster-ip-range`), which exists only in rules; the node network is the real VPC/LAN. They must not overlap with each other or with networks you peer or VPN into.

</details>

### Q75. How do Pods resolve external names and what is the `ndots:5` penalty? 🟡

<details>
<summary><b>Answer</b></summary>

With `ndots:5`, names with fewer than 5 dots are first tried with each search domain (`svc.cluster.local`, etc.) before the absolute name, producing several failed lookups for names like `api.example.com`. Add a trailing dot (`api.example.com.`), lower `ndots` via `dnsConfig`, or use NodeLocal DNSCache to reduce load.

</details>

### Q76. What is the purpose of `hostNetwork`, `hostPort` and `hostPID`? 🟡

<details>
<summary><b>Answer</b></summary>

`hostNetwork: true` puts the Pod in the node's network namespace (used by CNI agents, node exporters). `hostPort` maps a container port to a node port without a Service. `hostPID` shares the node's process namespace. All three weaken isolation and are blocked by the Baseline/Restricted Pod Security Standards.

</details>

### Q77. What happens in detail when a Pod calls `curl http://web` for a ClusterIP Service? 🔴

<details>
<summary><b>Answer</b></summary>

The Pod's resolver asks CoreDNS (the `kube-dns` ClusterIP) which returns the Service ClusterIP; the packet leaves the Pod's veth to the node; iptables/IPVS/eBPF DNATs the ClusterIP to a ready endpoint Pod IP; the CNI routes it to that Pod (same node or across nodes); replies are un-NATed by conntrack so the client sees the ClusterIP.

</details>

### Q78. How does the AWS VPC CNI allocate Pod IPs and what limits Pod density? 🔴

<details>
<summary><b>Answer</b></summary>

The `aws-node` DaemonSet attaches ENIs and assigns secondary private IPs to Pods from the node's subnet. Max Pods = ENIs x (IPs per ENI - 1) + 2, so small instance types hit IP limits quickly. Prefix delegation (`ENABLE_PREFIX_DELEGATION=true`) assigns /28 prefixes per ENI slot, and custom networking or IPv6 clusters relieve subnet exhaustion.

</details>

### Q79. Compare iptables, IPVS and nftables kube-proxy modes and eBPF replacement. 🔴

<details>
<summary><b>Answer</b></summary>

iptables is default, O(n) rule traversal and full-table rewrites; IPVS uses kernel hash tables with more schedulers (rr, lc, sh) and scales better; nftables mode (GA in 1.33) is the modern successor to iptables with faster updates. Cilium's kube-proxy replacement uses eBPF maps for O(1) service lookup and also enables DSR and Maglev hashing.

</details>

### Q80. How do topology-aware routing and `trafficDistribution` reduce cross-zone cost? 🔴

<details>
<summary><b>Answer</b></summary>

Setting `spec.trafficDistribution: PreferClose` (GA in 1.33) makes the data plane prefer endpoints in the client's zone, falling back elsewhere. This cuts cross-AZ data transfer charges and latency, but can unbalance load if zones have unequal replica counts, so combine it with topology spread constraints.

</details>

### Q81. How does a `LoadBalancer` Service on EKS get provisioned, and what are the target modes? 🔴

<details>
<summary><b>Answer</b></summary>

With the AWS Load Balancer Controller, a Service with `loadBalancerClass: service.k8s.aws/nlb` creates an NLB. In `instance` mode it targets NodePorts; in `ip` mode (annotation `service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip`) it registers Pod IPs directly, skipping an extra hop, and requires VPC-routable Pod IPs. Without the controller, the legacy in-tree controller makes a Classic LB.

</details>

### Q82. Why does a NetworkPolicy with `egress` rules often break DNS, and how do you fix it? 🔴

<details>
<summary><b>Answer</b></summary>

Once egress is restricted, the Pod can no longer reach CoreDNS on port 53 UDP/TCP. Add an explicit rule allowing egress to the `kube-system` namespace's `k8s-app=kube-dns` Pods.
```yaml
egress:
- to:
  - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } }
    podSelector: { matchLabels: { k8s-app: kube-dns } }
  ports: [{ protocol: UDP, port: 53 }, { protocol: TCP, port: 53 }]
```

</details>

### Q83. What is a service mesh doing at the network layer compared to plain Kubernetes Services? 🔴

<details>
<summary><b>Answer</b></summary>

A mesh (Istio, Linkerd, Cilium) adds L7 features per Pod: mTLS between workloads, retries, timeouts, circuit breaking, traffic splitting and request-level telemetry. Classic meshes inject sidecar proxies; Istio ambient mode uses per-node ztunnel plus optional waypoint proxies to reduce per-Pod overhead. Plain Services only balance L4 connections.

</details>

## Configuration & Storage

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q84. What is a ConfigMap and how can a Pod consume it? 🟢

<details>
<summary><b>Answer</b></summary>

A ConfigMap stores non-confidential key/value config. Pods consume it as environment variables (`env.valueFrom.configMapKeyRef` or `envFrom`), as files via a volume mount, or via the API. Volume-mounted ConfigMaps update eventually; environment variables never refresh until the Pod restarts.
```bash
kubectl create configmap app-config --from-literal=LOG_LEVEL=info --from-file=app.properties
```

</details>

### Q85. What is a Secret and is it secure by default? 🟢

<details>
<summary><b>Answer</b></summary>

A Secret holds sensitive data (passwords, tokens, TLS keys) separate from Pod specs. By default values are only **base64-encoded**, not encrypted, in etcd. Real protection needs encryption at rest (KMS provider), tight RBAC on `get/list secrets`, and ideally an external secrets manager.
```bash
kubectl create secret generic db --from-literal=password='S3cr3t!'
kubectl get secret db -o jsonpath='{.data.password}' | base64 -d
```

</details>

### Q86. What are the common Secret types? 🟢

<details>
<summary><b>Answer</b></summary>

`Opaque` (generic), `kubernetes.io/tls` (tls.crt, tls.key), `kubernetes.io/dockerconfigjson` (image pull credentials), `kubernetes.io/service-account-token`, `kubernetes.io/basic-auth` and `kubernetes.io/ssh-auth`. The type enforces required keys.

</details>

### Q87. What is a Volume and how does it differ from a container filesystem? 🟢

<details>
<summary><b>Answer</b></summary>

A container's writable layer disappears when the container restarts. A volume is defined at Pod level, mounted into containers, and survives container restarts (and, depending on type, Pod deletion). Types include `emptyDir`, `configMap`, `secret`, `projected`, `hostPath`, and PVC-backed volumes.

</details>

### Q88. What is `emptyDir` and when is it used? 🟢

<details>
<summary><b>Answer</b></summary>

An empty directory created when the Pod is scheduled and deleted with the Pod. It is shared by all containers in the Pod, so it suits scratch space, caches and sidecar hand-offs. `medium: Memory` backs it with tmpfs (counts against the memory limit); set `sizeLimit` to bound it.

</details>

### Q89. What are PersistentVolume (PV), PersistentVolumeClaim (PVC) and StorageClass? 🟢

<details>
<summary><b>Answer</b></summary>

A PV is a piece of storage in the cluster; a PVC is a user's request for storage (size, access mode, class); a StorageClass describes a "type" of storage and its provisioner for **dynamic provisioning**. A PVC binds to a PV, and Pods reference the PVC.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3
  resources: { requests: { storage: 20Gi } }
```

</details>

### Q90. What are the PV access modes? 🟢

<details>
<summary><b>Answer</b></summary>

`ReadWriteOnce` (one node read-write), `ReadOnlyMany`, `ReadWriteMany` (many nodes read-write), and `ReadWriteOncePod` (one Pod only, GA 1.29). EBS volumes are RWO; for RWX on AWS you need EFS or FSx. Note RWO means one **node**, so multiple Pods on the same node can still share it.

</details>

### Q91. Explain PV reclaim policies. 🟡

<details>
<summary><b>Answer</b></summary>

`Delete` removes the backing storage when the PVC is deleted (default for dynamically provisioned volumes); `Retain` keeps the PV and data for manual recovery, leaving it `Released` and not reusable until an admin clears the claimRef. `Recycle` is deprecated.
```bash
kubectl patch pv pvc-123 -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

</details>

### Q92. What does `volumeBindingMode: WaitForFirstConsumer` do and why does it matter? 🟡

<details>
<summary><b>Answer</b></summary>

With `Immediate`, the volume is provisioned as soon as the PVC is created, possibly in the wrong availability zone for where the Pod later schedules. `WaitForFirstConsumer` delays provisioning until a Pod is scheduled, so the volume is created in the Pod's zone. For zonal storage like EBS this is the right default.

</details>

### Q93. How does the CSI architecture work? 🟡

<details>
<summary><b>Answer</b></summary>

A CSI driver has a **controller** component (create, delete, attach, snapshot, resize; runs as a Deployment with sidecars such as external-provisioner and external-attacher) and a **node** component (a DaemonSet that stages and mounts volumes for the kubelet). Kubernetes talks to it over gRPC through a socket, replacing in-tree plugins.

</details>

### Q94. How do you expand a PVC? 🟡

<details>
<summary><b>Answer</b></summary>

The StorageClass must set `allowVolumeExpansion: true`; edit the PVC's `spec.resources.requests.storage` to a larger value; the CSI driver resizes the volume, and the filesystem is grown online (or at next mount). Shrinking is not supported.
```bash
kubectl patch pvc data -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
kubectl get pvc data -w
```

</details>

### Q95. What are VolumeSnapshots and how are they used? 🟡

<details>
<summary><b>Answer</b></summary>

CSI snapshot APIs: `VolumeSnapshot`, `VolumeSnapshotContent` and `VolumeSnapshotClass`, installed with the snapshot controller and CRDs. You snapshot a PVC and later create a new PVC with `dataSource` pointing to the snapshot, to clone or restore data. Snapshots are crash-consistent unless the app is quiesced.

</details>

### Q96. How are ConfigMap and Secret volume updates propagated, and what is `subPath` caveat? 🟡

<details>
<summary><b>Answer</b></summary>

The kubelet syncs mounted ConfigMaps/Secrets periodically (default up to about a minute) via atomic symlink swaps, so apps must re-read files or support reload. Files mounted with `subPath`, and env vars, are **never** updated. Marking them `immutable: true` stops updates and reduces API server watch load.

</details>

### Q97. What is a projected volume? 🟡

<details>
<summary><b>Answer</b></summary>

It merges several sources (Secret, ConfigMap, downwardAPI, serviceAccountToken) into one directory. The most common use is a bound, audience-scoped, auto-rotating ServiceAccount token that the kubelet refreshes (default 1 hour expiry), which is how Pods authenticate to the API or to AWS/GCP via workload identity.

</details>

### Q98. What is the Downward API? 🟡

<details>
<summary><b>Answer</b></summary>

It exposes Pod and container metadata to the container without calling the API, as env vars or files: Pod name, namespace, IP, labels, annotations, node name, and resource requests/limits.
```yaml
env:
- name: POD_NAME
  valueFrom: { fieldRef: { fieldPath: metadata.name } }
- name: MEM_LIMIT
  valueFrom: { resourceFieldRef: { containerName: app, resource: limits.memory } }
```

</details>

### Q99. Why is `hostPath` discouraged and what are the safer alternatives? 🟡

<details>
<summary><b>Answer</b></summary>

It mounts a node directory into the Pod, tying it to a node and allowing container escape or host data exposure if it reaches `/` or the container runtime socket. Use PVCs, `emptyDir`, or local PersistentVolumes with node affinity. Legitimate uses are node agents in DaemonSets, ideally read-only.

</details>

### Q100. What are local PersistentVolumes and ephemeral inline volumes? 🟡

<details>
<summary><b>Answer</b></summary>

Local PVs expose a node's disk as a PV with `nodeAffinity` so Pods are scheduled to that node; they give high IOPS but no resilience if the node dies. Generic ephemeral volumes (`ephemeral.volumeClaimTemplate`) create a PVC tied to the Pod's lifetime, which is handy for scratch space on real storage classes.

</details>

### Q101. What is the full lifecycle of dynamic provisioning from PVC creation to mount? 🔴

<details>
<summary><b>Answer</b></summary>

PVC created -> (WaitForFirstConsumer) Pod scheduled -> the external-provisioner sidecar calls CSI `CreateVolume` -> a PV is created and bound to the PVC -> attach-detach controller or external-attacher calls `ControllerPublishVolume` (EBS attach to the node) -> kubelet calls CSI node `NodeStageVolume` (format/mount globally) and `NodePublishVolume` (bind-mount into the Pod) -> containers start.

</details>

### Q102. Why do StatefulSet Pods sometimes get "Multi-Attach error" and "volume node affinity conflict"? 🔴

<details>
<summary><b>Answer</b></summary>

Multi-Attach: an RWO EBS volume is still attached to the old node (the node died or detach is slow) while the new Pod is on another node; it clears after the force-detach timeout (~6 minutes) or when the old node is removed. Node affinity conflict: the PV lives in one AZ but the Pod is constrained or scheduled to another zone.
```bash
kubectl describe pod db-0 | grep -A5 Events
```

</details>

### Q103. How does encryption at rest for Secrets work with a KMS provider? 🔴

<details>
<summary><b>Answer</b></summary>

You configure the API server with an `EncryptionConfiguration`; with KMS v2 (stable in 1.29), each Secret is encrypted by a data encryption key (DEK) that is itself wrapped by a key in an external KMS (AWS KMS, Vault); the DEK rotates automatically. On EKS you enable it with envelope encryption using a KMS key at cluster creation or later. Existing Secrets must be rewritten to be re-encrypted.
```bash
kubectl get secrets -A -o json | kubectl replace -f -
```

</details>

### Q104. How do external secret solutions (External Secrets Operator, Secrets Store CSI Driver) differ? 🔴

<details>
<summary><b>Answer</b></summary>

External Secrets Operator syncs values from AWS Secrets Manager/Vault into native Kubernetes Secrets via `ExternalSecret` CRs, so apps consume normal Secrets (the value still lands in etcd). The Secrets Store CSI Driver mounts secrets as files straight into the Pod from the provider, with optional sync to a Secret, and avoids storing them in etcd if you skip sync.

</details>

### Q105. What are the limits and performance implications of large ConfigMaps/Secrets? 🔴

<details>
<summary><b>Answer</b></summary>

Each is capped at 1 MiB by etcd's object size limits, and they replicate in memory on every node using them and in every kubelet watch. Thousands of watched ConfigMaps or Secrets stress the API server; immutable ones skip watches. Large blobs belong in object storage or an init-container download.

</details>

### Q106. How does `ReadWriteOncePod` differ from `ReadWriteOnce`, and when does it matter? 🔴

<details>
<summary><b>Answer</b></summary>

RWO allows several Pods on the same node to mount a volume; RWOP guarantees only one Pod in the whole cluster can use it, and a second Pod stays Pending. It prevents two replicas corrupting a single-writer filesystem during rolling updates, at the cost of requiring `Recreate`-style handoff.

</details>

### Q107. How do volume snapshots, backups and tools like Velero differ? 🔴

<details>
<summary><b>Answer</b></summary>

A CSI snapshot is storage-level (for EBS, an EBS snapshot) tied to the cloud account and often the same region. Velero backs up Kubernetes objects to object storage and can trigger CSI snapshots or file-system backup (Kopia) of volume data, giving a portable, restorable cluster-level backup with schedules and TTLs.

</details>

## Scheduling, Resources & Autoscaling

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q108. What are resource requests and limits? 🟢

<details>
<summary><b>Answer</b></summary>

A **request** is what the scheduler reserves for the container and the basis for CPU shares; a **limit** is the cap enforced at runtime. CPU over its limit is throttled; memory over its limit gets the container OOMKilled. Requests drive placement, limits drive runtime behavior.
```yaml
resources:
  requests: { cpu: 250m, memory: 256Mi }
  limits:   { memory: 512Mi }
```

</details>

### Q109. What are the three QoS classes? 🟢

<details>
<summary><b>Answer</b></summary>

`Guaranteed` (every container has requests == limits for CPU and memory), `Burstable` (at least one request or limit set, but not Guaranteed), and `BestEffort` (nothing set). Under node memory pressure the kubelet evicts BestEffort first, then Burstable Pods exceeding requests, and Guaranteed last.
```bash
kubectl get pod web-0 -o jsonpath='{.status.qosClass}'
```

</details>

### Q110. What are liveness, readiness and startup probes? 🟢

<details>
<summary><b>Answer</b></summary>

**Liveness** failing makes the kubelet restart the container; **readiness** failing removes the Pod from Service endpoints without restarting; **startup** disables the other two until the app has started, protecting slow-booting apps. Probe types: `httpGet`, `tcpSocket`, `exec` and `grpc`.
```yaml
readinessProbe:
  httpGet: { path: /ready, port: 8080 }
  periodSeconds: 5
  failureThreshold: 3
```

</details>

### Q111. What is a `nodeSelector`? 🟢

<details>
<summary><b>Answer</b></summary>

The simplest scheduling constraint: a map of node labels the Pod requires. The Pod only schedules on nodes that carry all of those labels.
```yaml
spec:
  nodeSelector: { disktype: ssd }
```

</details>

### Q112. What is the difference between taints and tolerations? 🟢

<details>
<summary><b>Answer</b></summary>

A **taint** on a node repels Pods (`key=value:effect`); a **toleration** on a Pod allows it to be scheduled there. Effects are `NoSchedule`, `PreferNoSchedule` and `NoExecute` (also evicts running Pods). Tolerating a taint does not attract the Pod; combine it with a node selector or affinity for dedicated nodes.
```bash
kubectl taint nodes gpu-1 dedicated=gpu:NoSchedule
```

</details>

### Q113. What does the Horizontal Pod Autoscaler (HPA) do? 🟢

<details>
<summary><b>Answer</b></summary>

It adjusts a workload's `replicas` based on metrics, mainly CPU or memory utilization relative to **requests** (from metrics-server), but also custom or external metrics. It needs requests set, and it checks about every 15 seconds.
```bash
kubectl autoscale deploy web --cpu-percent=70 --min=2 --max=10
```

</details>

### Q114. What is the Cluster Autoscaler at a high level? 🟢

<details>
<summary><b>Answer</b></summary>

It adds nodes when Pods are Pending because nothing fits, and removes underused nodes whose Pods can be moved elsewhere. It works on node groups (Auto Scaling Groups on AWS). Karpenter is the alternative that provisions right-sized instances directly without node groups.

</details>

### Q115. Explain the HPA scaling formula and behavior tuning. 🟡

<details>
<summary><b>Answer</b></summary>

`desiredReplicas = ceil(currentReplicas x currentMetric / targetMetric)`, with a 10% tolerance. The `autoscaling/v2` `behavior` field controls scale speed and stabilization (default scale-down stabilization 300 s).
```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
    policies: [{ type: Percent, value: 20, periodSeconds: 60 }]
```

</details>

### Q116. How does the scheduler use requests for placement and what is "allocatable"? 🟡

<details>
<summary><b>Answer</b></summary>

Nodes advertise `capacity`, and `allocatable` equals capacity minus `kube-reserved`, `system-reserved` and eviction thresholds. The scheduler sums the **requests** of Pods on a node and places a new Pod only if its requests fit within allocatable, regardless of real usage. Overcommitted limits are allowed; overcommitted requests are not.

</details>

### Q117. What is the difference between node affinity and `nodeSelector`? 🟡

<details>
<summary><b>Answer</b></summary>

Node affinity is richer: operators (`In`, `NotIn`, `Exists`, `Gt`, `Lt`), and **required** (`requiredDuringSchedulingIgnoredDuringExecution`) vs **preferred** with weights. `nodeSelector` is only exact-match AND. "IgnoredDuringExecution" means running Pods are not moved if labels change later.

</details>

### Q118. How does Pod affinity and anti-affinity work, and what is `topologyKey`? 🟡

<details>
<summary><b>Answer</b></summary>

Pod affinity attracts Pods to nodes (or zones) that already run Pods matching a selector; anti-affinity repels them. `topologyKey` defines the domain: `kubernetes.io/hostname` for per-node, `topology.kubernetes.io/zone` for per-zone. Required anti-affinity is expensive to evaluate at scale; spread constraints are cheaper.

</details>

### Q119. What are topology spread constraints? 🟡

<details>
<summary><b>Answer</b></summary>

They spread matching Pods evenly across domains with a tolerated `maxSkew`, `topologyKey` and `whenUnsatisfiable` (`DoNotSchedule` or `ScheduleAnyway`). They give even distribution that anti-affinity cannot (it is all-or-nothing).
```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector: { matchLabels: { app: web } }
```

</details>

### Q120. What are PriorityClasses and preemption? 🟡

<details>
<summary><b>Answer</b></summary>

A `PriorityClass` assigns an integer priority to Pods. When a high-priority Pod is Pending, the scheduler may **preempt** (evict) lower-priority Pods on a node to make room. Built-in `system-cluster-critical` and `system-node-critical` protect core add-ons. `preemptionPolicy: Never` queues without evicting.

</details>

### Q121. What happens on a container OOMKill versus node-pressure eviction? 🟡

<details>
<summary><b>Answer</b></summary>

OOMKill is the kernel killing a container that exceeded its **cgroup memory limit** (exit code 137, `Reason: OOMKilled`) and the container restarts. Node-pressure eviction is the kubelet proactively evicting whole Pods when node memory, disk or PIDs cross eviction thresholds; the Pod is marked Failed with reason `Evicted` and rescheduled by its controller.

</details>

### Q122. Why can CPU limits cause latency even when the node is idle? 🟡

<details>
<summary><b>Answer</b></summary>

CPU limits are enforced by the CFS quota per 100 ms period: a multithreaded container can burn its quota in a few milliseconds and then be throttled until the next period, regardless of free CPU. Many teams set CPU requests but no CPU limits, and watch `container_cpu_cfs_throttled_periods_total`.

</details>

### Q123. How do liveness probes make things worse if misconfigured? 🟡

<details>
<summary><b>Answer</b></summary>

A liveness probe that checks dependencies (database, downstream API) restarts healthy Pods when the dependency is down, causing cascading failure. Liveness should check only that the process is not deadlocked; use readiness for dependency health, and a startup probe for slow starts so liveness does not kill the app during boot.

</details>

### Q124. What do VPA and its modes do? 🟡

<details>
<summary><b>Answer</b></summary>

The Vertical Pod Autoscaler recommends (and optionally applies) CPU/memory requests from observed usage. Modes: `Off` (recommendations only), `Initial` (set at Pod creation), `Recreate`/`Auto` (evict to apply; in-place updates when supported). Do not let VPA and HPA act on the same CPU/memory metric.

</details>

### Q125. How does the Cluster Autoscaler decide to scale down, and what blocks it? 🔴

<details>
<summary><b>Answer</b></summary>

A node is a candidate when requested utilization is below 50% for 10 minutes (defaults) and all its Pods can be rescheduled. It is blocked by Pods without a controller, local storage (`emptyDir`) unless annotated, restrictive PDBs, anti-affinity that has no other home, and the annotation `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"`.

</details>

### Q126. How does Karpenter provision nodes differently from Cluster Autoscaler? 🔴

<details>
<summary><b>Answer</b></summary>

Karpenter watches unschedulable Pods, batches them, and picks the cheapest fitting instance types (Spot or On-Demand) from `NodePool` constraints and `EC2NodeClass` settings, launching EC2 directly via the Fleet API in under a minute. It also **consolidates** (replaces or removes nodes to cut cost) and handles drift and expiry (`expireAfter`).

</details>

### Q127. What are the scheduler's filter and score steps in detail, with examples of each? 🔴

<details>
<summary><b>Answer</b></summary>

Filters (hard): NodeResourcesFit, NodeAffinity, TaintToleration, VolumeBinding/VolumeZone, PodTopologySpread (DoNotSchedule), InterPodAffinity (required). Scores (soft): LeastAllocated (or MostAllocated for bin-packing), BalancedAllocation, ImageLocality, preferred affinity weights, spread with ScheduleAnyway. Scores are normalized 0-100 and weighted per plugin.

</details>

### Q128. How do HPA custom/external metrics work and what do you need installed? 🔴

<details>
<summary><b>Answer</b></summary>

HPA queries aggregated APIs: `metrics.k8s.io` (metrics-server), `custom.metrics.k8s.io` (via Prometheus Adapter or similar, per-object metrics like requests/sec) and `external.metrics.k8s.io` (queue depth, CloudWatch). KEDA wraps this with event-driven scalers and can scale to zero, which plain HPA cannot (minReplicas >= 1 unless the alpha HPAScaleToZero feature gate is enabled).

</details>

### Q129. Explain what happens with CPU requests under contention (cgroups) versus limits. 🔴

<details>
<summary><b>Answer</b></summary>

Requests become cgroup `cpu.weight` (shares): under contention each container gets CPU proportional to its request, and there is no cap when the node has idle CPU. Limits become `cpu.max` quota. So requests give a guaranteed minimum under contention; limits give a hard ceiling that can throttle even without contention.

</details>

### Q130. How do scheduling gates, DRA and gang scheduling address advanced workloads? 🔴

<details>
<summary><b>Answer</b></summary>

`schedulingGates` hold a Pod from being scheduled until a controller removes the gate (used by quota or batch systems). Dynamic Resource Allocation (GA in 1.34) models devices like GPUs through `ResourceClaim`/`DeviceClass`. Native gang scheduling is not in core yet; Kueue, Volcano and the coscheduling plugin provide all-or-nothing placement for ML jobs.

</details>

### Q131. How does the descheduler complement the scheduler? 🔴

<details>
<summary><b>Answer</b></summary>

The scheduler decides only at Pod creation time and never moves running Pods. The descheduler (a separate project) evicts Pods that violate current policy, such as duplicates on one node, taint violations, spread skew or underused nodes, so the scheduler can place them better. It respects PDBs and should run as a CronJob or Deployment.

</details>

## Security: RBAC, ServiceAccounts & Pod Security

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q132. What is RBAC in Kubernetes? 🟢

<details>
<summary><b>Answer</b></summary>

Role-based access control authorizes API requests with four objects: `Role` and `ClusterRole` (sets of verbs on resources) and `RoleBinding` and `ClusterRoleBinding` (grant them to users, groups or ServiceAccounts). Permissions are purely additive; there are no deny rules.
```bash
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev
kubectl create rolebinding rb --role=pod-reader --user=alice -n dev
```

</details>

### Q133. What is the difference between Role and ClusterRole? 🟢

<details>
<summary><b>Answer</b></summary>

A `Role` is namespaced and grants access to resources in its namespace only. A `ClusterRole` is cluster-wide and can cover cluster-scoped resources (nodes, PVs), non-resource URLs, or be reused across namespaces by binding it with a `RoleBinding`.

</details>

### Q134. How do you check what a user can do? 🟢

<details>
<summary><b>Answer</b></summary>

Use `kubectl auth can-i`, optionally impersonating.
```bash
kubectl auth can-i create deployments -n prod
kubectl auth can-i --list --as=system:serviceaccount:prod:ci -n prod
```

</details>

### Q135. What is a ServiceAccount? 🟢

<details>
<summary><b>Answer</b></summary>

An identity for processes running in Pods. Every namespace has a `default` ServiceAccount; Pods run as it unless `serviceAccountName` is set. Bind RBAC roles to ServiceAccounts to control what a workload may do through the API.
```bash
kubectl create serviceaccount ci -n prod
```

</details>

### Q136. How do users authenticate to Kubernetes? 🟢

<details>
<summary><b>Answer</b></summary>

Kubernetes has no user database. Authenticators include client certificates, bearer tokens (ServiceAccount tokens), OIDC ID tokens (Okta, Entra ID, Google) and webhook auth. On EKS, `aws eks get-token` produces a token validated by the IAM authenticator, mapped to Kubernetes via access entries.

</details>

### Q137. What are the Pod Security Standards? 🟢

<details>
<summary><b>Answer</b></summary>

Three policy levels: **Privileged** (unrestricted), **Baseline** (blocks known privilege escalations such as hostNetwork, privileged containers), and **Restricted** (hardened: non-root, drop ALL capabilities, seccomp RuntimeDefault, no privilege escalation).

</details>

### Q138. What is a `securityContext`? 🟢

<details>
<summary><b>Answer</b></summary>

Pod- and container-level settings for privileges: `runAsUser`, `runAsNonRoot`, `fsGroup`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation`, `capabilities`, and `seccompProfile`.
```yaml
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities: { drop: ["ALL"] }
```

</details>

### Q139. How does Pod Security Admission (PSA) work? 🟡

<details>
<summary><b>Answer</b></summary>

A built-in admission controller enforces the Pod Security Standards per namespace using labels, with three modes: `enforce` (reject), `audit` (log) and `warn` (client warning).
```bash
kubectl label ns shop pod-security.kubernetes.io/enforce=restricted pod-security.kubernetes.io/warn=restricted
```
It replaced PodSecurityPolicy, removed in 1.25. It checks Pods (and warns on controllers), but is not customizable; use Kyverno or Gatekeeper for custom rules.

</details>

### Q140. What changed with ServiceAccount tokens in Kubernetes 1.24 onward? 🟡

<details>
<summary><b>Answer</b></summary>

Pods now receive **bound, time-limited, audience-scoped** tokens via a projected volume instead of long-lived Secret-based tokens, which are no longer auto-created. Tokens are invalidated when the Pod is deleted. For legacy needs you create a Secret with type `kubernetes.io/service-account-token` explicitly, or use `kubectl create token`.

</details>

### Q141. Why set `automountServiceAccountToken: false`? 🟡

<details>
<summary><b>Answer</b></summary>

By default every Pod mounts a token at `/var/run/secrets/kubernetes.io/serviceaccount`, even if it never calls the API. If the app is compromised, the attacker gets that identity. Disable mounting on the ServiceAccount or Pod when the workload does not need the Kubernetes API.

</details>

### Q142. How does IRSA (and EKS Pod Identity) give Pods AWS permissions? 🟡

<details>
<summary><b>Answer</b></summary>

IRSA: the ServiceAccount is annotated with an IAM role ARN; a projected OIDC token is exchanged through `sts:AssumeRoleWithWebIdentity` against the cluster's OIDC provider, so the Pod gets short-lived role credentials. EKS Pod Identity (newer) uses an agent and `pods.eks.amazonaws.com` trust, an association API and no per-cluster OIDC provider setup.
```yaml
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/s3-reader
```

</details>

### Q143. What dangerous RBAC permissions should be treated as cluster-admin equivalent? 🟡

<details>
<summary><b>Answer</b></summary>

`create` on Pods (can mount any ServiceAccount or hostPath), `get/list` on Secrets, `escalate`, `bind` and `impersonate` verbs, `create` on `serviceaccounts/token`, wildcard `*` verbs/resources, and write access to `nodes/proxy`. Use least privilege and audit with `kubectl-who-can` or `rbac-police`.

</details>

### Q144. How do aggregated ClusterRoles (`view`, `edit`, `admin`) work? 🟡

<details>
<summary><b>Answer</b></summary>

Built-in ClusterRoles use `aggregationRule` with label selectors such as `rbac.authorization.k8s.io/aggregate-to-view: "true"`. When you add a CRD, you create a small ClusterRole with that label and its rules are automatically merged into `view`/`edit`/`admin`, so standard roles cover your custom resources.

</details>

### Q145. What is the difference between `system:masters` and `cluster-admin`? 🟡

<details>
<summary><b>Answer</b></summary>

`cluster-admin` is a ClusterRole bound by RBAC; revoke the binding and access is gone. The `system:masters` group is hard-coded to bypass authorization and cannot be revoked via RBAC, so certificates with that group are a permanent backdoor. Avoid issuing client certs with it.

</details>

### Q146. How do NetworkPolicy default-deny and namespace isolation work together? 🟡

<details>
<summary><b>Answer</b></summary>

Create a policy in each namespace selecting all Pods (`podSelector: {}`) with `policyTypes: [Ingress, Egress]` and no rules, which denies everything, then add allow policies for specific flows. Allow rules combine with OR; inside a single `from` element, `namespaceSelector` plus `podSelector` combine with AND.

</details>

### Q147. How are Pod and container capabilities, seccomp and AppArmor combined for defense in depth? 🔴

<details>
<summary><b>Answer</b></summary>

Drop all Linux capabilities and add back only needed ones; use seccomp `RuntimeDefault` (blocks about 40+ risky syscalls) or a custom profile; apply AppArmor or SELinux profiles; run as non-root with a read-only root filesystem; and consider sandboxed runtimes (gVisor, Kata) via `RuntimeClass` for untrusted code. Restricted PSS requires most of these.

</details>

### Q148. How do audit logs work and what policy would you start with? 🔴

<details>
<summary><b>Answer</b></summary>

The API server writes audit events per stage (RequestReceived, ResponseStarted, ResponseComplete) according to an `audit.k8s.io/v1` Policy with levels `None`, `Metadata`, `Request`, `RequestResponse`. A sane start: `Metadata` for most, `RequestResponse` for RBAC and exec/attach, omit noisy `get`s from system users, and never log Secret bodies. On EKS enable the `audit` log type to CloudWatch.

</details>

### Q149. How does Kubernetes handle the certificate hierarchy and what expires? 🔴

<details>
<summary><b>Answer</b></summary>

kubeadm creates a cluster CA, an etcd CA and a front-proxy CA; component certs (apiserver, kubelet client, etcd peers) are valid for 1 year, the CA for 10. Kubelet client/server certs can auto-rotate. Check with `kubeadm certs check-expiration` and renew with `kubeadm certs renew all` (upgrading also renews). Managed services rotate automatically.

</details>

### Q150. What are the risks of the Kubernetes API's `exec`, `proxy` and `portforward` subresources? 🔴

<details>
<summary><b>Answer</b></summary>

`pods/exec` gives a shell inside any container the user can reach, including reading mounted Secrets and tokens; `nodes/proxy` can reach the kubelet API and execute in Pods on that node; `pods/portforward` bypasses NetworkPolicy from the user's side. Grant them rarely and audit them.

</details>

### Q151. Explain admission policy engines: Kyverno vs OPA Gatekeeper vs ValidatingAdmissionPolicy. 🔴

<details>
<summary><b>Answer</b></summary>

Kyverno writes policies as Kubernetes YAML and can validate, mutate, generate and verify image signatures. Gatekeeper uses Rego with `ConstraintTemplate`s and is strong for complex logic. `ValidatingAdmissionPolicy` is built in, runs CEL without a webhook (no extra availability risk), and suits simple validation. Many teams combine PSA + VAP + Kyverno.

</details>

## Extending Kubernetes: Helm, CRDs, Operators & kubectl

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q152. What is Helm and what problem does it solve? 🟢

<details>
<summary><b>Answer</b></summary>

Helm is the package manager for Kubernetes. A **chart** bundles templated manifests plus default `values.yaml`; installing it creates a **release** you can upgrade, roll back and uninstall. It removes copy-pasted YAML across environments.
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install db bitnami/postgresql -n data --create-namespace -f values.yaml
```

</details>

### Q153. What are the basic Helm commands? 🟢

<details>
<summary><b>Answer</b></summary>

`helm install`, `helm upgrade --install`, `helm list`, `helm status`, `helm rollback <rel> <rev>`, `helm uninstall`, `helm template` (render locally), `helm lint`, and `helm get values`.
```bash
helm upgrade --install web ./chart -n shop -f prod.yaml --atomic --wait
helm history web -n shop
```

</details>

### Q154. What is a CustomResourceDefinition (CRD)? 🟢

<details>
<summary><b>Answer</b></summary>

A CRD registers a new resource type (for example `Certificate`, `Prometheus`) with the API server, which then stores and serves it like built-in types, with schema validation (OpenAPI v3), versions and subresources. A CRD alone does nothing; a controller acts on the custom resources.
```bash
kubectl get crd
kubectl explain certificate.spec
```

</details>

### Q155. What is an Operator? 🟢

<details>
<summary><b>Answer</b></summary>

An Operator is a custom controller plus CRDs that encodes operational knowledge for an application: install, scale, back up, upgrade and fail over, for example the Prometheus Operator or a Postgres operator (CloudNativePG). You declare a `Cluster` object and the operator does what a human DBA would do.

</details>

### Q156. What are `kubectl get`, `describe`, `logs` and `exec` used for? 🟢

<details>
<summary><b>Answer</b></summary>

`get` lists objects (`-o wide|yaml|json|jsonpath`), `describe` shows details and recent **events**, `logs` prints container output (`-f`, `--previous`, `-c`), and `exec` runs a command in a container.
```bash
kubectl logs deploy/web -c app --since=10m
kubectl exec -it web-0 -- sh
```

</details>

### Q157. What is a kubeconfig file and context? 🟢

<details>
<summary><b>Answer</b></summary>

`~/.kube/config` holds clusters (server + CA), users (credentials) and **contexts** (cluster + user + namespace). `kubectl config use-context` switches clusters, and `KUBECONFIG` can merge multiple files.
```bash
kubectl config get-contexts
kubectl config set-context --current --namespace=shop
```

</details>

### Q158. How is a Helm chart structured and how does templating work? 🟡

<details>
<summary><b>Answer</b></summary>

A chart has `Chart.yaml`, `values.yaml`, `templates/` (Go templates with Sprig functions), `charts/` for dependencies, and optional `crds/`. Templates reference `.Values`, `.Release`, `.Chart` and helpers from `_helpers.tpl`.
```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
replicas: {{ .Values.replicaCount }}
```

</details>

### Q159. How does Helm store release state, and what does `--atomic` do? 🟡

<details>
<summary><b>Answer</b></summary>

Helm 3 stores each release revision as a Secret (`sh.helm.release.v1.<name>.v<rev>`) in the release namespace; there is no Tiller. `--atomic` makes a failed upgrade roll back automatically (implies `--wait`), and `--wait` blocks until resources are ready, bounded by `--timeout`.

</details>

### Q160. What are Helm hooks and chart tests? 🟡

<details>
<summary><b>Answer</b></summary>

Hooks are annotated resources (`helm.sh/hook: pre-install,pre-upgrade`) run at lifecycle points, for example a DB migration Job, with `hook-delete-policy` for cleanup. Tests are Pods annotated `helm.sh/hook: test`, run by `helm test <release>`. Hook resources are not managed as part of the release afterwards.

</details>

### Q161. Helm vs Kustomize: when do you pick which? 🟡

<details>
<summary><b>Answer</b></summary>

Helm templates a parameterized package with release management; good for distributing third-party software and apps with many knobs. Kustomize overlays plain YAML with patches, no templating, built into `kubectl apply -k`; great for per-environment differences on your own manifests. Many teams use Helm for vendor charts and Kustomize or Helm values for their own, with Argo CD or Flux rendering either.
```bash
kubectl apply -k overlays/prod
```

</details>

### Q162. How does a CRD version conversion and schema validation work? 🟡

<details>
<summary><b>Answer</b></summary>

A CRD lists versions (`v1alpha1`, `v1`) with one marked `storage: true`; the API server converts between them (`None` strategy just changes `apiVersion`, `Webhook` calls a conversion service). Structural OpenAPI schemas with `x-kubernetes-validations` (CEL) validate objects at admission without a webhook.

</details>

### Q163. How does the controller-runtime/kubebuilder pattern work for writing an operator? 🟡

<details>
<summary><b>Answer</b></summary>

You define Go types for the CRD spec/status, generate manifests with `controller-gen`, and write a `Reconcile(ctx, req)` function triggered by watches on your CR and owned objects. Reconcile must be idempotent, use owner references for cleanup, update `status.conditions`, and return `RequeueAfter` or errors for retries. Finalizers handle external cleanup.

</details>

### Q164. What is JSONPath/custom-columns output and `kubectl explain` used for? 🟡

<details>
<summary><b>Answer</b></summary>

They let you extract and learn fields. `kubectl explain deploy.spec.strategy --recursive` documents the schema live from the cluster's OpenAPI.
```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'
```

</details>

### Q165. How do `kubectl debug` and `kubectl diff` help day to day? 🟡

<details>
<summary><b>Answer</b></summary>

`kubectl debug` attaches an ephemeral container, copies a Pod with a different image or command, or starts a privileged Pod on a node (`kubectl debug node/n1 -it --image=busybox`). `kubectl diff -f` shows what an apply would change against live state, ideal in CI before a deploy.

</details>

### Q166. How does Helm three-way merge differ from `kubectl apply` and what causes drift problems? 🔴

<details>
<summary><b>Answer</b></summary>

Helm 3 compares the old manifest, the live state and the new manifest, so manual changes to fields not in the chart survive, but it does not detect edits to fields it manages unless you change them. Helm does not upgrade or delete CRDs in `crds/`, so CRD changes need a separate pipeline.

</details>

### Q167. How would you structure the status and conditions of a custom resource? 🔴

<details>
<summary><b>Answer</b></summary>

Follow the API convention: `status.observedGeneration`, and a `conditions` list of `{type, status, reason, message, lastTransitionTime, observedGeneration}`, with positive-polarity types like `Ready` and `Available`. Users and tooling (`kubectl wait --for=condition=Ready`) rely on it. Keep spec as desired state only and never let controllers write to spec.

</details>

### Q168. What is the difference between webhooks for conversion, admission and the API aggregation approach for extending the API? 🔴

<details>
<summary><b>Answer</b></summary>

Admission webhooks intercept writes to any resource; conversion webhooks translate between CRD versions; the aggregated API server serves a whole API group with custom logic. Webhooks add latency and are a single point of failure if `failurePolicy: Fail`, so exclude `kube-system` and the webhook's own namespace via `namespaceSelector`.

</details>

### Q169. How do `kubectl wait`, `rollout status` and `--dry-run=server` fit into safe automation? 🔴

<details>
<summary><b>Answer</b></summary>

`kubectl wait --for=condition=Available deploy/web --timeout=120s` blocks on conditions; `rollout status` tracks a rollout; `--dry-run=server` sends the request through admission and validation without persisting, catching webhook and schema errors that `--dry-run=client` cannot.
```bash
kubectl apply -f app.yaml --dry-run=server
kubectl wait --for=condition=Ready pod -l app=web --timeout=90s
```

</details>

### Q170. How does Server-Side Apply handle ownership transitions between `kubectl` and controllers such as HPA? 🔴

<details>
<summary><b>Answer</b></summary>

If a manifest sets `replicas` and an HPA also changes it, each apply fights the HPA. With SSA the field manager owning `replicas` is the applier unless you omit the field from your manifest, in which case ownership is released to the HPA. Best practice: omit `spec.replicas` when using an HPA.

</details>

## EKS, Upgrades & etcd Backup

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q171. What is Amazon EKS and what does AWS manage? 🟢

<details>
<summary><b>Answer</b></summary>

EKS is AWS's managed Kubernetes control plane: AWS runs and patches the API servers and etcd across three AZs behind a managed endpoint. You manage the data plane: managed node groups, self-managed nodes, Fargate profiles or EKS Auto Mode, plus add-ons and workloads.
```bash
aws eks update-kubeconfig --name prod --region ap-south-1
```

</details>

### Q172. What are the EKS node options? 🟢

<details>
<summary><b>Answer</b></summary>

Managed node groups (AWS-managed ASGs with AMI lifecycle), self-managed nodes (your own ASG/AMI), Fargate (serverless Pods, no DaemonSets or privileged Pods), Karpenter-provisioned EC2, and EKS Auto Mode, where AWS also manages compute, storage and load balancing components.

</details>

### Q173. What is the EKS API endpoint access setting? 🟢

<details>
<summary><b>Answer</b></summary>

The cluster endpoint can be public, private, or both. Private-only requires access from the VPC (VPN, Direct Connect, bastion). You can also restrict the public endpoint to specific CIDRs. For production we usually run private (or public with tight allowed CIDRs).

</details>

### Q174. What are EKS add-ons? 🟢

<details>
<summary><b>Answer</b></summary>

Managed installs of operational software with AWS-tested versions: `vpc-cni`, `coredns`, `kube-proxy`, `aws-ebs-csi-driver`, `eks-pod-identity-agent`, metrics-server and others. You can upgrade them via `aws eks update-addon` and pin versions.
```bash
aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.33
```

</details>

### Q175. How does kubectl authenticate to EKS? 🟢

<details>
<summary><b>Answer</b></summary>

kubeconfig uses an `exec` credential plugin: `aws eks get-token` signs an STS `GetCallerIdentity` request into a token. EKS validates it and maps the IAM identity to Kubernetes via **access entries** (preferred, API-based) or the legacy `aws-auth` ConfigMap, then RBAC authorizes it.

</details>

### Q176. What is the Kubernetes version skew policy? 🟢

<details>
<summary><b>Answer</b></summary>

`kubectl` may differ by one minor version from the API server; kubelets may be up to **three** minor versions older than the API server (since 1.28); controller-manager and scheduler may be one minor older than the API server. The API server must be upgraded first, never the reverse.

</details>

### Q177. How often does Kubernetes release and how long is a version supported? 🟢

<details>
<summary><b>Answer</b></summary>

About three minor releases a year, each supported upstream for roughly 14 months. On EKS, a version gets 14 months of standard support and 12 more of extended support at an extra per-cluster-hour charge, after which clusters are force-upgraded.

</details>

### Q178. What is `kubectl drain` and `cordon`? 🟢

<details>
<summary><b>Answer</b></summary>

`cordon` marks a node unschedulable without touching its Pods. `drain` cordons and then evicts Pods (respecting PDBs), ready for maintenance. DaemonSet Pods and local data need flags.
```bash
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon node-1
```

</details>

### Q179. How do you view cluster and Pod resource usage? 🟢

<details>
<summary><b>Answer</b></summary>

Install metrics-server and use `kubectl top`.
```bash
kubectl top nodes
kubectl top pods -A --sort-by=memory
```

</details>

### Q180. Explain the order of a Kubernetes cluster upgrade. 🟡

<details>
<summary><b>Answer</b></summary>

Read release notes and check deprecated APIs; back up etcd; upgrade the control plane (one minor version at a time); upgrade add-ons (CNI, CoreDNS, kube-proxy, CSI); upgrade nodes by cordon/drain or by replacing node groups; then validate. With kubeadm: `kubeadm upgrade plan`, `kubeadm upgrade apply v1.xx.y`, then upgrade kubelet per node.

</details>

### Q181. How do you find deprecated or removed API usage before an upgrade? 🟡

<details>
<summary><b>Answer</b></summary>

Check the API server metric `apiserver_requested_deprecated_apis`, audit logs, and scan manifests/Helm charts with tools such as `kubent` (kube-no-trouble), `pluto` or EKS upgrade insights.
```bash
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
aws eks list-insights --cluster-name prod
```

</details>

### Q182. How does an EKS managed node group update work? 🟡

<details>
<summary><b>Answer</b></summary>

It creates new nodes with the new AMI/launch template, cordons and drains old nodes respecting PDBs (up to `maxUnavailable`), then terminates them. If a PDB blocks eviction beyond the timeout the update fails unless you use the force option. Keep the node version within skew of the control plane.
```bash
aws eks update-nodegroup-version --cluster-name prod --nodegroup-name ng-1
```

</details>

### Q183. What is the EKS access entry model and how does it differ from aws-auth? 🟡

<details>
<summary><b>Answer</b></summary>

Access entries are a first-class API (`aws eks create-access-entry`) that map IAM principals to Kubernetes groups or to AWS-managed access policies (`AmazonEKSClusterAdminPolicy`, `AmazonEKSViewPolicy`) scoped to namespaces. They are auditable in CloudTrail and cannot lock you out via a malformed ConfigMap, unlike `aws-auth`.

</details>

### Q184. How does the AWS Load Balancer Controller handle Ingress? 🟡

<details>
<summary><b>Answer</b></summary>

It watches Ingress (class `alb`) and creates an ALB with target groups; in `ip` target type it registers Pod IPs. Annotations configure scheme, certificates and health checks, and `group.name` shares one ALB between Ingresses. It also reconciles NLBs for Services and needs an IAM role (IRSA or Pod Identity).
```yaml
annotations:
  alb.ingress.kubernetes.io/scheme: internet-facing
  alb.ingress.kubernetes.io/target-type: ip
```

</details>

### Q185. What does the EBS CSI driver require on EKS? 🟡

<details>
<summary><b>Answer</b></summary>

The `aws-ebs-csi-driver` add-on plus an IAM role with the EBS permissions (the `AmazonEBSCSIDriverPolicy` managed policy) attached through IRSA or Pod Identity. Without it, PVCs on the `gp3` class stay Pending ("failed to provision volume ... could not create volume in EC2: UnauthorizedOperation"). The legacy `gp2` in-tree class is deprecated.

</details>

### Q186. How do Fargate profiles select Pods, and what are its limits? 🟡

<details>
<summary><b>Answer</b></summary>

A Fargate profile matches namespace and optional labels; matching Pods run in dedicated micro-VMs, each billed by requested vCPU and memory. Limits: no DaemonSets, no privileged or hostNetwork Pods, no GPUs, EBS volumes not supported (EFS is), and Pods need private subnets.

</details>

### Q187. What are the etcd snapshot backup and restore basics? 🟡

<details>
<summary><b>Answer</b></summary>

Take snapshots with `etcdctl snapshot save` using the cluster's certificates, verify with `snapshot status`, and restore into a new data directory with `etcdutl snapshot restore` (etcd 3.6) then point etcd to it.
```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-$(date +%F).db
```

</details>

### Q188. What do `kubeadm init` and `kubeadm join` do? 🟡

<details>
<summary><b>Answer</b></summary>

`kubeadm init` runs preflight checks, creates certificates and kubeconfigs, writes static Pod manifests for control-plane components, starts etcd, uploads config and prints a join command. `kubeadm join` uses a bootstrap token and CA hash to TLS-bootstrap a node's kubelet. You then install a CNI because nodes stay NotReady without one.

</details>

### Q189. How do node problems and NotReady states get handled automatically? 🟡

<details>
<summary><b>Answer</b></summary>

The node controller taints nodes `node.kubernetes.io/not-ready` or `unreachable` with `NoExecute`; Pods tolerate these for 300 seconds by default (`tolerationSeconds`) and are then evicted and recreated elsewhere. Tools like Node Problem Detector and the EKS node monitoring agent surface kernel and runtime issues as conditions.

</details>

### Q190. What is a Kubernetes Lease-based node heartbeat versus node status updates? 🟡

<details>
<summary><b>Answer</b></summary>

Kubelet updates a lightweight Lease every 10 seconds, and only writes the larger Node status when something changes or every 5 minutes. This cut etcd write traffic and made large clusters practical, while still letting the controller detect failure within the grace period.

</details>

### Q191. What is EKS Auto Mode? 🟡

<details>
<summary><b>Answer</b></summary>

A mode where EKS manages the data plane: it provisions and patches nodes (Bottlerocket-based, using Karpenter internally), and runs managed load balancing, block storage and Pod networking capabilities. You define `NodePool`s and `NodeClass`es for constraints and nodes are replaced on a maximum lifetime (21 days). It cuts operations work but limits customization.

</details>

### Q192. How are logs and metrics usually collected from EKS? 🟡

<details>
<summary><b>Answer</b></summary>

Control-plane logs (api, audit, authenticator, controllerManager, scheduler) go to CloudWatch if enabled. For workloads: Fluent Bit DaemonSet or CloudWatch Observability add-on for logs, and Prometheus (managed via Amazon Managed Service for Prometheus or kube-prometheus-stack) with Grafana for metrics.
```bash
aws eks update-cluster-config --name prod --logging '{"clusterLogging":[{"types":["api","audit","authenticator"],"enabled":true}]}'
```

</details>

### Q193. Why can't you snapshot etcd on EKS and what do you back up instead? 🔴

<details>
<summary><b>Answer</b></summary>

AWS owns the control plane and etcd, so no `etcdctl` access exists. You back up cluster state declaratively: manifests in Git (GitOps) and Velero for objects and PV data, plus snapshots for EBS/EFS. AWS restores control plane availability itself; your recovery plan covers workloads, data and accidental deletion.

</details>

### Q194. Walk through restoring a kubeadm etcd cluster from a snapshot. 🔴

<details>
<summary><b>Answer</b></summary>

Stop the API server and etcd static Pods (move the manifests out of `/etc/kubernetes/manifests`); restore the snapshot to a new directory with `etcdutl snapshot restore snap.db --data-dir /var/lib/etcd-restore` (on multi-member clusters pass `--name`, `--initial-cluster`, `--initial-advertise-peer-urls` on each node with the same snapshot); edit the etcd manifest's `hostPath` to the new dir; put the manifests back; verify with `etcdctl endpoint health`. Restore re-creates the membership, so all members must be restored together.

</details>

### Q195. How would you do a zero-downtime node pool replacement during a major upgrade? 🔴

<details>
<summary><b>Answer</b></summary>

Create a new node group with the new AMI/version (blue-green), taint or cordon the old group, drain it gradually with PDBs and `maxUnavailable` honoured, watch rollout metrics, then delete the old group. Rolling in place works for minor patches, while blue-green gives an instant rollback path by uncordoning the old group.

</details>

### Q196. What are the risks of skipping versions or running the kubelet far behind the control plane? 🔴

<details>
<summary><b>Answer</b></summary>

Upstream and EKS only support upgrading one minor version at a time for the control plane. Old kubelets lose support for new API fields and features and may break with removed APIs; a node past skew becomes unsupported. Plan quarterly upgrades; extended support is costly and delays the problem.

</details>

### Q197. How do EKS cluster endpoint, security groups and ENIs fit together? 🔴

<details>
<summary><b>Answer</b></summary>

EKS creates cross-account ENIs in your subnets for the control plane to reach nodes and a cluster security group attached to both. Nodes reach the API through the endpoint (public or private ENIs). Security group rules and subnets (with enough free IPs, at least 2 AZs) are how you control control-plane-to-node communication, and exhausting subnet IPs breaks upgrades.

</details>

### Q198. What are ResourceQuota and LimitRange? 🟡

<details>
<summary><b>Answer</b></summary>

`ResourceQuota` caps aggregate consumption in a namespace (CPU/memory requests and limits, object counts, PVC storage). `LimitRange` sets per-container or per-Pod defaults and min/max, so Pods without requests get sane values (and are accepted under a quota that demands them).
```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: team-a }
spec:
  hard: { requests.cpu: "20", requests.memory: 64Gi, pods: "100" }
```

</details>

### Q199. What are Kubernetes Events and how long are they kept? 🟡

<details>
<summary><b>Answer</b></summary>

Events are short-lived objects (reason, message, involved object) emitted by the kubelet, scheduler and controllers, for example `FailedScheduling` or `BackOff`. They are kept about one hour by default, so ship them to your logging stack if you need history.
```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -20
```

</details>

### Q200. How do you restart all Pods of a Deployment safely? 🟡

<details>
<summary><b>Answer</b></summary>

Use `kubectl rollout restart`, which stamps a new annotation on the Pod template so a normal rolling update happens with `maxSurge`, probes and PDBs honoured. Deleting Pods by hand ignores rollout strategy and can drop capacity.
```bash
kubectl rollout restart deploy/web
kubectl rollout status deploy/web
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
