# GitOps & ArgoCD — Assignments

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

> 25 practical assignments: 10 🟢 Beginner, 10 🟡 Intermediate, 5 🔴 Advanced. Do them on a real cluster (kind is enough for almost all of them); reading is not enough.

[⬅ Back to GitOps & ArgoCD](./README.md) · [🏠 Home](../README.md)

## Index

| # | Level | Assignment |
|---|-------|------------|
| 1 | 🟢 | Install Argo CD and map its components |
| 2 | 🟢 | Your first Application from the CLI |
| 3 | 🟢 | Declarative Applications and reading status |
| 4 | 🟢 | Automated sync, prune and self-heal |
| 5 | 🟢 | Deploy a Helm chart from a Helm repository |
| 6 | 🟢 | Kustomize overlays as separate Applications |
| 7 | 🟢 | History, rollback and the GitOps way to undo |
| 8 | 🟢 | Private repositories and credential templates |
| 9 | 🟢 | Order resources with sync waves |
| 10 | 🟢 | Local accounts and a read-only user |
| 11 | 🟡 | Migration hooks and failure hooks |
| 12 | 🟡 | Kill an OutOfSync loop |
| 13 | 🟡 | Custom health checks in Lua |
| 14 | 🟡 | Multi-tenant AppProjects with sync windows |
| 15 | 🟡 | RBAC and SSO group mapping |
| 16 | 🟡 | Bootstrap a cluster with app-of-apps |
| 17 | 🟡 | ApplicationSets with list and git generators |
| 18 | 🟡 | Sealed Secrets end to end, including key recovery |
| 19 | 🟡 | CI that commits to the config repo, promotion by PR |
| 20 | 🟡 | Notifications on failure and degradation |
| 21 | 🔴 | Multi-cluster fleet with cluster and matrix generators |
| 22 | 🔴 | Progressive delivery with Argo Rollouts and Prometheus |
| 23 | 🔴 | Multiple sources and Image Updater with Git write-back |
| 24 | 🔴 | Disaster recovery: delete Argo CD and rebuild it |
| 25 | 🔴 | Scale Argo CD to hundreds of apps with sharding |

---

## 🟢 Beginner

### Assignment 1. Install Argo CD and map its components 🟢

**Objective:** Get Argo CD running and know what every pod is for.

**Tasks**
1. Create a kind cluster and install Argo CD from the `stable` manifests into the `argocd` namespace.
2. List every Deployment, StatefulSet and Service in `argocd`.
3. Retrieve the initial admin password, port-forward `argocd-server`, and log in with both the UI and the CLI.
4. Write one sentence per component explaining its job: api-server, repo-server, application-controller, Redis, Dex, ApplicationSet controller, notifications controller.
5. Delete the `argocd-redis` pod and confirm the UI recovers without losing any Application.

**Expected outcome / acceptance criteria**
- `argocd version` shows both client and server versions.
- Your component notes correctly say which component clones Git and which one writes to the cluster.
- You can explain why deleting Redis lost nothing.

<details>
<summary>Hints</summary>

`kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`. Password: `argocd admin initial-password -n argocd`. The controller is a StatefulSet (`argocd-application-controller-0`). Applications are CRs stored in etcd, Redis is only a cache.

</details>

**Bonus:** Install the HA manifests on a three-node kind cluster and list what changes (Redis HA with HAProxy, replica counts, anti-affinity).

### Assignment 2. Your first Application from the CLI 🟢

**Objective:** Deploy from Git with an imperative `argocd app create` and understand each flag.

**Tasks**
1. Create a public Git repo with a folder `apps/nginx` containing a Deployment (2 replicas) and a Service.
2. Create an Application `nginx` with `argocd app create` pointing at that path, destination namespace `web`, project `default`.
3. Observe that it is `OutOfSync` and `Missing`. Sync it with `argocd app sync`.
4. Change the replica count in Git, push, and run `argocd app get nginx --refresh`. Sync again.
5. Run `argocd app diff nginx` before the second sync and read the output.

