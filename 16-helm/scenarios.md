# Helm — Scenario-Based Questions

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

> Real-world Helm situations and how to reason through them in an interview · 100 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Helm](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Chart Design & Authoring](#chart-design--authoring) — Q1–Q18
- [Values & Environments](#values--environments) — Q19–Q34
- [Dependencies, Subcharts & Library Charts](#dependencies-subcharts--library-charts) — Q35–Q46
- [Releases, Upgrades & Rollbacks](#releases-upgrades--rollbacks) — Q47–Q64
- [Hooks, Migrations & Testing](#hooks-migrations--testing) — Q65–Q78
- [OCI, CI/CD & GitOps](#oci-cicd--gitops) — Q79–Q92
- [Secrets & Security](#secrets--security) — Q93–Q100

---

## Chart Design & Authoring

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. Your team has 15 near-identical YAML folders, one per microservice per environment. How do you introduce Helm? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Copy-paste manifests drift; a fix applied to one service never reaches the others. The variation is small: image, port, replicas, env vars, resources.

**What I'd do →**
1. Diff the folders to find what actually varies; that list becomes `values.yaml`.
2. `helm create` one chart, strip it to Deployment, Service, ServiceAccount, HPA, PDB, Ingress, with helpers for names and labels.
3. Convert one low-risk service, compare `helm template` output with the existing YAML (`diff <(helm template ...) <(cat old/*.yaml)`), and deploy to dev.
4. Roll out service by service; adopt existing objects with Helm ownership metadata (or `--take-ownership` on Helm 3.17+).
5. Once three services fit, extract common templates into a library chart.

**Why →** Starting from real variation avoids an over-engineered "chart of everything".

**Interview tip →** Mention that you validated equivalence by diffing rendered output before cutting over.

</details>

### Q2. Should you build one generic chart for all microservices or one chart per service? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A generic chart gives consistency; per-service charts give flexibility. Both have failure modes: a generic chart grows hundreds of flags, per-service charts diverge.

**What I'd do →**
1. Use a **library chart** for shared templates (Deployment, Service, HPA, PDB, labels, security defaults).
2. Give each service a thin application chart that depends on the library and adds only what is unique (a CronJob, a sidecar).
3. Version the library; services bump it deliberately via `Chart.lock`.
4. Keep a "golden" generic app chart only for truly standard stateless services.

**Why →** Shared code with a versioned contract, without forcing every edge case into one template.

**Interview tip →** Say "library chart plus thin service charts" and explain the versioning benefit.

</details>

### Q3. A developer hardcoded `namespace: production` in every template. What is wrong and how do you fix it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The chart can no longer be installed into any other namespace; `helm install -n staging` would still create objects in `production`, while the release record lands in `staging`.

**What I'd do →**
1. Remove `metadata.namespace` from templates, or set it to `{{ .Release.Namespace }}` for clarity.
2. Replace hardcoded namespaces in DNS names with `{{ .Release.Namespace }}` (`api.{{ .Release.Namespace }}.svc.cluster.local`).
3. Add a CI check: `helm template x ./chart -n ci-test | grep -E '^\s+namespace:' | grep -v ci-test` must be empty.

**Why →** The namespace is a deploy-time decision; charts must stay portable.

**Interview tip →** Note that release records and objects in different namespaces confuse `helm uninstall`.

</details>

### Q4. Changing a ConfigMap value in the chart does not restart the pods, so the app keeps old config. What do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pods read config at startup (env vars or a file loaded once). A ConfigMap update does not change the pod template, so no rollout happens.

**What I'd do →**
1. Add a checksum annotation to the pod template:
```yaml
annotations:
  checksum/config: {{ include (print $.Template.BasePath "/api-configmap.yaml") . | sha256sum }}
```
2. Do the same for Secrets the chart renders.
3. For externally managed config, use a tool like Stakater Reloader, or include a version value in the name of the ConfigMap.

**Why →** Any config change now alters the pod template hash and triggers a normal rolling update.

**Interview tip →** Avoid `.Release.Revision` annotations; they restart pods on every deploy even when nothing changed.

</details>

### Q5. You need the same chart to support both Ingress and Gateway API HTTPRoute. How do you design it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Some clusters run ingress-nginx, newer ones use Gateway API. The chart must not force either.

**What I'd do →**
1. Add two independent blocks: `ingress.enabled` and `httpRoute.enabled`, each with its own template.
2. Guard the HTTPRoute with `.Capabilities.APIVersions.Has "gateway.networking.k8s.io/v1/HTTPRoute"` and `fail` with a clear message if enabled but missing.
3. Share host and path values through a helper so both render from the same source.
4. Add a schema rule that rejects both enabled at once if that is not supported.

**Why →** Feature flags plus capability checks keep one chart portable across cluster generations.

**Interview tip →** Mention testing both paths with `helm template --api-versions`.

</details>

### Q6. Your chart's Deployment template has grown to 300 lines of nested `if`s. How do you clean it up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Every consumer request became a flag; nobody can review the template.

**What I'd do →**
1. Move repeated fragments (probes, security context, env, volume mounts) into named helpers in `_helpers.tpl` or a library.
2. Replace field-by-field options with `toYaml` passthrough for standard Kubernetes blocks (`affinity`, `tolerations`, `securityContext`).
3. Add escape hatches (`extraEnv`, `extraVolumes`, `extraContainers`) instead of new flags.
4. Add helm-unittest suites that pin current output, then refactor safely.
5. Delete unused flags in a major version bump.

**Why →** Templates should express opinions; passthroughs handle the long tail.

**Interview tip →** Say you put unit tests in place before refactoring templates.

</details>

### Q7. How do you make a chart work for both a stateless API and a worker that has no Service? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same code base, two run modes; the worker must not get a Service, Ingress or HTTP probes.

**What I'd do →**
1. Model components in values: `api.enabled`, `worker.enabled`, each with image args, replicas and resources.
2. Render the Service and Ingress only for `api`, wrapping each template in `{{- if .Values.api.enabled }}`.
3. Give the worker exec or no probes, and its own `app.kubernetes.io/component: worker` selector label.
4. Share image settings through a common `image` block with per-component overrides.

**Why →** Components keep selectors distinct, so the API Service never routes to worker pods.

**Interview tip →** Highlight the `component` label in selectors as the thing that keeps them separate.

</details>

### Q8. A security review requires every pod to run non-root with a read-only filesystem. How do you enforce it in your charts? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Defaults must be secure, and overriding them should be visible.

**What I'd do →**
1. Default values:
```yaml
podSecurityContext: { runAsNonRoot: true, runAsUser: 10001, fsGroup: 10001, seccompProfile: { type: RuntimeDefault } }
securityContext: { allowPrivilegeEscalation: false, readOnlyRootFilesystem: true, capabilities: { drop: [ALL] } }
```
2. Add an `emptyDir` for `/tmp` so apps can still write scratch files.
3. Put these defaults in the library chart so all services inherit them.
4. Enforce at the cluster with Pod Security Admission `restricted` or Kyverno, so charts that override them fail loudly.

**Why →** Chart defaults make the right thing easy; admission policy makes the wrong thing impossible.

**Interview tip →** Defense in depth: chart defaults plus admission control, not one or the other.

</details>

### Q9. Your app needs a different config file per environment, with values injected. How do you package it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Config files are long; embedding them as YAML strings in values is unreadable.

**What I'd do →**
1. Keep the template config in `files/app.conf.tpl` inside the chart with placeholders like `{{ .Values.app.logLevel }}`.
2. Render it into a ConfigMap with `tpl`:
```yaml
data:
  app.conf: |-
{{ tpl (.Files.Get "files/app.conf.tpl") . | indent 4 }}
```
3. Expose only the variable parts as values per environment.
4. Add a checksum annotation so changes roll the pods.

**Why →** The file stays readable and diffable; environments differ only in values.

**Interview tip →** Mention `.helmignore` and that `.Files` cannot read files under `templates/`.

</details>

### Q10. Two releases of your chart in the same cluster collide on a ClusterRole name. How do you fix the chart? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cluster-scoped resources are not namespaced, so a static name like `shop-reader` is shared by every release and the second install fails with "already exists".

**What I'd do →**
1. Name cluster-scoped objects with release and namespace: `{{ printf "%s-%s" .Release.Namespace (include "shop.fullname" .) | trunc 63 | trimSuffix "-" }}`.
2. Ask whether a namespaced Role is enough; prefer it.
3. Make cluster-scoped resources optional (`rbac.clusterWide: false`).

**Why →** Unique names make the chart installable multiple times, which is a basic chart requirement.

**Interview tip →** Call out CRDs as the one cluster-scoped thing that should be shared, and managed separately.

</details>

### Q11. A teammate wants to put the image tag in `Chart.yaml` `appVersion` only. Another wants it only in values. What do you recommend? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both work; the question is what is the source of truth and how rollbacks behave.

**What I'd do →**
1. Template: `{{ .Values.image.tag | default .Chart.AppVersion }}`, so both are possible.
2. For internal apps released together with their chart, set `appVersion` at package time (`helm package --app-version $VERSION`) so one chart version pins one image.
3. For GitOps environments where images change often, set the tag in the environment values file and let CI or Image Updater commit it.
4. Never use `latest`.

**Why →** One pinned value per deploy keeps `helm history` meaningful and rollbacks exact.

**Interview tip →** Say the important thing is immutability and traceability, not where the tag lives.

</details>

### Q12. Your chart must run on OpenShift and vanilla Kubernetes. What needs care? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** OpenShift's restricted SCC assigns a random UID per namespace and rejects fixed `runAsUser`; it also uses Routes.

**What I'd do →**
1. Make `runAsUser`/`fsGroup` optional (render only when set) and ensure images work with arbitrary UIDs (group 0 writable dirs).
2. Detect OpenShift with `.Capabilities.APIVersions.Has "route.openshift.io/v1"` and optionally render a Route.
3. Avoid privileged ports (<1024) and hostPath.
4. Test with `helm template --api-versions route.openshift.io/v1` and on a CRC or OKD cluster in CI.

**Why →** Platform differences belong in capability checks and values, not forks.

**Interview tip →** Mention arbitrary UID support as the most common OpenShift chart failure.

</details>

### Q13. You need to deploy the same app to 40 tenant namespaces with small differences. How do you organize it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** 40 releases of one chart; differences per tenant (domain, quotas, feature flags).

**What I'd do →**
1. One chart, one release per tenant namespace (`tenant-a`, `tenant-b`), never a single release looping over tenants (a single failure would block all).
2. Generate releases with an Argo CD ApplicationSet (list or Git directory generator) or Helmfile with a tenants file.
3. Per-tenant values files are tiny: domain, size tier, flags.
4. Roll out chart versions progressively: canary tenants first, then the rest, using sync waves or Helmfile labels.

**Why →** Independent releases give per-tenant blast radius, history and rollback.

**Interview tip →** Explain why "one release per tenant" beats one giant release.

</details>

### Q14. Product wants a "maintenance mode" toggle that swaps the frontend for a static page. How would you implement it in the chart? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** It must be fast to toggle and reversible.

**What I'd do →**
1. Add `maintenance.enabled` and render a small nginx Deployment with a ConfigMap page when true.
2. Switch the Service selector or Ingress backend to the maintenance component in the same render.
3. Toggle through the normal pipeline: the full values files plus `--set maintenance.enabled=true` (not `--reuse-values`), or a one-line commit in GitOps.
4. Keep the main Deployment running, so switching back is instant.

**Why →** Selector switch is atomic at the Service level; no pods need to restart.

**Interview tip →** Show you thought about the toggle being a normal, reviewed deploy, not a manual edit.

</details>

### Q15. You inherit a chart with no labels helper; every template has different labels. How do you fix it without downtime? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Adding labels is safe, but changing selectors on Deployments is not (immutable).

**What I'd do →**
1. Add `shop.labels` and `shop.selectorLabels` helpers; apply `labels` to all metadata.
2. Keep the existing selectors exactly as they are for now (copy them into `selectorLabels`).
3. If selectors must change, do it in a major chart version with a documented migration: delete the Deployment with `kubectl delete deploy shop-api --cascade=orphan`, upgrade (new Deployment adopts nothing because labels differ, so it creates new pods), then delete orphaned old ReplicaSets after the new pods are Ready.
4. Use `helm diff` to confirm only metadata labels change in step 1.

**Why →** Metadata labels are mutable; selectors are not.

**Interview tip →** Saying "selectors are immutable, so I orphan-delete" shows real experience.

</details>

### Q16. Leadership asks why you are adopting Helm instead of Kustomize. How do you answer? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** It is a trade-off question; neither tool is universally better.

**What I'd do →**
1. Helm for packaging and distribution: versioned charts, dependencies, release history, rollback, hooks; most third-party software ships as charts.
2. Kustomize for small overlays on our own plain manifests; no templating language to learn.
3. Propose Helm for apps and platform components, with Kustomize allowed as a post-renderer or Argo CD layer for last-mile patches.

**Why →** Using the right tool per layer avoids forcing templates where patches suffice.

**Interview tip →** Avoid tool wars; frame it as packaging vs patching.

</details>

### Q17. The chart must support an optional sidecar (log shipper) that some environments need. How do you add it cleanly? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Optional container, with its own image, resources and config.

**What I'd do →**
1. Add `logShipper.enabled`, `logShipper.image`, `logShipper.resources` in values.
2. In the pod spec, render the container inside `{{- if .Values.logShipper.enabled }}`, plus its ConfigMap and shared `emptyDir` volume.
3. Alternatively offer generic `extraContainers` (rendered with `tpl (toYaml .Values.extraContainers) .`) for any sidecar.
4. Consider native sidecars (init container with `restartPolicy: Always`, Kubernetes 1.29+) so it starts before and stops after the app.

**Why →** A dedicated block for common sidecars, plus a generic escape hatch for rare ones.

**Interview tip →** Mentioning native sidecars shows you track current Kubernetes features.

</details>

### Q18. How would you test chart templates in CI without a cluster? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want fast feedback on template changes on every PR.

**What I'd do →**
1. `helm lint --strict` against each environment's values files.
2. `helm template` for each environment piped into `kubeconform -strict` for schema validity.
3. helm-unittest suites asserting specific output (selector labels, replicas absent when HPA enabled, image tag).
4. Policy checks with Conftest/Kyverno CLI (non-root, limits set).
5. Golden-file diffs: commit rendered output and fail if it changes unexpectedly.

**Why →** These layers catch syntax, schema, logic and policy errors in seconds.

**Interview tip →** Name helm-unittest and kubeconform explicitly.

</details>

## Values & Environments

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q19. Dev, staging and prod values files have drifted and nobody knows which settings differ intentionally. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each file copied the full `values.yaml` and edited it over time, so real differences are buried.

**What I'd do →**
1. Render each environment and diff: `diff <(helm template s ./shop -f values-staging.yaml) <(helm template s ./shop -f values-prod.yaml)`.
2. Rewrite each env file to contain only overrides of chart defaults; move shared settings into `values-common.yaml`.
3. Annotate intentional differences with comments ("prod: 3 replicas for HA").
4. Add a CI job that prints the staging-vs-prod rendered diff on every PR.

**Why →** Small override files make differences reviewable and staging a real rehearsal for prod.

**Interview tip →** Say env files should contain overrides only, never a copy of defaults.

</details>

### Q20. A deploy pipeline uses `--reuse-values` and a new chart version just broke production. Why, and what do you change? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The new chart added `api.probes.startup` in `values.yaml`, and the template reads it. With `--reuse-values`, Helm used the old release's values as the base, so the new default never arrived and the template rendered an empty probe.

**What I'd do →**
1. Roll back: `helm rollback shop -n shop`.
2. Change the pipeline to pass every values file explicitly and drop `--reuse-values` (or use `--reset-then-reuse-values` if you must keep ad-hoc overrides).
3. Make templates defensive (`with`, `default`) for newly added keys.
4. Keep env values in Git so the full set is always available.

**Why →** A deploy should be fully reproducible from Git, not dependent on what the previous run stored.

**Interview tip →** Explaining the mechanics of `--reuse-values` is a strong signal.

</details>

### Q21. You need a per-environment image tag set by CI, while everything else lives in Git. How do you wire it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The tag changes on every build; the rest is reviewed config.

**What I'd do →**
1. Imperative CI: `helm upgrade --install shop ./shop -f values-prod.yaml --set-string api.image.tag=$GIT_SHA`.
2. GitOps: CI opens a PR or commits `api.image.tag: <sha>` to `envs/prod/values.yaml`; Argo CD syncs.
3. Use `--set-string` so tags like `1.10` or `0123abc` keep their exact text.
4. Record the SHA in `--description` for traceability.

**Why →** Either way the deployed tag is traceable; GitOps also makes rollback a revert.

**Interview tip →** Mention `--set-string` and why.

</details>

### Q22. A value that should be `false` in prod keeps rendering as enabled. Where do you look? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic causes: `default true .Values.x`, `--set-string x=false` (the string "false" is truthy), or a later values file setting it back.

**What I'd do →**
1. `helm get values shop -n shop --all | yq '.feature'` to see the final merged value and its type.
2. Grep the templates for `default true` on that key.
3. Check the order of `-f` files and any `--set` in the pipeline.
4. Fix the template to use `hasKey` or put the default in `values.yaml`; use `--set` (not `--set-string`) for booleans.

**Why →** Helm's emptiness rules treat `false` as unset in `default`.

**Interview tip →** Quote the `default` boolean trap; interviewers love it.

</details>

### Q23. Different clusters need different storage classes and ingress classes. How do you keep the chart portable? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** EKS uses `gp3`, the on-prem cluster uses `longhorn`; ingress classes differ too.

**What I'd do →**
1. Expose `persistence.storageClass` and `ingress.className` in values; render `storageClassName` only when set so the cluster default applies otherwise.
2. Support `global.storageClass` so subcharts share it.
3. Put cluster-specific values in a per-cluster file (`clusters/eks-prod.yaml`) layered under env files.

**Why →** Cluster facts are deploy-time inputs, not chart constants.

**Interview tip →** Note that an empty `storageClassName: ""` disables dynamic provisioning, which is why you omit it rather than render empty.

</details>

### Q24. A developer passes 20 `--set` flags in a shell script. What would you change? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Long `--set` chains are unreadable, hard to review, error prone with commas and dots, and invisible in Git history.

**What I'd do →**
1. Move them into a values file committed next to the deploy definition.
2. Keep `--set-string` only for truly dynamic values (image tag, build number).
3. Use `--set-file` for file content and secrets via `secrets://` files.

**Why →** Values files are diffable and reviewable; flags are not.

**Interview tip →** Reproducibility from Git is the argument that wins.

</details>

### Q25. You must deploy to a region with an air-gapped cluster. How do you handle values and images? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** No internet: chart dependencies, images and OCI pulls must come from inside.

**What I'd do →**
1. Mirror charts (`helm pull` then `helm push` to the internal registry) including subcharts, and vendor them into `charts/` at package time.
2. Mirror every image referenced by the rendered manifests: `helm template ... | yq '..|.image? | select(.)' | sort -u`.
3. Override registries via `global.imageRegistry` or per-image values.
4. Package a single `.tgz` with dependencies included so install needs no repo access.
5. Verify with `helm template` that no image still points to a public registry.

**Why →** Air-gapped deploys fail on the first external reference; rendering is how you find them all.

**Interview tip →** Mention extracting images from rendered output rather than trusting the README.

</details>

### Q26. A feature flag must be enabled for 10% of tenants first. How does your Helm setup support it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Helm configures deployments, not request-level flags; but per-release values can stage rollouts.

**What I'd do →**
1. Keep the flag in values (`features.newCheckout: false`) and set it true in a "canary" tenant group's values file.
2. In Helmfile or ApplicationSet, label tenants by wave and roll the change wave by wave.
3. For per-request percentages, use a feature flag service (OpenFeature, LaunchDarkly, Unleash) and keep only its endpoint/key in values.

**Why →** Release-level flags give blast-radius control; request-level ones need a flag service.

**Interview tip →** Distinguish config rollout from request-level feature flagging.

</details>

### Q27. A values file accidentally set `resources: {}` in prod and pods have no limits. How do you prevent this class of mistake? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** An override erased the safe defaults; nothing complained.

**What I'd do →**
1. Add schema constraints: `resources.limits.memory` required for the API in `values.schema.json`.
2. Enforce at admission (Kyverno `require-limits` policy) so manifests without limits are rejected.
3. Add a CI policy check on rendered output (Conftest) per environment.
4. Use LimitRange defaults in namespaces as a last line.

**Why →** Schema stops bad values early; admission policy stops bad manifests regardless of source.

**Interview tip →** Say "schema plus admission policy" for guardrails.

</details>

### Q28. You want developers to preview exactly what their values change will do to prod. What do you provide? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** People change values blind and find out during deploy.

**What I'd do →**
1. A PR job running `helm diff upgrade shop <chart> -n shop -f values-prod.yaml --context 3 --suppress-secrets` with read-only cluster credentials, posting output as a PR comment.
2. For GitOps, use Argo CD's diff (`argocd app diff`) or a rendered-manifests diff in the PR.
3. Require approval on prod diffs.

**Why →** A visible diff turns a values tweak into a reviewable infrastructure change.

**Interview tip →** Compare `helm diff` to `terraform plan`.

</details>

### Q29. Some values are secret, some are not, and both are needed per environment. How do you lay out the files? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Mixing them forces either encrypting everything (unreadable reviews) or leaking secrets.

**What I'd do →**
1. `values-prod.yaml` for plain config, `secrets.prod.yaml` encrypted with SOPS (only secret keys).
2. Use the same key structure so they deep-merge: `postgres.auth.password` lives in the secrets file only.
3. Deploy with `-f values-prod.yaml -f secrets://secrets.prod.yaml`.
4. `.sops.yaml` creation rules choose the KMS key per environment.

**Why →** Plain config stays reviewable; secrets stay encrypted at rest in Git.

**Interview tip →** Mention that SOPS encrypts values, not keys, so diffs still show which secret changed.

</details>

### Q30. A third-party chart has 2,000 lines of values. How do you manage your overrides? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Copying the full defaults file makes upgrades painful.

**What I'd do →**
1. `helm show values repo/chart --version X > upstream-X.yaml` for reference only (not deployed).
2. Keep our override file minimal: only keys we change, with comments.
3. On chart upgrades, diff `upstream-X.yaml` vs `upstream-Y.yaml` to spot renamed keys, then run `helm diff`.
4. Pin the chart version.

**Why →** A minimal override file keeps upgrades understandable.

**Interview tip →** Say you diff upstream defaults between versions before upgrading.

</details>

### Q31. You need to inject a JSON config blob as a single value. What is the cleanest way? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** JSON inside YAML strings is error prone with quoting.

**What I'd do →**
1. Model it as structured YAML in values and render it with `toJson` (or `toPrettyJson`) into the ConfigMap:
```yaml
data:
  config.json: {{ .Values.app.config | toJson | quote }}
```
2. For one-off structured overrides on the CLI, use `--set-json 'app.config={"retries":3}'`.
3. Validate the structure in `values.schema.json`.

**Why →** Structured values merge across files; opaque strings do not.

**Interview tip →** Mention that structured values allow per-environment partial overrides.

</details>

### Q32. Prod uses RDS while dev uses an in-cluster Postgres subchart. How do you configure the chart? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same app, different database source.

**What I'd do →**
1. Dependency with `condition: postgres.enabled`; dev sets `true`, prod `false`.
2. Add `externalDatabase.host`, `port`, `existingSecret` values for prod.
3. A helper `shop.dbHost` returns the subchart Service name or the external host, and fails if neither is configured.
4. Prod credentials come from an ExternalSecret referencing Secrets Manager.

**Why →** One chart, explicit switch, with a helper hiding the difference from templates.

**Interview tip →** Show the `fail` guard for the "nothing configured" case.

</details>

### Q33. A values key rename is needed (`image.name` to `image.repository`). How do you avoid breaking users? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Consumers have overrides with the old key; a silent rename would deploy the default image.

**What I'd do →**
1. Support both for one minor version: `{{ .Values.image.repository | default .Values.image.name }}`.
2. Emit a deprecation warning in `NOTES.txt` when the old key is set.
3. Remove it in the next major version, and in that version `fail` if the old key is still present, with instructions.
4. Document it in the chart changelog (`artifacthub.io/changes` annotation).

**Why →** A deprecation window plus a hard failure is far safer than a silent behavior change.

**Interview tip →** "Fail loudly on removed keys" is the key phrase.

</details>

### Q34. Your team edits live resources with `kubectl edit` during incidents and then Helm reverts them. How do you handle this? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Live fixes are lost at the next deploy, or worse, kept silently in some fields because of the three-way merge.

**What I'd do →**
1. Allow emergency edits but require a follow-up PR to the values within the incident.
2. Run `helm diff upgrade --three-way-merge` before deploys so drift is visible.
3. With Argo CD, enable self-heal so manual changes are reverted immediately and the only path is Git.
4. Lock down `edit` on prod for humans via RBAC; use break-glass roles.

**Why →** Git as source of truth only works if drift is visible and short-lived.

**Interview tip →** Tie this to the three-way merge and why drift may or may not survive.

</details>

## Dependencies, Subcharts & Library Charts

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q35. You depend on a public Postgres chart, and its default images stopped being freely available. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Vendors do change distribution (in 2025 one major chart vendor moved most free images to a legacy repository), and pods suddenly fail with `ImagePullBackOff` on new nodes.

**What I'd do →**
1. Pin image registry, repository and tag explicitly in our values instead of relying on chart defaults.
2. Mirror the images we use into our own registry (ECR/Harbor) and point the chart there.
3. Evaluate alternatives: an operator (CloudNativePG), a managed database for prod, or our own small subchart for dev.
4. Pin the chart version and track the vendor's announcements.

**Why →** Chart defaults are someone else's decision; production must not depend on them.

**Interview tip →** Saying "we mirror every image and pin digests" covers this whole class of incident.

</details>

### Q36. Your umbrella chart takes 25 minutes to upgrade and a failure in one component rolls back everything. How do you restructure? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** One release containing ten subcharts: shared fate, huge release Secret, and any slow rollout blocks all.

**What I'd do →**
1. Split into one release per component, managed by Helmfile (`needs:` for ordering) or Argo CD Applications with sync waves.
2. Keep the umbrella only for tightly coupled pieces that must deploy together.
3. Migrate without downtime by adopting existing objects into the new releases (ownership annotations, or `--take-ownership` on Helm 3.17+), then remove the old release with `helm uninstall --keep-history` only after the objects are re-owned.
4. Per-component pipelines and alerts.

**Why →** Independent releases give independent rollbacks and faster, smaller upgrades.

**Interview tip →** Mention the release Secret size limit (1 MiB) as a hidden cost of umbrellas.

</details>

### Q37. Ten service charts copy the same `_helpers.tpl`. A security fix to labels is needed everywhere. What do you build? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Copy-paste helpers mean ten PRs per fix.

**What I'd do →**
1. Create a library chart `shacklib` (`type: library`) with `shacklib.labels`, `shacklib.deployment`, `shacklib.service` and security defaults.
2. Publish it to the OCI registry with SemVer.
3. Each service chart adds it as a dependency and replaces its templates with `{{ include "shacklib.deployment" . }}`.
4. Renovate or Dependabot opens PRs to bump the library version across services.

**Why →** One versioned source for shared logic, rolled out like any dependency.

**Interview tip →** Mention automated dependency bumps to close the loop.

</details>

### Q38. Two subcharts both define a template named `common.labels` and resources get the wrong labels. How do you fix it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Template names are global; the last loaded definition wins silently.

**What I'd do →**
1. Find duplicates: `grep -rho 'define "[^"]*"' charts/ templates/ | sort | uniq -d` (after `helm pull --untar` of dependencies).
2. Rename our own templates with a unique prefix.
3. For third-party conflicts, align both on the same shared library version (they usually come from a common library chart) or vendor and patch one.
4. Add a CI check for duplicate define names.

**Why →** Unique prefixes are the only protection Helm offers.

**Interview tip →** Explain why prefixing helper names is not just style.

</details>

### Q39. How would you let a parent chart override a subchart's image registry for all subcharts at once? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Corporate policy requires all images from the internal registry.

**What I'd do →**
1. Set `global.imageRegistry: registry.internal` in the parent values.
2. In our own subcharts, build images with `coalesce .Values.image.registry .Values.global.imageRegistry`.
3. For third-party subcharts, check which global keys they support; otherwise override each image explicitly.
4. Verify with `helm template ... | grep 'image:'`.

**Why →** `global` is the only value Helm propagates to every subchart automatically.

**Interview tip →** Verify through rendered output, not assumptions.

</details>

### Q40. A subchart upgrade (minor version) changed a default and broke your app. How do you prevent surprises? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Range constraints like `^12.0.0` pulled a new minor with different defaults.

**What I'd do →**
1. Commit `Chart.lock` and build with `helm dependency build`, so versions change only via deliberate `helm dependency update` PRs.
2. In those PRs, show `helm diff` / rendered-manifest diffs for each environment.
3. Pin the values that matter to us explicitly rather than relying on subchart defaults.

**Why →** Lockfiles plus rendered diffs turn dependency bumps into reviewed changes.

**Interview tip →** Same reasoning as package lockfiles in application code.

</details>

### Q41. You need two Redis instances (cache and queue) from the same chart in one release. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same chart, two instances, different configs.

**What I'd do →**
1. Declare the dependency twice with `alias: cache` and `alias: queue`.
2. Configure each under its alias key in values.
3. Reference Services by the alias-based names (`<release>-cache-master`, depending on the chart's naming).
4. Give each its own `condition` so either can be disabled.

**Why →** Aliases create separate value scopes and resource names from one chart.

**Interview tip →** Mention that `.Chart.Name` becomes the alias inside the subchart.

</details>

### Q42. A subchart needs a password that the parent generates. How do you share it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Subcharts cannot see parent values, and generating it twice gives two different values.

**What I'd do →**
1. Prefer an "existing secret" pattern: the parent (or an ExternalSecret) creates a Secret with a known name, and the subchart is configured with `auth.existingSecret: shop-db`.
2. Or pass the same value explicitly to both: parent `db.password` and `postgres.auth.password` from one secrets file.
3. Avoid `randAlphaNum` in both places.

**Why →** One Secret object, referenced by name, avoids duplication and drift.

**Interview tip →** "existingSecret" is the pattern most good charts support; mention it.

</details>

### Q43. A team wants their chart to work standalone and as a subchart of a platform umbrella. What do you watch for? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** As a subchart, names, globals and value scopes change.

**What I'd do →**
1. Never read parent-only values; use `global` for shared settings and document which ones.
2. Use the `fullname` helper (release + chart name) so names stay unique under the umbrella.
3. Avoid relying on `.Release.Name` alone for resource names.
4. Test both: `helm template` standalone, and as a dependency in a test umbrella chart in CI.

**Why →** Subcharts share the release but must remain self-contained.

**Interview tip →** Testing both modes in CI is the concrete answer.

</details>

### Q44. You have to disable a subchart's bundled `helm test` that fails in your environment. What are your options? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The parent's `helm test` runs subchart tests too.

**What I'd do →**
1. Check the subchart's values for a test toggle (many have `tests.enabled`).
2. Otherwise filter: `helm test shop --filter '!name=shop-redis-test-connection'`.
3. As a last resort, a post-renderer that drops the test Pod.

**Why →** Filtering keeps your own tests while skipping noisy third-party ones.

**Interview tip →** Know the `--filter` syntax.

</details>

### Q45. How would you handle a dependency that is only available in a private HTTP chart repo in CI? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `helm dependency build` fails without repo configuration and credentials.

**What I'd do →**
1. In CI: `helm repo add internal https://charts.internal --username "$U" --password "$P"` before building.
2. Isolate Helm config per job (`HELM_CONFIG_HOME`, `HELM_CACHE_HOME` in the workspace).
3. Better: move internal charts to OCI and use `helm registry login` with short-lived tokens (OIDC to ECR/GHCR).

**Why →** OCI removes the repo-add step and reuses registry auth you already have for images.

**Interview tip →** Mention short-lived credentials over static passwords.

</details>

### Q46. A library chart change accidentally altered selectors in all services. How do you limit blast radius next time? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Libraries amplify mistakes; every bump touches every service.

**What I'd do →**
1. helm-unittest suites in the library that assert selector labels never change.
2. A consumer test matrix: render each service chart with the new library version and diff against the previous output in CI.
3. Treat selector or name changes as major versions; Renovate auto-merges only patch bumps.
4. Roll out library bumps to a canary service first.

**Why →** Library changes need contract tests, like any shared API.

**Interview tip →** Frame the library as an API with consumers.

</details>

## Releases, Upgrades & Rollbacks

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q47. A production upgrade starts failing health checks five minutes after the deploy. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The new revision is live and degrading; restore service first, investigate second.

**What I'd do →**
1. `helm history shop -n shop` to find the last good revision.
2. `helm rollback shop <rev> -n shop --wait --timeout 5m`.
3. Confirm: `kubectl rollout status deploy/shop-api -n shop`, error rates back to normal.
4. Then compare: `helm diff revision shop <good> <bad>` and look at logs from the failed pods.
5. Fix forward in a new chart or image version.

**Why →** Rollback is the fastest mitigation, as long as the change was backward compatible.

**Interview tip →** Say "mitigate, then diagnose", and mention checking DB compatibility before rolling back.

</details>

### Q48. Why would you insist on `--atomic --wait` in CI, and what is the trade-off? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Without them, Helm reports success as soon as the API accepts objects, and failed upgrades leave a `failed` release.

**What I'd do →**
1. Use `helm upgrade --install --atomic --wait --timeout 10m`.
2. Size `--timeout` for the slowest rollout plus hooks.
3. Make readiness probes meaningful so "ready" means "works".

**Why →** CI gets a true success/failure signal and failures self-heal. The trade-off: an automatic rollback erases the broken state you might want to inspect, and a too-short timeout causes needless rollbacks. Capture `kubectl describe` and logs in the failure step before Helm rolls back, or use `--wait` alone with an explicit rollback step.

**Interview tip →** Mention capturing diagnostics because `--atomic` destroys evidence.

</details>

### Q49. You need zero-downtime upgrades of the API with Helm. What must be true in the chart? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Helm only patches the Deployment; zero downtime depends on the pod spec and rollout settings.

**What I'd do →**
1. `strategy.rollingUpdate: { maxUnavailable: 0, maxSurge: 25% }`.
2. Readiness probes that reflect real readiness; `preStop` sleep of a few seconds and graceful `SIGTERM` handling with `terminationGracePeriodSeconds`.
3. PDB with `minAvailable` or `maxUnavailable: 1`; at least two replicas.
4. Backward-compatible DB changes.
5. Do not set `replicas` when HPA is enabled.

**Why →** Rolling updates are only safe when Kubernetes knows when pods can and cannot take traffic.

**Interview tip →** Mention the preStop delay to cover endpoint propagation.

</details>

### Q50. A pipeline was cancelled mid-upgrade and now every deploy fails with "another operation is in progress". What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The Helm process died while the release was in `pending-upgrade`; the lock never cleared.

**What I'd do →**
1. Confirm no other pipeline is running for this environment.
2. `helm history shop -n shop` and find the `pending-upgrade` revision and the last `deployed` one.
3. `helm rollback shop <last-deployed-rev> -n shop --wait`.
4. If rollback refuses, delete only the pending revision Secret: `kubectl delete secret sh.helm.release.v1.shop.v<pending> -n shop`, then redeploy.
5. Prevent: CI `concurrency` groups, don't cancel deploy jobs mid-way, and keep `--timeout` below the CI job timeout.

**Why →** Helm's status field acts as a lock; clearing it safely means returning to a known revision.

**Interview tip →** Mention the CI timeout vs Helm timeout ordering as root cause prevention.

</details>

### Q51. A rollback is requested, but the failed release ran a migration that dropped a column. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rolling back the pods brings back code that reads a column that no longer exists; Helm will happily do it and break things further.

**What I'd do →**
1. Do not roll back blindly; check migration content and the previous code's DB expectations.
2. If data is lost, restore the column from backup or PITR into a new column and backfill, or fix forward with a hotfix image.
3. Communicate the impact and decide between restore and fix forward with the data owner.
4. Prevent: expand/contract migrations, never destructive changes in the same release as code changes, mandatory backups in a `pre-upgrade` hook for risky migrations.

**Why →** Helm rollback covers Kubernetes objects only.

**Interview tip →** State clearly that Helm cannot roll back a database.

</details>

### Q52. The team scaled a Deployment manually to 10 replicas during a traffic spike. The next deploy scaled it back to 3. Explain and fix. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The three-way merge sees live `10`, chart `3`, and patches back to `3`.

**What I'd do →**
1. Enable an HPA and omit `replicas` from the template when autoscaling is on.
2. If manual scaling is a policy, put the new count in values and deploy it.
3. With Argo CD, add `ignoreDifferences` on `/spec/replicas` when HPA manages it.

**Why →** Only one controller should own a field.

**Interview tip →** Use the phrase "field ownership" and mention server-side apply.

</details>

### Q53. How do you safely upgrade a third-party chart across a major version (for example ingress-nginx)? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Major versions may rename values, change selectors or bump CRDs.

**What I'd do →**
1. Read the changelog and upgrade notes between versions.
2. `helm show values` for old and new; diff to find renamed keys; update our override file.
3. `helm diff upgrade` against staging; watch for Deployment selector changes (immutable) or Service type changes.
4. Apply CRDs separately if the chart ships new ones in `crds/`.
5. Upgrade staging, run smoke tests, then prod in a window with a rollback plan.

**Why →** Most major-upgrade outages come from silent value renames or immutable fields.

**Interview tip →** Mention CRDs not being upgraded by Helm.

</details>

### Q54. Upgrades take a long time because `--wait` waits on a StatefulSet with `OrderedReady` and 5 replicas. Options? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rolling a StatefulSet one pod at a time, each waiting for readiness, can exceed the timeout.

**What I'd do →**
1. Increase `--timeout` realistically (pods x startup time plus margin).
2. Use `podManagementPolicy: Parallel` if the app does not need ordered startup (affects scale operations, not rolling updates).
3. Use `updateStrategy.rollingUpdate.maxUnavailable` (beta in recent Kubernetes versions) where supported to roll several pods at once.
4. Separate the stateful component into its own release so app deploys do not wait on it.

**Why →** Waiting is correct; the fix is structure and realistic timeouts.

**Interview tip →** Splitting stateful components into their own release is the strong answer.

</details>

### Q55. You discover a release in `failed` status from last week, but the app is working. Do you care? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A failed upgrade without `--atomic` may have partially applied; the next upgrade compares against the last *successful* revision's manifest, which can produce surprising diffs.

**What I'd do →**
1. `helm history` and `helm get manifest --revision <failed>` to see what was intended.
2. `helm diff upgrade` with current values to see what the next deploy will do.
3. Run a clean upgrade (or rollback) to get back to a `deployed` state.
4. Add alerting on releases in `failed` or `pending-*` status.

**Why →** Hidden failed states make the next deploy unpredictable.

**Interview tip →** Mention monitoring release status (for example a script over `helm list -A --failed --pending`).

</details>

### Q56. How do you implement a canary release when your deploy tool is Helm? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Helm itself does all-at-once rolling updates; canaries need traffic splitting.

**What I'd do →**
1. Use Argo Rollouts: the chart renders a `Rollout` instead of a `Deployment` with canary steps and analysis.
2. Or run two releases (`shop-stable`, `shop-canary`) behind a weighted Ingress/Gateway route, with weights in values.
3. Promote by upgrading stable to the canary version and removing the canary.
4. Automate analysis with Prometheus metrics.

**Why →** Helm packages and versions the pieces; a progressive delivery controller controls traffic.

**Interview tip →** Say Helm is not a progressive delivery tool and name Argo Rollouts or Flagger.

</details>

### Q57. Releases have 300 revisions each and `helm list` is slow. What happened and what do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Someone set `--history-max 0` (unlimited) on a frequently deployed release.

**What I'd do →**
1. Set `--history-max 15` in the pipeline; the next upgrade prunes old revisions.
2. If needed immediately, delete old revision Secrets except the last N: `kubectl get secret -n shop -l owner=helm,name=shop --sort-by=.metadata.creationTimestamp -o name | head -n -15 | xargs kubectl delete -n shop`.
3. Check etcd size trends.

**Why →** Each revision is a Secret up to 1 MiB; hundreds of them burden etcd and the API server.

**Interview tip →** Know the default is 10 on upgrade.

</details>

### Q58. Someone uninstalled the production release by mistake. How do you recover, and how do you prevent it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Workloads are gone; PVCs may also be gone unless kept.

**What I'd do →**
1. If `--keep-history` was used: `helm rollback shop <rev>`.
2. Otherwise reinstall from the same chart version and values in Git: `helm upgrade --install ...`.
3. Check PVCs: if retained (`resource-policy: keep` or `Retain` reclaim policy), new pods rebind; otherwise restore from backups (Velero, snapshots).
4. Prevent: RBAC so humans cannot delete in prod, `helm.sh/resource-policy: keep` on PVCs and critical Secrets, GitOps with prune protections, and Velero schedules.

**Why →** Reproducible deploys from Git make reinstall trivial; data needs its own protection.

**Interview tip →** Separate "recreate workloads" (easy) from "recover data" (needs backups).

</details>

### Q59. You need to move a release to a different namespace. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Helm cannot move a release; release records are namespaced.

**What I'd do →**
1. Install a new release in the target namespace with the same chart and values (new names may clash with cluster-scoped resources; check).
2. Migrate data (DB dump/restore or volume snapshot) and switch traffic (DNS/Ingress).
3. Uninstall the old release after validation.

**Why →** Namespaced objects cannot be moved; it is a blue/green migration.

**Interview tip →** Mention cluster-scoped name collisions during the overlap.

</details>

### Q60. During an upgrade, Helm reports success but a new CronJob was never created. What do you check? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Helm did what the rendered output said; the template probably did not render it.

**What I'd do →**
1. `helm get manifest shop -n shop | grep -A3 'kind: CronJob'`.
2. `helm get values shop --all` to check the enabling flag.
3. `helm template` locally with the same values and `-s templates/cronjob.yaml`.
4. Look for a condition typo (`.Values.cronjobs.enabled` vs `.Values.cronJobs.enabled`).
5. Add `additionalProperties: false` on that block in the schema to catch typos.

**Why →** If it is not in the manifest, Kubernetes never saw it.

**Interview tip →** Start from `helm get manifest`; it is ground truth.

</details>

### Q61. Your release takes ownership of a Namespace object. Someone uninstalls the release. What happens? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** If the chart templated the Namespace, uninstall deletes it, and with it everything inside, including other releases' resources and PVCs.

**What I'd do →**
1. Never template the release's own namespace; use `--create-namespace` (not tracked by the release) or create namespaces via a platform pipeline.
2. If a chart must create namespaces, annotate them `helm.sh/resource-policy: keep`.
3. Audit existing charts: `helm get manifest <rel> | grep -B2 -A2 'kind: Namespace'`.

**Why →** Namespace deletion cascades to everything inside.

**Interview tip →** Note that `--create-namespace` namespaces are not deleted by `helm uninstall`.

</details>

### Q62. How do you deploy the same release to three clusters consistently? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multi-cluster: same chart version, cluster-specific values.

**What I'd do →**
1. GitOps: an Argo CD ApplicationSet with a cluster generator renders one Application per cluster, values from `clusters/<name>/values.yaml`.
2. Or Helmfile with environments per cluster and `--kube-context`.
3. Roll out in waves (staging cluster, one prod cluster, then the rest).
4. Charts come from one OCI registry with replication to each region.

**Why →** Declarative multi-cluster definitions prevent snowflake clusters.

**Interview tip →** ApplicationSet cluster generator is the expected keyword.

</details>

### Q63. A deploy must not happen if the rendered diff touches a StatefulSet's volumeClaimTemplates. How do you enforce it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** `volumeClaimTemplates` are immutable; such a change fails the upgrade, or tempts someone to use `--force`.

**What I'd do →**
1. In CI, render old and new (`helm get manifest` vs `helm template`) and compare the StatefulSet's `volumeClaimTemplates` with `yq`; fail the job on change.
2. Or use helm-diff output filtered to the StatefulSet.
3. Document the manual procedure for real storage changes (orphan-delete the StatefulSet, resize PVCs, recreate).

**Why →** Catching immutable-field changes before deploy avoids failed upgrades in prod.

**Interview tip →** Mention `kubectl delete sts --cascade=orphan` as the safe procedure.

</details>

### Q64. Your org wants an audit trail of who deployed what with Helm. What do you set up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Helm history shows revisions, not people.

**What I'd do →**
1. Deploy only from CI with `--description "sha=$GIT_SHA actor=$ACTOR run=$RUN_URL"`.
2. Keep Kubernetes API audit logs (who modified Secrets/Deployments).
3. With GitOps, Git history plus Argo CD sync history is the audit trail.
4. Restrict human prod access to read-only.

**Why →** Combining Git, CI and API audit logs gives who, what and when.

**Interview tip →** `--description` is a nice small detail to mention.

</details>

## Hooks, Migrations & Testing

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q65. Design database migrations for a Helm-deployed app where the database is a subchart. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Migrations must run once per deploy, before new code needs the new schema, and the DB must exist first.

**What I'd do →**
1. A Job hook annotated `post-install,pre-upgrade`: on install the DB is created first; on upgrade it runs before new pods roll out.
2. Put the SQL in a hook ConfigMap with weight `-5`; the Job has weight `0` (on `pre-upgrade`, regular ConfigMaps are not yet updated).
3. An init container waits for the DB with `pg_isready`.
4. `hook-delete-policy: before-hook-creation,hook-succeeded`; `backoffLimit: 1`; `activeDeadlineSeconds`.
5. Idempotent, backward-compatible migrations (or a migration tool with a lock table: Flyway, Liquibase, golang-migrate).

**Why →** Correct hook timing plus idempotency gives repeatable, ordered schema changes.

**Interview tip →** Explain why `pre-install` fails with an in-release database.

</details>

### Q66. The migration Job takes 12 minutes on prod data and the upgrade times out. What do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `--timeout` (default 5m) covers hooks; the hook is killed from Helm's view and the upgrade fails.

**What I'd do →**
1. Short term: raise `--timeout 20m` for this deploy.
2. Split long data backfills out of the deploy path: run them as a separate Job or background process after the schema change.
3. Keep deploy-time migrations to fast DDL; use online schema change tools for big tables.

**Why →** Deploys should be fast and predictable; long data work belongs elsewhere.

**Interview tip →** Distinguish schema migrations (deploy path) from data backfills (async).

</details>

### Q67. A migration hook failed and the Job pod was deleted, so you cannot see why. How do you fix the chart? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `hook-delete-policy` included `hook-failed`, or the Job had `ttlSecondsAfterFinished` too low.

**What I'd do →**
1. Use `before-hook-creation,hook-succeeded` (no `hook-failed`) so failed Jobs remain.
2. Ship hook logs to the central logging stack so they survive deletion.
3. In CI, on failure, run `kubectl logs job/shop-migrate -n shop --all-containers` before anything else.

**Why →** Failed hooks are the ones you need to inspect.

**Interview tip →** Know the three delete policies and which to omit.

</details>

### Q68. You want a backup before every production upgrade. How do you do it with Helm? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The backup must finish before the migration runs.

**What I'd do →**
1. `pre-upgrade` Job with weight `-10` running `pg_dump` to S3 (or triggering a Velero/RDS snapshot), using IRSA or Pod Identity for credentials.
2. Migration Job at weight `0`.
3. Enable it only in prod via `backup.enabled`.
4. Alert if the backup Job fails; it fails the upgrade by design.

**Why →** Hook weights give strict ordering: backup, then migrate, then roll out.

**Interview tip →** Mention verifying restores periodically; a backup never restored is a hope.

</details>

### Q69. A `post-install` hook seeds demo data; it ran again on every upgrade and duplicated rows. Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The hook annotation probably lists `post-install,post-upgrade`, or the seed is not idempotent.

**What I'd do →**
1. Restrict to `post-install` if seeding is a one-time action.
2. Make the seed idempotent (`INSERT ... ON CONFLICT DO NOTHING`).
3. Note: under Argo CD, `post-install` maps to `PostSync`, which runs on every sync, so idempotency is mandatory there.

**Why →** Hooks can run more often than you expect; design them to be safe on repeat.

**Interview tip →** Mention the Argo CD mapping.

</details>

### Q70. How do you clean up external resources (DNS records, S3 buckets) when a release is uninstalled? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Helm deletes Kubernetes objects only.

**What I'd do →**
1. Prefer controllers that own lifecycle: external-dns removes records when the Ingress disappears; Crossplane/ACK delete cloud resources when their CRs are deleted.
2. If unavoidable, a `pre-delete` Job that runs a cleanup script with scoped IAM.
3. Add `helm.sh/hook-delete-policy: hook-succeeded` and a timeout; document that a failed hook blocks uninstall (`--no-hooks` skips it).

**Why →** Controllers reconcile; one-shot hooks can fail and leak resources.

**Interview tip →** Prefer reconciliation over imperative hooks.

</details>

### Q71. What would you put in `helm test` for a 3-tier app? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tests should prove the deployed wiring works.

**What I'd do →**
1. Test Pod 1: `curl -fsS http://<release>-api:3000/readyz` (API up, DB reachable).
2. Test Pod 2: `curl -fsS http://<release>-frontend:8080/api/items` (frontend proxies to API).
3. Test Pod 3 (optional): `psql -c 'select 1 from items limit 1'` (migration applied).
4. Run `helm test shop --logs` in CI right after upgrade.

**Why →** Each test validates one hop of the request path.

**Interview tip →** Map tests to hops in the architecture.

</details>

### Q72. Your hook Job never completes because Istio's sidecar keeps the pod running. How do you fix it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The main container exits, but the injected `istio-proxy` keeps running, so the Job never reaches Complete and Helm waits until timeout.

**What I'd do →**
1. With native sidecars (Kubernetes 1.29+, Istio configured to use them), the proxy stops automatically when the main container ends.
2. Otherwise disable injection for the hook: `sidecar.istio.io/inject: "false"` on the pod template if it does not need the mesh.
3. Or call `curl -X POST localhost:15020/quitquitquit` at the end of the script.

**Why →** Jobs complete only when all containers exit.

**Interview tip →** Mention native sidecars as the modern fix.

</details>

### Q73. Two hooks with the same weight run in an unexpected order. How does Helm decide? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same weight means Helm falls back to kind order, then name order.

**What I'd do →**
1. Give explicit, distinct weights to anything with a dependency (`-10`, `-5`, `0`, `5`).
2. Document weights in comments.
3. Keep the number of hooks small; combine steps into one Job if they are always sequential.

**Why →** Relying on implicit tie-breaking is fragile.

**Interview tip →** Know the sort: weight, then kind, then name.

</details>

### Q74. A migration hook needs the DB password, but the Secret is created by the same chart. Is that a problem? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** It depends on the hook event. On `post-install` and `pre-upgrade` of later releases the Secret already exists. On `pre-install` it does not.

**What I'd do →**
1. Use `post-install,pre-upgrade` timing.
2. If the password changes in this upgrade, the regular Secret is not yet updated at `pre-upgrade`; then also render the credentials into a hook Secret (lower weight) or avoid rotating secrets in the same release as migrations.
3. Or use an ExternalSecret that exists independently of the release.

**Why →** Regular resources are applied after `pre-*` hooks.

**Interview tip →** "Pre hooks see the old state of regular resources" is the key insight.

</details>

### Q75. How would you run integration tests against a chart in CI with a real cluster? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unit-level checks pass, but you want proof that it installs and works.

**What I'd do →**
1. Create an ephemeral kind cluster in the job (`helm/kind-action` in GitHub Actions).
2. Load images with `kind load docker-image`.
3. `helm install --wait --timeout 5m` with CI values; `helm test --logs`.
4. Also test upgrade from the previously released chart version to this one, and a rollback.
5. Or use chart-testing (`ct lint-and-install --upgrade`), which does install and upgrade tests for changed charts.

**Why →** Install and upgrade paths fail in different ways; test both.

**Interview tip →** Mention `ct` (chart-testing) and upgrade testing.

</details>

### Q76. A hook Job keeps failing with "already exists" on the second deploy. Why? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The first run's Job still exists, and Jobs are immutable, so Helm cannot create a new one with the same name.

**What I'd do →**
1. Add `"helm.sh/hook-delete-policy": before-hook-creation`.
2. Delete the stale Job once: `kubectl delete job shop-migrate -n shop`.

**Why →** `before-hook-creation` removes the previous instance before each run.

**Interview tip →** Note it is the default only when no delete policy annotation is set.

</details>

### Q77. You want smoke tests to block promotion from staging to prod. How do you wire `helm test` into the pipeline? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Promotion should be automatic only if staging is healthy.

**What I'd do →**
1. Staging job: `helm upgrade --install ... --atomic --wait`, then `helm test shop -n shop --logs --timeout 3m`.
2. Fail the job on non-zero exit; prod job depends on it.
3. Promote the exact same chart version and image digest to prod.

**Why →** You promote an artifact that was tested, not a rebuild.

**Interview tip →** "Promote the same artifact" is the key phrase.

</details>

### Q78. A `pre-delete` hook fails and now `helm uninstall` is stuck. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Uninstall waits for the hook and fails, leaving the release in place (or in `uninstalling`).

**What I'd do →**
1. Inspect the hook Job logs to decide whether the cleanup matters.
2. If not, `helm uninstall shop -n shop --no-hooks`.
3. Fix the hook (permissions, timeouts) and make it tolerate already-deleted dependencies.

**Why →** `--no-hooks` skips hooks for exactly this case.

**Interview tip →** Mention that hook failures block lifecycle operations by design.

</details>

## OCI, CI/CD & GitOps

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q79. Design the CI pipeline for an internal Helm chart. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Charts are code; they need tests, versioning and publishing.

**What I'd do →**
1. On PR: `helm lint --strict` per env, `helm template | kubeconform`, helm-unittest, `ct install` on kind, `helm diff` against staging.
2. Enforce a version bump if chart files changed (`ct lint` checks this).
3. On merge: `helm package`, `helm push` to OCI (ECR/GHCR) with immutable tags, cosign sign.
4. Update the environment config (Helmfile or Argo CD) via PR for promotion.

**Why →** Every published chart version is tested, immutable and traceable.

**Interview tip →** Mention version-bump enforcement and immutable tags.

</details>

### Q80. Your org is moving charts from ChartMuseum to an OCI registry. What changes for consumers? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** OCI has no `index.yaml`, no `helm repo add`, no `helm search`.

**What I'd do →**
1. Push every existing version: loop over `.tgz` files with `helm push`.
2. Update dependencies to `repository: oci://registry/charts`.
3. Update install commands to `oci://registry/charts/<name> --version X`.
4. Update Argo CD repo config (OCI enabled, no `oci://` prefix in `repoURL`).
5. Keep ChartMuseum read-only for a deprecation period.

**Why →** Same artifacts, different addressing and auth.

**Interview tip →** Mention the Argo CD `repoURL` format quirk.

</details>

### Q81. A chart version was overwritten in the registry and two clusters now run different content for "1.4.0". How do you prevent it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Mutable tags broke the assumption that a version means one content.

**What I'd do →**
1. Enable tag immutability on the registry (ECR `imageTagMutability: IMMUTABLE`; GHCR via policy or CI check).
2. CI checks whether the version exists before pushing (`helm show chart oci://.../shop --version 1.4.0` succeeds means fail).
3. Deploy by digest in sensitive environments.
4. Fix: publish 1.4.1 with the intended content and upgrade both clusters.

**Why →** Immutability is what makes versions meaningful for rollback and audit.

**Interview tip →** Mention deploying by digest.

</details>

### Q82. How would you give GitHub Actions push access to ECR for charts without long-lived keys? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Static AWS keys in CI secrets are a liability.

**What I'd do →**
1. Configure GitHub OIDC provider in AWS and an IAM role trusted for `repo:org/repo:ref:refs/heads/main`.
2. Use `aws-actions/configure-aws-credentials` with `role-to-assume`.
3. `aws ecr get-login-password | helm registry login --username AWS --password-stdin $REG`.
4. Scope the role to `ecr:PutImage`, `ecr:InitiateLayerUpload`, etc. on `charts/*` only.

**Why →** Short-lived, scoped credentials remove the leaked-key risk.

**Interview tip →** OIDC federation is the expected answer in 2025–2026.

</details>

### Q83. You are adopting Argo CD for apps currently deployed with `helm upgrade` from CI. How do you migrate? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Existing objects are Helm-managed; Argo CD will apply the same manifests and track them with its own labels.

**What I'd do →**
1. Create the Argo CD Application with the same chart, version, values and `releaseName` so rendered names match.
2. Sync without prune first; Argo CD adopts existing objects (it applies over them).
3. Verify the app is Synced/Healthy, then delete the Helm release records only (`kubectl delete secret -l owner=helm,name=shop`), not `helm uninstall`, which would delete workloads.
4. Remove the CI deploy step; check hooks behave correctly as Argo CD sync hooks.

**Why →** You transfer ownership without recreating resources.

**Interview tip →** The "delete release Secrets, not helm uninstall" detail shows real experience.

</details>

### Q84. Argo CD shows a Helm app permanently OutOfSync even right after a sync. What do you look for? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Something renders differently every time, or a controller mutates fields.

**What I'd do →**
1. `argocd app diff shop-prod` to see the fields.
2. Random or time-based template functions (`randAlphaNum`, `genCA`, `now`) or `lookup` returning empty; fix the chart to be deterministic or move secrets out.
3. Fields mutated by controllers (HPA replicas, webhook-injected values, defaulted fields); add `ignoreDifferences`.
4. Consider `ServerSideApply=true` sync option.

**Why →** GitOps needs deterministic rendering.

**Interview tip →** Name the random functions explicitly.

</details>

### Q85. How do you promote a chart version from staging to prod in a GitOps setup? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Promotion should be a reviewed change of one value.

**What I'd do →**
1. Each environment's Application or Helmfile pins `targetRevision`/`version`.
2. After staging tests pass, a PR bumps prod's version (manual or automated with Kargo/Renovate).
3. Merge triggers Argo CD sync; rollback is reverting the PR.

**Why →** Git history becomes the deploy history.

**Interview tip →** Rollback equals git revert in GitOps.

</details>

### Q86. Should you use Helmfile or Argo CD ApplicationSets to manage 30 platform charts? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both compose many releases; they differ in push vs pull model.

**What I'd do →**
1. Helmfile: CI-driven push, real Helm releases (hooks, `helm list`), great diffs, simple for few clusters; needs cluster credentials in CI.
2. ApplicationSets: pull-based, continuous reconciliation and drift correction, UI, multi-cluster at scale; Helm used only for templating.
3. For 30 charts across several clusters, I would pick Argo CD with ApplicationSets; Helmfile for small setups or bootstrap (installing Argo CD itself).

**Why →** Reconciliation and multi-cluster visibility matter more as scale grows.

**Interview tip →** Mention using Helmfile to bootstrap Argo CD.

</details>

### Q87. Developers want to deploy feature branches as preview environments using your chart. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ephemeral releases per PR, cleaned up automatically.

**What I'd do →**
1. Argo CD ApplicationSet with the Pull Request generator: one Application per open PR, namespace `pr-<number>`, values setting the image tag to the PR build and host `pr-<number>.preview.example.com`.
2. Use small resources and an in-cluster DB subchart with no persistence.
3. When the PR closes, the generator deletes the Application and its namespace.
4. Without Argo CD: CI runs `helm upgrade --install shop-pr-123 -n pr-123 --create-namespace` and `helm uninstall` plus namespace delete on close.

**Why →** Automated lifecycle avoids orphaned preview environments.

**Interview tip →** Name the ApplicationSet PR generator.

</details>

### Q88. How do you make Helm deploys in CI resilient to flaky network or API timeouts? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A network blip mid-upgrade can leave a pending release.

**What I'd do →**
1. Pre-pull chart and dependencies early in the job (`helm pull`, `helm dependency build`).
2. Use `--atomic` and a Helm `--timeout` lower than the CI job timeout.
3. On retry, first check status: if `pending-*`, roll back before retrying.
4. Serialize deploys per environment.

**Why →** Retries are only safe if the release is in a clean state.

**Interview tip →** Mention checking release status before retrying.

</details>

### Q89. Argo CD fails to render your chart that uses `lookup` to keep a generated password. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Argo CD renders without cluster access, so `lookup` returns empty and a new password is generated each time, causing drift (and potential outages).

**What I'd do →**
1. Remove secret generation from the chart; create the Secret externally (ExternalSecret, SealedSecret, or a one-time bootstrap).
2. Use the chart's `existingSecret` value to reference it.
3. If needed, add `ignoreDifferences` on that Secret's data while migrating.

**Why →** GitOps charts must be deterministic.

**Interview tip →** "lookup doesn't work in Argo CD" is a must-know fact.

</details>

### Q90. How do you verify chart signatures before deployment? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Supply chain: only trusted, signed charts should reach the cluster.

**What I'd do →**
1. Sign in CI with cosign keyless (GitHub OIDC) after `helm push`.
2. Verify in CD: `cosign verify <chart-ref>@<digest> --certificate-identity-regexp ... --certificate-oidc-issuer ...` before `helm upgrade`.
3. For images inside, enforce signature verification with Kyverno `verifyImages` or Sigstore policy-controller.
4. For classic repos, `helm install --verify` with PGP provenance.

**Why →** Verifying at deploy time blocks tampered or unofficial artifacts.

**Interview tip →** Mention both chart and image signing.

</details>

### Q91. CI deploys with `helm upgrade` and also Argo CD watches the same namespace. Is that OK? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two controllers owning the same objects will fight: Argo CD self-heal reverts CI changes, or CI overwrites Argo CD state.

**What I'd do →**
1. Pick one owner per release. Remove the CI deploy, or exclude the namespace from Argo CD.
2. CI can still build, push charts and images, and update Git; Argo CD applies.

**Why →** Single ownership is the foundation of GitOps.

**Interview tip →** "One writer per resource" is the principle.

</details>

### Q92. How would you bootstrap a brand new cluster with platform charts? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Chicken-and-egg: Argo CD itself must be installed somehow.

**What I'd do →**
1. Terraform creates the cluster, then a Helm provider or Helmfile installs Argo CD (and CRDs).
2. Argo CD's root "app of apps" or ApplicationSet installs everything else (ingress, cert-manager, external-dns, monitoring) with sync waves for ordering.
3. Argo CD then manages itself from Git.

**Why →** Minimal imperative bootstrap; everything after is declarative.

**Interview tip →** Mention "app of apps" and sync waves.

</details>

## Secrets & Security

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q93. A secret was committed in plaintext in `values-prod.yaml`. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The secret is compromised the moment it is pushed; removing it from Git does not un-leak it.

**What I'd do →**
1. Rotate the credential immediately (database password, API key).
2. Remove it from the file; move it to a SOPS-encrypted file or a secret manager.
3. Optionally rewrite history (`git filter-repo`), but assume it is already copied.
4. Add secret scanning (gitleaks, GitHub push protection) to block recurrence.

**Why →** Rotation is the only real fix.

**Interview tip →** Say "rotate first" before anything else.

</details>

### Q94. Who can read secrets passed to Helm, and how do you reduce exposure? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Values containing secrets are stored in the release Secret, readable by anyone with `get secrets` in the namespace; they also appear in `--debug` output and CI logs.

**What I'd do →**
1. Restrict `get/list secrets` via RBAC; developers get read access to other resources only.
2. Use `existingSecret` references so values contain names, not secret values.
3. Use External Secrets Operator so values never contain secrets.
4. Never run `--debug` or `helm get values` in shared CI logs for prod.

**Why →** The release record is a copy of every secret you pass through Helm.

**Interview tip →** Knowing the release Secret leak path is a strong signal.

</details>

### Q95. Set up helm-secrets with SOPS and AWS KMS for prod and age for dev. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Different keys per environment; prod decryptable only by the prod deploy role.

**What I'd do →**
1. `.sops.yaml`:
```yaml
creation_rules:
  - path_regex: secrets/prod/.*\.yaml$
    kms: arn:aws:kms:ap-south-1:123456789012:key/abcd-1234
  - path_regex: secrets/dev/.*\.yaml$
    age: age1x7...dev
```
2. `sops -e -i secrets/prod/shop.yaml`, commit.
3. CI prod role has `kms:Decrypt` on that key only; dev CI uses `SOPS_AGE_KEY` from CI secrets.
4. Deploy with `-f secrets://secrets/prod/shop.yaml`.

**Why →** Encryption keys enforce who can deploy which environment's secrets.

**Interview tip →** Mention per-environment key separation.

</details>

### Q96. A chart from the internet requests cluster-admin. How do you evaluate and contain it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Many charts over-request RBAC.

**What I'd do →**
1. `helm template` and review every ClusterRole/Role and binding.
2. Check whether namespaced mode exists (`rbac.clusterWide: false`, `watchNamespace`).
3. If not, replace with a reduced ClusterRole via values or a post-renderer.
4. Install in a dedicated namespace with NetworkPolicies, Pod Security restricted, and image signature verification.

**Why →** Rendered output tells you exactly what you are granting.

**Interview tip →** Always review rendered RBAC before installing third-party charts.

</details>

### Q97. Your security team asks you to scan Helm charts for misconfigurations in CI. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Misconfigurations live in rendered manifests.

**What I'd do →**
1. `trivy config ./shop` (scans Helm charts directly) or `helm template ... | checkov -f -` / `kube-linter lint ./shop`.
2. Run per environment's values, since prod settings differ.
3. Fail on HIGH findings; document exceptions in code.

**Why →** Scanning rendered output reflects what will actually run.

**Interview tip →** Mention scanning with each environment's values.

</details>

### Q98. How do you rotate a database password managed through Helm without downtime? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Changing the password in values updates the Secret, but the DB still has the old one, and pods need restarting.

**What I'd do →**
1. Prefer dual credentials: create a second DB user or set the new password on the DB first (ALTER ROLE).
2. Update the secret source (SOPS file or Secrets Manager), deploy; checksum annotation rolls pods with the new password.
3. Verify, then revoke the old password.
4. With External Secrets plus Reloader, rotation happens without a Helm deploy.

**Why →** Rotation must be coordinated between the DB and consumers.

**Interview tip →** Say "change at the source first, then roll consumers, then revoke".

</details>

### Q99. Your CI service account for Helm has cluster-admin "because it was easier". How do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A compromised pipeline would own the cluster.

**What I'd do →**
1. Per-namespace ServiceAccounts or IAM-mapped roles with a Role covering only the kinds the chart renders plus Secrets (release storage).
2. List needed kinds: `helm template ... | yq '.kind' | sort -u`.
3. Separate platform pipelines (which need cluster-scoped kinds) from app pipelines.
4. Audit with `kubectl auth can-i --list --as system:serviceaccount:ci:deployer -n shop`.

**Why →** Least privilege limits blast radius of CI compromise.

**Interview tip →** Mention deriving the permission list from rendered kinds.

</details>

### Q100. How do you stop someone installing an unapproved chart into production? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Helm itself does not enforce sources; the cluster must.

**What I'd do →**
1. No human write access to prod; only CI or Argo CD with restricted Argo CD Projects (`sourceRepos` allow-list).
2. Admission policies (Kyverno) requiring images from approved registries and signed.
3. Network egress restrictions so CI pulls charts only from the internal registry mirror.
4. Audit logs and alerts on new Helm release Secrets created outside CI identities.

**Why →** Controls at the deploy path and admission layer, not trust in users.

**Interview tip →** Argo CD Project `sourceRepos` is a nice specific detail.

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
