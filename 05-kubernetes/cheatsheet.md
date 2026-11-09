# Kubernetes — Cheatsheet

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

[⬅ Back to Kubernetes](./README.md) · [🏠 Home](../README.md)

## Setup and productivity

```bash
alias k=kubectl
source <(kubectl completion bash) && complete -o default -F __start_kubectl k
export do="--dry-run=client -o yaml"          # k run x --image=nginx $do
kubectl config get-contexts
kubectl config use-context <ctx>
kubectl config set-context --current --namespace=<ns>
kubectl api-resources                          # kinds, short names, namespaced or not
kubectl explain deploy.spec.strategy --recursive
```

## Create, apply, delete

| Task | Command |
|---|---|
| Apply file or dir | `kubectl apply -f x.yaml` / `-f dir/` / `-k overlay/` |
| Preview change | `kubectl diff -f x.yaml` |
| Server-side validate | `kubectl apply -f x.yaml --dry-run=server` |
| Run a Pod | `kubectl run web --image=nginx:1.27 --port=80` |
| Create Deployment | `kubectl create deploy web --image=nginx:1.27 --replicas=3` |
| Expose | `kubectl expose deploy web --port=80 --target-port=8080` |
| Delete | `kubectl delete -f x.yaml` / `kubectl delete pod x --grace-period=0 --force` |
| Replace stuck object | `kubectl replace --force -f x.yaml` |

## Inspect and query

| Task | Command |
|---|---|
| List with detail | `kubectl get pods -o wide -A` |
| Watch | `kubectl get pods -w` |
| Labels | `kubectl get pods --show-labels -l app=web,tier!=db` |
| Field select | `kubectl get pods --field-selector status.phase=Pending,spec.nodeName=n1` |
| JSONPath | `kubectl get pod x -o jsonpath='{.status.podIP}'` |
| Custom columns | `kubectl get pods -o custom-columns=N:.metadata.name,NODE:.spec.nodeName` |
| Sort | `kubectl get pods --sort-by=.status.startTime` |
| Events | `kubectl get events --sort-by=.lastTimestamp -A` |
| Describe | `kubectl describe pod x` |
| Raw object | `kubectl get pod x -o yaml` |
| Resource usage | `kubectl top pods --containers` / `kubectl top nodes` |

## Debugging

```bash
kubectl logs x                       # add -c <container> -f --tail=100 --since=10m
kubectl logs x --previous            # last crashed container
kubectl logs -l app=web --all-containers --prefix
kubectl exec -it x -- sh
kubectl port-forward svc/web 8080:80
kubectl debug -it x --image=busybox:1.36 --target=app         # ephemeral container
kubectl debug node/n1 -it --image=ubuntu                       # host filesystem at /host
kubectl run tmp --rm -it --restart=Never --image=nicolaka/netshoot -- bash
kubectl cp x:/path/file ./file
```

### Pod status cheat table

| Status | Meaning | First look |
|---|---|---|
| `Pending` | Not scheduled or volume or image pending | `describe pod` events: `Insufficient cpu`, `taints`, `unbound PVC` |
| `ImagePullBackOff` / `ErrImagePull` | Cannot pull image | Tag typo, registry auth, `imagePullSecrets` |
| `CrashLoopBackOff` | Container keeps exiting | `logs --previous`, exit code, probes |
| `OOMKilled` (exit 137) | Memory limit exceeded | Raise limit, find leak |
| `CreateContainerConfigError` | Missing ConfigMap, Secret or key | `describe pod` |
| `Init:Error` / `Init:CrashLoopBackOff` | Init container failing | `logs -c <init>` |
| `Evicted` | Node pressure | `describe node`, ephemeral storage, memory |
| `Terminating` (stuck) | Finalizer or node down | `get pod -o yaml` finalizers, kubelet state |
| `Running` but `0/1` | Readiness failing | Probe path and port, `describe` |

Exit codes: `0` ok, `1` app error, `126` not executable, `127` command not found, `137` SIGKILL (OOM or kill), `139` segfault, `143` SIGTERM.

## Workloads

```bash
kubectl scale deploy web --replicas=5
kubectl set image deploy/web nginx=nginx:1.27.1
kubectl set resources deploy/web --requests=cpu=100m,memory=128Mi --limits=memory=256Mi
kubectl rollout status deploy/web
kubectl rollout history deploy/web --revision=2
kubectl rollout undo deploy/web --to-revision=1
kubectl rollout restart deploy/web
kubectl rollout pause deploy/web      # then resume
kubectl autoscale deploy web --min=2 --max=10 --cpu-percent=60
kubectl create job once --image=busybox -- date
kubectl create job manual --from=cronjob/nightly
```