**Expected outcome / acceptance criteria**
- The app ends `Synced` and `Healthy` with the new replica count.
- You can explain why the first sync failed if the `web` namespace did not exist, and fix it with `--sync-option CreateNamespace=true`.

<details>
<summary>Hints</summary>

`argocd app create nginx --repo https://github.com/<you>/gitops-play.git --path apps/nginx --dest-server https://kubernetes.default.svc --dest-namespace web --sync-option CreateNamespace=true`. `argocd app diff` exits 1 when there is a difference, which is handy in scripts.

</details>

**Bonus:** Export the Application as YAML with `kubectl -n argocd get app nginx -o yaml`, strip the status, and commit it. That is your first declarative app.

### Assignment 3. Declarative Applications and reading status 🟢

**Objective:** Manage Applications as YAML and read sync and health correctly.

**Tasks**
1. Write `nginx-app.yaml` with `project`, `source`, `destination` and the `resources-finalizer.argocd.argoproj.io` finalizer, then `kubectl apply` it.
2. Break the Deployment on purpose in Git (an image tag that does not exist). Push and sync.
3. Record the app's sync status and health status. Find the failing resource in the UI tree and in `argocd app get`.
4. Fix the tag and confirm the app returns to `Healthy`.
5. Delete the Application with `kubectl delete` and confirm that the finalizer removed the workload too.

**Expected outcome / acceptance criteria**
- In step 3 the app is `Synced` but `Progressing` then `Degraded`, and you can explain why sync and health disagree.
- Step 5 leaves no resources behind in `web`.

<details>
<summary>Hints</summary>

Deployment health turns `Degraded` when `progressDeadlineSeconds` (600s by default) is exceeded; lower it to 60 for a faster demo. Without the finalizer, deleting the Application orphans its resources.

</details>

**Bonus:** Repeat step 5 without the finalizer and then adopt the orphaned resources with a new Application. Notice the tracking annotation or label Argo CD adds.

### Assignment 4. Automated sync, prune and self-heal 🟢

**Objective:** See exactly what each automation flag does.

**Tasks**
1. Enable `automated` sync without prune or selfHeal. Delete the Service from Git and push. Observe the result.
2. Enable `prune: true`. Observe the result.
3. Run `kubectl scale deploy nginx -n web --replicas=5`. Observe with selfHeal off.
4. Enable `selfHeal: true` and repeat step 3.
5. Add a ConfigMap with `kubectl create` directly in `web` (not in Git). Does Argo CD touch it?

**Expected outcome / acceptance criteria**
- A table with five rows: action, flag state, what Argo CD did.
- You can explain why prune is off by default and why the manually created ConfigMap is ignored (it is not tracked by the app).

<details>
<summary>Hints</summary>

Without prune, a deleted resource shows as `OutOfSync` with "requires pruning". `argocd app set nginx --sync-policy automated --auto-prune --self-heal`. Untracked resources only show up if you enable orphaned resource monitoring in the project.

</details>

**Bonus:** Enable `orphanedResources: {warn: true}` on the project and find the warning for the ConfigMap.

### Assignment 5. Deploy a Helm chart from a Helm repository 🟢

**Objective:** Use Argo CD as a Helm deployment tool and understand what it does differently.

**Tasks**
1. Create an Application for the `podinfo` chart from `https://stefanprodan.github.io/podinfo`, pinned to a specific chart version.
2. Override `replicaCount` and `ui.message` using `helm.valuesObject`.
3. Run `helm list -A` and explain the result.
4. Upgrade the chart version by editing `targetRevision` and watch the diff before syncing.
5. Change `targetRevision` to a semver range (for example `6.*`) and explain the risk.

**Expected outcome / acceptance criteria**
- podinfo runs with your message.
- You can explain why `helm list` is empty and what that means for `helm rollback`.

<details>
<summary>Hints</summary>

Use `chart: podinfo` instead of `path`. Argo CD runs `helm template` and applies the output, so no release Secret exists. A range means an unreviewed chart upgrade can land on the next refresh.

