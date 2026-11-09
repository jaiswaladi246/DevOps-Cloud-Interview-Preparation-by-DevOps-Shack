# GitOps & ArgoCD — Troubleshooting Drills

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

> Real GitOps & ArgoCD errors and symptoms: likely causes, diagnosis, fix and prevention · 100 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to GitOps & ArgoCD](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [OutOfSync Loops & Diff Problems](#outofsync-loops--diff-problems) — Q1–Q20
- [Manifest Generation & ComparisonError](#manifest-generation--comparisonerror) — Q21–Q38
- [Repos, Clusters, Projects & Permissions](#repos-clusters-projects--permissions) — Q39–Q60
- [Sync Operations, Hooks & Health](#sync-operations-hooks--health) — Q61–Q74
- [ApplicationSets, Rollouts & Image Updater](#applicationsets-rollouts--image-updater) — Q75–Q88
- [Secrets, Controller Performance & Recovery](#secrets-controller-performance--recovery) — Q89–Q100

---

## OutOfSync Loops & Diff Problems

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. An app flips between `Synced` and `OutOfSync` every few seconds and the sync history grows constantly. 🟡

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

### Q2. `argocd app diff` shows only `replicas: 2` vs `replicas: 5` on a Deployment that has an HPA, and the app never stays Synced. 🟢

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

### Q3. A MutatingWebhookConfiguration is always OutOfSync on `webhooks[0].clientConfig.caBundle`. 🟡

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

### Q4. After installing Istio, every Deployment shows OutOfSync because of injected sidecar fields. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Usually it is not the Deployment but Pods or objects you declared that Istio mutates (for example annotations Istio adds to Services or webhook configs), or label tracking colliding with Istio labels. Pod injection happens on Pods, which Argo CD does not diff against Git.

**Diagnose →** `argocd app diff` to find the exact resource and field; check whether you declare objects Istio controls (for example `istio-injection` labels on namespaces managed by two places).

**Fix →** Ignore the injected fields with `jqPathExpressions` or `managedFieldsManagers: [pilot-discovery]`, or enable `ServerSideDiff=true` so webhook mutations are part of the expected state.

**Prevent →** Server-side diff for meshes and policy engines that mutate on admission.

</details>

### Q5. A ConfigMap is OutOfSync and the diff shows the same data in a different format (a number became a string). 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** YAML typing: `data` values in a ConfigMap must be strings; `port: 8080` unquoted is coerced, or a Helm template outputs a number.

**Diagnose →** `argocd app diff web` shows `port: 8080` vs `port: "8080"`; `helm template` output confirms the unquoted value.

**Fix →** Quote values (`{{ .Values.port | quote }}`) or write `"8080"` in plain YAML.

**Prevent →** Lint rendered manifests with kubeconform in CI; it rejects non-string ConfigMap data.

</details>

### Q6. A Deployment is OutOfSync on `resources.limits.cpu`: Git says `1000m`, live says `1`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The API server normalizes quantities. Argo CD normalizes most cases, but some combinations (especially inside CRDs, which are not normalized by Argo CD's known-type logic) remain.

**Diagnose →** `argocd app diff` shows the quantity formats; check whether the object is a built-in kind or a CRD.

**Fix →** Write canonical values (`cpu: "1"`, `memory: 512Mi`) or enable `ServerSideDiff=true`.

**Prevent →** Standardize quantity formats in chart defaults.

</details>

### Q7. An app is OutOfSync on a field that is not in Git at all, after an Argo CD upgrade. 🔴

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

### Q8. The app shows `SharedResourceWarning: Namespace/payments is part of applications argocd/payments-api and payments-worker`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both apps render the same Namespace (or ConfigMap, ClusterRole).

**Diagnose →** `argocd app manifests payments-api | grep -A3 'kind: Namespace'` and the same for the other app.

**Fix →** Keep the resource in exactly one app (or use `CreateNamespace=true` and `managedNamespaceMetadata` in one app only) and remove it from the other.

**Prevent →** `FailOnSharedResource=true` in sync options turns the warning into a failure.

</details>

### Q9. A Secret generated by a Helm chart (`genCA`/`randAlphaNum`) shows OutOfSync after every refresh. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Argo CD runs `helm template` on each render, so random or generated values change each time, and `lookup` (which charts use to reuse existing values) returns nothing.

**Diagnose →** `argocd app diff` shows the Secret's data changing; the chart template contains `randAlphaNum`, `genCA` or `lookup`.

**Fix →** Supply the value via `existingSecret` (Sealed Secrets or ESO), or set the value explicitly in values. Short-term: `ignoreDifferences` on `/data` for that Secret plus `RespectIgnoreDifferences=true`.

**Prevent →** Prefer charts that support `existingSecret`; check for `lookup` before adopting a chart.

</details>

### Q10. An app stays OutOfSync showing a resource marked "requires pruning", but auto-sync is on. 🟢

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

### Q11. After switching to annotation tracking, the UI shows every resource twice or many as "extraneous". 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Resources still have the old `app.kubernetes.io/instance` label and do not yet have the `argocd.argoproj.io/tracking-id` annotation; or operator-created objects copied the label and now look orphaned or extraneous.

**Diagnose →** `kubectl get <kind> <name> -o yaml | grep -E 'instance|tracking-id'`; `argocd app get <app>` for extraneous resources.

**Fix →** Sync each app once so the tracking annotation is written. Consider `annotation+label` during the transition. Mark operator-generated objects with `IgnoreExtraneous`.

**Prevent →** Plan the tracking change as a migration: staged, with syncs per app.

</details>

### Q12. An `ExternalSecret` is always OutOfSync with fields like `conversionStrategy: Default` and `decodingStrategy: None` appearing in live. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ESO's CRD defaults these fields; your manifest omits them, and CRD defaults are not normalized by client-side diff.

**Diagnose →** `argocd app diff` lists exactly those defaulted fields under `remoteRef`.

**Fix →** Add the default values explicitly in Git, or enable `ServerSideDiff=true` for the app.

**Prevent →** Use server-side diff for apps heavy in CRDs with schema defaults.

</details>

### Q13. A CRD is OutOfSync on `status` or `metadata.annotations` fields that its controller sets. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You included `status` in the manifest (copied from `kubectl get -o yaml`), or the controller writes annotations you also declare.

**Diagnose →** `argocd app diff` shows `status:` or controller annotations.

**Fix →** Remove `status` and server-populated fields from Git (`kubectl neat` style cleanup); ignore controller-owned annotations via jq paths.

**Prevent →** Never commit raw `kubectl get -o yaml` output; strip `status`, `uid`, `resourceVersion`, `managedFields`, `creationTimestamp`.

</details>

### Q14. `ignoreDifferences` is configured for `/spec/replicas`, the diff is clean, but replicas still get reset on every sync. 🟡

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

### Q15. Your `ignoreDifferences` rule with a jq expression has no effect; the field is still shown in the diff. 🟡

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

### Q16. An app is OutOfSync because the live object has an empty field that Git omits, for example `securityContext: {}` or `env: []`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Usually the reverse: Git declares an empty value (`env: []`, `annotations: {}`) that the API server drops, so the diff sees a declared field missing in live.

**Diagnose →** `argocd app diff` shows the empty structure; check the rendered manifest.

**Fix →** Remove empty maps and lists from templates (Helm: `{{- with .Values.env }}` blocks).

**Prevent →** Template patterns that omit empty blocks.

</details>

### Q17. A Service of type `LoadBalancer` shows OutOfSync on `spec.ports[0].nodePort` and `healthCheckNodePort`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You declared `nodePort` values that differ from what was allocated, or a previous manifest declared them (last-applied still lists them).

**Diagnose →** Compare `argocd app diff` with `kubectl get svc -o yaml`.

**Fix →** Remove `nodePort` from Git unless you need fixed ports; resync to refresh last-applied.

**Prevent →** Let Kubernetes allocate node ports.

</details>

### Q18. A self-heal loop makes the application controller CPU spike and other apps lag. 🔴

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

### Q19. An app is OutOfSync because the Helm chart renders `apiVersion: policy/v1beta1` for PodDisruptionBudget, and live shows `policy/v1`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Old chart using a removed API, or `.Capabilities.APIVersions` not seeing `policy/v1` so the chart picks the old one; on 1.25+ the old API no longer exists and apply fails.

**Diagnose →** `argocd app manifests web | grep -B2 PodDisruptionBudget`; check the chart's capability conditions.

**Fix →** Upgrade the chart; if needed set `kubeVersion` or `apiVersions` under `spec.source.helm` so the chart renders correctly.

**Prevent →** Run `pluto` or kubeconform with the target Kubernetes version in CI.

</details>

### Q20. An app shows `OutOfSync` with the message `ComparisonError: ... unable to resolve 'main' to a commit SHA`. 🟢

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

## Manifest Generation & ComparisonError

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q21. `ComparisonError: Failed to load target state: failed to generate manifest for source 1 of 1: rpc error: code = Unknown desc = Manifest generation error (cached): ...` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Rendering failed (Helm or Kustomize error) and the repo-server cached the failure; it returns the cached error until the cache expires or something changes.

**Diagnose →** Read the rest of the message for the real error; reproduce locally with `helm template` or `kustomize build` at the same commit; check `kubectl -n argocd logs deploy/argocd-repo-server`.

**Fix →** Fix the manifests and push, then `argocd app get web --hard-refresh` to bypass the cached error.

**Prevent →** Render every app in CI on PRs to the config repo.

</details>

### Q22. `rpc error: code = Unknown desc = `helm template . --name-template web --namespace web ... --include-crds` failed exit status 1: Error: template: web/templates/deployment.yaml:24:20: executing "web/templates/deployment.yaml" at <.Values.image.tag>: nil pointer evaluating interface {}.tag` 🟢

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

### Q23. `Error: open envs/prod/values.yaml: no such file or directory` when the app renders. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `valueFiles` paths are relative to the chart `path`, not the repo root.

**Diagnose →** Compare `spec.source.path` (for example `charts/web`) with the value file location.

**Fix →** Use `../../envs/prod/values.yaml`, or with multiple sources reference `$values/envs/prod/values.yaml`. Use `ignoreMissingValueFiles: true` only for optional files.

**Prevent →** Render in CI with exactly the same relative paths Argo CD uses.

</details>

### Q24. `Error: values file '../../envs/prod/values.yaml' resolved to outside repository root` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The relative path climbs out of the repository (too many `../`), or points at a file outside the repo; Argo CD blocks this for security.

**Diagnose →** Count directories from the chart path to the repo root.

**Fix →** Correct the relative path. If values live in another repo, use multiple sources with `ref: values` and `$values/...`.

**Prevent →** Keep a consistent depth for chart folders or use multi-source references.

</details>

### Q25. Kustomize app fails: `kustomize build ... failed exit status 1: Error: accumulating resources: accumulation err='accumulating resources from '../../base': ... must build at directory: not a valid directory'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong relative path in `resources`, a folder renamed, or a remote base unreachable.

**Diagnose →** `kustomize build apps/web/overlays/prod` locally at the same commit.

**Fix →** Fix the path in `kustomization.yaml`. For remote bases, pin a ref and ensure the repo-server can reach and authenticate to it.

**Prevent →** `kustomize build` every overlay in CI.

</details>

### Q26. `Error: trouble configuring builtin HelmChartInflationGenerator with config: ... must specify --enable-helm` 🟡

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

### Q27. `rpc error: code = DeadlineExceeded desc = context deadline exceeded` on several apps' comparison. 🔴

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

### Q28. `rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp 10.96.12.4:8081: connect: connection refused"` 🟡

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

### Q29. Helm dependency error: `Error: no cached repository for helm-manager-... found. (try 'helm repo update')` or `Error: found in Chart.yaml, but missing in charts/ directory: postgresql`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An umbrella chart's dependency repository is not registered or reachable from Argo CD, or credentials for a private dependency repo are missing.

**Diagnose →** Check `Chart.yaml` dependencies and `argocd repo list`; try `helm dependency build` locally.

**Fix →** Register the dependency repository in Argo CD (`argocd repo add <url> --type helm --name <name>`), add credentials, or vendor the dependency tarballs into `charts/` with `Chart.lock`.

**Prevent →** Vendor dependencies for production charts so renders do not depend on external repos.

</details>

### Q30. `failed to fetch chart: failed to download chart: unexpected status code 401` for an OCI Helm chart. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing or wrong registry credentials; for ECR, an expired token (12 hours); OCI not enabled on the repo Secret.

**Diagnose →** `argocd repo list`; check the repo Secret for `type: helm` and `enableOCI: "true"`; `helm pull oci://<registry>/<chart> --version <v>` with the same creds.

**Fix →** Update credentials; for ECR, refresh with a CronJob or an ESO `ECRAuthorizationToken` generator writing the repo Secret.

**Prevent →** Use short-lived credential automation for cloud registries.

</details>

### Q31. The repo-server logs `Error: unknown command "kustomize" for "argocd"` or a plugin app shows `couldn't find cmp-server plugin`. 🔴

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

### Q32. App fails with `Unable to generate manifests in apps/web: rpc error: code = Unknown desc = apps/web: app path does not exist`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong `path`, a folder renamed in Git, or `targetRevision` points at a branch where the path does not exist yet.

**Diagnose →** `git ls-tree -d <revision> apps/` in the config repo; `argocd app get web -o yaml | yq '.spec.source'`.

**Fix →** Correct the path or revision.

**Prevent →** Generate Applications with a git directory generator so paths always exist.

</details>

### Q33. `Kustomize build failed: ... json: unknown field "patchesStrategicMerge"` warnings, or a deprecated-field error after an Argo CD upgrade. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Kustomize version bundled with the new Argo CD deprecates or changes fields (`patchesStrategicMerge`, `commonLabels`, `bases`).

**Diagnose →** `kubectl -n argocd exec deploy/argocd-repo-server -- kustomize version`; run the same version locally.

**Fix →** Run `kustomize edit fix` to migrate files; or pin an older Kustomize version via `kustomize.path.<version>` in `argocd-cm` and `spec.source.kustomize.version` temporarily.

**Prevent →** Track the bundled tool versions in Argo CD release notes.

</details>

### Q34. A Jsonnet or directory app renders nothing and the app is Synced with zero resources, though files exist. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Files are in subfolders and `directory.recurse` is false, or `include`/`exclude` globs filter them out, or the files have extensions Argo CD ignores.

**Diagnose →** `argocd app manifests web` returns empty; check `spec.source.directory`.

**Fix →** Set `directory: {recurse: true}` or fix globs; use `.yaml`, `.yml`, `.json` or `.jsonnet` extensions.

**Prevent →** Keep `allowEmpty: false` on automated sync so an empty render cannot prune everything.

</details>

### Q35. `ComparisonError: ... error unmarshaling JSON: while decoding JSON: json: cannot unmarshal string into Go value of type map[string]interface {}` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A rendered file contains something that is not a Kubernetes object: a stray YAML document with only a string, a values file inside the manifests path, a README in YAML, or broken indentation.

**Diagnose →** `argocd app manifests web --source git` or local render; look for documents without `apiVersion`/`kind`.

**Fix →** Remove non-manifest YAML from the path, or exclude it with `directory.exclude`, or fix the template output.

**Prevent →** kubeconform in CI rejects non-object documents.

</details>

### Q36. App rendering succeeds locally but fails in Argo CD with `Error: chart requires kubeVersion: >=1.29.0-0 which is incompatible with Kubernetes v1.20.0`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Helm's default kubeVersion is used, because Argo CD does not have the destination cluster's version (cluster info not yet loaded, cluster unreachable), or an explicit wrong `kubeVersion` override.

**Diagnose →** `argocd cluster get <cluster>` for server version and connection status.

**Fix →** Fix cluster connectivity so cluster info is populated; or set `spec.source.helm.kubeVersion` explicitly.

**Prevent →** Monitor cluster connection status; avoid hardcoding kubeVersion.

</details>

### Q37. `Unable to create application: application spec for web is invalid: InvalidSpecError: Unable to generate manifests in apps/web: rpc error: code = Unknown desc = ... repository not accessible` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Argo CD validates the source at create time and cannot access the repo (credentials, URL typo, network).

**Diagnose →** `argocd repo list` to see if the repo is registered and its connection status; `argocd repo get <url>`.

**Fix →** Register the repo with correct credentials or fix the URL; or create with `--validate=false` and fix later (not recommended).

**Prevent →** Credential templates for the org and consistent URL forms.

</details>

### Q38. Multi-source app error: `$values` is not substituted, Helm says `open $values/envs/prod/values.yaml: no such file or directory`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No source has `ref: values`, the ref name differs (`ref: config` but `$values` used), or the app uses `source` (singular) instead of `sources`.

**Diagnose →** `argocd app get web -o yaml | yq '.spec.sources'`.

**Fix →** Add `ref: values` to the Git source, match the name in the path, and use `sources:` list form.

**Prevent →** Template multi-source apps from a known-good example.

</details>

## Repos, Clusters, Projects & Permissions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q39. Adding a private GitHub repo over HTTPS fails with `rpc error: code = Unknown desc = authentication required`. 🟢

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

### Q40. An SSH repo fails with `ssh: handshake failed: knownhosts: key is unknown`. 🟢

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

### Q41. SSH repo: `ssh: handshake failed: ssh: unable to authenticate, attempted methods [none publickey], no supported methods remain`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong private key, the public half was never added as a deploy key, the key has a passphrase (Argo CD cannot use it), or the URL uses a user other than `git`.

**Diagnose →** `ssh -i key -T git@github.com` from a workstation; check the secret's `sshPrivateKey` field starts with `-----BEGIN OPENSSH PRIVATE KEY-----` and has a trailing newline.

**Fix →** Generate a passphrase-less deploy key (`ssh-keygen -t ed25519 -N ""`), add the public key read-only on the repo, and register: `argocd repo add git@github.com:org/config.git --ssh-private-key-path ./key --upsert`.

**Prevent →** One deploy key per repo, rotated on a schedule, stored through External Secrets.

</details>

### Q42. A self-hosted GitLab repo fails with `x509: certificate signed by unknown authority`. 🟡

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

### Q43. A credential template exists for `https://github.com/org`, yet new repos under the org still fail with `authentication required`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** URL mismatch: the template is `https://github.com/org` but apps use `git@github.com:org/...` (SSH), or a different host alias like `https://www.github.com`. Templates match by URL prefix, so protocol and host must match exactly. A labelled `repo-creds` secret is also required (`secret-type: repo-creds`).

**Diagnose →** `argocd repocreds list`; compare its URL with the exact `repoURL` in the Application.

**Fix →** Standardise all `repoURL`s on one form, or add a template for the other form.

**Prevent →** Validate `repoURL` format in CI with a simple lint rule.

</details>

### Q44. Sync fails with `application repo https://github.com/org/payments-config.git is not permitted in project 'team-a'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The AppProject's `sourceRepos` does not include the repo, often because of a missing `.git` suffix or a different URL form.

**Diagnose →** `argocd proj get team-a` and look at Source Repositories.

**Fix →** `argocd proj add-source team-a https://github.com/org/payments-config.git`, or use a glob in Git: `sourceRepos: ["https://github.com/org/payments-*"]`.

**Prevent →** Manage AppProjects in Git, reviewed by the platform team, with globs scoped per team.

</details>

### Q45. `application destination server 'https://kubernetes.default.svc' and namespace 'payments' do not match any of the allowed destinations in project 'team-a'`. 🟢

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

### Q46. Sync fails with `resource :Namespace is not permitted in project team-a.` 🟡

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

### Q47. A namespaced resource is blocked: `resource networking.k8s.io:NetworkPolicy is not permitted in project team-a.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The project has a `namespaceResourceBlacklist` entry for that kind, or a `namespaceResourceWhitelist` that omits it. Platforms often block NetworkPolicy, ResourceQuota and LimitRange so tenants cannot loosen guardrails.

**Diagnose →** `argocd proj get team-a -o yaml | yq '.spec.namespaceResource*'`.

**Fix →** If the block is intended, move the object to the platform repo. If not, remove it from the blacklist through a reviewed PR.

**Prevent →** Document which kinds tenants own; return a clear error in a CI policy check before merge.

</details>

### Q48. `InvalidSpecError: application references project team-a which does not exist`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The AppProject was not created yet (ordering in the bootstrap), it lives in a different namespace than the Argo CD control plane namespace, or a typo.

**Diagnose →** `kubectl -n argocd get appprojects`.

**Fix →** Create the project first. In app-of-apps bootstraps, put AppProjects in an earlier sync wave (`argocd.argoproj.io/sync-wave: "-1"`) than the Applications.

**Prevent →** Bootstrap order: projects, then platform apps, then tenant apps.

</details>

### Q49. With "apps in any namespace" enabled, an Application in `team-a` namespace shows `app is not allowed in project 'team-a', or the project does not exist`. 🔴

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

### Q50. A user clicks Sync and gets `rpc error: code = PermissionDenied desc = permission denied: applications, sync, team-a/web, sub: alice@example.com`. 🟡

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

### Q51. SSO login works but every user sees no applications. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `policy.default` is empty and the groups claim is not reaching Argo CD: the IdP does not send `groups` in the ID token, Azure AD sends group object IDs instead of names, or Okta's groups claim filter excludes the groups.

**Diagnose →** `argocd account get-user-info` after login; decode the ID token at the CLI (`argocd account get-user-info` shows `Groups:`); check Dex logs: `kubectl -n argocd logs deploy/argocd-dex-server`.

**Fix →** Configure the IdP to emit groups (Azure: `groupMembershipClaims`, map by object ID in `policy.csv`); set `policy.default: role:readonly` if read access for all is acceptable.

**Prevent →** Keep one break-glass local admin, disabled by default, and document the claim mapping.

</details>

### Q52. SSO login fails with `failed to get token: oauth2: cannot fetch token: 401 Unauthorized` or `invalid_client`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong or rotated client secret, the secret referenced as `$oidc.azure.clientSecret` does not exist in `argocd-secret`, or the redirect URI registered at the IdP differs from `https://argocd.example.com/auth/callback` (or `/api/dex/callback` for Dex).

**Diagnose →** `kubectl -n argocd logs deploy/argocd-server | grep -i oidc`; confirm `url:` in `argocd-cm` is the public URL users hit.

**Fix →** Update the client secret in `argocd-secret` (or the external secret it references), fix `url` and the registered redirect URI, then restart argocd-server.

**Prevent →** Calendar the IdP secret expiry (Azure secrets expire), and sync the secret through External Secrets.

</details>

### Q53. `argocd app list` from a laptop fails with `transport: received the unexpected content-type "text/plain; charset=utf-8"`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CLI speaks gRPC, but an ingress or load balancer in front of argocd-server only proxies HTTP/1.1 and returns an HTML or text error page.

**Diagnose →** Retry with `--grpc-web`; if that works, the proxy is the problem.

**Fix →** Use `argocd login argocd.example.com --grpc-web`, or configure the ingress for gRPC (separate gRPC host with `backend-protocol: GRPC` on ingress-nginx, or an ALB with a gRPC target group).

**Prevent →** Document the login command with `--grpc-web` for users behind corporate proxies.

</details>

### Q54. The UI behind ingress-nginx loops with `ERR_TOO_MANY_REDIRECTS`. 🟢

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

### Q55. A CI job's `argocd app sync` fails with `account 'ci-bot' does not have apiKey capability`. 🟢

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

### Q56. A cluster shows `ConnectionStatus: Failed` with `the server has asked for the client to provide credentials`. 🟡

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

### Q57. Destination fails with `unable to find destination server: there are no clusters with this name: prod-eu`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The cluster secret's `name` field differs (`prod-eu-1`), the secret is missing the `argocd.argoproj.io/secret-type: cluster` label, or the cluster was registered under a server URL only.

**Diagnose →** `argocd cluster list` shows names and servers.

**Fix →** Rename the cluster in its secret (`stringData.name: prod-eu`) or update the Application's `destination.name`.

**Prevent →** Generate cluster secrets from a single source (Terraform or a cluster ApplicationSet) with a naming convention.

</details>

### Q58. A cluster added with namespace scope shows `failed to sync cluster: ... deployments.apps is forbidden: User "system:serviceaccount:kube-system:argocd-manager" cannot list resource "deployments" in API group "apps" at the cluster scope`. 🔴

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

### Q59. Repo connection works from a laptop, but in the cluster: `dial tcp 140.82.112.4:443: i/o timeout`. 🟡

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

### Q60. Argo CD stops refreshing many apps, and repo-server logs show `remote: Repository not found` or `429 Too Many Requests` from GitHub during busy hours. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Hundreds of apps poll the same repos every 3 minutes with one token, hitting GitHub rate limits; GitHub returns misleading "not found" when a token is throttled or lacks access.

**Diagnose →** Count `git ls-remote` calls in repo-server logs; check the rate limit for the token: `curl -H "Authorization: Bearer $TOKEN" https://api.github.com/rate_limit`.

**Fix →** Configure webhooks and raise `timeout.reconciliation` (for example `600s`) in `argocd-cm`; switch to a GitHub App credential (higher limits); consolidate apps on fewer repos.

**Prevent →** Webhook-driven refresh plus jittered polling (`timeout.reconciliation.jitter`) as the fallback.

</details>

## Sync Operations, Hooks & Health

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q61. Sync fails with `one or more synchronization tasks are not valid` and `the server could not find the requested resource` for a custom resource. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CRD and its CRs are in the same sync, and the dry-run runs before the CRD exists, so the API server does not know the kind.

**Diagnose →** `argocd app get web` lists the failing CR kind; `kubectl get crd | grep <group>` on the target.

**Fix →** Add `argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true` to the CRs, or put the CRD in an earlier sync wave, or install CRDs from a separate app.

**Prevent →** Operators and their CRDs belong to a platform app that syncs before tenant apps.

</details>

### Q62. `one or more objects failed to apply, reason: Deployment.apps "web" is invalid: spec.selector: Invalid value: ...: field is immutable`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Someone changed labels used in `spec.selector` (often via a chart upgrade or a new `commonLabels` in Kustomize). Selectors and some fields (Job `spec.template`, Service `clusterIP`) cannot be updated in place.

**Diagnose →** `argocd app diff web` shows the selector change.

**Fix →** Revert the selector change, or let Argo CD recreate just that resource: annotate it with `argocd.argoproj.io/sync-options: Replace=true` (or `Force=true,Replace=true`) for one sync. Expect a brief outage for that Deployment.

**Prevent →** Use `labels` with `includeSelectors: false` in Kustomize instead of `commonLabels`; review chart upgrades for selector changes.

</details>

### Q63. Sync of a large CRD fails with `metadata.annotations: Too long: must have at most 262144 bytes`. 🟢

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

### Q64. An app has been stuck at `Syncing` for an hour with `waiting for completion of hook batch/Job/db-migrate`. 🟡

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

### Q65. A PreSync hook fails with `jobs.batch "db-migrate" already exists`. 🟢

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

### Q66. A PostSync smoke-test hook never runs, even though the sync says succeeded. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** PostSync hooks run only after all resources are Healthy. If something stays Progressing (an Ingress without an address, a PVC waiting for a consumer, a Rollout paused), the hook waits. Also, a hook placed in a file that the source excludes never renders.

**Diagnose →** `argocd app get web` and look for resources not Healthy; confirm the hook is in `argocd app manifests web`.

**Fix →** Fix the stuck resource or its health check (see Q167), then re-sync.

**Prevent →** Keep a short list of kinds with known slow health and add custom health checks for them.

</details>

### Q67. An Ingress stays `Progressing` forever, holding the app at Progressing. 🟡

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

### Q68. App health is `Degraded` with `Deployment "web" exceeded its progress deadline`. 🟢

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

### Q69. `argocd app rollback web 12` fails with `rollback cannot be initiated when auto-sync is enabled`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** By design: with auto-sync on, Git is the only source of truth, so a rollback would be undone immediately.

**Diagnose →** `argocd app get web -o yaml | yq '.spec.syncPolicy.automated'`.

**Fix →** The GitOps way: `git revert <bad-commit>` and push. For an emergency, disable auto-sync (`argocd app set web --sync-policy none`), roll back, then fix Git and re-enable.

**Prevent →** Teach on-call that rollback means revert in Git; keep the emergency path in the runbook with a reminder to re-enable auto-sync.

</details>

### Q70. Manual sync is refused with a message that the app is blocked by a sync window. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A `deny` window is active in the AppProject (for example a weekend freeze), or an `allow` window exists and the current time is outside it.

**Diagnose →** `argocd proj windows list team-a` shows active windows and whether `manualSync` is allowed.

**Fix →** If the change is an approved hotfix and the window has `manualSync: true`, sync manually. Otherwise ask the project owner to adjust the window through Git.

**Prevent →** Use `manualSync: true` on freeze windows so humans can still ship approved fixes; set `timeZone` explicitly to avoid UTC surprises.

</details>

### Q71. Deleting an Application hangs with the app stuck in `Deleting`. 🟡

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

### Q72. Sync fails with `namespaces "payments" not found`. 🟢

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

### Q73. Sync shows `Retrying attempt #5` and the operation finally fails with `admission webhook "validation.gatekeeper.sh" denied the request`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A policy engine (Gatekeeper, Kyverno) rejects the object, for example missing resource limits or a disallowed registry. Retries cannot help.

**Diagnose →** `argocd app get web` shows the full denial message; `kubectl get constraints` or `kubectl get clusterpolicy` on the cluster.

**Fix →** Fix the manifest in Git to meet the policy. Do not add policy exceptions to unblock one deploy without review.

**Prevent →** Run the same policies in CI (`gator test`, `kyverno apply`) against rendered manifests before merge.

</details>

### Q74. A sync succeeds but some resources are applied before their dependencies despite sync waves. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Waves only order resources inside one Application. The dependency lives in another app; or the wave annotation is on a Helm value that never renders; or the wave value is not a quoted string.

**Diagnose →** `argocd app manifests web | grep -B2 -A2 sync-wave`; check which app owns each resource.

**Fix →** Move dependent resources into the same app, or order Applications themselves with waves in the parent app-of-apps. Note that Application health must be enabled for waves between apps to wait on child health: add the `argoproj.io/Application` health customization in `argocd-cm`.

**Prevent →** Keep tightly coupled resources in one app and quote wave values (`"-1"`).

</details>

## ApplicationSets, Rollouts & Image Updater

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q75. The ApplicationSet status shows `ApplicationSet argocd/team-apps contains applications with duplicate name: web`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two generator results produce the same `metadata.name`, for example a git directory generator finding `apps/web` in two envs while the name template uses only `{{.path.basename}}`.

**Diagnose →** `kubectl -n argocd get applicationset team-apps -o yaml | yq '.status.conditions'`.

**Fix →** Make the name unique per element: `name: '{{.path.basename}}-{{index .path.segments 1}}'` or include the cluster name.

**Prevent →** Name template convention `<app>-<env>-<cluster>` for every ApplicationSet.

</details>

### Q76. `ApplicationGenerationFromParamsError: failed to execute go template ...: map has no entry for key "cluster"`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With `goTemplate: true` and `goTemplateOptions: ["missingkey=error"]`, the template references a key the generator does not produce. Cluster generator keys are `name`, `server`, `metadata.labels.<key>`; the git directory generator produces `path.*`.

**Diagnose →** Print the generator output by temporarily templating a ConfigMap, or check the generator's documented parameters.

**Fix →** Use the right key (`{{.name}}` not `{{.cluster}}`); with a matrix generator, check that the child generator providing the key is actually there.

**Prevent →** Keep `missingkey=error`; it catches typos instead of creating apps named `<no value>`.

</details>

### Q77. A git outage made the git generator return nothing, and the ApplicationSet deleted all its Applications and their workloads. 🔴

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

### Q78. Manual edits to an Application owned by an ApplicationSet (for example a changed `targetRevision`) keep getting reverted. 🟢

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

### Q79. A newly registered cluster gets no Applications from a cluster-generator ApplicationSet. 🟡

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

### Q80. The SCM provider generator stops discovering repos, with `GET https://api.github.com/orgs/org/repos: 403 API rate limit exceeded`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No token or a shared token, and a short `requeueAfterSeconds`, so the controller lists the org too often.

**Diagnose →** ApplicationSet controller logs; check the token's rate limit.

**Fix →** Use a GitHub App (`appSecretName`) or a dedicated token via `tokenRef`, raise `requeueAfterSeconds` to 1800 or more, and add filters (`repositoryMatch`, `labelMatch`) to reduce API calls.

**Prevent →** Webhooks to the ApplicationSet controller for event-driven refresh instead of tight polling.

</details>

### Q81. Pull request generator preview environments are not created for new PRs. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The PR does not carry the required label filter, the token lacks pull request read permission, the `requeueAfterSeconds` is long and no webhook is set, or the PR is from a fork (token cannot see it).

**Diagnose →** Controller logs for the set; `kubectl -n argocd get applicationset previews -o yaml | yq '.status'`.

**Fix →** Add the label, fix token scopes, and configure the ApplicationSet webhook (`/api/webhook` on the applicationset-controller service) for `pull_request` events.

**Prevent →** Document the label and add a CI check that comments the preview URL so missing previews are noticed.

</details>

### Q82. Progressive sync (RollingSync) settings are ignored; all apps update at once. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Progressive syncs are off by default in the ApplicationSet controller, or the step `matchExpressions` match no app labels, or the generated apps have their own `automated` sync, which RollingSync needs to control.

**Diagnose →** `kubectl -n argocd get cm argocd-cmd-params-cm -o yaml | grep progressive`; `kubectl -n argocd get applicationset fleet -o yaml | yq '.status.applicationStatus'`.

**Fix →** Set `applicationsetcontroller.enable.progressive.syncs: "true"`, restart the controller, and ensure template labels match the step selectors (`env: canary`, `env: prod`).

**Prevent →** Test the step order with a no-op change before relying on it for a real rollout.

</details>

### Q83. Argo CD shows the app `Suspended`, and the deploy pipeline waiting on `argocd app wait --health` times out. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An Argo Rollout is paused at a canary step (`pause: {}` with no duration) waiting for promotion. Argo CD maps a paused Rollout to Suspended health.

**Diagnose →** `kubectl argo rollouts get rollout web -n web`.

**Fix →** Promote if metrics look good: `kubectl argo rollouts promote web -n web`, or abort. For pipelines, use `argocd app wait web --health --suspended` or switch to timed pauses with analysis.

**Prevent →** Replace indefinite pauses with `pause: {duration: 10m}` plus an AnalysisTemplate, so promotion is automated.

</details>

### Q84. A Rollout aborts with `RolloutAborted: Rollout aborted update to revision 7: Metric "success-rate" assessed Failed due to failed (3) > failureLimit (2)`. 🟡

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

### Q85. An AnalysisRun shows `Error` with `Post "http://prometheus.monitoring:9090/api/v1/query": dial tcp: lookup prometheus.monitoring on 10.96.0.10:53: no such host`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong Prometheus address (short name instead of `prometheus-server.monitoring.svc.cluster.local`, wrong port), or a NetworkPolicy blocks the rollout namespace from reaching monitoring.

**Diagnose →** `kubectl -n monitoring get svc`; test with `kubectl run tmp --rm -it --image=curlimages/curl -- curl <address>/-/ready`.

**Fix →** Correct `provider.prometheus.address` in the AnalysisTemplate, or allow the traffic. Note that measurement errors count against `consecutiveErrorLimit` (default 4) and then abort.

**Prevent →** Use a ClusterAnalysisTemplate maintained by the platform team with the verified address.

</details>

### Q86. Argo CD Image Updater logs `could not get tags from registry: denied: requested access to the resource is denied` for a private ECR or GHCR image. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No registry credentials configured for that image, or the credentials (ECR tokens expire every 12 hours) went stale.

**Diagnose →** `kubectl -n argocd logs deploy/argocd-image-updater | grep denied`.

**Fix →** Configure `pullsecret:namespace/secret` for the image or registry in `registries.conf`, and for ECR use an `ext:` credential script or an ECR token refresher that updates the secret before expiry.

**Prevent →** Alert on Image Updater errors; a silent updater means images silently stop rolling.

</details>

### Q87. Image Updater bumps the tag, but within minutes the app reverts to the old image. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The write-back method is `argocd` (it writes parameter overrides on the Application object), and the Application is itself managed by an app-of-apps or ApplicationSet that overwrites the spec from Git on the next sync.

**Diagnose →** `argocd app get web -o yaml | yq '.spec.source.helm.parameters'` before and after the revert; check `ownerReferences` or the parent app.

**Fix →** Use Git write-back so the change lives in Git: `write-back-method: git` with `write-back-target: kustomization` or a Helm values file (`.argocd-source-<app>.yaml`) and a credential that can push.

**Prevent →** Standardise on Git write-back for any app managed by GitOps of GitOps; the git history doubles as the deploy audit trail.

</details>

### Q88. Image Updater with Git write-back fails with `failed to push: authentication required` or `protected branch hook declined`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** It reuses the repo credentials Argo CD uses for reading, which are read-only deploy keys; or the target branch is protected against direct pushes.

**Diagnose →** Image Updater logs show the push error and the branch.

**Fix →** Provide a write credential (`git:secret:argocd/git-write-creds`) and either allow the bot to bypass protection on that branch or write to a separate branch (`write-back-branch: image-updates`) that a CI job merges via PR.

**Prevent →** Separate read and write credentials; keep a branch protection rule that allows only the bot's identity.

</details>

## Secrets, Controller Performance & Recovery

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q89. A Sealed Secret is not decrypted: the SealedSecret shows `no key could decrypt secret (password)`. 🟡

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

### Q90. A KSOPS or SOPS-based app fails with `Error: failed to get the data key required to decrypt the SOPS file`. 🟡

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

### Q91. argocd-vault-plugin apps fail with `Replace: could not replace all placeholders in Template: ... secret not found`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The path in the `<path:...#key>` placeholder or the `avp.kubernetes.io/path` annotation is wrong, the KV version mismatches (KV v2 needs `data/` in the path), or the plugin's Vault role cannot read it.

**Diagnose →** `vault kv get <path>` with the same role; CMP sidecar logs in repo-server.

**Fix →** Correct the path, KV version, or Vault policy. Note that AVP renders secrets into manifests, so they appear in Argo CD's cache and diff; consider moving to External Secrets Operator.

**Prevent →** Prefer controllers that fetch secrets in-cluster (ESO, Vault Secrets Operator) so secret values never pass through the repo-server.

</details>

### Q92. An ExternalSecret is `SecretSyncedError` with `could not get secret data from provider`, and the app is Degraded. 🟢

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

### Q93. The application controller is OOMKilled repeatedly after onboarding a large cluster. 🔴

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

### Q94. Reconciliation is slow for all apps; `argocd_app_reconcile` p95 is minutes and the workqueue depth keeps growing. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Too few processors for the number of apps, a slow repo-server (manifest generation backlog), or one heavy cluster starving others on a single shard.

**Diagnose →** Check `workqueue_depth`, `argocd_app_reconcile_bucket` and `argocd_git_request_duration_seconds` in Prometheus; `argocd admin cluster shards`.

**Fix →** Raise `controller.status.processors` (for example 50) and `controller.operation.processors` (25) in `argocd-cmd-params-cm`; scale repo-server replicas; add controller shards and use the `round-robin` or `consistent-hashing` sharding algorithm.

**Prevent →** Capacity plan: track apps and resources per shard and scale before the queue grows.

</details>

### Q95. The UI and CLI are slow and logs show `Failed to get cached app state: dial tcp 10.96.40.2:6379: connect: connection refused`. 🟡

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

### Q96. Repo-server pods fail renders with `write /tmp/...: no space left on device`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Large monorepos and Helm chart caches fill the repo-server's ephemeral storage, especially with many revisions checked out in parallel.

**Diagnose →** `kubectl -n argocd exec deploy/argocd-repo-server -- df -h /tmp`; check pod events for ephemeral storage eviction.

**Fix →** Mount a larger `emptyDir` (with `sizeLimit`) or a dedicated volume on `/tmp`; scale out repo-server so each pod handles fewer repos; restart the pod to clear stale checkouts.

**Prevent →** Split huge monorepos, and use `argocd.argoproj.io/manifest-generate-paths` so changes in unrelated paths do not trigger renders.

</details>

### Q97. After upgrading Argo CD, applies fail with `strict decoding error: unknown field "spec.sourceHydrator"` or the CRD update fails with `metadata.annotations: Too long`. 🔴

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

### Q98. The admin password is lost and `argocd-initial-admin-secret` was deleted long ago. 🟢

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

### Q99. Someone deleted an app-of-apps parent with cascade, and dozens of child apps and workloads disappeared. How do you recover? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The parent and children all had the `resources-finalizer.argocd.argoproj.io` finalizer, so deletion cascaded down to every workload.

**Diagnose →** `kubectl -n argocd get events`, audit logs for the delete call; confirm which namespaces lost workloads.

**Fix →** Re-apply the root Application from Git (`kubectl apply -f bootstrap/root-app.yaml`); Argo CD recreates children and workloads from Git. Restore stateful data (PVs, databases) from backups, since Git does not hold data.

**Prevent →** Remove the finalizer from parent apps (only children should cascade), restrict `delete` on Applications in RBAC, enable deletion protection with `argocd.argoproj.io/sync-options: Delete=false` on critical resources, and back up Argo CD state with `argocd admin export`.

</details>

### Q100. Argo CD must be rebuilt on a new management cluster after the old one was lost. Apps show `Unknown` and clusters `Failed` after restoring from backup. 🔴

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
