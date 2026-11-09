# GitOps & ArgoCD — Core Concepts

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

> Principles, Argo CD internals, the Application CRD, sync, health, diffing, tenancy, ApplicationSets, Helm and Kustomize, progressive delivery, secrets, scaling, DR, best practices and rapid-fire commands · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to GitOps & ArgoCD](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [GitOps Principles & Delivery Models](#gitops-principles--delivery-models) — Q1–Q18
- [Argo CD Architecture & Internals](#argo-cd-architecture--internals) — Q19–Q42
- [The Application CRD & Sync Policies](#the-application-crd--sync-policies) — Q43–Q66
- [Sync Waves, Hooks, Health & Diffing](#sync-waves-hooks-health--diffing) — Q67–Q92
- [AppProjects, RBAC & SSO](#appprojects-rbac--sso) — Q93–Q110
- [ApplicationSets, App-of-Apps & Multi-Cluster](#applicationsets-app-of-apps--multi-cluster) — Q111–Q132
- [Helm, Kustomize & Multiple Sources](#helm-kustomize--multiple-sources) — Q133–Q144
- [Progressive Delivery, Image Updater & Notifications](#progressive-delivery-image-updater--notifications) — Q145–Q160
- [Secrets in GitOps](#secrets-in-gitops) — Q161–Q170
- [Repo Structure, Flux, Scaling & DR](#repo-structure-flux-scaling--dr) — Q171–Q190
- [Rapid-Fire Commands](#rapid-fire-commands) — Q191–Q200

---

## GitOps Principles & Delivery Models

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. What is GitOps, in one or two sentences? 🟢

<details>
<summary><b>Answer</b></summary>

GitOps is an operating model where the desired state of your systems is declared in Git, and an agent running next to the system continuously pulls that state and reconciles reality to match it. Git becomes the single source of truth and the audit log; deployments happen by merging commits, not by running commands against the cluster.

</details>

### Q2. What are the four OpenGitOps principles? 🟢

<details>
<summary><b>Answer</b></summary>

The CNCF OpenGitOps project defines them as:

1. **Declarative**: the desired state is expressed declaratively (YAML, Helm values, Kustomize), not as a sequence of steps.
2. **Versioned and immutable**: the desired state is stored in a way that enforces immutability and keeps a complete version history (Git commits).
3. **Pulled automatically**: software agents pull the desired state from the source.
4. **Continuously reconciled**: agents observe the actual state and attempt to apply the desired state, forever, not once per pipeline run.

</details>

### Q3. Explain push-based vs pull-based deployment. 🟢

<details>
<summary><b>Answer</b></summary>

**Push**: the CI/CD pipeline holds cluster credentials and runs `kubectl apply` or `helm upgrade` at the end of a run. It acts only when the pipeline runs.

**Pull**: an agent inside (or next to) the cluster, such as Argo CD or Flux, watches Git and applies changes itself. CI only builds artifacts and updates Git.

Pull removes cluster-admin credentials from CI, corrects drift continuously, and works for clusters behind firewalls because the connection is outbound to Git.

</details>

### Q4. Is "we keep our Kubernetes YAML in Git and CI runs `kubectl apply`" GitOps? 🟢

<details>
<summary><b>Answer</b></summary>

Not by the OpenGitOps definition. It is declarative and versioned, but it is pushed, not pulled, and nothing reconciles between pipeline runs. If someone runs `kubectl edit` in production, the change stays until the next deploy, and nobody is told. It is "infrastructure as code with CI", a good step, but you lose drift detection and you keep powerful credentials in CI.

</details>

### Q5. What are the main benefits of GitOps that you would pitch to a team? 🟢

<details>
<summary><b>Answer</b></summary>

- **Audit and compliance**: every change is a commit with author, review and timestamp.
- **Rollback**: `git revert` restores a known state.
- **Security**: CI has no cluster credentials; the agent's access is scoped and lives in the cluster.
- **Drift detection and correction**: manual changes are visible or reverted.
- **Disaster recovery**: a new cluster converges to the declared state from Git.
- **Developer experience**: deploying is a pull request, the same workflow as code.

</details>

### Q6. What are the honest downsides or pain points of GitOps? 🟡

<details>
<summary><b>Answer</b></summary>

- Secrets need extra tooling (Sealed Secrets, External Secrets, SOPS).
- Imperative tasks (one-off migrations, data fixes) do not fit naturally; you model them as hooks or Jobs.
- Promotion between environments is not built in; you build it with PRs, scripts, Image Updater or Kargo.
- "Commit noise": CI bot commits for every image tag.
- Debugging moves from pipeline logs to controller state, which teams must learn.
- Ordering and dependencies across apps need sync waves or app-of-apps design.

</details>

### Q7. What does "drift" mean in GitOps, and how is it handled? 🟢

<details>
<summary><b>Answer</b></summary>

Drift is any difference between the desired state in Git and the live state in the cluster, caused by `kubectl edit`, a controller, a webhook or a failed partial apply. Argo CD detects it on every reconcile and marks the app `OutOfSync`. With `selfHeal: true` it re-applies Git automatically; without it, it only reports, and a human syncs. Legitimate drift (HPA replicas, injected sidecars) is excluded with `ignoreDifferences`.

</details>

### Q8. What is the difference between "desired state" and "live state"? 🟢

<details>
<summary><b>Answer</b></summary>

Desired state is what the rendered manifests from Git say should exist (after Helm or Kustomize rendering). Live state is what the Kubernetes API currently returns for those objects. Argo CD caches live state through watches, normalizes both sides (removing defaults and ignored fields), diffs them, and that diff drives the sync status.

</details>

### Q9. Why is the pull model considered more secure? 🟡

<details>
<summary><b>Answer</b></summary>

In push, every pipeline (and anyone who can edit a pipeline) effectively has cluster-admin. A leaked CI secret means a compromised cluster. In pull, the credential to change the cluster lives only inside the cluster (the Argo CD controller's ServiceAccount, or scoped cluster Secrets on a hub). CI needs only write access to a Git repo, which is protected by branch rules and review. Inbound access to the API server is not required, so private clusters stay private.

</details>

### Q10. Can GitOps be used for things other than Kubernetes? 🟡

<details>
<summary><b>Answer</b></summary>

Yes, as long as there is a declarative API and a reconciling agent. Examples: Crossplane or AWS Controllers for Kubernetes (ACK) let Argo CD manage cloud resources as CRs; the Terraform controller (tofu-controller) for Flux reconciles Terraform; Ansible pull mode is a weak form of it. In practice most GitOps tooling targets Kubernetes because its API is declarative and watchable by design.

</details>

### Q11. How does GitOps change the role of the CI pipeline? 🟢

<details>
<summary><b>Answer</b></summary>

CI stops being a deploy tool. It tests, builds, scans and pushes an immutable artifact, then proposes or makes a change in Git (bump an image tag, open a PR). The GitOps agent does the deploy. This splits responsibilities: CI owns "what is the artifact", Git owns "what should run where", and the agent owns "make it so".

</details>

### Q12. What is reconciliation, and how often does Argo CD reconcile? 🟢

<details>
<summary><b>Answer</b></summary>

Reconciliation is the loop that compares desired and live state and acts on the difference. Argo CD refreshes every Application every `timeout.reconciliation` (180 seconds by default, with jitter), immediately on a Git webhook, and whenever a watched live resource changes (live changes update the diff without re-rendering Git). A hard refresh also clears the manifest cache.

</details>

### Q13. Why should environment state be declared rather than scripted? 🟡

<details>
<summary><b>Answer</b></summary>

A script describes how to get from one state to another and depends on the starting point; running it twice or from an unexpected state can break things. A declaration describes the end state, so a controller can compute the difference from any starting point, re-run safely (idempotence), detect drift, and rebuild from nothing. That is what makes continuous reconciliation and DR possible.

</details>

### Q14. What does "Git as the single source of truth" mean in practice, and where does it break down? 🟡

<details>
<summary><b>Answer</b></summary>

It means no change reaches the cluster except through Git: no `kubectl apply` by hand, no parameter overrides in the UI, no Helm installs from laptops. It breaks down for: secret values (Git holds references or ciphertext), runtime-generated state (HPA replica counts, operator-created objects), and bootstrap (something has to install the agent first). Good teams document those exceptions explicitly.

</details>

### Q15. What is the difference between GitOps and "ClickOps" or imperative ops? 🟢

<details>
<summary><b>Answer</b></summary>

ClickOps means changing systems through consoles and UIs; imperative ops means running commands by hand. Both leave no reviewable record of intent, are hard to reproduce, and drift silently. GitOps records intent as code, gets it reviewed, and has a machine apply it repeatedly. The cluster state is reproducible from the repo.

</details>

### Q16. Name the main GitOps tools in the ecosystem and what each is known for. 🟢

<details>
<summary><b>Answer</b></summary>

- **Argo CD**: CNCF graduated, Application-centric, strong UI, SSO and RBAC, ApplicationSets, hub-and-spoke multi-cluster.
- **Flux**: CNCF graduated, a toolkit of controllers (source, kustomize, helm, notification, image automation), CLI-first, usually one per cluster.
- **Argo Rollouts / Flagger**: progressive delivery (canary, blue-green) on top of either.
- **Kargo**: promotion orchestration between environments.
- **Sealed Secrets, External Secrets Operator, SOPS**: secrets handling.

</details>

### Q17. In GitOps, what plays the role of "deployment history"? 🟡

<details>
<summary><b>Answer</b></summary>

The Git log of the config repo is the authoritative history: who changed what, when, reviewed by whom. Argo CD also keeps a short sync history per Application (`argocd app history`, limited by `revisionHistoryLimit`, default 10) listing revisions and sync times. For audit we rely on Git plus Argo CD's Kubernetes Events and notifications, not the in-app history.

</details>

### Q18. What is the "rendered manifests" pattern, and why do some teams adopt it? 🔴

<details>
<summary><b>Answer</b></summary>

Instead of Argo CD rendering Helm or Kustomize at sync time, CI (or Argo CD's Source Hydrator feature in newer releases) renders the final plain YAML and commits it to an environment branch or folder. Argo CD then deploys plain manifests.

Benefits: the PR diff shows exactly what changes in the cluster (a chart bump that silently changes 40 objects becomes visible), rendering is reproducible, and the repo-server does almost no work. Costs: more commits, a rendering pipeline to maintain, and drift between the source and rendered branches if the pipeline fails.

</details>

---

## Argo CD Architecture & Internals

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q19. What are the main components of Argo CD? 🟢

<details>
<summary><b>Answer</b></summary>

- **argocd-server**: gRPC/REST API, web UI, webhook receiver, auth.
- **argocd-repo-server**: clones repos and renders manifests (Helm, Kustomize, plain YAML, plugins).
- **argocd-application-controller**: compares desired vs live state, syncs, assesses health (StatefulSet).
- **argocd-redis**: cache for manifests, app state and cluster info.
- **argocd-dex-server**: optional SSO broker (GitHub, LDAP, SAML, OIDC connectors).
- **argocd-applicationset-controller**: generates Applications from ApplicationSets.
- **argocd-notifications-controller**: sends notifications based on triggers.

</details>

### Q20. What does the argocd-repo-server do, and what does it not do? 🟢

<details>
<summary><b>Answer</b></summary>

It clones or fetches Git repositories (and Helm/OCI charts), resolves the revision to a commit SHA, and generates Kubernetes manifests by running `helm template`, `kustomize build`, reading plain YAML/Jsonnet, or calling Config Management Plugins. It caches results keyed by commit and parameters. It does **not** talk to target clusters and does not apply anything; it is a stateless rendering service.

</details>

### Q21. What does the application controller do on each reconcile? 🟡

<details>
<summary><b>Answer</b></summary>

1. Asks the repo-server for the desired manifests at the resolved revision.
2. Reads live state from its in-memory cluster cache (built from watches on every resource kind in each managed cluster).
3. Normalizes and diffs desired vs live, applying `ignoreDifferences`.
4. Computes sync status and health (built-in and Lua checks).
5. If automated sync applies, or a user requested a sync, runs the sync operation (phases, waves, hooks) with `kubectl apply` semantics.
6. Writes the result into the Application's `.status`.

</details>

### Q22. What is stored in Redis, and what happens if Redis is lost? 🟡

<details>
<summary><b>Answer</b></summary>

Redis holds caches: generated manifests, app resource trees, cluster info, and some session-related data. It is not the source of truth; Applications, Projects, settings and credentials live in Kubernetes objects. If Redis is lost, Argo CD regenerates the cache. You may see a brief burst of load on the repo-server and messages such as `cache: key is missing` until the cache refills. That is why many teams run Redis without persistence.

</details>

### Q23. Why is the application controller a StatefulSet rather than a Deployment? 🔴

<details>
<summary><b>Answer</b></summary>

For sharding. Each replica needs a stable ordinal (`argocd-application-controller-0`, `-1`...) to know which clusters it owns. With `ARGOCD_CONTROLLER_REPLICAS=N`, each pod computes its shard from its ordinal and processes only clusters assigned to that shard. A Deployment's random pod names would not give that stable identity. (Newer releases also offer dynamic cluster distribution, which runs the controller as a Deployment and stores assignments in a ConfigMap.)

</details>

### Q24. Where does Argo CD keep its configuration? 🟢

<details>
<summary><b>Answer</b></summary>

In Kubernetes objects in the `argocd` namespace:

- `argocd-cm`: general settings (URL, SSO, resource customizations, tracking method, reconciliation timeout).
- `argocd-rbac-cm`: RBAC policy.
- `argocd-cmd-params-cm`: component flags (processors, timeouts, insecure mode).
- `argocd-secret`: server secret key, admin password hash, webhook secrets.
- `argocd-tls-certs-cm`, `argocd-ssh-known-hosts-cm`, `argocd-gpg-keys-cm`.
- Secrets labelled `argocd.argoproj.io/secret-type: repository|repo-creds|cluster`.
- `Application`, `AppProject`, `ApplicationSet` custom resources.

</details>

### Q25. How does the Argo CD CLI talk to the server? 🟢

<details>
<summary><b>Answer</b></summary>

Over gRPC (HTTP/2) to `argocd-server`, with a token from `argocd login`. If a proxy or load balancer does not support HTTP/2 end to end, use `--grpc-web`. For clusters where you do not run the API server at all, `argocd login --core` makes the CLI talk directly to the Kubernetes API using your kubeconfig and Kubernetes RBAC.

</details>

### Q26. What is Argo CD "core" mode? 🟡

<details>
<summary><b>Answer</b></summary>

A minimal install (`core-install.yaml`) with only the application controller, repo-server, Redis and the ApplicationSet controller: no API server, UI, Dex or Argo CD RBAC. Users interact through `kubectl` and `argocd --core`, and access is governed by Kubernetes RBAC on the Application CRs. It suits platform teams that want headless GitOps per cluster, similar to Flux's model. `argocd admin dashboard` can run the UI locally when needed.

</details>

### Q27. How does Argo CD know which live resources belong to an Application? 🟡

<details>
<summary><b>Answer</b></summary>

Through resource tracking. Argo CD 3.x defaults to **annotation** tracking: it writes `argocd.argoproj.io/tracking-id: <app>:<group>/<kind>:<namespace>/<name>` on each resource. Older versions defaulted to the `app.kubernetes.io/instance` label, which clashed with Helm charts that set the same label and was truncated at 63 characters. Set the method with `application.resourceTrackingMethod` in `argocd-cm` (`label`, `annotation`, `annotation+label`).

</details>

### Q28. What is the gitops-engine? 🔴

<details>
<summary><b>Answer</b></summary>

`gitops-engine` is the Go library extracted from Argo CD that implements the core: the cluster cache (watch-based live state for all kinds), diffing with normalization, health assessment, and the sync engine (phases, waves, hooks, pruning). Argo CD's controller is built on it. Knowing it exists helps when reading controller logs and issues, which often reference cluster cache sync, resource versions and "watch" restarts.

</details>

### Q29. How does Argo CD connect to the cluster it is installed in, and to remote clusters? 🟢

<details>
<summary><b>Answer</b></summary>

The local cluster is `https://kubernetes.default.svc` (named `in-cluster`), reached with the controller's own ServiceAccount. Remote clusters are registered as Secrets labelled `argocd.argoproj.io/secret-type: cluster` containing the API server URL and credentials (bearer token, client certs, or exec/IAM auth such as `awsAuthConfig`). `argocd cluster add <context>` creates an `argocd-manager` ServiceAccount with a ClusterRole in the target cluster and stores its token in that Secret.

</details>

### Q30. How do Git webhooks make Argo CD faster? 🟢

<details>
<summary><b>Answer</b></summary>

Without webhooks, Argo CD polls every 180 seconds. A webhook from GitHub, GitLab, Bitbucket or Gogs to `https://argocd.example.com/api/webhook` tells argocd-server which repo and branch changed, and it immediately refreshes the matching Applications. Secure it with a shared secret (`webhook.github.secret` in `argocd-secret`). With webhooks in place, you can raise the polling interval and cut Git API load.

</details>

### Q31. What happens step by step when you click "Sync" in the UI? 🟡

<details>
<summary><b>Answer</b></summary>

1. The UI calls argocd-server, which checks RBAC (`applications, sync, project/app`).
2. The server writes an `operation` field into the Application CR.
3. The controller picks up the operation, gets manifests from the repo-server for the target revision, and computes the diff.
4. It runs the sync: PreSync hooks, then Sync-phase resources by wave (applying with kubectl semantics, pruning if requested), then PostSync hooks, waiting for health between waves.
5. It records `status.operationState` (phase `Succeeded`/`Failed`, per-resource results) and adds an entry to `status.history`.

</details>

### Q32. What is a Config Management Plugin (CMP)? 🔴

<details>
<summary><b>Answer</b></summary>

A way to render manifests with a tool Argo CD does not support natively (cdk8s, Tanka beyond Jsonnet, SOPS decryption, custom scripts). Since 2.4 the supported form is a **sidecar** on the repo-server running `argocd-cmp-server` with a `plugin.yaml` (`ConfigManagementPlugin`) that defines `init` and `generate` commands and optional `discover` rules. The generate command must print YAML or JSON manifests on stdout. Sidecars isolate plugin tooling and credentials from the main repo-server.

</details>

### Q33. What is the role of Dex, and when would you skip it? 🟡

<details>
<summary><b>Answer</b></summary>

Dex is an OIDC broker bundled with Argo CD. It lets Argo CD log users in through providers that are not OIDC themselves (GitHub OAuth, LDAP, SAML) and normalizes groups claims. If your IdP already speaks OIDC (Okta, Entra ID, Keycloak, Google), configure `oidc.config` in `argocd-cm` directly and disable Dex: one fewer component and a direct token flow.

</details>

### Q34. How does Argo CD cache manifests, and when does a hard refresh matter? 🟡

<details>
<summary><b>Answer</b></summary>

The repo-server caches generated manifests in Redis, keyed by repo, resolved commit SHA and the app's rendering parameters, for 24 hours by default (`--repo-cache-expiration`). A normal refresh re-resolves the revision and re-diffs, reusing the cache if the SHA did not change. A **hard refresh** invalidates the cache and re-renders. You need it when rendering depends on something outside the commit: a Helm chart dependency at the same version that was republished, a plugin reading external data, or a cached error.

</details>

### Q35. What is the ApplicationSet controller's relationship to the application controller? 🟡

<details>
<summary><b>Answer</b></summary>

They are independent. The ApplicationSet controller only creates, updates and deletes `Application` CRs based on generators and templates. It never deploys anything. The application controller then reconciles those Applications like any other. So an ApplicationSet problem shows up as missing or wrong Applications, while a deployment problem shows up on the Application itself.

</details>

### Q36. Which component needs the most memory, and why? 🔴

<details>
<summary><b>Answer</b></summary>

Usually the application controller, because it keeps an in-memory cache of every watched resource in every managed cluster (not just the ones Argo CD deploys), plus resource trees per app. Big clusters with many ConfigMaps, Events-like CRDs or high-churn custom resources inflate it. The repo-server can spike too during heavy Helm renders or with large monorepos. Mitigations: `resource.exclusions` for kinds you do not manage, sharding, and `reposerver.parallelism.limit`.

</details>

### Q37. What are "apps in any namespace"? 🔴

<details>
<summary><b>Answer</b></summary>

By default Application CRs must live in the `argocd` namespace. Since 2.5 you can allow them in other namespaces by setting `application.namespaces` (for example `team-*`) in `argocd-cmd-params-cm` for the server and controller, and listing the allowed namespaces in each AppProject's `sourceNamespaces`. Teams can then manage their own Application objects with Kubernetes RBAC in their namespace, while the project still restricts what those apps can deploy. RBAC objects become `<project>/<namespace>/<app>`.

</details>

### Q38. How does the controller assess whether a sync succeeded? 🟡

<details>
<summary><b>Answer</b></summary>

A sync operation succeeds when every task applied without error and every hook completed successfully. Health is assessed separately: after the operation, resources may still be `Progressing`. Between waves the controller waits for health, but the final operation phase reflects apply results, not long-term health. That is why you use `argocd app wait --health` in pipelines rather than trusting `Succeeded` alone.

</details>

### Q39. What ports and protocols does argocd-server expose, and how do you put it behind an ingress? 🟡

<details>
<summary><b>Answer</b></summary>

It serves HTTPS (UI, REST and gRPC on the same port, 8080 in the container, Service ports 80 and 443). Behind an ingress you either pass TLS through (SSL passthrough, so gRPC works natively), or terminate TLS at the ingress and run the server with `server.insecure: true`. When terminating, gRPC needs an HTTP/2-capable path (for example a separate gRPC ingress) or clients use `--grpc-web`. Also set `url` in `argocd-cm` to the external URL for SSO callbacks.

</details>

### Q40. What does `argocd-cmd-params-cm` control, and how is it different from `argocd-cm`? 🟡

<details>
<summary><b>Answer</b></summary>

`argocd-cmd-params-cm` maps to command-line flags and environment variables of the components: `controller.status.processors`, `controller.operation.processors`, `server.insecure`, `reposerver.parallelism.limit`, `application.namespaces`, `controller.sharding.algorithm`. Components read it at startup, so you restart them after changes. `argocd-cm` holds runtime settings (SSO, resource customizations, URL, tracking method) that are mostly picked up live.

</details>

### Q41. How does Argo CD watch so many resource kinds without polling the API server? 🔴

<details>
<summary><b>Answer</b></summary>

On startup per cluster, the controller discovers all API resources and starts a watch (list then watch) for each kind it is allowed to see, except excluded ones. Updates stream into the cluster cache, which maintains parent-child relationships via ownerReferences to build resource trees. When a watched object belonging to an app changes, the app is queued for a refresh. That is efficient for reads but costly in memory and initial sync time on large clusters, which is why `resource.exclusions` and `resource.inclusions` exist.

</details>

### Q42. What are status processors and operation processors? 🔴

<details>
<summary><b>Answer</b></summary>

They are worker pools in the application controller. **Status processors** (`controller.status.processors`, default 20) handle refresh and comparison: rendering requests, diffing, health. **Operation processors** (`controller.operation.processors`, default 10) execute syncs. If many apps are queued for refresh, comparison latency grows; if many syncs run at once (big release day), operations queue. The usual guidance is roughly 50 status and 25 operation processors per 1,000 apps, watched against CPU and workqueue depth metrics.

</details>

---

## The Application CRD & Sync Policies

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q43. Walk through the key fields of an Argo CD `Application`. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-prod
  namespace: argocd
spec:
  project: payments              # AppProject that constrains it
  source:
    repoURL: https://github.com/org/cfg.git
    targetRevision: main         # branch, tag, SHA (or chart version)
    path: apps/payments/overlays/prod
  destination:
    server: https://kubernetes.default.svc   # or name: prod-eu
    namespace: payments
  syncPolicy:
    automated: {prune: true, selfHeal: true}
```

`source` says what to render, `destination` says where, `project` says what is allowed, and `syncPolicy` says how changes are applied.

</details>

### Q44. What values can `targetRevision` take? 🟢

<details>
<summary><b>Answer</b></summary>

For Git: a branch (`main`), a tag (`v1.4.2`), a full or short commit SHA, or `HEAD` (the default branch). For Helm repositories: an exact chart version (`65.1.1`) or a semver constraint (`65.*`, `>=1.2.0 <2.0.0`). Branches and ranges move; tags and SHAs are fixed. In production we pin tags or SHAs for anything shared, and use branches for dev.

</details>

### Q45. What is the difference between `destination.server` and `destination.name`? 🟢

<details>
<summary><b>Answer</b></summary>

`server` is the API server URL (`https://kubernetes.default.svc`); `name` is the cluster name from the cluster Secret (`in-cluster`, `prod-eu`). Use one, not both. `name` is friendlier and survives an endpoint change (for example an EKS cluster recreated with a new URL, as long as you update the Secret), so it is the better choice in ApplicationSets and app-of-apps.

</details>

### Q46. What does `syncPolicy.automated` do, and what are its sub-fields? 🟢

<details>
<summary><b>Answer</b></summary>

It makes Argo CD sync automatically whenever the app is `OutOfSync` due to a new Git revision. Sub-fields:

- `prune` (default false): delete resources no longer in Git.
- `selfHeal` (default false): also sync when live state drifts without a new commit.
- `allowEmpty` (default false): permit syncs that would leave the app with zero resources, a guard against an accidentally emptied path deleting everything.

Automated sync does not retry the same failed revision on its own unless `retry` is set.

</details>

### Q47. Why is `prune` disabled by default? 🟢

<details>
<summary><b>Answer</b></summary>

Pruning deletes live resources. A bad refactor (renamed folder, broken Kustomize path, wrong branch) would otherwise delete production workloads, PVCs or namespaces on the next sync. Making it opt-in forces a conscious decision. Even with prune on, protect critical objects with `argocd.argoproj.io/sync-options: Prune=false` and keep `allowEmpty: false`.

</details>

### Q48. Explain `selfHeal`. When might you not want it? 🟡

<details>
<summary><b>Answer</b></summary>

With `selfHeal: true`, Argo CD syncs whenever live state diverges from Git, even without a new commit, so manual edits are reverted within seconds. You might disable it during incident response (an on-call engineer scales a Deployment and does not want it undone), for prod where every change must be a human-initiated sync, or for resources that controllers legitimately modify and you have not yet excluded. Prefer pausing via `argocd app set --self-heal=false` temporarily, with the change tracked.

</details>

### Q49. What is the `retry` block in a sync policy? 🟡

<details>
<summary><b>Answer</b></summary>

It retries a failed sync operation with exponential backoff:

```yaml
syncPolicy:
  retry:
    limit: 5            # -1 for unlimited
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

Useful for transient failures: a CRD that another app installs a few seconds later, a webhook that is not ready, API throttling. It is not a fix for broken manifests; those will fail five times instead of once.

</details>

### Q50. What does the `resources-finalizer.argocd.argoproj.io` finalizer do? 🟢

<details>
<summary><b>Answer</b></summary>

When an Application with this finalizer is deleted, Argo CD first deletes all resources it manages (cascade delete), then removes the finalizer so the Application disappears. Without it, deleting the Application leaves the workloads running unmanaged. The variant `resources-finalizer.argocd.argoproj.io/background` uses background propagation. The CLI equivalent is `argocd app delete --cascade` (default) vs `--cascade=false`.

</details>

### Q51. What does `CreateNamespace=true` do, and what does `managedNamespaceMetadata` add? 🟢

<details>
<summary><b>Answer</b></summary>

`CreateNamespace=true` makes Argo CD create the destination namespace if it does not exist. `managedNamespaceMetadata` lets you set labels and annotations on that namespace (for example Pod Security Admission labels or a team label):

```yaml
syncPolicy:
  syncOptions: [CreateNamespace=true]
  managedNamespaceMetadata:
    labels:
      pod-security.kubernetes.io/enforce: restricted
```

The project must allow the `Namespace` kind in `clusterResourceWhitelist`.

</details>

### Q52. When would you use `ServerSideApply=true`? 🟡

<details>
<summary><b>Answer</b></summary>

- Large objects (big CRDs, dashboards in ConfigMaps) that fail client-side apply with `metadata.annotations: Too long: must have at most 262144 bytes`, because client-side apply stores the full object in `last-applied-configuration`.
- When several managers own different fields of one object and you want proper field ownership instead of overwriting.
- Patching resources partially (with `ServerSideApply` you can apply just the fields you declare to an object created elsewhere).

It can be set per app (`syncOptions`) or per resource (annotation).

</details>

### Q53. What does `ApplyOutOfSyncOnly=true` do, and why does it help large apps? 🟡

<details>
<summary><b>Answer</b></summary>

By default a sync applies every resource in the app, even unchanged ones. `ApplyOutOfSyncOnly=true` applies only resources that are out of sync. For an app with hundreds or thousands of objects (a monitoring stack, a CRD bundle), it cuts sync time and API server load significantly. Hooks and waves still run in order.

</details>

### Q54. What is `PruneLast=true` for? 🟡

<details>
<summary><b>Answer</b></summary>

It defers pruning until after all other resources in all waves have been applied and are healthy. Example: you rename a Service from `api` to `api-v2`. Without PruneLast, the old Service might be deleted before the new one is ready, briefly breaking traffic. With PruneLast, `api-v2` comes up first, then `api` is removed.

</details>

### Q55. When would you use `Replace=true` and `Force=true`? 🔴

<details>
<summary><b>Answer</b></summary>

`Replace=true` makes Argo CD use `kubectl replace` (or `create`) instead of `apply`, which avoids the last-applied annotation size limit and three-way merge surprises. Adding `Force=true` deletes and recreates the object, needed when changing immutable fields such as a Deployment's `spec.selector`, a Job's template, or a Service's `clusterIP`. Both are dangerous on stateful or traffic-serving objects, so set them per resource with an annotation, not per app.

</details>

### Q56. What do `Prune=false`, `Prune=confirm`, `Delete=false` and `Delete=confirm` do? 🟡

<details>
<summary><b>Answer</b></summary>

They are per-resource sync options set in `argocd.argoproj.io/sync-options`:

- `Prune=false`: never prune this resource even if removed from Git (it shows as requiring pruning).
- `Prune=confirm`: pruning waits for a manual confirmation in the UI/CLI.
- `Delete=false`: keep the resource when the Application is deleted (classic use: PVCs, namespaces).
- `Delete=confirm`: deletion with the app requires confirmation.

</details>

### Q57. What is `SkipDryRunOnMissingResource=true`? 🟡

<details>
<summary><b>Answer</b></summary>

Before applying, Argo CD dry-runs every resource. If a resource's CRD does not exist yet (because the same sync, or another app, installs it), the dry-run fails with `the server could not find the requested resource` and the whole sync fails. This option skips the dry-run for kinds that are not yet known so the sync can proceed and the CRD in an earlier wave gets created first. Combine with sync waves and retries.

</details>

### Q58. What is `Validate=false`? 🟡

<details>
<summary><b>Answer</b></summary>

It disables kubectl schema validation (`--validate=false`) during apply. Use it narrowly for resources that are valid to the API server but fail client-side validation, such as objects with fields from a newer API than the schema Argo CD has, or `x-kubernetes-preserve-unknown-fields` edge cases. It hides genuine typos too, so set it per resource rather than per app.

</details>

### Q59. What is a sync operation vs a refresh? 🟢

<details>
<summary><b>Answer</b></summary>

A **refresh** recomputes the desired vs live comparison (re-resolving the Git revision, re-rendering if needed) and updates sync and health status. It changes nothing in the cluster. A **sync** applies the desired state to the cluster. Automated sync policies trigger syncs after refreshes that show a new revision or drift.

</details>

### Q60. How does `argocd app rollback` work, and why is it discouraged in GitOps? 🟡

<details>
<summary><b>Answer</b></summary>

It re-syncs the app to a revision from its sync history (`argocd app rollback web 12`). It requires auto-sync to be disabled, otherwise you get `rollback cannot be initiated when auto-sync is enabled`. The problem: Git still points at the newer, bad revision, so the app shows `OutOfSync` and the next person who syncs, or re-enables auto-sync, redeploys the bad version. The GitOps way is `git revert`, which keeps Git and cluster aligned. Use CLI rollback only as an emergency brake.

</details>

### Q61. What are parameter overrides, and why should they be restricted? 🔴

<details>
<summary><b>Answer</b></summary>

You can set Helm parameters, value files or Kustomize images directly on the Application (`argocd app set web -p image.tag=1.5.0`), or override them during a sync. They live in the Application spec, not in the app's source repo, so they bypass review of the config repo. If Applications are themselves managed in Git (app-of-apps, ApplicationSets), a CLI override is reverted. Restrict with RBAC: only grant the `override` action to automation, and generally keep `update` for platform admins.

</details>

### Q62. What happens when you change `spec.source.path` on an existing Application with prune enabled? 🔴

<details>
<summary><b>Answer</b></summary>

Argo CD renders the new path; anything tracked by this app that is not in the new manifests is now extraneous and gets pruned, while new resources are created. If the new path contains renamed objects, you get delete-and-create rather than update, which can mean downtime or data loss for PVCs and StatefulSets. Do it with prune temporarily disabled, sync, verify, then prune manually, or mark critical objects with `Prune=false`.

</details>

### Q63. What does `revisionHistoryLimit` on an Application control? 🟢

<details>
<summary><b>Answer</b></summary>

How many past sync entries Argo CD keeps in `status.history` (default 10). It bounds the size of the Application object in etcd and limits how far `argocd app rollback` can go back. It has nothing to do with the `revisionHistoryLimit` on Deployments, which controls old ReplicaSets.

</details>

### Q64. What does `status.operationState` tell you? 🟡

<details>
<summary><b>Answer</b></summary>

The last or current sync operation: `phase` (`Running`, `Succeeded`, `Failed`, `Error`, `Terminating`), `message`, start and finish times, who or what initiated it (`initiatedBy.username` or `automated: true`), the synced revision, and per-resource `syncResult.resources` with status and message. It is the first place to look when a sync failed, and what notifications triggers read.

</details>

### Q65. How do you sync only a subset of resources? 🟡

<details>
<summary><b>Answer</b></summary>

Selective sync, from the UI or CLI:

```bash
argocd app sync web --resource apps:Deployment:web
argocd app sync web --resource :ConfigMap:web-config --resource :Service:web
```

The format is `GROUP:KIND:NAME` (empty group for core kinds), optionally with namespace `GROUP:KIND:NAMESPACE/NAME`. Selective syncs do not run hooks and are not recorded as full syncs, so use them for surgical fixes, not routine deploys.

</details>

### Q66. What does `argocd.argoproj.io/skip-reconcile: "true"` do? 🔴

<details>
<summary><b>Answer</b></summary>

Placed on an Application, it tells the controller to stop reconciling that app: no refresh, no sync, status frozen. It is useful when migrating an app between Argo CD instances or shards, or when an app causes controller trouble and you need to pause it without deleting it (which might cascade). Remove the annotation to resume. Do not leave it in place; the app silently stops being GitOps-managed.

</details>

---

## Sync Waves, Hooks, Health & Diffing

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q67. What are sync waves and how is ordering decided? 🟢

<details>
<summary><b>Answer</b></summary>

A wave is an integer set with `argocd.argoproj.io/sync-wave` (default `"0"`, negatives allowed). Within each sync phase, Argo CD applies resources in ascending wave order and waits for all resources in a wave to be healthy before starting the next. Within a single wave, ordering is by kind (Namespaces and CRDs first, then RBAC, ConfigMaps, Secrets, Services, workloads, Ingress), then by name.

</details>

### Q68. What are the sync phases and hook types? 🟢

<details>
<summary><b>Answer</b></summary>

Phases run in order: **PreSync**, **Sync**, **PostSync**. Hook types set with `argocd.argoproj.io/hook`:

- `PreSync`: before the main apply (migrations, backups).
- `Sync`: alongside normal resources, ordered by wave.
- `PostSync`: after all Sync resources are healthy (smoke tests, notifications).
- `SyncFail`: only when the sync fails (cleanup, alerting).
- `PostDelete`: after the app's resources are deleted (cleanup of external state).
- `Skip`: tells Argo CD not to apply this manifest at all.

</details>

### Q69. What are hook delete policies? 🟢

<details>
<summary><b>Answer</b></summary>

Set with `argocd.argoproj.io/hook-delete-policy`:

- `BeforeHookCreation` (default): delete the previous hook object just before creating the new one, so the last run stays inspectable.
- `HookSucceeded`: delete after success.
- `HookFailed`: delete after failure.

Combine them, for example `BeforeHookCreation,HookSucceeded`. Without `BeforeHookCreation`, a Job hook with a fixed name fails the next sync because a Job's template is immutable.

</details>

### Q70. How would you run a database migration with Argo CD? 🟡

<details>
<summary><b>Answer</b></summary>

As a `PreSync` hook Job that uses the new image:

```yaml
metadata:
  name: api-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  backoffLimit: 1
  activeDeadlineSeconds: 600
```

If it fails, the sync stops and the old version keeps running. Migrations must be backward compatible (expand/contract) because old pods run against the new schema during the rollout. For the very first install, the database itself must already exist (separate app or earlier sync).

</details>

### Q71. How are Helm hooks treated by Argo CD? 🟡

<details>
<summary><b>Answer</b></summary>

Argo CD maps them: `helm.sh/hook: pre-install` and `pre-upgrade` become `PreSync`; `post-install` and `post-upgrade` become `PostSync`; `helm.sh/hook-weight` maps to sync wave; `helm.sh/hook-delete-policy` values map to Argo CD equivalents (`before-hook-creation` to `BeforeHookCreation`, `hook-succeeded` to `HookSucceeded`). Because Argo CD has no install vs upgrade distinction, install-only hooks run on every sync. `pre-delete`/`post-delete` and `test` hooks are not run in the same way, so check chart behaviour.

</details>

### Q72. Is there a delay between waves? 🔴

<details>
<summary><b>Answer</b></summary>

Yes, two seconds by default between waves, controlled by the `ARGOCD_SYNC_WAVE_DELAY` environment variable on the controller. It exists so that other controllers have time to update status (for example a Deployment controller setting `observedGeneration`) before Argo CD evaluates health; otherwise a resource could look healthy based on stale status.

</details>

### Q73. What health statuses can a resource have? 🟢

<details>
<summary><b>Answer</b></summary>

`Healthy`, `Progressing`, `Degraded`, `Suspended` (paused, for example a suspended CronJob or a paused Rollout), `Missing` (in Git but not in the cluster), and `Unknown` (health check errored). The Application's health is the worst health among its resources, in the order Degraded/Missing/Progressing/Suspended over Healthy.

</details>

### Q74. How does Argo CD decide a Deployment is healthy? 🟡

<details>
<summary><b>Answer</b></summary>

It mirrors `kubectl rollout status`: `status.observedGeneration` must be at least `metadata.generation`, updated replicas must equal desired replicas, available replicas must match, and there must be no `Progressing` condition with reason `ProgressDeadlineExceeded` (which makes it `Degraded`). A paused Deployment is `Suspended`. While pods roll it is `Progressing`.

</details>

### Q75. Which resources does Argo CD consider healthy immediately? 🟡

<details>
<summary><b>Answer</b></summary>

Resources with no health check, such as ConfigMaps, Secrets, ServiceAccounts, RBAC objects, ClusterIP Services and most CRDs without a built-in or custom check, are treated as healthy as soon as they exist. That matters for waves: a CR from an operator in wave 0 counts as healthy instantly even if the operator has not reconciled it, so wave 1 starts too early unless you add a Lua health check.

</details>

### Q76. How do you write a custom health check in Lua? 🟡

<details>
<summary><b>Answer</b></summary>

Add a key in `argocd-cm` named `resource.customizations.health.<group>_<kind>`. The script receives `obj` and returns a table with `status` and `message`:

```yaml
resource.customizations.health.example.com_Database: |
  hs = { status = "Progressing", message = "Provisioning" }
  if obj.status ~= nil and obj.status.phase == "Ready" then
    hs.status = "Healthy"; hs.message = "Database ready"
  elseif obj.status ~= nil and obj.status.phase == "Failed" then
    hs.status = "Degraded"; hs.message = obj.status.reason or "Failed"
  end
  return hs
```

Test offline with `argocd admin settings resource-overrides health obj.yaml --argocd-cm-path argocd-cm.yaml`.

</details>

### Q77. Why are Application resources not health-checked by default, and why does that matter for app-of-apps? 🔴

<details>
<summary><b>Answer</b></summary>

Health assessment for `argoproj.io/Application` was removed in 1.8 because a parent app would reflect every child's health, which confused users. The consequence: in app-of-apps, child Applications count as healthy as soon as they are created, so sync waves on them do not wait for the child to actually deploy. To make waves meaningful you add a Lua check under `resource.customizations.health.argoproj.io_Application` that returns `obj.status.health.status`.

</details>

### Q78. What are custom resource actions? 🔴

<details>
<summary><b>Answer</b></summary>

Lua-defined operations exposed in the UI and CLI under a resource, defined in `resource.customizations.actions.<group>_<kind>` with a `discovery.lua` (which actions are available) and per-action `action.lua` that returns the modified object. Built-in ones include `restart` for Deployments and StatefulSets and `resume`, `promote-full`, `abort`, `retry` for Rollouts. Running one needs RBAC permission `action/<group>/<kind>/<action>`. They let on-call engineers do safe, audited operations without `kubectl` access.

</details>

### Q79. How does Argo CD compute a diff? 🟡

<details>
<summary><b>Answer</b></summary>

By default it does a three-way comparison using the `kubectl.kubernetes.io/last-applied-configuration` annotation: desired (rendered), last applied, and live. Fields present in live but never declared (added by defaulting or other controllers) are ignored, while fields you declared and someone changed are flagged. It also normalizes known quirks (quantities, empty values, defaults for certain kinds) and applies `ignoreDifferences`. With server-side diff, it instead asks the API server for a dry-run apply result and compares that with live.

</details>

### Q80. What is server-side diff, and why use it? 🔴

<details>
<summary><b>Answer</b></summary>

Enabled with `argocd.argoproj.io/compare-options: ServerSideDiff=true` on an app or `controller.diff.server.side: "true"` globally. Argo CD sends a server-side apply dry-run of the desired manifest and compares the API server's result with live. Defaulting, mutating webhooks and field ownership are applied as the server would, which eliminates most false `OutOfSync` caused by defaulted fields and admission webhooks. Cost: one dry-run request per resource on comparison (cached per revision).

</details>

### Q81. How do you configure `ignoreDifferences` on an Application? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      name: web
      jsonPointers:
        - /spec/replicas
    - group: ""
      kind: Secret
      jqPathExpressions:
        - .data["tls.crt"]
    - group: "*"
      kind: "*"
      managedFieldsManagers:
        - kube-controller-manager
```

JSON pointers target exact paths, jq expressions handle arrays and conditionals, and `managedFieldsManagers` ignores all fields owned by a given field manager.

</details>

### Q82. Why do you need `RespectIgnoreDifferences=true` in addition to `ignoreDifferences`? 🟡

<details>
<summary><b>Answer</b></summary>

`ignoreDifferences` only affects the comparison. When a sync happens for any other reason, Argo CD still applies the full desired manifest, overwriting the ignored field (for example resetting replicas the HPA scaled up). `RespectIgnoreDifferences=true` makes the sync patch the desired object with live values for ignored fields before applying, so those fields are left alone. It applies to resources that already exist; on creation the desired value is used.

</details>

### Q83. How do you ignore differences globally instead of per app? 🟡

<details>
<summary><b>Answer</b></summary>

In `argocd-cm`:

```yaml
resource.customizations.ignoreDifferences.admissionregistration.k8s.io_MutatingWebhookConfiguration: |
  jqPathExpressions:
    - '.webhooks[]?.clientConfig.caBundle'
resource.customizations.ignoreDifferences.all: |
  managedFieldsManagers:
    - kube-controller-manager
```

Use `<group>_<kind>` keys for one kind or `.all` for every kind. Global ignores are powerful and hide real drift, so keep them to cases that are noisy for everyone.

</details>

### Q84. What does `IgnoreExtraneous` do? 🟡

<details>
<summary><b>Answer</b></summary>

`argocd.argoproj.io/compare-options: IgnoreExtraneous` on a resource excludes it from the app's sync status when it is not in Git. It is used for resources generated at runtime that inherit tracking metadata, for example objects created by a Kustomize generator or operator output copied from a tracked parent. They still appear in the tree but do not make the app `OutOfSync` or get pruned.

</details>

### Q85. What is the difference between `resource.exclusions` and `ignoreDifferences`? 🔴

<details>
<summary><b>Answer</b></summary>

`ignoreDifferences` still watches the resource and only skips certain fields when diffing. `resource.exclusions` (in `argocd-cm`) removes whole API groups or kinds from the controller's view: they are not watched, cached, shown or managed. Exclusions reduce memory and API load (for example Cilium identities, Kyverno reports, high-churn CRDs), but anything excluded cannot be deployed by Argo CD. `resource.inclusions` is the allow-list form.

</details>

### Q86. What is `ignoreResourceUpdates`? 🔴

<details>
<summary><b>Answer</b></summary>

A setting (`resource.customizations.ignoreResourceUpdates.<group>_<kind>` or `.all`, plus `resource.ignoreResourceUpdatesEnabled`) that tells the controller not to trigger an app refresh when only the listed fields change, typically `/status` or `metadata.resourceVersion` of noisy objects. Unlike `ignoreDifferences`, it is about reconcile frequency, not diff results. On clusters with operators that update status every few seconds, it can cut controller CPU drastically.

</details>

### Q87. What are SharedResourceWarning and FailOnSharedResource? 🟡

<details>
<summary><b>Answer</b></summary>

If two Applications render the same resource (same group, kind, namespace and name), they fight over it: each sync overwrites the other and tracking flips. Argo CD raises a `SharedResourceWarning` condition naming both apps. `FailOnSharedResource=true` in sync options makes the sync fail instead of silently taking ownership. Fix the root cause by giving each resource one owner (often a namespace or a shared ConfigMap declared twice).

</details>

### Q88. What does "OrphanedResourceWarning" mean? 🟡

<details>
<summary><b>Answer</b></summary>

When a project has `orphanedResources` monitoring enabled, Argo CD lists resources in the app's destination namespace that no Application tracks (excluding owned children and some defaults). It warns because those objects were probably created by hand and are outside GitOps. With `warn: true` the app gets a condition; you can ignore specific names with `ignore` rules in the project.

</details>

### Q89. Why can a resource show `OutOfSync` even though nothing changed in Git? 🟡

<details>
<summary><b>Answer</b></summary>

Common reasons: a controller or HPA changed a field you declare; a mutating webhook injected or altered fields; the API server defaults or normalizes a value differently than you wrote it (`cpu: 1000m` vs `1`, `"80"` vs `80`); Helm functions like `randAlphaNum` or `genCA` generate new values on every render; a CRD's conversion changes representation; or another Application manages the same resource. `argocd app diff` shows the exact field.

</details>

### Q90. How do health checks interact with sync waves when a resource is `Degraded`? 🟡

<details>
<summary><b>Answer</b></summary>

If a resource in the current wave becomes `Degraded`, the sync stops progressing to later waves and eventually fails (it does not keep waiting forever once health is terminally bad). `Progressing` resources make the sync wait. This is how a broken migration or failing Deployment in wave 0 prevents the Ingress switch in wave 1. A SyncFail hook can then run.

</details>

### Q91. How do you make a Job that runs on every sync but is not a hook? 🔴

<details>
<summary><b>Answer</b></summary>

You mostly cannot, and should not try. A plain (non-hook) Job with a fixed name is created once; on later syncs it stays `Synced` and never reruns, and a changed pod template fails with `field is immutable`. If it must run every sync, make it a hook (`Sync` or `PostSync`) with `BeforeHookCreation`, or give the hook a `generateName` so each sync creates a new Job. If it must run only when its spec changes, keep it a normal resource and add `argocd.argoproj.io/sync-options: Replace=true,Force=true` so a changed template recreates it. Choosing deliberately avoids both the "never reruns" and the "immutable field" traps.

</details>

### Q92. What is the `Skip` hook used for? 🔴

<details>
<summary><b>Answer</b></summary>

`argocd.argoproj.io/hook: Skip` tells Argo CD not to apply the manifest. It is handy when a chart or Kustomize base produces an object you do not want Argo CD to create (for example a Helm test Pod, or a resource that a separate tool manages), and you cannot easily remove it from the rendered output. A cleaner option, when possible, is to disable it via values.

</details>

---

## AppProjects, RBAC & SSO

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q93. What is an AppProject and what can it restrict? 🟢

<details>
<summary><b>Answer</b></summary>

An `AppProject` groups Applications and sets guardrails:

- `sourceRepos`: which Git/Helm repos apps may use (globs allowed, `!` to negate).
- `destinations`: which clusters and namespaces apps may deploy to.
- `clusterResourceWhitelist` / `clusterResourceBlacklist`: cluster-scoped kinds allowed.
- `namespaceResourceWhitelist` / `namespaceResourceBlacklist`: namespaced kinds allowed.
- `roles`: project-scoped roles with policies, SSO groups and JWT tokens.
- `syncWindows`, `orphanedResources`, `signatureKeys`, `sourceNamespaces`.

</details>

### Q94. Why should teams not use the `default` project? 🟢

<details>
<summary><b>Answer</b></summary>

The `default` project allows any source repo, any destination cluster and namespace, and any resource kind, including ClusterRoles and CRDs. A team app in `default` can deploy to `kube-system` or grant itself cluster-admin. Create one project per team or domain, and lock down `default` (empty `sourceRepos` and `destinations`) so nothing can accidentally use it.

</details>

### Q95. What does an empty `clusterResourceWhitelist` mean? 🟡

<details>
<summary><b>Answer</b></summary>

If `clusterResourceWhitelist` is omitted or empty, the project may not create any cluster-scoped resources at all (Namespaces, ClusterRoles, CRDs, PersistentVolumes, StorageClasses). Namespaced resources are allowed by default unless blacklisted. A typical team project whitelists only `Namespace` (for `CreateNamespace=true`), and platform projects whitelist `*`.

</details>

### Q96. How do project roles and JWT tokens work? 🟡

<details>
<summary><b>Answer</b></summary>

A project role has policies scoped to that project and can be bound to SSO groups or issued tokens:

```bash
argocd proj role create team-a ci
argocd proj role add-policy team-a ci --action sync --object '*' --permission allow
argocd proj role create-token team-a ci --expires-in 720h
```

The token is a JWT the CI system uses with `--auth-token`. Its issued-at time is recorded in the project, so you can revoke a token by deleting it from the role without affecting other tokens.

</details>

### Q97. What are sync windows? 🟡

<details>
<summary><b>Answer</b></summary>

Project-level time windows that allow or deny syncs:

```yaml
syncWindows:
  - kind: deny
    schedule: "0 22 * * *"
    duration: 8h
    applications: ["*-prod"]
    manualSync: true
    timeZone: "Asia/Kolkata"
```

`deny` blocks syncs during the window; `allow` permits syncs only inside it. `manualSync: true` lets humans sync during a deny window while blocking automated syncs. They are used for change freezes, business hours and festive-season freezes.

</details>

### Q98. How is Argo CD RBAC structured? 🟢

<details>
<summary><b>Answer</b></summary>

Casbin policy in `argocd-rbac-cm`. Policy lines: `p, <subject>, <resource>, <action>, <object>, <effect>`. Group lines: `g, <user-or-group>, <role>`.

```csv
p, role:team-a-dev, applications, sync, team-a/*, allow
g, my-org:team-a, role:team-a-dev
```

Resources include `applications`, `applicationsets`, `projects`, `clusters`, `repositories`, `accounts`, `certificates`, `gpgkeys`, `logs`, `exec`. Built-in roles are `role:readonly` and `role:admin`. `policy.default` is applied to every authenticated user.

</details>

### Q99. What do `allow` and `deny` effects do when both match? 🟡

<details>
<summary><b>Answer</b></summary>

Deny wins. If a user has a role that allows `sync` on `team-a/*` and another policy denies `sync` on `team-a/*-prod`, they cannot sync prod apps. That lets you grant broad access and carve out exceptions, which is easier to audit than enumerating every allowed object.

</details>

### Q100. What is the `scopes` setting in `argocd-rbac-cm`? 🟡

<details>
<summary><b>Answer</b></summary>

It lists which OIDC token claims are used as subjects for `g` lines, by default `[groups]`. Set `scopes: "[groups, email]"` to map individual emails to roles. If your IdP sends groups under a different claim, you either map it in the IdP or in Dex, or add that claim name to scopes. When group-based RBAC "does not work", wrong scopes or a missing groups claim is the usual cause.

</details>

### Q101. How do you test RBAC changes before applying them? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
argocd admin settings rbac validate --policy-file policy.csv
argocd admin settings rbac can role:team-a-dev sync applications 'team-a/web-prod' --policy-file policy.csv
argocd admin settings rbac can my-org:team-a get logs 'team-a/web-dev' --policy-file policy.csv
```

Or test against the live config map with `--namespace argocd`. Users can check themselves with `argocd account can-i sync applications 'team-a/*'`.

</details>

### Q102. What permissions are needed to view logs and exec into pods from the UI? 🟡

<details>
<summary><b>Answer</b></summary>

Logs require `p, <role>, logs, get, <project>/<app>, allow` (enforced separately from `applications, get` in modern versions). Exec (web terminal) must be enabled with `exec.enabled: "true"` in `argocd-cm` and granted with `p, <role>, exec, create, <project>/<app>, allow`, and the argocd-server ServiceAccount needs `pods/exec` in the target cluster. Most teams grant logs widely and exec rarely, if at all.

</details>

### Q103. How do you configure SSO with an OIDC provider directly? 🟡

<details>
<summary><b>Answer</b></summary>

In `argocd-cm`:

```yaml
url: https://argocd.example.com
oidc.config: |
  name: Okta
  issuer: https://example.okta.com/oauth2/default
  clientID: 0oa1abcd
  clientSecret: $oidc.okta.clientSecret
  requestedScopes: ["openid", "profile", "email", "groups"]
```

The `$oidc.okta.clientSecret` reference reads a key from `argocd-secret`. Register `https://argocd.example.com/auth/callback` as the redirect URI in the IdP, and make sure the groups claim is included.

</details>

### Q104. How do you configure GitHub login through Dex? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
url: https://argocd.example.com
dex.config: |
  connectors:
    - type: github
      id: github
      name: GitHub
      config:
        clientID: $dex.github.clientID
        clientSecret: $dex.github.clientSecret
        orgs:
          - name: my-org
```

The OAuth app's callback is `https://argocd.example.com/api/dex/callback`. Groups arrive as `my-org:team-slug`, which you bind in `argocd-rbac-cm`.

</details>

### Q105. How do you disable the local admin user safely? 🟢

<details>
<summary><b>Answer</b></summary>

First confirm SSO works and at least one SSO group is mapped to `role:admin`. Then set `admin.enabled: "false"` in `argocd-cm` and delete `argocd-initial-admin-secret`. Keep a break-glass path documented: the Kubernetes admin can re-enable the account through the ConfigMap, or use `argocd login --core` with cluster credentials.

</details>

### Q106. What are local accounts and when are they appropriate? 🟢

<details>
<summary><b>Answer</b></summary>

Accounts defined in `argocd-cm` as `accounts.<name>: login, apiKey`. `login` allows UI/CLI password login, `apiKey` allows generating tokens. They suit automation (a `ci` account with an API key) and small labs. For humans in a company, use SSO so joiners and leavers are handled by the IdP; project role tokens are an alternative for CI scoped to one project.

</details>

### Q107. What is the difference between global RBAC and project roles? 🔴

<details>
<summary><b>Answer</b></summary>

Global RBAC in `argocd-rbac-cm` is managed by Argo CD admins and can reference any project. Project roles live inside the AppProject spec (so project owners can manage them in Git), can only grant permissions within that project, and are referenced as `proj:<project>:<role>`. Both are evaluated together. A clean model: global RBAC maps SSO groups to broad read roles, project roles grant team-specific write access.

</details>

### Q108. How can you restrict which namespaces an AppProject's apps can be created in? 🔴

<details>
<summary><b>Answer</b></summary>

With apps-in-any-namespace enabled, the project's `sourceNamespaces` lists the namespaces where Application CRs belonging to this project may live, for example `[team-a-apps]`. An Application created in another namespace that references this project is rejected. This is different from `destinations`, which restricts where the app deploys.

</details>

### Q109. How do you stop a project's apps from deploying to the Argo CD cluster's control-plane namespaces? 🟡

<details>
<summary><b>Answer</b></summary>

Use precise `destinations` with namespace globs that cannot match system namespaces (`team-a-*`), and negations where needed:

```yaml
destinations:
  - server: https://kubernetes.default.svc
    namespace: team-a-*
  - server: "*"
    namespace: "!kube-system"
```

Combine with a `namespaceResourceBlacklist` for quota and network policies so teams cannot weaken platform controls.

</details>

### Q110. What is destination service account impersonation in Argo CD? 🔴

<details>
<summary><b>Answer</b></summary>

A newer feature (alpha in 2.13) where an AppProject maps destinations to Kubernetes ServiceAccounts via `destinationServiceAccounts`, and the controller impersonates that ServiceAccount when syncing, instead of using its own powerful identity. It is enabled with `application.sync.impersonation.enabled: "true"` in `argocd-cm`. It brings Kubernetes RBAC into play for each team's deployments, similar to Flux's per-Kustomization `serviceAccountName`.

</details>

---

## ApplicationSets, App-of-Apps & Multi-Cluster

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q111. What problem does an ApplicationSet solve? 🟢

<details>
<summary><b>Answer</b></summary>

It generates many Applications from one template. Instead of hand-writing `web-dev`, `web-staging`, `web-prod` for every service and every cluster, a generator produces parameter sets (environments, clusters, directories, PRs) and the template stamps out an Application for each. Adding a cluster or a folder creates apps automatically, and removing it deletes them.

</details>

### Q112. List the ApplicationSet generators. 🟢

<details>
<summary><b>Answer</b></summary>

- **List**: fixed elements.
- **Cluster**: registered clusters (cluster Secrets), filtered by label selector.
- **Git**: directories or files in a repo.
- **Matrix**: Cartesian product of two generators.
- **Merge**: combine generators, overriding by key.
- **SCM Provider**: repositories in a GitHub/GitLab/Bitbucket/Gitea org.
- **Pull Request**: open PRs (preview environments).
- **Cluster Decision Resource**: placement decisions from an external CR (for example Open Cluster Management).
- **Plugin**: parameters from your own HTTP service.

</details>

### Q113. Show a list generator example. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
spec:
  goTemplate: true
  generators:
    - list:
        elements:
          - env: dev
            cluster: in-cluster
          - env: prod
            cluster: prod-eu
  template:
    metadata:
      name: "web-{{ .env }}"
    spec:
      project: web
      source:
        repoURL: https://github.com/org/cfg.git
        targetRevision: main
        path: "apps/web/overlays/{{ .env }}"
      destination:
        name: "{{ .cluster }}"
        namespace: web
```

</details>

### Q114. How does the cluster generator work? 🟡

<details>
<summary><b>Answer</b></summary>

It produces one parameter set per cluster Secret in the `argocd` namespace (and the local cluster when no selector excludes it). Parameters include `name`, `nameNormalized`, `server`, `metadata.labels.<key>` and `metadata.annotations.<key>`, plus any `values` you define:

```yaml
generators:
  - clusters:
      selector:
        matchLabels:
          env: prod
      values:
        revision: v2.3.0
```

Labeling a cluster Secret `env=prod` instantly adds every prod app to it.

</details>

### Q115. Compare the git directory and git files generators. 🟡

<details>
<summary><b>Answer</b></summary>

The **directory** generator emits one set per directory matching a glob (`apps/*`), with parameters `path.path`, `path.basename`, `path.segments`; `exclude: true` entries skip folders. It is ideal for "one folder, one app".

The **files** generator reads JSON or YAML files matching a glob (`envs/*/config.json`) and turns their keys into parameters, plus `path.*` for the file's directory. It is ideal when each app or environment needs structured settings (cluster, namespace, flags) beyond its name.

</details>

### Q116. What is a matrix generator and what are its limits? 🟡

<details>
<summary><b>Answer</b></summary>

It combines two child generators into their Cartesian product, for example every cluster times every add-on directory. The second generator can reference parameters from the first (like using `{{ .metadata.labels.region }}` in a git path). Limits: exactly two child generators per matrix, and nesting is limited to two levels (a matrix inside a matrix or merge). Products grow fast, so filter with selectors.

</details>

### Q117. What is the merge generator used for? 🔴

<details>
<summary><b>Answer</b></summary>

It merges the parameter sets of several generators using `mergeKeys`. The first generator provides the base list; later generators override values for matching keys. Classic use: generate apps for all clusters (cluster generator), then override `revision` or `values` for a few clusters with a list generator matched on `server`. It avoids duplicating whole ApplicationSets for small exceptions.

</details>

### Q118. Why use `goTemplate: true` and `missingkey=error`? 🟡

<details>
<summary><b>Answer</b></summary>

The default fasttemplate syntax (`{{name}}`) only does string substitution and silently leaves `{{name}}` in place or empty on typos. Go templates (`{{ .name }}`) support functions (Sprig: `lower`, `default`, `replace`), conditionals and nested fields. `goTemplateOptions: ["missingkey=error"]` makes a missing key fail generation with a clear error instead of creating an app named `web-<no value>` that deploys somewhere unexpected.

</details>

### Q119. What happens to generated Applications when an element disappears from a generator? 🟡

<details>
<summary><b>Answer</b></summary>

By default the ApplicationSet controller deletes the Application, and if the Application has the resources finalizer, its resources are deleted too. Protections: `syncPolicy.preserveResourcesOnDeletion: true` (delete Applications but leave workloads), and `applicationsSync: create-only` or `create-update` to stop the controller deleting Applications at all. A bad commit that removes a `config.json` could otherwise wipe an environment.

</details>

### Q120. Explain the `applicationsSync` policy values. 🔴

<details>
<summary><b>Answer</b></summary>

`spec.syncPolicy.applicationsSync` controls what the ApplicationSet controller may do to generated Applications:

- `sync` (default): create, update and delete.
- `create-update`: create and update, never delete.
- `create-delete`: create and delete, never update existing ones.
- `create-only`: create, never update or delete.

The controller can also be restricted globally with `--policy`. `create-only` is useful when teams are allowed to tweak generated apps afterwards.

</details>

### Q121. What is `templatePatch` in an ApplicationSet? 🔴

<details>
<summary><b>Answer</b></summary>

A Go-template string that renders to a patch applied on top of the generated Application, available with `goTemplate: true`. It enables conditional structure that the plain template cannot express, such as adding `automated` sync only for non-prod or adding extra value files when a parameter is set:

```yaml
templatePatch: |
  {{- if eq .env "dev" }}
  spec:
    syncPolicy:
      automated: {prune: true, selfHeal: true}
  {{- end }}
```

</details>

### Q122. What is the RollingSync strategy? 🔴

<details>
<summary><b>Answer</b></summary>

A progressive sync strategy for ApplicationSets (`strategy.type: RollingSync`) that syncs generated apps in ordered steps selected by labels, waiting for each step's apps to be healthy before continuing, with `maxUpdate` limiting how many sync at once:

```yaml
strategy:
  type: RollingSync
  rollingSync:
    steps:
      - matchExpressions: [{key: env, operator: In, values: [staging]}]
      - matchExpressions: [{key: env, operator: In, values: [prod]}]
        maxUpdate: 25%
```

It must be enabled on the controller (progressive syncs flag). It gives fleet-level waves for a change that touches many clusters.

</details>

### Q123. What is the app-of-apps pattern? 🟢

<details>
<summary><b>Answer</b></summary>

A parent Application whose source path contains Application manifests (and often AppProjects and ApplicationSets). Syncing the parent creates the children, which then deploy their own workloads. It is used to bootstrap a cluster from a single manifest and to manage Applications declaratively in Git. Pair it with sync waves and a Lua health check on Application resources for ordering.

</details>

### Q124. App-of-apps vs ApplicationSet: when do you choose which? 🟡

<details>
<summary><b>Answer</b></summary>

- **App-of-apps**: the children are heterogeneous (ingress controller, cert-manager, monitoring, each with its own config) or need explicit ordering with waves. It is simple and explicit, but repetitive for similar apps.
- **ApplicationSet**: the children follow one pattern varying by cluster, environment or directory. It is DRY and scales to fleets, but the template is harder to read and less flexible per app.

Most real setups use both: a root app-of-apps that installs ApplicationSets.

</details>

### Q125. How do you register an EKS cluster without static tokens? 🔴

<details>
<summary><b>Answer</b></summary>

Use IAM-based auth in the cluster Secret:

```json
{
  "awsAuthConfig": {
    "clusterName": "prod-eu",
    "roleARN": "arn:aws:iam::111122223333:role/argocd-deployer"
  },
  "tlsClientConfig": { "caData": "<base64 CA>" }
}
```

Argo CD's controller and server run with IRSA or EKS Pod Identity, assume the role, and get short-lived tokens. On the target cluster, map that role to a Kubernetes group with an EKS access entry (or `aws-auth`), bound to the RBAC Argo CD needs. No long-lived bearer token is stored.

</details>

### Q126. Hub-and-spoke vs one Argo CD per cluster: what are the trade-offs? 🔴

<details>
<summary><b>Answer</b></summary>

**Hub-and-spoke** (one Argo CD managing many clusters): single UI, central RBAC and ApplicationSets across the fleet, less to operate. Downsides: the hub holds credentials to every cluster (large blast radius), needs network reach to every API server, and must scale with sharding.

**Per-cluster** instances: isolation, no inbound reach needed, failure stays local. Downsides: many instances to upgrade and configure, no single pane of glass (unless you add a control plane such as Akuity or a portal).

A common middle ground: one hub per environment tier or region.

</details>

### Q127. How do you pass per-cluster values from cluster Secrets to apps? 🟡

<details>
<summary><b>Answer</b></summary>

Put labels or annotations on the cluster Secret (`region: eu-west-1`, `ingress-class: alb`) and reference them in the ApplicationSet template through the cluster generator, for example `{{ index .metadata.labels "region" }}` in a Helm `valuesObject` or value file path (`values-{{ .metadata.labels.region }}.yaml`). This keeps per-cluster facts in one place and avoids per-cluster ApplicationSets.

</details>

### Q128. How does the pull request generator support preview environments? 🟡

<details>
<summary><b>Answer</b></summary>

It polls GitHub, GitLab, Gitea, Bitbucket or Azure DevOps for open PRs (optionally filtered by label) and provides `number`, `branch`, `branch_slug`, `head_sha` and `head_short_sha`. The template deploys each PR to its own namespace, for example `preview-{{ .number }}`, using an image tag built from the head SHA. When the PR closes, the Application and namespace are deleted. Add `requeueAfterSeconds` or a webhook for responsiveness.

</details>

### Q129. Can the child Applications of an app-of-apps live in different projects? 🟡

<details>
<summary><b>Answer</b></summary>

Yes. The parent Application must be in a project that allows deploying Application resources into the `argocd` namespace (often a dedicated `bootstrap` or the default project for platform admins). Each child sets its own `spec.project`, and the child's restrictions are enforced when the child syncs. Be careful: whoever can write to the parent's path can create Applications in any project, so protect that path like you protect admin access.

</details>

### Q130. How do you stop an ApplicationSet from overwriting manual changes to a generated Application? 🔴

<details>
<summary><b>Answer</b></summary>

Either use `applicationsSync: create-only`/`create-delete`, or list fields to ignore with `ignoreApplicationDifferences`:

```yaml
spec:
  ignoreApplicationDifferences:
    - jsonPointers:
        - /spec/syncPolicy
    - name: web-prod
      jqPathExpressions:
        - .spec.source.helm.parameters
```

This lets operators pause auto-sync on one generated app during an incident without the ApplicationSet controller restoring it.

</details>

### Q131. What permissions does Argo CD need on a managed cluster? 🟡

<details>
<summary><b>Answer</b></summary>

`argocd cluster add` grants the `argocd-manager` ServiceAccount a ClusterRole with all verbs on all resources (effectively cluster-admin) by default. For tighter setups, register with `--namespace` to limit to specific namespaces (namespace-scoped mode) or provide your own Secret with a token bound to a narrower ClusterRole. Remember the controller needs at least list/watch on everything it manages, plus create/patch/delete for syncing.

</details>

### Q132. What is namespace-scoped Argo CD? 🔴

<details>
<summary><b>Answer</b></summary>

An installation (`namespace-install.yaml`) that does not get cluster-wide permissions on its own cluster; it manages only namespaces explicitly granted, and cluster Secrets list `namespaces` to restrict what the controller watches. Teams that cannot get cluster-admin (shared clusters, regulated environments) use it. Cluster-scoped resources and CRDs must be installed by someone else.

</details>

---

## Helm, Kustomize & Multiple Sources

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q133. How does Argo CD deploy a Helm chart? 🟢

<details>
<summary><b>Answer</b></summary>

The repo-server runs `helm template` with the release name, namespace, value files, `values`/`valuesObject` and parameters, producing plain manifests. The controller then diffs and applies them like any other app. Charts can come from a Git path (`path`) or a Helm/OCI repository (`chart` plus `targetRevision` version).

</details>

### Q134. What are the consequences of Argo CD using `helm template` rather than `helm install`? 🟡

<details>
<summary><b>Answer</b></summary>

- No Helm release Secret: `helm list` and `helm history` show nothing, and `helm rollback` cannot be used.
- The `lookup` function returns empty, so charts that read existing Secrets or ConfigMaps at render time misbehave.
- Random functions (`randAlphaNum`, `genCA`) produce new values on every render, causing permanent diffs.
- Install vs upgrade (`.Release.IsInstall`) cannot be distinguished.
- `.Capabilities.APIVersions` comes from the target cluster, which Argo CD passes in.

</details>

### Q135. What are the ways to provide Helm values in an Application? 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
helm:
  releaseName: web
  valueFiles: [values.yaml, values-prod.yaml]
  ignoreMissingValueFiles: true
  values: |
    replicaCount: 3
  valuesObject:
    image: {tag: "1.4.2"}
  parameters:
    - name: ingress.enabled
      value: "true"
```

Precedence from low to high: chart defaults, `valueFiles` in order, `values`/`valuesObject`, then `parameters`. Prefer value files in Git; inline values are harder to review.

</details>

### Q136. How do you deploy a chart from an OCI registry? 🟡

<details>
<summary><b>Answer</b></summary>

Register the registry as a Helm repo with OCI enabled, then reference the chart:

```yaml
source:
  repoURL: ghcr.io/org/charts   # no oci:// prefix in the classic form
  chart: web
  targetRevision: 1.4.2
```

The repository Secret needs `type: helm`, `enableOCI: "true"`, and credentials if private. Recent Argo CD releases also accept `oci://` URLs directly as a native OCI source. Charts are pulled by version; digests give immutability.

</details>

### Q137. How does Argo CD deploy Kustomize, and what overrides can the Application set? 🟢

<details>
<summary><b>Answer</b></summary>

If the path contains `kustomization.yaml`, the repo-server runs `kustomize build`. The Application can override: `images` (tags or digests), `namePrefix`/`nameSuffix`, `commonLabels`/`commonAnnotations`, `namespace`, `replicas`, `patches`, `components` and the Kustomize `version` (if you registered several binaries). Build flags such as `--enable-helm` are set globally with `kustomize.buildOptions` in `argocd-cm`.

</details>

### Q138. What are multiple sources, and what is the `$values` reference? 🟡

<details>
<summary><b>Answer</b></summary>

`spec.sources` (plural) lets one Application combine several sources. The most common use is a remote Helm chart plus values from your Git repo:

```yaml
sources:
  - repoURL: https://charts.bitnami.com/bitnami
    chart: redis
    targetRevision: 20.1.0
    helm:
      valueFiles:
        - $values/platform/redis/values-prod.yaml
  - repoURL: https://github.com/org/cfg.git
    targetRevision: main
    ref: values
```

The source with `ref: values` is not rendered itself; `$values` resolves to its checkout root. Without multiple sources you would need an umbrella chart.

</details>

### Q139. What are the caveats of multiple sources? 🔴

<details>
<summary><b>Answer</b></summary>

- The UI and some CLI commands (rollback, parameter overrides per source) had limited support in early versions; check your version.
- Several renderable sources in one app merge their manifests, so name collisions become shared resources inside one app.
- A change in either repo triggers a sync, and sync history records multiple revisions.
- Image Updater and some tooling only understand single-source apps.

Many teams use multiple sources only for the "chart plus values" case.

</details>

### Q140. What is an umbrella chart, and how does it compare to multiple sources? 🟡

<details>
<summary><b>Answer</b></summary>

An umbrella chart is your own chart that declares the upstream chart as a dependency in `Chart.yaml`, with values nested under the dependency name. Argo CD runs `helm dependency build` before templating. It works on every Argo CD version and lets you add your own templates alongside (extra Secrets, NetworkPolicies). Multiple sources avoid the wrapper and the nested values, but cannot add templates. Umbrella charts also pin dependencies in `Chart.lock`.

</details>

### Q141. How do you pass Argo CD build environment values into manifests? 🔴

<details>
<summary><b>Answer</b></summary>

Argo CD exposes build environment variables to Helm, Kustomize and plugins, such as `ARGOCD_APP_NAME`, `ARGOCD_APP_NAMESPACE`, `ARGOCD_APP_REVISION`, `ARGOCD_APP_REVISION_SHORT`, `ARGOCD_APP_SOURCE_REPO_URL` and `ARGOCD_APP_SOURCE_PATH`. In Helm you use them via parameters:

```yaml
helm:
  parameters:
    - name: gitCommit
      value: $ARGOCD_APP_REVISION
```

They are useful for labelling resources with the deployed commit.

</details>

### Q142. How do you use Helm inside Kustomize with Argo CD? 🟡

<details>
<summary><b>Answer</b></summary>

Add `kustomize.buildOptions: --enable-helm` to `argocd-cm`, then use the `helmCharts` field in `kustomization.yaml`:

```yaml
helmCharts:
  - name: ingress-nginx
    repo: https://kubernetes.github.io/ingress-nginx
    version: 4.11.3
    releaseName: ingress
    valuesFile: values.yaml
patches:
  - path: patch-service.yaml
```

This lets you patch chart output with Kustomize, useful when a chart lacks a value for what you need.

</details>

### Q143. What are Jsonnet and directory sources in Argo CD? 🟡

<details>
<summary><b>Answer</b></summary>

If a path has no `Chart.yaml` or `kustomization.yaml`, Argo CD treats it as a directory of plain YAML/JSON, and also evaluates `*.jsonnet` files. Options under `source.directory`: `recurse: true`, `include`/`exclude` globs, and `jsonnet` settings (`tlas`, `extVars`, `libs`). Plain directories are the simplest and most transparent form, often used for rendered manifests.

</details>

### Q144. How does Argo CD detect which tool to use for a path? 🟢

<details>
<summary><b>Answer</b></summary>

Unless the Application sets `helm`, `kustomize`, `directory` or `plugin` explicitly, the repo-server inspects the path: `Chart.yaml` means Helm, `kustomization.yaml`/`kustomization.yml`/`Kustomization` means Kustomize, a matching CMP `discover` rule means that plugin, otherwise plain directory (with Jsonnet). Setting the tool explicitly avoids surprises when someone adds a stray `kustomization.yaml`.

</details>

---

## Progressive Delivery, Image Updater & Notifications

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q145. What is Argo Rollouts and how does it relate to Argo CD? 🟢

<details>
<summary><b>Answer</b></summary>

Argo Rollouts is a separate controller that adds a `Rollout` CRD, a drop-in replacement for `Deployment` with canary and blue-green strategies, traffic shaping and automated analysis. Argo CD deploys the Rollout manifest like any other resource; Rollouts controls how the new version becomes live. Argo CD has built-in health for Rollouts (`Suspended` when paused, `Degraded` when aborted) and UI actions to promote or abort.

</details>

### Q146. How does a canary Rollout work without a service mesh or ingress integration? 🟡

<details>
<summary><b>Answer</b></summary>

Weights are approximated with replica counts. With 10 replicas and `setWeight: 20`, Rollouts runs 2 canary pods and 8 stable pods behind the same Service, so roughly 20% of requests hit the canary. It is coarse (with 3 replicas, 20% rounds to one pod, 33%) and depends on load balancing, but needs nothing extra. With `canaryService`/`stableService` specified, Rollouts also adds selectors so you can address each version separately for analysis.

</details>

### Q147. Which traffic routers does Argo Rollouts support? 🟡

<details>
<summary><b>Answer</b></summary>

NGINX Ingress (canary Ingress with weight annotations), AWS ALB (weighted target groups), Istio (VirtualService weights, also subset-based), Linkerd/SMI, Traefik, Ambassador/Emissary, Apache APISIX, Kong and Gateway API through a plugin. With a router, the canary weight is exact regardless of replica count, and you can do header-based routing (send internal testers to the canary) with `setHeaderRoute`.

</details>

### Q148. Describe the main canary step types. 🟢

<details>
<summary><b>Answer</b></summary>

- `setWeight: N`: send N% of traffic to the canary.
- `pause: {duration: 5m}`: wait; `pause: {}` waits indefinitely for `promote`.
- `analysis`: run an AnalysisRun inline and block until it finishes.
- `setCanaryScale`: control canary replica count independent of weight.
- `setHeaderRoute` / `setMirrorRoute`: header routing or traffic mirroring (with supported routers).
- `experiment`: run baseline and canary ReplicaSets side by side with analysis.

</details>

### Q149. What is an AnalysisTemplate, and what providers can it use? 🟡

<details>
<summary><b>Answer</b></summary>

An `AnalysisTemplate` (or cluster-wide `ClusterAnalysisTemplate`) defines metrics to evaluate during a rollout, each with a provider, `interval`, `count`, `successCondition`/`failureCondition` and `failureLimit`. Providers include Prometheus, Datadog, New Relic, CloudWatch, Wavefront, Graphite, InfluxDB, Kayenta, a generic `web` (HTTP JSON) check, and `job` (a Kubernetes Job whose success or failure is the result). An `AnalysisRun` is an instance of the template for one rollout.

</details>

### Q150. What do `failureLimit`, `inconclusiveLimit` and `consecutiveErrorLimit` mean? 🔴

<details>
<summary><b>Answer</b></summary>

- `failureLimit`: how many failed measurements are tolerated before the metric (and the run) fails. `0` means the first failure aborts.
- `inconclusiveLimit`: how many inconclusive results (neither success nor failure condition met) before the run becomes inconclusive, which pauses the rollout for a human.
- `consecutiveErrorLimit` (default 4): how many consecutive provider errors (Prometheus unreachable, query error) before the metric is marked `Error`, which also fails the analysis.

</details>

### Q151. Inline analysis vs background analysis: what is the difference? 🟡

<details>
<summary><b>Answer</b></summary>

**Inline** analysis is a canary step: the rollout blocks at that step until the AnalysisRun completes. **Background** analysis is declared under `strategy.canary.analysis` with an optional `startingStep`; it runs continuously alongside the steps and aborts the rollout as soon as it fails. Background is good for continuous error-rate monitoring; inline is good for gate-style checks such as a smoke test before increasing weight.

</details>

### Q152. How does a blue-green Rollout work? 🟡

<details>
<summary><b>Answer</b></summary>

Two Services: `activeService` points at the current version, `previewService` at the new one. On a change, Rollouts brings up the full new ReplicaSet, points preview at it, optionally runs `prePromotionAnalysis`, then switches the active Service selector (instant cutover) when promoted (`autoPromotionEnabled: false` requires a manual `promote`). `postPromotionAnalysis` can still roll back, and the old ReplicaSet is scaled down after `scaleDownDelaySeconds` (30 by default) so in-flight connections drain.

</details>

### Q153. What happens when a Rollout is aborted? 🟡

<details>
<summary><b>Answer</b></summary>

The canary ReplicaSet is scaled down, traffic weight returns to 0% canary, and the stable ReplicaSet keeps serving. The Rollout status becomes `Degraded` with a message like `RolloutAborted: ... Metric "x" assessed Failed`. Git still declares the new version, so Argo CD shows the app as Synced but Degraded. To proceed you either fix forward with a new version, revert in Git, or `kubectl argo rollouts retry rollout` after fixing the underlying problem.

</details>

### Q154. How do you migrate an existing Deployment to a Rollout with no downtime? 🔴

<details>
<summary><b>Answer</b></summary>

Use `workloadRef` so the Rollout references the existing Deployment's pod template instead of copying it:

```yaml
spec:
  replicas: 5
  workloadRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
    scaleDown: progressively   # or onsuccess / never
```

Rollouts creates its own ReplicaSet from the template and scales the Deployment down as Rollout pods become ready. Once complete, remove the Deployment or keep it at zero replicas, and set the Deployment's replicas out of Argo CD's control to avoid a fight.

</details>

### Q155. What is Argo CD Image Updater? 🟢

<details>
<summary><b>Answer</b></summary>

A separate controller that watches container registries for new image tags matching rules you set, and updates Argo CD Applications to use them. Strategies include `semver` (highest version within a constraint), `digest` (track a mutable tag's digest), plus most-recent-build and alphabetical strategies. It supports Helm and Kustomize sources only. It lets you automate "new image, deploy to dev" without CI committing to the config repo.

</details>

### Q156. Compare Image Updater's `argocd` and `git` write-back methods. 🟡

<details>
<summary><b>Answer</b></summary>

- `argocd` (default): sets a parameter override on the Application through the Argo CD API. Nothing is committed; the change disappears if the Application is recreated from Git, so it is not really GitOps.
- `git`: commits the change to the repo, by default as `.argocd-source-<app>.yaml` next to the app path (or a Kustomize image edit), optionally to a separate branch (`git-branch: main:image-updater`). It keeps Git as source of truth and is the production choice.

</details>

### Q157. What are the components of Argo CD notifications? 🟢

<details>
<summary><b>Answer</b></summary>

Configured in `argocd-notifications-cm` with secrets in `argocd-notifications-secret`:

- **Services**: where to send (Slack, Teams, email, webhook, GitHub, PagerDuty, Opsgenie, Telegram, Grafana).
- **Templates**: the message body, using Go templates over the `app` object.
- **Triggers**: conditions (`when` expressions) that select templates, such as `on-sync-failed`, `on-health-degraded`, `on-deployed`.
- **Subscriptions**: annotations on Applications or AppProjects like `notifications.argoproj.io/subscribe.on-sync-failed.slack: deploys`.

</details>

### Q158. How do you avoid duplicate notifications? 🟡

<details>
<summary><b>Answer</b></summary>

Use `oncePer` in the trigger condition so a notification is sent once per value of a field, typically the revision:

```yaml
trigger.on-deployed: |
  - when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
    oncePer: app.status.sync.revision
    send: [app-deployed]
```

The controller records sent notifications in an annotation on the Application, so restarts do not resend.

</details>

### Q159. How can Argo CD report deployment status back to GitHub? 🔴

<details>
<summary><b>Answer</b></summary>

With the notifications GitHub service: configure a GitHub App in `service.github` and a template with a `github.status` (commit status) or `github.deployment` block. On `on-deployed` or `on-sync-failed`, Argo CD sets a status on the synced commit of the config repo, so PRs and commits show green or red for "deployed to prod". This closes the loop for developers who never open the Argo CD UI.

</details>

### Q160. How do Argo CD and Argo Rollouts divide responsibility during a canary? 🟡

<details>
<summary><b>Answer</b></summary>

Argo CD's sync finishes as soon as the new Rollout spec is applied; the app then shows `Progressing` or `Suspended` while Rollouts executes steps. Argo CD should not manage objects Rollouts creates (canary Ingress, ReplicaSets, AnalysisRuns); they are children through ownerReferences. Promote/abort can happen via `kubectl argo rollouts` or Argo CD resource actions. Never use `selfHeal` reverts to "fix" a paused Rollout; the pause is intentional.

</details>

---

## Secrets in GitOps

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q161. Why are plain Kubernetes Secrets a problem in GitOps? 🟢

<details>
<summary><b>Answer</b></summary>

A Kubernetes Secret is only base64-encoded, not encrypted. Committing it to Git exposes the value to everyone with repo access, forever, in every clone and in history, even if you delete it later. GitOps needs either encrypted secrets in Git (Sealed Secrets, SOPS) or references to an external secret store (External Secrets Operator, Vault agents, CSI driver).

</details>

### Q162. How does Sealed Secrets work? 🟢

<details>
<summary><b>Answer</b></summary>

The Sealed Secrets controller in the cluster holds a private key; `kubeseal` encrypts a Secret with the controller's public certificate into a `SealedSecret` CR, which is safe to commit. Argo CD applies the SealedSecret; the controller decrypts it and creates the real Secret, owned by the SealedSecret. Only that controller (holding that private key) can decrypt.

```bash
kubectl create secret generic db --from-literal=password=s3cr3t -n app --dry-run=client -o yaml \
  | kubeseal --format yaml > db-sealed.yaml
```

</details>

### Q163. What are Sealed Secrets scopes? 🟡

<details>
<summary><b>Answer</b></summary>

- `strict` (default): ciphertext is bound to the Secret's name and namespace; it cannot be renamed or moved.
- `namespace-wide`: can be renamed within the same namespace.
- `cluster-wide`: can be used in any namespace with any name.

Strict prevents someone copying an encrypted DB password into their own namespace to read it. Set with `kubeseal --scope` or the annotations `sealedsecrets.bitnami.com/namespace-wide` / `cluster-wide`.

</details>

### Q164. How does Sealed Secrets key rotation work, and what must you back up? 🔴

<details>
<summary><b>Answer</b></summary>

The controller generates a new sealing key every 30 days by default (`--key-renew-period`), keeping all old keys so existing SealedSecrets still decrypt; new seals use the newest key. Key renewal is not secret rotation: the underlying values do not change, so rotate actual credentials separately and reseal. You must back up the key Secrets (label `sealedsecrets.bitnami.com/sealed-secrets-key`); losing them after a cluster rebuild makes every SealedSecret in Git undecryptable. `kubeseal --re-encrypt` refreshes ciphertext to the latest key.

</details>

### Q165. How does External Secrets Operator fit into GitOps? 🟢

<details>
<summary><b>Answer</b></summary>

ESO keeps secrets in a real secret manager (AWS Secrets Manager, Parameter Store, Vault, GCP Secret Manager, Azure Key Vault). Git contains only an `ExternalSecret` describing which remote key to fetch and which Kubernetes Secret to create, plus a `SecretStore`/`ClusterSecretStore` describing how to authenticate (IRSA, workload identity, Vault Kubernetes auth). ESO refreshes on `refreshInterval`, so rotation in the secret manager reaches the cluster without a Git commit.

</details>

### Q166. Show an ExternalSecret for AWS Secrets Manager. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db
  namespace: payments
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets-manager
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: prod/payments/db
        property: password
```

The ClusterSecretStore uses `provider.aws.service: SecretsManager` with IRSA via `auth.jwt.serviceAccountRef`.

</details>

### Q167. How does SOPS work, and how do you use it with Argo CD? 🟡

<details>
<summary><b>Answer</b></summary>

SOPS encrypts values (not keys) inside YAML/JSON files with a data key that is itself encrypted for one or more recipients: age, PGP, AWS KMS, GCP KMS, Azure Key Vault or Vault Transit. Rules live in `.sops.yaml` (`path_regex`, `encrypted_regex: ^(data|stringData)$`). Argo CD cannot decrypt natively, so you add a plugin: KSOPS (a Kustomize generator, needs `--enable-alpha-plugins --enable-exec`), helm-secrets, or a CMP sidecar with access to the key (for example via IRSA for KMS).

</details>

### Q168. Compare Sealed Secrets, External Secrets and SOPS. 🟡

<details>
<summary><b>Answer</b></summary>

| | Sealed Secrets | External Secrets | SOPS |
|---|---|---|---|
| What is in Git | Ciphertext per cluster | Reference only | Ciphertext, readable structure |
| Source of truth for value | Git | Secret manager | Git |
| Rotation | Reseal and commit | Change in manager, auto-sync | Re-encrypt and commit |
| Multi-cluster | Seal per cluster (or share key) | Same reference everywhere | Same file, shared KMS |
| Argo CD integration | Native (a CRD) | Native (a CRD) | Needs a plugin |
| Main risk | Losing the controller key | Secret manager outage, IAM setup | Key management, plugin upkeep |

</details>

### Q169. What is the Secrets Store CSI Driver, and when would you choose it over ESO? 🔴

<details>
<summary><b>Answer</b></summary>

The CSI driver mounts secrets from a provider (AWS, Azure, GCP, Vault) directly into pods as files at pod start, through a `SecretProviderClass`, optionally syncing to a Kubernetes Secret. Choose it when policy forbids secrets existing as Kubernetes Secret objects in etcd, or for file-based credentials. Downsides: the pod depends on the provider at start, env var usage still requires sync to a Secret, and rotation requires rotation support plus app reloads. ESO is simpler for most apps.

</details>

### Q170. How do you make pods pick up a rotated secret? 🟡

<details>
<summary><b>Answer</b></summary>

Environment variables are read only at container start, and mounted Secret files update eventually (not with `subPath`). Options: a checksum annotation on the pod template that changes when the secret changes (works when the secret is rendered from Git), Stakater Reloader watching Secrets and restarting workloads, or apps that re-read mounted files. ESO-managed values that change outside Git need Reloader or app-side reloading, because nothing in Git changes.

</details>

---

## Repo Structure, Flux, Scaling & DR

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q171. Why separate the application source repo from the config (GitOps) repo? 🟢

<details>
<summary><b>Answer</b></summary>

- CI commits (tag bumps) do not trigger new app builds or loop.
- Different access: many developers write code; fewer people approve production config.
- Clean audit: the config repo history is exactly the deployment history.
- Argo CD does not have to fetch a large source repo with binaries and tests.
- One config repo can describe many services and the platform.

Small teams sometimes keep a `deploy/` folder in the app repo; it works, but the trade-offs above appear as you grow.

</details>

### Q172. Folders per environment or branches per environment? 🟡

<details>
<summary><b>Answer</b></summary>

Folders per environment on one branch (`envs/dev`, `envs/staging`, `envs/prod`) is the recommended pattern. Branch-per-environment leads to merge conflicts, environment-specific commits that should not propagate, cherry-pick chaos and invisible differences (you cannot diff dev and prod side by side in one tree). With folders, promotion is a visible diff between two files, and Kustomize overlays or Helm value files keep shared config in one base.

</details>

### Q173. How do you promote a release from staging to production in GitOps? 🟡

<details>
<summary><b>Answer</b></summary>

Copy the exact, already-tested artifact reference (image tag or digest, chart version) from the staging folder to the prod folder in a pull request, reviewed by the right owners (CODEOWNERS), and merged when the change window allows. Automate PR creation with a script, a workflow, Image Updater for lower environments, or Kargo for stage-based promotion. Never rebuild for prod; promote the same artifact.

</details>

### Q174. What does a typical multi-team config repo layout look like? 🟡

<details>
<summary><b>Answer</b></summary>

```text
platform/                 # cluster add-ons, owned by platform team
  ingress-nginx/  cert-manager/  monitoring/
apps/
  payments/
    base/
    overlays/{dev,staging,prod}/
  checkout/
    chart-values/{dev,staging,prod}.yaml
clusters/
  prod-eu/  prod-us/      # per-cluster bootstrap (root apps, cluster-specific values)
bootstrap/                # AppProjects, ApplicationSets
```

CODEOWNERS maps `apps/payments/` to the payments team and `platform/` to platform engineers. ApplicationSets generate apps from the directories.

</details>

### Q175. How does Flux differ from Argo CD architecturally? 🟡

<details>
<summary><b>Answer</b></summary>

Flux is a set of independent controllers: source-controller (GitRepository, HelmRepository, OCIRepository, Bucket), kustomize-controller (Kustomization: apply, prune, health), helm-controller (HelmRelease using the real Helm SDK), notification-controller, and image-reflector/automation controllers. There is no central Application object, no built-in UI, and multi-tenancy uses Kubernetes RBAC via service account impersonation. Argo CD centralizes around Applications with its own API, UI, SSO and RBAC.

</details>

### Q176. When would you pick Flux over Argo CD, and vice versa? 🟡

<details>
<summary><b>Answer</b></summary>

**Flux**: you want lightweight, per-cluster, CLI-and-Git-only operation; native SOPS; real Helm releases (`helm list` works, Helm hooks behave as Helm intends); built-in image automation; and Kubernetes-native tenancy.

**Argo CD**: you need a UI for many developers, SSO with fine-grained RBAC, ApplicationSets for fleets, central hub-and-spoke management, sync waves and visual diffs.

Both are CNCF graduated and production grade; the decision is about operating model, not capability gaps.

</details>

### Q177. How do you bootstrap Flux on a cluster? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
export GITHUB_TOKEN=<pat>
flux bootstrap github \
  --owner=my-org --repository=fleet \
  --branch=main --path=clusters/prod-eu \
  --personal=false
```

It installs Flux controllers, commits their manifests to `clusters/prod-eu/flux-system` in the repo, and creates a GitRepository and Kustomization that make Flux manage itself from Git. Afterwards `flux get kustomizations` and `flux reconcile kustomization flux-system --with-source` are the daily commands.

</details>

### Q178. How do you run Argo CD in high availability? 🟡

<details>
<summary><b>Answer</b></summary>

Use the HA manifests or Helm chart HA values: multiple argocd-server replicas (stateless, behind a Service), multiple repo-server replicas, Redis HA (three Redis with Sentinel behind HAProxy), and the application controller sharded across replicas. Add PodDisruptionBudgets and anti-affinity across nodes and zones. Note that a single controller shard is still a single active pod for its clusters; HA there means fast restart, not active-active.

</details>

### Q179. How does controller sharding work? 🔴

<details>
<summary><b>Answer</b></summary>

Scale the `argocd-application-controller` StatefulSet to N and set `ARGOCD_CONTROLLER_REPLICAS=N`. Each replica handles the clusters mapped to its shard. Algorithms (`controller.sharding.algorithm`): `legacy` (hash of the cluster Secret UID, can be uneven), `round-robin` (even distribution by ordering), `consistent-hashing` (even and stable as shards change). You can pin a cluster with `shard: "1"` in its Secret. Sharding is by cluster, so one giant cluster with thousands of apps does not benefit; then you scale processors, or split clusters across Argo CD instances.

</details>

### Q180. How do you scale the repo-server, and what limits it? 🔴

<details>
<summary><b>Answer</b></summary>

Increase replicas (stateless, they share the Redis cache) and resources. Limit concurrent renders with `reposerver.parallelism.limit` to avoid OOM on heavy Helm charts. Raise `ARGOCD_EXEC_TIMEOUT` (default 90s) and `controller.repo.server.timeout.seconds` for slow charts. Monorepos hurt: every refresh of any app re-resolves the repo; use `argocd.argoproj.io/manifest-generate-paths` with webhooks so commits only refresh apps whose paths changed, and avoid huge binary files in the config repo.

</details>

### Q181. What Prometheus metrics would you watch for Argo CD? 🔴

<details>
<summary><b>Answer</b></summary>

- `argocd_app_info` (labels `sync_status`, `health_status`): count of OutOfSync/Degraded apps.
- `argocd_app_sync_total` by phase: failed syncs.
- `argocd_app_reconcile` histogram: reconcile duration.
- `argocd_app_k8s_request_total`, `argocd_cluster_api_resource_objects`, `argocd_cluster_connection_status`.
- `argocd_git_request_total` and `argocd_git_request_duration_seconds`: Git load and latency.
- `argocd_repo_pending_request_total`, `argocd_kubectl_exec_pending`: saturation.
- Workqueue depth metrics for the app refresh and operation queues.

Alert on Degraded prod apps, sync failures, cluster connection loss and rising reconcile time.

</details>

### Q182. How do you back up and restore Argo CD? 🟡

<details>
<summary><b>Answer</b></summary>

Ideally everything is already in Git: Applications, Projects, ApplicationSets and Argo CD's own config via app-of-apps. For the rest (cluster Secrets, repo credentials, settings changed in the UI):

```bash
argocd admin export -n argocd > argocd-backup-$(date +%F).yaml
argocd admin import -n argocd - < argocd-backup-2025-06-01.yaml
```

Also back up Sealed Secrets keys and any external dependencies (SSO client secrets). Restore order: install Argo CD, restore secrets and settings, apply the root app.

</details>

### Q183. What happens to running workloads if Argo CD goes down? 🟢

<details>
<summary><b>Answer</b></summary>

Nothing. Workloads keep running; Argo CD is not in the data path. You lose deployments, drift correction and visibility until it is back. That is a key interview point: a GitOps controller outage is a delivery outage, not a production outage, as long as nothing deletes Applications with cascading finalizers during recovery.

</details>

### Q184. How do you upgrade Argo CD safely? 🔴

<details>
<summary><b>Answer</b></summary>

Read the upgrade notes for every minor version you cross (3.0 changed defaults such as annotation tracking and fine-grained RBAC behaviour). Upgrade one minor at a time, in a non-prod instance first. Manage Argo CD itself with Argo CD or Helm from Git, apply CRDs with server-side apply, and watch for diffs in all apps after the upgrade (a new diff normalization can flip many apps to OutOfSync). Keep a backup from `argocd admin export` beforehand.

</details>

### Q185. Should Argo CD manage itself? 🟡

<details>
<summary><b>Answer</b></summary>

Yes, commonly: an Application for the Argo CD Helm chart or Kustomize base, in the app-of-apps, so upgrades and config changes are Git commits. Guardrails: disable `prune` for its CRDs and namespace (or use `Prune=false`), test upgrades on a staging Argo CD, and keep a documented manual recovery path (`helm upgrade` from a laptop) in case a bad commit breaks the server or controller.

</details>

### Q186. What are the main security hardening steps for Argo CD? 🔴

<details>
<summary><b>Answer</b></summary>

- SSO only, admin disabled, `policy.default` read-only or empty, least-privilege project roles.
- No apps in `default`; projects restrict repos, destinations and kinds.
- Repo credentials via GitHub App or deploy keys, read-only.
- Cluster credentials short-lived (IRSA/workload identity), namespace-scoped where possible.
- Exec disabled unless needed; logs RBAC enforced.
- Network policies around argocd namespace; argocd-server not public, or behind SSO-aware proxy.
- GPG commit signature verification via project `signatureKeys` for sensitive projects.
- Keep Argo CD patched; several CVEs have affected the API server and repo-server.

</details>

### Q187. How does GPG signature verification work in Argo CD? 🔴

<details>
<summary><b>Answer</b></summary>

Import public keys with `argocd gpg add --from key.asc` (stored in `argocd-gpg-keys-cm`) and list allowed key IDs in the AppProject's `signatureKeys`. Argo CD then refuses to sync any revision whose commit (or annotated tag) is not signed by one of those keys, showing an error in the app conditions. It protects against someone pushing unsigned commits directly to the config repo. It only works for Git sources, not Helm repositories.

</details>

### Q188. What is Kargo and what problem does it address? 🟡

<details>
<summary><b>Answer</b></summary>

Kargo (from Akuity) is a promotion orchestration layer that sits on top of Argo CD. It models **Warehouses** (watch for new images, charts, commits), **Freight** (a versioned bundle of artifacts) and **Stages** (environments), and promotes Freight through Stages with verification, by writing to Git and triggering Argo CD syncs. It replaces hand-written promotion scripts with an auditable pipeline across environments.

</details>

### Q189. What are common GitOps anti-patterns you have seen? 🟡

<details>
<summary><b>Answer</b></summary>

- Branch per environment with endless cherry-picks.
- Using `latest` or mutable tags, so Git does not describe what runs.
- CI still running `kubectl apply` "just for hotfixes".
- Every team in the `default` project with admin RBAC.
- Secrets base64-encoded in Git.
- Parameter overrides in the UI that nobody can find in Git.
- `ignoreDifferences` used to hide real drift.
- Giant single Application for the whole cluster, so one broken file blocks everything.

</details>

### Q190. How do you handle CRDs and their custom resources in the same Application? 🔴

<details>
<summary><b>Answer</b></summary>

Put CRDs in an earlier wave (or a separate Application in an earlier wave of the app-of-apps), add `SkipDryRunOnMissingResource=true` for the CRs, and use `ServerSideApply=true` because CRDs are often too large for client-side apply. Protect CRDs from pruning (`Prune=false`), since deleting a CRD deletes every CR of that kind cluster-wide. Many charts offer `crds.enabled`/`installCRDs`; platform teams often manage CRDs separately from the operator to control upgrades.

</details>

---

## Rapid-Fire Commands

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q191. How do you get the initial admin password and log in from the CLI? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
argocd admin initial-password -n argocd
kubectl -n argocd port-forward svc/argocd-server 8080:443
argocd login localhost:8080 --username admin --insecure
```

Then change it with `argocd account update-password` and delete `argocd-initial-admin-secret`.

</details>

### Q192. How do you create an Application from the CLI with auto-sync? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
argocd app create web \
  --repo https://github.com/org/cfg.git --path apps/web/overlays/dev \
  --dest-server https://kubernetes.default.svc --dest-namespace web \
  --project team-a --sync-policy automated --auto-prune --self-heal \
  --sync-option CreateNamespace=true
```

</details>

### Q193. How do you see exactly what would change before syncing? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
argocd app diff web                      # Git vs live
argocd app diff web --local ./apps/web   # local working copy vs live
argocd app sync web --dry-run            # what the sync would do
```

`argocd app diff` exits with code 1 when differences exist, so it works as a CI gate.

</details>

### Q194. How do you force Argo CD to re-render manifests ignoring its cache? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
argocd app get web --hard-refresh
# or
kubectl -n argocd annotate application web argocd.argoproj.io/refresh=hard --overwrite
```

A normal refresh (`--refresh` or `refresh=normal`) re-checks the revision but reuses cached manifests for the same commit.

</details>

### Q195. How do you wait in a pipeline until an app is synced and healthy? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
argocd app sync web --timeout 300
argocd app wait web --sync --health --operation --timeout 600
```

It exits non-zero on timeout or if the app becomes Degraded, which fails the pipeline step. For Rollouts, `kubectl argo rollouts status web -n web --timeout 15m` waits for the rollout itself.

</details>

### Q196. How do you register a new cluster and list clusters? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl config get-contexts
argocd cluster add prod-eu-context --name prod-eu --label env=prod
argocd cluster list
argocd cluster get prod-eu
```

`cluster add` uses your current kubeconfig credentials once to create the `argocd-manager` ServiceAccount, then Argo CD uses that account's token.

</details>

### Q197. How do you stop a stuck sync operation? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
argocd app terminate-op web
argocd app get web
```

This ends the running operation (for example waiting forever on a hook Job or an unhealthy wave). Then fix the cause, delete any stuck hook resource if needed, and sync again.

</details>

### Q198. How do you list and inspect ApplicationSets? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
argocd appset list
argocd appset get shack-web
kubectl -n argocd get applicationset shack-web -o jsonpath='{.status.conditions}'
kubectl -n argocd get applications -l app.kubernetes.io/part-of=shack-web
```

Conditions like `ErrorOccurred` and `ParametersGenerated` explain generation failures.

</details>

### Q199. How do you promote and abort an Argo Rollout from the CLI? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl argo rollouts get rollout web -n web --watch
kubectl argo rollouts promote web -n web          # next step
kubectl argo rollouts promote web -n web --full   # skip remaining steps
kubectl argo rollouts abort web -n web
kubectl argo rollouts retry rollout web -n web
```

The same actions are available from the Argo CD UI as resource actions on the Rollout.

</details>

### Q200. How do you seal a secret and verify the controller decrypted it? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubeseal --fetch-cert > pub-cert.pem
kubectl create secret generic api-key --from-literal=key=abc123 -n web --dry-run=client -o yaml \
  | kubeseal --cert pub-cert.pem --format yaml > api-key-sealed.yaml
git add api-key-sealed.yaml && git commit -m "Add sealed api-key" && git push
kubectl -n web get sealedsecret api-key -o jsonpath='{.status.conditions}'
kubectl -n web get secret api-key
```

A `Synced: True` condition on the SealedSecret and the presence of the Secret confirm decryption.

</details>

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