</details>

**Bonus:** Add a Helm `pre-upgrade` hook Job to a chart of your own and confirm Argo CD runs it as a `PreSync` hook.

### Assignment 6. Kustomize overlays as separate Applications 🟢

**Objective:** Model environments with Kustomize and one Application per environment.

**Tasks**
1. Create `apps/web/base` (Deployment, Service) and overlays `dev` and `prod` that change replicas, add a label, and set a different image tag.
2. Create two Applications, `web-dev` and `web-prod`, into namespaces `web-dev` and `web-prod`.
3. Use `spec.source.kustomize.images` on `web-dev` to override the tag without changing the overlay.
4. Run `argocd app manifests web-prod --source git` and compare with `kustomize build` locally.

**Expected outcome / acceptance criteria**
- Both apps are healthy with different replica counts and tags.
- You can explain when to put an override in the overlay vs in the Application spec (Git review vs ad hoc).

<details>
<summary>Hints</summary>

`kustomize: {images: ["nginx=nginx:1.27.2"]}`. Prefer overlays: overrides in the Application spec are easy to forget and do not show up when someone reads the repo.

</details>

**Bonus:** Set `kustomize.buildOptions: --enable-helm` in `argocd-cm` and use a `helmCharts` entry in a kustomization.

### Assignment 7. History, rollback and the GitOps way to undo 🟢

**Objective:** Know both rollback paths and why one is preferred.

**Tasks**
1. Make three commits that change the image tag of an app and sync after each.
2. Run `argocd app history` and roll back to the first entry with `argocd app rollback`.
3. Try the rollback with auto-sync enabled and record the error.
4. Undo the same change with `git revert` instead, and compare the end state of Git and the cluster in both approaches.

**Expected outcome / acceptance criteria**
- You captured the error `rollback cannot be initiated when auto-sync is enabled`.
- You can explain why `argocd app rollback` leaves Git and the cluster disagreeing, and why `git revert` is the standard in production.

<details>
<summary>Hints</summary>

`argocd app set web --sync-policy none` before rollback. After a CLI rollback the app shows `OutOfSync` against the branch head because Git still has the newest tag.

</details>

**Bonus:** Set `revisionHistoryLimit: 3` on the Application and observe how history is trimmed.

### Assignment 8. Private repositories and credential templates 🟢

**Objective:** Connect Argo CD to private Git repos the way a real team does.

**Tasks**
1. Make a repo private and observe the error the Application shows.
2. Add the repo with an SSH deploy key using `argocd repo add`, after adding GitHub's host key.
3. Switch to HTTPS with a fine-grained token, declared as a Secret with label `argocd.argoproj.io/secret-type: repository`.
4. Replace the per-repo Secret with a `repo-creds` credential template for the whole GitHub org or user.

**Expected outcome / acceptance criteria**
- Step 1 error text recorded (for example `authentication required` or `repository not found`).
- After step 4, a new private repo under the same owner works with no further configuration.

<details>
<summary>Hints</summary>

`ssh-keyscan github.com | argocd cert add-ssh --batch`. Credential templates match by URL prefix, so `https://github.com/<you>` covers every repo under it.

</details>

**Bonus:** Use a GitHub App instead of a token (`githubAppID`, `githubAppInstallationID`, `githubAppPrivateKey`) and explain why it is better for an organization.

### Assignment 9. Order resources with sync waves 🟢

**Objective:** Control apply order and see that waves wait on health.

**Tasks**
1. Create an app with a Namespace, a ConfigMap, a Deployment that needs the ConfigMap, and a Job that curls the Service.
2. Assign waves `-2`, `-1`, `0`, `1` respectively.
3. Make the Deployment slow to become ready (an `initContainer` with `sleep 30`) and watch the Job wait.
4. Remove the waves and re-sync from scratch. What breaks or races?

**Expected outcome / acceptance criteria**
- The sync operation details show resources applied wave by wave.
- You can explain that a wave only starts after the previous wave is healthy, and that resources without health checks count as healthy immediately.

