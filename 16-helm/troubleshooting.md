# Helm — Troubleshooting Drills

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

> Real Helm errors and symptoms: likely causes, diagnosis, fix and prevention · 100 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Helm](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Install & Upgrade Failures](#install--upgrade-failures) — Q1–Q25
- [Templating & Rendering Errors](#templating--rendering-errors) — Q26–Q48
- [Values, Schema & Type Errors](#values-schema--type-errors) — Q49–Q63
- [Dependencies, Repositories & OCI](#dependencies-repositories--oci) — Q64–Q80
- [Hooks, Tests, GitOps & Secrets](#hooks-tests-gitops--secrets) — Q81–Q100

---

## Install & Upgrade Failures

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. `Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress` 🟢

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

### Q2. `Error: INSTALLATION FAILED: rendered manifests contain a resource that already exists. Unable to continue with install: Service "shop-api" in namespace "shop" exists and cannot be imported into the current release: invalid ownership metadata; label validation error: missing key "app.kubernetes.io/managed-by": must be set to "Helm"; annotation validation error: missing key "meta.helm.sh/release-name": must be set to "shop"` 🔴

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

### Q3. `Error: INSTALLATION FAILED: cannot re-use a name that is still in use` 🟢

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

### Q4. `Error: UPGRADE FAILED: context deadline exceeded` 🟢

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

### Q5. `Error: UPGRADE FAILED: release shop failed, and has been rolled back due to atomic being set: context deadline exceeded` 🟡

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

### Q6. `Error: UPGRADE FAILED: cannot patch "shop-api" with kind Deployment: Deployment.apps "shop-api" is invalid: spec.selector: Invalid value: v1.LabelSelector{...}: field is immutable` 🔴

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

### Q7. `Error: UPGRADE FAILED: "shop" has no deployed releases` 🔴

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

### Q8. `Error: INSTALLATION FAILED: create: failed to create: namespaces "shop" not found` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Helm does not create the target namespace by default.

**Diagnose →** `kubectl get ns shop`.

**Fix →** Add `--create-namespace`, or create it beforehand (`kubectl create ns shop`), ideally through your platform pipeline with labels and quotas.

**Prevent →** Namespaces managed declaratively by the platform team; app pipelines use `--create-namespace` only in dev.

</details>

### Q9. `Error: Kubernetes cluster unreachable: Get "http://localhost:8080/version": dial tcp [::1]:8080: connect: connection refused` 🟢

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

### Q10. `Error: INSTALLATION FAILED: unable to build kubernetes objects from release manifest: resource mapping not found for name: "shop" namespace: "" from "": no matches for kind "ServiceMonitor" in version "monitoring.coreos.com/v1" ensure CRDs are installed first` 🟡

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

### Q11. `Error: UPGRADE FAILED: unable to build kubernetes objects from current release manifest: resource mapping not found for name: "shop-api" namespace: "" from "": no matches for kind "PodDisruptionBudget" in version "policy/v1beta1"` 🔴

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

### Q12. `Error: INSTALLATION FAILED: chart requires kubeVersion: >=1.29.0 which is incompatible with Kubernetes v1.30.4-eks-a737599` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The constraint lacks the `-0` pre-release suffix, so SemVer treats `v1.30.4-eks-...` as a pre-release that does not satisfy `>=1.29.0`. Or the cluster is genuinely too old.

**Diagnose →** `kubectl version` and check `kubeVersion` in `Chart.yaml`.

**Fix →** Change to `kubeVersion: ">=1.29.0-0"`. For a third-party chart, pull, patch and repackage, or ask upstream.

**Prevent →** Always append `-0` in `kubeVersion` constraints and `semverCompare` calls.

</details>

### Q13. `Error: INSTALLATION FAILED: release name "customer-analytics-platform-reporting-service-production" exceeds max length of 53` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Release names are limited to 53 characters.

**Diagnose →** `echo -n "<name>" | wc -c`.

**Fix →** Shorten the release name (`reporting-prod`) and use `fullnameOverride` if resource names must stay specific.

**Prevent →** A naming convention for releases: short app name plus environment.

</details>

### Q14. `Error: INSTALLATION FAILED: Service "shop-frontend-...-very-long" is invalid: metadata.name: Invalid value: "...": must be no more than 63 characters` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A template builds names by concatenation without truncating.

**Diagnose →** `helm template ... | yq 'select(.metadata.name | length > 63) | .metadata.name'`.

**Fix →** Use the `fullname` helper pattern with `trunc 63 | trimSuffix "-"` for every generated name, including suffixed ones (`printf "%s-frontend" $fullname | trunc 63 | trimSuffix "-"`).

**Prevent →** A unit test rendering with a 53-character release name.

</details>

### Q15. `Error: UPGRADE FAILED: failed to replace object: Service "shop-api" is invalid: spec.clusterIP: Invalid value: "": field is immutable` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `--force` was used; Helm replaced the whole object, and the rendered Service has no `clusterIP`, which the API server treats as an attempt to change an immutable field.

**Diagnose →** Check the pipeline flags for `--force`.

**Fix →** Remove `--force` and upgrade normally. If you truly need to recreate the object, delete it explicitly and upgrade.

**Prevent →** Ban `--force` in CI; handle immutable-field changes with explicit runbooks.

</details>

### Q16. `Error: create: failed to create: Secret "sh.helm.release.v1.platform.v12" is invalid: data: Too long: must have at most 1048576 bytes` 🔴

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

### Q17. `Error: INSTALLATION FAILED: secrets is forbidden: User "system:serviceaccount:ci:deployer" cannot list resource "secrets" in API group "" in the namespace "shop"` 🟢

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

### Q18. `Error: UPGRADE FAILED: an error occurred while rolling back the release. original upgrade error: context deadline exceeded: ...` 🔴

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

### Q19. `Error: UPGRADE FAILED: failed to create resource: Ingress.networking.k8s.io "shop" is invalid: spec.rules[0].host: Invalid value: "Shop.Example.com": a lowercase RFC 1123 subdomain must consist of lower case alphanumeric characters, '-' or '.'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A value contains uppercase or invalid characters.

**Diagnose →** `helm get values shop -n shop --all | yq '.ingress'`.

**Fix →** Correct the value; in the template, normalize with `| lower`.

**Prevent →** A `pattern` constraint in `values.schema.json` for hostnames (`^[a-z0-9]([-a-z0-9.]*[a-z0-9])?$`).

</details>

### Q20. `Error: INSTALLATION FAILED: admission webhook "validate.kyverno.svc-fail" denied the request: resource Deployment/shop/shop-api was blocked due to the following policies require-run-as-nonroot: ...` 🟡

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

### Q21. `Error: UPGRADE FAILED: cannot patch "shop-postgres" with kind StatefulSet: StatefulSet.apps "shop-postgres" is invalid: spec: Forbidden: updates to statefulset spec for fields other than 'replicas', 'ordinals', 'template', 'updateStrategy', 'persistentVolumeClaimRetentionPolicy' and 'minReadySeconds' are forbidden` 🔴

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

### Q22. `Error: INSTALLATION FAILED: failed to install CRD crds/crd.yaml: ... the server could not find the requested resource` or CRD apply errors on a fresh cluster. 🟡

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

### Q23. After an upgrade, the new chart version deployed but pods still run the old image. 🟡

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

### Q24. `Error: INSTALLATION FAILED: Unable to continue with install: CustomResourceDefinition "certificates.cert-manager.io" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata; annotation validation error: key "meta.helm.sh/release-namespace" must equal "cert-manager": current value is "kube-system"` 🔴

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

### Q25. `helm upgrade` succeeded but `helm list` shows the release `failed` a minute later after a retry. 🟡

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

## Templating & Rendering Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. `Error: template: shop/templates/api-deployment.yaml:24:28: executing "shop/templates/api-deployment.yaml" at <.Values.api.image.tag>: nil pointer evaluating interface {}.tag` 🟢

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

### Q27. `Error: YAML parse error on shop/templates/api-deployment.yaml: error converting YAML to JSON: yaml: line 31: did not find expected key` 🟢

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

### Q28. `Error: YAML parse error on shop/templates/configmap.yaml: error converting YAML to JSON: yaml: line 7: mapping values are not allowed in this context` 🟡

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

### Q29. `Error: template: shop/templates/api-deployment.yaml:18:12: executing "..." at <include "shop.labels" .>: error calling include: template: no template "shop.labels" associated with template "gotpl"` 🟢

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

### Q30. `Error: template: shop/templates/_helpers.tpl:14:14: executing "shop.fullname" at <.Chart.Name>: can't evaluate field Chart in type string` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The helper was called with the wrong context, for example inside `range` or `with` where `.` is a string, or with a dict that does not carry the root.

**Diagnose →** Look at the call site: `{{ include "shop.fullname" . }}` inside `{{ range .Values.hosts }}`.

**Fix →** Pass the root: `{{ include "shop.fullname" $ }}`, or for multi-parameter helpers pass `(dict "root" $ "host" .)` and use `.root` inside.

**Prevent →** Document each helper's expected context in a template comment.

</details>

### Q31. `Error: parse error at (shop/templates/api-deployment.yaml:40): function "toYml" not defined` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Typo or a function from another tool (`toYml`, `indentYaml`, `env` from older Sprig docs that Helm disables).

**Diagnose →** Check the function name against Helm's template function list.

**Fix →** Use the real name (`toYaml`). Note that Helm removes `env` and `expandenv` for security, so environment variables cannot be read in templates; pass them as values instead.

**Prevent →** `helm lint` in pre-commit.

</details>

### Q32. `Error: parse error at (shop/templates/ingress.yaml:42): unexpected {{end}}` or `unexpected EOF` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Unbalanced control structures: an extra `{{ end }}`, a missing one, or an `if` closed inside a `range` that opened outside.

**Diagnose →** Count openers and closers: `grep -oE '\{\{-? *(if|range|with|define|end)' templates/ingress.yaml`.

**Fix →** Balance blocks; indent template logic consistently so structure is visible.

**Prevent →** Editor support for Helm templates (VS Code Kubernetes extension, helm-ls language server).

</details>

### Q33. `Error: template: ... at <.Release.Name>: can't evaluate field Release in type map[string]interface {}` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `.Release` used inside `range` or `with`, where `.` is now a values item.

**Diagnose →** Find the enclosing `range`/`with` at the reported line.

**Fix →** Use `$.Release.Name` (and `$.Values`, `$.Chart`) inside scoped blocks.

**Prevent →** Habit: always use `$` for root objects inside loops.

</details>

### Q34. `Error: execution error at (shop/templates/secret.yaml:9:15): postgres.auth.password is required` 🟢

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

### Q35. `Error: template: shop/templates/configmap.yaml:8:18: executing "..." at <.Values.app.config>: wrong type for value; expected string; got map[string]interface {}` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A string function (`quote`, `b64enc`, `trunc`) received a map: the values structure changed from a scalar to a nested object.

**Diagnose →** `yq '.app.config' values*.yaml` to see its type in each layer.

**Fix →** Use `toYaml`/`toJson` for maps, or reference the scalar inside the map.

**Prevent →** Schema `type` for that key so a wrong structure fails with a clear message.

</details>

### Q36. Rendered output contains `%!s(float64=1.1)` or `%!d(string=3000)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `printf` verbs do not match the value types (YAML number parsed as float64, or a quoted string passed to `%d`).

**Diagnose →** `helm template ... | grep '%!'`.

**Fix →** Convert: `printf "%s:%s" .repo (.tag | toString)`, `printf "%d" (.port | int)`.

**Prevent →** Quote string-like values in `values.yaml`; `toString` before `%s`.

</details>

### Q37. `error validating data: ValidationError(Deployment.spec.template.spec.containers[0].env[2].value): invalid type for io.k8s.api.core.v1.EnvVar.value: got "boolean", expected "string"` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An env value rendered unquoted as `true`, `false` or a number.

**Diagnose →** Render and inspect the `env` block.

**Fix →** `value: {{ .Values.featureFlag | quote }}` for every env var.

**Prevent →** Render env vars from a map with `range` that always quotes values.

</details>

### Q38. `json: cannot unmarshal string into Go struct field ContainerPort.spec.template.spec.containers.ports.containerPort of type int32` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The port was quoted in values (`port: "3000"`) or in the template (`{{ .Values.port | quote }}`).

**Diagnose →** `helm template ... | yq '.spec.template.spec.containers[].ports'`.

**Fix →** Render as integer: `containerPort: {{ .Values.api.port | int }}`; keep values unquoted.

**Prevent →** Schema with `"type": "integer"` for ports.

</details>

### Q39. Rendered YAML has two keys glued on one line, e.g. `app.kubernetes.io/name: shopapp.kubernetes.io/instance: shop`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Over-aggressive whitespace trimming: `-}}` at the end of a line inside a `define`, or `{{-` on an include without a following `nindent`.

**Diagnose →** Look at the helper's `define` block and the call site.

**Fix →** Inside helpers, trim on `define`/`end` lines only; at call sites use `{{- include "shop.labels" . | nindent 4 }}`.

**Prevent →** Unit tests asserting labels as a map rather than as raw text.

</details>

### Q40. A template renders empty, and Helm silently skips the resource you expected. 🟡

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

### Q41. `Error: template: shop/templates/configmap.yaml:6:3: executing "..." at <tpl .Values.app.config $>: error calling tpl: cannot retrieve Template.Basepath from values inside tpl function: ...` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `tpl` was called with the wrong context (a sub-map instead of the root), so templates inside the string cannot access `.Template` or `.Values`.

**Diagnose →** Inspect the `tpl` call: second argument must be the root context (`$` or `.` at top level).

**Fix →** `{{ tpl .Values.app.config $ }}`.

**Prevent →** Wrap `tpl` usage in a helper (`shop.render` taking `dict "value" ... "context" $`) used consistently.

</details>

### Q42. `Error: INSTALLATION FAILED: YAML parse error on shop/templates/secret.yaml: error converting YAML to JSON: yaml: invalid leading UTF-8 octet` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Binary content inserted into YAML, typically `.Files.Get` on a binary file (keystore, image) into `data` without encoding, or a value with invalid encoding.

**Diagnose →** `file files/*` to identify binaries.

**Fix →** Base64-encode binaries: `{{ .Files.Get "files/keystore.jks" | b64enc }}` into a Secret's `data`, or use `.Files.Glob(...).AsSecrets`.

**Prevent →** Keep binaries out of charts where possible; mount them from secret managers.

</details>

### Q43. Every `helm upgrade` restarts pods even when nothing changed. 🟡

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

### Q44. `helm template` works, but `helm install` fails on `lookup`-based logic with an unexpected nil. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** During `helm template`, `lookup` returns an empty map, so a different branch runs. During install it returns the real object, and code like `index $secret.data "password"` fails when the key name differs or the object exists but lacks that key.

**Diagnose →** `kubectl get secret <name> -n shop -o yaml` and compare with the template's expectations; use `--dry-run=server` to exercise the live branch.

**Fix →** Guard with `hasKey` before `index`: `{{ if and $secret (hasKey $secret.data "password") }}`.

**Prevent →** Test both branches: `helm template` (empty) and `--dry-run=server` against a cluster with the object.

</details>

### Q45. `Error: template: ... at <semverCompare ">=1.25" .Capabilities.KubeVersion.Version>: error calling semverCompare: ...` or the wrong branch renders on EKS/GKE. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Provider versions like `v1.30.4-eks-a737599` are pre-releases in SemVer; `>=1.25` does not match them, so the "old Kubernetes" branch renders.

**Diagnose →** `kubectl version -o json | jq -r .serverVersion.gitVersion`.

**Fix →** Use `>=1.25-0`.

**Prevent →** Always use the `-0` suffix in version comparisons.

</details>

### Q46. `Error: rendered manifests contain a resource that already exists. Unable to continue with install: ConfigMap "shop-config" in namespace "shop" exists and cannot be imported into the current release ...` but the ConfigMap is defined twice in your own chart. 🟡

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

### Q47. `walk.go:74: found symbolic link in path: /charts/shop/files/config resolves to /etc/shop/config. Contents of linked file included and used` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The chart directory contains a symlink, so Helm follows it and packages content from outside the chart, potentially leaking host files.

**Diagnose →** `find ./shop -type l -ls`.

**Fix →** Replace symlinks with real files inside the chart, or exclude them in `.helmignore`.

**Prevent →** Never symlink into charts; CI fails on this warning.

</details>

### Q48. `NOTES.txt` prints a broken command after install, like `kubectl port-forward svc/ 8080:80`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** NOTES uses a helper or value that does not exist in this configuration (for example the frontend disabled, or an old key name).

**Diagnose →** `helm get notes shop -n shop` and compare with `helm get values --all`.

**Fix →** Use the same helpers as the templates (`include "shop.fullname" .`), and branch on enabled components.

**Prevent →** Treat NOTES as a template under test: `helm template` does not print it, but `helm install --dry-run` does, so render it that way in CI for each component combination.

</details>

## Values, Schema & Type Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q49. `Error: values don't meet the specifications of the schema(s) in the following chart(s): shop: - api.replicaCount: Invalid type. Expected: integer, given: string` 🟢

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

### Q50. `Error: values don't meet the specifications of the schema(s) in the following chart(s): shop: - (root): Additional property replicacount is not allowed` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A typo in a values key; the schema uses `additionalProperties: false`.

**Diagnose →** Compare the key with `helm show values ./shop`.

**Fix →** Correct the key (`replicaCount`).

**Prevent →** This is the schema working as intended; keep `additionalProperties: false` on well-defined objects.

</details>

### Q51. `Error: values don't meet the specifications of the schema(s) in the following chart(s): shop-postgres: - auth: password is required` but you did set `postgres.auth.password`. 🔴

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

### Q52. `Error: failed parsing --set data: key "b.example.com" has no value` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A comma in a `--set` value: `--set ingress.hosts=a.example.com,b.example.com`. Commas separate assignments.

**Diagnose →** Inspect the command line.

**Fix →** Use list syntax `--set 'ingress.hosts={a.example.com,b.example.com}'`, escape commas (`a\,b`), use `--set-literal`, or a values file.

**Prevent →** Values files for anything beyond simple scalars.

</details>

### Q53. `Error: failed to parse values-prod.yaml: error converting YAML to JSON: yaml: line 12: found character that cannot start any token` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tabs used for indentation, or an unquoted value starting with a reserved character (`@`, `` ` ``, `%`).

**Diagnose →** `grep -nP '\t' values-prod.yaml`; `yq '.' values-prod.yaml`.

**Fix →** Replace tabs with spaces; quote values that start with special characters.

**Prevent →** yamllint in pre-commit and CI.

</details>

### Q54. `coalesce.go:286: warning: cannot overwrite table with non table for shop.api.resources (map[limits:map[memory:256Mi]])` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An override set `api.resources` to a scalar or empty string where the chart default is a map; Helm ignores the override.

**Diagnose →** Find the layer: `yq '.api.resources' values*.yaml`.

**Fix →** Use a map (`resources: {}` to clear) or `null` to delete the default.

**Prevent →** Fail CI on coalesce warnings: `helm template ... 2>&1 >/dev/null | grep -q 'warning:' && exit 1`.

</details>

### Q55. Image tag `1.10` deploys as `1.1` and pods fail with `ImagePullBackOff`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** YAML parsed `tag: 1.10` as a float; `--set api.image.tag=1.10` does the same.

**Diagnose →** `helm get manifest shop -n shop | grep image:`.

**Fix →** Quote in YAML (`tag: "1.10"`), use `--set-string`, and `toString` in the template.

**Prevent →** Schema `"type": "string"` for tags, which rejects unquoted numbers.

</details>

### Q56. A feature turned on in prod although the prod file says `enabled: "false"`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The quoted string `"false"` is non-empty and therefore truthy in `if`.

**Diagnose →** `helm get values shop --all -o json | jq '.feature.enabled | type'` returns `"string"`.

**Fix →** Use an unquoted boolean `enabled: false`.

**Prevent →** Schema `"type": "boolean"`.

</details>

### Q57. After `helm upgrade` with a new values file, settings from the previous deploy disappeared. 🟡

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

### Q58. `Error: INSTALLATION FAILED: unable to build kubernetes objects from release manifest: error validating "": error validating data: [ValidationError(Deployment.spec.template.spec.containers[0].resources): unknown field "limit" in io.k8s.api.core.v1.ResourceRequirements]` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A typo in a values block passed through `toYaml` (`limit` instead of `limits`).

**Diagnose →** `yq '.api.resources' values*.yaml`.

**Fix →** Correct the key.

**Prevent →** Schema for `resources` (or reference the Kubernetes JSON schema), plus kubeconform `-strict` in CI.

</details>

### Q59. `Error: values don't meet the specifications of the schema(s) in the following chart(s): shop: - global: Additional property global is not allowed` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Root-level `additionalProperties: false` in the parent schema without listing `global` (and subchart keys). Helm always injects `global`, so this fails even when the user did not set it.

**Diagnose →** `jq '.additionalProperties, (.properties | keys)' shop/values.schema.json`.

**Fix →** Add `"global": {"type": "object"}` and each subchart key to `properties`, or drop `additionalProperties: false` at the root.

**Prevent →** Apply strictness to nested objects rather than the root.

</details>

### Q60. `--set podAnnotations.prometheus.io/scrape=true` creates nested keys instead of one annotation. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Dots in keys are path separators in `--set`.

**Diagnose →** `helm get values shop -o yaml` shows `prometheus: {io/scrape: "true"}`.

**Fix →** Escape the dots: `--set-string 'podAnnotations.prometheus\.io/scrape=true'`, or use a values file.

**Prevent →** Annotations and labels belong in values files.

</details>

### Q61. Your prod override added one toleration, and all the chart's default tolerations disappeared. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Lists are replaced, not merged, across values layers.

**Diagnose →** `helm get values shop --all | yq '.api.tolerations'`.

**Fix →** Repeat the defaults you want to keep in your override.

**Prevent →** In charts you own, model collections as maps keyed by name and convert to lists in templates.

</details>

### Q62. `Error: open secrets/prod.yaml: no such file or directory` in CI only. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Relative paths resolved from a different working directory in CI, a sparse checkout, or the file is in `.gitignore`.

**Diagnose →** `pwd; ls -la secrets/` in the job.

**Fix →** Use paths relative to the repo root with an explicit `working-directory`, or absolute paths from `$GITHUB_WORKSPACE`.

**Prevent →** A deploy script that resolves its own directory (`cd "$(dirname "$0")"`).

</details>

### Q63. `Error: values don't meet the specifications of the schema(s) ... - api.image.tag: Invalid type. Expected: string, given: null` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `tag:` left empty in YAML means `null`; or `--set api.image.tag=null`.

**Diagnose →** Check values files for empty keys.

**Fix →** Set `tag: ""` if empty is allowed and the template defaults to `.Chart.AppVersion`, or provide a real tag.

**Prevent →** Schema `"type": ["string", "null"]` if null should be valid, and a template default.

</details>

## Dependencies, Repositories & OCI

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q64. ``Error: An error occurred while checking for chart dependencies. You may need to run `helm dependency build` to fetch missing dependencies: found in Chart.yaml, but missing in charts/ directory: shop-postgres`` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `charts/` is empty because `.tgz` files are not committed (correct) and the pipeline did not fetch dependencies.

**Diagnose →** `helm dependency list ./shop` shows `missing`.

**Fix →** `helm dependency build ./shop` before `template`/`install`/`package`.

**Prevent →** Add the build step to CI and to the Makefile; `helm package --dependency-update` for packaging.

</details>

### Q65. `Error: no repository definition for https://charts.example.com/stable. Please add the missing repos via 'helm repo add'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A dependency points to an HTTP repo not configured on this machine or runner.

**Diagnose →** `helm repo list`.

**Fix →** `helm repo add example https://charts.example.com/stable && helm repo update`, then `helm dependency build`.

**Prevent →** Script repo setup in CI from `Chart.yaml`, or switch dependencies to OCI.

</details>

### Q66. `Error: the lock file (Chart.lock) is out of sync with the dependencies file (Chart.yaml). Please update the dependencies` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Someone edited `dependencies` in `Chart.yaml` (version, repository, alias) without regenerating the lock.

**Diagnose →** `git diff Chart.yaml Chart.lock`.

**Fix →** `helm dependency update ./shop` and commit the new `Chart.lock`.

**Prevent →** A CI check that runs `helm dependency update` and fails if `Chart.lock` changes.

</details>

### Q67. `Error: looks like "https://charts.example.com" is not a valid chart repository or cannot be reached: failed to fetch https://charts.example.com/index.yaml : 404 Not Found` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong URL (missing path), the repo moved to OCI, the index was never generated, or authentication returns 404 for private content.

**Diagnose →** `curl -sI https://charts.example.com/index.yaml`.

**Fix →** Correct the URL; if the project moved to OCI, use `oci://` references instead of `helm repo add`; pass credentials for private repos.

**Prevent →** Track upstream announcements; mirror important charts internally.

</details>

### Q68. `Error: chart "shop" version "1.4.2" not found in https://charts.example.com repository` 🟢

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

### Q69. `Error: failed to authorize: failed to fetch oauth token: unexpected status from GET request to https://ghcr.io/token?...: 401 Unauthorized` on `helm pull oci://ghcr.io/...` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Not logged in, expired token, token lacking `read:packages`, or the package is private and the token's user has no access.

**Diagnose →** `cat "$(helm env HELM_REGISTRY_CONFIG)" | jq '.auths | keys'`.

**Fix →** `echo $TOKEN | helm registry login ghcr.io -u <user> --password-stdin`, with a token that has package read rights; grant the repo access to the package.

**Prevent →** In GitHub Actions use `GITHUB_TOKEN` with `packages: read` and link the package to the repository.

</details>

### Q70. `helm push` to ECR fails with `name unknown: The repository with name 'charts/shop' does not exist in the registry with id '123456789012'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ECR requires the repository to exist; it is not created on push. The repository name is the push path plus the chart name.

**Diagnose →** `aws ecr describe-repositories --repository-names charts/shop`.

**Fix →** `aws ecr create-repository --repository-name charts/shop --image-tag-mutability IMMUTABLE`, then push to `oci://$REG/charts`.

**Prevent →** Create chart repositories with Terraform alongside image repositories.

</details>

### Q71. `Error: unexpected status from HEAD request to https://123456789012.dkr.ecr.ap-south-1.amazonaws.com/v2/charts/shop/manifests/1.4.0: 403 Forbidden` 🟡

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

### Q72. `Error: invalid_reference: invalid tag` when pushing or pulling a chart with version `1.4.0+build.7`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** OCI tags cannot contain `+`. Helm converts `+` to `_` on push, so the tag in the registry is `1.4.0_build.7`; tools or scripts using the raw version fail.

**Diagnose →** `aws ecr list-images --repository-name charts/shop` or `crane ls`.

**Fix →** Reference the version with `+` in `helm pull --version` (Helm translates), or avoid build metadata in chart versions; use pre-release (`1.4.0-build.7`) or a plain patch bump.

**Prevent →** CI derives chart versions without `+`.

</details>

### Q73. `Error: failed to download "oci://ghcr.io/devopsshack/charts/shop" at version "1.4.0"` with an `http: server gave HTTP response to HTTPS client` error, using a local registry. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The local registry (`localhost:5001`, kind registry) speaks plain HTTP, and Helm defaults to HTTPS.

**Diagnose →** `curl -s http://localhost:5001/v2/_catalog`.

**Fix →** Add `--plain-http` (Helm 3.13+) to `push`, `pull`, `install`, `template`, `show`; for self-signed TLS use `--insecure-skip-tls-verify` or `--ca-file`.

**Prevent →** Use TLS in shared registries; reserve `--plain-http` for local labs.

</details>

### Q74. `Error: non-absolute URLs should be in form of repo_name/path_to_chart, got: postgresql` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You ran `helm install pg postgresql` expecting Helm to find it; Helm needs `repo/chart`, a path, a URL or an `oci://` reference.

**Diagnose →** `helm search repo postgresql`.

**Fix →** `helm install pg bitnami/postgresql` after adding the repo, or the OCI reference `oci://registry-1.docker.io/bitnamicharts/postgresql`.

**Prevent →** Always use fully qualified chart references in scripts.

</details>

### Q75. `Error: INSTALLATION FAILED: path "./shop" not found` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong working directory, or a local directory name collides with what you meant as a repo reference (or the reverse: a directory named like `bitnami/postgresql` shadows the repo).

**Diagnose →** `pwd; ls`.

**Fix →** Use the correct path or reference.

**Prevent →** Absolute paths in scripts.

</details>

### Q76. `helm dependency update` hangs or fails with `x509: certificate signed by unknown authority` behind a corporate proxy. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** TLS interception by the proxy with a corporate CA that Helm (Go) does not trust, or proxy variables not set.

**Diagnose →** `env | grep -i proxy`; `curl -v https://charts.example.com/index.yaml`.

**Fix →** Add the corporate CA to the OS trust store (Go uses it), set `HTTPS_PROXY`/`NO_PROXY`, or pass `--ca-file` when adding the repo.

**Prevent →** Base CI images with the corporate CA installed; internal chart mirror.

</details>

### Q77. Installing an OCI chart without `--version` deployed an unexpected pre-release. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Without `--version`, Helm resolves the highest SemVer tag in the registry; tags that are not stable releases can be picked, depending on tag naming.

**Diagnose →** `helm list -n shop` shows the chart version; check registry tags.

**Fix →** Pin `--version` (or a digest) and roll back.

**Prevent →** Always pin versions; publish pre-releases to a separate repository path.

</details>

### Q78. `helm dependency build` downloads a different subchart version than a colleague got. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `Chart.lock` is not committed (or is ignored), so `build` falls back to resolving ranges.

**Diagnose →** `git ls-files shop/Chart.lock`.

**Fix →** Commit `Chart.lock`; use `build`, not `update`, in CI.

**Prevent →** CI check that the lock exists and is in sync.

</details>

### Q79. `Error: Chart.yaml file is missing` when installing from a directory that clearly has charts. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You pointed at the parent directory (`./charts`) instead of the chart directory (`./charts/shop`), or the file is named `chart.yaml` (case sensitive).

**Diagnose →** `ls ./charts/shop/Chart.yaml`.

**Fix →** Use the chart directory path; rename the file to `Chart.yaml`.

**Prevent →** `helm lint` in CI for each chart directory.

</details>

### Q80. `Error: validation: chart.metadata.version "v1.4" is invalid` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Chart `version` must be strict SemVer 2 (`1.4.0`), not `v1.4` or `1.4`.

**Diagnose →** `yq '.version' shop/Chart.yaml`.

**Fix →** `version: 1.4.0`, or pass `helm package --version 1.4.0`.

**Prevent →** CI derives chart versions from Git tags with a SemVer check.

</details>

## Hooks, Tests, GitOps & Secrets

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q81. `Error: INSTALLATION FAILED: failed post-install: job failed: BackoffLimitExceeded` 🟢

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

### Q82. `Error: UPGRADE FAILED: pre-upgrade hooks failed: timed out waiting for the condition` 🟡

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

### Q83. `Error: UPGRADE FAILED: pre-upgrade hooks failed: warning: Hook pre-upgrade shop/templates/migrate-job.yaml failed: jobs.batch "shop-migrate" already exists` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A previous hook Job was not deleted (no `before-hook-creation` policy, or a policy with only `hook-succeeded` and the last run failed).

**Diagnose →** `kubectl get job shop-migrate -n shop`.

**Fix →** `kubectl delete job shop-migrate -n shop` and rerun; add `"helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded`.

**Prevent →** Always include `before-hook-creation` on hook Jobs and test Pods.

</details>

### Q84. The migration hook fails with `psql: error: connection to server at "shop-postgres" (10.96.12.4), port 5432 failed: Connection refused` on first install. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The hook runs as `pre-install` (DB does not exist yet), or as `post-install` without `--wait` and before Postgres is ready.

**Diagnose →** `helm get hooks shop -n shop | grep 'helm.sh/hook:'`; `kubectl get pods -n shop` timestamps.

**Fix →** Use `post-install,pre-upgrade` and add an init container that loops on `pg_isready -h shop-postgres -p 5432` before migrating.

**Prevent →** Treat DB readiness as an explicit dependency in hooks.

</details>

### Q85. On upgrade, the migration Job runs the **old** SQL even though the chart contains new migration files. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The SQL lives in a regular ConfigMap, which is updated only after `pre-upgrade` hooks run, so the Job mounts the previous version.

**Diagnose →** `kubectl get cm shop-migrations -n shop -o yaml` during the hook shows old content.

**Fix →** Make the migrations ConfigMap a hook too (`pre-upgrade`, weight `-5`, `before-hook-creation`) so it is created before the Job (weight `0`), or bake migrations into the migration image.

**Prevent →** Remember that `pre-*` hooks see the old state of regular resources.

</details>

### Q86. `helm test` fails with `Error: unable to get pod logs for shop-test-api: pods "shop-test-api" not found`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The test Pod was deleted before Helm fetched logs (`hook-delete-policy: hook-succeeded` or `hook-failed`) while using `--logs`.

**Diagnose →** Check the test template annotations.

**Fix →** Use `before-hook-creation` only for test Pods so they remain until the next run.

**Prevent →** Standardize test Pod annotations in the library chart.

</details>

### Q87. `helm test` hangs until timeout. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The test Pod has `restartPolicy: Always` (default for Pods is Always), so it never reaches `Succeeded`; or it is stuck in `Pending`/`ImagePullBackOff`.

**Diagnose →** `kubectl get pod shop-test-api -n shop -o jsonpath='{.spec.restartPolicy} {.status.phase}'`.

**Fix →** Set `restartPolicy: Never` in test Pods; fix the image or scheduling issue.

**Prevent →** Template test Pods from a shared helper with correct defaults.

</details>

### Q88. Argo CD Application shows `ComparisonError: ... failed to execute helm template command: ... Error: execution error at (shop/templates/secret.yaml:9:15): postgres.auth.password is required` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Argo CD renders without your secrets file: helm-secrets is not configured in the repo server, or the values file path is wrong in `valueFiles`.

**Diagnose →** `argocd app manifests shop-prod --source live` and the Application's `spec.source.helm.valueFiles`.

**Fix →** Use `existingSecret` with an ExternalSecret (preferred), or configure the helm-secrets integration in the Argo CD repo server (SOPS keys provided via IRSA/KMS) and reference `secrets://` value files.

**Prevent →** Keep secrets out of Helm values in GitOps setups.

</details>

### Q89. Argo CD shows the app `OutOfSync` forever because a Secret's data changes every refresh. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The chart generates the secret with `randAlphaNum` (or `genCA`) and relies on `lookup` to keep it, which does not work in Argo CD.

**Diagnose →** `argocd app diff shop-prod` shows only that Secret's data changing.

**Fix →** Create the Secret outside the chart and use `existingSecret`; temporarily `ignoreDifferences` on `/data` for that Secret.

**Prevent →** Deterministic charts for GitOps.

</details>

### Q90. A `pre-install` hook in a third-party chart runs on every Argo CD sync and resets data. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Argo CD maps both `pre-install` and `pre-upgrade` to `PreSync`, so install-only hooks run on each sync.

**Diagnose →** `kubectl get jobs -n <ns> --sort-by=.metadata.creationTimestamp` shows a new Job per sync.

**Fix →** Override with Argo CD annotations through a post-render step (Kustomize patch adding `argocd.argoproj.io/hook: Skip` or a different hook), disable the hook via chart values if offered, or make the job idempotent.

**Prevent →** Review chart hooks before adopting third-party charts in Argo CD.

</details>

### Q91. Argo CD fails with `Error: failed to fetch chart: ... oci://... : not found` while `helm pull` works locally. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Argo CD expects OCI Helm `repoURL` without the `oci://` scheme (`123456789012.dkr.ecr.ap-south-1.amazonaws.com/charts`) and the repo credential registered with `enableOCI: true`; or ECR tokens expired in the repo secret.

**Diagnose →** `argocd repo list`; inspect the Application `source`.

**Fix →** Correct `repoURL` and `chart`; register the repo as OCI; refresh ECR credentials automatically (a CronJob or an ECR credential helper, or IRSA-based integration).

**Prevent →** Manage Argo CD repo secrets declaratively with automated token refresh.

</details>

### Q92. `Error: plugin "secrets" exited with error` and `Failed to get the data key required to decrypt the SOPS file.` 🟢

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

### Q93. `Error: unknown command "secrets" for "helm"` (or `"diff"`) in CI. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Plugins are installed per user under `HELM_PLUGINS`; the CI image or runner does not have them, or `HELM_DATA_HOME` points elsewhere.

**Diagnose →** `helm plugin list; helm env HELM_PLUGINS`.

**Fix →** Install in the job: `helm plugin install https://github.com/jkroepke/helm-secrets --version v4.6.2` (pin the version), or bake plugins into the CI image.

**Prevent →** A pinned CI image with Helm, plugins, sops and kubeconform.

</details>

### Q94. helm-secrets leaves a decrypted `secrets.prod.yaml.dec` file in the workspace after a failed run. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrapper mode writes temporary decrypted files; a crash or kill skips cleanup.

**Diagnose →** `find . -name '*.dec'`.

**Fix →** Delete them; switch to getter mode (`-f secrets://...`), which streams decrypted content, and add `*.dec` to `.gitignore`.

**Prevent →** Ephemeral CI workspaces and a gitleaks pre-commit hook.

</details>

### Q95. Helmfile fails with `in ./helmfile.yaml: failed to read helmfile.yaml: reading document at index 1: yaml: line 14: did not find expected node content` after upgrading to Helmfile 1.x. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file contains Go template expressions (`{{ .Environment.Name }}`), and Helmfile 1.x only renders templates in files with the `.gotmpl` extension, so the raw braces reach the YAML parser.

**Diagnose →** `grep -n '{{' helmfile.yaml`.

**Fix →** Rename to `helmfile.yaml.gotmpl` (and templated values files to `*.yaml.gotmpl`), or remove templating.

**Prevent →** Read the major-version migration notes; pin the Helmfile version in CI.

</details>

### Q96. `helmfile apply` reports changes on every run even though nothing changed. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Non-deterministic templates (random values, timestamps), or secrets showing as changed because helm-diff cannot compare encrypted data consistently.

**Diagnose →** `helmfile -e prod diff --context 3` and inspect which fields differ.

**Fix →** Make templates deterministic; use `--suppress-secrets` for display; externalize generated secrets.

**Prevent →** The same render-twice determinism check used for Argo CD.

</details>

### Q97. After rollback, the app still reads the new config. 🟡

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

### Q98. `helm uninstall` completed, but PVCs, a hook ConfigMap and a CRD are still in the cluster. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Expected behavior: PVCs created by StatefulSet `volumeClaimTemplates` are not part of the release; hook resources without a delete policy are not tracked; CRDs from `crds/` are never deleted; `resource-policy: keep` objects are retained.

**Diagnose →** `kubectl get pvc,cm,secret -n shop`; `kubectl get crd | grep shop`.

**Fix →** Delete leftovers deliberately (`kubectl delete pvc -n shop -l app.kubernetes.io/instance=shop`), after confirming data is not needed.

**Prevent →** Document what uninstall leaves behind; delete policies on hooks.

</details>

### Q99. `Error: uninstallation completed with 1 error(s): uninstall: Failed to purge the release: release: not found` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Release Secrets were partly deleted manually or by another process during uninstall; Helm deleted resources but could not find the record to purge.

**Diagnose →** `kubectl get secret -n shop -l owner=helm,name=shop`; `helm list -n shop --all`.

**Fix →** Usually nothing left to do; clean up any remaining revision Secrets for that release and leftover resources.

**Prevent →** Avoid concurrent manual operations on release Secrets.

</details>

### Q100. Pods fail with `CreateContainerConfigError: secret "shop-db" not found` after moving secrets out of the chart into External Secrets. 🟡

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
