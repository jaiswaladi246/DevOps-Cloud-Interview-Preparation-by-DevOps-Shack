# Helm — Cheatsheet

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

> Dense reference for daily use and interview prep. Helm 3.17+ syntax (Helm 4 notes where flags changed), helm-diff, OCI registries, Helmfile and helm-secrets.

[⬅ Back to Helm](./README.md) · [🏠 Home](../README.md)

## Contents

[Install and upgrade](#install-and-upgrade) · [Inspect releases](#inspect-releases) · [History and rollback](#history-and-rollback) · [Render, lint, diff](#render-lint-and-diff) · [Repositories](#repositories) · [Package and OCI](#package-and-oci) · [Dependencies](#dependencies) · [Plugins and env](#plugins-and-environment) · [Chart layout](#chart-layout) · [Chart.yaml](#chartyaml) · [Built-in objects](#built-in-objects) · [Template functions](#template-functions) · [Template patterns](#template-patterns) · [Values precedence](#values-precedence) · [Schema](#valuesschemajson) · [Hooks](#hooks-and-weights) · [Release storage](#release-storage) · [Helmfile and secrets](#helmfile-and-secrets) · [Errors](#common-errors-and-fixes) · [Top 15](#top-15-interview-one-liners)

---

## Install and upgrade

| Task | Command |
|------|---------|
| Install from a repo, pinned | `helm install web bitnami/nginx --version 18.2.4 -n web --create-namespace` |
| Install from OCI | `helm install shop oci://ghcr.io/acme/charts/shop --version 1.4.0 -n shop` |
| Install from a local dir or archive | `helm install shop ./charts/shop` · `helm install shop shop-1.4.0.tgz` |
| Generate the release name | `helm install bitnami/nginx --generate-name` |
| **CI-safe deploy (idempotent)** | `helm upgrade --install shop ./shop -n shop --create-namespace -f values-prod.yaml --atomic --wait --timeout 10m --history-max 10` |
| Wait for Jobs too | add `--wait-for-jobs` (with `--wait`) |
| Upgrade keeping previous values (risky) | `helm upgrade shop ./shop --reuse-values --set image.tag=1.5.0` |
| Upgrade: previous values as base, then new chart defaults merged | `helm upgrade shop ./shop --reset-then-reuse-values` (3.14+) |
| Reset to chart defaults + what you pass | `helm upgrade shop ./shop --reset-values -f values-prod.yaml` |
| Skip hooks | `--no-hooks` |
| Re-create pods via annotation instead of `--force` | `checksum/config` annotation (see patterns) |
| Force replace on conflict | `helm upgrade ... --force` (delete and recreate on patch failure, causes downtime) |
| Server-side validation without persisting | `helm upgrade --install shop ./shop --dry-run=server` |
| Uninstall | `helm uninstall shop -n shop` |
| Uninstall, keep history | `helm uninstall shop -n shop --keep-history` |
| Uninstall and wait for deletion | `helm uninstall shop -n shop --wait` |

Flag notes:

- `--atomic` implies `--wait`; on failure it rolls back an upgrade or uninstalls a first install. Helm 4 renames it to `--rollback-on-failure`.
- `--timeout` defaults to `5m0s` and applies to the whole operation, including each hook.
- `--wait` checks Deployments, StatefulSets, DaemonSets, Pods, PVCs and Services (LoadBalancer IP) for readiness. It does not wait for Jobs unless `--wait-for-jobs` is set.
- `--create-namespace` creates but never deletes the namespace, and the namespace is not part of the release.

### Setting values on the command line

| Flag | Example | Use it for |
|------|---------|-----------|
| `-f` / `--values` | `-f values.yaml -f values-prod.yaml` | Files, merged left to right |
| `--set` | `--set replicaCount=3,service.type=ClusterIP` | Quick overrides, types are guessed |
| `--set-string` | `--set-string image.tag=1.10` | Force string (tags, annotations, `"true"`) |
| `--set-file` | `--set-file config.nginx=./nginx.conf` | File contents as a value |
| `--set-json` | `--set-json 'tolerations=[{"key":"gpu","operator":"Exists"}]'` | Structured values |
| List index | `--set ingress.hosts[0].host=shop.example.com` | Lists (replaces the whole list) |
| Escape dots | `--set podAnnotations."prometheus\.io/scrape"=true` | Keys containing dots |
| Delete a default | `--set resources.limits=null` | Remove a key from defaults |

## Inspect releases

| Task | Command |
|------|---------|
| List releases in a namespace / all | `helm list -n shop` · `helm list -A` |
| Include failed, pending, uninstalled | `helm list -n shop -a` · `--failed` · `--pending` · `--uninstalled` |
| Filter by name regex | `helm list -A --filter '^shop'` |
| Machine-readable | `helm list -A -o json \| jq -r '.[] \| "\(.name) \(.status)"'` |
| Status of a release | `helm status shop -n shop` (add `--show-resources`) |
| User-supplied values | `helm get values shop -n shop` |
| All merged values | `helm get values shop -n shop --all` |
| Values of an old revision | `helm get values shop -n shop --revision 3` |
| Rendered manifest (no hooks) | `helm get manifest shop -n shop` |
| Hook manifests | `helm get hooks shop -n shop` |
| NOTES.txt output | `helm get notes shop -n shop` |
| Everything | `helm get all shop -n shop` |
| Chart metadata of the release | `helm get metadata shop -n shop` (3.13+) |
| Show chart info before installing | `helm show chart \| values \| readme \| crds \| all <chart> --version X` |

## History and rollback

| Task | Command |
|------|---------|
| History | `helm history shop -n shop` (`--max 5`, `-o yaml`) |
| Roll back to previous revision | `helm rollback shop -n shop` (or revision `0`) |
| Roll back to a specific revision | `helm rollback shop 4 -n shop --wait --timeout 5m` |
| Roll back and clean up new resources on failure | `helm rollback shop 4 --cleanup-on-fail` |
| Find the last deployed revision | `helm history shop -n shop -o json \| jq '[.[] \| select(.status=="deployed")][-1].revision'` |

Rollback re-applies revision N as a new revision (N+1 content = N). It runs `pre-rollback`/`post-rollback` hooks only, never reverts database migrations, and does not touch CRDs in `crds/` or PVCs.

## Render, lint and diff

| Task | Command |
|------|---------|
| Render locally | `helm template shop ./shop -n shop -f values-prod.yaml` |
| Render one file | `helm template shop ./shop -s templates/deployment.yaml` |
| Render with a target cluster version | `helm template ./shop --kube-version 1.31.0 --api-versions monitoring.coreos.com/v1` |
| Render to files | `helm template shop ./shop --output-dir out/` |
| Include CRDs in output | `helm template ./shop --include-crds` |
| Show bad YAML when rendering fails | `helm template ./shop --debug` |
| Server-side dry run (validates, `lookup` works) | `helm install shop ./shop --dry-run=server` |
| Lint | `helm lint ./shop -f values-prod.yaml --strict` |
| Lint chart and subcharts | `helm lint ./shop --with-subcharts` |
| Validate against K8s schemas | `helm template ./shop \| kubeconform -strict -summary -kubernetes-version 1.31.0` |
| Diff an upgrade (plugin) | `helm diff upgrade shop ./shop -n shop -f values-prod.yaml` |
| Diff with context lines, fail on change | `helm diff upgrade shop ./shop -C 3 --detailed-exitcode` (exit 2 = changes) |
| Diff two revisions | `helm diff revision shop 3 5 -n shop` |
| Diff a rollback | `helm diff rollback shop 3 -n shop` |
| Show secrets in diff (careful) | `helm diff upgrade ... --show-secrets` |

## Repositories

| Task | Command |
|------|---------|
| Add | `helm repo add bitnami https://charts.bitnami.com/bitnami` |
| Add private (basic auth) | `helm repo add acme https://charts.acme.io --username u --password-stdin` |
| Update index cache | `helm repo update` (or `helm repo update bitnami`) |
| List / remove | `helm repo list` · `helm repo remove bitnami` |
| Search added repos | `helm search repo nginx --versions` (`--devel` for pre-releases) |
| Search Artifact Hub | `helm search hub nginx` |
| Download a chart | `helm pull bitnami/nginx --version 18.2.4 --untar` |
| Build a classic repo index | `helm repo index ./repo --url https://charts.acme.io --merge ./repo/index.yaml` |

OCI registries are not "added": no `helm repo add`, no `helm repo update`, no `helm search` support. Reference them directly with `oci://`.

## Package and OCI

| Task | Command |
|------|---------|
| Package | `helm package ./shop -d dist/` |
| Package, overriding versions | `helm package ./shop --version 1.4.1 --app-version 2.3.0` |
| Package after resolving deps | `helm package ./shop -u` (`--dependency-update`) |
| Sign (PGP provenance) | `helm package ./shop --sign --key 'DevOps Shack' --keyring ~/.gnupg/secring.gpg` |
| Verify a signed chart | `helm verify shop-1.4.0.tgz` · `helm install --verify` |
| Log in to a registry | `helm registry login ghcr.io -u "$GITHUB_ACTOR" --password-stdin` |
| ECR login | `aws ecr get-login-password --region ap-south-1 \| helm registry login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com` |
| Push | `helm push dist/shop-1.4.0.tgz oci://ghcr.io/acme/charts` |
| Push to a plain-HTTP local registry | `helm push dist/shop-1.4.0.tgz oci://localhost:5001/charts --plain-http` |
| Pull from OCI | `helm pull oci://ghcr.io/acme/charts/shop --version 1.4.0` |
| Pull by digest | `helm pull oci://ghcr.io/acme/charts/shop@sha256:<digest>` |
| Show / template / install from OCI | `helm show values oci://ghcr.io/acme/charts/shop --version 1.4.0` |
| Log out | `helm registry logout ghcr.io` |

OCI rules: push to the namespace (`oci://host/charts`), Helm appends the chart name; the chart `version` becomes the tag; a `+` in SemVer is stored as `_` in the tag. ECR needs the repository (`charts/shop`) created first, and immutable tags reject re-pushing a version.

## Dependencies

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "16.x.x"                                    # SemVer range
    repository: "oci://registry-1.docker.io/bitnamicharts"  # or https://..., or file://../local-chart, or @repo-alias
    alias: db                                            # values key and name suffix become "db"
    condition: db.enabled                                # toggled by a parent value
    tags: [databases]                                    # toggled as a group: --set tags.databases=false
    import-values:
      - child: primary.service
        parent: dbService
```

| Task | Command |
|------|---------|
| Resolve ranges, download, write `Chart.lock` | `helm dependency update ./shop` |
| Download exactly what `Chart.lock` says | `helm dependency build ./shop` |
| Show dependency status | `helm dependency list ./shop` |

Rules: commit `Chart.lock`, gitignore `charts/*.tgz`, run `dependency build` in CI. A subchart reads its own key and `global`, never the parent's other values. `condition` wins over `tags`.

## Plugins and environment

| Task | Command |
|------|---------|
| Install plugins | `helm plugin install https://github.com/databus23/helm-diff` · `https://github.com/jkroepke/helm-secrets` · `https://github.com/helm-unittest/helm-unittest` · `https://github.com/helm/helm-mapkubeapis` |
| List / update / remove | `helm plugin list` · `helm plugin update diff` · `helm plugin uninstall diff` |
| Fix removed APIs in a stored release | `helm mapkubeapis shop -n shop --dry-run`, then without `--dry-run` |
| Unit-test templates | `helm unittest ./shop` (tests in `tests/*_test.yaml`) |
| Show Helm paths | `helm env` (`HELM_CACHE_HOME`, `HELM_CONFIG_HOME`, `HELM_DATA_HOME`) |
| Storage backend | `HELM_DRIVER=secret` (default), `configmap`, `memory`, `sql` |
| Use another kube context | `helm --kube-context prod-eks ...` or `--kubeconfig ./kc` |
| Debug logging | `helm upgrade ... --debug` |
| Shell completion | `source <(helm completion bash)` |

---

## Chart layout

```text
shop/
├── Chart.yaml            # required: apiVersion v2, name, version
├── Chart.lock            # generated by dependency update; commit it
├── values.yaml           # defaults = the chart's public API
├── values.schema.json    # optional JSON Schema, validated on install/upgrade/lint/template
├── .helmignore           # excluded from helm package
├── README.md / LICENSE
├── charts/               # vendored dependency .tgz files or unpacked subcharts
├── crds/                 # plain YAML CRDs: installed once, never templated, upgraded or deleted
└── templates/
    ├── _helpers.tpl      # files starting with "_" are not rendered as manifests
    ├── deployment.yaml
    ├── service.yaml
    ├── NOTES.txt         # rendered and printed after install/upgrade
    └── tests/
        └── test-connection.yaml   # helm.sh/hook: test
```

Install order: Helm sorts resources by kind (Namespace, NetworkPolicy, ResourceQuota, LimitRange, ServiceAccount, Secret, ConfigMap, StorageClass, PV, PVC, CRD, ClusterRole, Role, bindings, Service, DaemonSet, Pod, ReplicaSet, Deployment, HPA, StatefulSet, Job, CronJob, Ingress, ...). Uninstall uses the reverse order.

## Chart.yaml

```yaml
apiVersion: v2                 # v2 = Helm 3+ (v1 was Helm 2)
name: shop
description: Shack Shop application
type: application              # or library (only named templates, cannot be installed)
version: 1.4.0                 # chart SemVer: bump on every published change
appVersion: "2.3.0"            # app inside, quoted; often the default image tag
kubeVersion: ">=1.27.0-0"      # install fails on clusters outside the range
icon: https://devopsshack.com/icon.png
keywords: [ecommerce, nodejs]
home: https://devopsshack.com
sources: [https://github.com/acme/shop]
maintainers:
  - name: DevOps Shack
    email: office@devopsshack.com
annotations:
  artifacthub.io/changes: "- Add PodDisruptionBudget"
deprecated: false
dependencies: []               # see Dependencies
```

## Built-in objects

| Object | Useful fields |
|--------|---------------|
| `.Release` | `.Name`, `.Namespace`, `.Revision`, `.IsInstall`, `.IsUpgrade`, `.Service` (= `Helm`) |
| `.Chart` | `.Name`, `.Version`, `.AppVersion`, `.Type`, `.Annotations` (fields of `Chart.yaml`, capitalized) |
| `.Values` | Merged values |
| `.Capabilities` | `.KubeVersion.Version`, `.KubeVersion.Minor`, `.APIVersions.Has "policy/v1"`, `.HelmVersion` |
| `.Template` | `.Name` (current file path), `.BasePath` (`shop/templates`) |
| `.Files` | `.Get "conf/app.ini"`, `.Glob "migrations/*.sql"`, `.AsConfig`, `.AsSecrets`, `.Lines` |
| `$` | Root context, always available, even inside `range` and `with` |

## Template functions

| Function | Example | Notes |
|----------|---------|-------|
| `include` | `{{ include "shop.labels" . \| nindent 4 }}` | Returns a string, so it can be piped. Prefer over `template` |
| `template` | `{{ template "shop.labels" . }}` | Action, cannot be piped; avoid |
| `define` | `{{- define "shop.name" -}}...{{- end }}` | In `_helpers.tpl`; names are global, prefix with the chart name |
| `nindent` / `indent` | `{{- toYaml .Values.resources \| nindent 12 }}` | `nindent` adds a leading newline; pair it with `{{-` |
| `toYaml` | `{{- toYaml .Values.tolerations \| nindent 8 }}` | Inject maps and lists. Also `toJson`, `fromYaml`, `fromJson`, `toPrettyJson` |
| `required` | `{{ required "image.repository is required" .Values.image.repository }}` | Fails render with your message on empty value |
| `default` | `{{ .Values.image.tag \| default .Chart.AppVersion }}` | Treats `false`, `0`, `""` and empty as unset; use `hasKey` for booleans |
| `fail` | `{{- fail "minReplicas > maxReplicas" }}` | Abort with a message for invalid combinations |
| `tpl` | `{{ tpl .Values.extraConfig . }}` | Render a string from values as a template; pass the context |
| `quote` / `squote` | `{{ .Values.port \| quote }}` | Env values and annotations must be strings |
| `lookup` | `{{ lookup "v1" "Secret" .Release.Namespace "db" }}` | Reads the live cluster; empty in `helm template` and Argo CD |
| `printf` | `{{ printf "%s-%s" .Release.Name "api" }}` | Build names |
| `trunc` / `trimSuffix` | `{{ $n \| trunc 63 \| trimSuffix "-" }}` | DNS-1123 63-char limit |
| `sha256sum` | `{{ include (print $.Template.BasePath "/cm.yaml") . \| sha256sum }}` | Config checksum annotations |
| `b64enc` / `b64dec` | `{{ .Values.token \| b64enc }}` | For `data:` in Secrets (or use `stringData`) |
| `dict` / `list` | `{{ include "shop.image" (dict "image" .Values.api.image "root" $) }}` | Pass multiple parameters to a helper |
| `merge` / `mergeOverwrite` | `{{ $cfg := merge .Values.override .Values.base }}` | `merge`: left wins; `mergeOverwrite`: right wins |
| `hasKey` / `empty` / `coalesce` | `{{ if hasKey .Values "enabled" }}` · `{{ coalesce .Values.a .Values.b "x" }}` | Presence vs emptiness |
| `ternary` | `{{ ternary "Always" "IfNotPresent" .Values.dev }}` | Inline if/else |
| `semverCompare` | `{{ if semverCompare ">=1.25-0" .Capabilities.KubeVersion.Version }}` | Version-gate API choices |
| `int` / `toString` / `float64` | `containerPort: {{ .Values.port \| int }}` | Fix type surprises |
| `randAlphaNum` / `uuidv4` / `now` | `{{ randAlphaNum 24 }}` | New value every render: causes diffs and rollouts |
| Comparisons | `eq`, `ne`, `lt`, `gt`, `and`, `or`, `not` | Prefix form: `{{ if and .Values.a (eq .Values.b "x") }}` |
| Whitespace | `{{-` trims left, `-}}` trims right | Most "blank line" and YAML errors come from this |
| Comments | `{{/* not rendered */}}` | YAML `#` comments are rendered into output |