<details>
<summary>Hints</summary>

`metadata.annotations: {argocd.argoproj.io/sync-wave: "-1"}`, always a quoted string. The default wave is 0. Watch with `argocd app get web --refresh` during the sync.

</details>

**Bonus:** Measure the delay between waves and find the environment variable that controls it (`ARGOCD_SYNC_WAVE_DELAY`).

### Assignment 10. Local accounts and a read-only user 🟢

**Objective:** Stop sharing `admin`.

**Tasks**
1. Add local accounts `viewer` (login only) and `ci` (apiKey only) in `argocd-cm`.
2. Set passwords and generate a token for `ci`.
3. In `argocd-rbac-cm`, give `viewer` read-only access and `ci` permission to sync apps in project `default`.
4. Verify with `argocd account can-i` as each user.
5. Disable the `admin` account.

**Expected outcome / acceptance criteria**
- `viewer` can see but not sync; `ci` can sync using only its token.
- You can still administer Argo CD after disabling `admin` (because you kept an admin role somewhere).

<details>
<summary>Hints</summary>

`accounts.viewer: login`, `accounts.ci: apiKey`, `admin.enabled: "false"`. `argocd account update-password --account viewer --current-password <admin pwd>`. `p, ci, applications, sync, default/*, allow`.

</details>

**Bonus:** Set `policy.default` to empty and confirm users with no mapping see nothing.

---

## 🟡 Intermediate

### Assignment 11. Migration hooks and failure hooks 🟡

**Objective:** Run database migrations safely in a GitOps sync.

**Tasks**
1. Deploy PostgreSQL in wave `-1` and an app in wave 0.
2. Add a `PreSync` Job that runs a migration (for example `psql -c 'CREATE TABLE IF NOT EXISTS ...'`) with `hook-delete-policy: BeforeHookCreation`.
3. Make the migration fail on purpose and confirm the app is not updated.
4. Add a `SyncFail` hook that posts to a webhook (use `https://webhook.site`) or logs a message.
5. Add a `PostSync` smoke test Job.

**Expected outcome / acceptance criteria**
- A failed migration leaves the old version running and triggers the SyncFail hook.
- Hooks are cleaned up according to their delete policies.

<details>
<summary>Hints</summary>

The PreSync hook runs before any Sync-phase resource, so on a brand-new install the database from wave -1 does not exist yet. Either run Postgres in a separate app, or make the migration a `Sync` hook in wave 1. Set `activeDeadlineSeconds` so a hung migration does not block forever.

</details>

**Bonus:** Make the migration idempotent and backward compatible (expand/contract) and explain why that matters for rollbacks.

### Assignment 12. Kill an OutOfSync loop 🟡

**Objective:** Reproduce and fix the most common Argo CD complaint.

**Tasks**
1. Deploy an app whose Deployment declares `replicas: 2`, then add an HPA with `minReplicas: 3`.
2. Enable auto-sync with selfHeal and observe the fight (`kubectl get deploy -w`).
3. Fix it in two ways: (a) remove `replicas` from the manifest, (b) keep it but add `ignoreDifferences` for `/spec/replicas` plus `RespectIgnoreDifferences=true`.
4. Add a MutatingWebhookConfiguration from cert-manager or similar and fix a `caBundle` diff with `jqPathExpressions`.
5. Try `ServerSideDiff=true` and note which false positives disappear.

**Expected outcome / acceptance criteria**
- The app stays `Synced` for 30 minutes with the HPA scaling freely.
- You can explain why `ignoreDifferences` alone does not stop a sync from resetting the field.

<details>
<summary>Hints</summary>

Without `RespectIgnoreDifferences=true`, Argo CD ignores the field in the diff but still applies the full manifest when something else triggers a sync. Use `argocd app diff` to confirm each fix.

</details>

**Bonus:** Configure the replicas ignore globally for all Deployments via `resource.customizations.ignoreDifferences.apps_Deployment` and discuss the risk.

