# GitOps & ArgoCD — Cheatsheet

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

> Dense reference for daily use and interview prep. Argo CD 2.13+ / 3.x, Argo Rollouts 1.7+, Sealed Secrets, External Secrets Operator, SOPS, Flux 2.x for comparison.

[⬅ Back to GitOps & ArgoCD](./README.md) · [🏠 Home](../README.md)

## Contents

[Install and access](#install-and-access) · [Login and accounts](#login-and-accounts) · [Applications](#application-commands) · [Sync and rollback](#sync-refresh-and-rollback) · [Projects](#projects) · [Clusters and repos](#clusters-and-repos) · [Admin](#admin-and-debugging) · [Application spec](#application-spec) · [Sync options](#sync-options) · [Annotations](#annotations) · [Hooks and waves](#hooks-and-waves) · [Health and diffing](#health-and-diffing) · [RBAC](#rbac) · [ApplicationSets](#applicationsets) · [Helm, Kustomize, multi-source](#helm-kustomize-and-multiple-sources) · [Rollouts](#argo-rollouts) · [Image Updater](#image-updater) · [Notifications](#notifications) · [Secrets](#secrets) · [Scaling](#scaling-and-tuning) · [Flux vs Argo CD](#flux-vs-argo-cd) · [Errors](#common-errors) · [Top 15](#top-15-interview-one-liners)

---

## Install and access

| Task | Command |
|------|---------|
| Install (manifests, non-HA) | `kubectl create ns argocd && kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml` |
| Install HA manifests | same with `.../manifests/ha/install.yaml` |
| Core mode (no UI/API server) | `.../manifests/core-install.yaml`, then `argocd login --core` |
| Install with Helm | `helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace -f values.yaml` |
| Initial admin password | `argocd admin initial-password -n argocd` |
| Same, raw | `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' \| base64 -d` |
| Port-forward UI/API | `kubectl -n argocd port-forward svc/argocd-server 8080:443` |
| Local dashboard in core mode | `argocd admin dashboard -n argocd` |
| Install CLI (Linux) | `curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 && sudo install -m 555 argocd /usr/local/bin/` |

## Login and accounts

| Task | Command |
|------|---------|
| Login (TLS, self-signed) | `argocd login argocd.example.com --username admin --insecure` |
| Login through plaintext port-forward | `argocd login localhost:8080 --plaintext` |
| Login via SSO | `argocd login argocd.example.com --sso` |
| Behind an L7 proxy without HTTP/2 | add `--grpc-web` |
| Current user and permissions | `argocd account get-user-info`, `argocd account can-i sync applications 'team-a/*'` |
| Change password | `argocd account update-password` |
| Token for a local CI account | `argocd account generate-token --account ci --expires-in 720h` |
| Project role token | `argocd proj role create-token team-a ci --expires-in 720h` |
| Use a token | `export ARGOCD_AUTH_TOKEN=...; export ARGOCD_SERVER=argocd.example.com` |

## Application commands

| Task | Command |
|------|---------|
| Create from Git path | `argocd app create web --repo https://github.com/org/cfg.git --path apps/web --dest-server https://kubernetes.default.svc --dest-namespace web --project team-a` |
| Create from Helm repo | `argocd app create redis --repo https://charts.bitnami.com/bitnami --helm-chart redis --revision 20.1.0 --dest-namespace redis --dest-server https://kubernetes.default.svc` |
| Create or update from file | `argocd app create -f app.yaml --upsert` (or `kubectl apply -f app.yaml`) |
| List / filter | `argocd app list -p team-a -l env=prod -o wide` |
| Details, tree of resources | `argocd app get web`, `argocd app resources web` |
| Diff Git vs live | `argocd app diff web` (exit 1 if diff) |
| Diff local files vs live | `argocd app diff web --local ./apps/web` |
| Rendered manifests | `argocd app manifests web --source git` (or `live`) |
| Set Helm parameter / values file | `argocd app set web -p image.tag=1.4.2`, `--values values-prod.yaml` |
| Set Kustomize image | `argocd app set web --kustomize-image ghcr.io/org/web:1.4.2` |
| Enable auto-sync | `argocd app set web --sync-policy automated --auto-prune --self-heal` |
| Disable auto-sync | `argocd app set web --sync-policy none` |
| Logs of a workload | `argocd app logs web --kind Deployment --name web --tail 100 -f` |
| Run a resource action | `argocd app actions run web restart --kind Deployment --resource-name web` |
| Delete app and resources | `argocd app delete web --cascade` |
| Delete app, keep resources | `argocd app delete web --cascade=false` |
| Wait for state | `argocd app wait web --sync --health --timeout 600` |

## Sync, refresh and rollback

| Task | Command |
|------|---------|
| Sync | `argocd app sync web` |
| Sync with prune / dry run | `argocd app sync web --prune`, `--dry-run` |
| Sync a single resource | `argocd app sync web --resource apps:Deployment:web` |
| Sync a specific revision | `argocd app sync web --revision 3f2a1bc` |
| Force replace (delete + create) | `argocd app sync web --force --replace` |
| Server-side apply for one sync | `argocd app sync web --server-side` |
| Sync with retries | `argocd app sync web --retry-limit 3 --retry-backoff-duration 10s` |
| Normal refresh (re-diff) | `argocd app get web --refresh` |
| Hard refresh (drop manifest cache) | `argocd app get web --hard-refresh` |
| Stop a running operation | `argocd app terminate-op web` |
| History | `argocd app history web` |
| Roll back to history ID (auto-sync must be off) | `argocd app rollback web 12` |
| Refresh via kubectl | `kubectl -n argocd annotate app web argocd.argoproj.io/refresh=hard --overwrite` |

## Projects

| Task | Command |
|------|---------|
| Create | `argocd proj create team-a -d https://kubernetes.default.svc,team-a-* -s https://github.com/org/*` |
| Add destination / source | `argocd proj add-destination team-a https://kubernetes.default.svc team-a-dev`, `argocd proj add-source team-a https://github.com/org/cfg.git` |
| Allow a cluster-scoped kind | `argocd proj allow-cluster-resource team-a "" Namespace` |
| Deny a namespaced kind | `argocd proj deny-namespace-resource team-a "" ResourceQuota` |
| Create role + policy | `argocd proj role create team-a ci && argocd proj role add-policy team-a ci -a sync -o '*' -p allow` |
| Bind SSO group to role | `argocd proj role add-group team-a ci my-org:team-a` |
| Sync window (deny nights) | `argocd proj windows add team-a --kind deny --schedule "0 22 * * *" --duration 8h --applications '*' --manual-sync` |
| List windows | `argocd proj windows list team-a` |

## Clusters and repos

| Task | Command |
|------|---------|
| Register cluster from kubeconfig context | `argocd cluster add prod-eu --name prod-eu --label env=prod` |
| List / remove | `argocd cluster list`, `argocd cluster rm prod-eu` |
| Rotate the argocd-manager token | `argocd cluster rotate-auth prod-eu` |
| Add HTTPS repo | `argocd repo add https://github.com/org/cfg.git --username git --password $TOKEN` |
| Add SSH repo | `argocd repo add git@github.com:org/cfg.git --ssh-private-key-path ~/.ssh/argocd` |
| Add GitHub App repo | `argocd repo add https://github.com/org/cfg.git --github-app-id 123 --github-app-installation-id 456 --github-app-private-key-path app.pem` |
| Credential template for all org repos | `argocd repocreds add https://github.com/org --username git --password $TOKEN` |
| Helm OCI repo | `argocd repo add ghcr.io/org/charts --type helm --name org-charts --enable-oci --username u --password $TOKEN` |
| Known hosts | `ssh-keyscan github.com \| argocd cert add-ssh --batch` |

Declarative cluster and repo Secrets:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: prod-eu
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
    env: prod
stringData:
  name: prod-eu
  server: https://ABC123.gr7.eu-west-1.eks.amazonaws.com
  config: |
    {
      "awsAuthConfig": { "clusterName": "prod-eu", "roleARN": "arn:aws:iam::111122223333:role/argocd-deployer" },
      "tlsClientConfig": { "caData": "LS0tLS1CRUdJTi..." }
    }
---
apiVersion: v1
kind: Secret
metadata:
  name: org-repos
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  type: git
  url: https://github.com/org
  githubAppID: "123456"
  githubAppInstallationID: "7890123"
  githubAppPrivateKey: |
    -----BEGIN RSA PRIVATE KEY-----
    ...
```

## Admin and debugging

| Task | Command |
|------|---------|
| Export everything (backup) | `argocd admin export -n argocd > argocd-backup.yaml` |
| Import | `argocd admin import -n argocd - < argocd-backup.yaml` |
| Test an RBAC decision | `argocd admin settings rbac can role:dev sync applications 'team-a/web' --policy-file policy.csv` |
| Validate RBAC file | `argocd admin settings rbac validate --policy-file policy.csv` |
| Test a Lua health check | `argocd admin settings resource-overrides health ./cert.yaml --argocd-cm-path ./argocd-cm.yaml` |
| Test ignoreDifferences | `argocd admin settings resource-overrides ignore-differences ./deploy.yaml --argocd-cm-path ./argocd-cm.yaml` |
| Shard distribution | `argocd admin cluster shards -n argocd` |
| Cluster cache stats | `argocd admin cluster stats -n argocd` |
| Generate an Application spec | `argocd admin app generate-spec web --repo ... --path ... --dest-namespace web --dest-server https://kubernetes.default.svc` |
| Controller logs for one app | `kubectl -n argocd logs sts/argocd-application-controller \| grep 'application=argocd/web'` |
| Repo-server logs | `kubectl -n argocd logs deploy/argocd-repo-server --tail 200` |
| Restart components after config change | `kubectl -n argocd rollout restart deploy/argocd-server deploy/argocd-repo-server sts/argocd-application-controller` |

## Application spec

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io      # cascade delete
  annotations:
    argocd.argoproj.io/manifest-generate-paths: .  # webhook refresh only on changes here
spec:
  project: team-a
  source:
    repoURL: https://github.com/org/cfg.git
    targetRevision: main          # branch, tag, SHA, or semver range for Helm charts
    path: apps/web/overlays/prod
  destination:
    name: prod-eu                 # or server: https://...
    namespace: web
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
      - PruneLast=true
      - RespectIgnoreDifferences=true
    managedNamespaceMetadata:
      labels:
        team: team-a
    retry:
      limit: 5
      backoff: {duration: 5s, factor: 2, maxDuration: 3m}
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers: [/spec/replicas]
    - group: admissionregistration.k8s.io
      kind: MutatingWebhookConfiguration
      jqPathExpressions: ['.webhooks[]?.clientConfig.caBundle']
  revisionHistoryLimit: 10
```

| Status | Values |
|--------|--------|
| Sync | `Synced`, `OutOfSync`, `Unknown` |
| Health | `Healthy`, `Progressing`, `Degraded`, `Suspended`, `Missing`, `Unknown` |
| Operation phase | `Running`, `Succeeded`, `Failed`, `Error`, `Terminating` |

## Sync options

| Option | Where | Effect |
|--------|-------|--------|
| `CreateNamespace=true` | app | Create destination namespace if missing |
| `ServerSideApply=true` | app or resource | Use server-side apply (large CRDs, field ownership) |
| `ServerSideDiff=true` | compare-options | Diff via API server dry-run (fewer false OutOfSync) |
| `PruneLast=true` | app or resource | Prune after all waves are healthy |
| `Prune=false` | resource | Never prune this resource |
| `Prune=confirm` / `Delete=confirm` | resource | Require manual confirmation |
| `Delete=false` | resource | Keep resource when the app is deleted |
| `ApplyOutOfSyncOnly=true` | app | Apply only changed resources (big apps) |
| `Replace=true` | app or resource | `kubectl replace/create` instead of apply |
| `Force=true` | resource (with Replace) | Delete and recreate (immutable fields) |
| `Validate=false` | app or resource | Skip schema validation |
| `SkipDryRunOnMissingResource=true` | app or resource | Do not dry-run kinds whose CRD is not installed yet |
| `PrunePropagationPolicy=foreground` | app | `foreground`, `background`, `orphan` |
| `RespectIgnoreDifferences=true` | app | Do not overwrite ignored fields on sync |
| `FailOnSharedResource=true` | app | Fail if a resource is managed by another app |

## Annotations

| Annotation | Purpose |
|------------|---------|
| `argocd.argoproj.io/sync-wave: "-1"` | Ordering within a phase (default 0) |
| `argocd.argoproj.io/hook: PreSync` | `PreSync`, `Sync`, `PostSync`, `SyncFail`, `PostDelete`, `Skip` |
| `argocd.argoproj.io/hook-delete-policy: HookSucceeded` | `BeforeHookCreation` (default), `HookSucceeded`, `HookFailed` |
| `argocd.argoproj.io/sync-options: Prune=false,ServerSideApply=true` | Per-resource sync options |
| `argocd.argoproj.io/compare-options: IgnoreExtraneous` | Exclude from app sync status (generated resources) |
| `argocd.argoproj.io/compare-options: ServerSideDiff=true` | Server-side diff for this app |
| `argocd.argoproj.io/tracking-id` | Written by Argo CD in annotation tracking mode (default in 3.x) |
| `argocd.argoproj.io/refresh: normal\|hard` | Trigger refresh on an Application |
| `argocd.argoproj.io/manifest-generate-paths` | Paths whose changes trigger refresh from webhooks |
| `argocd.argoproj.io/skip-reconcile: "true"` | Pause reconciliation of an Application |
| `notifications.argoproj.io/subscribe.on-sync-failed.slack: deploys` | Notification subscription |

## Hooks and waves

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
    argocd.argoproj.io/sync-wave: "-1"
spec:
  backoffLimit: 1
  activeDeadlineSeconds: 600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: ghcr.io/org/api:1.4.2
          command: ["./migrate", "up"]
```

Order: PreSync, then Sync by wave (low to high), then PostSync. Within a wave: kind order (Namespace, CRD, ServiceAccount, RBAC, ConfigMap, Secret, Service, workloads, Ingress...), then name. Each wave must be healthy before the next. SyncFail hooks run when the sync fails. Helm `pre-install`/`pre-upgrade` map to PreSync, `post-install`/`post-upgrade` to PostSync.

## Health and diffing

`argocd-cm` custom health in Lua:

```yaml
data:
  resource.customizations.health.cert-manager.io_Certificate: |
    hs = { status = "Progressing", message = "Waiting for certificate" }
    if obj.status ~= nil and obj.status.conditions ~= nil then
      for _, c in ipairs(obj.status.conditions) do
        if c.type == "Ready" and c.status == "True" then
          hs.status = "Healthy"; hs.message = c.message
        elseif c.type == "Ready" and c.status == "False" then
          hs.status = "Degraded"; hs.message = c.message
        end
      end
    end
    return hs
  resource.customizations.ignoreDifferences.all: |
    managedFieldsManagers:
      - kube-controller-manager
  resource.customizations.ignoreDifferences.apps_Deployment: |
    jsonPointers:
      - /spec/replicas
  resource.exclusions: |
    - apiGroups: ["cilium.io"]
      kinds: ["CiliumIdentity"]
      clusters: ["*"]
  resource.customizations.ignoreResourceUpdates.all: |
    jsonPointers:
      - /status
```

| Key in `argocd-cm` | Use |
|--------------------|-----|
| `resource.customizations.health.<group>_<kind>` | Lua health check |
| `resource.customizations.actions.<group>_<kind>` | Custom UI/CLI actions in Lua |
| `resource.customizations.ignoreDifferences.<group>_<kind>` / `.all` | Global diff ignores |
| `resource.customizations.ignoreResourceUpdates.<group>_<kind>` | Skip reconciles on noisy updates |
| `resource.exclusions` / `resource.inclusions` | Which kinds the controller watches |
| `resource.compareoptions` | e.g. `ignoreAggregatedRoles: true` |
| `timeout.reconciliation` | Polling interval (default `180s`) |
| `application.resourceTrackingMethod` | `label`, `annotation`, `annotation+label` |
| `kustomize.buildOptions` | e.g. `--enable-helm --load-restrictor LoadRestrictionsNone` |

## RBAC

`argocd-rbac-cm`:

```yaml
data:
  policy.default: role:readonly
  scopes: "[groups, email]"
  policy.csv: |
    # p, subject, resource, action, object, effect
    p, role:team-a-dev, applications, get,      team-a/*, allow
    p, role:team-a-dev, applications, sync,     team-a/*-dev, allow
    p, role:team-a-dev, applications, action/*, team-a/*-dev, allow
    p, role:team-a-dev, logs,         get,      team-a/*, allow
    p, role:team-a-dev, applications, sync,     team-a/*-prod, deny
    g, my-org:team-a, role:team-a-dev
    g, alice@example.com, role:admin
```

| Resources | Actions |
|-----------|---------|
| `applications`, `applicationsets`, `logs`, `exec` | `get`, `create`, `update`, `delete`, `sync`, `override`, `action/<group>/<kind>/<action>` |
| `clusters`, `projects`, `repositories`, `accounts`, `certificates`, `gpgkeys` | `get`, `create`, `update`, `delete` |
| `extensions` | `invoke` |

Built-in roles: `role:readonly`, `role:admin`. Deny wins over allow. App objects are `<project>/<app>` (or `<project>/<namespace>/<app>` for apps in any namespace).

## ApplicationSets

| Generator | Iterates | Key params |
|-----------|----------|-----------|
| `list` | Fixed `elements` | any keys you define |
| `clusters` | Cluster Secrets (plus in-cluster) filtered by `selector` | `name`, `server`, `metadata.labels.<k>`, `values.<k>` |
| `git` directories | Folders matching `path` (with `exclude`) | `path.path`, `path.basename`, `path.segments` |
| `git` files | JSON/YAML files matching `path` | file keys, `path.path` |
| `matrix` | Cartesian product of 2 generators | params from both |
| `merge` | Merge generators on `mergeKeys` | overrides by key |
| `scmProvider` | Repos in a GitHub/GitLab org | `repository`, `branch`, `url` |
| `pullRequest` | Open PRs | `number`, `branch`, `head_sha` |
| `clusterDecisionResource` | External placement (OCM) | from the CR status |
| `plugin` | Your HTTP service | anything returned |

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - matrix:
        generators:
          - clusters:
              selector:
                matchLabels:
                  env: prod
          - git:
              repoURL: https://github.com/org/cfg.git
              revision: main
              directories:
                - path: apps/*
  template:
    metadata:
      name: "{{ .path.basename }}-{{ .name }}"
    spec:
      project: platform
      source:
        repoURL: https://github.com/org/cfg.git
        targetRevision: main
        path: "{{ .path.path }}"
      destination:
        server: "{{ .server }}"
        namespace: "{{ .path.basename }}"
  syncPolicy:
    preserveResourcesOnDeletion: true
    applicationsSync: create-update       # create-only | create-update | create-delete | sync
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions:
            - {key: env, operator: In, values: [staging]}
        - matchExpressions:
            - {key: env, operator: In, values: [prod]}
          maxUpdate: 25%
```

## Helm, Kustomize and multiple sources

```yaml
# Helm chart from a repo, values from Git (multiple sources)
spec:
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: 65.1.1
      helm:
        releaseName: monitoring
        valueFiles:
          - $values/platform/monitoring/values-prod.yaml
    - repoURL: https://github.com/org/cfg.git
      targetRevision: main
      ref: values
---
# Helm chart in Git with inline overrides
spec:
  source:
    repoURL: https://github.com/org/cfg.git
    path: charts/web
    helm:
      valueFiles: [values.yaml, values-prod.yaml]
      ignoreMissingValueFiles: true
      valuesObject:
        replicaCount: 4
      parameters:
        - name: image.tag
          value: "1.4.2"
---
# Kustomize overlay with overrides
spec:
  source:
    repoURL: https://github.com/org/cfg.git
    path: apps/web/overlays/prod
    kustomize:
      images:
        - ghcr.io/org/web:1.4.2
      namePrefix: prod-
      commonLabels:
        team: team-a
```

Helm in Argo CD = `helm template`: no release Secret, `helm list` is empty, `lookup` returns nothing, `.Capabilities` comes from the target cluster's API versions, and Helm hooks become Argo CD hooks.

## Argo Rollouts

| Task | Command |
|------|---------|
| Install | `kubectl create ns argo-rollouts && kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml` |
| Watch | `kubectl argo rollouts get rollout web -n prod --watch` |
| Status (exit code for CI) | `kubectl argo rollouts status web -n prod --timeout 10m` |
| Promote one step / fully | `kubectl argo rollouts promote web`, `--full` |
| Abort / retry | `kubectl argo rollouts abort web`, `kubectl argo rollouts retry rollout web` |
| Undo to previous revision | `kubectl argo rollouts undo web` |
| Restart pods | `kubectl argo rollouts restart web` |
| Dashboard | `kubectl argo rollouts dashboard` |
| Analysis runs | `kubectl get analysisrun -n prod`, `kubectl describe analysisrun <name>` |

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: {name: web}
spec:
  replicas: 10
  selector: {matchLabels: {app: web}}
  template: {...}                       # or workloadRef to an existing Deployment
  strategy:
    canary:
      canaryService: web-canary
      stableService: web-stable
      trafficRouting:
        nginx:
          stableIngress: web
      analysis:                         # background analysis from step 1
        templates: [{templateName: success-rate}]
        startingStep: 1
        args: [{name: service, value: web-canary}]
      steps:
        - setWeight: 10
        - pause: {duration: 2m}
        - setWeight: 50
        - pause: {}                     # manual gate
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: {name: success-rate}
spec:
  args: [{name: service}]
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      failureLimit: 1
      successCondition: result[0] >= 0.99
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service}}",code!~"5.."}[2m]))
            /
            sum(rate(http_requests_total{service="{{args.service}}"}[2m]))
```

Blue-green essentials: `activeService`, `previewService`, `autoPromotionEnabled: false`, `prePromotionAnalysis`, `postPromotionAnalysis`, `scaleDownDelaySeconds` (default 30).

## Image Updater

| Annotation (on the Application) | Example |
|-----------|---------|
| `argocd-image-updater.argoproj.io/image-list` | `web=ghcr.io/org/web` |
| `argocd-image-updater.argoproj.io/web.update-strategy` | `semver` (also digest and newest-build / alphabetical style strategies) |
| `argocd-image-updater.argoproj.io/web.allow-tags` | `regexp:^1\.[0-9]+\.[0-9]+$` |
| `argocd-image-updater.argoproj.io/web.helm.image-name` / `.helm.image-tag` | `image.repository` / `image.tag` |
| `argocd-image-updater.argoproj.io/write-back-method` | `argocd` (parameter override) or `git` |
| `argocd-image-updater.argoproj.io/git-branch` | `main` or `main:image-updates` (push to a separate branch) |

Git write-back creates `.argocd-source-<app>.yaml` next to the app's path. Only Helm and Kustomize sources are supported. Newer releases (1.x) move this configuration into an `ImageUpdater` custom resource.

## Notifications

```yaml
# argocd-notifications-cm
data:
  service.slack: |
    token: $slack-token                 # key in argocd-notifications-secret
  template.app-sync-failed: |
    message: "{{.app.metadata.name}} sync failed at {{.app.status.operationState.syncResult.revision}}: {{.app.status.operationState.message}}"
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      oncePer: app.status.sync.revision
      send: [app-health-degraded]
```

Subscribe with `notifications.argoproj.io/subscribe.on-sync-failed.slack: deploys` on an Application or AppProject. Test with `argocd admin notifications template notify app-sync-failed web -n argocd`.

## Secrets

| Task | Command |
|------|---------|
| Fetch Sealed Secrets cert | `kubeseal --fetch-cert > pub-cert.pem` |
| Seal a Secret | `kubectl create secret generic db --from-literal=password=s3cr3t -n app --dry-run=client -o yaml \| kubeseal --cert pub-cert.pem -o yaml > db-sealed.yaml` |
| Seal one value | `echo -n s3cr3t \| kubeseal --raw --namespace app --name db --from-file=/dev/stdin` |
| Scope | `--scope strict` (default), `namespace-wide`, `cluster-wide` |
| Re-encrypt with newest key | `kubeseal --re-encrypt < db-sealed.yaml > tmp && mv tmp db-sealed.yaml` |
| Back up keys | `kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > keys.yaml` |
| SOPS encrypt with age | `sops --encrypt --age age1... --encrypted-regex '^(data\|stringData)$' --in-place secret.yaml` |
| SOPS decrypt / edit | `sops -d secret.yaml`, `sops secret.yaml` |
| Generate an age key | `age-keygen -o key.txt` |

```yaml
# External Secrets Operator
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata: {name: db, namespace: app}
spec:
  refreshInterval: 1h
  secretStoreRef: {name: aws-sm, kind: ClusterSecretStore}
  target: {name: db, creationPolicy: Owner}
  data:
    - secretKey: password
      remoteRef: {key: prod/app/db, property: password}
---
# .sops.yaml
creation_rules:
  - path_regex: envs/prod/.*\.yaml
    encrypted_regex: ^(data|stringData)$
    kms: arn:aws:kms:eu-west-1:111122223333:key/abcd-1234
  - path_regex: .*\.yaml
    encrypted_regex: ^(data|stringData)$
    age: age1qyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqs3290gq
```

## Scaling and tuning

| Setting (`argocd-cmd-params-cm` unless noted) | Default | Use |
|---------------------|---------|-----|
| `controller.status.processors` | 20 | Raise for many apps (e.g. 50 per 1000 apps) |
| `controller.operation.processors` | 10 | Concurrent syncs |
| `controller.repo.server.timeout.seconds` | 60 | Slow Helm/Kustomize renders |
| `controller.sharding.algorithm` | `legacy` | `round-robin`, `consistent-hashing` |
| `ARGOCD_CONTROLLER_REPLICAS` (env on controller) | 1 | Must equal StatefulSet replicas for sharding |
| `controller.kubectl.parallelism.limit` | 20 | Concurrent kubectl operations |
| `reposerver.parallelism.limit` | 0 (unlimited) | Cap concurrent renders to avoid OOM |
| `ARGOCD_EXEC_TIMEOUT` (env on repo-server) | 90s | Timeout for helm/kustomize/git exec |
| `application.namespaces` | none | Enable apps in any namespace |
| `server.insecure` | false | Terminate TLS at the ingress |
| `timeout.reconciliation` (`argocd-cm`) | 180s | Raise when webhooks are configured |

Pin a cluster to a shard: add `shard: "2"` to the cluster Secret's data.

## Flux vs Argo CD

| | Argo CD | Flux |
|---|---------|------|
| Model | Application CRD, central controller | GitRepository + Kustomization + HelmRelease CRDs |
| UI | Built-in UI, SSO, RBAC | No built-in UI (third-party UIs exist) |
| Multi-cluster | Hub-and-spoke from one instance | Usually one Flux per cluster |
| Helm | `helm template` + apply, no release | Real Helm SDK releases, `helm list` works |
| Ordering | Sync waves and hooks | `dependsOn` between Kustomizations/HelmReleases |
| Templating fleets | ApplicationSet | Kustomize overlays, `postBuild.substitute` |
| Secrets | Plugins for SOPS; Sealed/External Secrets | Native SOPS decryption in Kustomization |
| Image automation | Image Updater (separate) | image-reflector + image-automation controllers |
| Tenancy | AppProjects + RBAC | Service account impersonation per Kustomization |
| Bootstrap | `kubectl apply` root app | `flux bootstrap github --owner org --repository fleet --path clusters/prod` |

## Common errors

| Error | Fix |
|-------|-----|
| `ComparisonError: ... Manifest generation error (cached)` | Fix the render error, then `argocd app get <app> --hard-refresh` |
| `application repo <url> is not permitted in project '<p>'` | Add the exact URL to `sourceRepos` |
| `... do not match any of the allowed destinations in project '<p>'` | Add the server/name and namespace to `destinations` |
| `resource <group>:<kind> is not permitted in project <p>` | Whitelist the kind or remove it from the app |
| `permission denied: applications, sync, <proj>/<app>, sub: ...` | RBAC policy or group mapping missing; test with `rbac can` |
| `rollback cannot be initiated when auto-sync is enabled` | Disable auto-sync or, better, `git revert` |
| `another operation is already in progress` | Wait, or `argocd app terminate-op <app>` |
| `metadata.annotations: Too long: must have at most 262144 bytes` | `ServerSideApply=true` (or `Replace=true` for that resource) |
| `field is immutable` | Change the name, or `Replace=true,Force=true` on that resource |
| `no matches for kind "X" in version "Y"` | Install the CRD in an earlier wave, `SkipDryRunOnMissingResource=true` |
| `rpc error: code = DeadlineExceeded desc = context deadline exceeded` | Repo-server slow: raise timeouts, scale repo-server, check Git/Helm reachability |
| `ssh: handshake failed: knownhosts: key is unknown` | Add the host key with `argocd cert add-ssh` |
| `authentication required` / `repository not found` | Repo credentials missing or wrong; check repo/repocreds Secrets |
| App stuck deleting | Resource with a finalizer blocks cascade; fix it or remove `resources-finalizer.argocd.argoproj.io` with care |

---

## Top 15 interview one-liners

1. **GitOps in one line:** declarative desired state, versioned in Git, pulled automatically by an agent, continuously reconciled against the live cluster.
2. **Push vs pull:** push gives CI cluster credentials and only acts on pipeline runs; pull keeps credentials inside the cluster and corrects drift continuously.
3. **Sync vs health:** sync status says whether live matches Git; health says whether it works. Synced and Degraded can happen together.
4. **prune vs selfHeal:** prune deletes what was removed from Git; selfHeal reverts changes made in the cluster. Both are off unless you turn them on.
5. **Repo-server vs controller:** the repo-server clones and renders manifests; the application controller diffs and applies. Only the controller writes to clusters.
6. **Redis:** a cache only; losing it costs a cache rebuild, not state.
7. **Sync waves:** ascending order within each phase, each wave must be healthy before the next; hooks run in PreSync, Sync, PostSync and SyncFail phases.
8. **OutOfSync loop:** something mutates a field you declare (HPA, webhook, defaulting); stop declaring it, use `ignoreDifferences` with `RespectIgnoreDifferences=true`, or server-side diff.
9. **AppProject:** the tenancy boundary that limits source repos, destinations, resource kinds, roles and sync windows; never let teams use `default`.
10. **ApplicationSet:** a generator (list, cluster, git, matrix, merge, SCM, PR) plus a template that stamps out Applications; use `goTemplate` with `missingkey=error`.
11. **App-of-apps:** one root Application manages other Application manifests, so a cluster bootstraps from one `kubectl apply`.
12. **Helm in Argo CD:** rendered with `helm template` and applied; no Helm release Secret, so `helm list` is empty and `lookup` does not work.
13. **Rollback the GitOps way:** `git revert` the commit; `argocd app rollback` works only with auto-sync off and leaves Git disagreeing with the cluster.
14. **Secrets:** never plain in Git; Sealed Secrets (encrypt for one cluster), External Secrets (reference a secret manager, best for rotation) or SOPS (encrypt values in place with KMS/age).
15. **Scaling Argo CD:** shard the controller by cluster with `ARGOCD_CONTROLLER_REPLICAS`, raise status and operation processors, scale repo-server, use webhooks plus `manifest-generate-paths`, and exclude noisy resources.

---

[⬅ Back to GitOps & ArgoCD](./README.md) · [🏠 Home](../README.md)

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