## Template patterns

**Name and label helpers (`_helpers.tpl`)**

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

{{- define "shop.selectorLabels" -}}
app.kubernetes.io/name: {{ default .Chart.Name .Values.nameOverride }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "shop.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" }}
{{ include "shop.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

**Blocks, optional sections, loops**

```yaml
metadata:
  labels:
    {{- include "shop.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
            - name: RELEASE
              value: {{ $.Release.Name | quote }}   # $ because range changed "."
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

**Booleans that default to true** (`default` would turn `false` into `true`):

```yaml
{{- $enabled := ternary .Values.metrics.enabled true (hasKey .Values.metrics "enabled") }}
```

**Generate a resource per list item**

```yaml
{{- range .Values.cronJobs }}
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{ include "shop.fullname" $ }}-{{ .name }}
spec:
  schedule: {{ .schedule | quote }}
  ...
{{- end }}
```

**Keep a generated password stable across upgrades** (does not work under Argo CD or `helm template`):

```yaml
{{- $existing := lookup "v1" "Secret" .Release.Namespace (include "shop.fullname" .) }}
{{- $pw := "" }}
{{- if $existing }}{{ $pw = index $existing.data "password" | b64dec }}{{ else }}{{ $pw = randAlphaNum 24 }}{{ end }}
stringData:
  password: {{ $pw | quote }}
```

**API version by cluster capability**

```yaml
{{- if .Capabilities.APIVersions.Has "autoscaling/v2" }}
apiVersion: autoscaling/v2
{{- else }}
apiVersion: autoscaling/v2beta2
{{- end }}
```

**Files into ConfigMaps**

```yaml
data:
  {{- (.Files.Glob "config/*").AsConfig | nindent 2 }}
  app.ini: |-
    {{- .Files.Get "files/app.ini" | nindent 4 }}
```

---

## Values precedence

Lowest to highest. The last writer wins; maps merge key by key, **lists are replaced whole**.

| # | Source | Example |
|---|--------|---------|
| 1 | Subchart's own `values.yaml` | `charts/postgresql/values.yaml` |
| 2 | Parent chart's `values.yaml` (under the subchart key or alias, plus `global`) | `db: { auth: { database: shop } }` |
| 3 | `-f` / `--values` files, left to right | `-f values.yaml -f values-prod.yaml` |
| 4 | `--set`, `--set-string`, `--set-file`, `--set-json` (later flags win over earlier ones) | `--set image.tag=1.5.0` |

Upgrade-time behavior:

| Flag | Starting point for the new revision |
|------|-----------------------------------|
| (none) | New chart defaults + what you pass now. Previous `--set` values are **dropped** |
| `--reuse-values` | Previous revision's merged values + what you pass; **new chart defaults are ignored** |
| `--reset-then-reuse-values` | New chart defaults, then previous user values, then what you pass (3.14+) |
| `--reset-values` | New chart defaults + what you pass (explicit form of the default) |

In CI always pass the complete set of `-f` files and avoid `--reuse-values`.

## values.schema.json

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["image"],
  "properties": {
    "replicaCount": { "type": "integer", "minimum": 1 },
    "image": {
      "type": "object",
      "required": ["repository"],
      "additionalProperties": false,
      "properties": {
        "repository": { "type": "string", "minLength": 1 },
        "tag": { "type": "string" },
        "pullPolicy": { "enum": ["Always", "IfNotPresent", "Never"] }
      }
    },
    "service": { "type": "object", "properties": { "type": { "enum": ["ClusterIP", "NodePort", "LoadBalancer"] } } }
  }
}
```

Validated on `install`, `upgrade`, `lint` and `template`, against the merged values. Subcharts validate their own section. Avoid `additionalProperties: false` at the root (Helm injects `global` and subchart keys). Skip validation in an emergency with `--skip-schema-validation` (3.16+).

---

## Hooks and weights

```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade,post-install
    "helm.sh/hook-weight": "-5"                     # string; ascending; default "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

| Hook | Runs |
|------|------|
| `pre-install` | After render, before any release resource is created |
| `post-install` | After all resources are created (and ready, with `--wait`) |
| `pre-upgrade` | After render, before any resource is updated |
| `post-upgrade` | After all resources are upgraded |
| `pre-rollback` / `post-rollback` | Around a rollback |
| `pre-delete` / `post-delete` | Around `helm uninstall` |
| `test` | Only on `helm test <release>` |

| Delete policy | Effect |
|---------------|--------|
| `before-hook-creation` | Delete the previous hook object before creating the new one (default when none is set) |
| `hook-succeeded` | Delete after the hook succeeds |
| `hook-failed` | Delete after the hook fails (you lose the logs; usually leave it out) |

Ordering within a hook phase: weight ascending, then kind, then name. Helm waits for Jobs and Pods to complete and for other kinds to be created. A failed hook fails the operation. Hook resources are **not** in the release manifest: `helm uninstall` and rollback do not remove them, and `helm get manifest` does not show them (`helm get hooks` does).

Related annotations:

| Annotation | Effect |
|------------|--------|
| `helm.sh/resource-policy: keep` | Resource is not deleted on uninstall or when removed from the chart |
| `meta.helm.sh/release-name` / `meta.helm.sh/release-namespace` + label `app.kubernetes.io/managed-by: Helm` | Ownership metadata; add them to adopt an existing object |

Tests:

```bash
helm test shop -n shop --logs                     # run all test hooks, print pod logs
helm test shop -n shop --filter name=shop-test-api
helm test shop -n shop --timeout 2m
```

## Release storage

| Item | Value |
|------|-------|
| Default driver | Secrets in the release namespace, type `helm.sh/release.v1` |
| Name | `sh.helm.release.v1.<release>.v<revision>` |
| Labels | `owner=helm`, `name=<release>`, `status=<deployed\|superseded\|failed\|pending-*>`, `version=<revision>` |
| Payload | `.data.release` = base64 (Kubernetes) of base64 (Helm) of gzip of JSON |
| Size limit | 1 MiB per Secret: huge charts or `.Files` can hit it |

```bash
kubectl get secret -n shop -l owner=helm -L status,version
kubectl get secret sh.helm.release.v1.shop.v5 -n shop -o jsonpath='{.data.release}' \
  | base64 -d | base64 -d | gunzip | jq '.info.status, .chart.metadata.version, .config'
```

Statuses: `unknown`, `deployed`, `uninstalled`, `superseded`, `failed`, `uninstalling`, `pending-install`, `pending-upgrade`, `pending-rollback`.

---

## Helmfile and secrets

```yaml
# helmfile.yaml.gotmpl (Helmfile v1 only renders {{ }} templates in .gotmpl files)
environments:
  dev:
    values: [env/dev.yaml]
  prod:
    values: [env/prod.yaml]
---
repositories:
  - name: ingress-nginx
    url: https://kubernetes.github.io/ingress-nginx
releases:
  - name: ingress-nginx
    namespace: ingress-nginx
    chart: ingress-nginx/ingress-nginx
    version: 4.11.3
  - name: shop
    namespace: shop
    chart: oci://ghcr.io/acme/charts/shop
    version: 1.4.0
    needs: [ingress-nginx/ingress-nginx]
    values: [values/shop-{{ .Environment.Name }}.yaml]
    secrets: [secrets/shop-{{ .Environment.Name }}.yaml]
```

| Task | Command |
|------|---------|
| Preview | `helmfile -e prod diff` |
| Converge (diff, then upgrade changed releases) | `helmfile -e prod apply` |
| Upgrade everything | `helmfile -e prod sync` |
| Render | `helmfile -e prod template` |
| One release | `helmfile -e prod -l name=shop apply` |
| Encrypt a values file (SOPS + age) | `sops -e -i secrets/shop-prod.yaml` (rules in `.sops.yaml`) |
| Edit encrypted file | `sops secrets/shop-prod.yaml` |
| Use with plain Helm | `helm upgrade --install shop ./shop -f secrets://secrets/shop-prod.yaml` (or `helm secrets upgrade ... -f secrets/shop-prod.yaml`) |

---

## Common errors and fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress` | Previous run killed; latest revision stuck in `pending-*` | `helm history`; `helm rollback <rel> <last-good>`; last resort delete only the `status=pending-*` Secret |
| `Error: INSTALLATION FAILED: cannot re-use a name that is still in use` | Release exists (maybe `failed`) | Use `helm upgrade --install`; check `helm list -a` |
| `Error: rendered manifests contain a resource that already exists. Unable to continue with install: ... invalid ownership metadata` | Object created outside Helm or by another release | Delete it, or adopt: label `app.kubernetes.io/managed-by=Helm`, annotations `meta.helm.sh/release-name`, `meta.helm.sh/release-namespace` |
| `Error: UPGRADE FAILED: cannot patch "x" with kind Deployment: ... spec.selector: Invalid value: ...: field is immutable` | Selector labels changed | Keep selector labels stable; delete with `--cascade=orphan` and re-upgrade, or rename |
| `Error: YAML parse error on shop/templates/x.yaml: error converting YAML to JSON: yaml: line N: did not find expected key` | Bad indentation, missing `{{-`, wrong `nindent` | `helm template --debug -s templates/x.yaml`; fix `nindent` value |
| `Error: template: shop/templates/x.yaml:12:20: executing ... at <.Values.foo.bar>: nil pointer evaluating interface {}.bar` | Parent key missing, or `.` changed inside `range`/`with` | Guard with `with`/`dig`, add the default, or use `$` |
| `error calling include: template: no template "shop.labels" associated with template "gotpl"` | Helper name typo or defined in another chart's namespace | Check the `define` name and chart prefix |
| `Error: values don't meet the specifications of the schema(s) in the following chart(s): ...` | Schema rejected values | Fix the value type (`--set-string` for tags) or the schema |
| `Error: execution error at (shop/templates/secret.yaml:8:...): db.password is required` | `required` triggered | Supply the value per environment |
| `Error: found in Chart.yaml, but missing in charts/ directory: postgresql` | Dependencies not vendored | `helm dependency build` |
| `Error: the lock file (Chart.lock) is out of sync with the dependencies file (Chart.yaml)` | `Chart.yaml` deps changed | `helm dependency update` and commit `Chart.lock` |
| `Error: UPGRADE FAILED: release shop failed, and has been rolled back due to atomic being set: context deadline exceeded` | Not Ready within `--timeout` | `kubectl get events --sort-by=.lastTimestamp`; fix image, probes, resources |
| `Error: failed pre-upgrade: ... job failed: BackoffLimitExceeded` / `failed post-install: ...` | Hook Job failed | `kubectl logs job/<name>`; keep failed hooks (no `hook-failed` policy) |
| `Error: UPGRADE FAILED: unable to build kubernetes objects from current release manifest: ... no matches for kind "PodDisruptionBudget" in version "policy/v1beta1"` | Stored manifest uses a removed API | `helm mapkubeapis <rel>`, then upgrade |
| `Error: INSTALLATION FAILED: chart requires kubeVersion: >=1.27.0-0 which is incompatible with Kubernetes v1.26.x` | `kubeVersion` constraint | Upgrade the cluster or relax the constraint (keep the `-0` suffix for pre-release/EKS versions) |
| `http: server gave HTTP response to HTTPS client` | Plain-HTTP OCI registry | `--plain-http` (local only) |
| `Error: ... 401 Unauthorized` / `denied: requested access to the resource is denied` on push/pull | Not logged in or token lacks scope | `helm registry login`; GHCR token needs `write:packages` |
| `Error: no repository definition for https://...` | Dependency repo not added | `helm repo add` it, or use `oci://` / `@alias` |
| Annotation error: `cannot unmarshal bool into Go struct field ObjectMeta.metadata.annotations of type string` | `--set` turned `"true"` into a boolean | `--set-string` or `\| quote` in the template |
| Image tag `1.10` deployed as `1.1` | `--set` parsed it as a float | `--set-string image.tag=1.10` and quote tags in values |
| Release Secret too large: `Secret "sh.helm.release.v1..." is invalid: data: Too long: must have at most 1048576 bytes` | Big chart or embedded files | Trim `.Files`, use `.helmignore`, split the chart, or `HELM_DRIVER=sql` |

---

## Top 15 interview one-liners

1. **Helm 3 vs Helm 2:** Helm 3 has no Tiller; the client talks to the API server with your own RBAC and stores releases as Secrets in the release namespace.
2. **Chart vs release vs revision:** a chart is the package, a release is one installation of it under a name in a namespace, and every install, upgrade or rollback creates a new revision.
3. **The CI deploy command:** `helm upgrade --install <rel> <chart> -n <ns> -f values-<env>.yaml --atomic --wait --timeout 10m --history-max 10`.
4. **Values precedence:** subchart defaults < parent `values.yaml` < `-f` files left to right < `--set` flags; maps merge, lists replace.
5. **`version` vs `appVersion`:** `version` is the chart's SemVer and changes on every published chart change; `appVersion` is the app inside and often the default image tag.
6. **`include` vs `template`:** `include` returns a string you can pipe into `nindent`; `template` cannot be piped, so always use `include`.
7. **Why `$`:** `range` and `with` rebind `.`; `$` always points at the root context, so use `$.Values` and `$.Release` inside them.
8. **What rollback does:** it re-applies an old revision as a new revision; it never reverts database migrations, CRDs or PVC data.
9. **`--atomic`:** implies `--wait`, rolls back a failed upgrade (or uninstalls a failed install) automatically; renamed `--rollback-on-failure` in Helm 4.
10. **Stuck `pending-upgrade`:** a killed Helm process left it; check `helm history`, roll back to the last deployed revision, and only as a last resort delete that pending release Secret.
11. **Hooks:** annotated resources that run at lifecycle points, ordered by `hook-weight` ascending, cleaned up by `hook-delete-policy`, and not part of the release manifest.
12. **Migration hook timing:** `pre-upgrade` for upgrades, but `post-install` on first install if the database is in the same release; migrations must be backward compatible because rollback does not undo them.
13. **Three-way merge:** Helm patches using the old manifest, the live object and the new manifest, so it reverts drift on fields the chart sets but leaves fields it never set.
14. **Validate before deploying:** `helm lint --strict`, `helm template | kubeconform`, `values.schema.json`, `--dry-run=server`, and `helm diff upgrade` posted on the PR.
15. **Helm under Argo CD:** Argo CD runs the equivalent of `helm template` and applies the output itself, so `helm list` shows nothing, `lookup` is empty, random values cause drift and hooks map to PreSync/PostSync.

[⬅ Back to Helm](./README.md) · [🏠 Home](../README.md)

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