### Assignment 13. Custom health checks in Lua 🟡

**Objective:** Make Argo CD understand a CRD's real health.

**Tasks**
1. Install cert-manager and create a `Certificate` from a self-signed issuer.
2. Write a Lua health check for `cert-manager.io/Certificate` that returns `Healthy` when the `Ready` condition is `True`, `Degraded` when `False`, and `Progressing` otherwise.
3. Test the script offline with `argocd admin settings resource-overrides health`.
4. Install it in `argocd-cm` and confirm the UI shows the right status for a good and a broken issuer.
5. Use the Certificate in wave -1 and an Ingress that needs its Secret in wave 0.

**Expected outcome / acceptance criteria**
- A broken issuer makes the app `Degraded` with the condition message, and wave 0 never starts.

<details>
<summary>Hints</summary>

Key: `resource.customizations.health.cert-manager.io_Certificate`. Check `obj.status ~= nil` before indexing. Argo CD ships a built-in check for many CRDs already; write your own anyway, then compare.

</details>

**Bonus:** Add a custom resource action in Lua (`resource.customizations.actions.<group>_<kind>`) that sets an annotation, and run it from the CLI.

### Assignment 14. Multi-tenant AppProjects with sync windows 🟡

**Objective:** Isolate two teams on one Argo CD.

**Tasks**
1. Create projects `team-a` and `team-b`, each allowed only its own repo and its own `team-x-*` namespaces.
2. Allow no cluster-scoped kinds except `Namespace`; blacklist `ResourceQuota`, `LimitRange` and `NetworkPolicy` in namespaces.
3. Try, as team-a, to deploy to `team-b-dev`, to deploy a ClusterRole, and to use team-b's repo. Record each error.
4. Add a deny sync window for `*-prod` apps every night 22:00 to 06:00, allowing manual sync.
5. Enable orphaned resource warnings on both projects.

**Expected outcome / acceptance criteria**
- Three distinct project-violation errors captured.
- A sync attempt during the window is blocked for automated sync and allowed manually.

<details>
<summary>Hints</summary>

`argocd proj windows add team-a --kind deny --schedule "0 22 * * *" --duration 8h --applications '*-prod' --manual-sync`. Set the schedule to the next few minutes to test it.

</details>

**Bonus:** Lock down the `default` project so it allows nothing (`sourceRepos: []`, `destinations: []`).

### Assignment 15. RBAC and SSO group mapping 🟡

**Objective:** Give people access through groups, not local users.

**Tasks**
1. Configure Dex with the GitHub connector (or run Keycloak in the cluster and use `oidc.config`).
2. Map a GitHub team (or Keycloak group) to `role:team-a-dev` that can view all team-a apps and sync only `*-dev`.
3. Add an explicit deny for syncing `team-a/*-prod`.
4. Write the policy to a file and test five decisions with `argocd admin settings rbac can`.
5. Set `policy.default: role:readonly` and then remove it; explain the effect.

**Expected outcome / acceptance criteria**
- A member of the group logs in through SSO and can sync dev but gets `permission denied` on prod.
- Your five offline RBAC tests match the live behaviour.

<details>
<summary>Hints</summary>

Set `url:` in `argocd-cm` to the external URL; the callback is `<url>/api/dex/callback`. `scopes: "[groups]"`. GitHub team groups look like `my-org:team-a`.

</details>

**Bonus:** Create a project role with a JWT token for CI and restrict it to `sync` on one app.

### Assignment 16. Bootstrap a cluster with app-of-apps 🟡

**Objective:** Go from an empty cluster to a full platform with one command.

**Tasks**
1. Create a `bootstrap/` folder with Applications for ingress-nginx, cert-manager, metrics-server and one demo app.
2. Create a root Application pointing at `bootstrap/`.
3. Add sync waves so CRD-providing apps come first, and add the Lua health check for `argoproj.io/Application` so waves really wait.
4. Delete the kind cluster, recreate it, install Argo CD and apply only the root app.

