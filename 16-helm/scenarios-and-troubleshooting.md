# Helm — Scenarios & Troubleshooting

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

> 100 real-world "what would you do" scenarios and 100 real Helm error messages with diagnosis and fixes · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Helm](./README.md) · [🏠 Home](../README.md)

## Table of contents

**Scenarios (Q1–Q100)**
- [Chart Design & Authoring](#chart-design--authoring) — Q1–Q18
- [Values & Environments](#values--environments) — Q19–Q34
- [Dependencies, Subcharts & Library Charts](#dependencies-subcharts--library-charts) — Q35–Q46
- [Releases, Upgrades & Rollbacks](#releases-upgrades--rollbacks) — Q47–Q64
- [Hooks, Migrations & Testing](#hooks-migrations--testing) — Q65–Q78
- [OCI, CI/CD & GitOps](#oci-cicd--gitops) — Q79–Q92
- [Secrets & Security](#secrets--security) — Q93–Q100

**Troubleshooting (Q101–Q200)**
- [Install & Upgrade Failures](#install--upgrade-failures) — Q101–Q125
- [Templating & Rendering Errors](#templating--rendering-errors) — Q126–Q148
- [Values, Schema & Type Errors](#values-schema--type-errors) — Q149–Q163
- [Dependencies, Repositories & OCI](#dependencies-repositories--oci) — Q164–Q180
- [Hooks, Tests, GitOps & Secrets](#hooks-tests-gitops--secrets) — Q181–Q200

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

---

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

---

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

---

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

---

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

---

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

---

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

---

## Install & Upgrade Failures

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. `Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The latest revision is stuck in `pending-install`, `pending-upgrade` or `pending-rollback` because a previous Helm process was killed (cancelled CI job, runner died, Ctrl-C), or another deploy really is running.

**Diagnose →**
```bash
helm history shop -n shop --max 5
helm list -n shop --pending
kubectl get secret -n shop -l owner=helm,name=shop -L status,version
```

**Fix →** Make sure nothing else is running. Then `helm rollback shop <last-deployed-rev> -n shop`. If that is refused, delete only the stuck revision Secret (`kubectl delete secret sh.helm.release.v1.shop.v<N> -n shop`) and redeploy. For a stuck `pending-install` with no prior revision, `helm uninstall shop -n shop`.

**Prevent →** CI concurrency groups per environment, Helm `--timeout` shorter than the CI job timeout, and never cancel deploy jobs mid-run.

</details>

### Q102. `Error: INSTALLATION FAILED: rendered manifests contain a resource that already exists. Unable to continue with install: Service "shop-api" in namespace "shop" exists and cannot be imported into the current release: invalid ownership metadata; label validation error: missing key "app.kubernetes.io/managed-by": must be set to "Helm"; annotation validation error: missing key "meta.helm.sh/release-name": must be set to "shop"` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The object was created outside Helm (`kubectl apply`, another tool), by another release, or survived a previous uninstall (`resource-policy: keep`, a failed uninstall).

**Diagnose →**
```bash
kubectl get svc shop-api -n shop -o yaml | yq '.metadata.labels, .metadata.annotations'
helm list -A | grep -i shop
```

**Fix →** If it is safe for this release to own it, adopt it:
```bash
kubectl label svc shop-api -n shop app.kubernetes.io/managed-by=Helm --overwrite
kubectl annotate svc shop-api -n shop meta.helm.sh/release-name=shop meta.helm.sh/release-namespace=shop --overwrite
```
On Helm 3.17+ you can use `--take-ownership` instead. If it belongs to another release, rename the resource in your chart (`fullnameOverride`) or delete the stale object.

**Prevent →** One tool owns each object; migrate kubectl-managed resources deliberately.

</details>

### Q103. `Error: INSTALLATION FAILED: cannot re-use a name that is still in use` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A release with that name already exists in the namespace, possibly in `failed` or `uninstalled` (kept history) status.

**Diagnose →**
```bash
helm list -n shop --all
helm history shop -n shop
```

**Fix →** Use `helm upgrade --install` instead of `install`. If the existing release is failed or uninstalled and should be replaced, `helm uninstall shop -n shop` first, or `helm install --replace`.

**Prevent →** Always use `upgrade --install` in automation.

</details>

### Q104. `Error: UPGRADE FAILED: context deadline exceeded` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With `--wait`, resources did not become ready within `--timeout` (default 5m): image pull failures, crash loops, failing readiness probes, pending pods (no capacity), a hook Job running long.

**Diagnose →**
```bash
kubectl get pods -n shop
kubectl describe pod -n shop -l app.kubernetes.io/instance=shop | sed -n '/Events/,$p'
kubectl logs -n shop deploy/shop-api --previous
kubectl get events -n shop --sort-by=.lastTimestamp | tail -20
```

**Fix →** Fix the underlying pod issue and redeploy; raise `--timeout` only if the rollout is legitimately slow. Roll back if needed.

**Prevent →** Realistic probes and timeouts, capacity headroom, and capture diagnostics in the CI failure step.

</details>

### Q105. `Error: UPGRADE FAILED: release shop failed, and has been rolled back due to atomic being set: context deadline exceeded` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Same as a timeout, but `--atomic` already restored the previous revision, so the broken pods are gone.

**Diagnose →**
```bash
helm history shop -n shop       # failed revision, then the rollback revision
helm get manifest shop -n shop --revision <failed-rev> | less
kubectl get events -n shop --sort-by=.lastTimestamp | tail -30
```
Reproduce in staging or deploy without `--atomic` to a test namespace to inspect pods.

**Fix →** Correct the image, config or probe that failed and redeploy.

**Prevent →** In CI, on failure, collect `kubectl describe` and logs while the rollout is in progress (a background watcher), because `--atomic` removes the evidence.

</details>

### Q106. `Error: UPGRADE FAILED: cannot patch "shop-api" with kind Deployment: Deployment.apps "shop-api" is invalid: spec.selector: Invalid value: v1.LabelSelector{...}: field is immutable` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The chart's selector labels changed: a new label added to `selectorLabels`, the chart version or appVersion included in the selector, or the release/name override changed.

**Diagnose →**
```bash
kubectl get deploy shop-api -n shop -o jsonpath='{.spec.selector.matchLabels}'; echo
helm template shop ./shop -f values-prod.yaml -s templates/api-deployment.yaml | yq '.spec.selector'
```

**Fix →** Revert the selector change if possible. If the change is intended, orphan-delete and redeploy:
```bash
kubectl delete deploy shop-api -n shop --cascade=orphan
helm upgrade shop ./shop -n shop -f values-prod.yaml --wait
kubectl delete rs -n shop -l <old-selector>     # after new pods are Ready
```

**Prevent →** Separate `selectorLabels` with only stable labels; unit test them.

</details>

### Q107. `Error: UPGRADE FAILED: "shop" has no deployed releases` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Every revision of the release is `failed` (the first install failed and was not cleaned up), so there is no successful baseline to upgrade from. Older Helm 3 versions hit this often; newer ones handle `upgrade --install` over a failed first install better.

**Diagnose →**
```bash
helm history shop -n shop
```

**Fix →** `helm uninstall shop -n shop` and install again (data in PVCs is affected if they are part of the release), or upgrade Helm and rerun `upgrade --install`.

**Prevent →** Use `--atomic` on installs so failures are cleaned up automatically.

</details>

### Q108. `Error: INSTALLATION FAILED: create: failed to create: namespaces "shop" not found` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Helm does not create the target namespace by default.

**Diagnose →** `kubectl get ns shop`.

**Fix →** Add `--create-namespace`, or create it beforehand (`kubectl create ns shop`), ideally through your platform pipeline with labels and quotas.

**Prevent →** Namespaces managed declaratively by the platform team; app pipelines use `--create-namespace` only in dev.

</details>

### Q109. `Error: Kubernetes cluster unreachable: Get "http://localhost:8080/version": dial tcp [::1]:8080: connect: connection refused` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Helm found no kubeconfig (it falls back to localhost:8080): `KUBECONFIG` unset in CI, wrong path, or the file is not readable by the user.

**Diagnose →**
```bash
echo $KUBECONFIG; ls -l ~/.kube/config
kubectl config current-context
helm env | grep KUBE
```

**Fix →** Export `KUBECONFIG`, pass `--kubeconfig`/`--kube-context`, or generate it (`aws eks update-kubeconfig --name prod --region ap-south-1`).

**Prevent →** CI steps that set up kubeconfig explicitly and verify with `kubectl version` before Helm runs.

</details>

### Q110. `Error: INSTALLATION FAILED: unable to build kubernetes objects from release manifest: resource mapping not found for name: "shop" namespace: "" from "": no matches for kind "ServiceMonitor" in version "monitoring.coreos.com/v1" ensure CRDs are installed first` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The chart renders a custom resource whose CRD is not installed (Prometheus Operator missing), or is installed in a different API version.

**Diagnose →**
```bash
kubectl api-resources | grep -i servicemonitor
kubectl get crd servicemonitors.monitoring.coreos.com
```

**Fix →** Install the CRD (kube-prometheus-stack or prometheus-operator-crds chart) first, or disable the feature (`metrics.serviceMonitor.enabled=false`). Better, guard the template with `.Capabilities.APIVersions.Has "monitoring.coreos.com/v1/ServiceMonitor"`.

**Prevent →** Capability checks in charts and ordered installs (Helmfile `needs`, Argo CD sync waves).

</details>

### Q111. `Error: UPGRADE FAILED: unable to build kubernetes objects from current release manifest: resource mapping not found for name: "shop-api" namespace: "" from "": no matches for kind "PodDisruptionBudget" in version "policy/v1beta1"` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The cluster was upgraded and removed an API version that the **stored** previous release manifest still uses. Helm must parse the old manifest to compute the diff and cannot.

**Diagnose →**
```bash
helm get manifest shop -n shop | grep -B1 -A1 'policy/v1beta1'
kubectl version
```

**Fix →**
```bash
helm plugin install https://github.com/helm/helm-mapkubeapis
helm mapkubeapis shop -n shop
helm upgrade shop ./shop -n shop -f values-prod.yaml   # chart updated to policy/v1
```

**Prevent →** Before cluster upgrades, scan releases with `pluto detect-helm -o wide` and upgrade charts to supported APIs first.

</details>

### Q112. `Error: INSTALLATION FAILED: chart requires kubeVersion: >=1.29.0 which is incompatible with Kubernetes v1.30.4-eks-a737599` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The constraint lacks the `-0` pre-release suffix, so SemVer treats `v1.30.4-eks-...` as a pre-release that does not satisfy `>=1.29.0`. Or the cluster is genuinely too old.

**Diagnose →** `kubectl version` and check `kubeVersion` in `Chart.yaml`.

**Fix →** Change to `kubeVersion: ">=1.29.0-0"`. For a third-party chart, pull, patch and repackage, or ask upstream.

**Prevent →** Always append `-0` in `kubeVersion` constraints and `semverCompare` calls.

</details>

### Q113. `Error: INSTALLATION FAILED: release name "customer-analytics-platform-reporting-service-production" exceeds max length of 53` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Release names are limited to 53 characters.

**Diagnose →** `echo -n "<name>" | wc -c`.

**Fix →** Shorten the release name (`reporting-prod`) and use `fullnameOverride` if resource names must stay specific.

**Prevent →** A naming convention for releases: short app name plus environment.

</details>

### Q114. `Error: INSTALLATION FAILED: Service "shop-frontend-...-very-long" is invalid: metadata.name: Invalid value: "...": must be no more than 63 characters` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A template builds names by concatenation without truncating.

**Diagnose →** `helm template ... | yq 'select(.metadata.name | length > 63) | .metadata.name'`.

**Fix →** Use the `fullname` helper pattern with `trunc 63 | trimSuffix "-"` for every generated name, including suffixed ones (`printf "%s-frontend" $fullname | trunc 63 | trimSuffix "-"`).

**Prevent →** A unit test rendering with a 53-character release name.

</details>

### Q115. `Error: UPGRADE FAILED: failed to replace object: Service "shop-api" is invalid: spec.clusterIP: Invalid value: "": field is immutable` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `--force` was used; Helm replaced the whole object, and the rendered Service has no `clusterIP`, which the API server treats as an attempt to change an immutable field.

**Diagnose →** Check the pipeline flags for `--force`.

**Fix →** Remove `--force` and upgrade normally. If you truly need to recreate the object, delete it explicitly and upgrade.

**Prevent →** Ban `--force` in CI; handle immutable-field changes with explicit runbooks.

</details>

### Q116. `Error: create: failed to create: Secret "sh.helm.release.v1.platform.v12" is invalid: data: Too long: must have at most 1048576 bytes` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The release record (chart files, values and manifests, gzipped) exceeds the 1 MiB object limit. Common with umbrella charts, large embedded files (dashboards, CRDs in templates), or junk included because `.helmignore` is incomplete.

**Diagnose →**
```bash
helm package ./platform && ls -lh platform-*.tgz
tar -tzvf platform-*.tgz | sort -k3 -n | tail -20
```

**Fix →** Exclude unneeded files via `.helmignore`, move big dashboards into separate ConfigMaps managed elsewhere, split the umbrella into multiple releases. As a last resort, `HELM_DRIVER=sql`.

**Prevent →** CI check on package size; prefer smaller, independent releases.

</details>

### Q117. `Error: INSTALLATION FAILED: secrets is forbidden: User "system:serviceaccount:ci:deployer" cannot list resource "secrets" in API group "" in the namespace "shop"` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CI identity lacks permission on Secrets, which Helm needs for its release storage (even if the chart creates no Secrets).

**Diagnose →**
```bash
kubectl auth can-i list secrets -n shop --as system:serviceaccount:ci:deployer
kubectl auth can-i create deployments -n shop --as system:serviceaccount:ci:deployer
```

**Fix →** Grant a Role with `get, list, watch, create, update, patch, delete` on `secrets` plus the kinds the chart renders, bound to the SA in that namespace.

**Prevent →** Template the deployer Role per namespace as part of namespace provisioning.

</details>

### Q118. `Error: UPGRADE FAILED: an error occurred while rolling back the release. original upgrade error: context deadline exceeded: ...` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `--atomic` tried to roll back but the rollback also failed: the previous revision is not deployable anymore (deleted image tag, removed API, immutable field change in between) or the rollback also timed out.

**Diagnose →**
```bash
helm history shop -n shop
kubectl get pods -n shop; kubectl get events -n shop --sort-by=.lastTimestamp | tail
```

**Fix →** Fix forward: deploy a corrected chart or values. If the release is stuck in `pending-rollback`, clear it as in Q101. Restore missing images from the registry if possible.

**Prevent →** Immutable image tags with retention covering your rollback window; avoid immutable-field changes.

</details>

### Q119. `Error: UPGRADE FAILED: failed to create resource: Ingress.networking.k8s.io "shop" is invalid: spec.rules[0].host: Invalid value: "Shop.Example.com": a lowercase RFC 1123 subdomain must consist of lower case alphanumeric characters, '-' or '.'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A value contains uppercase or invalid characters.

**Diagnose →** `helm get values shop -n shop --all | yq '.ingress'`.

**Fix →** Correct the value; in the template, normalize with `| lower`.

**Prevent →** A `pattern` constraint in `values.schema.json` for hostnames (`^[a-z0-9]([-a-z0-9.]*[a-z0-9])?$`).

</details>

### Q120. `Error: INSTALLATION FAILED: admission webhook "validate.kyverno.svc-fail" denied the request: resource Deployment/shop/shop-api was blocked due to the following policies require-run-as-nonroot: ...` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The rendered manifest violates a cluster policy (Kyverno, Gatekeeper, Pod Security Admission).

**Diagnose →**
```bash
helm template shop ./shop -f values-prod.yaml -s templates/api-deployment.yaml | yq '.spec.template.spec.securityContext, .spec.template.spec.containers[].securityContext'
kubectl get clusterpolicy require-run-as-nonroot -o yaml
```

**Fix →** Set the required security context in values or chart defaults. Request a documented policy exception only when justified.

**Prevent →** Run the same policies in CI against rendered output (Kyverno CLI, Conftest) and use `--dry-run=server` before deploying.

</details>

### Q121. `Error: UPGRADE FAILED: cannot patch "shop-postgres" with kind StatefulSet: StatefulSet.apps "shop-postgres" is invalid: spec: Forbidden: updates to statefulset spec for fields other than 'replicas', 'ordinals', 'template', 'updateStrategy', 'persistentVolumeClaimRetentionPolicy' and 'minReadySeconds' are forbidden` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The new chart changes `volumeClaimTemplates`, `serviceName`, `selector` or `podManagementPolicy`, which are immutable on StatefulSets.

**Diagnose →**
```bash
helm diff upgrade shop ./shop -n shop -f values-prod.yaml | grep -A20 'StatefulSet'
```

**Fix →** Revert the change, or recreate the StatefulSet without touching pods and PVCs:
```bash
kubectl delete sts shop-postgres -n shop --cascade=orphan
helm upgrade shop ./shop -n shop -f values-prod.yaml
```
For a storage size increase, resize the PVCs directly (if the StorageClass allows expansion) before recreating.

**Prevent →** CI check for StatefulSet immutable fields; document storage change procedures.

</details>

### Q122. `Error: INSTALLATION FAILED: failed to install CRD crds/crd.yaml: ... the server could not find the requested resource` or CRD apply errors on a fresh cluster. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CRD in `crds/` uses `apiextensions.k8s.io/v1beta1` (removed in 1.22), or the identity lacks cluster-scope rights for CRDs.

**Diagnose →**
```bash
helm show crds <chart> | grep apiVersion | sort -u
kubectl auth can-i create customresourcedefinitions
```

**Fix →** Use a chart version with `apiextensions.k8s.io/v1` CRDs; have a cluster admin pipeline install CRDs, then install the chart with `--skip-crds`.

**Prevent →** Separate CRD lifecycle from app releases.

</details>

### Q123. After an upgrade, the new chart version deployed but pods still run the old image. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The image tag did not change in the render (tag comes from values, but `appVersion` was bumped; or the reverse), `--reuse-values` kept the old tag, or the tag is mutable (`latest`) with `IfNotPresent` pull policy.

**Diagnose →**
```bash
helm get values shop -n shop --all | yq '.api.image'
helm get manifest shop -n shop | grep 'image:'
kubectl get pods -n shop -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}'
```

**Fix →** Pass the tag explicitly (`--set-string api.image.tag=...`) or default to `.Chart.AppVersion` consistently; drop `--reuse-values`; use immutable tags.

**Prevent →** Unique immutable tags per build and a clear single source of truth for the tag.

</details>

### Q124. `Error: INSTALLATION FAILED: Unable to continue with install: CustomResourceDefinition "certificates.cert-manager.io" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata; annotation validation error: key "meta.helm.sh/release-namespace" must equal "cert-manager": current value is "kube-system"` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A previous installation of the same software in another namespace (or another release) owns the cluster-scoped CRDs.

**Diagnose →**
```bash
kubectl get crd certificates.cert-manager.io -o yaml | yq '.metadata.annotations, .metadata.labels'
helm list -A | grep cert-manager
```

**Fix →** Decide the single owner. Either uninstall the old release first (careful: deleting CRDs deletes all Certificates), or re-annotate the CRDs to the new release, or install the chart with CRD installation disabled (`crds.enabled=false` / `installCRDs=false`) and manage CRDs separately.

**Prevent →** Cluster-wide components are installed once by the platform team; CRDs managed in their own pipeline.

</details>

### Q125. `helm upgrade` succeeded but `helm list` shows the release `failed` a minute later after a retry. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two pipelines deployed concurrently (one succeeded, the other failed and wrote a newer failed revision), or a retry ran after a timeout while the first operation finished.

**Diagnose →**
```bash
helm history shop -n shop --max 10      # look at timestamps and descriptions
```

**Fix →** Run a fresh `upgrade --install` from the intended commit to get back to `deployed`.

**Prevent →** Deploy concurrency control (`concurrency: deploy-prod` in GitHub Actions, `resource_group` in GitLab) and `--description` with the pipeline ID.

</details>

---

## Templating & Rendering Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. `Error: template: shop/templates/api-deployment.yaml:24:28: executing "shop/templates/api-deployment.yaml" at <.Values.api.image.tag>: nil pointer evaluating interface {}.tag` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A parent key in the path is missing: `api.image` does not exist in the merged values (typo, a values file set `image: null`, or the key was renamed).

**Diagnose →**
```bash
helm template shop ./shop -f values-prod.yaml --debug 2>&1 | head -5
yq '.api' values.yaml values-prod.yaml
```

**Fix →** Add the missing default to `values.yaml`, or make the template defensive: `{{ (.Values.api.image | default dict).tag | default .Chart.AppVersion }}`, or wrap in `with`.

**Prevent →** Defaults for every referenced key and a schema requiring the parent objects.

</details>

### Q127. `Error: YAML parse error on shop/templates/api-deployment.yaml: error converting YAML to JSON: yaml: line 31: did not find expected key` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong indentation in the rendered output: `indent` instead of `nindent`, a wrong number, or `{{-` that ate a newline. The line number refers to the **rendered** document, not the template.

**Diagnose →**
```bash
helm template shop ./shop -f values-prod.yaml -s templates/api-deployment.yaml --debug | cat -n | sed -n '25,35p'
```

**Fix →** Use the pattern `key:` on its own line followed by `{{- toYaml .Values.x | nindent N }}` with N equal to the key's indentation plus two.

**Prevent →** Render all templates in CI with every environment's values; unit tests on tricky blocks.

</details>

### Q128. `Error: YAML parse error on shop/templates/configmap.yaml: error converting YAML to JSON: yaml: line 7: mapping values are not allowed in this context` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A rendered scalar contains `: ` (for example a URL or a message) and is not quoted, so YAML reads it as a nested mapping.

**Diagnose →** Render the file with `--debug` and look at line 7.

**Fix →** Quote values: `{{ .Values.app.banner | quote }}`; for multi-line content use a block scalar with `nindent`:
```yaml
data:
  banner: |-
    {{- .Values.app.banner | nindent 4 }}
```

**Prevent →** Quote every string value rendered into YAML unless it must be a number or boolean.

</details>

### Q129. `Error: template: shop/templates/api-deployment.yaml:18:12: executing "..." at <include "shop.labels" .>: error calling include: template: no template "shop.labels" associated with template "gotpl"` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The named template does not exist: typo, it is defined under another name (`shop.labels` vs `shop-labels`), the helpers file lacks the `define`, or it lives in a library chart that was not added as a dependency (or `helm dependency build` was not run).

**Diagnose →**
```bash
grep -rn 'define "' ./shop/templates ./shop/charts 2>/dev/null
helm dependency list ./shop
```

**Fix →** Correct the name or add the definition; run `helm dependency build` for library charts.

**Prevent →** Lint in CI and keep helper names prefixed consistently.

</details>

### Q130. `Error: template: shop/templates/_helpers.tpl:14:14: executing "shop.fullname" at <.Chart.Name>: can't evaluate field Chart in type string` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The helper was called with the wrong context, for example inside `range` or `with` where `.` is a string, or with a dict that does not carry the root.

**Diagnose →** Look at the call site: `{{ include "shop.fullname" . }}` inside `{{ range .Values.hosts }}`.

**Fix →** Pass the root: `{{ include "shop.fullname" $ }}`, or for multi-parameter helpers pass `(dict "root" $ "host" .)` and use `.root` inside.

**Prevent →** Document each helper's expected context in a template comment.

</details>

### Q131. `Error: parse error at (shop/templates/api-deployment.yaml:40): function "toYml" not defined` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Typo or a function from another tool (`toYml`, `indentYaml`, `env` from older Sprig docs that Helm disables).

**Diagnose →** Check the function name against Helm's template function list.

**Fix →** Use the real name (`toYaml`). Note that Helm removes `env` and `expandenv` for security, so environment variables cannot be read in templates; pass them as values instead.

**Prevent →** `helm lint` in pre-commit.

</details>

### Q132. `Error: parse error at (shop/templates/ingress.yaml:42): unexpected {{end}}` or `unexpected EOF` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Unbalanced control structures: an extra `{{ end }}`, a missing one, or an `if` closed inside a `range` that opened outside.

**Diagnose →** Count openers and closers: `grep -oE '\{\{-? *(if|range|with|define|end)' templates/ingress.yaml`.

**Fix →** Balance blocks; indent template logic consistently so structure is visible.

**Prevent →** Editor support for Helm templates (VS Code Kubernetes extension, helm-ls language server).

</details>

### Q133. `Error: template: ... at <.Release.Name>: can't evaluate field Release in type map[string]interface {}` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `.Release` used inside `range` or `with`, where `.` is now a values item.

**Diagnose →** Find the enclosing `range`/`with` at the reported line.

**Fix →** Use `$.Release.Name` (and `$.Values`, `$.Chart`) inside scoped blocks.

**Prevent →** Habit: always use `$` for root objects inside loops.

</details>

### Q134. `Error: execution error at (shop/templates/secret.yaml:9:15): postgres.auth.password is required` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The chart uses `required` and the value was not supplied: a secrets file missing from the command, helm-secrets failed to decrypt, or the key path differs.

**Diagnose →**
```bash
helm template shop ./shop -f values-dev.yaml -f secrets://secrets/dev.yaml >/dev/null
yq '.postgres.auth' values-dev.yaml
```

**Fix →** Supply the value (`-f` secrets file, `--set-string`), or reference an existing Secret if the chart supports it.

**Prevent →** Pipelines always include the secrets file; schema documents the requirement.

</details>

### Q135. `Error: template: shop/templates/configmap.yaml:8:18: executing "..." at <.Values.app.config>: wrong type for value; expected string; got map[string]interface {}` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A string function (`quote`, `b64enc`, `trunc`) received a map: the values structure changed from a scalar to a nested object.

**Diagnose →** `yq '.app.config' values*.yaml` to see its type in each layer.

**Fix →** Use `toYaml`/`toJson` for maps, or reference the scalar inside the map.

**Prevent →** Schema `type` for that key so a wrong structure fails with a clear message.

</details>

### Q136. Rendered output contains `%!s(float64=1.1)` or `%!d(string=3000)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `printf` verbs do not match the value types (YAML number parsed as float64, or a quoted string passed to `%d`).

**Diagnose →** `helm template ... | grep '%!'`.

**Fix →** Convert: `printf "%s:%s" .repo (.tag | toString)`, `printf "%d" (.port | int)`.

**Prevent →** Quote string-like values in `values.yaml`; `toString` before `%s`.

</details>

### Q137. `error validating data: ValidationError(Deployment.spec.template.spec.containers[0].env[2].value): invalid type for io.k8s.api.core.v1.EnvVar.value: got "boolean", expected "string"` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An env value rendered unquoted as `true`, `false` or a number.

**Diagnose →** Render and inspect the `env` block.

**Fix →** `value: {{ .Values.featureFlag | quote }}` for every env var.

**Prevent →** Render env vars from a map with `range` that always quotes values.

</details>

### Q138. `json: cannot unmarshal string into Go struct field ContainerPort.spec.template.spec.containers.ports.containerPort of type int32` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The port was quoted in values (`port: "3000"`) or in the template (`{{ .Values.port | quote }}`).

**Diagnose →** `helm template ... | yq '.spec.template.spec.containers[].ports'`.

**Fix →** Render as integer: `containerPort: {{ .Values.api.port | int }}`; keep values unquoted.

**Prevent →** Schema with `"type": "integer"` for ports.

</details>

### Q139. Rendered YAML has two keys glued on one line, e.g. `app.kubernetes.io/name: shopapp.kubernetes.io/instance: shop`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Over-aggressive whitespace trimming: `-}}` at the end of a line inside a `define`, or `{{-` on an include without a following `nindent`.

**Diagnose →** Look at the helper's `define` block and the call site.

**Fix →** Inside helpers, trim on `define`/`end` lines only; at call sites use `{{- include "shop.labels" . | nindent 4 }}`.

**Prevent →** Unit tests asserting labels as a map rather than as raw text.

</details>

### Q140. A template renders empty, and Helm silently skips the resource you expected. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The surrounding `if` condition is false (wrong key, `false`/`0` treated as empty), or the file uses `with` on an empty value.

**Diagnose →**
```bash
helm template shop ./shop -f values-prod.yaml -s templates/hpa.yaml
helm get values shop -n shop --all | yq '.api.autoscaling'
```
If `-s` says `could not find template`, the file rendered empty.

**Fix →** Correct the key or condition; use `hasKey` where `0`/`false` are valid.

**Prevent →** Unit tests per feature flag, and `additionalProperties: false` in the schema for feature blocks.

</details>

### Q141. `Error: template: shop/templates/configmap.yaml:6:3: executing "..." at <tpl .Values.app.config $>: error calling tpl: cannot retrieve Template.Basepath from values inside tpl function: ...` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `tpl` was called with the wrong context (a sub-map instead of the root), so templates inside the string cannot access `.Template` or `.Values`.

**Diagnose →** Inspect the `tpl` call: second argument must be the root context (`$` or `.` at top level).

**Fix →** `{{ tpl .Values.app.config $ }}`.

**Prevent →** Wrap `tpl` usage in a helper (`shop.render` taking `dict "value" ... "context" $`) used consistently.

</details>

### Q142. `Error: INSTALLATION FAILED: YAML parse error on shop/templates/secret.yaml: error converting YAML to JSON: yaml: invalid leading UTF-8 octet` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Binary content inserted into YAML, typically `.Files.Get` on a binary file (keystore, image) into `data` without encoding, or a value with invalid encoding.

**Diagnose →** `file files/*` to identify binaries.

**Fix →** Base64-encode binaries: `{{ .Files.Get "files/keystore.jks" | b64enc }}` into a Secret's `data`, or use `.Files.Glob(...).AsSecrets`.

**Prevent →** Keep binaries out of charts where possible; mount them from secret managers.

</details>

### Q143. Every `helm upgrade` restarts pods even when nothing changed. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Non-deterministic pod template: `.Release.Revision` or `now` in annotations, `randAlphaNum` in a Secret that is hashed into a checksum annotation, or `genCA` certificates regenerated.

**Diagnose →**
```bash
helm diff upgrade shop ./shop -n shop -f values-prod.yaml | grep -E '^\+|^-' | head -30
```

**Fix →** Remove time/revision annotations; reuse secrets via `lookup` or externalize them; use cert-manager for certificates.

**Prevent →** A CI check that renders twice and diffs the output; any difference means non-determinism.

</details>

### Q144. `helm template` works, but `helm install` fails on `lookup`-based logic with an unexpected nil. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** During `helm template`, `lookup` returns an empty map, so a different branch runs. During install it returns the real object, and code like `index $secret.data "password"` fails when the key name differs or the object exists but lacks that key.

**Diagnose →** `kubectl get secret <name> -n shop -o yaml` and compare with the template's expectations; use `--dry-run=server` to exercise the live branch.

**Fix →** Guard with `hasKey` before `index`: `{{ if and $secret (hasKey $secret.data "password") }}`.

**Prevent →** Test both branches: `helm template` (empty) and `--dry-run=server` against a cluster with the object.

</details>

### Q145. `Error: template: ... at <semverCompare ">=1.25" .Capabilities.KubeVersion.Version>: error calling semverCompare: ...` or the wrong branch renders on EKS/GKE. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Provider versions like `v1.30.4-eks-a737599` are pre-releases in SemVer; `>=1.25` does not match them, so the "old Kubernetes" branch renders.

**Diagnose →** `kubectl version -o json | jq -r .serverVersion.gitVersion`.

**Fix →** Use `>=1.25-0`.

**Prevent →** Always use the `-0` suffix in version comparisons.

</details>

### Q146. `Error: rendered manifests contain a resource that already exists. Unable to continue with install: ConfigMap "shop-config" in namespace "shop" exists and cannot be imported into the current release ...` but the ConfigMap is defined twice in your own chart. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two templates (or a parent and a subchart) render an object with the same kind and name, so the second conflicts with the first within one install. Often from a hardcoded name in a subchart or a copied template.

**Diagnose →**
```bash
helm template shop ./shop | yq -N '[.kind, .metadata.name] | join("/")' | sort | uniq -d
```

**Fix →** Give each object a unique name via the fullname helper with a component suffix.

**Prevent →** A CI check for duplicate kind/name pairs in rendered output.

</details>

### Q147. `walk.go:74: found symbolic link in path: /charts/shop/files/config resolves to /etc/shop/config. Contents of linked file included and used` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The chart directory contains a symlink, so Helm follows it and packages content from outside the chart, potentially leaking host files.

**Diagnose →** `find ./shop -type l -ls`.

**Fix →** Replace symlinks with real files inside the chart, or exclude them in `.helmignore`.

**Prevent →** Never symlink into charts; CI fails on this warning.

</details>

### Q148. `NOTES.txt` prints a broken command after install, like `kubectl port-forward svc/ 8080:80`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** NOTES uses a helper or value that does not exist in this configuration (for example the frontend disabled, or an old key name).

**Diagnose →** `helm get notes shop -n shop` and compare with `helm get values --all`.

**Fix →** Use the same helpers as the templates (`include "shop.fullname" .`), and branch on enabled components.

**Prevent →** Treat NOTES as a template under test: `helm template` does not print it, but `helm install --dry-run` does, so render it that way in CI for each component combination.

</details>

---

## Values, Schema & Type Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q149. `Error: values don't meet the specifications of the schema(s) in the following chart(s): shop: - api.replicaCount: Invalid type. Expected: integer, given: string` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The value was passed with `--set-string`, quoted in a values file (`"3"`), or injected from an environment variable as a string.

**Diagnose →**
```bash
grep -rn replicaCount values*.yaml
# check the pipeline for --set-string api.replicaCount
```

**Fix →** Pass numbers with `--set` (not `--set-string`) and unquoted in YAML.

**Prevent →** Keep the schema; it caught the bug before it reached the cluster.

</details>

### Q150. `Error: values don't meet the specifications of the schema(s) in the following chart(s): shop: - (root): Additional property replicacount is not allowed` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A typo in a values key; the schema uses `additionalProperties: false`.

**Diagnose →** Compare the key with `helm show values ./shop`.

**Fix →** Correct the key (`replicaCount`).

**Prevent →** This is the schema working as intended; keep `additionalProperties: false` on well-defined objects.

</details>

### Q151. `Error: values don't meet the specifications of the schema(s) in the following chart(s): shop-postgres: - auth: password is required` but you did set `postgres.auth.password`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The subchart was aliased or renamed, so its values key differs (`postgresql` vs `postgres`), or the value was set under the wrong nesting level.

**Diagnose →**
```bash
yq '.dependencies' shop/Chart.yaml        # name and alias
helm install x ./shop --dry-run=client --debug -f values-dev.yaml | sed -n '/^COMPUTED VALUES:/,/^HOOKS:/p' | head -60
```

**Fix →** Put the values under the alias key (or name, if no alias).

**Prevent →** Document subchart keys in the parent's `values.yaml` with commented examples.

</details>

### Q152. `Error: failed parsing --set data: key "b.example.com" has no value` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A comma in a `--set` value: `--set ingress.hosts=a.example.com,b.example.com`. Commas separate assignments.

**Diagnose →** Inspect the command line.

**Fix →** Use list syntax `--set 'ingress.hosts={a.example.com,b.example.com}'`, escape commas (`a\,b`), use `--set-literal`, or a values file.

**Prevent →** Values files for anything beyond simple scalars.

</details>

### Q153. `Error: failed to parse values-prod.yaml: error converting YAML to JSON: yaml: line 12: found character that cannot start any token` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tabs used for indentation, or an unquoted value starting with a reserved character (`@`, `` ` ``, `%`).

**Diagnose →** `grep -nP '\t' values-prod.yaml`; `yq '.' values-prod.yaml`.

**Fix →** Replace tabs with spaces; quote values that start with special characters.

**Prevent →** yamllint in pre-commit and CI.

</details>

### Q154. `coalesce.go:286: warning: cannot overwrite table with non table for shop.api.resources (map[limits:map[memory:256Mi]])` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An override set `api.resources` to a scalar or empty string where the chart default is a map; Helm ignores the override.

**Diagnose →** Find the layer: `yq '.api.resources' values*.yaml`.

**Fix →** Use a map (`resources: {}` to clear) or `null` to delete the default.

**Prevent →** Fail CI on coalesce warnings: `helm template ... 2>&1 >/dev/null | grep -q 'warning:' && exit 1`.

</details>

### Q155. Image tag `1.10` deploys as `1.1` and pods fail with `ImagePullBackOff`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** YAML parsed `tag: 1.10` as a float; `--set api.image.tag=1.10` does the same.

**Diagnose →** `helm get manifest shop -n shop | grep image:`.

**Fix →** Quote in YAML (`tag: "1.10"`), use `--set-string`, and `toString` in the template.

**Prevent →** Schema `"type": "string"` for tags, which rejects unquoted numbers.

</details>

### Q156. A feature turned on in prod although the prod file says `enabled: "false"`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The quoted string `"false"` is non-empty and therefore truthy in `if`.

**Diagnose →** `helm get values shop --all -o json | jq '.feature.enabled | type'` returns `"string"`.

**Fix →** Use an unquoted boolean `enabled: false`.

**Prevent →** Schema `"type": "boolean"`.

</details>

### Q157. After `helm upgrade` with a new values file, settings from the previous deploy disappeared. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** When any `-f` or `--set` is given, Helm does not reuse previous values; the prior deploy used extra `--set` flags or a file not passed this time.

**Diagnose →**
```bash
helm get values shop -n shop --revision <previous>
helm get values shop -n shop
```

**Fix →** Pass all required values files every time; move ad-hoc flags into files.

**Prevent →** Deploy only from CI with values in Git.

</details>

### Q158. `Error: INSTALLATION FAILED: unable to build kubernetes objects from release manifest: error validating "": error validating data: [ValidationError(Deployment.spec.template.spec.containers[0].resources): unknown field "limit" in io.k8s.api.core.v1.ResourceRequirements]` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A typo in a values block passed through `toYaml` (`limit` instead of `limits`).

**Diagnose →** `yq '.api.resources' values*.yaml`.

**Fix →** Correct the key.

**Prevent →** Schema for `resources` (or reference the Kubernetes JSON schema), plus kubeconform `-strict` in CI.

</details>

### Q159. `Error: values don't meet the specifications of the schema(s) in the following chart(s): shop: - global: Additional property global is not allowed` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Root-level `additionalProperties: false` in the parent schema without listing `global` (and subchart keys). Helm always injects `global`, so this fails even when the user did not set it.

**Diagnose →** `jq '.additionalProperties, (.properties | keys)' shop/values.schema.json`.

**Fix →** Add `"global": {"type": "object"}` and each subchart key to `properties`, or drop `additionalProperties: false` at the root.

**Prevent →** Apply strictness to nested objects rather than the root.

</details>

### Q160. `--set podAnnotations.prometheus.io/scrape=true` creates nested keys instead of one annotation. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Dots in keys are path separators in `--set`.

**Diagnose →** `helm get values shop -o yaml` shows `prometheus: {io/scrape: "true"}`.

**Fix →** Escape the dots: `--set-string 'podAnnotations.prometheus\.io/scrape=true'`, or use a values file.

**Prevent →** Annotations and labels belong in values files.

</details>

### Q161. Your prod override added one toleration, and all the chart's default tolerations disappeared. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Lists are replaced, not merged, across values layers.

**Diagnose →** `helm get values shop --all | yq '.api.tolerations'`.

**Fix →** Repeat the defaults you want to keep in your override.

**Prevent →** In charts you own, model collections as maps keyed by name and convert to lists in templates.

</details>

### Q162. `Error: open secrets/prod.yaml: no such file or directory` in CI only. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Relative paths resolved from a different working directory in CI, a sparse checkout, or the file is in `.gitignore`.

**Diagnose →** `pwd; ls -la secrets/` in the job.

**Fix →** Use paths relative to the repo root with an explicit `working-directory`, or absolute paths from `$GITHUB_WORKSPACE`.

**Prevent →** A deploy script that resolves its own directory (`cd "$(dirname "$0")"`).

</details>

### Q163. `Error: values don't meet the specifications of the schema(s) ... - api.image.tag: Invalid type. Expected: string, given: null` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `tag:` left empty in YAML means `null`; or `--set api.image.tag=null`.

**Diagnose →** Check values files for empty keys.

**Fix →** Set `tag: ""` if empty is allowed and the template defaults to `.Chart.AppVersion`, or provide a real tag.

**Prevent →** Schema `"type": ["string", "null"]` if null should be valid, and a template default.

</details>

---

## Dependencies, Repositories & OCI

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q164. ``Error: An error occurred while checking for chart dependencies. You may need to run `helm dependency build` to fetch missing dependencies: found in Chart.yaml, but missing in charts/ directory: shop-postgres`` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `charts/` is empty because `.tgz` files are not committed (correct) and the pipeline did not fetch dependencies.

**Diagnose →** `helm dependency list ./shop` shows `missing`.

**Fix →** `helm dependency build ./shop` before `template`/`install`/`package`.

**Prevent →** Add the build step to CI and to the Makefile; `helm package --dependency-update` for packaging.

</details>

### Q165. `Error: no repository definition for https://charts.example.com/stable. Please add the missing repos via 'helm repo add'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A dependency points to an HTTP repo not configured on this machine or runner.

**Diagnose →** `helm repo list`.

**Fix →** `helm repo add example https://charts.example.com/stable && helm repo update`, then `helm dependency build`.

**Prevent →** Script repo setup in CI from `Chart.yaml`, or switch dependencies to OCI.

</details>

### Q166. `Error: the lock file (Chart.lock) is out of sync with the dependencies file (Chart.yaml). Please update the dependencies` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Someone edited `dependencies` in `Chart.yaml` (version, repository, alias) without regenerating the lock.

**Diagnose →** `git diff Chart.yaml Chart.lock`.

**Fix →** `helm dependency update ./shop` and commit the new `Chart.lock`.

**Prevent →** A CI check that runs `helm dependency update` and fails if `Chart.lock` changes.

</details>

### Q167. `Error: looks like "https://charts.example.com" is not a valid chart repository or cannot be reached: failed to fetch https://charts.example.com/index.yaml : 404 Not Found` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong URL (missing path), the repo moved to OCI, the index was never generated, or authentication returns 404 for private content.

**Diagnose →** `curl -sI https://charts.example.com/index.yaml`.

**Fix →** Correct the URL; if the project moved to OCI, use `oci://` references instead of `helm repo add`; pass credentials for private repos.

**Prevent →** Track upstream announcements; mirror important charts internally.

</details>

### Q168. `Error: chart "shop" version "1.4.2" not found in https://charts.example.com repository` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Stale local index cache, the version was never published, or it is a pre-release that needs `--devel`.

**Diagnose →**
```bash
helm repo update example
helm search repo example/shop --versions | head
```

**Fix →** Run `helm repo update`, use an existing version, or add `--devel` for pre-releases.

**Prevent →** `helm repo update` in CI before installs; publish before referencing.

</details>

### Q169. `Error: failed to authorize: failed to fetch oauth token: unexpected status from GET request to https://ghcr.io/token?...: 401 Unauthorized` on `helm pull oci://ghcr.io/...` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Not logged in, expired token, token lacking `read:packages`, or the package is private and the token's user has no access.

**Diagnose →** `cat "$(helm env HELM_REGISTRY_CONFIG)" | jq '.auths | keys'`.

**Fix →** `echo $TOKEN | helm registry login ghcr.io -u <user> --password-stdin`, with a token that has package read rights; grant the repo access to the package.

**Prevent →** In GitHub Actions use `GITHUB_TOKEN` with `packages: read` and link the package to the repository.

</details>

### Q170. `helm push` to ECR fails with `name unknown: The repository with name 'charts/shop' does not exist in the registry with id '123456789012'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ECR requires the repository to exist; it is not created on push. The repository name is the push path plus the chart name.

**Diagnose →** `aws ecr describe-repositories --repository-names charts/shop`.

**Fix →** `aws ecr create-repository --repository-name charts/shop --image-tag-mutability IMMUTABLE`, then push to `oci://$REG/charts`.

**Prevent →** Create chart repositories with Terraform alongside image repositories.

</details>

### Q171. `Error: unexpected status from HEAD request to https://123456789012.dkr.ecr.ap-south-1.amazonaws.com/v2/charts/shop/manifests/1.4.0: 403 Forbidden` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ECR login token expired (valid 12 hours), wrong region in the login, or IAM lacks `ecr:BatchGetImage`/`ecr:GetDownloadUrlForLayer` (pull) or push permissions.

**Diagnose →**
```bash
aws sts get-caller-identity
aws ecr get-login-password --region ap-south-1 >/dev/null && echo ok
```

**Fix →** Re-login with the correct region; fix the IAM policy or repository policy (for cross-account pulls).

**Prevent →** Login as part of each pipeline job; Terraform-managed policies.

</details>

### Q172. `Error: invalid_reference: invalid tag` when pushing or pulling a chart with version `1.4.0+build.7`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** OCI tags cannot contain `+`. Helm converts `+` to `_` on push, so the tag in the registry is `1.4.0_build.7`; tools or scripts using the raw version fail.

**Diagnose →** `aws ecr list-images --repository-name charts/shop` or `crane ls`.

**Fix →** Reference the version with `+` in `helm pull --version` (Helm translates), or avoid build metadata in chart versions; use pre-release (`1.4.0-build.7`) or a plain patch bump.

**Prevent →** CI derives chart versions without `+`.

</details>

### Q173. `Error: failed to download "oci://ghcr.io/devopsshack/charts/shop" at version "1.4.0"` with an `http: server gave HTTP response to HTTPS client` error, using a local registry. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The local registry (`localhost:5001`, kind registry) speaks plain HTTP, and Helm defaults to HTTPS.

**Diagnose →** `curl -s http://localhost:5001/v2/_catalog`.

**Fix →** Add `--plain-http` (Helm 3.13+) to `push`, `pull`, `install`, `template`, `show`; for self-signed TLS use `--insecure-skip-tls-verify` or `--ca-file`.

**Prevent →** Use TLS in shared registries; reserve `--plain-http` for local labs.

</details>

### Q174. `Error: non-absolute URLs should be in form of repo_name/path_to_chart, got: postgresql` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You ran `helm install pg postgresql` expecting Helm to find it; Helm needs `repo/chart`, a path, a URL or an `oci://` reference.

**Diagnose →** `helm search repo postgresql`.

**Fix →** `helm install pg bitnami/postgresql` after adding the repo, or the OCI reference `oci://registry-1.docker.io/bitnamicharts/postgresql`.

**Prevent →** Always use fully qualified chart references in scripts.

</details>

### Q175. `Error: INSTALLATION FAILED: path "./shop" not found` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong working directory, or a local directory name collides with what you meant as a repo reference (or the reverse: a directory named like `bitnami/postgresql` shadows the repo).

**Diagnose →** `pwd; ls`.

**Fix →** Use the correct path or reference.

**Prevent →** Absolute paths in scripts.

</details>

### Q176. `helm dependency update` hangs or fails with `x509: certificate signed by unknown authority` behind a corporate proxy. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** TLS interception by the proxy with a corporate CA that Helm (Go) does not trust, or proxy variables not set.

**Diagnose →** `env | grep -i proxy`; `curl -v https://charts.example.com/index.yaml`.

**Fix →** Add the corporate CA to the OS trust store (Go uses it), set `HTTPS_PROXY`/`NO_PROXY`, or pass `--ca-file` when adding the repo.

**Prevent →** Base CI images with the corporate CA installed; internal chart mirror.

</details>

### Q177. Installing an OCI chart without `--version` deployed an unexpected pre-release. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Without `--version`, Helm resolves the highest SemVer tag in the registry; tags that are not stable releases can be picked, depending on tag naming.

**Diagnose →** `helm list -n shop` shows the chart version; check registry tags.

**Fix →** Pin `--version` (or a digest) and roll back.

**Prevent →** Always pin versions; publish pre-releases to a separate repository path.

</details>

### Q178. `helm dependency build` downloads a different subchart version than a colleague got. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `Chart.lock` is not committed (or is ignored), so `build` falls back to resolving ranges.

**Diagnose →** `git ls-files shop/Chart.lock`.

**Fix →** Commit `Chart.lock`; use `build`, not `update`, in CI.

**Prevent →** CI check that the lock exists and is in sync.

</details>

### Q179. `Error: Chart.yaml file is missing` when installing from a directory that clearly has charts. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You pointed at the parent directory (`./charts`) instead of the chart directory (`./charts/shop`), or the file is named `chart.yaml` (case sensitive).

**Diagnose →** `ls ./charts/shop/Chart.yaml`.

**Fix →** Use the chart directory path; rename the file to `Chart.yaml`.

**Prevent →** `helm lint` in CI for each chart directory.

</details>

### Q180. `Error: validation: chart.metadata.version "v1.4" is invalid` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Chart `version` must be strict SemVer 2 (`1.4.0`), not `v1.4` or `1.4`.

**Diagnose →** `yq '.version' shop/Chart.yaml`.

**Fix →** `version: 1.4.0`, or pass `helm package --version 1.4.0`.

**Prevent →** CI derives chart versions from Git tags with a SemVer check.

</details>

---

## Hooks, Tests, GitOps & Secrets

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q181. `Error: INSTALLATION FAILED: failed post-install: job failed: BackoffLimitExceeded` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The hook Job's container exited non-zero more times than `backoffLimit`: migration SQL error, DB not reachable, wrong credentials, missing ConfigMap.

**Diagnose →**
```bash
kubectl get jobs -n shop
kubectl logs -n shop job/shop-migrate --all-containers
kubectl describe job shop-migrate -n shop
```

**Fix →** Fix the script or connectivity, then rerun the upgrade (the Job is recreated thanks to `before-hook-creation`).

**Prevent →** Keep failed hook Jobs (no `hook-failed` delete policy), init container waiting for the DB, idempotent migrations.

</details>

### Q182. `Error: UPGRADE FAILED: pre-upgrade hooks failed: timed out waiting for the condition` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The hook Job is still running (long migration), stuck waiting for the DB, cannot schedule (resources, node selectors), or its pod never finishes because of a sidecar.

**Diagnose →**
```bash
kubectl get pods -n shop -l job-name=shop-migrate -o wide
kubectl describe pod -n shop -l job-name=shop-migrate | sed -n '/Events/,$p'
kubectl logs -n shop -l job-name=shop-migrate -c migrate
```

**Fix →** Address the cause; increase `--timeout` if the migration is legitimately slow; disable sidecar injection or use native sidecars for hook pods.

**Prevent →** `activeDeadlineSeconds` on hook Jobs; keep migrations fast.

</details>

### Q183. `Error: UPGRADE FAILED: pre-upgrade hooks failed: warning: Hook pre-upgrade shop/templates/migrate-job.yaml failed: jobs.batch "shop-migrate" already exists` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A previous hook Job was not deleted (no `before-hook-creation` policy, or a policy with only `hook-succeeded` and the last run failed).

**Diagnose →** `kubectl get job shop-migrate -n shop`.

**Fix →** `kubectl delete job shop-migrate -n shop` and rerun; add `"helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded`.

**Prevent →** Always include `before-hook-creation` on hook Jobs and test Pods.

</details>

### Q184. The migration hook fails with `psql: error: connection to server at "shop-postgres" (10.96.12.4), port 5432 failed: Connection refused` on first install. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The hook runs as `pre-install` (DB does not exist yet), or as `post-install` without `--wait` and before Postgres is ready.

**Diagnose →** `helm get hooks shop -n shop | grep 'helm.sh/hook:'`; `kubectl get pods -n shop` timestamps.

**Fix →** Use `post-install,pre-upgrade` and add an init container that loops on `pg_isready -h shop-postgres -p 5432` before migrating.

**Prevent →** Treat DB readiness as an explicit dependency in hooks.

</details>

### Q185. On upgrade, the migration Job runs the **old** SQL even though the chart contains new migration files. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The SQL lives in a regular ConfigMap, which is updated only after `pre-upgrade` hooks run, so the Job mounts the previous version.

**Diagnose →** `kubectl get cm shop-migrations -n shop -o yaml` during the hook shows old content.

**Fix →** Make the migrations ConfigMap a hook too (`pre-upgrade`, weight `-5`, `before-hook-creation`) so it is created before the Job (weight `0`), or bake migrations into the migration image.

**Prevent →** Remember that `pre-*` hooks see the old state of regular resources.

</details>

### Q186. `helm test` fails with `Error: unable to get pod logs for shop-test-api: pods "shop-test-api" not found`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The test Pod was deleted before Helm fetched logs (`hook-delete-policy: hook-succeeded` or `hook-failed`) while using `--logs`.

**Diagnose →** Check the test template annotations.

**Fix →** Use `before-hook-creation` only for test Pods so they remain until the next run.

**Prevent →** Standardize test Pod annotations in the library chart.

</details>

### Q187. `helm test` hangs until timeout. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The test Pod has `restartPolicy: Always` (default for Pods is Always), so it never reaches `Succeeded`; or it is stuck in `Pending`/`ImagePullBackOff`.

**Diagnose →** `kubectl get pod shop-test-api -n shop -o jsonpath='{.spec.restartPolicy} {.status.phase}'`.

**Fix →** Set `restartPolicy: Never` in test Pods; fix the image or scheduling issue.

**Prevent →** Template test Pods from a shared helper with correct defaults.

</details>

### Q188. Argo CD Application shows `ComparisonError: ... failed to execute helm template command: ... Error: execution error at (shop/templates/secret.yaml:9:15): postgres.auth.password is required` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Argo CD renders without your secrets file: helm-secrets is not configured in the repo server, or the values file path is wrong in `valueFiles`.

**Diagnose →** `argocd app manifests shop-prod --source live` and the Application's `spec.source.helm.valueFiles`.

**Fix →** Use `existingSecret` with an ExternalSecret (preferred), or configure the helm-secrets integration in the Argo CD repo server (SOPS keys provided via IRSA/KMS) and reference `secrets://` value files.

**Prevent →** Keep secrets out of Helm values in GitOps setups.

</details>

### Q189. Argo CD shows the app `OutOfSync` forever because a Secret's data changes every refresh. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The chart generates the secret with `randAlphaNum` (or `genCA`) and relies on `lookup` to keep it, which does not work in Argo CD.

**Diagnose →** `argocd app diff shop-prod` shows only that Secret's data changing.

**Fix →** Create the Secret outside the chart and use `existingSecret`; temporarily `ignoreDifferences` on `/data` for that Secret.

**Prevent →** Deterministic charts for GitOps.

</details>

### Q190. A `pre-install` hook in a third-party chart runs on every Argo CD sync and resets data. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Argo CD maps both `pre-install` and `pre-upgrade` to `PreSync`, so install-only hooks run on each sync.

**Diagnose →** `kubectl get jobs -n <ns> --sort-by=.metadata.creationTimestamp` shows a new Job per sync.

**Fix →** Override with Argo CD annotations through a post-render step (Kustomize patch adding `argocd.argoproj.io/hook: Skip` or a different hook), disable the hook via chart values if offered, or make the job idempotent.

**Prevent →** Review chart hooks before adopting third-party charts in Argo CD.

</details>

### Q191. Argo CD fails with `Error: failed to fetch chart: ... oci://... : not found` while `helm pull` works locally. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Argo CD expects OCI Helm `repoURL` without the `oci://` scheme (`123456789012.dkr.ecr.ap-south-1.amazonaws.com/charts`) and the repo credential registered with `enableOCI: true`; or ECR tokens expired in the repo secret.

**Diagnose →** `argocd repo list`; inspect the Application `source`.

**Fix →** Correct `repoURL` and `chart`; register the repo as OCI; refresh ECR credentials automatically (a CronJob or an ECR credential helper, or IRSA-based integration).

**Prevent →** Manage Argo CD repo secrets declaratively with automated token refresh.

</details>

### Q192. `Error: plugin "secrets" exited with error` and `Failed to get the data key required to decrypt the SOPS file.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The deploy identity cannot decrypt: no age key in `SOPS_AGE_KEY_FILE`/`SOPS_AGE_KEY`, or no `kms:Decrypt` on the KMS key, or the file was encrypted for different recipients.

**Diagnose →**
```bash
sops -d secrets/prod.yaml >/dev/null
yq '.sops.kms[].arn, .sops.age[].recipient' secrets/prod.yaml
aws sts get-caller-identity
```

**Fix →** Provide the right key or grant KMS decrypt to the CI role; re-encrypt for the correct recipients with `sops updatekeys secrets/prod.yaml`.

**Prevent →** `.sops.yaml` creation rules per environment and a CI preflight `sops -d` check.

</details>

### Q193. `Error: unknown command "secrets" for "helm"` (or `"diff"`) in CI. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Plugins are installed per user under `HELM_PLUGINS`; the CI image or runner does not have them, or `HELM_DATA_HOME` points elsewhere.

**Diagnose →** `helm plugin list; helm env HELM_PLUGINS`.

**Fix →** Install in the job: `helm plugin install https://github.com/jkroepke/helm-secrets --version v4.6.2` (pin the version), or bake plugins into the CI image.

**Prevent →** A pinned CI image with Helm, plugins, sops and kubeconform.

</details>

### Q194. helm-secrets leaves a decrypted `secrets.prod.yaml.dec` file in the workspace after a failed run. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrapper mode writes temporary decrypted files; a crash or kill skips cleanup.

**Diagnose →** `find . -name '*.dec'`.

**Fix →** Delete them; switch to getter mode (`-f secrets://...`), which streams decrypted content, and add `*.dec` to `.gitignore`.

**Prevent →** Ephemeral CI workspaces and a gitleaks pre-commit hook.

</details>

### Q195. Helmfile fails with `in ./helmfile.yaml: failed to read helmfile.yaml: reading document at index 1: yaml: line 14: did not find expected node content` after upgrading to Helmfile 1.x. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file contains Go template expressions (`{{ .Environment.Name }}`), and Helmfile 1.x only renders templates in files with the `.gotmpl` extension, so the raw braces reach the YAML parser.

**Diagnose →** `grep -n '{{' helmfile.yaml`.

**Fix →** Rename to `helmfile.yaml.gotmpl` (and templated values files to `*.yaml.gotmpl`), or remove templating.

**Prevent →** Read the major-version migration notes; pin the Helmfile version in CI.

</details>

### Q196. `helmfile apply` reports changes on every run even though nothing changed. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Non-deterministic templates (random values, timestamps), or secrets showing as changed because helm-diff cannot compare encrypted data consistently.

**Diagnose →** `helmfile -e prod diff --context 3` and inspect which fields differ.

**Fix →** Make templates deterministic; use `--suppress-secrets` for display; externalize generated secrets.

**Prevent →** The same render-twice determinism check used for Argo CD.

</details>

### Q197. After rollback, the app still reads the new config. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The config comes from a source Helm does not roll back (an ExternalSecret synced from a secret manager, a ConfigMap created by a hook, or a resource with `resource-policy: keep`), or pods were not restarted because the pod template did not change.

**Diagnose →**
```bash
helm get manifest shop -n shop | grep -A5 'kind: ConfigMap'
kubectl get cm,secret -n shop -o custom-columns=NAME:.metadata.name,MANAGED:.metadata.labels.app\\.kubernetes\\.io/managed-by
```

**Fix →** Revert the external config at its source, and restart pods (`kubectl rollout restart deploy/shop-api -n shop`) or rely on checksum annotations.

**Prevent →** Checksum annotations on chart-managed config; versioned external config.

</details>

### Q198. `helm uninstall` completed, but PVCs, a hook ConfigMap and a CRD are still in the cluster. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Expected behavior: PVCs created by StatefulSet `volumeClaimTemplates` are not part of the release; hook resources without a delete policy are not tracked; CRDs from `crds/` are never deleted; `resource-policy: keep` objects are retained.

**Diagnose →** `kubectl get pvc,cm,secret -n shop`; `kubectl get crd | grep shop`.

**Fix →** Delete leftovers deliberately (`kubectl delete pvc -n shop -l app.kubernetes.io/instance=shop`), after confirming data is not needed.

**Prevent →** Document what uninstall leaves behind; delete policies on hooks.

</details>

### Q199. `Error: uninstallation completed with 1 error(s): uninstall: Failed to purge the release: release: not found` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Release Secrets were partly deleted manually or by another process during uninstall; Helm deleted resources but could not find the record to purge.

**Diagnose →** `kubectl get secret -n shop -l owner=helm,name=shop`; `helm list -n shop --all`.

**Fix →** Usually nothing left to do; clean up any remaining revision Secrets for that release and leftover resources.

**Prevent →** Avoid concurrent manual operations on release Secrets.

</details>

### Q200. Pods fail with `CreateContainerConfigError: secret "shop-db" not found` after moving secrets out of the chart into External Secrets. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ExternalSecret is not yet synced (SecretStore auth failure, wrong remote key), or its `target.name` differs from the name the Deployment references.

**Diagnose →**
```bash
kubectl get externalsecret -n shop
kubectl describe externalsecret shop-db -n shop | sed -n '/Status/,$p'
kubectl get secretstore,clustersecretstore -A
```

**Fix →** Fix the store credentials (IRSA/Pod Identity role), the remote key path, or align `target.name` with the chart's `existingSecret` value.

**Prevent →** Deploy ExternalSecrets with a sync wave or hook before workloads, and alert on `SecretSyncedError`.

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
