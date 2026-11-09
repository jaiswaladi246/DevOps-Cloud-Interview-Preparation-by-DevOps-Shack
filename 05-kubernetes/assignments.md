# Kubernetes — Assignments

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

> 25 assignments · 🟢 Beginner (10) · 🟡 Intermediate (10) · 🔴 Advanced (5)

[⬅ Back to Kubernetes](./README.md) · [🏠 Home](../README.md)

Use a local kind or minikube cluster (a multi-node kind cluster for the advanced ones). Do each assignment in its own namespace so you can delete it cleanly. Try before opening the hints.

## Table of contents
- [🟢 Beginner](#-beginner) — A1–A10
- [🟡 Intermediate](#-intermediate) — A11–A20
- [🔴 Advanced](#-advanced) — A21–A25

---

## 🟢 Beginner

### A1. Your first Pod 🟢

**Objective:** Create, inspect and delete a Pod, and read its lifecycle.

**Tasks**
1. Run an `nginx:1.27` Pod named `web` in namespace `a1`.
2. Show its IP, node and status with one command.
3. Read its logs and exec into it to run `nginx -v`.
4. Describe it and find the event that shows the image being pulled.
5. Delete it and confirm it is gone.

**Expected outcome:** You can explain every field in `kubectl describe pod` output, and the Pod does not come back after deletion.

<details>
<summary>Hints</summary>

`kubectl create ns a1`, `kubectl -n a1 run web --image=nginx:1.27`, `kubectl -n a1 get pod web -o wide`. Events are at the bottom of `describe`. A bare Pod is not recreated because nothing owns it.
</details>

**Bonus:** Generate the YAML with `--dry-run=client -o yaml`, add a label, and apply it from a file.

### A2. Deployment and scaling 🟢

**Objective:** Run a Deployment and scale it.

**Tasks**
1. Create a Deployment `hello` with 3 replicas of `nginx:1.27`.
2. Scale it to 5, then to 2 using `kubectl scale`.
3. Delete one Pod and watch the controller replace it.
4. Show the ReplicaSet that owns the Pods.

**Expected outcome:** Pod count always converges to the desired replicas. You can name the owner chain Deployment, ReplicaSet, Pod.

<details>
<summary>Hints</summary>

`kubectl create deploy hello --image=nginx:1.27 --replicas=3`. Use `kubectl get pods -w` in a second terminal. `kubectl get pod <name> -o jsonpath='{.metadata.ownerReferences[0].kind}'`.
</details>

**Bonus:** Make a Pod stop being managed by changing its labels so it no longer matches the selector, then observe what happens.

### A3. Expose with a Service 🟢

**Objective:** Reach Pods through ClusterIP and NodePort Services.

**Tasks**
1. Expose `hello` from A2 as a ClusterIP Service on port 80.
2. From a temporary Pod, `curl` the Service by short name and by FQDN.
3. Change the Service type to NodePort and reach it from your machine.
4. Show the EndpointSlice and match its IPs to the Pod IPs.

**Expected outcome:** Curl succeeds from inside the cluster by DNS name, and endpoint IPs equal the Pod IPs.

<details>
<summary>Hints</summary>

`kubectl expose deploy hello --port=80`. FQDN is `hello.<ns>.svc.cluster.local`. On kind, NodePort is reachable on the node container IP (`docker inspect`) or use `kubectl port-forward`.
</details>

**Bonus:** Remove the label from one Pod and watch it disappear from the EndpointSlice.

### A4. Declarative YAML 🟢

**Objective:** Manage resources from files, not imperative commands.

**Tasks**
1. Write `deploy.yaml` with a Deployment, and `svc.yaml` with a Service.
2. Apply them, then change replicas in the file and re-apply.
3. Use `kubectl diff` before applying a change to the image tag.
4. Delete everything using the same files.

**Expected outcome:** The files are the source of truth; `kubectl diff` shows the change before you apply it.

<details>
<summary>Hints</summary>

`kubectl apply -f .`, `kubectl diff -f deploy.yaml`, `kubectl delete -f .`. Put `---` between documents if you combine them into one file.
</details>

**Bonus:** Validate manifests without a cluster using `kubectl apply --dry-run=server` and then try `kubeconform`.

### A5. ConfigMap as env and file 🟢

**Objective:** Inject configuration two ways and see the difference.

**Tasks**
1. Create a ConfigMap with keys `GREETING` and `app.properties`.
2. Inject `GREETING` as an env var and mount `app.properties` as a file in a busybox Pod.
3. Edit the ConfigMap. Observe which of the two changes inside the running Pod.
4. Restart the Pod and observe again.

**Expected outcome:** The mounted file updates after about a minute, the env var does not change until restart.

<details>
<summary>Hints</summary>

`kubectl create cm app --from-literal=GREETING=hi --from-file=app.properties`. Use `envFrom` or `valueFrom.configMapKeyRef`. `kubectl exec ... cat /etc/app/app.properties`. Mounts with `subPath` never update.
</details>

**Bonus:** Make the ConfigMap immutable and try to edit it.

### A6. Secrets 🟢

**Objective:** Handle Secrets and understand what protection they do and do not give.

**Tasks**
1. Create a Secret `db` with `username` and `password` using `--from-literal`.
2. Decode the password from `kubectl get secret -o yaml`.
3. Mount the Secret as a read-only volume and consume one key as an env var in another Pod.
4. Create a `kubernetes.io/dockerconfigjson` Secret with `kubectl create secret docker-registry` and reference it in `imagePullSecrets`.

**Expected outcome:** You can state that base64 is not encryption and name two mitigations (encryption at rest, external secret manager, tight RBAC).

<details>
<summary>Hints</summary>

`kubectl get secret db -o jsonpath='{.data.password}' | base64 -d`. Volume mounts of Secrets are tmpfs-backed on the node.
</details>

**Bonus:** Write an RBAC Role that allows reading ConfigMaps but not Secrets and prove it with `auth can-i`.

### A7. Namespaces, labels and selectors 🟢

**Objective:** Organize and query resources.

**Tasks**
1. Create namespaces `dev` and `staging` and run the same Deployment in both.
2. Label Pods with `app`, `tier`, `env`.
3. Query with `-l env=dev`, `-l 'tier in (web,api)'` and `--field-selector status.phase=Running`.
4. Set a default namespace in your kubeconfig context and verify with `kubectl config view --minify`.
5. Apply a ResourceQuota to `dev` limiting Pods to 5, then try to exceed it.

**Expected outcome:** The sixth Pod is rejected with `exceeded quota`.

<details>
<summary>Hints</summary>

`kubectl config set-context --current --namespace=dev`. Quota: `kubectl create quota q --hard=pods=5 -n dev`. A Deployment will show the quota error in the ReplicaSet events.
</details>

**Bonus:** Add a LimitRange that sets default requests and limits, then create a Pod without resources and inspect what it got.

### A8. Probes basics 🟢

**Objective:** See what liveness and readiness actually do.

**Tasks**
1. Deploy a Pod whose readiness probe is `exec: cat /tmp/ready` and liveness probe is `httpGet` on port 80.
2. Observe that the Pod is `0/1 Ready` until you `touch /tmp/ready` inside it.
3. Confirm a not-ready Pod is absent from the Service endpoints.
4. Break liveness (point to a wrong path) and watch the restart count and events.

**Expected outcome:** Readiness failure removes the Pod from endpoints without restarting it. Liveness failure restarts the container.

<details>
<summary>Hints</summary>

Look for `Readiness probe failed` and `Liveness probe failed: HTTP probe failed with statuscode: 404` in `kubectl describe pod`. Check `RESTARTS` column.
</details>

**Bonus:** Add a startupProbe and explain why it is better than a long `initialDelaySeconds`.

### A9. Resource requests and limits 🟢

**Objective:** Observe scheduling by requests and enforcement by limits.

**Tasks**
1. Deploy a Pod requesting 100m CPU and 64Mi memory with limits of 200m and 128Mi.
2. Check its QoS class.
3. Run a memory hog (for example `polinux/stress` with `--vm-bytes 200M`) with a 128Mi limit and observe `OOMKilled`.
4. Request more CPU than any node has and read the `Pending` reason.

**Expected outcome:** You see `Last State: Terminated, Reason: OOMKilled, Exit Code: 137` and `Insufficient cpu` in scheduling events.

<details>
<summary>Hints</summary>

`kubectl get pod -o jsonpath='{.status.qosClass}'`. `kubectl describe node` shows allocatable and allocated requests. Command for stress: `stress --vm 1 --vm-bytes 200M --vm-hang 1`.
</details>

**Bonus:** Set requests equal to limits and confirm the QoS class becomes `Guaranteed`.

### A10. Jobs and CronJobs 🟢

**Objective:** Run batch work correctly.

**Tasks**
1. Create a Job that prints the date and exits.
2. Create a Job with `completions: 5` and `parallelism: 2` and watch it run.
3. Create a CronJob that runs every minute with `concurrencyPolicy: Forbid`.
4. Set `ttlSecondsAfterFinished` and `successfulJobsHistoryLimit` so old Jobs clean themselves up.
5. Trigger the CronJob manually.

**Expected outcome:** Completed Pods are retained until TTL, and old Jobs do not accumulate.

<details>
<summary>Hints</summary>

`kubectl create job manual --from=cronjob/<name>`. Set `restartPolicy: OnFailure` or `Never` (Always is invalid for Jobs). `backoffLimit` controls retries.
</details>

**Bonus:** Make a Job fail on purpose and observe backoff delays and the `BackoffLimitExceeded` condition.

---

## 🟡 Intermediate

### A11. Rolling update and rollback 🟡

**Objective:** Control rollout speed and recover from a bad release.

**Tasks**
1. Deploy 4 replicas with `maxSurge: 1` and `maxUnavailable: 0`.
2. Update the image and watch the ReplicaSets trade replicas.
3. Update to a nonexistent tag and observe the stuck rollout.
4. Roll back and view `rollout history`.
5. Set `progressDeadlineSeconds: 60` and confirm the Deployment reports `ProgressDeadlineExceeded`.

**Expected outcome:** Available replicas never drop below 4 during the good rollout, and rollback restores the previous image.

<details>
<summary>Hints</summary>

`kubectl rollout status`, `kubectl rollout undo --to-revision=N`, `kubectl get rs -w`. Check `kubectl get deploy -o jsonpath='{.status.conditions}'`.
</details>

**Bonus:** Use `kubectl rollout pause`, make two changes, then `resume` for a single rollout.

### A12. StatefulSet with persistent storage 🟡

**Objective:** Prove stable identity and storage.

**Tasks**
1. Deploy a 3-replica StatefulSet with a headless Service and a `volumeClaimTemplates` entry of 1Gi.
2. Write the hostname into a file on each replica's volume.
3. Delete `web-1` and confirm it returns with the same name and the same data.
4. Resolve `web-0.<headless-svc>` from another Pod.
5. Scale down to 1 and check that PVCs for scaled-down Pods remain.

**Expected outcome:** Names are `web-0..2`, ordered start and stop, data persists, PVCs are retained after scale-down.

<details>
<summary>Hints</summary>

`kubectl get pvc`, `nslookup web-0.web`. Set `podManagementPolicy: Parallel` to see the difference. Retention behavior is controlled by `persistentVolumeClaimRetentionPolicy`.
</details>

**Bonus:** Expand a PVC from 1Gi to 2Gi (needs `allowVolumeExpansion: true` on the StorageClass).

### A13. Ingress with host and path routing 🟡

**Objective:** Route HTTP traffic to two services.

**Tasks**
1. Install ingress-nginx on a kind cluster with port mappings.
2. Deploy two apps (for example `hashicorp/http-echo` with different text).
3. Route `app.localtest.me/blue` and `app.localtest.me/green` to them.
4. Add a second host `admin.localtest.me` to a third service.
5. Add an annotation to rewrite `/blue` to `/` at the backend.

**Expected outcome:** Each path and host returns the right app, and unknown hosts return the default 404.

<details>
<summary>Hints</summary>

Use `ingressClassName: nginx`. Rewrite needs a capture group: `path: /blue(/|$)(.*)` with `pathType: ImplementationSpecific` and `nginx.ingress.kubernetes.io/rewrite-target: /$2`.
</details>

**Bonus:** Add TLS with a self-signed certificate and force an HTTPS redirect.

### A14. RBAC for a developer 🟡

**Objective:** Grant least privilege and prove it.

**Tasks**
1. Create namespace `team-a` and a ServiceAccount `dev`.
2. Write a Role that can get, list, watch pods and pod logs, and exec into pods, but nothing else.
3. Bind it with a RoleBinding.
4. Prove each allowed and denied action with `kubectl auth can-i --as`.
5. Build a kubeconfig that uses a token from `kubectl create token` and test it.

**Expected outcome:** The user can read logs and exec in `team-a` but cannot read Secrets or touch other namespaces.

<details>
<summary>Hints</summary>

Resources: `pods`, `pods/log`, `pods/exec` (exec needs `create`). `kubectl auth can-i --list --as=system:serviceaccount:team-a:dev -n team-a`.
</details>

**Bonus:** Replace the Role with a ClusterRole bound by RoleBindings in two namespaces and explain why that is cleaner.

### A15. HPA with CPU load 🟡

**Objective:** Autoscale a workload under load.

**Tasks**
1. Install metrics-server (with `--kubelet-insecure-tls` on kind).
2. Deploy a CPU-bound app with a CPU request of 100m.
3. Create an HPA at 50% CPU, min 1, max 8.
4. Generate load and record replica count over time.
5. Stop the load and observe scale-down timing.
6. Tune `behavior.scaleUp` and `behavior.scaleDown` policies.

**Expected outcome:** Replicas rise under load and fall after the stabilization window (default 300 s).

<details>
<summary>Hints</summary>

`kubectl autoscale deploy x --cpu-percent=50 --min=1 --max=8`. `<unknown>` targets mean missing requests or metrics-server problems. Load: `kubectl run -it load --image=busybox -- sh -c 'while :; do wget -qO- http://x; done'`.
</details>

**Bonus:** Add a memory metric and explain why scaling on memory is usually a poor fit.

### A16. NetworkPolicy default deny 🟡

**Objective:** Isolate a namespace and open only what is required.

**Tasks**
1. Use a CNI that enforces policy (Calico or Cilium on kind).
2. Deploy `frontend`, `api`, `db` Pods with labels.
3. Apply a default-deny for ingress and egress.
4. Allow DNS egress, frontend to api, and api to db only.
5. Prove allowed and blocked paths with `nc -zv -w 3`.

**Expected outcome:** Only the two intended paths work. frontend to db times out.

<details>
<summary>Hints</summary>

An empty `podSelector: {}` selects all Pods. Egress to DNS needs UDP and TCP 53 to kube-dns in `kube-system`. A policy applies in both directions only if you allow both the egress on the source and the ingress on the destination.
</details>

**Bonus:** Restrict egress to an external CIDR using `ipBlock` with an `except` range.

### A17. Scheduling with taints, affinity and spread 🟡

**Objective:** Control where Pods land.

**Tasks**
1. Label two kind workers `disk=ssd` and `disk=hdd`.
2. Use `nodeSelector` to pin a Pod to `ssd`.
3. Taint one worker `dedicated=batch:NoSchedule` and deploy a Pod with a matching toleration.
4. Add `podAntiAffinity` so replicas avoid the same node.
5. Use `topologySpreadConstraints` with `maxSkew: 1` over `kubernetes.io/hostname`.

**Expected outcome:** Pods land exactly where the rules say, and an impossible rule produces `Pending` with a clear scheduling reason.

<details>
<summary>Hints</summary>

`kubectl taint nodes <n> dedicated=batch:NoSchedule`. Remove with the trailing `-`. A toleration allows scheduling there; it does not force it, pair it with a nodeSelector or affinity.
</details>

**Bonus:** Compare `requiredDuringScheduling...` with `preferredDuringScheduling...` using 4 replicas and 3 nodes.

### A18. PodDisruptionBudget and node drain 🟡

**Objective:** Keep availability during maintenance.

**Tasks**
1. Run a 3-replica Deployment spread over workers.
2. Create a PDB with `minAvailable: 2`.
3. Drain a worker with `kubectl drain --ignore-daemonsets --delete-emptydir-data`.
4. Make the drain block by setting `minAvailable: 3`, and read the message.
5. Uncordon the node.

**Expected outcome:** Drain evicts Pods one at a time respecting the PDB, and a too-strict PDB blocks it with `Cannot evict pod as it would violate the pod's disruption budget`.

<details>
<summary>Hints</summary>

`kubectl get pdb` shows `ALLOWED DISRUPTIONS`. A PDB with `minAvailable` equal to replicas permanently blocks drains, which is a classic upgrade outage.
</details>

**Bonus:** Use `maxUnavailable: 1` instead and explain why it scales better with replica count.

### A19. Init containers, sidecars and shared volumes 🟡

**Objective:** Compose multi-container Pods.

**Tasks**
1. Write a Pod with an init container that downloads or generates a file into an `emptyDir`.
2. The main container serves that file with nginx.
3. Add a sidecar that tails an access log from a shared volume.
4. Make the init container fail and read the Pod status.
5. Convert the sidecar to a native sidecar (init container with `restartPolicy: Always`).

**Expected outcome:** Main container starts only after init succeeds, a failing init shows `Init:CrashLoopBackOff`, and the native sidecar starts before and stops after the main container.

<details>
<summary>Hints</summary>

`kubectl logs pod -c <container>`. Init logs: `-c <init-name>`. Native sidecars are GA from Kubernetes 1.33; on older clusters the feature gate is on by default since 1.29.
</details>

**Bonus:** Run a Job with a sidecar and explain why native sidecars fix the "Job never completes" problem.

### A20. Debug a broken app 🟡

**Objective:** Diagnose five injected faults using only kubectl.

**Tasks**
Apply a manifest you wrote with these deliberate bugs, then fix each one.
1. A Deployment with an image tag typo.
2. A Service whose selector does not match Pod labels.
3. A Pod with a memory limit too low (`OOMKilled`).
4. A readiness probe on the wrong port.
5. A Pod referencing a ConfigMap key that does not exist.

Write down the symptom, the command that found it, and the fix for each.

**Expected outcome:** All five resolved, each with a one-line root cause.

<details>
<summary>Hints</summary>

Order of attack: `get pods`, `describe pod`, `logs --previous`, `get endpointslices`, `get events --sort-by=.lastTimestamp`. Config errors show `CreateContainerConfigError`.
</details>

**Bonus:** Debug a distroless container with `kubectl debug -it pod/x --image=busybox --target=app`.

---

## 🔴 Advanced

### A21. Production-ready Helm chart 🔴

**Objective:** Package an app with sane defaults and safe upgrades.

**Tasks**
1. Create a chart with `helm create` and strip it to Deployment, Service, Ingress, HPA, PDB and ServiceAccount.
2. Make requests, limits, replicas, probes and ingress host configurable in `values.yaml`.
3. Add `values.schema.json` that rejects a missing image tag.
4. Add a pre-upgrade Job hook for a migration.
5. Run `helm lint`, `helm template | kubeconform`, and `helm upgrade --install --atomic`.
6. Force a failed upgrade and watch `--atomic` roll back.

**Expected outcome:** Bad values fail at lint or install time, a failed upgrade restores the previous release.

<details>
<summary>Hints</summary>

`helm history`, `helm rollback`. Add a checksum annotation of the ConfigMap to the Pod template so config changes roll the Pods: `checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}`.
</details>

**Bonus:** Publish the chart to an OCI registry with `helm push` and install it from there.

### A22. Admission policy and Pod Security 🔴

**Objective:** Enforce guardrails at the API.

**Tasks**
1. Label a namespace with Pod Security `enforce=restricted` and attempt to run a root Pod. Read the rejection.
2. Fix a Deployment to pass `restricted` (non-root, no privilege escalation, drop ALL, seccomp RuntimeDefault).
3. Write a `ValidatingAdmissionPolicy` (CEL) that rejects Deployments without resource limits.
4. Add a binding with `validationActions: [Deny]` scoped to one namespace.
5. Test with a compliant and a non-compliant manifest.

**Expected outcome:** Violations are rejected at apply time with a clear message and compliant workloads run.

<details>
<summary>Hints</summary>

`pod-security.kubernetes.io/enforce=restricted`. Use `warn` and `audit` first in real clusters. ValidatingAdmissionPolicy is GA in 1.30, expression example: `object.spec.template.spec.containers.all(c, has(c.resources.limits))`.
</details>

**Bonus:** Run the same rule in Kyverno or OPA Gatekeeper and compare the authoring experience.

### A23. GitOps with Argo CD or Flux 🔴

**Objective:** Make Git the only way changes reach the cluster.

**Tasks**
1. Install Argo CD (or Flux) on kind.
2. Create a Git repo with app manifests in `apps/` and an Application pointing at it with automated sync, `prune` and `selfHeal`.
3. Change replicas in Git and watch it sync.
4. Change replicas with `kubectl scale` and watch drift get reverted.
5. Break the manifest in Git and use `git revert` to recover.
6. Add a second environment using Kustomize overlays.

**Expected outcome:** Cluster state always converges to Git. Manual drift is reverted within minutes.

<details>
<summary>Hints</summary>

Argo CD initial admin password: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`. For Flux: `flux bootstrap github`. Use `argocd app diff` to see drift.
</details>

**Bonus:** Add Argo CD sync waves so a database migration Job runs before the Deployment.

### A24. Cluster upgrade rehearsal 🔴

**Objective:** Practice a minor version upgrade without downtime.

**Tasks**
1. Create a kubeadm or kind cluster at version N-1 with 2 workers.
2. Deploy an app with PDB, 3 replicas and a load generator measuring errors.
3. Run `kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis` and scan manifests with `pluto` or `kubent` for deprecated APIs.
4. Upgrade the control plane, then each worker one at a time using cordon, drain, upgrade, uncordon.
5. Record error rate and any Pod restarts.
6. Write the rollback plan: what can and cannot be undone (etcd snapshot).

**Expected outcome:** Zero failed requests during the upgrade, version skew policy respected, written runbook.

<details>
<summary>Hints</summary>

kubelet may be up to three minor versions older than the API server (since 1.28) but never newer. Take an etcd snapshot first: `etcdctl snapshot save`. Upgrade one minor version at a time.
</details>

**Bonus:** Repeat the upgrade the managed way on EKS (control plane, then managed node group rolling update) and compare.

### A25. Multi-tenant platform slice 🔴

**Objective:** Give two teams safe, isolated, cost-aware access to one cluster.

**Tasks**
1. Create namespaces `team-a` and `team-b` from a template (Kustomize or Helm).
2. In each, apply ResourceQuota, LimitRange, default-deny NetworkPolicy, and Pod Security `baseline` or `restricted`.
3. Create RBAC: team admins in their namespace only, read-only cluster-wide for a platform viewer.
4. Dedicate a node pool to team-b with a taint and a label, and enforce it with a mutating or validating policy.
5. Prove isolation: team-a cannot read team-b Secrets, reach team-b Pods, or schedule onto team-b nodes.
6. Document how you would bill each team (labels, requests, OpenCost).

**Expected outcome:** A test script that proves each isolation boundary passes, and a README explaining the model.

<details>
<summary>Hints</summary>

`kubectl auth can-i get secrets -n team-b --as=... `. Namespaces are not a hard security boundary; mention sandboxed runtimes (gVisor, Kata) or separate clusters for untrusted tenants. Consider the Hierarchical Namespace Controller or Capsule.
</details>

**Bonus:** Add a per-team ClusterQueue with Kueue, or per-team vCluster, and compare the isolation level.

[⬅ Back to Kubernetes](./README.md) · [🏠 Home](../README.md)

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