**Expected outcome / acceptance criteria**
- The rebuild in step 4 needs exactly two commands after Argo CD is installed: apply root, wait.
- You can explain why waves on Application resources do nothing without the custom health check.

<details>
<summary>Hints</summary>

Child Application manifests must be in the `argocd` namespace (unless apps-in-any-namespace is enabled). Give the root app `prune: true` so deleting a file removes the child app, and child apps the finalizer so their resources go too.

</details>

**Bonus:** Make Argo CD manage itself: add an Application for the Argo CD Helm chart to `bootstrap/` and upgrade Argo CD through a Git commit.

### Assignment 17. ApplicationSets with list and git generators 🟡

**Objective:** Stop copy-pasting Applications.

**Tasks**
1. Write an ApplicationSet with a list generator that creates `web-dev`, `web-staging`, `web-prod` with different namespaces and value files.
2. Rewrite it with a git directory generator over `apps/*` so each folder becomes an app.
3. Switch to `goTemplate: true` with `missingkey=error`, then break a key name and read the error in the ApplicationSet status.
4. Delete a folder and observe what happens to the generated Application and its resources. Then set `preserveResourcesOnDeletion: true` and repeat.
5. Try `applicationsSync: create-only` and edit a generated app by hand.

**Expected outcome / acceptance criteria**
- Adding a folder creates an app with no other change.
- You can explain every `applicationsSync` value and when you would use `create-only`.

<details>
<summary>Hints</summary>

`argocd appset get <name>` and `kubectl -n argocd get applicationset <name> -o yaml` show conditions. Use `exclude: true` on a directory entry to skip folders like `apps/_template`.

</details>

**Bonus:** Use `templatePatch` to add `automated` sync only for non-prod elements.

### Assignment 18. Sealed Secrets end to end, including key recovery 🟡

**Objective:** Keep secrets in Git safely and prove you can recover them.

**Tasks**
1. Install the Sealed Secrets controller through Argo CD.
2. Seal a database password for namespace `app` and deploy it as part of an Application.
3. Try to use the same SealedSecret in another namespace, then reseal it with `--scope namespace-wide` and retry.
4. Back up the sealing keys. Delete the cluster, rebuild it, restore the keys before the controller starts, and confirm the old SealedSecrets decrypt.
5. Rotate the actual password: generate a new value, reseal, commit, and confirm the pod restarts with it.

**Expected outcome / acceptance criteria**
- Step 3 shows a decryption error under strict scope and success under namespace-wide.
- Step 4 decrypts secrets that were sealed on the old cluster.

<details>
<summary>Hints</summary>

Keys are Secrets labelled `sealedsecrets.bitnami.com/sealed-secrets-key` in the controller namespace. Apply the backup before installing the controller (or restart the controller after applying). Pods only see new env values after a restart: use a checksum annotation or Reloader.

</details>

**Bonus:** Replace Sealed Secrets with External Secrets Operator backed by a Vault dev server and write down the trade-offs.

### Assignment 19. CI that commits to the config repo, promotion by PR 🟡

**Objective:** Build the classic two-repo GitOps flow.

**Tasks**
1. Create an app repo and a config repo with `envs/{dev,staging,prod}`.
2. Write a CI pipeline (GitHub Actions, GitLab CI or Jenkins) that builds, pushes `sha-<short>` and commits the tag to `envs/dev` using a token scoped to the config repo only.
3. Handle concurrent CI runs: make the push step retry with `git pull --rebase`.
4. Write a `promote.sh <from> <to>` script or workflow that opens a PR copying the tag forward.
5. Protect `envs/prod/**` with CODEOWNERS and branch protection.

**Expected outcome / acceptance criteria**
- A push to the app repo reaches dev with no human action.
- Prod changes are only possible through an approved PR.
- CI never has a kubeconfig.

<details>
<summary>Hints</summary>

`yq -i '.image.tag = strenv(TAG)' envs/dev/values.yaml`. Use a GitHub App token or deploy key in real setups. Guard against commit loops: the config repo must not trigger the app pipeline.

</details>

