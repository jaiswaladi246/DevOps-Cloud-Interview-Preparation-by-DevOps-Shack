# GitOps & ArgoCD — Scenarios & Troubleshooting

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

> Q1–Q100: real "what would you do" scenarios from GitOps adoption to fleet operations. Q101–Q200: real failures with the exact error text and how to fix them · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to GitOps & ArgoCD](./README.md) · [🏠 Home](../README.md)

## Table of contents

**Scenarios**
- [Adopting GitOps & Repo Design](#adopting-gitops--repo-design) — Q1–Q16
- [Sync Behaviour, Waves & Hooks](#sync-behaviour-waves--hooks) — Q17–Q32
- [Environments, ApplicationSets & Multi-Cluster](#environments-applicationsets--multi-cluster) — Q33–Q50
- [Tenancy, Security & Secrets](#tenancy-security--secrets) — Q51–Q68
- [CI Integration & Progressive Delivery](#ci-integration--progressive-delivery) — Q69–Q84
- [Operations, Scale, Upgrades & DR](#operations-scale-upgrades--dr) — Q85–Q100

**Troubleshooting**
- [OutOfSync Loops & Diff Problems](#outofsync-loops--diff-problems) — Q101–Q120
- [Manifest Generation & ComparisonError](#manifest-generation--comparisonerror) — Q121–Q138
- [Repos, Clusters, Projects & Permissions](#repos-clusters-projects--permissions) — Q139–Q160
- [Sync Operations, Hooks & Health](#sync-operations-hooks--health) — Q161–Q174
- [ApplicationSets, Rollouts & Image Updater](#applicationsets-rollouts--image-updater) — Q175–Q188
- [Secrets, Controller Performance & Recovery](#secrets-controller-performance--recovery) — Q189–Q200

---

## Adopting GitOps & Repo Design

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. Your team deploys with Jenkins running `kubectl apply` and wants to move to Argo CD without downtime. How do you migrate? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Workloads already exist; Argo CD must adopt them, not recreate them.

**What I'd do →**
1. Export what the pipeline applies into a config repo (same manifests, same names and namespaces).
2. Create Applications **without** automated sync, then `argocd app diff` each one until the diff is empty or understood.
3. Sync manually; Argo CD adopts the existing objects (it adds tracking metadata, pods are not restarted when specs match).
4. Remove the `kubectl apply` stage from Jenkins and replace it with a step that commits the new image tag.
5. Enable auto-sync, then prune and self-heal, one service at a time.

**Why →** Adopting matching manifests is non-disruptive; enabling prune before the diff is clean risks deleting resources Jenkins created that are not in Git.

**Interview tip →** Say "diff to zero before enabling automation" and mention removing CI's cluster credentials as the finish line.

</details>

### Q2. A startup with three services asks whether they need a separate config repo. What do you recommend? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Small team, low ceremony, but they will grow.

**What I'd do →**
1. Start with one config repo for all services and environments (not one per service), separate from app code.
2. Layout: `apps/<service>/{base,overlays/dev,staging,prod}` plus `bootstrap/` for Argo CD objects.
3. CI in each app repo commits tags to the config repo.
4. If they insist on one repo, use a `deploy/` folder and make sure CI tag commits do not retrigger builds (path filters, `[skip ci]`).

**Why →** A separate config repo gives a clean deploy history, simple access control for prod, and avoids build loops; one shared repo avoids sprawl.

**Interview tip →** Show you can scale the answer down: "one config repo" is the right size for most teams, not a repo per service per environment.

</details>

### Q3. A team wants a Git branch per environment (`dev`, `staging`, `prod`). How do you respond? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Branch-per-environment feels natural but causes drift between branches.

**What I'd do →**
1. Explain the problems: merge conflicts, environment-specific commits leaking on merge, cherry-picks, and no side-by-side view of differences.
2. Propose folders on `main`: `envs/dev`, `envs/staging`, `envs/prod` with Kustomize overlays or Helm value files.
3. Promotion becomes a PR that changes one value in the next folder, protected by CODEOWNERS.
4. Point every Application at `main` with a different `path`.

**Why →** Folders make differences explicit and reviewable; branches hide them in history.

**Interview tip →** Mention `git diff envs/staging envs/prod` as the killer demo for folders.

</details>

### Q4. Developers keep running `kubectl edit` in production to "quickly fix" things. How do you stop it without blocking incident response? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cultural and access problem; GitOps can enforce, but on-call needs a path.

**What I'd do →**
1. Enable `selfHeal: true` in prod so manual edits are reverted.
2. Remove write RBAC in Kubernetes from developers; give read and logs access.
3. Provide a fast Git path: a PR template for hotfixes with expedited approval.
4. For emergencies, document a break-glass: disable self-heal on one app via an audited Argo CD action, fix, then commit the change to Git and re-enable.
5. Alert on manual changes with notifications or audit logs.

**Why →** Enforcement plus a sanctioned emergency route works; enforcement alone pushes people to workarounds.

**Interview tip →** "Make the right way the fast way" lands well with interviewers.

</details>

### Q5. You must deploy one microservice to 4 environments with Helm. Which values structure do you choose? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shared defaults, small per-environment differences.

**What I'd do →**
1. Chart in Git (or an internal chart repo) with sensible defaults in `values.yaml`.
2. Per environment, a small file: `envs/<env>/values.yaml` with replicas, resources, ingress host, image tag.
3. One Application per environment (generated by an ApplicationSet) with `valueFiles` pointing at the environment file.
4. Keep the image tag in the environment file so CI and promotions edit exactly one line.

**Why →** Small diffs per environment make promotion and review trivial; defaults stay in one place.

**Interview tip →** Mention that the image tag should never live in the chart defaults, or every environment moves together.

</details>

### Q6. Your config repo has 400 Applications in one monorepo and every commit triggers a refresh of everything. What do you change? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Monorepo refresh fan-out loads the repo-server and Git.

**What I'd do →**
1. Configure the Git webhook so refreshes are event-driven, then raise `timeout.reconciliation` (for example to 10 minutes).
2. Add `argocd.argoproj.io/manifest-generate-paths: .` (or explicit paths) to each Application, so a webhook only refreshes apps whose paths changed.
3. Scale repo-server replicas and set `reposerver.parallelism.limit`.
4. Consider splitting very busy areas (CI bot commits) into separate repos.

**Why →** Without path scoping, Argo CD cannot know a commit to `apps/a` does not affect `apps/b`, so it re-resolves everything.

**Interview tip →** Naming `manifest-generate-paths` shows real experience with large installs.

</details>

### Q7. Management asks for proof of who deployed what to production and when, for an audit. How does GitOps answer this? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Auditors want traceability from change request to deployment.

**What I'd do →**
1. Point to the config repo history: every prod change is a merged PR with author, reviewers, approvals and timestamp.
2. Enforce branch protection and CODEOWNERS on `envs/prod/**`, and signed commits if required.
3. Show Argo CD sync history and Kubernetes Events (and notifications to a channel) linking each sync to a commit SHA.
4. Disable UI parameter overrides and manual syncs to non-Git revisions with RBAC so Git stays the only path.

**Why →** GitOps makes the audit trail a side effect of the workflow.

**Interview tip →** Mention commit status back to GitHub so the PR itself shows "deployed".

</details>

### Q8. A team wants to manage their Argo CD Application YAML themselves, but you do not want them editing the `argocd` namespace. What do you set up? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Self-service Applications with guardrails.

**What I'd do →**
1. Enable apps in any namespace: `application.namespaces: team-*-apps` in `argocd-cmd-params-cm`, restart server and controller.
2. Create namespace `team-a-apps` with Kubernetes RBAC letting team-a manage `applications.argoproj.io` there.
3. In the AppProject `team-a`, set `sourceNamespaces: [team-a-apps]` plus restricted repos and destinations.
4. Alternatively, keep Applications in Git under `teams/team-a/` with CODEOWNERS and an ApplicationSet that generates them.

**Why →** The project still bounds what their apps can deploy, while they own the Application objects.

**Interview tip →** Contrast "self-service through Git" (preferred) with "self-service through kubectl".

</details>

### Q9. You inherit a cluster with dozens of resources created by hand and nobody knows what is needed. How do you bring it under GitOps? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unknown state; risk of deleting something important.

**What I'd do →**
1. Export resources per namespace (`kubectl get ... -o yaml`), strip status and server fields (`kubectl neat` helps), and commit as a starting point.
2. Create Applications per namespace or service with manual sync and no prune.
3. Enable `orphanedResources: {warn: true}` on the project to list what is still not in Git.
4. Iterate until diffs are empty and orphans are explained, then enable automation.
5. Clean up: move hand-written YAML into proper Helm/Kustomize structure.

**Why →** Adopt first, refactor later; orphan warnings give you a to-do list.

**Interview tip →** Emphasize no prune until orphans are understood.

</details>

### Q10. You need to manage cluster add-ons (ingress, cert-manager, monitoring) for 20 clusters. How do you structure it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same add-ons, slightly different config per cluster.

**What I'd do →**
1. `platform/<addon>/` with a base values file plus `values-<env>.yaml` or per-cluster overrides.
2. Label cluster Secrets (`env=prod`, `region=eu`, `addon-monitoring=enabled`).
3. One ApplicationSet per add-on with a cluster generator selecting by label, using multiple sources (upstream chart plus `$values`).
4. A root app-of-apps that installs all add-on ApplicationSets, ordered by waves (cert-manager before things needing certificates).

**Why →** Labels make opting clusters in and out a one-line change; per-add-on ApplicationSets keep blast radius small.

**Interview tip →** Mention RollingSync or staged label rollout for add-on upgrades across the fleet.

</details>

### Q11. Your security team asks whether Argo CD should pull from GitHub directly or through a mirror. How do you decide? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Concerns are availability, egress rules and supply-chain trust.

**What I'd do →**
1. If clusters are in restricted networks, mirror the config repo and Helm/OCI charts to an internal Git server or registry and point Argo CD there.
2. Otherwise pull directly with read-only credentials (GitHub App or deploy key) and pin known hosts and TLS certs.
3. Require signed commits (`signatureKeys`) for prod projects either way.
4. Mirror third-party Helm charts to an internal OCI registry so upstream deletions or tampering do not break deploys.

**Why →** GitHub outages and upstream chart changes are real risks; mirrors trade freshness for control.

**Interview tip →** Note that a mirror adds a sync delay and becomes part of your critical path.

</details>

### Q12. A developer asks why their commit to the app repo did not deploy anything. What do you explain and check? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic misunderstanding of the two-repo model.

**What I'd do →**
1. Explain that Argo CD watches the config repo; app repo commits only produce an image.
2. Check the CI run: did it build, push and commit the new tag to the config repo?
3. Check the config repo log for the bot commit, then `argocd app get <app> --refresh` to confirm Argo CD sees the new revision.
4. Check the app's sync policy (manual in prod?) and sync windows.

**Why →** Most "did not deploy" issues are a broken CI write-back or a manual sync policy.

**Interview tip →** Walk the chain in order: build, push, commit, refresh, sync, health.

</details>

### Q13. You are asked to design the GitOps setup for a regulated bank. What extra controls do you add? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Segregation of duties, change approval and evidence matter as much as automation.

**What I'd do →**
1. Separate Argo CD instances or at least projects for prod, with SSO and no local users.
2. Prod sync policy manual, or automated only inside approved sync windows; CODEOWNERS approvals from a different team than the author.
3. Signed commits enforced with `signatureKeys`; branch protection with required reviews and status checks.
4. Secrets via External Secrets from an HSM-backed manager; no secret material in Git at all.
5. Notifications and commit statuses as deployment evidence; export Argo CD audit events to the SIEM.
6. Policy checks (Kyverno or Gatekeeper, plus conftest in CI) to block non-compliant manifests before merge.

**Why →** Regulators care about who approved, what changed, and proof it was what ran.

**Interview tip →** Mention change-ticket IDs in commit messages or PR titles, linked via notifications.

</details>

### Q14. The team debates Helm vs Kustomize for their own services in Argo CD. How do you advise? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both work; the choice is about who maintains the templates.

**What I'd do →**
1. Kustomize for internal services with few variations: plain YAML, easy diffs, overlays per environment.
2. Helm for distributing a reusable chart to many teams (a "golden" service chart) or when values-driven configuration is needed.
3. For third-party software, use the upstream Helm chart, optionally post-rendered with Kustomize (`--enable-helm`).
4. Standardize on one approach per category to avoid a zoo.

**Why →** Kustomize is transparent; Helm is parameterized. Argo CD supports both natively.

**Interview tip →** Mention that Argo CD renders Helm with `helm template`, so Helm's release features do not apply.

</details>

### Q15. Your CTO wants "everything in Git", including Argo CD's own configuration. How do you implement it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Argo CD self-management plus declarative config.

**What I'd do →**
1. Install Argo CD once with Helm, using a values file from the repo.
2. Add an Application `argocd` in `bootstrap/` pointing at the same chart and values (with `Prune=false` on CRDs).
3. Declare `argocd-cm`, `argocd-rbac-cm`, AppProjects, repo-creds (via External Secrets) and cluster Secrets in Git.
4. Root app-of-apps manages all of it; only the initial install and root app are applied by hand.

**Why →** Upgrades and RBAC changes become reviewed commits and DR becomes a re-bootstrap.

**Interview tip →** Admit the chicken-and-egg: something installs Argo CD the first time, and you keep a manual recovery path.

</details>

### Q16. Product teams complain that the config repo is a bottleneck owned by the platform team. How do you fix the workflow? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Centralized ownership slows teams down.

**What I'd do →**
1. Give each team a folder with CODEOWNERS so they approve their own non-prod changes.
2. Require platform approval only for `platform/`, `bootstrap/` and prod AppProjects.
3. Let ApplicationSets pick up new team folders automatically (git directory generator), so onboarding a service needs no platform PR.
4. Add CI checks (`kustomize build`, `helm template`, kubeconform, policy checks) so reviews focus on intent.

**Why →** Guardrails in projects and automated checks allow self-service without risk.

**Interview tip →** "Guardrails, not gates" is the phrase platform interviewers like.

</details>

---

## Sync Behaviour, Waves & Hooks

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q17. A new release needs a database migration that must finish before the new pods start. How do you implement it in Argo CD? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ordered, blocking step before the main apply.

**What I'd do →**
1. Add a Job with `argocd.argoproj.io/hook: PreSync` and `hook-delete-policy: BeforeHookCreation`, using the same image tag as the release.
2. Set `backoffLimit: 1` and `activeDeadlineSeconds` so it fails fast.
3. Write the migration to be backward compatible (expand now, contract in a later release).
4. Add a `SyncFail` hook to alert the team.

**Why →** PreSync blocks the rollout on migration failure, keeping the old version running.

**Interview tip →** Mention expand/contract, because old pods keep running during the rollout.

</details>

### Q18. Your app needs cert-manager's CRDs, an Issuer and then a Certificate, all in one Application. The first sync fails. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** CRDs and CRs in the same sync; dry-run fails for unknown kinds.

**What I'd do →**
1. Put CRDs in wave `-2`, the Issuer in `-1`, the Certificate and workloads in `0`.
2. Add `SkipDryRunOnMissingResource=true` so the dry-run does not fail on kinds whose CRD is not created yet.
3. Add a Lua health check for `Issuer`/`Certificate` (or rely on the built-in one) so waves wait for readiness.
4. Better: install cert-manager as its own Application in an earlier wave of the app-of-apps.

**Why →** Waves order the apply, and skipping the dry-run avoids failing on unknown types.

**Interview tip →** Say you prefer separating operators from their consumers into different apps.

</details>

### Q19. After enabling `prune: true`, someone renames a folder and the next sync would delete a production StatefulSet and its PVCs. How do you prevent this class of incident? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Prune turns a refactoring mistake into data loss.

**What I'd do →**
1. Annotate stateful resources and PVCs with `argocd.argoproj.io/sync-options: Prune=false,Delete=false`.
2. Keep `allowEmpty: false` so an empty render cannot wipe an app.
3. Add a CI check that runs `argocd app diff --local` or renders and compares resource counts on PRs to the config repo.
4. Use `Prune=confirm` on critical resources so pruning needs a human click.
5. Set StorageClass `reclaimPolicy: Retain` for important volumes.

**Why →** Defense in depth: GitOps automation should never be the only barrier before deleting data.

**Interview tip →** Mention PV `Retain` as the last line of defense.

</details>

### Q20. You want a smoke test to run after every deploy and mark the sync failed if it fails. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Post-deploy verification as part of the sync.

**What I'd do →**
1. Add a Job with `argocd.argoproj.io/hook: PostSync` and `hook-delete-policy: BeforeHookCreation`.
2. The Job curls key endpoints and exits non-zero on failure.
3. Add a notification trigger `on-sync-failed` so the team is alerted.

**Why →** PostSync runs after all Sync-phase resources are healthy, and a failed hook fails the operation.

**Interview tip →** Note that a failed PostSync does not roll back; pair it with Rollouts analysis for automatic protection.

</details>

### Q21. An HPA scales your Deployment, but Argo CD with self-heal keeps resetting replicas to 2. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two controllers fighting over `spec.replicas`.

**What I'd do →**
1. Preferred: remove `replicas` from the Deployment manifest (Helm: omit it when `autoscaling.enabled`).
2. If it must stay, add `ignoreDifferences` for `/spec/replicas` on that Deployment and `RespectIgnoreDifferences=true`.
3. Verify with `argocd app diff` and watch `kubectl get hpa,deploy -w`.

**Why →** Whoever owns a field should be the only one writing it.

**Interview tip →** Mention why `RespectIgnoreDifferences` is needed: otherwise any sync still overwrites the field.

</details>

### Q22. You must change a Deployment's label selector, which is immutable. How do you roll it out with Argo CD? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Apply fails with `field is immutable`.

**What I'd do →**
1. Safest: create a new Deployment with a new name and the new selector, shift the Service, then remove the old one (PruneLast helps).
2. If downtime is acceptable, annotate the Deployment with `argocd.argoproj.io/sync-options: Replace=true,Force=true` for one sync (delete and recreate), then remove the annotation.
3. Make sure the Service selector matches the new labels in the same change.

**Why →** Kubernetes forbids selector changes; you either recreate or run old and new side by side.

**Interview tip →** Say you would never leave `Force=true` on permanently.

</details>

### Q23. A team wants Argo CD to deploy only during business hours, except for emergencies. How do you set it up? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Sync windows with a manual override.

**What I'd do →**
1. Add an `allow` window to the project for working hours, for example `schedule: "0 9 * * 1-5"`, `duration: 9h`, with `timeZone` set.
2. Set `manualSync: true` so humans can still sync during closed hours.
3. Restrict manual sync permission for prod to on-call roles via RBAC.

**Why →** Automated syncs wait for the window; emergencies stay possible and audited.

**Interview tip →** Mention `argocd proj windows list` to show current window state.

</details>

### Q24. You need ConfigMap changes to restart pods automatically in an Argo CD-managed app. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Kubernetes does not restart pods when a referenced ConfigMap changes.

**What I'd do →**
1. Kustomize: use `configMapGenerator`, which appends a content hash to the name and updates references, so pods roll.
2. Helm: add `checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}` to the pod template annotations.
3. For values that change outside Git (ESO), use Stakater Reloader.

**Why →** A pod template change triggers a rolling update; Argo CD then shows it like any deploy.

**Interview tip →** Note that Kustomize hashed ConfigMaps leave old ones behind until pruned.

</details>

### Q25. A big app has 1,500 resources and every sync takes 10 minutes. How do you speed it up? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Sync applies everything, every time.

**What I'd do →**
1. Enable `ApplyOutOfSyncOnly=true` so only changed resources are applied.
2. Split the app into several Applications by component (CRDs, operators, workloads, dashboards).
3. Use `ServerSideApply=true` for large objects to avoid annotation bloat.
4. Raise `controller.kubectl.parallelism.limit` if the controller has headroom.
5. Review waves: many waves add delays and health waits.

**Why →** Most of the time is spent re-applying unchanged objects and waiting between waves.

**Interview tip →** Splitting apps also reduces blast radius when one manifest is broken.

</details>

### Q26. You need to delete an Application but keep all its workloads running (handing over to another team's app). How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Non-cascading delete.

**What I'd do →**
1. `argocd app delete web --cascade=false` (or remove the `resources-finalizer.argocd.argoproj.io` finalizer before `kubectl delete`).
2. Create the new Application pointing at the same manifests and sync; it adopts the resources.
3. Verify no pods restarted and the old tracking annotation is replaced.

**Why →** Without the finalizer, Argo CD removes only the Application object.

**Interview tip →** Warn that deleting a parent app-of-apps with prune can cascade into children unless they also lack finalizers.

</details>

### Q27. Your Helm chart uses `randAlphaNum` to generate a password, and the app is always OutOfSync. What is your fix? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `helm template` produces a new random value on every render.

**What I'd do →**
1. Stop generating secrets in templates; provide the password via External Secrets or Sealed Secrets.
2. If the chart supports `existingSecret`, use it.
3. Short-term, `ignoreDifferences` on that Secret's data field with `RespectIgnoreDifferences=true`, accepting the risk.

**Why →** Argo CD cannot use Helm's `lookup` to reuse the existing value, so random values are always new.

**Interview tip →** Mention `lookup` not working as the root cause; it shows Helm-in-Argo knowledge.

</details>

### Q28. Two Applications both include the same `Namespace` manifest and keep flipping ownership. How do you fix the design? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shared resource, `SharedResourceWarning`.

**What I'd do →**
1. Give the Namespace one owner: a platform app or `CreateNamespace=true` with `managedNamespaceMetadata` in exactly one app.
2. Remove it from the other app's manifests.
3. Enable `FailOnSharedResource=true` to catch future duplicates.

**Why →** One resource, one owner; otherwise each sync overwrites the other's labels.

**Interview tip →** Explain that the tracking annotation shows which app currently "owns" it.

</details>

### Q29. You want resources removed from Git to be deleted only after the new version is fully up. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pruning order matters during renames.

**What I'd do →**
1. Add `PruneLast=true` to the app's sync options.
2. For a single critical object, set it per resource with `argocd.argoproj.io/sync-options: PruneLast=true`.

**Why →** Pruning then happens after all waves are applied and healthy, so replacements exist first.

**Interview tip →** Give the Service rename example; it is the classic case.

</details>

### Q30. An operator creates child resources that inherit Argo CD's tracking metadata, and the app shows them as extraneous and wants to prune them. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Generated objects copied labels or annotations from a tracked parent.

**What I'd do →**
1. Switch to annotation tracking (default in 3.x) if still on label tracking, because operators often copy labels like `app.kubernetes.io/instance`.
2. Annotate generated objects (or configure the operator to set) `argocd.argoproj.io/compare-options: IgnoreExtraneous`.
3. Disable prune for those kinds if needed, and file an issue with the operator to stop copying tracking metadata.

**Why →** Argo CD believes it owns anything carrying its tracking ID; without care it prunes operator output.

**Interview tip →** This is exactly why annotation tracking became the default.

</details>

### Q31. A hook Job from the last sync failed and now every new sync is stuck waiting. How do you design hooks to avoid this? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hook lifecycle not managed.

**What I'd do →**
1. Always set `hook-delete-policy: BeforeHookCreation` (plus `HookSucceeded` if you do not need logs).
2. Set `activeDeadlineSeconds` and a low `backoffLimit` so hooks fail instead of hanging.
3. Use `generateName` for hooks that must create a fresh Job every sync.
4. For the stuck sync now: `argocd app terminate-op`, delete the old Job, sync again.

**Why →** Hooks are just resources; without delete policies and deadlines they collide or hang.

**Interview tip →** Mention `terminate-op` as the immediate unblock.

</details>

### Q32. You want a `Sync` to happen automatically, but never delete anything without a human looking first. How do you configure it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Auto-apply, manual delete.

**What I'd do →**
1. `syncPolicy.automated` with `prune: false`.
2. Resources removed from Git show as "requires pruning"; review them in the UI and run `argocd app sync web --prune` manually.
3. For extra safety on specific objects, `Prune=confirm`.

**Why →** Creation and updates are low risk; deletion is where incidents happen.

**Interview tip →** This is a sensible default for prod in teams new to GitOps.

</details>

---

## Environments, ApplicationSets & Multi-Cluster

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q33. You need to deploy the same app to dev, staging and prod namespaces with different values. Design it with an ApplicationSet. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** One template, three parameter sets.

**What I'd do →**
1. Repo: `charts/app/` plus `envs/<env>/values.yaml` and `envs/<env>/config.json` (`env`, `namespace`, `autoSync`).
2. ApplicationSet with a git files generator on `envs/*/config.json`, `goTemplate: true`, `missingkey=error`.
3. Template: name `app-{{ .env }}`, `valueFiles: ["../../{{ .path.path }}/values.yaml"]`, destination namespace from config.
4. `templatePatch` adds automated sync only when `autoSync` is true, so prod stays manual.

**Why →** Adding an environment becomes adding a folder; per-env differences stay in small files.

**Interview tip →** Mention `templatePatch` for conditional sync policy; it shows you know newer features.

</details>

### Q34. A new prod cluster comes online every month. How do you make sure it gets all standard add-ons automatically? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fleet onboarding should be one action.

**What I'd do →**
1. Each add-on has an ApplicationSet with a cluster generator selecting `env: prod` (and opt-out labels).
2. Onboarding a cluster = creating its cluster Secret with labels (declared in Git, credentials via External Secrets or IAM auth).
3. Per-cluster facts (region, ingress class) come from Secret labels or annotations used in the template.

**Why →** Labels on the cluster Secret drive everything; no per-cluster YAML to copy.

**Interview tip →** Say you would test with a `canary` label first for new add-on versions.

</details>

### Q35. A matrix ApplicationSet (clusters times apps) generated 300 Applications, but a bad commit to one app's chart broke all prod clusters at once. How do you redesign for safer rollouts? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fleet-wide blast radius from one commit.

**What I'd do →**
1. Pin per-environment revisions: staging tracks `main`, prod tracks a tag or a revision from cluster labels (`{{ .metadata.labels.revision }}`).
2. Use RollingSync with steps: canary clusters, then 25% of prod, then the rest.
3. Keep prod Applications at manual or windowed sync.
4. Add CI validation (render plus kubeconform plus policy) for every chart change.

**Why →** Decoupling prod revisions from `main` and staging the fleet rollout limits damage.

**Interview tip →** "Generate apps globally, roll out changes progressively" is the key message.

</details>

### Q36. You want preview environments for every pull request on the app repo. How do you do it with Argo CD? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ephemeral environments driven by PRs.

**What I'd do →**
1. CI builds an image tagged with the PR head SHA for PRs labelled `preview`.
2. ApplicationSet with a `pullRequest` generator (GitHub, label filter `preview`), creating `preview-{{ .number }}` in namespace `preview-{{ .number }}`.
3. Helm parameters set `image.tag` to `{{ .head_sha }}` and an ingress host like `pr-{{ .number }}.preview.example.com`.
4. Automated sync with prune; when the PR closes the Application and namespace are deleted.

**Why →** The generator ties environment lifecycle to PR lifecycle with no manual cleanup.

**Interview tip →** Mention resource quotas on preview namespaces to control cost.

</details>

### Q37. Some clusters need a different version of an add-on than the rest. How do you handle exceptions without duplicating ApplicationSets? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Mostly uniform fleet with a few overrides.

**What I'd do →**
1. Put the version in the cluster generator's `values` with a default, and override via a cluster Secret label (`ingress-version: 4.10.1`).
2. Or use a merge generator: cluster generator as the base, a list generator keyed on `name` overriding `revision` for the exceptions.
3. Template uses `{{ .values.revision }}` (or the label) as `targetRevision`.

**Why →** Exceptions live as data, not as copied YAML.

**Interview tip →** Show the merge generator with `mergeKeys: [name]`.

</details>

### Q38. How would you register 50 EKS clusters with Argo CD securely, without long-lived tokens? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Credential sprawl and rotation risk.

**What I'd do →**
1. Give the Argo CD controller and server an IAM role through IRSA or EKS Pod Identity.
2. In each target account, create a deployer role trusted by that role, and map it with an EKS access entry to a Kubernetes group with the needed RBAC.
3. Create cluster Secrets with `awsAuthConfig` (`clusterName`, `roleARN`) and the cluster CA, generated by Terraform or Crossplane and committed (no secret material inside).
4. Label the Secrets for ApplicationSets.

**Why →** Short-lived STS credentials, nothing to rotate, and access is revocable in IAM.

**Interview tip →** Mention that the cluster Secret then contains no credentials, only identifiers.

</details>

### Q39. A product team wants their own namespace in every cluster with the same RBAC and quotas. How do you automate it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tenant onboarding across the fleet.

**What I'd do →**
1. A `tenants/` folder with one file per team: name, quotas, groups.
2. ApplicationSet with a matrix of cluster generator times git files generator on `tenants/*.yaml`, rendering a tenant Helm chart (Namespace, ResourceQuota, LimitRange, RoleBindings, NetworkPolicy).
3. Owned by the platform project, so teams cannot alter quotas.

**Why →** One file per team, rendered everywhere, consistent and reviewable.

**Interview tip →** Mention Capsule or hierarchical namespaces as alternatives for large multi-tenant setups.

</details>

### Q40. Staging and prod must run in separate AWS accounts with no network path from staging to prod. Where do you run Argo CD? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Network isolation rules out a single hub reaching everything.

**What I'd do →**
1. One Argo CD per account or tier (a prod instance inside the prod account), each pulling from the same config repo via outbound HTTPS.
2. Each instance's ApplicationSets select only their environment folders.
3. Optionally a read-only aggregation (notifications, metrics) to a central place.

**Why →** Pull-based GitOps works naturally with isolated networks; the hub model would require cross-account API access that the rules forbid.

**Interview tip →** Contrast with hub-and-spoke and its single high-value credential store.

</details>

### Q41. You need to promote releases dev → staging → prod automatically, but prod requires approval. Design the flow. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Automated lower environments, gated prod.

**What I'd do →**
1. CI commits new tags to `envs/dev`. Argo CD auto-syncs.
2. A post-deploy workflow (triggered by a notification webhook or a scheduled check) verifies dev health and opens a PR to `envs/staging`, auto-merged after checks.
3. Same for prod, but the PR requires CODEOWNERS approval and prod syncs in a window or manually.
4. Alternatively use Kargo stages with verification and a manual promotion for prod.

**Why →** Every promotion is a Git change with a record; only the prod step needs a human.

**Interview tip →** Emphasize promoting the same artifact (tag or digest), never rebuilding.

</details>

### Q42. Your ApplicationSet uses a git directory generator, and someone adds a folder `apps/_templates` that should not deploy. How do you exclude it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Directory globs pick up everything.

**What I'd do →**
```yaml
generators:
  - git:
      repoURL: https://github.com/org/cfg.git
      revision: main
      directories:
        - path: apps/*
        - path: apps/_templates
          exclude: true
```

**Why →** Exclusions take precedence over includes in the directory generator.

**Interview tip →** Mention naming conventions (leading underscore) plus exclusion as a team standard.

</details>

### Q43. A cluster is being decommissioned. How do you remove it from Argo CD without breaking anything else? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Generated apps target this cluster; cascade delete may try to delete workloads in a cluster that is going away.

**What I'd do →**
1. Remove the cluster's labels (or Secret) in Git so generators drop its Applications.
2. If the cluster is already unreachable, set `preserveResourcesOnDeletion: true` temporarily or remove finalizers so deletion does not hang trying to reach it.
3. Delete hand-made Applications targeting it, then `argocd cluster rm`.
4. Check `argocd app list` for any app still referencing the server.

**Why →** Finalizers on apps for an unreachable cluster block deletion forever.

**Interview tip →** Mention apps stuck in `Terminating` as the classic symptom.

</details>

### Q44. Developers want to see dev, staging and prod of their service side by side in Argo CD. How do you organize it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** UI organization and labels.

**What I'd do →**
1. Label every Application with `app.kubernetes.io/part-of: <service>` and `env: <env>` (set in the ApplicationSet template).
2. Filter in the UI by label, or `argocd app list -l app.kubernetes.io/part-of=checkout`.
3. One project per team so the project filter shows only their apps.

**Why →** Consistent labels make the UI and CLI filtering useful at scale.

**Interview tip →** Small detail, but it shows you think about developer experience.

</details>

### Q45. You must deploy the same Helm chart to 30 clusters but only staging should track the latest chart version. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Different revision policy per environment.

**What I'd do →**
1. Cluster Secret labels define `chart-version` per cluster or environment.
2. ApplicationSet template sets `targetRevision: "{{ index .metadata.labels \"chart-version\" }}"`.
3. Staging clusters use a range (`4.*`), prod clusters an exact version updated by PR.

**Why →** Staging catches chart regressions early while prod upgrades deliberately.

**Interview tip →** Note that labels have value constraints (63 chars, limited characters); annotations are an alternative for longer values.

</details>

### Q46. A team wants to deploy to a cluster in a customer's private network that Argo CD cannot reach. What options do you have? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** No inbound path from the hub.

**What I'd do →**
1. Run a small Argo CD (core mode) or Flux inside the customer cluster, pulling from Git over outbound HTTPS.
2. Or use an agent-based model (for example Argo CD agent or a commercial control plane) where the in-cluster agent connects outbound to the hub.
3. Mirror required images and charts to a registry the customer network can reach.

**Why →** Pull-based agents only need outbound connectivity; the hub model needs inbound API access.

**Interview tip →** Mention that this is where the pull model is a hard requirement, not a preference.

</details>

### Q47. How would you structure Argo CD Applications for a 15-microservice product per environment? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Many similar services, a few environments.

**What I'd do →**
1. One ApplicationSet with a matrix: git directories (`services/*`) times a list of environments (or git files with env config).
2. Application names `{{ service }}-{{ env }}`, one Application per service per environment.
3. A shared "golden" Helm chart, with per-service values in each folder.

**Why →** Per-service Applications isolate failures and syncs; the generator avoids 45 hand-written files.

**Interview tip →** Explain why one giant Application for the product is a bad idea: one broken service blocks all.

</details>

### Q48. You need different sync policies per cluster in a cluster-generated ApplicationSet: auto for staging, manual for prod. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Conditional sync policy from cluster metadata.

**What I'd do →**
```yaml
templatePatch: |
  {{- if eq (index .metadata.labels "env") "staging" }}
  spec:
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
  {{- end }}
```

**Why →** The base template stays manual; the patch adds automation where the label allows it.

**Interview tip →** `templatePatch` needs `goTemplate: true`.

</details>

### Q49. Your ApplicationSet controller deleted all apps for an environment when someone accidentally removed `envs/prod/config.json`. How do you prevent a repeat? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Generator lost an element, controller deleted the app, finalizer cascaded.

**What I'd do →**
1. Set `syncPolicy.preserveResourcesOnDeletion: true` on the ApplicationSet so workloads survive an Application deletion.
2. Use `applicationsSync: create-update` for prod ApplicationSets so Applications are never deleted automatically.
3. Protect `envs/prod/**` with CODEOWNERS and a CI check that fails when an environment file is deleted.
4. Restore now by reverting the commit; the Application is recreated and adopts any surviving resources.

**Why →** The ApplicationSet controller faithfully reflects Git, including mistakes.

**Interview tip →** Name both settings; most candidates know only one.

</details>

### Q50. You need to roll out a risky Argo CD config change (new health checks) across 10 Argo CD instances. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Argo CD config is itself code with blast radius.

**What I'd do →**
1. Manage each instance's `argocd-cm` through Git (Helm values per instance).
2. Test the Lua with `argocd admin settings resource-overrides health` against real objects in CI.
3. Roll to a non-prod instance first, watch for mass health changes (`argocd_app_info` by health_status), then prod instances one at a time.
4. Have a revert PR ready.

**Why →** A broken health check can mark hundreds of apps Degraded and stall every wave-ordered sync.

**Interview tip →** Treat Argo CD config like application code: tests, staged rollout, quick revert.

</details>

---

## Tenancy, Security & Secrets

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. Ten teams share one Argo CD. How do you isolate them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multi-tenancy on a shared control plane.

**What I'd do →**
1. One AppProject per team: `sourceRepos` limited to their repos, `destinations` to their namespaces (`team-a-*`), cluster-scoped kinds limited to `Namespace`, quotas and network policies blacklisted.
2. SSO groups mapped to project roles: developers sync non-prod, leads sync prod.
3. Lock down the `default` project.
4. Optionally apps in any namespace so each team owns its Application objects.

**Why →** Projects bound what apps can do; RBAC bounds what people can do.

**Interview tip →** Separate "what an app may deploy" (project) from "who may act on it" (RBAC).

</details>

### Q52. A developer could create an Application in their project that deploys a ClusterRoleBinding granting cluster-admin. How do you prevent privilege escalation? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Projects that allow cluster-scoped RBAC kinds are an escalation path.

**What I'd do →**
1. Remove `rbac.authorization.k8s.io` cluster kinds from team projects' `clusterResourceWhitelist`.
2. Blacklist `Role`/`RoleBinding` in namespaces too if teams should not grant rights, or allow only via a platform-owned tenant chart.
3. Restrict `sourceRepos` so teams cannot point at arbitrary repos with arbitrary manifests.
4. Add Kyverno/Gatekeeper policies as a second layer, and consider destination service account impersonation.

**Why →** The controller runs with broad rights; the project is what prevents a team from borrowing them.

**Interview tip →** Say "Argo CD is a confused deputy unless projects restrict it".

</details>

### Q53. CI needs to trigger a sync of one app after a deploy commit. How do you give it access? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Least-privilege automation identity.

**What I'd do →**
1. Create a project role: `argocd proj role create team-a ci` with policy `applications, sync, team-a/web-dev, allow` (and `get`).
2. Generate a token with an expiry: `argocd proj role create-token team-a ci --expires-in 720h`.
3. Store it as a CI secret and call `argocd app sync team-a/web-dev --auth-token $TOKEN`.
4. Better still: rely on webhooks and auto-sync so CI needs no Argo CD access at all.

**Why →** Project tokens are scoped to one project and revocable individually.

**Interview tip →** The best answer is often "CI does not need Argo CD access; Git is the interface".

</details>

### Q54. An auditor says the Sealed Secrets private key is a single point of failure and a single point of compromise. How do you respond? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Valid concern for both availability and confidentiality.

**What I'd do →**
1. Availability: back up sealing keys encrypted (for example to a KMS-encrypted bucket or a vault), test restore regularly.
2. Confidentiality: restrict access to the controller namespace with RBAC, audit reads of those Secrets, rotate keys on schedule and reseal.
3. For high-sensitivity secrets, move to External Secrets with a managed secret store and HSM-backed KMS, so Git never holds ciphertext.
4. Document the threat model: anyone with the key can decrypt the full Git history of SealedSecrets.

**Why →** Encrypted-in-Git means old ciphertext stays decryptable forever with the old key.

**Interview tip →** Point out that key renewal does not re-encrypt history or rotate values.

</details>

### Q55. You need the same database password in 12 clusters. Which secrets approach do you choose and why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Sealed Secrets would require sealing 12 times (or sharing one key across clusters).

**What I'd do →**
1. Store the password once in AWS Secrets Manager (or Vault).
2. Deploy External Secrets Operator to every cluster with a ClusterSecretStore using workload identity.
3. One `ExternalSecret` manifest in Git, deployed by an ApplicationSet to all 12 clusters.
4. Rotate in the secret manager; ESO refreshes and Reloader restarts workloads.

**Why →** One source of truth for the value, no per-cluster encryption, rotation without commits.

**Interview tip →** Mention that sharing one Sealed Secrets key across clusters widens the blast radius of a key leak.

</details>

### Q56. Your team wants to use SOPS with Argo CD. What do you set up? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Argo CD cannot decrypt SOPS natively.

**What I'd do →**
1. `.sops.yaml` with creation rules: `encrypted_regex: ^(data|stringData)$`, AWS KMS key per environment.
2. Add KSOPS to the repo-server (init container copying the binary, or a CMP sidecar), set `kustomize.buildOptions: --enable-alpha-plugins --enable-exec`.
3. Give the repo-server (or sidecar) KMS decrypt permission via IRSA, scoped to the right keys.
4. Developers encrypt with `sops --encrypt --in-place`; CI checks no unencrypted Secret is committed.

**Why →** Decryption happens at render time inside the repo-server, so it needs the plugin and key access.

**Interview tip →** Note the security trade-off: the repo-server can decrypt every secret it renders, so limit who can create apps pointing at those paths.

</details>

### Q57. A plaintext Secret was committed to the config repo and pushed. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The secret is compromised the moment it is pushed.

**What I'd do →**
1. Rotate the credential immediately at the source (database, API provider).
2. Replace the manifest with a SealedSecret or ExternalSecret.
3. Optionally rewrite history (`git filter-repo`) and force-push, coordinating with everyone, but treat that as hygiene, not remediation.
4. Add secret scanning (gitleaks, GitHub push protection) as a pre-merge check.

**Why →** History rewrites do not undo clones, forks or caches; only rotation removes the risk.

**Interview tip →** Always lead with "rotate first".

</details>

### Q58. Developers need to view logs in Argo CD but must not exec into prod pods. How do you configure RBAC? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Separate permissions for logs and exec.

**What I'd do →**
```csv
p, role:dev, applications, get, */*, allow
p, role:dev, logs, get, */*, allow
p, role:oncall, exec, create, */*-prod, allow
g, my-org:developers, role:dev
g, my-org:sre-oncall, role:oncall
```
Keep `exec.enabled: "true"` only if on-call really needs the web terminal.

**Why →** Logs and exec are separate RBAC resources, so they can be granted independently.

**Interview tip →** Mention that exec actions are audited as Kubernetes events and server logs.

</details>

### Q59. Your company requires that only reviewed and signed commits ever reach production. How do you enforce it at the Argo CD level? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Defense against direct pushes or compromised CI tokens.

**What I'd do →**
1. Import the allowed GPG public keys (`argocd gpg add`).
2. Set `signatureKeys` on the prod AppProject; Argo CD refuses to sync unsigned or wrongly signed revisions.
3. On GitHub: branch protection requiring signed commits and reviews; merges done through the UI with a signing key, or squash merges by a signing bot.
4. Monitor app conditions for signature verification failures.

**Why →** Branch protection controls Git; `signatureKeys` makes Argo CD verify independently.

**Interview tip →** Note that GitHub web merges are signed with GitHub's key, which you must then trust.

</details>

### Q60. How would you give an external contractor access to one application only? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Least privilege for a temporary user.

**What I'd do →**
1. Add them to a dedicated IdP group (expiry handled in the IdP).
2. Map that group in a project role or global policy: `applications, get` and `sync` on `team-a/web-dev` only, plus `logs, get`.
3. Keep `policy.default` empty so they see nothing else.

**Why →** Group-based, time-bound access is auditable and removed automatically.

**Interview tip →** Avoid local accounts for humans; offboarding becomes manual.

</details>

### Q61. ESO is creating Secrets, but Argo CD shows the ExternalSecret as permanently OutOfSync. How do you approach it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** ESO's webhook or API defaults fields you did not specify.

**What I'd do →**
1. `argocd app diff` to see the fields, typically `conversionStrategy`, `decodingStrategy` and `metadataPolicy` in `remoteRef`.
2. Specify those defaults explicitly in your manifests, or
3. Enable `ServerSideDiff=true` for the app so defaults are applied in the comparison.
4. As a last resort, `ignoreDifferences` with jq paths on those fields.

**Why →** Declaring defaults or diffing server-side removes the false drift.

**Interview tip →** This exact issue is common; naming the fields shows hands-on experience.

</details>

### Q62. A team asks for permission to use any public Helm chart. How do you balance flexibility and safety? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Open sourceRepos means unreviewed third-party code in your clusters.

**What I'd do →**
1. Mirror approved charts to an internal OCI registry after review and scanning.
2. Allow only that registry and the team's Git repos in `sourceRepos`.
3. Provide a request process to add a chart to the mirror.
4. Keep cluster-scoped kinds restricted, so even an approved chart cannot install CRDs or ClusterRoles in a team project.

**Why →** Curated sources plus project kind restrictions limit supply-chain risk.

**Interview tip →** Mention that upstream chart repos can change or disappear; mirroring also improves reliability.

</details>

### Q63. You need to rotate the Git credentials Argo CD uses for 200 repos. How do you avoid downtime? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Per-repo credentials make rotation painful.

**What I'd do →**
1. Move to a credential template (`repo-creds`) for the org URL prefix, ideally backed by a GitHub App.
2. Create the new credential first, update the `repo-creds` Secret (or let ESO sync it), then revoke the old one.
3. Verify with `argocd repo list` and watch for `authentication required` in app conditions.

**Why →** One template covers all repos; GitHub App tokens are short-lived and auto-refreshed.

**Interview tip →** GitHub Apps also avoid tying access to a person's account.

</details>

### Q64. Your security team wants to prevent anyone from deploying privileged pods via Argo CD. Where do you enforce it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Argo CD projects restrict kinds, not pod spec content.

**What I'd do →**
1. Enforce at admission: Pod Security Admission `restricted` labels on namespaces (via `managedNamespaceMetadata`) and Kyverno/Gatekeeper policies.
2. Shift left: run the same policies in CI on the config repo (kyverno CLI, conftest).
3. Argo CD then surfaces admission denials as sync failures with the policy message.

**Why →** Admission control is the only layer that sees every create, whatever the source.

**Interview tip →** Say "Argo CD is not a policy engine; pair it with one".

</details>

### Q65. You want Argo CD to deploy with the minimum RBAC in each namespace rather than cluster-admin. What are your options? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The default controller identity is very powerful.

**What I'd do →**
1. Register clusters with namespace-scoped credentials (`argocd cluster add --namespace a --namespace b`), so the controller only watches and writes those namespaces.
2. Enable destination service account impersonation and map each project destination to a namespace-local ServiceAccount with a Role.
3. Install cluster-scoped prerequisites (CRDs, namespaces) through a separate platform pipeline or instance.

**Why →** The controller can only do what the credentials allow; impersonation brings Kubernetes RBAC per tenant.

**Interview tip →** Mention Flux's `serviceAccountName` per Kustomization as the comparable model.

</details>

### Q66. A Sealed Secret works in staging but you need to move the namespace name for prod. What happens and what do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Strict scope binds ciphertext to name and namespace.

**What I'd do →**
1. Reseal the secret for the prod namespace and name with the prod cluster's certificate.
2. Store per-environment SealedSecrets in each environment folder.
3. If many renames are expected, consider `namespace-wide` scope, accepting the reduced protection.

**Why →** Copying ciphertext to a different namespace fails with `no key could decrypt secret`.

**Interview tip →** Explain why strict scope is a security feature, not an inconvenience.

</details>

### Q67. You need secrets to be available before a Deployment starts, but ESO takes a few seconds to create them. How do you order it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The ExternalSecret exists but the Secret does not yet.

**What I'd do →**
1. Put the ExternalSecret in wave `-1`.
2. Rely on Argo CD's built-in health for `ExternalSecret` (Healthy when the `Ready` condition is true), or add a Lua check if your version lacks it.
3. The Deployment in wave 0 starts only after the Secret is created.

**Why →** Health-gated waves turn "eventually created" into "created before use".

**Interview tip →** Note the pod would otherwise sit in `CreateContainerConfigError` until the Secret appears.

</details>

### Q68. A developer accidentally deleted an Application in the UI and all its resources disappeared. How do you prevent this? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cascade delete via finalizer plus broad RBAC.

**What I'd do →**
1. Remove `delete` on applications from developer roles; deletions happen by removing files from Git.
2. For apps managed by app-of-apps or ApplicationSets, the parent recreates them; make sure that is in place for prod.
3. Add `Delete=false` on critical resources (PVCs, namespaces) so app deletion leaves them.
4. Recover now: revert or re-apply the Application from Git and sync.

**Why →** Deletion should go through review like any other change.

**Interview tip →** Mention that with declarative Applications, the UI delete is undone by the parent within minutes.

</details>

---

## CI Integration & Progressive Delivery

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q69. Design a CI pipeline that deploys through GitOps without giving CI cluster access. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** CI builds, Git deploys.

**What I'd do →**
1. Test, build, scan and push `image:sha-<short>` to the registry.
2. Check out the config repo with a token scoped to that repo only.
3. `yq -i '.image.tag = strenv(TAG)' envs/dev/values.yaml`, commit, push with retry on conflicts.
4. Argo CD auto-syncs dev; staging and prod are promoted by PR.

**Why →** CI's only privileged action is writing one file in Git, which is reviewed and audited.

**Interview tip →** Mention retry with `git pull --rebase` for concurrent pipeline runs.

</details>

### Q70. Two pipelines commit tag bumps to the config repo at the same time and one fails with "rejected, non-fast-forward". How do you make it robust? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Concurrent pushes to the same branch.

**What I'd do →**
1. Loop: `git pull --rebase && git push` up to N times with a short sleep.
2. Keep each service's tag in its own file so rebases never conflict.
3. Use a CI concurrency group per environment to serialize updates.

**Why →** Rebasing onto the latest head resolves non-conflicting concurrent commits.

**Interview tip →** File-per-service layout is what prevents real conflicts.

</details>

### Q71. Should CI wait for Argo CD to finish deploying before marking the pipeline green? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Trade-off between fast feedback and decoupling.

**What I'd do →**
1. For dev, optional: a job with a read-only project token runs `argocd app wait web-dev --sync --health --timeout 600` and reports.
2. Better: Argo CD notifications post commit statuses back to the config repo or app repo commit ("deployed to dev").
3. Promotions are gated on those statuses, not on pipeline success.

**Why →** Pipelines that poll Argo CD reintroduce coupling and credentials; statuses push the result where developers look.

**Interview tip →** Mention the GitHub commit status notification service.

</details>

### Q72. You want automatic rollback when error rates rise after a deploy. How do you build it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Argo CD alone does not watch business metrics.

**What I'd do →**
1. Convert the Deployment to an Argo Rollouts canary.
2. AnalysisTemplate with a Prometheus success-rate query, `successCondition: result[0] >= 0.99`, `failureLimit: 1`.
3. Background analysis from step 1 or inline after each `setWeight`.
4. On failure, Rollouts aborts and traffic returns to stable; then revert the Git commit to match.

**Why →** Rollouts acts in seconds based on metrics; Git revert keeps the source of truth honest.

**Interview tip →** Clarify that "rollback" here is abort to stable, and Git still needs the revert.

</details>

### Q73. Your canary uses replica-based weights with 3 replicas and `setWeight: 10`. What actually happens and what do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Weight is rounded to whole pods.

**What I'd do →**
1. Explain: 10% of 3 rounds up to one canary pod, so roughly 25% to 33% of traffic hits the canary immediately.
2. Add traffic routing (NGINX, ALB, Istio, Gateway API) so weights are exact regardless of pod count.
3. Or raise replicas, or use `setCanaryScale` with a router to control canary pod count separately.

**Why →** Without a router, traffic split equals pod ratio.

**Interview tip →** Shows you understand canary mechanics, not just YAML.

</details>

### Q74. Product wants internal staff to test the new version in production before any customer sees it. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Header-based routing to the canary at 0% weight.

**What I'd do →**
1. Rollout with a traffic router that supports header routing (Istio, NGINX via managed routes, ALB, Gateway API plugin).
2. Steps: `setCanaryScale: {replicas: 1}`, `setHeaderRoute` matching `X-Canary: internal`, `pause: {}`.
3. Staff use a browser extension or internal proxy to add the header; after sign-off, `promote` continues with weight steps.

**Why →** Header routes isolate testers from real users while the canary runs on production infrastructure.

**Interview tip →** Mention `managedRoutes` must be declared in `trafficRouting` for header routes.

</details>

### Q75. You want a release to stop at 50% in prod until a human approves. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Manual gate in canary steps.

**What I'd do →**
```yaml
steps:
  - setWeight: 20
  - pause: {duration: 10m}
  - setWeight: 50
  - pause: {}
  - setWeight: 100
```
Approve with `kubectl argo rollouts promote web -n prod` or the Argo CD "resume" action. Restrict the action with RBAC (`action/argoproj.io/Rollout/resume`).

**Why →** `pause: {}` waits indefinitely; Argo CD shows the app as `Suspended`.

**Interview tip →** Mention a notification on `Suspended` so the approver knows.

</details>

### Q76. Your team uses Image Updater, and it keeps deploying `latest`-like tags to prod. How do you fix the setup? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Overly broad tag rules and wrong scope.

**What I'd do →**
1. Annotate only dev (and maybe staging) Applications for Image Updater, never prod.
2. Use `semver` strategy with `allow-tags: regexp:^\d+\.\d+\.\d+$`, or constrain the version range in `image-list` (`web=ghcr.io/org/web:~1.4`).
3. Switch write-back to `git`, so every change is a commit; promote to prod by PR.

**Why →** Image Updater should automate the noisy lower environments; prod changes stay deliberate.

**Interview tip →** Mention that `argocd` write-back is lost if the Application is recreated.

</details>

### Q77. A blue-green release looks fine in preview, but after switching, latency spikes. How do you design protection for this? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Preview traffic was not representative.

**What I'd do →**
1. Add `postPromotionAnalysis` with latency and error metrics; failure switches the active Service back.
2. Keep `scaleDownDelaySeconds` long enough (for example 10 minutes) that the old ReplicaSet is still up for instant rollback.
3. Use `prePromotionAnalysis` with synthetic load against the preview Service.
4. Consider canary instead, so real traffic ramps gradually.

**Why →** Blue-green is all-or-nothing on cutover; post-promotion checks plus a warm old version make it reversible.

**Interview tip →** Explain why canary is safer when realistic traffic cannot be simulated.

</details>

### Q78. How do you run a load-dependent analysis in a low-traffic service where Prometheus queries return no data? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Empty query results make the metric inconclusive or error.

**What I'd do →**
1. Generate synthetic traffic during analysis with a `job` metric or a load-test step (k6 Job).
2. Write queries that handle no data (`or vector(0)` for errors, minimum request count conditions).
3. Use `inconclusiveLimit` so ambiguous results pause for a human rather than pass silently.
4. Combine with a `web` metric against a deep health endpoint.

**Why →** No data must not mean success.

**Interview tip →** Saying "absence of errors with no traffic proves nothing" shows maturity.

</details>

### Q79. The business wants to deploy 20 times a day to prod with no human approval. How do you make that safe with GitOps? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** High frequency requires automated verification at each stage.

**What I'd do →**
1. Automated promotion pipeline: dev and staging with full test suites and analysis gates.
2. Prod: Rollouts canary with metric analysis and automatic abort, small initial weights, background analysis.
3. Automated Git revert of the prod commit when a rollout aborts (a notification triggers a workflow).
4. Feature flags to decouple deploy from release; SLO-based release freezes when the error budget is exhausted.

**Why →** Speed is safe when every step has an automatic stop and rollback.

**Interview tip →** Tie it to DORA metrics: deployment frequency and change failure rate together.

</details>

### Q80. How do you keep the config repo from filling up with thousands of CI bot commits? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Noise makes human changes hard to find.

**What I'd do →**
1. Accept it for dev but keep commit messages structured (`dev: web sha-1a2b3c4`), so filtering is easy.
2. Use Image Updater or Kargo writing to a separate branch or folder for dev.
3. Squash promotions into one PR per release for staging and prod.
4. Use `git log -- envs/prod` and CODEOWNERS views for reviewers.

**Why →** History is the audit trail; structure beats suppression.

**Interview tip →** Do not suggest rewriting history in a GitOps repo.

</details>

### Q81. A release requires changes to both the app image and a ConfigMap. How do you ensure they deploy together? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Atomicity across manifests.

**What I'd do →**
1. Make both changes in the same commit to the config repo (CI can open a PR with both).
2. Ensure the ConfigMap change rolls pods (hashed name or checksum annotation).
3. Use waves if the ConfigMap must exist before pods (it usually is ordered by kind anyway).

**Why →** One commit is one desired state; Argo CD applies it in one sync.

**Interview tip →** Mention that splitting them across two commits creates an intermediate broken state.

</details>

### Q82. Your Argo Rollouts canary works, but Argo CD shows the app OutOfSync during every rollout. Why, and what do you change? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Something in Git is being modified by Rollouts.

**What I'd do →**
1. Check the diff: commonly the canary Ingress or Service selectors, or a Deployment referenced via `workloadRef` that Rollouts scales to zero.
2. Do not declare objects Rollouts creates (canary Ingress).
3. For a `workloadRef` Deployment, set its replicas to 0 in Git or ignore `/spec/replicas`.
4. If you define selectors on stable/canary Services, let Rollouts manage the `rollouts-pod-template-hash` key; three-way diff usually ignores it, server-side diff may need an ignore rule.

**Why →** Two controllers writing the same field always produces drift.

**Interview tip →** Same principle as HPA vs replicas: one owner per field.

</details>

### Q83. You need to run an integration test suite against staging after each deploy and block promotion on failure. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Post-deploy verification feeding the promotion gate.

**What I'd do →**
1. PostSync hook Job in staging that runs the test suite and fails on test failure.
2. Notification on `on-sync-succeeded` (meaning hooks passed) sets a commit status or triggers the promotion workflow.
3. Promotion PR to prod requires that status.

**Why →** The sync result encodes the test result, and the gate consumes it.

**Interview tip →** Alternatively an AnalysisRun with a `job` provider, if Rollouts is in use.

</details>

### Q84. A canary was aborted, and now on-call wants to retry the same version after fixing a downstream dependency. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same spec, aborted state.

**What I'd do →**
1. Confirm the dependency is fixed.
2. `kubectl argo rollouts retry rollout web -n prod` (or the Argo CD `retry` action).
3. Watch with `kubectl argo rollouts get rollout web -n prod --watch`.

**Why →** Retry restarts the steps for the current spec without a Git change.

**Interview tip →** If the version itself was bad, revert in Git instead of retrying.

</details>

---

## Operations, Scale, Upgrades & DR

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q85. Your Argo CD manages 2,000 apps across 40 clusters and the UI is slow and syncs lag. Where do you start? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Likely controller saturation, repo-server contention or Redis pressure.

**What I'd do →**
1. Check metrics: `argocd_app_reconcile` duration, workqueue depth, controller CPU/memory per pod, repo-server pending requests, Redis latency.
2. Shard the controller (for example 4 replicas, `consistent-hashing`) and confirm with `argocd admin cluster shards`.
3. Raise status and operation processors in line with CPU.
4. Scale the repo-server, set `reposerver.parallelism.limit`, add webhooks and `manifest-generate-paths`, raise `timeout.reconciliation`.
5. Exclude noisy kinds and enable `ignoreResourceUpdates` for status-only churn.

**Why →** Measure first; each component has a distinct bottleneck and fix.

**Interview tip →** Quote a metric you would look at first; generic "scale it" answers score low.

</details>

### Q86. You need to upgrade Argo CD from 2.10 to 3.x. Plan it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Several minor versions and a major with default changes.

**What I'd do →**
1. Read upgrade notes for each version; note 3.0 changes (annotation tracking default, fine-grained RBAC for update/delete, logs RBAC, removed legacy config).
2. Back up with `argocd admin export`.
3. Upgrade one minor at a time on a staging instance with copies of real Applications; watch for diff changes and health changes.
4. Apply CRDs with server-side apply; upgrade prod in a quiet window.
5. Post-upgrade: check `argocd app list` for new OutOfSync/Unknown apps and fix tracking or ignore rules.

**Why →** Diff normalization and tracking changes can make hundreds of apps OutOfSync and trigger unexpected syncs.

**Interview tip →** Mention disabling auto-sync globally is not possible, so test on staging first.

</details>

### Q87. The argocd namespace was accidentally deleted. Workloads are still running. How do you recover? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Control plane gone; data plane fine (unless finalizers cascaded).

**What I'd do →**
1. Check whether Applications were deleted with finalizers (workloads might be deleting); if namespace termination is stuck on Application finalizers, remove those finalizers to stop cascades.
2. Reinstall Argo CD with the same version and values.
3. Restore repo credentials, cluster Secrets and settings from backup (`argocd admin import`) or from Git plus External Secrets.
4. Restore Sealed Secrets keys if the controller lived there.
5. Apply the root app; Applications adopt existing resources without restarts.

**Why →** Argo CD is rebuildable from Git; the risk is cascade deletion during the incident.

**Interview tip →** Lead with "stop the cascade", then rebuild.

</details>

### Q88. You need to migrate all Applications from one Argo CD instance to another with no deploy gaps. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two controllers must never manage the same app at once.

**What I'd do →**
1. Prepare the new instance with the same repos, clusters, projects and tracking method.
2. For each batch: on the old instance, set `argocd.argoproj.io/skip-reconcile: "true"` or delete the Application with `--cascade=false`.
3. Create the Application on the new instance (manual sync first), confirm an empty diff, then enable automation.
4. Repeat; decommission the old instance when empty.

**Why →** Non-cascading removal plus adoption avoids both double management and deletion.

**Interview tip →** Mention that tracking IDs include the app name, so keep names identical.

</details>

### Q89. Redis in Argo CD keeps getting OOMKilled. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cache size grows with apps and resources.

**What I'd do →**
1. Raise Redis memory limits; set `maxmemory` with an eviction policy if appropriate.
2. Switch to Redis HA for resilience.
3. Reduce cache volume: exclude unneeded kinds, lower `revisionHistoryLimit`, split giant apps.
4. Consider enabling Redis compression (`redis.compression`, gzip default in recent versions).

**Why →** Redis is only a cache, but its restarts cause slow UIs and repo-server bursts.

**Interview tip →** Remind that losing Redis loses no state.

</details>

### Q90. Your team must prove a full disaster recovery: new region, new cluster, everything back in under an hour. How does GitOps help, and what is not covered? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** GitOps restores configuration, not data.

**What I'd do →**
1. Terraform the new cluster; install Argo CD; restore repo/cluster credentials; apply the root app. Add-ons and apps converge from Git.
2. Restore data separately: database snapshots, Velero for PVs, object storage replication.
3. Restore Sealed Secrets keys, or rely on External Secrets with a replicated secret manager.
4. Switch DNS and traffic.
5. Rehearse quarterly and time each step.

**Why →** Stateless configuration is quick to rebuild from Git; state and secrets are the long poles.

**Interview tip →** Explicitly separating config recovery from data recovery is what interviewers look for.

</details>

### Q91. A cluster's API server is unreachable and its apps show `Unknown`. Other clusters are slow too. What do you suspect and do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A dead cluster can tie up controller workers and connections on its shard.

**What I'd do →**
1. Confirm with `argocd cluster list` (connection status) and controller logs.
2. If the outage is long, temporarily remove or label the cluster out of generators, or skip-reconcile its apps.
3. Make sure sharding spreads clusters across controller replicas, so one dead cluster only affects its own shard.
4. When the cluster is back, run a hard refresh on its apps and confirm `argocd cluster list` shows `Successful`.

**Why →** Timeouts against an unreachable API server consume processing time shared with other clusters on that shard.

**Interview tip →** Shows awareness that clusters share shard resources.

</details>

### Q92. You are on call and need to stop all automated syncs across Argo CD immediately during an incident. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** There is no single global auto-sync switch.

**What I'd do →**
1. Add a `deny` sync window covering all applications in affected projects (`argocd proj windows add <proj> --kind deny --schedule "* * * * *" --duration 4h --applications '*'`), allowing manual sync if needed.
2. For a total freeze, scale the application controller to 0 (stops reconciliation entirely; workloads keep running).
3. Record the action in the incident channel and remove it afterwards.

**Why →** Sync windows are the supported, auditable freeze; scaling the controller is the big hammer.

**Interview tip →** Mention that self-heal is also an automated sync, so a deny window blocks it too, and the window is visible in the UI so everyone knows a freeze is active.

</details>

### Q93. The repo-server pods restart frequently with OOMKilled during busy periods. How do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Concurrent heavy renders (Helm dependencies, big Kustomize trees).

**What I'd do →**
1. Raise memory limits and add replicas.
2. Set `reposerver.parallelism.limit` (for example 10) to cap concurrent renders per pod.
3. Use webhooks plus `manifest-generate-paths` to reduce unnecessary renders.
4. Vendor Helm dependencies in Git to avoid repeated `helm dependency build`, and check for huge repos.

**Why →** Each render can use hundreds of MB; unlimited parallelism multiplies it.

**Interview tip →** Mention that OOM in the repo-server shows up as `ComparisonError` with `connection refused` or `Unavailable` on apps.

</details>

### Q94. You want alerting on Argo CD health. What alerts do you create? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Detect both delivery failures and Argo CD problems.

**What I'd do →**
1. Apps: `argocd_app_info{health_status="Degraded", project=~"prod.*"}` for more than 10 minutes; OutOfSync for over 30 minutes on auto-sync apps.
2. Syncs: increase of `argocd_app_sync_total{phase=~"Error|Failed"}`.
3. Platform: `argocd_cluster_connection_status == 0`, controller workqueue depth high, repo-server error rate, Redis down.
4. Route app alerts to owning teams (project label), platform alerts to platform on-call.

**Why →** Teams need to know their deploy failed; platform needs to know Argo CD itself is failing.

**Interview tip →** Combine Prometheus alerts for platform with notifications for per-app events.

</details>

### Q95. A new controller shard was added, and some apps briefly showed as Unknown and re-synced. Why, and how do you avoid it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reassignment of clusters across shards.

**What I'd do →**
1. Explain: changing replicas changes the shard mapping; clusters move to new pods that must rebuild their cluster cache, during which apps show Unknown.
2. Use `consistent-hashing` to minimize movement when shard count changes.
3. Make the change in a quiet window and pre-check distribution with `argocd admin cluster shards`.
4. Ensure `ARGOCD_CONTROLLER_REPLICAS` matches the StatefulSet replicas during the change.

**Why →** Cache warm-up on reassignment is unavoidable but can be minimized.

**Interview tip →** Knowing why consistent hashing exists shows depth.

</details>

### Q96. Leadership asks for a monthly report of deployment frequency and lead time from GitOps data. How do you produce it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** DORA metrics from Git and Argo CD.

**What I'd do →**
1. Deployment frequency: count successful prod syncs (`argocd_app_sync_total{phase="Succeeded"}` with prod project labels) or prod-folder merges.
2. Lead time: time from app repo commit to the prod sync of that tag (join CI metadata, config repo commits and Argo CD sync timestamps via notifications to a datastore).
3. Change failure rate: aborted rollouts and reverts in `envs/prod`.
4. Visualize in Grafana.

**Why →** GitOps gives timestamped events at each stage, which is exactly what DORA needs.

**Interview tip →** Mention emitting notification webhooks into a metrics pipeline.

</details>

### Q97. You want to test changes to the bootstrap app-of-apps without risking production clusters. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Bootstrap changes affect everything.

**What I'd do →**
1. A dedicated ephemeral cluster (kind in CI or a sandbox) that bootstraps from the PR branch: install Argo CD, apply root app with `targetRevision` set to the branch.
2. Wait for all apps healthy with `argocd app wait`, then tear down.
3. Production root apps track a tag or `main` only after merge.

**Why →** End-to-end bootstrap tests catch ordering and health problems that rendering alone misses.

**Interview tip →** This is the GitOps equivalent of integration tests for infrastructure.

</details>

### Q98. A team's app takes minutes to show changes after merge even with webhooks configured. What do you check? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Webhook not effective.

**What I'd do →**
1. Check the Git provider's webhook delivery log (response code from `/api/webhook`).
2. Check argocd-server logs for webhook receipt and secret validation errors.
3. Confirm the webhook URL reaches argocd-server through the ingress (not blocked by SSO proxy).
4. Check `manifest-generate-paths` on the app; if paths do not match the changed files, no refresh happens.

**Why →** Without a working webhook, the app waits for the polling interval.

**Interview tip →** Mention that a wrong webhook secret results in silent ignores on the Argo CD side.

</details>

### Q99. Your team wants to move from Argo CD to Flux (or the reverse). How do you migrate safely? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two GitOps engines must not manage the same objects.

**What I'd do →**
1. Keep manifests as they are; write Flux Kustomizations or HelmReleases (or Argo CD Applications) for the same paths.
2. Per app: remove it from the old tool without cascade (Argo CD `--cascade=false`; Flux `flux suspend`, then delete the Kustomization with `prune: false`).
3. Create it in the new tool, verify no recreation, enable pruning.
4. Note Helm differences: Flux's helm-controller creates real Helm releases where Argo CD had none, so plan adoption of Helm-managed objects.

**Why →** Overlap or cascade during migration is the main risk.

**Interview tip →** The Helm release difference is a frequently missed migration detail.

</details>

### Q100. After a big refactor of the config repo, you want to verify that nothing in the cluster will change. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Refactor should be a no-op for live state.

**What I'd do →**
1. Render both versions in CI (`helm template`, `kustomize build`) and diff the outputs; any difference must be intended.
2. Point a copy of an Application at the PR branch with no sync policy, or run `argocd app diff <app> --local <path>` / `--revision <branch>` for each affected app.
3. Merge only when diffs are empty, with prune disabled for the first sync.

**Why →** Rendering diffs prove equivalence before Argo CD acts.

**Interview tip →** Mention tools like `argocd-diff-preview` or rendered-manifest diffs in PR comments.

</details>

---

## OutOfSync Loops & Diff Problems

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. An app flips between `Synced` and `OutOfSync` every few seconds and the sync history grows constantly. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Self-heal fighting another writer: an HPA on `replicas`, an operator rewriting a field, a mutating webhook, or a second Application managing the same object.

**Diagnose →**
```bash
argocd app diff web
argocd app history web | tail -20
kubectl get deploy web -n web -o yaml --show-managed-fields | grep -B2 -A8 manager
```

**Fix →** Remove the contested field from Git, or add `ignoreDifferences` for it with `RespectIgnoreDifferences=true`. If two apps own the resource, give it one owner.

**Prevent →** One owner per field; enable `FailOnSharedResource=true`; review managed fields when adopting operators.

</details>

### Q102. `argocd app diff` shows only `replicas: 2` vs `replicas: 5` on a Deployment that has an HPA, and the app never stays Synced. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The manifest declares `spec.replicas` while the HPA scales it.

**Diagnose →**
```bash
kubectl get hpa -n web
argocd app diff web
```

**Fix →** Remove `replicas` from the manifest (Helm: wrap it in `{{- if not .Values.autoscaling.enabled }}`), or:
```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers: [/spec/replicas]
syncPolicy:
  syncOptions: [RespectIgnoreDifferences=true]
```

**Prevent →** Standard chart pattern: never render replicas when autoscaling is enabled.

</details>

### Q103. A MutatingWebhookConfiguration is always OutOfSync on `webhooks[0].clientConfig.caBundle`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** cert-manager's CA injector or the operator itself writes `caBundle` after Argo CD applies an empty or stale value.

**Diagnose →** `argocd app diff <app>` shows `caBundle` changes only; `kubectl get mutatingwebhookconfiguration <name> -o yaml` shows the `cert-manager.io/inject-ca-from` annotation.

**Fix →**
```yaml
ignoreDifferences:
  - group: admissionregistration.k8s.io
    kind: MutatingWebhookConfiguration
    jqPathExpressions:
      - '.webhooks[]?.clientConfig.caBundle'
```
Add `RespectIgnoreDifferences=true`.

**Prevent →** Put it in `resource.customizations.ignoreDifferences` globally for webhook configurations, since every injected webhook behaves this way.

</details>

### Q104. After installing Istio, every Deployment shows OutOfSync because of injected sidecar fields. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Usually it is not the Deployment but Pods or objects you declared that Istio mutates (for example annotations Istio adds to Services or webhook configs), or label tracking colliding with Istio labels. Pod injection happens on Pods, which Argo CD does not diff against Git.

**Diagnose →** `argocd app diff` to find the exact resource and field; check whether you declare objects Istio controls (for example `istio-injection` labels on namespaces managed by two places).

**Fix →** Ignore the injected fields with `jqPathExpressions` or `managedFieldsManagers: [pilot-discovery]`, or enable `ServerSideDiff=true` so webhook mutations are part of the expected state.

**Prevent →** Server-side diff for meshes and policy engines that mutate on admission.

</details>

### Q105. A ConfigMap is OutOfSync and the diff shows the same data in a different format (a number became a string). 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** YAML typing: `data` values in a ConfigMap must be strings; `port: 8080` unquoted is coerced, or a Helm template outputs a number.

**Diagnose →** `argocd app diff web` shows `port: 8080` vs `port: "8080"`; `helm template` output confirms the unquoted value.

**Fix →** Quote values (`{{ .Values.port | quote }}`) or write `"8080"` in plain YAML.

**Prevent →** Lint rendered manifests with kubeconform in CI; it rejects non-string ConfigMap data.

</details>

### Q106. A Deployment is OutOfSync on `resources.limits.cpu`: Git says `1000m`, live says `1`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The API server normalizes quantities. Argo CD normalizes most cases, but some combinations (especially inside CRDs, which are not normalized by Argo CD's known-type logic) remain.

**Diagnose →** `argocd app diff` shows the quantity formats; check whether the object is a built-in kind or a CRD.

**Fix →** Write canonical values (`cpu: "1"`, `memory: 512Mi`) or enable `ServerSideDiff=true`.

**Prevent →** Standardize quantity formats in chart defaults.

</details>

### Q107. An app is OutOfSync on a field that is not in Git at all, after an Argo CD upgrade. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Diff behaviour changed between versions (normalization, server-side diff defaults, tracking method change), or the `last-applied-configuration` annotation contains the field from an older manifest so three-way diff still considers it yours.

**Diagnose →**
```bash
argocd app diff web
kubectl get <kind> <name> -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}' | jq .
```

**Fix →** Sync once to refresh `last-applied-configuration`; if the field is defaulted, use server-side diff; if tracking changed, re-sync to add the new tracking annotation.

**Prevent →** Upgrade a staging instance first and compare the count of OutOfSync apps before and after.

</details>

### Q108. The app shows `SharedResourceWarning: Namespace/payments is part of applications argocd/payments-api and payments-worker`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both apps render the same Namespace (or ConfigMap, ClusterRole).

**Diagnose →** `argocd app manifests payments-api | grep -A3 'kind: Namespace'` and the same for the other app.

**Fix →** Keep the resource in exactly one app (or use `CreateNamespace=true` and `managedNamespaceMetadata` in one app only) and remove it from the other.

**Prevent →** `FailOnSharedResource=true` in sync options turns the warning into a failure.

</details>

### Q109. A Secret generated by a Helm chart (`genCA`/`randAlphaNum`) shows OutOfSync after every refresh. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Argo CD runs `helm template` on each render, so random or generated values change each time, and `lookup` (which charts use to reuse existing values) returns nothing.

**Diagnose →** `argocd app diff` shows the Secret's data changing; the chart template contains `randAlphaNum`, `genCA` or `lookup`.

**Fix →** Supply the value via `existingSecret` (Sealed Secrets or ESO), or set the value explicitly in values. Short-term: `ignoreDifferences` on `/data` for that Secret plus `RespectIgnoreDifferences=true`.

**Prevent →** Prefer charts that support `existingSecret`; check for `lookup` before adopting a chart.

</details>

### Q110. An app stays OutOfSync showing a resource marked "requires pruning", but auto-sync is on. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `automated.prune` is false, or the resource has `Prune=false`, or `Prune=confirm` waiting for approval.

**Diagnose →**
```bash
argocd app get web -o yaml | yq '.spec.syncPolicy'
kubectl get <kind> <name> -o jsonpath='{.metadata.annotations.argocd\.argoproj\.io/sync-options}'
```

**Fix →** `argocd app sync web --prune` after reviewing, or enable `prune: true`, or confirm the prune in the UI.

**Prevent →** Decide per app whether automated pruning is acceptable and document it.

</details>

### Q111. After switching to annotation tracking, the UI shows every resource twice or many as "extraneous". 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Resources still have the old `app.kubernetes.io/instance` label and do not yet have the `argocd.argoproj.io/tracking-id` annotation; or operator-created objects copied the label and now look orphaned or extraneous.

**Diagnose →** `kubectl get <kind> <name> -o yaml | grep -E 'instance|tracking-id'`; `argocd app get <app>` for extraneous resources.

**Fix →** Sync each app once so the tracking annotation is written. Consider `annotation+label` during the transition. Mark operator-generated objects with `IgnoreExtraneous`.

**Prevent →** Plan the tracking change as a migration: staged, with syncs per app.

</details>

### Q112. An `ExternalSecret` is always OutOfSync with fields like `conversionStrategy: Default` and `decodingStrategy: None` appearing in live. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ESO's CRD defaults these fields; your manifest omits them, and CRD defaults are not normalized by client-side diff.

**Diagnose →** `argocd app diff` lists exactly those defaulted fields under `remoteRef`.

**Fix →** Add the default values explicitly in Git, or enable `ServerSideDiff=true` for the app.

**Prevent →** Use server-side diff for apps heavy in CRDs with schema defaults.

</details>

### Q113. A CRD is OutOfSync on `status` or `metadata.annotations` fields that its controller sets. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You included `status` in the manifest (copied from `kubectl get -o yaml`), or the controller writes annotations you also declare.

**Diagnose →** `argocd app diff` shows `status:` or controller annotations.

**Fix →** Remove `status` and server-populated fields from Git (`kubectl neat` style cleanup); ignore controller-owned annotations via jq paths.

**Prevent →** Never commit raw `kubectl get -o yaml` output; strip `status`, `uid`, `resourceVersion`, `managedFields`, `creationTimestamp`.

</details>

### Q114. `ignoreDifferences` is configured for `/spec/replicas`, the diff is clean, but replicas still get reset on every sync. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing `RespectIgnoreDifferences=true`; the sync still applies the whole manifest.

**Diagnose →** `argocd app get web -o yaml | yq '.spec.syncPolicy.syncOptions'`.

**Fix →**
```yaml
syncPolicy:
  syncOptions:
    - RespectIgnoreDifferences=true
```

**Prevent →** Always pair field ignores with this option when another controller owns the field.

</details>

### Q115. Your `ignoreDifferences` rule with a jq expression has no effect; the field is still shown in the diff. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong group or kind (core group must be `""`), wrong name/namespace filter, or a jq expression that does not match (missing `[]?`, wrong path).

**Diagnose →** Test the expression on the live object:
```bash
kubectl get mutatingwebhookconfiguration my-hook -o json | jq '.webhooks[]?.clientConfig.caBundle'
argocd admin settings resource-overrides ignore-differences ./obj.yaml --argocd-cm-path ./argocd-cm.yaml
```

**Fix →** Correct the selector fields and expression; refresh the app.

**Prevent →** Keep a small test fixture per global ignore rule and test in CI.

</details>

### Q116. An app is OutOfSync because the live object has an empty field that Git omits, for example `securityContext: {}` or `env: []`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Usually the reverse: Git declares an empty value (`env: []`, `annotations: {}`) that the API server drops, so the diff sees a declared field missing in live.

**Diagnose →** `argocd app diff` shows the empty structure; check the rendered manifest.

**Fix →** Remove empty maps and lists from templates (Helm: `{{- with .Values.env }}` blocks).

**Prevent →** Template patterns that omit empty blocks.

</details>

### Q117. A Service of type `LoadBalancer` shows OutOfSync on `spec.ports[0].nodePort` and `healthCheckNodePort`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You declared `nodePort` values that differ from what was allocated, or a previous manifest declared them (last-applied still lists them).

**Diagnose →** Compare `argocd app diff` with `kubectl get svc -o yaml`.

**Fix →** Remove `nodePort` from Git unless you need fixed ports; resync to refresh last-applied.

**Prevent →** Let Kubernetes allocate node ports.

</details>

### Q118. A self-heal loop makes the application controller CPU spike and other apps lag. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A tight fight between self-heal and another controller; each revert triggers a watch event and another sync.

**Diagnose →**
```bash
argocd app list -o wide | grep OutOfSync
kubectl -n argocd logs sts/argocd-application-controller | grep -c 'Initiated automated sync'
```
Check `argocd_app_sync_total` rate per app.

**Fix →** Temporarily disable self-heal on the offending app, fix the field ownership, re-enable.

**Prevent →** Alert on sync rate per app; newer versions back off self-heal retries, keep them on defaults rather than shortening them.

</details>

### Q119. An app is OutOfSync because the Helm chart renders `apiVersion: policy/v1beta1` for PodDisruptionBudget, and live shows `policy/v1`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Old chart using a removed API, or `.Capabilities.APIVersions` not seeing `policy/v1` so the chart picks the old one; on 1.25+ the old API no longer exists and apply fails.

**Diagnose →** `argocd app manifests web | grep -B2 PodDisruptionBudget`; check the chart's capability conditions.

**Fix →** Upgrade the chart; if needed set `kubeVersion` or `apiVersions` under `spec.source.helm` so the chart renders correctly.

**Prevent →** Run `pluto` or kubeconform with the target Kubernetes version in CI.

</details>

### Q120. An app shows `OutOfSync` with the message `ComparisonError: ... unable to resolve 'main' to a commit SHA`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The branch or tag in `targetRevision` does not exist (renamed to `master`/`main`, typo, deleted tag).

**Diagnose →**
```bash
git ls-remote https://github.com/org/cfg.git | grep -E 'refs/heads|refs/tags' | head
argocd app get web -o yaml | yq '.spec.source.targetRevision'
```

**Fix →** Correct `targetRevision` or recreate the tag.

**Prevent →** Use `HEAD` for default-branch tracking, and protect release tags from deletion.

</details>

---

## Manifest Generation & ComparisonError

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q121. `ComparisonError: Failed to load target state: failed to generate manifest for source 1 of 1: rpc error: code = Unknown desc = Manifest generation error (cached): ...` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Rendering failed (Helm or Kustomize error) and the repo-server cached the failure; it returns the cached error until the cache expires or something changes.

**Diagnose →** Read the rest of the message for the real error; reproduce locally with `helm template` or `kustomize build` at the same commit; check `kubectl -n argocd logs deploy/argocd-repo-server`.

**Fix →** Fix the manifests and push, then `argocd app get web --hard-refresh` to bypass the cached error.

**Prevent →** Render every app in CI on PRs to the config repo.

</details>

### Q122. `rpc error: code = Unknown desc = `helm template . --name-template web --namespace web ... --include-crds` failed exit status 1: Error: template: web/templates/deployment.yaml:24:20: executing "web/templates/deployment.yaml" at <.Values.image.tag>: nil pointer evaluating interface {}.tag` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A values key is missing: the value file for this environment does not define `image`, or the value file path is wrong so it was never loaded.

**Diagnose →**
```bash
helm template web charts/web -f envs/prod/values.yaml --namespace web
argocd app get web -o yaml | yq '.spec.source.helm.valueFiles'
```

**Fix →** Add the missing values or default in the template (`{{ .Values.image.tag | default .Chart.AppVersion }}`), and fix the value file path.

**Prevent →** `helm lint` with each environment's values in CI; add `values.schema.json` to the chart.

</details>

### Q123. `Error: open envs/prod/values.yaml: no such file or directory` when the app renders. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `valueFiles` paths are relative to the chart `path`, not the repo root.

**Diagnose →** Compare `spec.source.path` (for example `charts/web`) with the value file location.

**Fix →** Use `../../envs/prod/values.yaml`, or with multiple sources reference `$values/envs/prod/values.yaml`. Use `ignoreMissingValueFiles: true` only for optional files.

**Prevent →** Render in CI with exactly the same relative paths Argo CD uses.

</details>

### Q124. `Error: values file '../../envs/prod/values.yaml' resolved to outside repository root` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The relative path climbs out of the repository (too many `../`), or points at a file outside the repo; Argo CD blocks this for security.

**Diagnose →** Count directories from the chart path to the repo root.

**Fix →** Correct the relative path. If values live in another repo, use multiple sources with `ref: values` and `$values/...`.

**Prevent →** Keep a consistent depth for chart folders or use multi-source references.

</details>

### Q125. Kustomize app fails: `kustomize build ... failed exit status 1: Error: accumulating resources: accumulation err='accumulating resources from '../../base': ... must build at directory: not a valid directory'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong relative path in `resources`, a folder renamed, or a remote base unreachable.

**Diagnose →** `kustomize build apps/web/overlays/prod` locally at the same commit.

**Fix →** Fix the path in `kustomization.yaml`. For remote bases, pin a ref and ensure the repo-server can reach and authenticate to it.

**Prevent →** `kustomize build` every overlay in CI.

</details>

### Q126. `Error: trouble configuring builtin HelmChartInflationGenerator with config: ... must specify --enable-helm` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A `kustomization.yaml` uses `helmCharts`, but Argo CD runs Kustomize without `--enable-helm`.

**Diagnose →** `kubectl -n argocd get cm argocd-cm -o yaml | grep kustomize.buildOptions`.

**Fix →**
```yaml
data:
  kustomize.buildOptions: --enable-helm
```
Restart is not required; refresh the app.

**Prevent →** Keep `argocd-cm` in Git with build options documented.

</details>

### Q127. `rpc error: code = DeadlineExceeded desc = context deadline exceeded` on several apps' comparison. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The repo-server is too slow: heavy Helm renders, large monorepo fetches, slow Git or Helm repo, saturated repo-server pods or OOM restarts.

**Diagnose →**
```bash
kubectl -n argocd top pods
kubectl -n argocd logs deploy/argocd-repo-server --tail 200 | grep -iE 'timeout|took'
kubectl -n argocd get pods -l app.kubernetes.io/name=argocd-repo-server
```

**Fix →** Raise `controller.repo.server.timeout.seconds` and `ARGOCD_EXEC_TIMEOUT`, scale repo-server replicas and memory, set `reposerver.parallelism.limit`.

**Prevent →** Webhooks plus `manifest-generate-paths`, vendored chart dependencies, and repo-server metrics alerts.

</details>

### Q128. `rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp 10.96.12.4:8081: connect: connection refused"` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The controller or server cannot reach `argocd-repo-server` on 8081: repo-server pods crashed (OOM), are restarting, or a NetworkPolicy blocks it.

**Diagnose →**
```bash
kubectl -n argocd get pods -l app.kubernetes.io/name=argocd-repo-server
kubectl -n argocd describe pod <repo-server-pod> | grep -A5 'Last State'
kubectl -n argocd get networkpolicy
```

**Fix →** Fix the crash (memory limits, parallelism), restore network policy rules allowing controller and server to repo-server.

**Prevent →** Multiple repo-server replicas with a PDB and memory alerts.

</details>

### Q129. Helm dependency error: `Error: no cached repository for helm-manager-... found. (try 'helm repo update')` or `Error: found in Chart.yaml, but missing in charts/ directory: postgresql`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An umbrella chart's dependency repository is not registered or reachable from Argo CD, or credentials for a private dependency repo are missing.

**Diagnose →** Check `Chart.yaml` dependencies and `argocd repo list`; try `helm dependency build` locally.

**Fix →** Register the dependency repository in Argo CD (`argocd repo add <url> --type helm --name <name>`), add credentials, or vendor the dependency tarballs into `charts/` with `Chart.lock`.

**Prevent →** Vendor dependencies for production charts so renders do not depend on external repos.

</details>

### Q130. `failed to fetch chart: failed to download chart: unexpected status code 401` for an OCI Helm chart. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing or wrong registry credentials; for ECR, an expired token (12 hours); OCI not enabled on the repo Secret.

**Diagnose →** `argocd repo list`; check the repo Secret for `type: helm` and `enableOCI: "true"`; `helm pull oci://<registry>/<chart> --version <v>` with the same creds.

**Fix →** Update credentials; for ECR, refresh with a CronJob or an ESO `ECRAuthorizationToken` generator writing the repo Secret.

**Prevent →** Use short-lived credential automation for cloud registries.

</details>

### Q131. The repo-server logs `Error: unknown command "kustomize" for "argocd"` or a plugin app shows `couldn't find cmp-server plugin`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Application references a plugin that is not installed as a sidecar, the sidecar is crash-looping, or the plugin name (including version suffix) does not match.

**Diagnose →**
```bash
kubectl -n argocd get pod -l app.kubernetes.io/name=argocd-repo-server -o jsonpath='{.items[0].spec.containers[*].name}'
kubectl -n argocd logs deploy/argocd-repo-server -c <plugin-sidecar>
```

**Fix →** Install or fix the CMP sidecar with its `plugin.yaml`; reference it as `plugin.name: <name>-<version>` when a version is set, or let discovery match it.

**Prevent →** Version and test plugin sidecars as part of the Argo CD deployment.

</details>

### Q132. App fails with `Unable to generate manifests in apps/web: rpc error: code = Unknown desc = apps/web: app path does not exist`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong `path`, a folder renamed in Git, or `targetRevision` points at a branch where the path does not exist yet.

**Diagnose →** `git ls-tree -d <revision> apps/` in the config repo; `argocd app get web -o yaml | yq '.spec.source'`.

**Fix →** Correct the path or revision.

**Prevent →** Generate Applications with a git directory generator so paths always exist.

</details>

### Q133. `Kustomize build failed: ... json: unknown field "patchesStrategicMerge"` warnings, or a deprecated-field error after an Argo CD upgrade. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Kustomize version bundled with the new Argo CD deprecates or changes fields (`patchesStrategicMerge`, `commonLabels`, `bases`).

**Diagnose →** `kubectl -n argocd exec deploy/argocd-repo-server -- kustomize version`; run the same version locally.

**Fix →** Run `kustomize edit fix` to migrate files; or pin an older Kustomize version via `kustomize.path.<version>` in `argocd-cm` and `spec.source.kustomize.version` temporarily.

**Prevent →** Track the bundled tool versions in Argo CD release notes.

</details>

### Q134. A Jsonnet or directory app renders nothing and the app is Synced with zero resources, though files exist. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Files are in subfolders and `directory.recurse` is false, or `include`/`exclude` globs filter them out, or the files have extensions Argo CD ignores.

**Diagnose →** `argocd app manifests web` returns empty; check `spec.source.directory`.

**Fix →** Set `directory: {recurse: true}` or fix globs; use `.yaml`, `.yml`, `.json` or `.jsonnet` extensions.

**Prevent →** Keep `allowEmpty: false` on automated sync so an empty render cannot prune everything.

</details>

### Q135. `ComparisonError: ... error unmarshaling JSON: while decoding JSON: json: cannot unmarshal string into Go value of type map[string]interface {}` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A rendered file contains something that is not a Kubernetes object: a stray YAML document with only a string, a values file inside the manifests path, a README in YAML, or broken indentation.

**Diagnose →** `argocd app manifests web --source git` or local render; look for documents without `apiVersion`/`kind`.

**Fix →** Remove non-manifest YAML from the path, or exclude it with `directory.exclude`, or fix the template output.

**Prevent →** kubeconform in CI rejects non-object documents.

</details>

### Q136. App rendering succeeds locally but fails in Argo CD with `Error: chart requires kubeVersion: >=1.29.0-0 which is incompatible with Kubernetes v1.20.0`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Helm's default kubeVersion is used, because Argo CD does not have the destination cluster's version (cluster info not yet loaded, cluster unreachable), or an explicit wrong `kubeVersion` override.

**Diagnose →** `argocd cluster get <cluster>` for server version and connection status.

**Fix →** Fix cluster connectivity so cluster info is populated; or set `spec.source.helm.kubeVersion` explicitly.

**Prevent →** Monitor cluster connection status; avoid hardcoding kubeVersion.

</details>

### Q137. `Unable to create application: application spec for web is invalid: InvalidSpecError: Unable to generate manifests in apps/web: rpc error: code = Unknown desc = ... repository not accessible` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Argo CD validates the source at create time and cannot access the repo (credentials, URL typo, network).

**Diagnose →** `argocd repo list` to see if the repo is registered and its connection status; `argocd repo get <url>`.

**Fix →** Register the repo with correct credentials or fix the URL; or create with `--validate=false` and fix later (not recommended).

**Prevent →** Credential templates for the org and consistent URL forms.

</details>

### Q138. Multi-source app error: `$values` is not substituted, Helm says `open $values/envs/prod/values.yaml: no such file or directory`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No source has `ref: values`, the ref name differs (`ref: config` but `$values` used), or the app uses `source` (singular) instead of `sources`.

**Diagnose →** `argocd app get web -o yaml | yq '.spec.sources'`.

**Fix →** Add `ref: values` to the Git source, match the name in the path, and use `sources:` list form.

**Prevent →** Template multi-source apps from a known-good example.

</details>

---

## Repos, Clusters, Projects & Permissions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q139. Adding a private GitHub repo over HTTPS fails with `rpc error: code = Unknown desc = authentication required`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No credentials stored for that URL, an expired or revoked personal access token, a token without `repo`/`contents:read` scope, or the repo secret is missing the `argocd.argoproj.io/secret-type: repository` label.

**Diagnose →**
```bash
argocd repo list
kubectl -n argocd get secrets -l argocd.argoproj.io/secret-type=repository
git ls-remote https://<user>:<token>@github.com/org/config.git   # test the token outside Argo CD
```

**Fix →** Re-add the repo with a valid token: `argocd repo add https://github.com/org/config.git --username git --password <token> --upsert`.

**Prevent →** Use a GitHub App credential template instead of personal tokens, and alert on repo connection failures.

</details>

### Q140. An SSH repo fails with `ssh: handshake failed: knownhosts: key is unknown`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Git host's SSH key is not in `argocd-ssh-known-hosts-cm`. Common with self-hosted GitLab/Bitbucket, or after the host rotated its key.

**Diagnose →** `argocd cert list --cert-type ssh` and compare with `ssh-keyscan git.example.com`.

**Fix →**
```bash
ssh-keyscan git.example.com | argocd cert add-ssh --batch
```
Or add the lines to `argocd-ssh-known-hosts-cm` declaratively. Do not set `insecure: true` in production.

**Prevent →** Manage known hosts in Git with the rest of the Argo CD config and update them when the host rotates keys.

</details>

### Q141. SSH repo: `ssh: handshake failed: ssh: unable to authenticate, attempted methods [none publickey], no supported methods remain`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong private key, the public half was never added as a deploy key, the key has a passphrase (Argo CD cannot use it), or the URL uses a user other than `git`.

**Diagnose →** `ssh -i key -T git@github.com` from a workstation; check the secret's `sshPrivateKey` field starts with `-----BEGIN OPENSSH PRIVATE KEY-----` and has a trailing newline.

**Fix →** Generate a passphrase-less deploy key (`ssh-keygen -t ed25519 -N ""`), add the public key read-only on the repo, and register: `argocd repo add git@github.com:org/config.git --ssh-private-key-path ./key --upsert`.

**Prevent →** One deploy key per repo, rotated on a schedule, stored through External Secrets.

</details>

### Q142. A self-hosted GitLab repo fails with `x509: certificate signed by unknown authority`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** GitLab uses a certificate from an internal CA that the repo-server does not trust.

**Diagnose →** `openssl s_client -connect gitlab.internal:443 -showcerts` to see the chain; `argocd cert list --cert-type https`.

**Fix →**
```bash
argocd cert add-tls gitlab.internal --from ./internal-ca.pem
```
This writes to `argocd-tls-certs-cm`, which is mounted into repo-server. For Helm OCI registries on the same CA, the same entry applies by hostname.

**Prevent →** Keep the internal CA bundle in the Argo CD install values; never use `--insecure-skip-server-verification` as a permanent fix.

</details>

### Q143. A credential template exists for `https://github.com/org`, yet new repos under the org still fail with `authentication required`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** URL mismatch: the template is `https://github.com/org` but apps use `git@github.com:org/...` (SSH), or a different host alias like `https://www.github.com`. Templates match by URL prefix, so protocol and host must match exactly. A labelled `repo-creds` secret is also required (`secret-type: repo-creds`).

**Diagnose →** `argocd repocreds list`; compare its URL with the exact `repoURL` in the Application.

**Fix →** Standardise all `repoURL`s on one form, or add a template for the other form.

**Prevent →** Validate `repoURL` format in CI with a simple lint rule.

</details>

### Q144. Sync fails with `application repo https://github.com/org/payments-config.git is not permitted in project 'team-a'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The AppProject's `sourceRepos` does not include the repo, often because of a missing `.git` suffix or a different URL form.

**Diagnose →** `argocd proj get team-a` and look at Source Repositories.

**Fix →** `argocd proj add-source team-a https://github.com/org/payments-config.git`, or use a glob in Git: `sourceRepos: ["https://github.com/org/payments-*"]`.

**Prevent →** Manage AppProjects in Git, reviewed by the platform team, with globs scoped per team.

</details>

### Q145. `application destination server 'https://kubernetes.default.svc' and namespace 'payments' do not match any of the allowed destinations in project 'team-a'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The project's `destinations` list does not include that server/namespace pair, or the app uses `destination.name` while the project lists only `server` (or the reverse in older versions).

**Diagnose →** `argocd proj get team-a -o yaml | yq '.spec.destinations'`.

**Fix →**
```yaml
spec:
  destinations:
    - server: https://kubernetes.default.svc
      namespace: payments
```
Use `namespace: 'payments-*'` for a team prefix.

**Prevent →** Give each team a namespace prefix and a matching destination glob.

</details>

### Q146. Sync fails with `resource :Namespace is not permitted in project team-a.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Namespace is a cluster-scoped kind, and the project's `clusterResourceWhitelist` is empty (the default for a new project). The same happens for ClusterRole, CRD and other cluster-scoped kinds.

**Diagnose →** `argocd proj get team-a -o yaml | yq '.spec.clusterResourceWhitelist'`.

**Fix →** Either let the platform team create the namespace and use `CreateNamespace=true` only where allowed, or whitelist the specific kind:
```yaml
clusterResourceWhitelist:
  - group: ""
    kind: Namespace
```

**Prevent →** Keep cluster-scoped resources in a platform-owned project; tenants get namespaced kinds only.

</details>

### Q147. A namespaced resource is blocked: `resource networking.k8s.io:NetworkPolicy is not permitted in project team-a.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The project has a `namespaceResourceBlacklist` entry for that kind, or a `namespaceResourceWhitelist` that omits it. Platforms often block NetworkPolicy, ResourceQuota and LimitRange so tenants cannot loosen guardrails.

**Diagnose →** `argocd proj get team-a -o yaml | yq '.spec.namespaceResource*'`.

**Fix →** If the block is intended, move the object to the platform repo. If not, remove it from the blacklist through a reviewed PR.

**Prevent →** Document which kinds tenants own; return a clear error in a CI policy check before merge.

</details>

### Q148. `InvalidSpecError: application references project team-a which does not exist`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The AppProject was not created yet (ordering in the bootstrap), it lives in a different namespace than the Argo CD control plane namespace, or a typo.

**Diagnose →** `kubectl -n argocd get appprojects`.

**Fix →** Create the project first. In app-of-apps bootstraps, put AppProjects in an earlier sync wave (`argocd.argoproj.io/sync-wave: "-1"`) than the Applications.

**Prevent →** Bootstrap order: projects, then platform apps, then tenant apps.

</details>

### Q149. With "apps in any namespace" enabled, an Application in `team-a` namespace shows `app is not allowed in project 'team-a', or the project does not exist`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The AppProject does not list the namespace in `spec.sourceNamespaces`, or the controller and server were not started with `--application-namespaces` covering it.

**Diagnose →**
```bash
kubectl -n argocd get cm argocd-cmd-params-cm -o yaml | grep application.namespaces
kubectl -n argocd get appproject team-a -o yaml | yq '.spec.sourceNamespaces'
```

**Fix →** Set `application.namespaces: team-*` in `argocd-cmd-params-cm`, restart server and controller, and add `sourceNamespaces: ["team-a"]` to the project.

**Prevent →** Treat both settings as one change in the platform repo; never use `"*"` in `sourceNamespaces` for tenant projects.

</details>

### Q150. A user clicks Sync and gets `rpc error: code = PermissionDenied desc = permission denied: applications, sync, team-a/web, sub: alice@example.com`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No RBAC policy grants `sync` on that project, the user's SSO group name does not match the group in `policy.csv`, or `scopes` in `argocd-rbac-cm` does not include the claim that carries groups.

**Diagnose →**
```bash
argocd account get-user-info                       # shows the groups Argo CD sees
argocd admin settings rbac can alice@example.com sync applications 'team-a/web' \
  --policy-file policy.csv
```

**Fix →** Add `p, role:team-a-dev, applications, sync, team-a/*, allow` and `g, <exact-group-name>, role:team-a-dev`. Set `scopes: '[groups, email]'` if groups are missing.

**Prevent →** Test RBAC changes in CI with `argocd admin settings rbac validate`.

</details>

### Q151. SSO login works but every user sees no applications. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `policy.default` is empty and the groups claim is not reaching Argo CD: the IdP does not send `groups` in the ID token, Azure AD sends group object IDs instead of names, or Okta's groups claim filter excludes the groups.

**Diagnose →** `argocd account get-user-info` after login; decode the ID token at the CLI (`argocd account get-user-info` shows `Groups:`); check Dex logs: `kubectl -n argocd logs deploy/argocd-dex-server`.

**Fix →** Configure the IdP to emit groups (Azure: `groupMembershipClaims`, map by object ID in `policy.csv`); set `policy.default: role:readonly` if read access for all is acceptable.

**Prevent →** Keep one break-glass local admin, disabled by default, and document the claim mapping.

</details>

### Q152. SSO login fails with `failed to get token: oauth2: cannot fetch token: 401 Unauthorized` or `invalid_client`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong or rotated client secret, the secret referenced as `$oidc.azure.clientSecret` does not exist in `argocd-secret`, or the redirect URI registered at the IdP differs from `https://argocd.example.com/auth/callback` (or `/api/dex/callback` for Dex).

**Diagnose →** `kubectl -n argocd logs deploy/argocd-server | grep -i oidc`; confirm `url:` in `argocd-cm` is the public URL users hit.

**Fix →** Update the client secret in `argocd-secret` (or the external secret it references), fix `url` and the registered redirect URI, then restart argocd-server.

**Prevent →** Calendar the IdP secret expiry (Azure secrets expire), and sync the secret through External Secrets.

</details>

### Q153. `argocd app list` from a laptop fails with `transport: received the unexpected content-type "text/plain; charset=utf-8"`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CLI speaks gRPC, but an ingress or load balancer in front of argocd-server only proxies HTTP/1.1 and returns an HTML or text error page.

**Diagnose →** Retry with `--grpc-web`; if that works, the proxy is the problem.

**Fix →** Use `argocd login argocd.example.com --grpc-web`, or configure the ingress for gRPC (separate gRPC host with `backend-protocol: GRPC` on ingress-nginx, or an ALB with a gRPC target group).

**Prevent →** Document the login command with `--grpc-web` for users behind corporate proxies.

</details>

### Q154. The UI behind ingress-nginx loops with `ERR_TOO_MANY_REDIRECTS`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** TLS terminates at the ingress, which talks HTTP to argocd-server, and argocd-server redirects HTTP to HTTPS, forever.

**Diagnose →** `curl -I http://argocd-server.argocd.svc` from a pod returns a 307 to https.

**Fix →** Either run the server in insecure mode behind TLS termination:
```yaml
# argocd-cmd-params-cm
data:
  server.insecure: "true"
```
or use SSL passthrough (`nginx.ingress.kubernetes.io/ssl-passthrough: "true"` plus `--enable-ssl-passthrough` on the controller). Restart argocd-server after the change.

**Prevent →** Pick one TLS model in the install values and keep it in Git.

</details>

### Q155. A CI job's `argocd app sync` fails with `account 'ci-bot' does not have apiKey capability`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The local account was created with only `login` capability, or not created at all.

**Diagnose →** `argocd account list` shows capabilities per account.

**Fix →**
```yaml
# argocd-cm
data:
  accounts.ci-bot: apiKey
```
Then `argocd account generate-token --account ci-bot --expires-in 720h` and add a scoped RBAC policy for `ci-bot`.

**Prevent →** Prefer project-scoped tokens (`argocd proj role create-token`) for CI, with an expiry, rotated automatically.

</details>

### Q156. A cluster shows `ConnectionStatus: Failed` with `the server has asked for the client to provide credentials`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The bearer token in the cluster secret expired or the ServiceAccount token secret was deleted; on EKS, the IAM role used by Argo CD lost its access entry or `aws-auth` mapping.

**Diagnose →**
```bash
argocd cluster get https://<api-endpoint>
kubectl -n argocd get secret -l argocd.argoproj.io/secret-type=cluster
```
For EKS: `aws eks list-access-entries --cluster-name prod`.

**Fix →** Re-run `argocd cluster add <context> --upsert` to mint a new token, or restore the EKS access entry for the Argo CD role (IRSA/Pod Identity with `awsAuthConfig` in the cluster secret).

**Prevent →** Use IAM-based auth (`execProviderConfig` or `awsAuthConfig`) instead of long-lived tokens; alert on `argocd_cluster_connection_status`.

</details>

### Q157. Destination fails with `unable to find destination server: there are no clusters with this name: prod-eu`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The cluster secret's `name` field differs (`prod-eu-1`), the secret is missing the `argocd.argoproj.io/secret-type: cluster` label, or the cluster was registered under a server URL only.

**Diagnose →** `argocd cluster list` shows names and servers.

**Fix →** Rename the cluster in its secret (`stringData.name: prod-eu`) or update the Application's `destination.name`.

**Prevent →** Generate cluster secrets from a single source (Terraform or a cluster ApplicationSet) with a naming convention.

</details>

### Q158. A cluster added with namespace scope shows `failed to sync cluster: ... deployments.apps is forbidden: User "system:serviceaccount:kube-system:argocd-manager" cannot list resource "deployments" in API group "apps" at the cluster scope`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ServiceAccount only has Roles in specific namespaces, but the cluster secret has no `namespaces` field, so the controller tries to watch cluster-wide.

**Diagnose →** `kubectl auth can-i list deployments -A --as system:serviceaccount:kube-system:argocd-manager` on the target cluster.

**Fix →**
```bash
argocd cluster add team-ctx --namespace payments --namespace payments-jobs --upsert
```
This sets `namespaces: payments,payments-jobs` in the secret. Add `clusterResources: "true"` only if cluster-scoped kinds are needed and granted.

**Prevent →** Pair namespace-scoped RBAC with the matching `namespaces` list, both generated from the same tenant definition.

</details>

### Q159. Repo connection works from a laptop, but in the cluster: `dial tcp 140.82.112.4:443: i/o timeout`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Egress is blocked: a NetworkPolicy on the `argocd` namespace, a firewall that only allows traffic via a corporate proxy, or a private cluster without NAT.

**Diagnose →**
```bash
kubectl -n argocd exec deploy/argocd-repo-server -- git ls-remote https://github.com/org/config.git
kubectl -n argocd get networkpolicy
```

**Fix →** Allow egress on 443/22 to the Git host, or set `proxy:` on the repository secret (or `HTTPS_PROXY`/`NO_PROXY` env on repo-server, with `NO_PROXY` including the cluster service CIDR and `.svc`).

**Prevent →** Include Argo CD egress requirements in the platform network design doc.

</details>

### Q160. Argo CD stops refreshing many apps, and repo-server logs show `remote: Repository not found` or `429 Too Many Requests` from GitHub during busy hours. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Hundreds of apps poll the same repos every 3 minutes with one token, hitting GitHub rate limits; GitHub returns misleading "not found" when a token is throttled or lacks access.

**Diagnose →** Count `git ls-remote` calls in repo-server logs; check the rate limit for the token: `curl -H "Authorization: Bearer $TOKEN" https://api.github.com/rate_limit`.

**Fix →** Configure webhooks and raise `timeout.reconciliation` (for example `600s`) in `argocd-cm`; switch to a GitHub App credential (higher limits); consolidate apps on fewer repos.

**Prevent →** Webhook-driven refresh plus jittered polling (`timeout.reconciliation.jitter`) as the fallback.

</details>

---

## Sync Operations, Hooks & Health

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q161. Sync fails with `one or more synchronization tasks are not valid` and `the server could not find the requested resource` for a custom resource. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CRD and its CRs are in the same sync, and the dry-run runs before the CRD exists, so the API server does not know the kind.

**Diagnose →** `argocd app get web` lists the failing CR kind; `kubectl get crd | grep <group>` on the target.

**Fix →** Add `argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true` to the CRs, or put the CRD in an earlier sync wave, or install CRDs from a separate app.

**Prevent →** Operators and their CRDs belong to a platform app that syncs before tenant apps.

</details>

### Q162. `one or more objects failed to apply, reason: Deployment.apps "web" is invalid: spec.selector: Invalid value: ...: field is immutable`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Someone changed labels used in `spec.selector` (often via a chart upgrade or a new `commonLabels` in Kustomize). Selectors and some fields (Job `spec.template`, Service `clusterIP`) cannot be updated in place.

**Diagnose →** `argocd app diff web` shows the selector change.

**Fix →** Revert the selector change, or let Argo CD recreate just that resource: annotate it with `argocd.argoproj.io/sync-options: Replace=true` (or `Force=true,Replace=true`) for one sync. Expect a brief outage for that Deployment.

**Prevent →** Use `labels` with `includeSelectors: false` in Kustomize instead of `commonLabels`; review chart upgrades for selector changes.

</details>

### Q163. Sync of a large CRD fails with `metadata.annotations: Too long: must have at most 262144 bytes`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Client-side apply stores the full object in the `kubectl.kubernetes.io/last-applied-configuration` annotation, which exceeds 256 KiB for big CRDs (Prometheus Operator, Crossplane, Argo's own).

**Diagnose →** The failing resource is almost always a CRD or a large ConfigMap.

**Fix →** Enable server-side apply for the app or that resource:
```yaml
syncPolicy:
  syncOptions:
    - ServerSideApply=true
```
`Replace=true` also works but recreates the object.

**Prevent →** Default to `ServerSideApply=true` for apps that ship CRDs.

</details>

### Q164. An app has been stuck at `Syncing` for an hour with `waiting for completion of hook batch/Job/db-migrate`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The hook Job never finishes: its pod is Pending (no capacity, missing secret, image pull error), or the migration hangs on a DB lock. Argo CD waits for hooks indefinitely unless the Job has a deadline.

**Diagnose →**
```bash
kubectl -n web get job db-migrate
kubectl -n web describe pod -l job-name=db-migrate
kubectl -n web logs job/db-migrate
```

**Fix →** Fix the pod problem, or end the operation: `argocd app terminate-op web`, then delete the Job and sync again.

**Prevent →** Set `activeDeadlineSeconds` and `backoffLimit` on every hook Job; alert on operations running longer than 15 minutes.

</details>

### Q165. A PreSync hook fails with `jobs.batch "db-migrate" already exists`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The previous run's Job was not deleted, because the hook has no `hook-delete-policy` or uses `HookSucceeded` and the last run failed.

**Diagnose →** `kubectl -n web get job db-migrate -o yaml | yq '.metadata.annotations'`.

**Fix →** Delete the old Job and set:
```yaml
annotations:
  argocd.argoproj.io/hook: PreSync
  argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
```
Or use `generateName` so every run gets a new Job.

**Prevent →** `BeforeHookCreation` as the default policy for all hook Jobs; it keeps the last failed Job for debugging.

</details>

### Q166. A PostSync smoke-test hook never runs, even though the sync says succeeded. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** PostSync hooks run only after all resources are Healthy. If something stays Progressing (an Ingress without an address, a PVC waiting for a consumer, a Rollout paused), the hook waits. Also, a hook placed in a file that the source excludes never renders.

**Diagnose →** `argocd app get web` and look for resources not Healthy; confirm the hook is in `argocd app manifests web`.

**Fix →** Fix the stuck resource or its health check (see Q167), then re-sync.

**Prevent →** Keep a short list of kinds with known slow health and add custom health checks for them.

</details>

### Q167. An Ingress stays `Progressing` forever, holding the app at Progressing. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Argo CD treats an Ingress as Healthy only when `status.loadBalancer.ingress` is set. Some controllers (or internal setups) never populate it.

**Diagnose →** `kubectl -n web get ingress web -o yaml | yq '.status'`.

**Fix →** Configure the controller to publish status (`--publish-service` on ingress-nginx), or override the health check:
```yaml
# argocd-cm
data:
  resource.customizations.health.networking.k8s.io_Ingress: |
    hs = {}
    hs.status = "Healthy"
    hs.message = "status not reported by this controller"
    return hs
```

**Prevent →** Verify health behaviour for each controller you adopt before onboarding apps.

</details>

### Q168. App health is `Degraded` with `Deployment "web" exceeded its progress deadline`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** New pods never became Ready within `progressDeadlineSeconds` (default 600): bad image tag, `CrashLoopBackOff`, failing readiness probe, or insufficient capacity.

**Diagnose →**
```bash
kubectl -n web rollout status deploy/web
kubectl -n web get pods
kubectl -n web describe pod <new-pod>
```

**Fix →** Fix the root cause in Git and let Argo CD sync. To restore service fast, revert the commit (`git revert`); `argocd app rollback` only works with auto-sync off.

**Prevent →** Readiness probes that reflect real health, and notifications on `on-health-degraded`.

</details>

### Q169. `argocd app rollback web 12` fails with `rollback cannot be initiated when auto-sync is enabled`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** By design: with auto-sync on, Git is the only source of truth, so a rollback would be undone immediately.

**Diagnose →** `argocd app get web -o yaml | yq '.spec.syncPolicy.automated'`.

**Fix →** The GitOps way: `git revert <bad-commit>` and push. For an emergency, disable auto-sync (`argocd app set web --sync-policy none`), roll back, then fix Git and re-enable.

**Prevent →** Teach on-call that rollback means revert in Git; keep the emergency path in the runbook with a reminder to re-enable auto-sync.

</details>

### Q170. Manual sync is refused with a message that the app is blocked by a sync window. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A `deny` window is active in the AppProject (for example a weekend freeze), or an `allow` window exists and the current time is outside it.

**Diagnose →** `argocd proj windows list team-a` shows active windows and whether `manualSync` is allowed.

**Fix →** If the change is an approved hotfix and the window has `manualSync: true`, sync manually. Otherwise ask the project owner to adjust the window through Git.

**Prevent →** Use `manualSync: true` on freeze windows so humans can still ship approved fixes; set `timeZone` explicitly to avoid UTC surprises.

</details>

### Q171. Deleting an Application hangs with the app stuck in `Deleting`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `resources-finalizer.argocd.argoproj.io` finalizer waits for all child resources to be deleted, and one cannot be: the destination cluster is gone, a child has its own stuck finalizer (a PVC, a namespace with a stuck CR), or RBAC denies delete.

**Diagnose →**
```bash
argocd app resources web --orphaned=false
kubectl -n web get all,pvc -o name
kubectl get ns web -o yaml | yq '.status.conditions'
```

**Fix →** Fix the stuck child. If the cluster no longer exists, remove the finalizer:
```bash
kubectl -n argocd patch app web --type json -p '[{"op":"remove","path":"/metadata/finalizers"}]'
```

**Prevent →** Delete Applications before decommissioning clusters.

</details>

### Q172. Sync fails with `namespaces "payments" not found`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The destination namespace does not exist and the app does not create it.

**Diagnose →** `kubectl get ns payments` on the target cluster.

**Fix →** Add the sync option, plus labels if needed:
```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
  managedNamespaceMetadata:
    labels:
      team: payments
```
Or create the namespace in a platform app first.

**Prevent →** Decide per tenancy model who owns namespaces and make it the default in the app template.

</details>

### Q173. Sync shows `Retrying attempt #5` and the operation finally fails with `admission webhook "validation.gatekeeper.sh" denied the request`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A policy engine (Gatekeeper, Kyverno) rejects the object, for example missing resource limits or a disallowed registry. Retries cannot help.

**Diagnose →** `argocd app get web` shows the full denial message; `kubectl get constraints` or `kubectl get clusterpolicy` on the cluster.

**Fix →** Fix the manifest in Git to meet the policy. Do not add policy exceptions to unblock one deploy without review.

**Prevent →** Run the same policies in CI (`gator test`, `kyverno apply`) against rendered manifests before merge.

</details>

### Q174. A sync succeeds but some resources are applied before their dependencies despite sync waves. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Waves only order resources inside one Application. The dependency lives in another app; or the wave annotation is on a Helm value that never renders; or the wave value is not a quoted string.

**Diagnose →** `argocd app manifests web | grep -B2 -A2 sync-wave`; check which app owns each resource.

**Fix →** Move dependent resources into the same app, or order Applications themselves with waves in the parent app-of-apps. Note that Application health must be enabled for waves between apps to wait on child health: add the `argoproj.io/Application` health customization in `argocd-cm`.

**Prevent →** Keep tightly coupled resources in one app and quote wave values (`"-1"`).

</details>

---

## ApplicationSets, Rollouts & Image Updater

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q175. The ApplicationSet status shows `ApplicationSet argocd/team-apps contains applications with duplicate name: web`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two generator results produce the same `metadata.name`, for example a git directory generator finding `apps/web` in two envs while the name template uses only `{{.path.basename}}`.

**Diagnose →** `kubectl -n argocd get applicationset team-apps -o yaml | yq '.status.conditions'`.

**Fix →** Make the name unique per element: `name: '{{.path.basename}}-{{index .path.segments 1}}'` or include the cluster name.

**Prevent →** Name template convention `<app>-<env>-<cluster>` for every ApplicationSet.

</details>

### Q176. `ApplicationGenerationFromParamsError: failed to execute go template ...: map has no entry for key "cluster"`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With `goTemplate: true` and `goTemplateOptions: ["missingkey=error"]`, the template references a key the generator does not produce. Cluster generator keys are `name`, `server`, `metadata.labels.<key>`; the git directory generator produces `path.*`.

**Diagnose →** Print the generator output by temporarily templating a ConfigMap, or check the generator's documented parameters.

**Fix →** Use the right key (`{{.name}}` not `{{.cluster}}`); with a matrix generator, check that the child generator providing the key is actually there.

**Prevent →** Keep `missingkey=error`; it catches typos instead of creating apps named `<no value>`.

</details>

### Q177. A git outage made the git generator return nothing, and the ApplicationSet deleted all its Applications and their workloads. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The default policy is `sync`, which deletes Applications that disappear from generator output; the Applications have the resources finalizer, so workloads were pruned too.

**Diagnose →** ApplicationSet controller logs around the outage: `kubectl -n argocd logs deploy/argocd-applicationset-controller`.

**Fix →** Restore Git access and let the set re-create apps. Then harden:
```yaml
spec:
  syncPolicy:
    applicationsSync: create-update     # never delete
    preserveResourcesOnDeletion: true   # keep workloads if an app is deleted
```

**Prevent →** Use `create-update` (or `--policy` on the controller) for production sets and require manual cleanup.

</details>

### Q178. Manual edits to an Application owned by an ApplicationSet (for example a changed `targetRevision`) keep getting reverted. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ApplicationSet controller owns the full spec and regenerates it on every reconcile.

**Diagnose →** `kubectl -n argocd get app web -o yaml | yq '.metadata.ownerReferences'`.

**Fix →** Change the value in the ApplicationSet template or generator data. If a field must be editable per app, list it in `ignoreApplicationDifferences`:
```yaml
spec:
  ignoreApplicationDifferences:
    - jsonPointers:
        - /spec/source/targetRevision
```

**Prevent →** Tell teams that generated apps are read-only and changes go through the set.

</details>

### Q179. A newly registered cluster gets no Applications from a cluster-generator ApplicationSet. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The generator's `selector.matchLabels` requires labels the cluster secret does not have. The local `in-cluster` has no secret, so label selectors never match it unless you create one.

**Diagnose →**
```bash
kubectl -n argocd get secret -l argocd.argoproj.io/secret-type=cluster --show-labels
```

**Fix →** Label the cluster secret (`kubectl label secret <cluster-secret> env=prod -n argocd`) or set labels at registration with `argocd cluster add ctx --label env=prod`.

**Prevent →** Make labels part of the cluster registration code (Terraform), not a manual step.

</details>

### Q180. The SCM provider generator stops discovering repos, with `GET https://api.github.com/orgs/org/repos: 403 API rate limit exceeded`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No token or a shared token, and a short `requeueAfterSeconds`, so the controller lists the org too often.

**Diagnose →** ApplicationSet controller logs; check the token's rate limit.

**Fix →** Use a GitHub App (`appSecretName`) or a dedicated token via `tokenRef`, raise `requeueAfterSeconds` to 1800 or more, and add filters (`repositoryMatch`, `labelMatch`) to reduce API calls.

**Prevent →** Webhooks to the ApplicationSet controller for event-driven refresh instead of tight polling.

</details>

### Q181. Pull request generator preview environments are not created for new PRs. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The PR does not carry the required label filter, the token lacks pull request read permission, the `requeueAfterSeconds` is long and no webhook is set, or the PR is from a fork (token cannot see it).

**Diagnose →** Controller logs for the set; `kubectl -n argocd get applicationset previews -o yaml | yq '.status'`.

**Fix →** Add the label, fix token scopes, and configure the ApplicationSet webhook (`/api/webhook` on the applicationset-controller service) for `pull_request` events.

**Prevent →** Document the label and add a CI check that comments the preview URL so missing previews are noticed.

</details>

### Q182. Progressive sync (RollingSync) settings are ignored; all apps update at once. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Progressive syncs are off by default in the ApplicationSet controller, or the step `matchExpressions` match no app labels, or the generated apps have their own `automated` sync, which RollingSync needs to control.

**Diagnose →** `kubectl -n argocd get cm argocd-cmd-params-cm -o yaml | grep progressive`; `kubectl -n argocd get applicationset fleet -o yaml | yq '.status.applicationStatus'`.

**Fix →** Set `applicationsetcontroller.enable.progressive.syncs: "true"`, restart the controller, and ensure template labels match the step selectors (`env: canary`, `env: prod`).

**Prevent →** Test the step order with a no-op change before relying on it for a real rollout.

</details>

### Q183. Argo CD shows the app `Suspended`, and the deploy pipeline waiting on `argocd app wait --health` times out. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An Argo Rollout is paused at a canary step (`pause: {}` with no duration) waiting for promotion. Argo CD maps a paused Rollout to Suspended health.

**Diagnose →** `kubectl argo rollouts get rollout web -n web`.

**Fix →** Promote if metrics look good: `kubectl argo rollouts promote web -n web`, or abort. For pipelines, use `argocd app wait web --health --suspended` or switch to timed pauses with analysis.

**Prevent →** Replace indefinite pauses with `pause: {duration: 10m}` plus an AnalysisTemplate, so promotion is automated.

</details>

### Q184. A Rollout aborts with `RolloutAborted: Rollout aborted update to revision 7: Metric "success-rate" assessed Failed due to failed (3) > failureLimit (2)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Either the new version really is bad (good, the system worked), or the query is wrong: it returns empty or NaN during low traffic, uses the wrong label for canary pods, or the threshold is unrealistic.

**Diagnose →**
```bash
kubectl -n web get analysisrun
kubectl -n web describe analysisrun <name>     # shows each measurement value
```
Run the PromQL manually in Prometheus for the same window.

**Fix →** If the release is bad, fix and push a new image. If the query is wrong, handle empty results (`... or vector(1)` with care) and scope by `rollouts-pod-template-hash`.

**Prevent →** Test analysis templates against historical data and require a minimum request count before judging.

</details>

### Q185. An AnalysisRun shows `Error` with `Post "http://prometheus.monitoring:9090/api/v1/query": dial tcp: lookup prometheus.monitoring on 10.96.0.10:53: no such host`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong Prometheus address (short name instead of `prometheus-server.monitoring.svc.cluster.local`, wrong port), or a NetworkPolicy blocks the rollout namespace from reaching monitoring.

**Diagnose →** `kubectl -n monitoring get svc`; test with `kubectl run tmp --rm -it --image=curlimages/curl -- curl <address>/-/ready`.

**Fix →** Correct `provider.prometheus.address` in the AnalysisTemplate, or allow the traffic. Note that measurement errors count against `consecutiveErrorLimit` (default 4) and then abort.

**Prevent →** Use a ClusterAnalysisTemplate maintained by the platform team with the verified address.

</details>

### Q186. Argo CD Image Updater logs `could not get tags from registry: denied: requested access to the resource is denied` for a private ECR or GHCR image. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No registry credentials configured for that image, or the credentials (ECR tokens expire every 12 hours) went stale.

**Diagnose →** `kubectl -n argocd logs deploy/argocd-image-updater | grep denied`.

**Fix →** Configure `pullsecret:namespace/secret` for the image or registry in `registries.conf`, and for ECR use an `ext:` credential script or an ECR token refresher that updates the secret before expiry.

**Prevent →** Alert on Image Updater errors; a silent updater means images silently stop rolling.

</details>

### Q187. Image Updater bumps the tag, but within minutes the app reverts to the old image. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The write-back method is `argocd` (it writes parameter overrides on the Application object), and the Application is itself managed by an app-of-apps or ApplicationSet that overwrites the spec from Git on the next sync.

**Diagnose →** `argocd app get web -o yaml | yq '.spec.source.helm.parameters'` before and after the revert; check `ownerReferences` or the parent app.

**Fix →** Use Git write-back so the change lives in Git: `write-back-method: git` with `write-back-target: kustomization` or a Helm values file (`.argocd-source-<app>.yaml`) and a credential that can push.

**Prevent →** Standardise on Git write-back for any app managed by GitOps of GitOps; the git history doubles as the deploy audit trail.

</details>

### Q188. Image Updater with Git write-back fails with `failed to push: authentication required` or `protected branch hook declined`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** It reuses the repo credentials Argo CD uses for reading, which are read-only deploy keys; or the target branch is protected against direct pushes.

**Diagnose →** Image Updater logs show the push error and the branch.

**Fix →** Provide a write credential (`git:secret:argocd/git-write-creds`) and either allow the bot to bypass protection on that branch or write to a separate branch (`write-back-branch: image-updates`) that a CI job merges via PR.

**Prevent →** Separate read and write credentials; keep a branch protection rule that allows only the bot's identity.

</details>

---

## Secrets, Controller Performance & Recovery

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q189. A Sealed Secret is not decrypted: the SealedSecret shows `no key could decrypt secret (password)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** It was sealed with another cluster's public key, sealed for a different namespace/name (strict scope), or the controller's sealing keys were lost when the cluster was rebuilt.

**Diagnose →**
```bash
kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key
kubectl -n web describe sealedsecret db-creds
```

**Fix →** Re-seal against the right cert (`kubeseal --fetch-cert --controller-namespace kube-system > cert.pem`) and the correct namespace; on a rebuilt cluster, restore the backed-up key secret and restart the controller.

**Prevent →** Back up sealing keys to a vault; this is part of cluster DR, not optional.

</details>

### Q190. A KSOPS or SOPS-based app fails with `Error: failed to get the data key required to decrypt the SOPS file`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The repo-server cannot access the key: missing age private key secret mount, IRSA role without `kms:Decrypt` on the CMK, or the file was encrypted with a recipient list that does not include the cluster's key.

**Diagnose →**
```bash
kubectl -n argocd exec deploy/argocd-repo-server -- env | grep SOPS
grep -A3 'kms\|age' secrets.enc.yaml
```

**Fix →** Mount the age key (`SOPS_AGE_KEY_FILE`) or grant KMS decrypt to the repo-server's role; re-encrypt with `sops updatekeys` after adding the cluster recipient to `.sops.yaml`.

**Prevent →** Keep recipients in `.sops.yaml` and fail CI when an encrypted file lacks the cluster recipient.

</details>

### Q191. argocd-vault-plugin apps fail with `Replace: could not replace all placeholders in Template: ... secret not found`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The path in the `<path:...#key>` placeholder or the `avp.kubernetes.io/path` annotation is wrong, the KV version mismatches (KV v2 needs `data/` in the path), or the plugin's Vault role cannot read it.

**Diagnose →** `vault kv get <path>` with the same role; CMP sidecar logs in repo-server.

**Fix →** Correct the path, KV version, or Vault policy. Note that AVP renders secrets into manifests, so they appear in Argo CD's cache and diff; consider moving to External Secrets Operator.

**Prevent →** Prefer controllers that fetch secrets in-cluster (ESO, Vault Secrets Operator) so secret values never pass through the repo-server.

</details>

### Q192. An ExternalSecret is `SecretSyncedError` with `could not get secret data from provider`, and the app is Degraded. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The remote key does not exist, the SecretStore's IAM role or service account cannot read it, or the key name/property differs.

**Diagnose →**
```bash
kubectl -n web describe externalsecret db-creds
kubectl -n web get secretstore,clustersecretstore -o wide
```

**Fix →** Create the key in the provider or fix the `remoteRef`; grant the role (for AWS, `secretsmanager:GetSecretValue` on that ARN).

**Prevent →** Create secrets in the provider before merging the ExternalSecret, and alert on `externalsecret_status_condition`.

</details>

### Q193. The application controller is OOMKilled repeatedly after onboarding a large cluster. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The controller caches every watched resource of every managed cluster; big clusters with many Events, Pods, ReplicaSets or CRs (Velero backups, Kyverno reports) blow up memory.

**Diagnose →**
```bash
argocd admin cluster stats
kubectl -n argocd top pod -l app.kubernetes.io/name=argocd-application-controller
```

**Fix →** Exclude noisy kinds:
```yaml
# argocd-cm
data:
  resource.exclusions: |
    - apiGroups: ["kyverno.io", "wgpolicyk8s.io"]
      kinds: ["*"]
      clusters: ["*"]
```
Raise memory limits, and shard with more controller replicas (`ARGOCD_CONTROLLER_REPLICAS`).

**Prevent →** Review exclusions when onboarding clusters; watch `argocd_cluster_api_resource_objects`.

</details>

### Q194. Reconciliation is slow for all apps; `argocd_app_reconcile` p95 is minutes and the workqueue depth keeps growing. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Too few processors for the number of apps, a slow repo-server (manifest generation backlog), or one heavy cluster starving others on a single shard.

**Diagnose →** Check `workqueue_depth`, `argocd_app_reconcile_bucket` and `argocd_git_request_duration_seconds` in Prometheus; `argocd admin cluster shards`.

**Fix →** Raise `controller.status.processors` (for example 50) and `controller.operation.processors` (25) in `argocd-cmd-params-cm`; scale repo-server replicas; add controller shards and use the `round-robin` or `consistent-hashing` sharding algorithm.

**Prevent →** Capacity plan: track apps and resources per shard and scale before the queue grows.

</details>

### Q195. The UI and CLI are slow and logs show `Failed to get cached app state: dial tcp 10.96.40.2:6379: connect: connection refused`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Redis is down or restarting (OOM, node eviction), or in HA mode the HAProxy in front of Redis cannot reach a master.

**Diagnose →**
```bash
kubectl -n argocd get pods -l app.kubernetes.io/name=argocd-redis
kubectl -n argocd logs deploy/argocd-redis --previous
```

**Fix →** Restore Redis; Argo CD recomputes everything because Redis is only a cache, so expect a temporary CPU spike. Raise its memory limit if OOM.

**Prevent →** Run the HA install (`redis-ha`) in production and alert on Redis availability.

</details>

### Q196. Repo-server pods fail renders with `write /tmp/...: no space left on device`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Large monorepos and Helm chart caches fill the repo-server's ephemeral storage, especially with many revisions checked out in parallel.

**Diagnose →** `kubectl -n argocd exec deploy/argocd-repo-server -- df -h /tmp`; check pod events for ephemeral storage eviction.

**Fix →** Mount a larger `emptyDir` (with `sizeLimit`) or a dedicated volume on `/tmp`; scale out repo-server so each pod handles fewer repos; restart the pod to clear stale checkouts.

**Prevent →** Split huge monorepos, and use `argocd.argoproj.io/manifest-generate-paths` so changes in unrelated paths do not trigger renders.

</details>

### Q197. After upgrading Argo CD, applies fail with `strict decoding error: unknown field "spec.sourceHydrator"` or the CRD update fails with `metadata.annotations: Too long`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The new controller expects newer CRD schemas, but the CRDs were not upgraded (Helm does not upgrade CRDs by default), or they were applied client-side and hit the annotation size limit.

**Diagnose →** `kubectl get crd applications.argoproj.io -o yaml | yq '.metadata.labels'` and compare the schema with the release.

**Fix →** Apply CRDs server-side for the target version:
```bash
kubectl apply -k "https://github.com/argoproj/argo-cd/manifests/crds?ref=v3.1.0" \
  --server-side --force-conflicts
```
With the Helm chart, keep `crds.install: true` and upgrade the chart, not only the image tags.

**Prevent →** Upgrade one minor version at a time, read the upgrade notes, and rehearse on a staging Argo CD.

</details>

### Q198. The admin password is lost and `argocd-initial-admin-secret` was deleted long ago. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The initial secret is meant to be deleted after first login; the bcrypt hash lives in `argocd-secret`.

**Diagnose →** `kubectl -n argocd get secret argocd-secret -o yaml | grep admin.password`.

**Fix →**
```bash
HASH=$(argocd account bcrypt --password 'N3w-Str0ng-Pass')
kubectl -n argocd patch secret argocd-secret \
  -p "{\"stringData\": {\"admin.password\": \"$HASH\", \"admin.passwordMtime\": \"$(date +%FT%T%Z)\"}}"
```
Then log in and rotate.

**Prevent →** Disable the local admin (`admin.enabled: "false"`) once SSO works, and keep a sealed break-glass procedure.

</details>

### Q199. Someone deleted an app-of-apps parent with cascade, and dozens of child apps and workloads disappeared. How do you recover? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The parent and children all had the `resources-finalizer.argocd.argoproj.io` finalizer, so deletion cascaded down to every workload.

**Diagnose →** `kubectl -n argocd get events`, audit logs for the delete call; confirm which namespaces lost workloads.

**Fix →** Re-apply the root Application from Git (`kubectl apply -f bootstrap/root-app.yaml`); Argo CD recreates children and workloads from Git. Restore stateful data (PVs, databases) from backups, since Git does not hold data.

**Prevent →** Remove the finalizer from parent apps (only children should cascade), restrict `delete` on Applications in RBAC, enable deletion protection with `argocd.argoproj.io/sync-options: Delete=false` on critical resources, and back up Argo CD state with `argocd admin export`.

</details>

### Q200. Argo CD must be rebuilt on a new management cluster after the old one was lost. Apps show `Unknown` and clusters `Failed` after restoring from backup. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The backup restored Applications and settings, but cluster secrets reference tokens or IAM roles tied to the old cluster's identity (IRSA trust policy for the old OIDC provider), and repo credentials or SSO secrets were not in the backup.

**Diagnose →**
```bash
argocd cluster list           # connection status per cluster
argocd repo list
kubectl -n argocd logs deploy/argocd-application-controller | grep -i "failed to sync cluster"
```

**Fix →** Restore with `argocd admin import - < backup.yaml` (from `argocd admin export`), then update the IAM trust policies (or EKS access entries) for the new controller role, re-sync secrets from the external secret store, and refresh: `argocd app list -o name | xargs -n1 argocd app get --hard-refresh`.

**Prevent →** Keep Argo CD itself declarative in Git (install values, projects, clusters via Terraform), store credentials in an external secret manager, and run a timed DR drill every quarter.

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