### Deployment skeleton

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: web, labels: { app: web } }
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 25%, maxUnavailable: 0 }
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: web
          image: nginx:1.27
          ports: [{ containerPort: 80, name: http }]
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits: { memory: 256Mi }
          startupProbe:   { httpGet: { path: /healthz, port: http }, failureThreshold: 30, periodSeconds: 2 }
          livenessProbe:  { httpGet: { path: /healthz, port: http }, periodSeconds: 10 }
          readinessProbe: { httpGet: { path: /ready, port: http }, periodSeconds: 5 }
          lifecycle:
            preStop: { exec: { command: ["sleep", "5"] } }
          securityContext:
            runAsNonRoot: true
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
            seccompProfile: { type: RuntimeDefault }
```

## Services, Ingress, DNS

| Type | Use |
|---|---|
| `ClusterIP` | Internal virtual IP (default) |
| `NodePort` | Port 30000-32767 on every node |
| `LoadBalancer` | Cloud load balancer |
| `ExternalName` | CNAME to an external host |
| `clusterIP: None` | Headless: DNS returns Pod IPs, used by StatefulSets |

```text
<svc>.<ns>.svc.cluster.local            Service
<pod-name>.<headless-svc>.<ns>.svc...   StatefulSet Pod
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations: { nginx.ingress.kubernetes.io/proxy-body-size: "20m" }
spec:
  ingressClassName: nginx
  tls: [{ hosts: [app.example.com], secretName: app-tls }]
  rules:
    - host: app.example.com
      http:
        paths:
          - { path: /, pathType: Prefix, backend: { service: { name: web, port: { number: 80 } } } }
```

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web
kubectl create secret tls app-tls --cert=tls.crt --key=tls.key
kubectl run dns --rm -it --restart=Never --image=busybox:1.36 -- nslookup web.default.svc.cluster.local
```

## Config and secrets

```bash
kubectl create cm app --from-literal=K=V --from-file=app.conf
kubectl create secret generic db --from-literal=password='s3cr3t'
kubectl create secret docker-registry reg --docker-server=ghcr.io --docker-username=u --docker-password=$TOKEN
kubectl get secret db -o jsonpath='{.data.password}' | base64 -d
kubectl set env deploy/web --from=configmap/app
```

```yaml
envFrom:
  - configMapRef: { name: app }
env:
  - name: DB_PASSWORD
    valueFrom: { secretKeyRef: { name: db, key: password } }
volumeMounts: [{ name: cfg, mountPath: /etc/app, readOnly: true }]
volumes: [{ name: cfg, configMap: { name: app } }]
```

## Storage

```bash
kubectl get sc,pv,pvc
kubectl patch pvc data -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'   # needs allowVolumeExpansion
kubectl patch sc standard -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

| Access mode | Meaning |
|---|---|
| `ReadWriteOnce` | One node read-write |
| `ReadWriteOncePod` | One Pod read-write |
| `ReadOnlyMany` | Many nodes read-only |
| `ReadWriteMany` | Many nodes read-write (EFS, NFS, CephFS) |

Reclaim policy: `Delete` removes the volume with the PVC, `Retain` keeps it.

## Scheduling

```bash
kubectl label node n1 disk=ssd
kubectl taint node n1 dedicated=gpu:NoSchedule        # remove with dedicated=gpu:NoSchedule-
kubectl cordon n1 && kubectl drain n1 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon n1
```

```yaml
nodeSelector: { disk: ssd }
tolerations: [{ key: dedicated, operator: Equal, value: gpu, effect: NoSchedule }]
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector: { matchLabels: { app: web } }
        topologyKey: kubernetes.io/hostname
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector: { matchLabels: { app: web } }
```

| QoS | Condition | Evicted |
|---|---|---|
| Guaranteed | requests = limits for all containers (CPU and memory) | last |
| Burstable | some request or limit set | middle |
| BestEffort | none set | first |

## RBAC

```bash
kubectl create role r --verb=get,list,watch --resource=pods -n ns
kubectl create rolebinding rb --role=r --serviceaccount=ns:sa -n ns
kubectl create clusterrole cr --verb=get,list --resource=nodes
kubectl create clusterrolebinding crb --clusterrole=cr --user=alice
kubectl auth can-i delete pods -n ns --as=system:serviceaccount:ns:sa
kubectl auth can-i --list -n ns --as=alice
kubectl create token sa --duration=1h -n ns
kubectl get rolebindings,clusterrolebindings -A -o wide | grep <subject>
```

## NetworkPolicy

```yaml
# deny all in and out for the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# allow api -> db:5432
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: db-from-api }
spec:
  podSelector: { matchLabels: { tier: db } }
  policyTypes: [Ingress]
  ingress:
    - from: [{ podSelector: { matchLabels: { tier: api } } }]
      ports: [{ protocol: TCP, port: 5432 }]