**Bonus:** Replace the shell promotion with Kargo and compare.

### Assignment 20. Notifications on failure and degradation 🟡

**Objective:** Find out about failed syncs before users do.

**Tasks**
1. Configure the notifications controller with a webhook service (webhook.site) or Slack.
2. Define templates and triggers for `on-sync-failed`, `on-health-degraded` and `on-deployed`.
3. Subscribe one Application by annotation and one whole project through the AppProject annotation.
4. Cause each event and confirm exactly one message per event (use `oncePer`).
5. Include the commit SHA, author and a link to the app in the message.

**Expected outcome / acceptance criteria**
- Three distinct notifications received with useful content and no duplicates for the same revision.

<details>
<summary>Hints</summary>

`notifications.argoproj.io/subscribe.on-sync-failed.webhook.<name>`. `oncePer: app.status.sync.revision` stops repeats. Template fields: `{{.app.metadata.name}}`, `{{.app.status.sync.revision}}`, `{{(call .repo.GetCommitMetadata .app.status.sync.revision).Author}}`, and `context.argocdUrl`.

</details>

**Bonus:** Use the GitHub service to set a commit status on the config repo commit when a sync succeeds.

---

## 🔴 Advanced

### Assignment 21. Multi-cluster fleet with cluster and matrix generators 🔴

**Objective:** Manage several clusters from one Argo CD.

**Tasks**
1. Create three kind clusters: `hub` (runs Argo CD), `stage-1` and `prod-1`. Make the API servers reachable from the hub (use the Docker network IPs, not 127.0.0.1).
2. Register both spokes with `argocd cluster add` and labels `env=staging` / `env=prod`, then declare the same registration as cluster Secrets.
3. Write a matrix ApplicationSet: clusters (by label) times git directories (`platform/*`), deploying metrics-server and a demo app everywhere.
4. Use cluster Secret labels to opt a cluster in and out of an add-on.
5. Use the RollingSync strategy so staging syncs before prod.

**Expected outcome / acceptance criteria**
- Adding a label to a cluster Secret deploys or removes an add-on with no other change.
- Prod apps update only after staging apps are healthy.

<details>
<summary>Hints</summary>

`kind get kubeconfig --internal --name stage-1` gives a server URL reachable from other kind containers. RollingSync requires `--enable-progressive-syncs` on the ApplicationSet controller (`applicationsetcontroller.enable.progressive.syncs: true` in `argocd-cmd-params-cm`) and apps with automated sync configured by the strategy.

</details>

**Bonus:** Replace the bearer-token cluster config with exec-based auth (for EKS, `awsAuthConfig` with an IAM role) and explain the credential lifecycle.

### Assignment 22. Progressive delivery with Argo Rollouts and Prometheus 🔴

**Objective:** Release automatically based on real metrics.

**Tasks**
1. Install kube-prometheus-stack, ingress-nginx and Argo Rollouts through Argo CD.
2. Convert a Deployment to a Rollout with canary steps 10%, 30%, 60% and NGINX traffic routing.
3. Write an AnalysisTemplate using a Prometheus success-rate query (from ingress-nginx metrics) with `successCondition: result[0] >= 0.99`.
4. Release a version that returns 500 on 20% of requests and confirm the automatic abort.
5. Implement a blue-green Rollout for a second service with `prePromotionAnalysis` and `autoPromotionEnabled: false`.

**Expected outcome / acceptance criteria**
- The bad version is aborted with no more than 10% of traffic affected, and stable keeps serving.
- The Argo CD app shows `Suspended` during manual pauses and `Degraded` after abort.

<details>
<summary>Hints</summary>

Generate load with a `hey` or `k6` Job. With NGINX routing Rollouts manages a canary Ingress with `nginx.ingress.kubernetes.io/canary-weight`. Do not let Argo CD fight over that canary Ingress: it is created by Rollouts, not in Git.

</details>

**Bonus:** Add a background analysis that runs for the whole rollout, and an `Experiment` that runs baseline and canary side by side.

