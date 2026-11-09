# Helm — Core Concepts

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

> Architecture, chart structure, templating, values, dependencies, hooks, releases, OCI, schema, Helmfile, Argo CD, secrets, best practices and rapid-fire commands · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Helm](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Helm 3 Architecture & Release Storage](#helm-3-architecture--release-storage) — Q1–Q25
- [Chart Structure & Metadata](#chart-structure--metadata) — Q26–Q50
- [Templating: Go Templates & Sprig](#templating-go-templates--sprig) — Q51–Q90
- [Values & Precedence](#values--precedence) — Q91–Q110
- [Dependencies, Subcharts & Library Charts](#dependencies-subcharts--library-charts) — Q111–Q130
- [Hooks, Tests & Validation](#hooks-tests--validation) — Q131–Q150
- [Release Lifecycle, Repositories & OCI](#release-lifecycle-repositories--oci) — Q151–Q172
- [Schema, Helmfile, Argo CD & Secrets](#schema-helmfile-argo-cd--secrets) — Q173–Q188
- [Best Practices & Rapid-Fire Commands](#best-practices--rapid-fire-commands) — Q189–Q200

---

## Helm 3 Architecture & Release Storage

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. What is Helm, in one sentence? 🟢

<details>
<summary><b>Answer</b></summary>

Helm is the package manager for Kubernetes: it bundles templated manifests and default configuration into a versioned **chart**, renders it with environment-specific values, and tracks each installation as a **release** with revision history so you can upgrade and roll back. It solves the "20 nearly identical YAML files per environment" problem and gives you a release unit you can version and promote.

</details>

### Q2. Define chart, release, revision and repository. 🟢

<details>
<summary><b>Answer</b></summary>

- **Chart:** a package (directory or `.tgz`) of templates, default values and metadata.
- **Release:** one installed instance of a chart in a namespace, under a unique name (`shop-prod`).
- **Revision:** a numbered version of a release; every install, upgrade or rollback adds one.
- **Repository:** where charts are published, either an HTTP repo with `index.yaml` or an OCI registry.

The same chart can back many releases: `helm install shop-dev ./shop -n dev` and `helm install shop-qa ./shop -n qa`.

</details>

### Q3. What was Tiller and why was it removed in Helm 3? 🟢

<details>
<summary><b>Answer</b></summary>

Tiller was Helm 2's in-cluster server component. The CLI sent charts to Tiller, which rendered and applied them using its own service account, usually with cluster-admin. Anyone who could reach Tiller's gRPC port could act as cluster-admin, and Helm's permissions were disconnected from Kubernetes RBAC.

Helm 3 removed it. The client renders locally and calls the API server with the user's kubeconfig credentials, so RBAC applies per user or per CI service account, and there is nothing extra running in the cluster.

</details>

### Q4. Where does Helm 3 store release information? 🟢

<details>
<summary><b>Answer</b></summary>

By default as Kubernetes **Secrets** in the release's namespace, one per revision, named `sh.helm.release.v1.<release>.v<revision>` with type `helm.sh/release.v1`.

```bash
kubectl get secrets -n shop -l owner=helm
# NAME                         TYPE                 DATA
# sh.helm.release.v1.shop.v1   helm.sh/release.v1   1
# sh.helm.release.v1.shop.v2   helm.sh/release.v1   1
```

Helm 2 stored them as ConfigMaps in `kube-system`. Helm 3 moved to Secrets because the record contains the merged values, which often include credentials.

</details>

### Q5. What is inside a release Secret and how do you decode it? 🔴

<details>
<summary><b>Answer</b></summary>

The `release` key holds a JSON document (chart metadata and templates, merged config values, rendered manifest, hooks, status, revision, timestamps) that Helm gzips and base64-encodes. Kubernetes then base64-encodes the Secret data again.

```bash
kubectl get secret sh.helm.release.v1.shop.v3 -n shop \
  -o jsonpath='{.data.release}' | base64 -d | base64 -d | gunzip | jq '.info.status, .config'
```

Labels on the Secret (`name`, `owner=helm`, `status`, `version`) let Helm query releases without decoding every record.

</details>

### Q6. What storage drivers does Helm support and how do you switch? 🔴

<details>
<summary><b>Answer</b></summary>

Set `HELM_DRIVER`:

- `secret` (default): Secrets in the release namespace.
- `configmap`: ConfigMaps, readable by more people, so avoid it if values contain secrets.
- `memory`: in-process only, used for tests and `helm template`-like flows.
- `sql`: PostgreSQL backend (`HELM_DRIVER_SQL_CONNECTION_STRING`), useful when release records exceed the 1 MiB object limit or you want central auditing.

```bash
export HELM_DRIVER=configmap
helm list -n shop
```

Every client touching a release must use the same driver, otherwise it will not see the release.

</details>

### Q7. Why does a release belong to a namespace in Helm 3? 🟢

<details>
<summary><b>Answer</b></summary>

The release record lives in the namespace you install into, so release names are unique per namespace, not cluster-wide. `shop` can exist in `dev` and `prod` independently. This also means `helm list` without `-n` or `-A` only shows the current context's namespace, which surprises people who "lost" a release. Cluster-scoped resources (ClusterRoles, CRDs) in a chart are still created cluster-wide, so two releases of the same chart can collide on them.

</details>

### Q8. What permissions does a CI service account need to run `helm upgrade --install`? 🔴

<details>
<summary><b>Answer</b></summary>

Exactly what the rendered manifests need, plus access to the release storage:

- `get`, `list`, `create`, `update`, `patch`, `delete` on Secrets in the namespace (release records).
- The same verbs on every kind the chart creates (Deployments, Services, Jobs, ConfigMaps, Ingresses, HPAs...).
- `watch` and `list` on Pods, ReplicaSets and Jobs if you use `--wait`.
- Cluster-scoped kinds (ClusterRole, CRD) only if the chart contains them.

A namespaced Role plus RoleBinding is usually enough for application charts, which is far tighter than Helm 2's cluster-admin Tiller.

</details>

### Q9. How does `helm upgrade` decide what to change? Explain the three-way merge. 🔴

<details>
<summary><b>Answer</b></summary>

Helm 3 compares three inputs for each object: the manifest from the **previous release**, the **live object** in the cluster, and the **new rendered manifest**. It builds a strategic merge patch (JSON merge patch for CRDs) that moves the live state to the new manifest while respecting changes made by others.

Consequences:
- A field you removed from the chart is removed from the live object, because it existed in the old manifest.
- A field someone added with `kubectl edit` that the chart never managed is usually kept.
- If someone scaled a Deployment manually and the chart value did not change, Helm 3 still resets `replicas` to the chart value because live differs from the new manifest.

Helm 2 only compared old vs new manifests, which ignored live drift entirely.

</details>

### Q10. What happens step by step when you run `helm install shop ./shop -n shop`? 🟡

<details>
<summary><b>Answer</b></summary>

1. Load the chart and dependencies from `charts/`; check `kubeVersion` and `Chart.yaml` validity.
2. Merge values (chart defaults, `-f`, `--set`) and validate against `values.schema.json`.
3. Check that the name is unused in the namespace; write the revision 1 record as `pending-install`.
4. Render templates with built-in objects, split hooks from regular manifests, sort by kind.
5. Install CRDs from `crds/`, run `pre-install` hooks in weight order and wait for them.
6. Create the resources; with `--wait`, wait until they are ready.
7. Run `post-install` hooks, mark the release `deployed`, print `NOTES.txt`.

Any failure marks the release `failed` (or uninstalls it with `--atomic`).

</details>

### Q11. In what order does Helm create resources of different kinds? 🟡

<details>
<summary><b>Answer</b></summary>

Helm sorts manifests by a built-in kind order so dependencies exist first: Namespace, NetworkPolicy, ResourceQuota, LimitRange, PodSecurityPolicy, PodDisruptionBudget, ServiceAccount, Secret, SecretList, ConfigMap, StorageClass, PersistentVolume, PersistentVolumeClaim, CustomResourceDefinition, ClusterRole, ClusterRoleBinding, Role, RoleBinding, Service, DaemonSet, Pod, ReplicationController, ReplicaSet, Deployment, HorizontalPodAutoscaler, StatefulSet, Job, CronJob, IngressClass, Ingress, APIService, and unknown kinds last. Uninstall goes in reverse.

This ordering is about API submission only; it does not wait for readiness between kinds. If the API needs the database ready, use probes, init containers or hooks.

</details>

### Q12. Does Helm 3 need anything installed in the cluster? 🟢

<details>
<summary><b>Answer</b></summary>

No. Helm 3 is a single static binary; it needs a kubeconfig context and RBAC permissions. Release records are ordinary Secrets created on demand. The only "in-cluster" artifacts are those records and whatever your charts create, which is why `helm version` prints only a client version in Helm 3:

```bash
helm version --short
# v3.17.3+ge4da497
```

</details>

### Q13. What is `.Capabilities` and when is it populated from the real cluster? 🟡

<details>
<summary><b>Answer</b></summary>

`.Capabilities` exposes the Kubernetes version (`.Capabilities.KubeVersion.Version`, `.Minor`) and available API versions (`.Capabilities.APIVersions.Has "policy/v1"`). During `install`, `upgrade` and `--dry-run=server` it comes from the live API server's discovery. During `helm template` or `--dry-run=client` Helm uses built-in defaults, unless you pass `--kube-version 1.30` and `--api-versions monitoring.coreos.com/v1`.

```yaml
{{- if .Capabilities.APIVersions.Has "monitoring.coreos.com/v1/ServiceMonitor" }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
{{- end }}
```

</details>

### Q14. What is the `lookup` function and what are its limits? 🔴

<details>
<summary><b>Answer</b></summary>

`lookup` queries the live cluster during rendering: `lookup "v1" "Secret" .Release.Namespace "shop-db"` returns the object as a dict, an empty dict if missing, or a list when the name is empty.

```yaml
{{- $existing := lookup "v1" "Secret" .Release.Namespace "shop-db" }}
password: {{ if $existing }}{{ index $existing.data "password" }}{{ else }}{{ randAlphaNum 24 | b64enc }}{{ end }}
```

Limits: it returns empty in `helm template` and `--dry-run=client` (works with `--dry-run=server`), it never works under Argo CD (which renders without cluster access), and it needs RBAC `get`/`list` on the looked-up kind. Charts that depend on it render differently offline, so keep it to "preserve an existing generated value" cases.

</details>

### Q15. How does Helm know which resources belong to a release? 🟡

<details>
<summary><b>Answer</b></summary>

From the manifest stored in the release record, not from labels. Helm also stamps every resource it manages with ownership metadata, which it checks before touching an existing object:

```yaml
metadata:
  labels:
    app.kubernetes.io/managed-by: Helm
  annotations:
    meta.helm.sh/release-name: shop
    meta.helm.sh/release-namespace: shop
```

If an object with the same kind/name exists without matching metadata, install or upgrade fails with `rendered manifests contain a resource that already exists ... invalid ownership metadata`.

</details>

### Q16. What happens to resources if you delete the release Secrets manually? 🟡

<details>
<summary><b>Answer</b></summary>

Nothing happens to the workloads; they keep running. Helm simply forgets the release: `helm list` no longer shows it and history and rollback are gone. Because the objects still carry `meta.helm.sh/release-name` and `meta.helm.sh/release-namespace` annotations, a new `helm install` with the **same** name and namespace adopts them and starts again at revision 1; a different name fails with `invalid ownership metadata`.

Deleting only the latest stuck `pending-upgrade` revision Secret is a known (last resort) recovery technique for a blocked release.

</details>

### Q17. What does `--history-max` do and why does it matter? 🟡

<details>
<summary><b>Answer</b></summary>

It limits how many revisions Helm keeps for a release; older revision Secrets are pruned on upgrade. `helm upgrade` defaults to 10. `0` means unlimited, which is a mistake for releases deployed many times a day: hundreds of Secrets slow down `helm list`, bloat etcd, and each can be close to 1 MiB for large charts.

```bash
helm upgrade --install shop ./shop -n shop --history-max 15
```

Keep enough revisions to cover your realistic rollback window.

</details>

### Q18. How is the Helm client configured locally? 🟢

<details>
<summary><b>Answer</b></summary>

Through the kubeconfig (`--kube-context`, `--kubeconfig`, `KUBECONFIG`) for cluster access, and through XDG paths for Helm's own state: repositories (`repositories.yaml`), repo index cache, registry credentials and plugins.

```bash
helm env | grep -E 'HELM_(CACHE|CONFIG|DATA)_HOME|HELM_REGISTRY_CONFIG|HELM_PLUGINS'
```

In CI we set `HELM_CACHE_HOME` and `HELM_CONFIG_HOME` to the workspace so jobs never share repo or registry credentials.

</details>

### Q19. Why do release names have a 53-character limit? 🟡

<details>
<summary><b>Answer</b></summary>

Kubernetes resource names (and label values) are limited to 63 characters, and charts commonly build names like `<release>-<chart>` or `<release>-api`. Helm caps release names at 53 to leave room for suffixes. Charts still `trunc 63 | trimSuffix "-"` generated names, because a 53-character release plus `-frontend` already overflows. Exceeding it fails with `release name "..." exceeds max length of 53`.

</details>

### Q20. What is the difference between `helm template`, `--dry-run=client` and `--dry-run=server`? 🔴

<details>
<summary><b>Answer</b></summary>

- `helm template`: renders entirely offline, no cluster access, no release checks, `lookup` empty, `.Capabilities` defaulted.
- `helm install/upgrade --dry-run` (`=client`): renders through the install/upgrade code path, checks the release state, prints the release and manifests, but does not contact the API server for validation; `lookup` is still empty.
- `--dry-run=server` (Helm 3.13+): sends the objects to the API server in dry-run mode, so admission webhooks, schema validation and `lookup` all work, yet nothing is persisted.

Use `template` in PR checks without cluster credentials, `--dry-run=server` right before a production deploy.

</details>

### Q21. How does Helm handle CRDs, and why is it controversial? 🔴

<details>
<summary><b>Answer</b></summary>

CRDs placed in the chart's `crds/` directory are installed on `helm install` before anything else, are not templated, and are **never upgraded or deleted** by Helm. Skipping them is `--skip-crds`. This is deliberate: deleting a CRD deletes every custom resource of that type cluster-wide, and two releases may share one CRD.

The trade-off is that CRD schema changes in a new chart version are silently ignored. In production we either apply CRDs separately (`kubectl apply --server-side -f crds/`) as a pipeline step, use a dedicated CRD chart in `templates/` with `helm.sh/resource-policy: keep`, or let Argo CD manage them with server-side apply.

</details>

### Q22. What is `helm.sh/resource-policy: keep`? 🟡

<details>
<summary><b>Answer</b></summary>

An annotation that tells Helm not to delete the resource during `helm uninstall`, or when it disappears from the chart during an upgrade. The object becomes orphaned and must be deleted by hand.

```yaml
metadata:
  annotations:
    helm.sh/resource-policy: keep
```

We use it on PersistentVolumeClaims and on generated Secrets (database passwords) so an accidental uninstall does not destroy data. Remember that reinstalling will then hit "resource already exists" unless the ownership annotations still match.

</details>

### Q23. What are post-renderers? 🔴

<details>
<summary><b>Answer</b></summary>

A post-renderer is an executable that receives the fully rendered manifests on stdin and must print modified manifests on stdout before Helm applies them. It is the supported way to patch third-party charts without forking them, typically with Kustomize.

```bash
cat > kustomize-post.sh <<'SH'
#!/bin/sh
cat > base.yaml
kustomize build .
SH
chmod +x kustomize-post.sh
helm upgrade --install ingress ingress-nginx/ingress-nginx --post-renderer ./kustomize-post.sh
```

The release record stores the post-rendered manifest, so diffs and rollbacks see the real applied objects. In Helm 4, post-renderers are implemented as plugins.

</details>

### Q24. How does Helm 3 compare with plain Kustomize? 🟡

<details>
<summary><b>Answer</b></summary>

- **Helm:** templating with logic, parameterized by values, packaged and versioned charts, dependency management, release tracking with history and rollback, hooks. Best for distributing reusable software (third-party or internal platform charts).
- **Kustomize:** template-free overlays and patches on plain YAML, built into `kubectl -k`, no release state. Best for environment-specific tweaks of your own manifests.

Many teams use both: Helm charts for packaging, Kustomize as a post-renderer or Argo CD layer for last-mile patches.

</details>

### Q25. What changed in Helm 4 that an interviewer might ask about? 🔴

<details>
<summary><b>Answer</b></summary>

Helm 4 shipped in November 2025. Key points:

- Existing `apiVersion: v2` charts install unchanged; Helm 3 remains widely deployed and supported for a transition period.
- Server-side apply is used for new releases, which improves field ownership and conflict detection with controllers and other tools.
- Readiness waiting is based on kstatus.
- A redesigned plugin system (including WebAssembly plugins); post-renderers become plugins.
- Several flags were renamed, for example `--atomic` to `--rollback-on-failure` and `--force` to `--force-replace`, with the old names deprecated.

In an interview, say which major version your team runs and why, and that CI scripts should pin the Helm version.

</details>

---

## Chart Structure & Metadata

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. What files does `helm create mychart` generate and which ones matter? 🟢

<details>
<summary><b>Answer</b></summary>

`Chart.yaml`, `values.yaml`, `.helmignore`, `charts/` (empty), and `templates/` containing `deployment.yaml`, `service.yaml`, `serviceaccount.yaml`, `ingress.yaml`, `hpa.yaml`, `NOTES.txt`, `_helpers.tpl` and `tests/test-connection.yaml`.

The scaffold is a good starting point for helpers and labels, but in production we delete what we do not need (the generated ingress and HPA logic is generic) and add `values.schema.json`, a PDB and our own probes and security context.

</details>

### Q27. What are the required fields in `Chart.yaml`? 🟢

<details>
<summary><b>Answer</b></summary>

`apiVersion`, `name` and `version`. For Helm 3 charts `apiVersion` is `v2`; `v1` charts (Helm 2 format, with `requirements.yaml`) still install but should be migrated.

```yaml
apiVersion: v2
name: shop
version: 1.4.0          # chart SemVer, required
appVersion: "2.3.1"     # app version, quoted string
type: application
description: Shack Shop 3-tier application
kubeVersion: ">=1.27.0-0"
```

`helm lint` fails with `[ERROR] Chart.yaml: version is required` if `version` is missing.

</details>

### Q28. What is the difference between `version` and `appVersion`? 🟢

<details>
<summary><b>Answer</b></summary>

`version` is the chart package's SemVer 2 version. It is used for repository lookups, `--version` constraints, the package file name (`shop-1.4.0.tgz`) and the OCI tag. It must change whenever chart content changes.

`appVersion` describes the application the chart deploys. It is informational, any string, and commonly the default image tag. A template-only fix bumps `version` from 1.4.0 to 1.4.1 and leaves `appVersion` at `2.3.1`; a new app release usually bumps both.

</details>

### Q29. Why should `appVersion` be quoted in `Chart.yaml`? 🟢

<details>
<summary><b>Answer</b></summary>

YAML parses unquoted `1.10` as the float `1.1`, and `1.0` as `1`. The image tag rendered from `.Chart.AppVersion` would then be wrong (`app:1.1` instead of `app:1.10`). Always write `appVersion: "1.10"`. The same rule applies to image tags in `values.yaml`, and it is why we pipe tags through `quote` or `toString` in templates.

</details>

### Q30. What does `type: library` vs `type: application` mean? 🟢

<details>
<summary><b>Answer</b></summary>

`application` (the default) is a normal installable chart. `library` charts contain only named templates (`define` blocks) and no rendered manifests; Helm refuses to install them directly (`Error: library charts are not installable`). Application charts list a library chart as a dependency and call its templates with `include`. Libraries are how platform teams share consistent labels, Deployment skeletons and security defaults across dozens of service charts.

</details>

### Q31. What is the `kubeVersion` field used for? 🟡

<details>
<summary><b>Answer</b></summary>

A SemVer constraint on the target cluster's Kubernetes version. Helm checks it before rendering and fails early instead of half-installing with unsupported APIs.

```yaml
kubeVersion: ">=1.27.0-0"
```

The `-0` suffix matters: managed clusters report versions like `v1.30.4-eks-a737599`, which SemVer treats as a pre-release, and a constraint without `-0` excludes pre-releases. Failure looks like `chart requires kubeVersion: >=1.27.0 which is incompatible with Kubernetes v1.30.4-eks-a737599`.

</details>

### Q32. What other optional fields can `Chart.yaml` contain? 🟢

<details>
<summary><b>Answer</b></summary>

`description`, `keywords`, `home`, `sources`, `maintainers` (name, email, url), `icon`, `deprecated: true`, `annotations` (free-form, used by Artifact Hub, for example `artifacthub.io/changes`), and `dependencies`. These drive search results and catalogs. Mark old charts `deprecated: true` so `helm search` and Artifact Hub warn users.

</details>

### Q33. What does the `templates/` directory contain and how are files treated? 🟢

<details>
<summary><b>Answer</b></summary>

Every file in `templates/` is rendered through the template engine. Files producing YAML are sent to Kubernetes; a file can contain multiple documents separated by `---`. Exceptions:

- Files starting with `_` (like `_helpers.tpl`) are not emitted as manifests; they hold `define` blocks.
- `NOTES.txt` is rendered and printed to the user after install or upgrade.
- Files that render to only whitespace or comments are skipped, which is how `{{- if .Values.ingress.enabled }}` wrappers work.

Subdirectories such as `templates/tests/` are rendered too.

</details>

### Q34. What is `_helpers.tpl` and what usually goes in it? 🟢

<details>
<summary><b>Answer</b></summary>

A conventional file of named templates that keep names and labels consistent. Typical helpers: `<chart>.name`, `<chart>.fullname`, `<chart>.chart`, `<chart>.labels`, `<chart>.selectorLabels`, `<chart>.serviceAccountName`, and an image reference helper.

```yaml
{{- define "shop.selectorLabels" -}}
app.kubernetes.io/name: {{ include "shop.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

The underscore prefix stops Helm rendering it as a manifest; the `.tpl` extension is convention only.

</details>

### Q35. How does the standard `fullname` helper work and why is it so convoluted? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
{{- define "shop.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}
```

It lets users force an exact name (`fullnameOverride`), avoids stutter like `shop-shop` when the release name already contains the chart name, and stays within the 63-character DNS label limit without a trailing hyphen.

</details>

### Q36. What is `NOTES.txt` for? 🟢

<details>
<summary><b>Answer</b></summary>

A template rendered after `install` and `upgrade` and printed to the terminal, typically with how to reach the app: the Ingress host, a `kubectl port-forward` command, or how to fetch a generated password. It has full access to values and built-ins. Retrieve it later with `helm get notes shop -n shop`. Keep it short and accurate; a stale NOTES with wrong commands is worse than none.

</details>

### Q37. What is `.helmignore`? 🟢

<details>
<summary><b>Answer</b></summary>

A `.gitignore`-style file listing paths excluded when Helm packages or loads the chart: `.git/`, CI files, `*.swp`, local `values-*.local.yaml`, test fixtures. Excluded files are also invisible to `.Files.Get`. It keeps packages small and stops accidental inclusion of secrets lying around the chart directory. Charts over the size limit of a release Secret (1 MiB after compression) often contain files that should have been ignored.

</details>

### Q38. What is the `charts/` directory? 🟢

<details>
<summary><b>Answer</b></summary>

It holds the chart's dependencies, either as packaged `.tgz` archives fetched by `helm dependency update`/`build`, or as unpacked chart directories you copied in manually. Everything in `charts/` is treated as a subchart. In Git we usually do not commit the `.tgz` files; CI runs `helm dependency build` from `Chart.lock` instead, which keeps the repo clean and reproducible.

</details>

### Q39. What does the `crds/` directory do differently from `templates/`? 🟡

<details>
<summary><b>Answer</b></summary>

Files in `crds/` are plain YAML (no templating), installed before anything else on first install, and ignored on upgrade and uninstall. They are not part of the release manifest, so `helm get manifest` does not show them. Use `helm show crds <chart>` to view them and `--skip-crds` to skip them. If a chart needs to template a CRD (for example with a conversion webhook CA), it must go in `templates/` and accept the upgrade and deletion risks.

</details>

### Q40. How do you include non-template files such as config files in a chart? 🟡

<details>
<summary><b>Answer</b></summary>

Place them anywhere in the chart outside `templates/` (for example `files/` or `config/`) and read them with `.Files`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "shop.fullname" . }}-nginx
data:
  nginx.conf: |-
{{ .Files.Get "files/nginx.conf" | indent 4 }}
```

`.Files` cannot read files in `templates/` or files excluded by `.helmignore`. Files in `files/` are not rendered, so if they need values, wrap with `tpl (.Files.Get "files/nginx.conf") .`.

</details>

### Q41. What do `.Files.Glob`, `.AsConfig` and `.AsSecrets` do? 🟡

<details>
<summary><b>Answer</b></summary>

`.Files.Glob` returns all files matching a pattern; `.AsConfig` turns them into ConfigMap `data` entries and `.AsSecrets` into base64-encoded Secret `data`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "shop.fullname" . }}-migrations
data:
{{ (.Files.Glob "migrations/*.sql").AsConfig | indent 2 }}
```

Each file name becomes a key. This is how the hands-on lab ships SQL migrations into a hook Job.

</details>

### Q42. Which built-in objects are available in templates? 🟢

<details>
<summary><b>Answer</b></summary>

- `.Values`: merged values.
- `.Release`: `Name`, `Namespace`, `Revision`, `IsInstall`, `IsUpgrade`, `Service` (always `Helm`).
- `.Chart`: contents of `Chart.yaml` (`.Chart.Name`, `.Chart.Version`, `.Chart.AppVersion`).
- `.Capabilities`: Kubernetes and API versions, Helm version.
- `.Files`: access to non-template files.
- `.Template`: `Name` (current file path) and `BasePath`.
- `.Subcharts`: access to subchart scopes (Helm 3.13+).

Field names on built-ins are capitalized; values keys keep whatever case you wrote.

</details>

### Q43. When would you use `.Release.IsInstall` or `.Release.Revision`? 🟡

<details>
<summary><b>Answer</b></summary>

`.Release.IsInstall` and `.Release.IsUpgrade` let a template behave differently on first install, for example generating an initial admin password only on install. `.Release.Revision` changes on every upgrade; putting it in a pod annotation forces a rollout on every deploy:

```yaml
annotations:
  rollme: {{ .Release.Revision | quote }}
```

That is usually an anti-pattern (needless restarts); prefer a checksum of the actual config. Neither works reliably under Argo CD, which always renders as an install.

</details>

### Q44. What are the recommended labels for chart resources? 🟡

<details>
<summary><b>Answer</b></summary>

The Kubernetes recommended labels:

```yaml
app.kubernetes.io/name: shop
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/component: api
app.kubernetes.io/part-of: shack-shop
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
```

Only the stable ones (`name`, `instance`, `component`) go into selectors, because Deployment selectors are immutable.

</details>

### Q45. Why must Deployment selector labels never include the chart version or appVersion? 🔴

<details>
<summary><b>Answer</b></summary>

`spec.selector` of a Deployment, StatefulSet and DaemonSet is immutable. If the selector includes `helm.sh/chart: shop-1.4.0` or `app.kubernetes.io/version`, the next chart bump produces a different selector and the upgrade fails:

```text
Error: UPGRADE FAILED: cannot patch "shop-api" with kind Deployment: Deployment.apps "shop-api" is invalid: spec.selector: Invalid value: ...: field is immutable
```

That is why charts have separate `labels` (all labels, on metadata) and `selectorLabels` (only name, instance, component) helpers. Fixing it after the fact requires deleting the Deployment (with `--cascade=orphan` to keep pods) and re-running the upgrade.

</details>

### Q46. Why does the `helm.sh/chart` label replace `+` with `_`? 🟡

<details>
<summary><b>Answer</b></summary>

SemVer allows build metadata (`1.4.0+build.17`), but `+` is not a valid character in Kubernetes label values (allowed: alphanumerics, `-`, `_`, `.`, max 63 chars). Without the replace, a chart version with build metadata makes every resource fail validation with `metadata.labels: Invalid value: "shop-1.4.0+build.17"`. OCI tags have the same restriction, so `helm push` converts `+` to `_` in the tag.

</details>

### Q47. What does `helm show` expose about a chart? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
helm show chart  bitnami/postgresql            # Chart.yaml
helm show values bitnami/postgresql > pg.yaml  # default values.yaml
helm show readme bitnami/postgresql
helm show crds   prometheus-community/kube-prometheus-stack
helm show all    oci://ghcr.io/acme/charts/shop --version 1.4.0
```

`helm show values` is the first command we run before installing any third-party chart: it is the chart's API, and we copy only the keys we change into our own values file.

</details>

### Q48. How do you document a chart's values so users can use it? 🟡

<details>
<summary><b>Answer</b></summary>

Comment every key in `values.yaml` (purpose, type, default, example), ship a `README.md` with a values table, and enforce types with `values.schema.json`. Many teams generate the README table from comments using `helm-docs` with `# --` annotations:

```yaml
# -- Number of API replicas. Ignored when autoscaling.enabled is true.
replicaCount: 2
```

Treat `values.yaml` as a public interface: renaming a key is a breaking change that deserves a major chart version.

</details>

### Q49. How should chart versions follow SemVer in practice? 🟡

<details>
<summary><b>Answer</b></summary>

- **Patch** (1.4.0 to 1.4.1): bug fixes in templates, no values changes.
- **Minor** (1.4.x to 1.5.0): new optional values or features, backward compatible.
- **Major** (1.x to 2.0.0): removed or renamed values, changed selectors or resource names, anything that requires user action or causes recreation.

Never republish the same version with different content: caches, Argo CD and OCI digests make it non-deterministic which content a cluster gets. CI should fail the build if the version already exists in the registry.

</details>

### Q50. What makes a chart "production grade" structurally? 🔴

<details>
<summary><b>Answer</b></summary>

- Consistent naming and label helpers, stable selectors, `fullnameOverride` support.
- Commented `values.yaml` with safe defaults (resources set, non-root, read-only root FS, probes enabled).
- `values.schema.json` with `required` fields and types.
- Optional components behind `enabled` flags; no hardcoded namespaces (use `.Release.Namespace`).
- Checksum annotations so config changes roll pods.
- PDB, HPA, topology spread and ServiceAccount templates.
- `helm test` hooks, `NOTES.txt`, and CI that lints, renders against each environment's values and validates with kubeconform.
- SemVer discipline and published to an OCI registry, signed.

</details>

---

## Templating: Go Templates & Sprig

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. What template language does Helm use? 🟢

<details>
<summary><b>Answer</b></summary>

Go's `text/template` package, extended with the **Sprig** function library (string, math, list, dict, crypto, date functions) and Helm-specific functions (`include`, `tpl`, `required`, `toYaml`, `fromYaml`, `toJson`, `lookup`). Actions are written between `{{ }}`; everything else is copied verbatim. Helm templates are text templating, not YAML-aware, so indentation mistakes produce YAML parse errors only after rendering.

</details>

### Q52. What does the dot (`.`) mean in a Helm template? 🟢

<details>
<summary><b>Answer</b></summary>

The current scope (context). At the top of a template file it is the root object containing `.Values`, `.Release`, `.Chart` and so on. Inside `with` or `range` it changes to the selected value or the current element. When you `include "shop.labels" .` you pass the current scope to the named template, which then sees it as its own `.`.

</details>

### Q53. What is `$` and when do you need it? 🟡

<details>
<summary><b>Answer</b></summary>

`$` is always bound to the root context passed to the template, no matter how deeply `with`/`range` changed `.`. You need it to reach `.Release` or `.Values` from inside a loop:

```yaml
{{- range .Values.api.extraEnv }}
- name: {{ .name }}
  value: {{ .value | quote }}
- name: {{ .name }}_RELEASE
  value: {{ $.Release.Name }}
{{- end }}
```

Writing `.Release.Name` inside the `range` fails with `can't evaluate field Release in type map[string]interface {}`.

</details>

### Q54. How does whitespace control (`{{-` and `-}}`) work? 🟢

<details>
<summary><b>Answer</b></summary>

`{{-` trims all whitespace, including newlines, immediately before the action; `-}}` trims whitespace after it. Use them on control-flow lines (`if`, `range`, `end`, `define`) so they do not leave blank lines or break indentation.

```yaml
metadata:
  labels:
    {{- include "shop.labels" . | nindent 4 }}
```

Over-trimming is a classic bug: `{{- .Values.x -}}` can glue two lines together and produce `key: valueother: x`. Check with `helm template` when in doubt.

</details>

### Q55. How do `if`, `else if` and `else` evaluate truthiness? 🟢

<details>
<summary><b>Answer</b></summary>

A pipeline is false if it is `false`, `0`, `nil`, an empty string, or an empty map, list or slice. Everything else is true.

```yaml
{{- if .Values.ingress.enabled }}
...
{{- else if .Values.route.enabled }}
...
{{- end }}
```

Watch for numeric zero: `if .Values.replicaCount` is false for `0`, which may be a legitimate value (scale to zero). Use `hasKey` or `kindIs "invalid"` to distinguish "unset" from "zero".

</details>

### Q56. What do `and`, `or`, `not`, `eq`, `ne`, `lt`, `gt` look like in Helm? 🟢

<details>
<summary><b>Answer</b></summary>

They are prefix functions, not infix operators:

```yaml
{{- if and .Values.autoscaling.enabled (gt (int .Values.autoscaling.maxReplicas) 1) }}
{{- if or (eq .Values.env "prod") (eq .Values.env "staging") }}
{{- if not .Values.serviceAccount.create }}
{{- if ne .Values.service.type "ClusterIP" }}
```

Comparisons are type-strict: `eq 1 "1"` errors or is false, and values from YAML are often `float64`, so convert with `int` before comparing numbers.

</details>

### Q57. How does `with` work, and what is its common pitfall? 🟡

<details>
<summary><b>Answer</b></summary>

`with` sets `.` to the value if it is non-empty and skips the block otherwise. It is ideal for optional blocks:

```yaml
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

Pitfall: inside the block you cannot reach `.Values` or `.Release` through `.`; use `$`. Also, a value of `0` or `false` skips the block.

</details>

### Q58. How do you loop over a list and over a map with `range`? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
# list
{{- range $i, $host := .Values.ingress.hosts }}
- host: {{ $host | quote }}   # $i is the index
{{- end }}

# map (keys iterate in sorted order)
env:
{{- range $key, $val := .Values.api.env }}
  - name: {{ $key }}
    value: {{ $val | quote }}
{{- end }}
```

Maps iterate in sorted key order, so output is deterministic. Generate a numeric sequence with `until 3` or `untilStep 0 10 2`.

</details>

### Q59. How do you declare and reassign variables in templates? 🟡

<details>
<summary><b>Answer</b></summary>

`:=` declares, `=` reassigns (Go 1.11+ templates):

```yaml
{{- $fullName := include "shop.fullname" . -}}
{{- $port := 3000 -}}
{{- if .Values.api.port }}{{ $port = .Values.api.port }}{{ end -}}
```

Variables are scoped to the block where they are declared (`if`, `range`, `with`, `define`). A variable declared inside an `if` is not visible after its `end`, so declare first, then reassign inside the block.

</details>

### Q60. What does `define` do? 🟢

<details>
<summary><b>Answer</b></summary>

`define` creates a named template that renders nothing on its own:

```yaml
{{- define "shop.image" -}}
{{- printf "%s:%s" .repository (.tag | default "latest" | toString) -}}
{{- end }}
```

You render it with `include` or `template`. Names are global across the parent chart and all subcharts, and a later definition with the same name silently overrides an earlier one, which is why names are always prefixed with the chart name.

</details>

### Q61. What is the difference between `include` and `template`? 🟡

<details>
<summary><b>Answer</b></summary>

`template` is a Go template action: it writes output directly and cannot be used in a pipeline. `include` is a Helm function: it returns the rendered string, so you can pipe it into `nindent`, `indent`, `quote`, `sha256sum` or assign it to a variable.

```yaml
metadata:
  labels:
    {{- include "shop.labels" . | nindent 4 }}   # works
    {{ template "shop.labels" . }}               # no way to indent multi-line output
```

Rule in production charts: always use `include`.

</details>

### Q62. Why does `nindent` exist alongside `indent`? 🟢

<details>
<summary><b>Answer</b></summary>

`indent N` prefixes every line with N spaces. `nindent N` does the same and adds a leading newline. This lets you put the action on the parent key's line and keep the template readable:

```yaml
resources:
  {{- toYaml .Values.resources | nindent 2 }}
```

Combined with `{{-`, which removes the preceding newline and spaces, `nindent` restores exactly one newline and the right indentation. With `indent` you must start the action at column 0 to avoid doubled indentation.

</details>

### Q63. What does `toYaml` do and when do you use it? 🟢

<details>
<summary><b>Answer</b></summary>

`toYaml` serializes a value (map, list, scalar) as YAML text. It is the standard way to pass through whole blocks the chart does not want to model field by field: `resources`, `securityContext`, `affinity`, `tolerations`, `nodeSelector`, extra volumes.

```yaml
securityContext:
  {{- toYaml .Values.api.securityContext | nindent 12 }}
```

It also handles quoting of special strings correctly, unlike hand-written templates.

</details>

### Q64. What are `toJson`, `fromYaml`, `fromJson` and `fromYamlArray` used for? 🔴

<details>
<summary><b>Answer</b></summary>

- `toJson`: emit compact JSON, for example inside an annotation or a JSON config file.
- `fromYaml` / `fromJson`: parse a string into a dict, often the output of `include` or `.Files.Get`, so you can merge or inspect it.
- `fromYamlArray` / `fromJsonArray`: parse into a list.

```yaml
{{- $base := include "shop.podDefaults" . | fromYaml }}
{{- $merged := mergeOverwrite $base .Values.api.podOverrides }}
{{- toYaml $merged | nindent 6 }}
```

If parsing fails, `fromYaml` returns a dict with an `Error` key instead of failing, so check it in library code.

</details>

### Q65. How does `default` work and what is its trap? 🟡

<details>
<summary><b>Answer</b></summary>

`default DEFAULT VALUE` returns `VALUE` unless it is empty, in which case it returns `DEFAULT`:

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

Trap: "empty" includes `false`, `0` and `""`. `default true .Values.enabled` always yields `true`, even when the user set `enabled: false`. For booleans, use `hasKey`, `ternary`, or put the default in `values.yaml` instead.

</details>

### Q66. What does `required` do? 🟢

<details>
<summary><b>Answer</b></summary>

`required "message" VALUE` fails rendering with your message if the value is empty:

```yaml
password: {{ required "postgres.auth.password is required (use --set-string or a secrets file)" .Values.postgres.auth.password | b64enc | quote }}
```

The user sees `Error: execution error at (shop/templates/secret.yaml:9:15): postgres.auth.password is required ...`. It is better than a broken deployment with an empty password; for type checks across many fields, add `values.schema.json` too.

</details>

### Q67. When would you use `fail`? 🟡

<details>
<summary><b>Answer</b></summary>

To reject invalid combinations that a single `required` cannot express:

```yaml
{{- if and .Values.autoscaling.enabled .Values.api.replicaCount }}
{{- if gt (int .Values.api.replicaCount) (int .Values.autoscaling.maxReplicas) }}
{{- fail "api.replicaCount cannot exceed autoscaling.maxReplicas" }}
{{- end }}
{{- end }}
{{- if and (eq .Values.env "prod") (not .Values.postgres.persistence.enabled) }}
{{- fail "persistence must be enabled in prod" }}
{{- end }}
```

Put such validations in a `_validate.tpl` helper included from one template so they run once.

</details>

### Q68. What does `tpl` do and why is it powerful? 🔴

<details>
<summary><b>Answer</b></summary>

`tpl STRING CONTEXT` renders a string as a template at runtime. It lets values themselves contain template expressions:

```yaml
# values.yaml
api:
  env:
    DB_HOST: "{{ .Release.Name }}-postgres"
# template
value: {{ tpl .Values.api.env.DB_HOST . | quote }}
```

It is also used to template external files: `tpl (.Files.Get "files/app.conf") .`. Power cost: users can execute arbitrary template logic through values, rendering is slower, and errors point to `tpl` rather than a file line. Use it only for values documented as templatable.

</details>

### Q69. How do you quote values safely? 🟢

<details>
<summary><b>Answer</b></summary>

Use `quote` (double quotes, with escaping) or `squote`:

```yaml
env:
  - name: LOG_LEVEL
    value: {{ .Values.logLevel | quote }}
  - name: FEATURE_FLAG
    value: {{ .Values.featureFlag | quote }}   # true -> "true", required for env values
```

Environment variable values and annotations must be strings; an unquoted `true` or `8080` fails with `cannot unmarshal bool into Go struct field EnvVar.value of type string`. Do not quote fields that must be integers, such as `containerPort` or `replicas`.

</details>

### Q70. Which Sprig string functions do you use most in charts? 🟢

<details>
<summary><b>Answer</b></summary>

`trunc`, `trimSuffix`, `trimPrefix`, `trim`, `lower`, `upper`, `replace`, `contains`, `hasPrefix`, `hasSuffix`, `printf`, `quote`, `nospace`, `regexReplaceAll`, `splitList`, `join`, `b64enc`, `b64dec`, `sha256sum`, `randAlphaNum`.

```yaml
name: {{ printf "%s-%s" (include "shop.fullname" .) "api" | trunc 63 | trimSuffix "-" }}
host: {{ .Values.ingress.host | lower | quote }}
```

</details>

### Q71. How do you work with lists and dicts in templates? 🟡

<details>
<summary><b>Answer</b></summary>

Sprig gives you `list`, `append`, `first`, `last`, `has`, `uniq`, `compact`, `without`, `sortAlpha` for lists, and `dict`, `get`, `set`, `unset`, `hasKey`, `keys`, `values`, `pick`, `omit`, `merge`, `mergeOverwrite`, `deepCopy` for dicts.

```yaml
{{- $ctx := dict "root" $ "component" "api" "port" 3000 }}
{{- include "shop.deployment" $ctx }}
{{- if has "debug" .Values.api.flags }}...{{ end }}
```

`set` and `unset` mutate the dict in place and return it; use `deepCopy` before modifying `.Values` sub-maps, otherwise you change them for every later template.

</details>

### Q72. How do you pass multiple parameters to a named template? 🔴

<details>
<summary><b>Answer</b></summary>

A named template receives exactly one argument, so build a dict that carries the root context and your extra parameters:

```yaml
{{- define "shop.service" -}}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "shop.fullname" .root }}-{{ .component }}
  labels:
    {{- include "shop.labels" .root | nindent 4 }}
spec:
  ports:
    - port: {{ .port }}
      targetPort: http
  selector:
    {{- include "shop.selectorLabels" .root | nindent 4 }}
    app.kubernetes.io/component: {{ .component }}
{{- end }}

{{ include "shop.service" (dict "root" . "component" "api" "port" 3000) }}
```

Forgetting to pass the root is the cause of `can't evaluate field Release in type string`.

</details>

### Q73. What is the difference between `merge` and `mergeOverwrite`? 🔴

<details>
<summary><b>Answer</b></summary>

Both deep-merge dicts into the first argument (mutating it). `merge $dst $src` keeps existing keys in `$dst` and only fills missing ones, so precedence goes to the left. `mergeOverwrite $dst $src` lets `$src` overwrite, so precedence goes to the right.

```yaml
{{- $defaults := dict "replicas" 1 "port" 3000 }}
{{- $final := mergeOverwrite (deepCopy $defaults) .Values.api }}
```

Lists are replaced, not merged. Always `deepCopy` the destination when it is part of `.Values`, otherwise you mutate global state.

</details>

### Q74. How do you add a checksum annotation so pods restart when a ConfigMap changes? 🟡

<details>
<summary><b>Answer</b></summary>

Changing a ConfigMap does not restart pods that consume it via `envFrom` or files mounted as env. Hash the rendered ConfigMap into the pod template so any change alters the template and triggers a rollout:

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/frontend-configmap.yaml") . | sha256sum }}
```

Use the same trick for Secrets the chart renders. Do not hash an externally managed Secret this way; it is not in the chart.

</details>

### Q75. How do you generate a random password that does not change on every upgrade? 🔴

<details>
<summary><b>Answer</b></summary>

`randAlphaNum 24` produces a new value on every render, so each `helm upgrade` would rotate the password and break the app. Reuse the existing value via `lookup`:

```yaml
{{- $secret := lookup "v1" "Secret" .Release.Namespace (printf "%s-db" (include "shop.fullname" .)) }}
{{- $pw := "" }}
{{- if $secret }}
{{- $pw = index $secret.data "password" | b64dec }}
{{- else }}
{{- $pw = .Values.db.password | default (randAlphaNum 24) }}
{{- end }}
data:
  password: {{ $pw | b64enc | quote }}
```

Add `helm.sh/resource-policy: keep`. Under Argo CD or `helm template`, `lookup` is empty, so the value churns; there, generate secrets outside Helm (External Secrets, SealedSecrets).

</details>

### Q76. How can a template check whether an API version exists in the cluster? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
{{- if .Capabilities.APIVersions.Has "autoscaling/v2" }}
apiVersion: autoscaling/v2
{{- else }}
apiVersion: autoscaling/v2beta2
{{- end }}
kind: HorizontalPodAutoscaler
```

You can test a specific kind too: `.Capabilities.APIVersions.Has "monitoring.coreos.com/v1/ServiceMonitor"`. Remember that `helm template` uses defaults unless you pass `--api-versions`, so CI renders should set it to match the target cluster.

</details>

### Q77. How do you compare Kubernetes versions in a template? 🟡

<details>
<summary><b>Answer</b></summary>

Use `semverCompare` against `.Capabilities.KubeVersion.Version` (or `.GitVersion`):

```yaml
{{- if semverCompare ">=1.25-0" .Capabilities.KubeVersion.Version }}
apiVersion: policy/v1
{{- else }}
apiVersion: policy/v1beta1
{{- end }}
kind: PodDisruptionBudget
```

The `-0` makes the constraint match provider pre-release suffixes like `-eks-` or `-gke.`.

</details>

### Q78. What do `ternary`, `coalesce` and `empty` do? 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
pullPolicy: {{ ternary "Always" "IfNotPresent" .Values.dev | quote }}       # true-value false-value condition
registry:   {{ coalesce .Values.image.registry .Values.global.imageRegistry "docker.io" }}  # first non-empty
{{- if empty .Values.api.extraEnv }}# nothing to add{{ end }}
```

`coalesce` is the cleanest way to implement "local value, else global value, else default" in subchart-friendly charts.

</details>

### Q79. How do you render a value only when the key exists, even if it is `false` or `0`? 🔴

<details>
<summary><b>Answer</b></summary>

Use `hasKey` on the parent map, because `if` and `default` treat `false` and `0` as unset:

```yaml
{{- if hasKey .Values.api "automountServiceAccountToken" }}
automountServiceAccountToken: {{ .Values.api.automountServiceAccountToken }}
{{- end }}
{{- if hasKey .Values.api "replicaCount" }}
replicas: {{ .Values.api.replicaCount }}
{{- end }}
```

The second example is important when HPA manages replicas: omitting `replicas` stops Helm from resetting the HPA's scaling on every upgrade.

</details>

### Q80. Why do numbers from values sometimes render as `1e+06` or `3000.0`? 🔴

<details>
<summary><b>Answer</b></summary>

YAML numbers are parsed into `float64` in Go maps. Large integers then format in scientific notation, and arithmetic results may carry decimals. Convert explicitly:

```yaml
maxBytes: {{ .Values.maxBytes | int64 }}   # 1000000 instead of 1e+06
port: {{ .Values.port | int }}
```

Use `int`, `int64` or `float64` before `printf "%d"`, comparisons and math (`add`, `mul`, `div`). For identifiers like account IDs, keep them as quoted strings in values.

</details>

### Q81. How do you base64-encode values for a Secret, and when should you use `stringData`? 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: v1
kind: Secret
type: Opaque
data:
  password: {{ .Values.db.password | b64enc | quote }}
# or
stringData:
  password: {{ .Values.db.password | quote }}
```

`stringData` is simpler and avoids double encoding mistakes; the API server converts it to `data`. Either way, the plain value is stored in the Helm release Secret (merged values), so anyone who can read Secrets in the namespace can read it. Base64 is encoding, not encryption.

</details>

### Q82. How do you include a whole YAML block from values only if it is set? 🟢

<details>
<summary><b>Answer</b></summary>

Combine `with`, `toYaml` and `nindent`:

```yaml
      {{- with .Values.api.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.api.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

Empty lists and maps skip the block entirely, so the rendered manifest stays clean and the server-side default applies.

</details>

### Q83. How do comments work in templates, and which kind ends up in the manifest? 🟢

<details>
<summary><b>Answer</b></summary>

- `{{/* This is a template comment */}}` is removed during rendering (use `{{- /* ... */}}` to also trim whitespace).
- `# This is a YAML comment` stays in the rendered output and is visible in `helm get manifest`.

Use template comments to document helper parameters and YAML comments only when the reader of the rendered manifest benefits. Never put secrets or internal notes in YAML comments.

</details>

### Q84. What is `printf` useful for in charts? 🟢

<details>
<summary><b>Answer</b></summary>

Building strings from parts with Go format verbs:

```yaml
image: {{ printf "%s/%s:%s" .Values.image.registry .Values.image.repository (.Values.image.tag | default .Chart.AppVersion | toString) | quote }}
url:   {{ printf "postgres://%s@%s-postgres:5432/%s" .Values.db.user .Release.Name .Values.db.name | quote }}
```

Mixing types breaks `%s` (a float renders as `%!s(float64=1.1)`), which is another reason to `toString` tags.

</details>

### Q85. How do you render one manifest per item in a list, for example a CronJob per job definition? 🟡

<details>
<summary><b>Answer</b></summary>

`range` over the list and separate documents with `---`:

```yaml
{{- range $name, $job := .Values.cronJobs }}
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{ include "shop.fullname" $ }}-{{ $name }}
  labels:
    {{- include "shop.labels" $ | nindent 4 }}
spec:
  schedule: {{ $job.schedule | quote }}
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: {{ $name }}
              image: {{ $job.image | quote }}
              args: {{- toYaml $job.args | nindent 16 }}
{{- end }}
```

Using a map keyed by name rather than a list makes overrides in env files easier, because lists are replaced wholesale.

</details>

### Q86. Why does Helm say `wrong type for value; expected string; got map[string]interface {}`? 🟡

<details>
<summary><b>Answer</b></summary>

A function that expects a string (`quote`, `trunc`, `printf "%s"`, `b64enc`) received a map, usually because a values key that used to be a scalar became a nested block, or `.` is not what you think inside `with`/`range`. Fix by pointing to the scalar field or using `toYaml` for blocks. `helm template --debug` prints the partially rendered output to locate the line.

</details>

### Q87. What is the `.Template` object good for? 🔴

<details>
<summary><b>Answer</b></summary>

`.Template.Name` is the path of the current file (`shop/templates/api-deployment.yaml`) and `.Template.BasePath` is the chart's templates directory (`shop/templates`). Uses:

```yaml
checksum/config: {{ include (print $.Template.BasePath "/api-configmap.yaml") . | sha256sum }}
{{- fail (printf "%s: image.repository is required" .Template.Name) }}
```

Inside subcharts, `BasePath` points at the subchart's templates directory.

</details>

### Q88. How do you debug a template that renders wrong output? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
helm template shop ./shop -f values-dev.yaml --debug                 # prints rendered YAML even on YAML errors
helm template shop ./shop -s templates/api-deployment.yaml           # render one file
helm template shop ./shop --set api.replicaCount=0 | yq '.spec.replicas'
helm lint ./shop -f values-prod.yaml --strict
```

Temporarily print values with `{{ toYaml .Values.api | fail }}` or `{{ printf "%#v" . }}` inside the template. For YAML errors, the `--debug` output shows the broken document, which is faster than guessing the line number.

</details>

### Q89. Can a template generate certificates, and should it? 🔴

<details>
<summary><b>Answer</b></summary>

Yes, Sprig has `genCA`, `genSignedCert`, `genSelfSignedCert` and `genPrivateKey`:

```yaml
{{- $ca := genCA "shop-webhook-ca" 365 }}
{{- $cert := genSignedCert (printf "%s.%s.svc" (include "shop.fullname" .) .Release.Namespace) nil nil 365 $ca }}
tls.crt: {{ $cert.Cert | b64enc }}
tls.key: {{ $cert.Key | b64enc }}
```

It is common for admission webhook CAs, but every render produces a new cert, so upgrades rotate them, GitOps tools show permanent drift, and certificates expire silently. In production we prefer cert-manager to issue and rotate them.

</details>

### Q90. Why are template names "global", and how does that bite you with subcharts? 🔴

<details>
<summary><b>Answer</b></summary>

All templates from the parent and every subchart are loaded into one namespace before rendering. If both the parent and a subchart define `"fullname"` or `"labels"`, one definition wins (the last loaded), and the other chart renders with the wrong helper, producing wrong names or labels with no error.

That is why every `define` is prefixed with the chart name (`shop.fullname`, `postgres.fullname`), and why library charts use a distinct prefix (`shacklib.deployment`). It is also what makes library charts possible: the parent can call a subchart's helpers directly.

</details>

---

## Values & Precedence

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q91. What is the precedence order of values in Helm? 🟢

<details>
<summary><b>Answer</b></summary>

From lowest to highest:

1. The chart's own `values.yaml`.
2. For a subchart, values set by the parent chart's `values.yaml` under the subchart's key.
3. `-f` / `--values` files, in the order given; later files win.
4. `--set`, `--set-string`, `--set-file`, `--set-json`, `--set-literal`, which override all files.

```bash
helm upgrade --install shop ./shop -f values.yaml -f values-prod.yaml --set api.image.tag=2.3.1
```

Maps are deep-merged; lists and scalars are replaced.

</details>

### Q92. How are lists merged across values files? 🟡

<details>
<summary><b>Answer</b></summary>

They are not merged. A list in a higher-precedence file replaces the whole list from lower layers. If `values.yaml` has three `tolerations` and `values-prod.yaml` sets one, prod gets exactly one.

That is why mature charts model collections as maps keyed by name (`extraEnv: {LOG_LEVEL: info}`), which deep-merge, and convert them to lists in the template with `range $k, $v`.

</details>

### Q93. What is the difference between `--set` and `--set-string`? 🟢

<details>
<summary><b>Answer</b></summary>

`--set` infers types: `true`/`false` become booleans, digits become numbers (`1.10` becomes `1.1`, `0123` may lose the zero). `--set-string` keeps the value as a string.

```bash
helm upgrade shop ./shop --set-string api.image.tag=1.10 --set-string podAnnotations.team=007
```

Use `--set-string` for image tags, versions, account IDs, ports used in annotations and anything that looks numeric but is an identifier.

</details>

### Q94. How do you set nested keys, list items and keys containing dots with `--set`? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
--set api.resources.limits.memory=512Mi                        # nested
--set ingress.hosts[0].host=shop.example.com                   # list index
--set 'ingress.hosts={a.example.com,b.example.com}'            # list of scalars
--set podAnnotations."prometheus\.io/scrape"=true              # escaped dot in key
--set-string 'podAnnotations.prometheus\.io/port=3000'
--set api.extraArgs=null                                        # delete a default
```

Commas separate multiple assignments, so a value containing a comma must be escaped (`a\,b`) or passed via a file. Beyond trivial overrides, a values file is more readable and reviewable.

</details>

### Q95. What are `--set-file`, `--set-json` and `--set-literal` for? 🟡

<details>
<summary><b>Answer</b></summary>

- `--set-file key=path`: sets the value to the file's contents (certificates, config files, scripts).
- `--set-json 'api.resources={"limits":{"memory":"512Mi"}}'`: sets structured JSON values in one flag.
- `--set-literal key=value`: takes the value verbatim with no parsing of commas, braces or escapes, handy for passwords with special characters.

```bash
helm upgrade shop ./shop --set-file tls.crt=./certs/tls.crt --set-literal db.password='p@ss,w{rd}'
```

</details>

### Q96. How do you remove a default value set in a chart's `values.yaml`? 🟡

<details>
<summary><b>Answer</b></summary>

Set it to `null` in your values file or with `--set key=null`. Helm deletes the key during coalescing, so `with`/`if` blocks treat it as unset.

```yaml
api:
  resources:
    limits:
      cpu: null   # remove the chart's CPU limit, keep memory
```

This is the only way to "unset" a map key; setting it to `{}` keeps the key with an empty map, which also works for `with` blocks but not for all templates.

</details>

### Q97. What does `--reuse-values` do and why is it dangerous? 🔴

<details>
<summary><b>Answer</b></summary>

On upgrade, `--reuse-values` takes the previous release's fully coalesced values (old chart defaults plus old overrides) as the new chart's defaults, then merges this command's `-f` and `--set` on top. The new chart's own `values.yaml` is effectively ignored.

Danger: a chart upgrade that adds a new required default key (for example `api.probes.readiness`) renders with that key missing and fails or misbehaves. In CI we never use it; we pass the full set of values files every time so the deploy is reproducible from Git. Helm 3.14+ adds `--reset-then-reuse-values`, which applies new chart defaults and then the previous overrides.

</details>

### Q98. What does `--reset-values` do? 🟡

<details>
<summary><b>Answer</b></summary>

It discards values from previous releases and uses only the chart's defaults plus what you pass in this command. Without `--reuse-values` or `--reset-values`, `helm upgrade` already behaves like reset when you pass any `-f` or `--set`. If you pass none, Helm reuses the previous values. Explicitly using `--reset-values` makes the intent obvious in scripts.

</details>

### Q99. How do you see which values a release was deployed with? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
helm get values shop -n shop                 # user-supplied overrides only
helm get values shop -n shop --all           # fully merged, including chart defaults
helm get values shop -n shop --revision 4 -o yaml > rev4-values.yaml
```

`--revision` lets you compare what changed between two deploys. These values may contain secrets, so avoid printing them in shared CI logs.

</details>

### Q100. How do you structure values files for multiple environments? 🟡

<details>
<summary><b>Answer</b></summary>

Keep chart defaults in `values.yaml` safe and environment-agnostic, then layer small override files:

```text
shop/values.yaml                  # defaults (in the chart)
deploy/values-common.yaml         # org-wide settings
deploy/values-dev.yaml            # 1 replica, debug logging, no persistence
deploy/values-prod.yaml           # 3 replicas, HPA, PDB, ingress TLS, persistence
deploy/secrets.prod.yaml          # SOPS-encrypted
```

```bash
helm upgrade --install shop oci://.../shop --version 1.4.0 \
  -f deploy/values-common.yaml -f deploy/values-prod.yaml -f secrets://deploy/secrets.prod.yaml
```

Diff the env files periodically; drift between them is where "works in staging" bugs come from.

</details>

### Q101. What is `global` in values? 🟡

<details>
<summary><b>Answer</b></summary>

`.Values.global` is the one key Helm copies into every subchart. Setting `global.imageRegistry: 123456789012.dkr.ecr.ap-south-1.amazonaws.com` in the parent makes it available as `.Values.global.imageRegistry` in the parent and in all subcharts, at every nesting depth.

```yaml
global:
  imageRegistry: registry.internal
  storageClass: gp3
```

Only charts that read `global.*` benefit; check `helm show values` of each dependency.

</details>

### Q102. How does a parent chart pass values to a subchart? 🟢

<details>
<summary><b>Answer</b></summary>

Under a top-level key matching the subchart's name (or its `alias`):

```yaml
# parent values.yaml
postgres:
  enabled: true
  auth:
    database: shop
    username: shop
  persistence:
    size: 5Gi
```

Inside the subchart those appear as `.Values.auth.database` etc. The parent can read them too (`.Values.postgres.auth.database`), which is handy for building connection strings.

</details>

### Q103. Can a subchart read the parent chart's values? 🟡

<details>
<summary><b>Answer</b></summary>

No. A subchart sees only its own values block (from its `values.yaml` merged with what the parent passes under its key) and `global`. This isolation is deliberate: a subchart must work standalone. If several subcharts need the same setting, put it under `global` or pass it explicitly to each. For the reverse direction, use `import-values` to pull subchart values into the parent.

</details>

### Q104. What is a typical "type coercion" bug with values, and how do you avoid it? 🟡

<details>
<summary><b>Answer</b></summary>

Examples:
- `tag: 1.10` in YAML becomes `1.1`.
- `--set enabled=false` is a boolean, but `--set-string enabled=false` is the string `"false"`, which is truthy in `if`.
- `port: "3000"` in values renders a quoted string into `containerPort`, causing `invalid type ... got "string", expected "integer"`.

Avoid with quoted strings for identifiers in values, `int`/`toString` conversions in templates, and `values.schema.json` declaring types so wrong types fail at install time.

</details>

### Q105. How do `-f` files with remote URLs work? 🔴

<details>
<summary><b>Answer</b></summary>

`-f` accepts local paths, `-` (stdin) and URLs (`https://...`), and any scheme registered by a getter plugin, such as helm-secrets' `secrets://`:

```bash
helm upgrade shop ./shop -f https://config.internal/shop/prod.yaml
sops -d secrets.prod.yaml | helm upgrade shop ./shop -f values-prod.yaml -f -
helm upgrade shop ./shop -f secrets://secrets.prod.yaml
```

Remote URLs make deploys depend on an external server and are not version-pinned; we prefer values files in the same Git commit as the deploy definition.

</details>

### Q106. How does Helm treat a values key set to an empty string vs not set? 🟡

<details>
<summary><b>Answer</b></summary>

Both are "empty" for `if`, `with`, `default` and `required`. `hasKey` distinguishes them: an empty string is present, an absent key is not. In JSON Schema, `minLength: 1` rejects the empty string, while `required` only checks presence. For a mandatory password, use both: schema `required` plus `minLength`, and template `required` as a safety net.

</details>

### Q107. What warning does Helm print when a values file overrides a map with a scalar? 🔴

<details>
<summary><b>Answer</b></summary>

When types clash during coalescing, Helm keeps going and prints warnings like:

```text
coalesce.go:286: warning: cannot overwrite table with non table for shop.api.resources (map[limits:map[memory:256Mi]])
```

It means a higher layer set `api.resources: ""` (or a list) where the default is a map. The override is ignored or partially applied. Treat these warnings as errors in CI: `helm template ... 2>&1 | grep -q 'warning: cannot overwrite' && exit 1`.

</details>

### Q108. How do you keep image tags out of values files when CI builds images? 🟡

<details>
<summary><b>Answer</b></summary>

Two common patterns:

1. CI passes the tag at deploy time: `--set-string api.image.tag=$GIT_SHA`. Simple, but the deployed tag is not in Git.
2. GitOps: CI commits the new tag into `values-prod.yaml` (or Argo CD Image Updater does), so Git is the source of truth and rollback is a `git revert`.

Alternatively, package the chart with `--app-version $VERSION` and let templates default the tag to `.Chart.AppVersion`, so the chart version itself pins the image.

</details>

### Q109. What are values "contracts" and why should you avoid exposing raw Kubernetes structures everywhere? 🔴

<details>
<summary><b>Answer</b></summary>

`values.yaml` is the chart's API. Passing raw blocks (`resources`, `affinity`, `securityContext`) through with `toYaml` is fine because they are well-known Kubernetes types. But if every field is a raw passthrough, the chart offers no guardrails and every consumer duplicates boilerplate.

Good charts offer opinionated, validated fields for the common path (`api.port`, `api.replicas`, `ingress.host`), raw passthrough for escape hatches (`extraEnv`, `extraVolumes`, `podSpecOverrides`), and a schema covering both.

</details>

### Q110. How do you print the computed values for a release before installing it? 🟡

<details>
<summary><b>Answer</b></summary>

`helm template` does not print values, but `--dry-run` shows them, and you can also render a debug ConfigMap. The quickest reliable ways:

```bash
helm install shop ./shop -f values-prod.yaml --dry-run=client --debug | sed -n '/^COMPUTED VALUES:/,/^HOOKS:/p'
helm show values ./shop                      # defaults only
```

`--debug` prints `USER-SUPPLIED VALUES` and `COMPUTED VALUES` sections, which is the merged result after all layers and subchart coalescing.

</details>

---

## Dependencies, Subcharts & Library Charts

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q111. How do you declare a chart dependency in Helm 3? 🟢

<details>
<summary><b>Answer</b></summary>

In the `dependencies` list of `Chart.yaml` (Helm 2 used a separate `requirements.yaml`):

```yaml
dependencies:
  - name: postgresql
    version: "16.x.x"
    repository: "oci://registry-1.docker.io/bitnamicharts"
    condition: postgresql.enabled
  - name: shop-postgres
    version: "0.1.0"
    repository: "file://../shop-postgres"
    alias: postgres
```

`repository` can be an HTTP repo URL, an `@alias` of a repo you added locally, an `oci://` reference or a `file://` path.

</details>

### Q112. What is the difference between `helm dependency update` and `helm dependency build`? 🟡

<details>
<summary><b>Answer</b></summary>

- `update`: resolves version ranges in `Chart.yaml` against the repositories, downloads the newest matching versions into `charts/`, and rewrites `Chart.lock`.
- `build`: downloads exactly the versions pinned in `Chart.lock` (falls back to update if there is no lock).

```bash
helm dependency update ./shop   # developer, intentionally bumping
helm dependency build ./shop    # CI, reproducible
helm dependency list ./shop     # shows status: ok, missing, wrong version
```

Commit `Chart.lock`, not the `.tgz` files.

</details>

### Q113. What is `Chart.lock` and why commit it? 🟢

<details>
<summary><b>Answer</b></summary>

It records the exact resolved version of each dependency plus a digest of the dependency definitions. With version ranges like `16.x.x` in `Chart.yaml`, two builds a week apart could pull different subchart versions; `helm dependency build` from the lock makes builds reproducible, the same role as `package-lock.json`. If `Chart.yaml` changes, Helm detects the digest mismatch and asks you to run `update`.

</details>

### Q114. How does `condition` work on a dependency? 🟢

<details>
<summary><b>Answer</b></summary>

`condition` is a values path (or comma-separated list of paths, first one found wins) that enables or disables the subchart:

```yaml
dependencies:
  - name: shop-postgres
    condition: postgres.enabled
```

```bash
helm upgrade shop ./shop --set postgres.enabled=false --set externalDatabase.host=shop.xxxx.rds.amazonaws.com
```

When disabled, none of the subchart's templates render. This is the standard way to use an in-cluster database in dev and RDS in prod with the same chart.

</details>

### Q115. What are `tags` on dependencies? 🟡

<details>
<summary><b>Answer</b></summary>

`tags` group dependencies so users can toggle several at once:

```yaml
dependencies:
  - name: prometheus
    tags: [monitoring]
  - name: grafana
    tags: [monitoring]
```

```bash
helm install obs ./platform --set tags.monitoring=false
```

If a dependency has both, `condition` takes precedence over `tags`. Tags are rarely needed; explicit conditions per component are clearer.

</details>

### Q116. What does `alias` do on a dependency? 🟡

<details>
<summary><b>Answer</b></summary>

It renames the subchart instance, which changes its values key and lets you include the same chart more than once:

```yaml
dependencies:
  - name: redis
    version: "20.x.x"
    repository: oci://registry-1.docker.io/bitnamicharts
    alias: cache
  - name: redis
    version: "20.x.x"
    repository: oci://registry-1.docker.io/bitnamicharts
    alias: queue
```

Configure them as `cache:` and `queue:` in values. `.Chart.Name` inside the subchart becomes the alias, so resource names differ.

</details>

### Q117. What is `import-values`? 🔴

<details>
<summary><b>Answer</b></summary>

It copies values from a subchart into the parent's values, the reverse of normal flow. Either from the subchart's `exports` block, or with explicit child/parent paths:

```yaml
dependencies:
  - name: shop-postgres
    repository: file://../shop-postgres
    import-values:
      - child: service
        parent: dbService
```

Then the parent can read `.Values.dbService.port` with the subchart's default. It is useful for discovering a subchart's port or service name without duplicating it, but it is easy to misunderstand; many teams avoid it and use helpers instead.

</details>

### Q118. How does a parent chart reference a subchart's Service name reliably? 🔴

<details>
<summary><b>Answer</b></summary>

Options, from simplest to cleanest:

1. Rely on the subchart's naming convention and replicate it: `{{ .Release.Name }}-postgres`. Breaks if the user sets `postgres.fullnameOverride`.
2. Read the override in the parent: `{{ .Values.postgres.fullnameOverride | default (printf "%s-postgres" .Release.Name) }}`.
3. Call the subchart's own helper with a constructed context (works on any Helm 3 version; recent versions also expose the subchart scope as `.Subcharts.postgres`):

```yaml
{{- $pgCtx := dict "Values" .Values.postgres "Release" .Release "Chart" (dict "Name" "postgres") }}
host: {{ include "shop-postgres.fullname" $pgCtx }}
```

Document the chosen approach in `values.yaml` so overrides stay consistent.

</details>

### Q119. What is a library chart and when would you build one? 🟡

<details>
<summary><b>Answer</b></summary>

A chart with `type: library` that only contains named templates. Platform teams build one (for example `shacklib`) when many microservice charts need the same Deployment, Service, HPA and PDB shapes, labels and security defaults. Each service chart depends on it and its templates become a few lines:

```yaml
# templates/deployment.yaml in a service chart
{{ include "shacklib.deployment" . }}
```

Fixing a security default in the library and bumping the dependency propagates it to every service.

</details>

### Q120. How do you consume a library chart? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
# Chart.yaml of the service chart
dependencies:
  - name: shacklib
    version: "1.2.0"
    repository: "oci://ghcr.io/devopsshack/charts"
```

```bash
helm dependency build ./orders
```

Then call its templates. A common pattern is a "merge" helper: the library defines a base Deployment, the service passes an overlay template name, and the library merges both with `mergeOverwrite` before `toYaml`, which lets services customize without copying the whole manifest.

</details>

### Q121. How do library charts differ from just copying `_helpers.tpl` into every chart? 🟡

<details>
<summary><b>Answer</b></summary>

Copying works until you have 40 charts with 40 slightly different helpers. A library chart is versioned and distributed like any chart: fixes ship by bumping one dependency version, you can test the library once, and `Chart.lock` shows exactly which library version each service uses. The cost is an extra layer of indirection that makes rendered output harder to trace, so keep the library small and well documented.

</details>

### Q122. What is an umbrella chart? 🟡

<details>
<summary><b>Answer</b></summary>

A parent chart with few or no templates of its own whose purpose is to install several subcharts together as one release (for example `platform` with ingress-nginx, cert-manager, external-dns). Pros: one install command, one version for the bundle. Cons: one giant release (slow upgrades, all-or-nothing rollback, huge release Secret, everything in one namespace). In 2025–2026 most teams prefer Helmfile or Argo CD ApplicationSets to compose separate releases instead.

</details>

### Q123. How do you override values of a subchart's subchart? 🟡

<details>
<summary><b>Answer</b></summary>

Nest keys by chart name (or alias) at each level:

```yaml
platform:          # subchart of the umbrella
  monitoring:      # subchart of platform
    grafana:       # subchart of monitoring
      adminPassword: ...
```

`global` reaches all levels in one place. Deep nesting is a smell; if you need it often, flatten the structure.

</details>

### Q124. How do dependency version constraints work? 🟢

<details>
<summary><b>Answer</b></summary>

They follow SemVer ranges: `"1.2.3"` exact, `"~1.2.0"` (>=1.2.0 <1.3.0), `"^1.2.0"` (>=1.2.0 <2.0.0), `"1.x.x"`, `">=1.2.0 <1.5.0"`. Pre-releases are excluded unless the constraint includes one. We use caret or tilde ranges in `Chart.yaml` and rely on `Chart.lock` for the exact version, so upgrades are explicit `helm dependency update` commits reviewed in PRs.

</details>

### Q125. Can you use a dependency from a repository you have not added with `helm repo add`? 🟡

<details>
<summary><b>Answer</b></summary>

For OCI and `file://` dependencies, yes. For HTTP repositories, Helm requires the repo to be configured locally:

```text
Error: no repository definition for https://charts.example.com. Please add the missing repos via 'helm repo add'
```

In CI, add every HTTP repo before `helm dependency build`, or switch dependencies to OCI references, which need only `helm registry login` for private registries.

</details>

### Q126. How do conditions and globals interact when a subchart is disabled? 🔴

<details>
<summary><b>Answer</b></summary>

When the condition is false, the subchart's templates are skipped, but its values still exist in the parent's `.Values.<name>` (defaults are coalesced). So parent templates that read `.Values.postgres.auth.database` still get a value. That is useful (you can keep defaults) and dangerous (a parent template may build a connection string to a database that is not deployed). Gate parent logic on the same flag:

```yaml
{{- if .Values.postgres.enabled }}
host: {{ .Release.Name }}-postgres
{{- else }}
host: {{ required "externalDatabase.host is required when postgres.enabled=false" .Values.externalDatabase.host }}
{{- end }}
```

</details>

### Q127. Do subchart hooks and tests run as part of the parent release? 🟡

<details>
<summary><b>Answer</b></summary>

Yes. Subcharts are rendered into the same release, so their hooks run alongside the parent's, ordered together by weight and kind. `helm test` also runs subchart test Pods unless you filter with `--filter name=...`. This surprises people when a third-party subchart's test fails the parent's CI. Disable it via its values if possible, or filter it out.

</details>

### Q128. How do you vendor and patch a third-party subchart safely? 🔴

<details>
<summary><b>Answer</b></summary>

Order of preference:

1. Use its values (most needs are covered).
2. Use a post-renderer (Kustomize patch) without forking.
3. Pull and vendor it (`helm pull repo/chart --version X --untar -d charts/`), commit it, and apply a small, documented patch, tracking upstream with a script.

Forking means you own upgrades and CVE fixes forever, so we only do it when upstream is unmaintained.

</details>

### Q129. What is the effect of a subchart's `values.schema.json`? 🔴

<details>
<summary><b>Answer</b></summary>

Helm validates each subchart's values against its own schema, using the values the parent passed down. A parent setting `postgres.persistence.size: 5` (number instead of string) can fail the whole install with an error naming the subchart: `values don't meet the specifications of the schema(s) in the following chart(s): shop-postgres: ...`. Read the subchart's schema before overriding, and run `helm lint --with-subcharts`.

</details>

### Q130. How do you check what a dependency will render before trusting it? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
helm pull oci://registry-1.docker.io/bitnamicharts/postgresql --version 16.4.1 --untar -d /tmp/pg
helm template pg /tmp/pg/postgresql -f my-pg-values.yaml | kubeconform -strict -summary
helm template pg /tmp/pg/postgresql | grep -E 'image:|runAsUser|hostPath|privileged'
```

Check images and registries (some vendors changed image distribution in 2025, which broke default image references for many users), security contexts, RBAC it creates, and whether it uses hooks or `lookup`.

</details>

---

## Hooks, Tests & Validation

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q131. What are Helm hooks? 🟢

<details>
<summary><b>Answer</b></summary>

Hooks are normal manifests (usually Jobs) annotated with `helm.sh/hook` that Helm runs at specific points in the release lifecycle instead of applying them with the rest of the chart. Typical uses: database migrations before an upgrade, backups before uninstall, cache warmup after install, smoke tests.

```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

</details>

### Q132. List all hook types and when they run. 🟢

<details>
<summary><b>Answer</b></summary>

| Hook | Runs |
|------|------|
| `pre-install` | after rendering, before any release resource is created |
| `post-install` | after all resources are created (and ready, with `--wait`) |
| `pre-upgrade` | after rendering, before any resource is updated |
| `post-upgrade` | after resources are updated |
| `pre-rollback` / `post-rollback` | around a rollback |
| `pre-delete` / `post-delete` | around `helm uninstall` |
| `test` | only when you run `helm test` |

One resource can carry several: `"helm.sh/hook": post-install,pre-upgrade`.

</details>

### Q133. How do hook weights work? 🟡

<details>
<summary><b>Answer</b></summary>

`helm.sh/hook-weight` is a string containing an integer (negative allowed). For each hook event, Helm sorts hooks by weight ascending, then by kind, then by name, and runs them in that order, waiting for each to be ready (Jobs: complete) before the next.

```yaml
"helm.sh/hook-weight": "-5"   # ConfigMap with migration SQL
"helm.sh/hook-weight": "0"    # migration Job that mounts it
```

It must be quoted: an unquoted number makes the annotation value an integer and the manifest fails validation (annotations must be strings).

</details>

### Q134. What are the hook delete policies? 🟡

<details>
<summary><b>Answer</b></summary>

- `before-hook-creation`: delete the previous hook resource before creating the new one (the default when no policy is set).
- `hook-succeeded`: delete after the hook succeeds.
- `hook-failed`: delete if it fails.

```yaml
"helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

We always include `before-hook-creation` so the next run does not fail with `jobs.batch "shop-migrate" already exists`, and we usually omit `hook-failed` so failed Job pods stay around for `kubectl logs`.

</details>

### Q135. Are hook resources part of the release? What happens on uninstall? 🟡

<details>
<summary><b>Answer</b></summary>

No. Hook resources are tracked separately (`helm get hooks shop`) and are not in `helm get manifest`. Helm does not manage their lifecycle beyond the delete policy, so `helm uninstall` leaves behind any hook resource that was not deleted by policy, for example a ConfigMap hook or a Job without `hook-succeeded`. Do not put long-lived resources (Secrets the app needs, PVCs) in hooks.

</details>

### Q136. How does Helm decide a hook is "done"? 🔴

<details>
<summary><b>Answer</b></summary>

For Jobs, Helm waits until the Job completes successfully; a Job that reaches `backoffLimit` fails the hook and the release operation. For Pods, it waits for `Succeeded`. For all other kinds (ConfigMap, Secret, ServiceAccount), the hook is considered ready as soon as the API server accepts the object. The wait is bounded by `--timeout` (default 5m), which covers hooks too, so a slow migration needs a larger `--timeout`.

</details>

### Q137. Why is running a migration as a `pre-install` hook often a mistake? 🔴

<details>
<summary><b>Answer</b></summary>

`pre-install` runs before any release resource exists. If the database is a subchart in the same release, its StatefulSet, Service and Secret do not exist yet, so the migration Job cannot connect and fails, failing the install. Common fixes:

- Use `post-install,pre-upgrade`: on first install the DB comes up first; on upgrades the DB already exists and migrations run before the new app version rolls out.
- Or run migrations from an init container or at app startup with a migration lock.
- Make the Job wait for the DB with an init container running `pg_isready`.

Also put any ConfigMap or Secret the hook needs into the hook itself (with a lower weight), because on `pre-upgrade` the new non-hook ConfigMap has not been applied yet.

</details>

### Q138. What happens to your app during a `pre-upgrade` migration if the migration is not backward compatible? 🔴

<details>
<summary><b>Answer</b></summary>

During a `pre-upgrade` hook the **old** application pods are still serving traffic against the newly migrated schema. If the migration drops or renames a column the old code uses, you get errors until the rollout finishes. And if the upgrade then fails with `--atomic`, Helm rolls back the pods but not the schema.

The fix is the expand/contract pattern: release N adds the new column (both versions work), release N+1 switches the code, release N+2 drops the old column.

</details>

### Q139. How do you write a `helm test`? 🟢

<details>
<summary><b>Answer</b></summary>

A Pod (or Job) in `templates/tests/` annotated with `helm.sh/hook: test` whose container exits 0 on success:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "shop.fullname" . }}-test-api
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  restartPolicy: Never
  containers:
    - name: curl
      image: curlimages/curl:8.10.1
      args: ["-fsS", "http://{{ include "shop.fullname" . }}-api:3000/readyz"]
```

```bash
helm test shop -n shop --logs --timeout 2m
```

</details>

### Q140. What should a Helm test actually check? 🟡

<details>
<summary><b>Answer</b></summary>

Things that prove the release works as deployed, not unit tests: the Service resolves and responds, the API can reach the database (`/readyz` that runs `SELECT 1`), the frontend proxies to the API, a migration created the expected table. Keep tests fast (seconds), idempotent, and free of side effects on production data. Run them in the pipeline right after `helm upgrade` and treat failure as a rollback trigger.

</details>

### Q141. What does `helm lint` check? 🟢

<details>
<summary><b>Answer</b></summary>

Chart structure and best practices: `Chart.yaml` required fields and SemVer, presence of `values.yaml`, templates that fail to render or produce invalid YAML, missing `metadata.name`, deprecated APIs for the target version, icon and other recommendations (as `[INFO]`/`[WARNING]`), and schema validation of values.

```bash
helm lint ./shop -f values-prod.yaml --strict      # warnings become failures
helm lint ./shop --with-subcharts
```

It does not validate fields against Kubernetes schemas; pair it with kubeconform.

</details>

### Q142. How do you validate rendered manifests against Kubernetes schemas offline? 🟡

<details>
<summary><b>Answer</b></summary>

Render with `helm template` and pipe into a schema validator such as kubeconform, with CRD schemas for custom kinds:

```bash
helm template shop ./shop -f values-prod.yaml --kube-version 1.31 \
  | kubeconform -strict -summary -kubernetes-version 1.31.0 \
      -schema-location default \
      -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json'
```

`-strict` rejects unknown fields, catching typos like `resource:` instead of `resources:` that Kubernetes would otherwise prune silently (or reject, with server-side field validation).

</details>

### Q143. What is the helm-diff plugin and why do teams depend on it? 🟢

<details>
<summary><b>Answer</b></summary>

A plugin that shows a colorized diff between the deployed release and what an upgrade would apply:

```bash
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade shop ./shop -n shop -f values-prod.yaml --context 3
```

It turns "I hope this upgrade is safe" into a reviewable change, like `terraform plan`. We post its output to the PR and require approval for production. Helmfile uses it for `helmfile diff` and `apply`.

</details>

### Q144. What useful flags does `helm diff` have? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
# --context: lines of context, --detailed-exitcode: exit 2 when there are changes (CI gate)
# --suppress-secrets: hide Secret data, --normalize-manifests: ignore formatting-only changes
# --three-way-merge: include manual drift in the live objects
helm diff upgrade shop ./shop -f values-prod.yaml \
  --context 5 --detailed-exitcode --suppress-secrets --normalize-manifests --three-way-merge

helm diff revision shop 4 5     # compare two revisions
helm diff rollback shop 3       # preview a rollback
```

With `--three-way-merge`, changes someone made with `kubectl edit` show up in the diff instead of being silently overwritten.

</details>

### Q145. What validation does `values.schema.json` provide that `required` does not? 🟡

<details>
<summary><b>Answer</b></summary>

`required` checks a single value for emptiness while rendering. The schema validates the entire merged values object **before** rendering: types, enums, patterns, min/max, required properties, `additionalProperties: false` to catch typos, and conditional rules. It runs on `install`, `upgrade`, `template` and `lint`, produces one clear error listing all violations, and documents the API for IDEs. Use both: schema for structure, `required`/`fail` for cross-field logic.

</details>

### Q146. Show a minimal `values.schema.json`. 🟡

<details>
<summary><b>Answer</b></summary>

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["api", "postgres"],
  "properties": {
    "api": {
      "type": "object",
      "required": ["image", "replicaCount"],
      "properties": {
        "replicaCount": { "type": "integer", "minimum": 1 },
        "image": {
          "type": "object",
          "required": ["repository"],
          "properties": {
            "repository": { "type": "string", "minLength": 1 },
            "tag": { "type": "string" },
            "pullPolicy": { "enum": ["Always", "IfNotPresent", "Never"] }
          }
        }
      }
    }
  }
}
```

A bad value fails with `values don't meet the specifications of the schema(s) in the following chart(s): shop: - api.replicaCount: Invalid type. Expected: integer, given: string`.

</details>

### Q147. Can you use `additionalProperties: false` in a values schema? What is the catch? 🔴

<details>
<summary><b>Answer</b></summary>

Yes, and it catches misspelled keys (`replicacount`) that otherwise silently do nothing. The catches:

- The top-level schema must allow `global` and each subchart's key, because Helm validates the parent schema against values that include them.
- Users can no longer add arbitrary keys for `tpl` use or vendor extensions.
- Escape-hatch maps (`podAnnotations`, `extraEnv`) need `additionalProperties: {"type": "string"}` or no restriction.

We apply it to well-defined objects (`image`, `probes`, `ingress`) rather than the root.

</details>

### Q148. What is the difference between `helm test` and a `post-install` hook Job? 🟡

<details>
<summary><b>Answer</b></summary>

A `post-install` hook runs automatically during install and its failure fails (or with `--atomic`, rolls back) the release. A `test` hook runs only when someone executes `helm test`, after the release is deployed, and its result does not change the release status. Use hooks for actions the release needs (migrations, seeding) and tests for verification you trigger from the pipeline.

</details>

### Q149. How do you dry-run an upgrade against the live cluster, including admission webhooks? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
helm upgrade --install shop ./shop -n shop -f values-prod.yaml --dry-run=server
```

The API server receives every object with `dryRun=All`, so OPA Gatekeeper or Kyverno policies, the OpenAPI schema and immutable-field checks all run, and nothing is persisted. Hooks are not executed. Combined with `helm diff`, this catches most failures before a real deploy.

</details>

### Q150. How do you run only some tests or keep test pods for debugging? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
helm test shop -n shop --filter name=shop-test-api --logs
helm test shop -n shop --filter '!name=postgres-test'    # exclude by name
```

Without a `hook-delete-policy` that deletes on success, test Pods remain after the run so you can `kubectl logs` and `kubectl describe` them. Use `before-hook-creation` so the next `helm test` replaces them instead of failing with "already exists".

</details>

---

## Release Lifecycle, Repositories & OCI

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. Why is `helm upgrade --install` the standard CI command? 🟢

<details>
<summary><b>Answer</b></summary>

It is idempotent: it installs the release if it does not exist and upgrades it if it does, so the same pipeline step works for the first deploy and every later one.

```bash
helm upgrade --install shop ./shop -n shop --create-namespace \
  -f values-prod.yaml --atomic --wait --timeout 10m --history-max 15
```

Separate `install` and `upgrade` steps need branching logic and fail when someone deleted or pre-created the release.

</details>

### Q152. What does `--wait` actually wait for? 🟡

<details>
<summary><b>Answer</b></summary>

Until all release resources are ready, or `--timeout` (default `5m0s`) expires:

- Deployments: desired replicas updated and available (respecting `maxUnavailable`).
- StatefulSets and DaemonSets: rollout complete.
- Pods: Ready.
- PVCs: Bound. Services of type LoadBalancer: have an external IP.
- Jobs: only with `--wait-for-jobs`.

Without `--wait`, Helm reports success as soon as the API server accepts the objects, even if the new pods crash-loop.

</details>

### Q153. What does `--atomic` do? 🟢

<details>
<summary><b>Answer</b></summary>

It makes the operation all-or-nothing. On a failed install, Helm uninstalls the release; on a failed upgrade, it rolls back to the last successful revision. It implies `--wait`. The error then reads:

```text
Error: UPGRADE FAILED: release shop failed, and has been rolled back due to atomic being set: context deadline exceeded
```

In Helm 4 the flag is called `--rollback-on-failure`.

</details>

### Q154. What is `--cleanup-on-fail`? 🔴

<details>
<summary><b>Answer</b></summary>

On a failed upgrade, it deletes resources that were newly created by that upgrade (for example a new ConfigMap or CronJob added in this chart version). Without it, those orphans stay in the cluster and the next upgrade may fail with "resource already exists" for them. `--atomic` handles rollback of existing objects; combining both gives the cleanest failure behavior.

</details>

### Q155. How does `helm rollback` work? 🟢

<details>
<summary><b>Answer</b></summary>

It takes the manifest and values stored in a previous revision and applies them as a **new** revision. History is never rewritten.

```bash
helm history shop -n shop
# REVISION  STATUS      CHART       APP VERSION  DESCRIPTION
# 4         superseded  shop-1.3.0  2.2.0        Upgrade complete
# 5         deployed    shop-1.4.0  2.3.0        Upgrade complete
helm rollback shop 4 -n shop --wait --timeout 5m
# revision 6 created, content of revision 4
```

`helm rollback shop` with no revision goes to the previous one.

</details>

### Q156. What does a Helm rollback NOT undo? 🔴

<details>
<summary><b>Answer</b></summary>

- Database schema changes and data written by migrations or the app.
- Effects of hooks (a backup was taken, a message was published).
- Resources Helm does not manage: CRDs in `crds/`, hook resources, objects with `resource-policy: keep`, things created by operators.
- External state: DNS records, cloud resources created by controllers, cache contents.
- PVCs resized larger (volumes cannot shrink).

That is why we design upgrades to be backward compatible and treat rollback as a fast path for code issues, not a time machine.

</details>

### Q157. Explain release statuses you might see in `helm list -a` or `helm history`. 🟡

<details>
<summary><b>Answer</b></summary>

- `deployed`: current active revision.
- `superseded`: an older revision replaced by a later one.
- `failed`: the operation for that revision failed.
- `pending-install`, `pending-upgrade`, `pending-rollback`: operation in progress, or interrupted.
- `uninstalling`, `uninstalled`: in progress, or done with `--keep-history`.
- `unknown`: rarely, corrupted state.

```bash
helm list -A --all            # every status, all namespaces
helm list -n shop --pending   # only stuck ones
helm list -n shop --failed
```

</details>

### Q158. What does `helm uninstall --keep-history` do? 🟡

<details>
<summary><b>Answer</b></summary>

It deletes the release's resources but keeps the revision Secrets marked `uninstalled`. You can still read `helm history`, and you can `helm rollback` to bring the release back. It is useful when decommissioning something you might need to restore, and for audit. A plain `helm install` with the same name then fails with `cannot re-use a name that is still in use`; restore with `helm rollback shop <revision>`, or start over with `helm install --replace`.

</details>

### Q159. What `helm get` subcommands exist? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
helm get manifest shop -n shop          # rendered resources (no hooks, no crds/)
helm get values   shop -n shop --all    # merged values
helm get hooks    shop -n shop          # rendered hooks
helm get notes    shop -n shop          # NOTES.txt output
helm get metadata shop -n shop          # chart, version, appVersion, status, revision
helm get all      shop -n shop --revision 3
```

`helm get manifest shop | kubectl diff -f -` is a quick way to detect drift between the last release and live state.

</details>

### Q160. How do classic HTTP chart repositories work? 🟢

<details>
<summary><b>Answer</b></summary>

A repository is any HTTP server hosting packaged charts plus an `index.yaml` that lists every chart version with its URL and digest. Clients download the index on `helm repo add`/`update` and search it locally.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo prometheus-community/kube-prometheus-stack --versions | head
helm repo index ./packages --url https://charts.example.com     # generate index.yaml
```

GitHub Pages, S3 and ChartMuseum are common hosts. The index grows with every version, which is one reason teams moved to OCI.

</details>

### Q161. What is an OCI-based Helm registry? 🟢

<details>
<summary><b>Answer</b></summary>

Since Helm 3.8, charts can be stored as OCI artifacts in any OCI-compliant container registry (ECR, GHCR, ACR, Artifact Registry, Harbor, Docker Hub). A chart version is a tag; the chart content is a layer with media type `application/vnd.cncf.helm.chart.content.v1.tar+gzip`.

Advantages: one registry and one auth system for images and charts, immutable digests, registry-native access control, replication and signing (cosign), and no `index.yaml` to maintain. There is no `helm repo add` and no `helm search` for OCI; you reference the chart directly.

</details>

### Q162. How do you push a chart to Amazon ECR? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
export AWS_REGION=ap-south-1 ACCOUNT=123456789012
REG=$ACCOUNT.dkr.ecr.$AWS_REGION.amazonaws.com
aws ecr create-repository --repository-name charts/shop --region $AWS_REGION
aws ecr get-login-password --region $AWS_REGION | helm registry login --username AWS --password-stdin $REG
helm package ./shop --version 1.4.0 --app-version 2.3.0
helm push shop-1.4.0.tgz oci://$REG/charts
# Pushed: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/charts/shop:1.4.0
helm install shop oci://$REG/charts/shop --version 1.4.0 -n shop --create-namespace
```

The ECR repository name is the push path plus the chart name (`charts/shop`) and must exist first. Enable tag immutability on it.

</details>

### Q163. How do you push a chart to GitHub Container Registry? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
echo "$GITHUB_TOKEN" | helm registry login ghcr.io -u "$GITHUB_ACTOR" --password-stdin
helm package ./shop
helm push shop-1.4.0.tgz oci://ghcr.io/devopsshack/charts
helm pull oci://ghcr.io/devopsshack/charts/shop --version 1.4.0
```

In GitHub Actions the workflow needs `permissions: packages: write`. The owner part must be lowercase. New packages are private by default; make them public or grant repository access in package settings.

</details>

### Q164. How do you inspect or install a chart directly from OCI? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
helm show values oci://ghcr.io/devopsshack/charts/shop --version 1.4.0
helm template shop oci://ghcr.io/devopsshack/charts/shop --version 1.4.0 -f values-prod.yaml
helm upgrade --install shop oci://ghcr.io/devopsshack/charts/shop --version 1.4.0 -n shop
helm install shop oci://ghcr.io/devopsshack/charts/shop@sha256:3f1c... -n shop   # pin by digest
```

Always pass `--version` (or a digest). Without it Helm resolves the latest semver tag, which makes deploys non-reproducible.

</details>

### Q165. How do chart signing and provenance work? 🔴

<details>
<summary><b>Answer</b></summary>

Classic Helm provenance uses PGP: `helm package --sign --key 'DevOps Shack' --keyring ~/.gnupg/secring.gpg` creates `shop-1.4.0.tgz.prov` with the chart digest and a signature; `helm verify shop-1.4.0.tgz` or `helm install --verify` checks it.

For OCI charts, most teams in 2025–2026 sign with Sigstore cosign, exactly like images:

```bash
cosign sign --yes ghcr.io/devopsshack/charts/shop@sha256:3f1c...
cosign verify ghcr.io/devopsshack/charts/shop:1.4.0 --certificate-identity-regexp 'https://github.com/devopsshack/.*' --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

Admission policies (Kyverno) or Argo CD then verify before deploying.

</details>

### Q166. How do you mirror third-party charts into a private registry? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
helm pull ingress-nginx/ingress-nginx --version 4.11.3
helm push ingress-nginx-4.11.3.tgz oci://$REG/mirror
```

Then reference `oci://$REG/mirror/ingress-nginx` in Helmfile or Argo CD. Mirroring protects you from upstream outages, deleted versions and rate limits, and lets you scan charts and pin images. Remember to mirror the container images the chart uses too, and override their registry in values.

</details>

### Q167. How do you search for public charts? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
helm search hub postgresql             # Artifact Hub
helm search repo bitnami/postgresql --versions
helm search repo shop --devel          # include pre-releases
```

`helm search hub` queries Artifact Hub; `search repo` queries indexes of repos you added. Neither searches OCI registries. Before adopting a chart, check maintainer activity, verified publisher status, security report and how it handles images.

</details>

### Q168. How do you pin and constrain chart versions when installing? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
helm upgrade --install pg bitnami/postgresql --version 16.4.1        # exact
helm upgrade --install pg bitnami/postgresql --version '~16.4.0'     # patch updates only
helm upgrade --install shop oci://$REG/charts/shop --version 1.4.0
```

In production we use exact versions; ranges belong only in `Chart.yaml` dependencies resolved by `Chart.lock`.

</details>

### Q169. What does `helm package` do and which flags matter? 🟢

<details>
<summary><b>Answer</b></summary>

It validates and archives a chart directory into `<name>-<version>.tgz`, applying `.helmignore`:

```bash
helm package ./shop --version 1.4.0 --app-version 2.3.0 --dependency-update -d dist/
```

`--version` and `--app-version` override `Chart.yaml` without editing it, which is how CI injects the build version. `--dependency-update` refreshes `charts/` first. The archive is what you push to a registry.

</details>

### Q170. How does `--force` behave on upgrade, and why is it risky? 🔴

<details>
<summary><b>Answer</b></summary>

In Helm 3, `--force` replaces resources (delete and recreate via a PUT replace) instead of patching. It can get past some immutable-field errors, but it causes downtime for replaced objects and fails on fields the API server defaults, such as a Service's `clusterIP`:

```text
Error: UPGRADE FAILED: failed to replace object: Service "shop-api" is invalid: spec.clusterIP: Invalid value: "": field is immutable
```

We avoid it. For immutable selector changes we orphan-delete the specific Deployment and upgrade normally. Helm 4 renames it `--force-replace`.

</details>

### Q171. What is the `--description` flag and why use it? 🟡

<details>
<summary><b>Answer</b></summary>

It sets the DESCRIPTION column of the revision in `helm history`:

```bash
helm upgrade --install shop ./shop -f values-prod.yaml \
  --description "deploy $GIT_SHA by $GITHUB_ACTOR (PR #482)"
```

During an incident, a history that says which commit and pipeline produced each revision is far more useful than ten lines of "Upgrade complete".

</details>

### Q172. Can two people upgrade the same release at once? 🔴

<details>
<summary><b>Answer</b></summary>

Helm guards against it with the `pending-*` status: the second operation sees an in-progress revision and fails with `another operation (install/upgrade/rollback) is in progress`. It is not a strict distributed lock, so pipelines should also serialize deploys per environment (GitHub Actions `concurrency:` groups, GitLab `resource_group`). If a process is killed, the pending status remains and must be cleared manually.

</details>

---

## Schema, Helmfile, Argo CD & Secrets

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q173. When is `values.schema.json` validated, and can it be skipped? 🟡

<details>
<summary><b>Answer</b></summary>

On `helm install`, `upgrade`, `template` and `lint`, against the fully merged values, for the parent and each subchart. Helm uses JSON Schema (draft 7 was the long-standing supported draft; newer Helm 3 releases also accept later drafts). Recent versions add `--skip-schema-validation` for emergencies, for example when a subchart's schema is buggy. Using it routinely defeats the purpose; fix the values or pin a fixed subchart version instead.

</details>

### Q174. How do you express "password is required only when the internal database is enabled" in a schema? 🔴

<details>
<summary><b>Answer</b></summary>

With `if`/`then` (JSON Schema draft 7):

```json
"postgres": {
  "type": "object",
  "properties": {
    "enabled": { "type": "boolean" },
    "auth": { "type": "object", "properties": { "password": { "type": "string" } } }
  },
  "if": { "properties": { "enabled": { "const": true } } },
  "then": {
    "required": ["auth"],
    "properties": { "auth": { "required": ["password"], "properties": { "password": { "minLength": 12 } } } }
  }
}
```

Keep schemas readable; deeply nested conditionals are better replaced with a `fail` in a template helper with a clear message.

</details>

### Q175. What is Helmfile? 🟢

<details>
<summary><b>Answer</b></summary>

A declarative tool that describes many Helm releases, their chart versions, values and environments in one `helmfile.yaml`, then reconciles the cluster with `helmfile apply`. It is "docker compose for Helm releases": repositories, environments, release dependencies (`needs`), templated values, and diffs before applying via helm-diff. Helmfile 1.x (released 2025) is the current major line.

</details>

### Q176. Show a realistic `helmfile.yaml`. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
repositories:
  - name: ingress-nginx
    url: https://kubernetes.github.io/ingress-nginx
  - name: shack
    url: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/charts
    oci: true

environments:
  dev:
    values: [env/dev.yaml]
  prod:
    values: [env/prod.yaml]
---
releases:
  - name: ingress-nginx
    namespace: ingress-nginx
    chart: ingress-nginx/ingress-nginx
    version: 4.11.3
  - name: shop
    namespace: shop
    chart: shack/shop
    version: 1.4.0
    needs: [ingress-nginx/ingress-nginx]
    values:
      - values/shop-common.yaml
      - values/shop-{{ .Environment.Name }}.yaml
    secrets:
      - secrets/shop-{{ .Environment.Name }}.yaml
```

```bash
helmfile -e prod diff
helmfile -e prod apply
```

In Helmfile 1.x, files that use Go templating should carry the `.gotmpl` extension (`helmfile.yaml.gotmpl`).

</details>

### Q177. What is the difference between `helmfile sync` and `helmfile apply`? 🟡

<details>
<summary><b>Answer</b></summary>

- `sync`: runs `helm upgrade --install` for every release, whether or not anything changed.
- `apply`: runs `helm diff` first and upgrades only releases with changes, then deletes releases marked `installed: false`. Exit status can signal changes (`--detailed-exitcode`).

`apply` is what CI should use: quieter history, faster, and the diff is printed for audit. Other useful commands: `helmfile template`, `helmfile lint`, `helmfile destroy`, `helmfile -l name=shop apply` (label selector).

</details>

### Q178. How does Argo CD deploy a Helm chart? 🔴

<details>
<summary><b>Answer</b></summary>

Argo CD's repo server renders the chart with the equivalent of `helm template`, passing the release name, namespace and values, and then applies the manifests itself (kubectl-style apply or server-side apply) and tracks health and drift. Helm is used purely as a templating engine:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shop-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/charts
    chart: shop
    targetRevision: 1.4.0
    helm:
      releaseName: shop
      valuesObject:
        api:
          replicaCount: 3
  destination:
    server: https://kubernetes.default.svc
    namespace: shop
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

For OCI, `repoURL` has no `oci://` prefix and the registry must be registered in Argo CD as a Helm repo with OCI enabled.

</details>

### Q179. What are the consequences of Argo CD using `helm template` instead of `helm install`? 🔴

<details>
<summary><b>Answer</b></summary>

- No release Secrets: `helm list`, `helm history`, `helm rollback` and `helm test` do not apply. Rollback is Argo CD history or a Git revert.
- `lookup` returns empty, so "keep existing password" logic regenerates values every render.
- Random functions (`randAlphaNum`, `genCA`) and `.Release.Revision` cause permanent OutOfSync.
- `.Release.IsInstall` is always true.
- `.Capabilities` comes from Argo CD's view of the cluster (configurable with `kubeVersion`/`apiVersions` in the source).
- Hooks are translated to Argo CD sync hooks rather than executed by Helm.

Charts meant for GitOps should be deterministic.

</details>

### Q180. How are Helm hooks mapped in Argo CD? 🔴

<details>
<summary><b>Answer</b></summary>

Argo CD maps Helm hook annotations to its own phases:

| Helm | Argo CD |
|------|---------|
| `pre-install`, `pre-upgrade` | `PreSync` |
| `post-install`, `post-upgrade` | `PostSync` |
| `pre-delete` | `PreDelete` (recent Argo CD versions) |
| `post-delete` | `PostDelete` |
| `test` | not run |
| `pre-rollback`, `post-rollback` | ignored |

`hook-weight` maps to sync wave ordering within the phase and `hook-delete-policy` maps to Argo CD delete policies. Note that `pre-install` and `pre-upgrade` both become `PreSync`, so a hook meant only for install runs on every sync. If a chart defines Argo CD's own `argocd.argoproj.io/hook` annotation, that takes precedence.

</details>

### Q181. How do you pass values to a Helm chart from a separate Git repo in Argo CD? 🟡

<details>
<summary><b>Answer</b></summary>

Use multiple sources with a `ref`, so the chart comes from a registry and values from your config repo:

```yaml
spec:
  sources:
    - repoURL: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/charts
      chart: shop
      targetRevision: 1.4.0
      helm:
        releaseName: shop
        valueFiles:
          - $values/envs/prod/shop-values.yaml
    - repoURL: https://github.com/devopsshack/platform-config.git
      targetRevision: main
      ref: values
```

This keeps chart releases and environment config versioned independently, and an ApplicationSet can generate one Application per environment.

</details>

### Q182. Why should you not store plaintext secrets in values files? 🟢

<details>
<summary><b>Answer</b></summary>

Values files live in Git, so anyone with repo access (and every clone, fork and CI log) sees them forever. Helm also stores merged values in the release Secret, and `helm get values` prints them. Options: encrypt secrets in Git with SOPS and decrypt at deploy time (helm-secrets), keep them in a secret manager and sync with External Secrets Operator, or inject via `--set-string` from CI secrets (still lands in the release record).

</details>

### Q183. How do helm-secrets and SOPS work together? 🟡

<details>
<summary><b>Answer</b></summary>

SOPS encrypts the values of a YAML file (keys stay readable) using age, PGP, AWS KMS, GCP KMS or Azure Key Vault. helm-secrets is a Helm plugin that decrypts SOPS files on the fly during `install`/`upgrade`/`template` and removes the plaintext afterwards.

```yaml
# .sops.yaml
creation_rules:
  - path_regex: secrets/.*prod.*\.yaml$
    kms: arn:aws:kms:ap-south-1:123456789012:key/1111-2222
  - path_regex: secrets/.*\.yaml$
    age: age1qy8...devkey
```

```bash
helm plugin install https://github.com/jkroepke/helm-secrets
sops --encrypt --in-place secrets/shop-prod.yaml
helm upgrade --install shop ./shop -f values-prod.yaml -f secrets://secrets/shop-prod.yaml
```

</details>

### Q184. What are the helm-secrets commands you use day to day? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
helm secrets encrypt -i secrets/shop-dev.yaml       # encrypt in place
helm secrets edit secrets/shop-dev.yaml             # decrypt to $EDITOR, re-encrypt on save
helm secrets decrypt secrets/shop-dev.yaml          # print plaintext (careful)
helm secrets upgrade --install shop ./shop -f secrets/shop-dev.yaml   # wrapper mode
helm upgrade --install shop ./shop -f secrets://secrets/shop-dev.yaml # getter mode (preferred)
```

The getter mode (`secrets://`) works with plain `helm` and with Helmfile and Argo CD integrations. CI needs the decryption identity: `SOPS_AGE_KEY_FILE` or an IAM role with `kms:Decrypt`.

</details>

### Q185. When would you choose External Secrets Operator over helm-secrets? 🟡

<details>
<summary><b>Answer</b></summary>

When the source of truth is a secret manager (AWS Secrets Manager, Vault, GCP Secret Manager) rather than Git, when secrets rotate independently of deploys, and when using Argo CD (which needs extra plugins for SOPS). The chart then renders an `ExternalSecret` that references a key name, never the value:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "shop.fullname" . }}-db
spec:
  refreshInterval: 1h
  secretStoreRef: { kind: ClusterSecretStore, name: aws-sm }
  target: { name: {{ include "shop.fullname" . }}-db }
  data:
    - secretKey: password
      remoteRef: { key: prod/shop/db, property: password }
```

No secret value ever enters the Helm release record. Use the API version your installed ESO release serves (`v1` on recent releases).

</details>

### Q186. How do you use Helm with Kustomize in Argo CD or kubectl? 🔴

<details>
<summary><b>Answer</b></summary>

Kustomize can inflate charts with its `helmCharts` field (requires `--enable-helm`):

```yaml
# kustomization.yaml
helmCharts:
  - name: ingress-nginx
    repo: https://kubernetes.github.io/ingress-nginx
    version: 4.11.3
    releaseName: ingress
    namespace: ingress-nginx
    valuesFile: values.yaml
patches:
  - path: add-annotations.yaml
```

```bash
kustomize build --enable-helm . | kubectl apply -f -
```

Argo CD supports it with `kustomize.buildOptions: --enable-helm`. Like Argo CD's Helm support, this is template-only: no release record, no Helm hooks execution.

</details>

### Q187. What is the "render in CI, commit manifests" pattern? 🔴

<details>
<summary><b>Answer</b></summary>

CI runs `helm template` for each environment and commits the rendered YAML to an environment repo (or branch); Argo CD or Flux applies plain manifests. Benefits: reviewers see the exact YAML that will hit the cluster, no templating at deploy time, easy diffs. Costs: large diffs, generated files in Git, and lost Helm features (hooks become Argo CD hooks, no `lookup`). It works well for strict audit environments; the hydrator features in newer Argo CD versions automate it.

</details>

### Q188. How does Flux handle Helm differently from Argo CD? 🔴

<details>
<summary><b>Answer</b></summary>

Flux's helm-controller uses the Helm SDK to perform real `install`/`upgrade` operations declared in a `HelmRelease` custom resource. So releases do exist in Helm storage, `helm list` shows them, hooks run as Helm hooks, `lookup` works, and Flux can run `helm test` and remediate failures (retries, rollback) per the `HelmRelease` spec. Argo CD only templates. In interviews, this is the cleanest one-line distinction between the two GitOps tools' Helm support.

</details>

---

## Best Practices & Rapid-Fire Commands

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q189. What are your top best practices for chart authoring? 🟡

<details>
<summary><b>Answer</b></summary>

- Prefix every named template with the chart name; use `include`, never `template`.
- Separate `labels` and `selectorLabels`; never put versions in selectors.
- Use `.Release.Namespace`, never hardcode namespaces.
- Safe defaults: resources, probes, non-root, read-only root FS, `automountServiceAccountToken: false` unless needed.
- Optional features behind `enabled` flags; validate with `values.schema.json`.
- Checksum annotations for config; quote strings, convert numbers.
- Deterministic output (no random values without `lookup`), so GitOps diffs stay clean.
- Document every value; bump `version` on every change.

</details>

### Q190. What are your top best practices for releasing with Helm in production? 🟡

<details>
<summary><b>Answer</b></summary>

- One CI command: `helm upgrade --install --atomic --wait --timeout` with all values files explicit; no `--reuse-values`.
- Pin chart versions (or digests) and Helm itself; charts come from an OCI registry, not a laptop directory.
- Gate with `helm lint --strict`, `helm template | kubeconform`, `helm diff` on the PR, `--dry-run=server` before apply.
- Serialize deploys per environment; keep `--history-max` around 10–20.
- Run `helm test` after every upgrade; alert on failed releases.
- Backward-compatible migrations; documented rollback runbook.
- Secrets encrypted (SOPS) or external (ESO), never plaintext in Git.

</details>

### Q191. How do you set resource requests and limits in a chart without forcing them on users? 🟡

<details>
<summary><b>Answer</b></summary>

Provide sensible defaults in `values.yaml` and render them with `toYaml`, so users can override or null out individual fields:

```yaml
# values.yaml
api:
  resources:
    requests: { cpu: 100m, memory: 128Mi }
    limits: { memory: 256Mi }
```

```yaml
          resources:
            {{- toYaml .Values.api.resources | nindent 12 }}
```

Many teams set memory limits but omit CPU limits to avoid CFS throttling; the chart should not hardcode that opinion, only default it.

</details>

### Q192. How do you keep an HPA and Helm from fighting over `replicas`? 🔴

<details>
<summary><b>Answer</b></summary>

If the Deployment template always sets `replicas`, every `helm upgrade` resets the count to the chart value, undoing the HPA's scaling until it reacts again (a capacity dip during deploys). Omit `replicas` when autoscaling is enabled:

```yaml
spec:
  {{- if not .Values.api.autoscaling.enabled }}
  replicas: {{ .Values.api.replicaCount }}
  {{- end }}
```

With Argo CD, also add `ignoreDifferences` on `/spec/replicas` if an older revision set it. Under server-side apply, field ownership makes this cleaner.

</details>

### Q193. How would you migrate a release whose stored manifest uses an API removed in the new Kubernetes version? 🔴

<details>
<summary><b>Answer</b></summary>

After a cluster upgrade, Helm cannot build the previous release's objects (`unable to build kubernetes objects from current release manifest: resource mapping not found for name: "shop" ... no matches for kind "PodDisruptionBudget" in version "policy/v1beta1"`). Fix the stored release with the mapkubeapis plugin, then upgrade with the updated chart:

```bash
helm plugin install https://github.com/helm/helm-mapkubeapis
helm mapkubeapis shop -n shop --dry-run
helm mapkubeapis shop -n shop
helm upgrade shop ./shop -n shop -f values-prod.yaml
```

Prevent it by updating chart API versions and running `pluto` or `kubent` before every cluster upgrade.

</details>

### Q194. Rapid fire: commands to inspect a release. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
helm list -A                                  # all releases, all namespaces
helm status shop -n shop --show-resources     # status plus resources
helm history shop -n shop --max 10
helm get values shop -n shop --all
helm get manifest shop -n shop --revision 4
kubectl get secrets -n shop -l owner=helm,name=shop
```

</details>

### Q195. Rapid fire: commands to work with charts locally. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
helm create shop
helm lint ./shop --strict -f values-prod.yaml
helm template shop ./shop -f values-dev.yaml -s templates/api-deployment.yaml
helm dependency update ./shop && helm dependency list ./shop
helm package ./shop --version 1.4.0 --app-version 2.3.0 -d dist/
helm show values ./shop
```

</details>

### Q196. Rapid fire: commands for repositories and OCI. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami && helm repo update
helm search repo bitnami/postgresql --versions | head -5
helm pull bitnami/postgresql --version 16.4.1 --untar
helm registry login ghcr.io -u "$USER" --password-stdin <<< "$TOKEN"
helm push dist/shop-1.4.0.tgz oci://ghcr.io/devopsshack/charts
helm pull oci://ghcr.io/devopsshack/charts/shop --version 1.4.0
```

</details>

### Q197. Rapid fire: commands for safe deploys and rollbacks. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
helm diff upgrade shop ./shop -n shop -f values-prod.yaml
helm upgrade --install shop ./shop -n shop --create-namespace -f values-prod.yaml \
  --atomic --wait --timeout 10m --cleanup-on-fail --history-max 15 --description "sha $GIT_SHA"
helm test shop -n shop --logs
helm rollback shop 0 -n shop --wait        # 0 (or omitted) = previous revision
helm uninstall shop -n shop --keep-history
```

</details>

### Q198. Rapid fire: commands to recover a stuck release. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
helm history shop -n shop                     # find the pending-* revision and last deployed one
helm rollback shop 7 -n shop                  # roll back to last good revision (clears pending)
# last resort if rollback refuses:
kubectl delete secret sh.helm.release.v1.shop.v8 -n shop     # the pending revision only
# pending-install with no good revision:
helm uninstall shop -n shop
```

Check first that no other Helm process (another pipeline) is actually running.

</details>

### Q199. Rapid fire: plugins every Helm user should know. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
helm plugin install https://github.com/databus23/helm-diff          # helm diff
helm plugin install https://github.com/jkroepke/helm-secrets        # SOPS integration
helm plugin install https://github.com/helm/helm-mapkubeapis        # fix removed APIs in releases
helm plugin install https://github.com/helm-unittest/helm-unittest  # unit tests for templates
helm plugin list
helm plugin update diff
```

helm-unittest lets you assert on rendered output (`helm unittest ./shop`) with YAML test suites in `tests/`, which catches template regressions in CI without a cluster.

</details>

### Q200. Rapid fire: one-line answers to classic Helm interview questions. 🟡

<details>
<summary><b>Answer</b></summary>

- **Where are releases stored?** Secrets `sh.helm.release.v1.<name>.v<rev>` in the release namespace.
- **include vs template?** `include` returns a string you can pipe; `template` cannot be piped.
- **Highest-precedence values?** `--set` family, then `-f` files right to left, then parent values, then chart defaults.
- **version vs appVersion?** Chart package SemVer vs the deployed application's version.
- **Does rollback undo migrations?** No; design expand/contract migrations.
- **Why `--atomic`?** Failed upgrade rolls back automatically instead of leaving a half-broken release.
- **"another operation is in progress"?** A `pending-*` revision left by a killed process; roll back or delete that revision Secret.
- **Does Argo CD use `helm install`?** No, `helm template` plus its own apply.

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