```

Remember: always allow DNS (UDP and TCP 53 to kube-dns) when you deny egress. Policies are additive; there is no deny rule.

## Autoscaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: web }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: web }
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 60 } }
  behavior:
    scaleDown: { stabilizationWindowSeconds: 300 }
```

Formula: `desired = ceil(current * currentMetric / targetMetric)`. Utilization is relative to the request.

## Cluster and node operations

```bash
kubectl get nodes -o wide
kubectl describe node n1 | sed -n '/Allocated resources/,/Events/p'
kubectl get componentstatuses 2>/dev/null; kubectl get --raw='/readyz?verbose'
kubectl cluster-info dump | head
kubectl get pods -n kube-system
kubeadm certs check-expiration                          # on a kubeadm control plane node
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

On the node (containerd): `crictl ps`, `crictl images`, `crictl logs <id>`, `journalctl -u kubelet -f`.

## Kustomize and Helm

```bash
kubectl kustomize overlays/prod | kubectl apply -f -
helm repo add bitnami https://charts.bitnami.com/bitnami && helm repo update
helm upgrade --install web ./chart -n prod --create-namespace -f values-prod.yaml --atomic --timeout 5m
helm template web ./chart -f values.yaml | kubectl apply --dry-run=server -f -
helm list -A && helm history web -n prod && helm rollback web 1 -n prod
helm get values web -n prod
```

## kind and minikube

```bash
kind create cluster --name dev --config kind.yaml
kind load docker-image app:1.0 --name dev
kind delete cluster --name dev
minikube start --nodes 3 --cpus 4 --memory 6g
minikube addons enable ingress metrics-server
minikube service web --url
minikube delete
```

## Top 15 interview one-liners

1. **Pod:** the smallest deployable unit, one or more containers sharing network and storage.
2. **Deployment vs StatefulSet:** Deployment pods are interchangeable; StatefulSet pods have stable names, ordered rollout and their own PVC.
3. **Service:** a stable virtual IP and DNS name in front of Pods, backed by EndpointSlices of ready Pods only.
4. **Ingress vs LoadBalancer:** a LoadBalancer Service exposes one service at L4 per load balancer; Ingress routes many hosts and paths at L7 through one entry point.
5. **Liveness vs readiness:** liveness restarts the container, readiness removes it from traffic, startup delays both until the app is up.
6. **Requests vs limits:** requests are for scheduling and guaranteed share, limits cap usage; CPU over limit throttles, memory over limit kills.
7. **ConfigMap vs Secret:** both are key-value config; Secrets are base64, not encrypted, unless encryption at rest or an external store is used.
8. **What happens on `kubectl apply`:** authn, authz, admission, write to etcd; controllers create ReplicaSet and Pods, scheduler binds, kubelet runs them.
9. **CrashLoopBackOff:** the container starts and exits repeatedly with exponential backoff up to five minutes; read `logs --previous` and the exit code.
10. **Rolling update control:** `maxSurge` and `maxUnavailable`, gated by readiness; rollback is `kubectl rollout undo`.
11. **RBAC:** deny by default; Role plus RoleBinding in a namespace, ClusterRole plus ClusterRoleBinding cluster-wide; verify with `auth can-i`.
12. **NetworkPolicy:** allow-list only; once a Pod is selected, anything not allowed is denied, and your CNI must enforce it.
13. **HPA:** scales replicas on metrics, needs metrics-server and CPU requests, and uses `ceil(current * metric / target)`.
14. **Taint vs affinity:** a taint repels Pods unless tolerated; affinity attracts Pods to nodes. Use both to dedicate a node pool.
15. **Zero-downtime deploy:** readiness probe, `maxUnavailable: 0`, a `preStop` sleep, adequate `terminationGracePeriodSeconds`, and a PodDisruptionBudget.

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