### Assignment 23. Multiple sources and Image Updater with Git write-back 🔴

**Objective:** Combine upstream charts with your own values and automate tag bumps without CI writing to Git.

**Tasks**
1. Deploy an upstream chart (for example `ingress-nginx`) with `sources`: the chart from its Helm repo and the values file from your config repo via `ref: values`.
2. Pin the chart version and add Renovate (or Dependabot) to open PRs for new chart versions.
3. Install Argo CD Image Updater with credentials for your registry.
4. Annotate a dev Application to track `semver` tags matching `^1\.\d+\.\d+$` with Git write-back to a separate branch.
5. Push a new image tag and confirm the commit Image Updater makes, and that Argo CD syncs it.

**Expected outcome / acceptance criteria**
- The multi-source app renders the upstream chart with your values; changing only the values file triggers a sync.
- Image Updater commits `.argocd-source-<app>.yaml` and never touches prod.

<details>
<summary>Hints</summary>

`valueFiles: [$values/platform/ingress/values.yaml]`. Image Updater only supports Helm and Kustomize sources, and Git write-back needs push credentials (`write-back-method: git:secret:argocd/git-creds`). Check its logs for `Successfully updated image`.

</details>

**Bonus:** Add SOPS-encrypted values to the same app through a Config Management Plugin sidecar.

### Assignment 24. Disaster recovery: delete Argo CD and rebuild it 🔴

**Objective:** Prove your platform survives losing its control plane.

**Tasks**
1. Start from the hands-on lab state (or similar: root app, ApplicationSet, private repo, Sealed Secrets).
2. Take a backup with `argocd admin export`, and separately back up the Sealed Secrets keys.
3. Delete the `argocd` namespace with `--cascade=orphan`-style care so workloads keep running, and confirm the apps keep serving traffic.
4. Reinstall Argo CD, restore repo credentials and cluster Secrets, and apply the root app.
5. Confirm all apps come back `Synced` without redeploying pods (no restarts), and write a runbook with timings.

**Expected outcome / acceptance criteria**
- Zero application downtime during the whole exercise.
- Recovery time measured, and the runbook lists exactly what was not in Git.

<details>
<summary>Hints</summary>

Removing the `resources-finalizer.argocd.argoproj.io` finalizer from every Application before deleting the namespace prevents cascade deletion of workloads. Argo CD adopts existing resources on the first sync because the manifests match; pods restart only if the spec differs.

</details>

**Bonus:** Run the same exercise with a hub-and-spoke setup and restore to a brand new hub cluster.

### Assignment 25. Scale Argo CD to hundreds of apps with sharding 🔴

**Objective:** Find and remove Argo CD's bottlenecks.

**Tasks**
1. Generate 300 small Applications with an ApplicationSet (a list or git generator over a script-generated set of folders) across 3 kind clusters.
2. Install Prometheus and the Argo CD dashboards; record reconcile duration (`argocd_app_reconcile`), workqueue depth and repo-server request latency.
3. Shard the controller to 3 replicas with `ARGOCD_CONTROLLER_REPLICAS`, try `round-robin` and `consistent-hashing`, and check distribution with `argocd admin cluster shards`.
4. Tune `controller.status.processors`, `controller.operation.processors` and `reposerver.parallelism.limit`, and scale the repo-server to 3.
5. Configure a Git webhook (use a tunnel like ngrok for the lab) plus `manifest-generate-paths`, then raise `timeout.reconciliation`.

**Expected outcome / acceptance criteria**
- A before/after table of reconcile time, memory per controller pod and Git request rate.
- A one-page recommendation for a 1,000-app installation.

<details>
<summary>Hints</summary>

Sharding is per cluster: three clusters spread over three shards; a single huge cluster does not benefit. Set the env var to the same number as StatefulSet replicas. Exclude high-churn kinds you do not deploy with `resource.exclusions`.

</details>

**Bonus:** Compare with a Flux-per-cluster design for the same fleet and argue which one you would run, including UI, tenancy and blast radius.

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
