# Helm — Assignments

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

> 25 practical assignments · 🟢 10 Beginner · 🟡 10 Intermediate · 🔴 5 Advanced

[⬅ Back to Helm](./README.md) · [🏠 Home](../README.md)

Use a local kind or minikube cluster with Helm 3.17+ (Helm 4 works too; rename `--atomic` to `--rollback-on-failure` if you want to avoid deprecation warnings). Install the diff plugin once with `helm plugin install https://github.com/databus23/helm-diff`. Keep every chart you write in a Git repository: several later assignments build on earlier ones. Open a hint only after you have tried for 10 minutes.

## Table of contents
- [🟢 Beginner](#beginner) — A1–A10
- [🟡 Intermediate](#intermediate) — A11–A20
- [🔴 Advanced](#advanced) — A21–A25

---

## Beginner

### A1. Install a public chart and read what it did 🟢

**Objective:** Get comfortable with the basic release workflow against a real chart.

**Tasks**
1. Add the Bitnami repository (or use `oci://registry-1.docker.io/bitnamicharts/nginx` directly) and run `helm search repo nginx --versions | head`.
2. Run `helm show chart`, `helm show values` and `helm show readme` for the nginx chart and note the chart `version` and `appVersion`.
3. Install it as release `web` in a new namespace `web-lab` with a pinned `--version`.
4. List the release, show its status, and list every Kubernetes object it created.
5. Print the rendered manifest Helm stored for the release.

**Expected outcome:** `helm list -n web-lab` shows `web` as `deployed`, revision 1, and you can explain the difference between the chart version and the app version from the `show chart` output.

<details>
<summary>Hints</summary>

`helm install web bitnami/nginx -n web-lab --create-namespace --version <x.y.z>`. `helm status web -n web-lab`, `helm get manifest web -n web-lab`, and `kubectl get all -n web-lab -l app.kubernetes.io/instance=web`. Always pin `--version`; without it you get whatever is latest today.

</details>

**Bonus:** Install the same chart a second time as `web2` in the same namespace and explain why two releases of one chart do not collide.

### A2. Override values three ways 🟢

**Objective:** Learn values precedence by observing it, not memorizing it.

**Tasks**
1. Save the default values of the nginx chart from A1 to `defaults.yaml`.
2. Create `small.yaml` setting `replicaCount: 2` and the Service type to `ClusterIP`.
3. Create `big.yaml` setting `replicaCount: 4`.
4. Upgrade `web` with `-f small.yaml -f big.yaml --set replicaCount=3` and predict the result before running it.
5. Compare `helm get values web` with `helm get values web --all`.

**Expected outcome:** The Deployment runs 3 replicas, and you can state the order: chart defaults < `-f` files left to right < `--set`. You can explain why `get values` without `--all` only shows the user-supplied values.

<details>
<summary>Hints</summary>

`helm show values bitnami/nginx > defaults.yaml`. Check the result with `kubectl get deploy -n web-lab`. Then drop the `--set` and rerun to see `big.yaml` win over `small.yaml`.

</details>

**Bonus:** Set an annotation value of `"true"` with `--set` and then with `--set-string`, and show which one Kubernetes rejects and why.

### A3. Upgrade, history and rollback 🟢

**Objective:** Practice the release lifecycle and read the history like an on-call engineer.

**Tasks**
1. Make three upgrades of `web`, each changing one value (replicas, a resource limit, an extra label).
2. Run `helm history web -n web-lab` and describe each revision's status.
3. Roll back to revision 2.
4. Run `helm history` again and explain the new revision number.
5. Use `helm get values web --revision 2` and `--revision 4` to prove the rollback restored revision 2's values.

**Expected outcome:** History shows revisions 1 to 5, with revision 5 described as `Rollback to 2`, and its values match revision 2.

<details>
<summary>Hints</summary>

`helm rollback web 2 -n web-lab --wait`. Rollback creates a new revision with old content; it never deletes or rewrites history.

</details>

**Bonus:** Add `--history-max 3` to an upgrade and count the release Secrets before and after.

### A4. Find and decode the release Secret 🟢

**Objective:** Understand where Helm 3 keeps its state.

**Tasks**
1. List the Secrets in `web-lab` that belong to Helm, showing their `status` and `version` labels.
2. Decode the current revision's Secret and print the release status, chart version and the first 20 lines of the manifest.
3. Explain the two layers of base64 and the gzip.
4. In a throwaway namespace, install a chart, delete all its release Secrets, and run `helm list`.
5. Show that the workloads are still running.

**Expected outcome:** You can pull any field out of a release Secret with one pipeline, and you can explain why deleting the Secrets makes Helm forget the release without touching the workloads.

<details>
<summary>Hints</summary>

`kubectl get secret -n web-lab -l owner=helm -L status,version`. Decode with `kubectl get secret sh.helm.release.v1.web.v3 -n web-lab -o jsonpath='{.data.release}' | base64 -d | base64 -d | gunzip | jq '.info.status, .chart.metadata.version'`.

</details>

**Bonus:** Explain what `HELM_DRIVER=configmap` changes and why Secrets are the default.

### A5. Scaffold a chart and explain every file 🟢

**Objective:** Know the chart anatomy well enough to defend each file in an interview.

**Tasks**
1. Run `helm create hello` and list the tree.
2. Write one sentence for each file and folder on what it does and when it is used.
3. Change the image to `nginxinc/nginx-unprivileged` with tag `1.27-alpine` and the container port to 8080.
4. Delete the files you do not need (for example `hpa.yaml`, `ingress.yaml`, `serviceaccount.yaml`) and fix any template that referenced them.
5. Install it, port-forward to it and get a 200.

**Expected outcome:** A minimal working chart with a Deployment and a Service, installed and reachable, plus your file-by-file notes.

<details>
<summary>Hints</summary>

Check `values.yaml` for `serviceAccount.create` and `ingress.enabled` references left in other templates, and `NOTES.txt`, which references the Ingress and Service type. `helm lint hello` tells you what broke after deleting files.

</details>

**Bonus:** Add a `.helmignore` rule and prove with `helm package` plus `tar -tzf` that the file is excluded.

### A6. Render offline and lint 🟢

**Objective:** Use the tools that catch mistakes before a cluster ever sees them.

**Tasks**
1. Render your `hello` chart with `helm template` and save the output.
2. Render only the Deployment with `-s templates/deployment.yaml`.
3. Break the chart three ways (a missing `{{ end }}`, a wrong indentation, a reference to a value that does not exist) and record each error from `helm lint` and `helm template`.
4. Run `helm lint --strict` and explain which extra issues it reports.
5. Compare `helm template` with `helm install --dry-run=server`.

**Expected outcome:** A short table of the three errors with their exact messages and fixes, and a one-line explanation of when to use `template` vs `--dry-run=server`.

<details>
<summary>Hints</summary>

A missing value does not fail by default; it renders as empty or `<no value>`. `--strict` turns lint warnings into failures. `--dry-run=server` contacts the API server, so it validates against real API versions and lets `lookup` return data.

</details>

**Bonus:** Pipe `helm template` into `kubeconform -strict -summary` and make it pass.

### A7. Chart version vs appVersion 🟢

**Objective:** Version charts correctly.

**Tasks**
1. Set `version: 0.1.0` and `appVersion: "1.27.0"` in `hello/Chart.yaml`.
2. Make the image tag default to `.Chart.AppVersion` when `image.tag` is empty.
3. Make a template-only change and decide which version to bump.
4. Ship a new app version and decide which versions to bump.
5. Package each version and show the archive names.

**Expected outcome:** `hello-0.1.0.tgz`, `hello-0.1.1.tgz` (template fix) and `hello-0.2.0.tgz` (new app), with `helm show chart` reflecting the right `appVersion` in each.

<details>
<summary>Hints</summary>

`image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"`. `helm package hello --version 0.1.1 --app-version 1.27.0` can override both at package time, which is how CI usually does it.

</details>

**Bonus:** Explain why `appVersion` should be quoted in `Chart.yaml`.

### A8. NOTES.txt that helps the person who just installed 🟢

**Objective:** Write useful post-install output with conditionals.

**Tasks**
1. Write a `NOTES.txt` that prints the release name, namespace, chart version and revision.
2. Print a `kubectl port-forward` command when the Service type is `ClusterIP`.
3. Print the external IP lookup command when the type is `LoadBalancer`.
4. Print the Ingress URL when `ingress.enabled` is true.
5. Re-display the notes later without reinstalling.

**Expected outcome:** Installing with different values prints different, copy-pasteable instructions, and `helm get notes` shows them again.

<details>
<summary>Hints</summary>

Use `{{ .Release.Name }}`, `{{ .Release.Namespace }}`, `{{ .Chart.Version }}`, `{{ .Release.Revision }}`, and `{{- if eq .Values.service.type "LoadBalancer" }}`. `helm get notes <release>`.

</details>

**Bonus:** Make NOTES.txt print a warning when `replicaCount` is 1.

### A9. Repositories and searching 🟢

**Objective:** Work with classic HTTP chart repositories confidently.

**Tasks**
1. Add two repositories (for example `prometheus-community` and `ingress-nginx`).
2. Update the repo cache and search both for a chart with `--versions`.
3. Pull a chart archive with a pinned version and untar it.
4. Remove one repository and show that installs from it now fail.
5. Explain what `index.yaml` is and where Helm caches it.

**Expected outcome:** You can add, update, search, pull and remove repos, and you can show the cached `index.yaml` files on disk.

<details>
<summary>Hints</summary>

`helm repo add`, `helm repo update`, `helm search repo ingress-nginx/ingress-nginx --versions`, `helm pull ingress-nginx/ingress-nginx --version <v> --untar`. The cache lives under `helm env HELM_REPOSITORY_CACHE`.

</details>

**Bonus:** Build your own HTTP repo: `helm package`, `helm repo index . --url http://localhost:8000`, serve it with `python3 -m http.server 8000`, add it and install from it.

### A10. Uninstall cleanly 🟢

**Objective:** Know exactly what `helm uninstall` removes and what it leaves behind.

**Tasks**
1. Install a chart that creates a StatefulSet with `volumeClaimTemplates` (for example a database chart).
2. Add a resource with the annotation `helm.sh/resource-policy: keep` to your `hello` chart and install it.
3. Uninstall both releases.
4. List what is left in each namespace.
5. Uninstall another release with `--keep-history` and show it in `helm list -a`.

**Expected outcome:** You can name three things that survive `helm uninstall` (PVCs from StatefulSets, `resource-policy: keep` resources, hook resources) and the cleanup command for each.

<details>
<summary>Hints</summary>

`kubectl get pvc,cm,secret -n <ns>` after uninstalling. `helm list -n <ns> -a` (or `--uninstalled`) shows releases kept with `--keep-history`.

</details>

**Bonus:** Use `helm uninstall --wait` and explain what it waits for.

---

## Intermediate

### A11. Helpers for names and labels 🟡

**Objective:** Build the naming and labelling foundation every good chart has.

**Tasks**
1. In `_helpers.tpl`, define `<chart>.name`, `<chart>.fullname`, `<chart>.chart`, `<chart>.labels` and `<chart>.selectorLabels`.
2. Make `fullname` respect `fullnameOverride`, use `nameOverride`, avoid repeating the chart name if the release name contains it, and truncate to 63 characters.
3. Use the helpers in the Deployment, Service and every other template.
4. Keep only stable labels in `spec.selector`.
5. Install releases named `hello`, `hello-dev` and `team-a` and show the resulting resource names.

**Expected outcome:** Names are `hello`, `hello-dev` and `team-a-hello`; every object carries the recommended `app.kubernetes.io/*` labels; selectors contain only `name` and `instance`.

<details>
<summary>Hints</summary>

`{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}` and `{{- if contains $name .Release.Name }}`. Use `include` (not `template`) so you can pipe into `nindent`. Prefix every helper name with the chart name because names are global across subcharts.

</details>

**Bonus:** Upgrade a release after changing a selector label and capture the `field is immutable` error.

### A12. Lists, maps and context: `range`, `with`, `$` 🟡

**Objective:** Master the template constructs behind most rendering bugs.

**Tasks**
1. Add `env` as a map in values and render it as a list of `name`/`value` pairs with `range $k, $v`.
2. Add `extraVolumes` and `extraVolumeMounts` as lists rendered with `toYaml | nindent`.
3. Use `with` for optional blocks (`nodeSelector`, `tolerations`, `podAnnotations`) so empty values render nothing.
4. Inside a `with` block, reference `.Release.Name` correctly.
5. Render with and without each value set and diff the output.

**Expected outcome:** No empty keys like `nodeSelector:` with nothing under them, no `<no value>` strings, and correct indentation in every combination.

<details>
<summary>Hints</summary>

Inside `with` and `range`, `.` changes; use `$.Release.Name`. `{{- with .Values.nodeSelector }}` then `nodeSelector:` then `{{- toYaml . | nindent 8 }}` then `{{- end }}`. Quote env values with `| quote`.

</details>

**Bonus:** Sort the env map keys explicitly and explain whether `range` over a map is already ordered in Helm.

### A13. Guardrails: `required`, `fail` and `values.schema.json` 🟡

**Objective:** Make bad values fail fast with clear messages.

**Tasks**
1. Make `image.repository` required with a custom message using `required`.
2. Use `fail` to reject `autoscaling.minReplicas > autoscaling.maxReplicas`.
3. Write `values.schema.json` that enforces `replicaCount` as an integer of at least 1, `service.type` as an enum, and `image` with no unknown keys.
4. Make `database.password` required only when `database.enabled` is true, using `if`/`then`.
5. Trigger each guardrail and capture the error.

**Expected outcome:** Five distinct, readable errors, all raised by `helm template` before anything reaches a cluster.

<details>
<summary>Hints</summary>

`{{ required "image.repository is required" .Values.image.repository }}`. In the schema use `"additionalProperties": false` on `image`, and `"if": {"properties": {"enabled": {"const": true}}}, "then": {"required": ["password"]}`. Do not set `additionalProperties: false` at the root if you have subcharts or `global`.

</details>

**Bonus:** Explain why the schema rejects `--set replicaCount=two` but a `required` check would not catch a wrong type.

### A14. Subchart dependency with alias, condition and globals 🟡

**Objective:** Compose charts the way platform teams do.

**Tasks**
1. Add a Redis or PostgreSQL chart as a dependency of `hello` with a pinned version range.
2. Give it an `alias` and a `condition`, and configure it from the parent's `values.yaml`.
3. Add `global.imageRegistry` and use it in both the parent and your own local subchart.
4. Run `helm dependency update`, commit `Chart.lock`, and rebuild from a clean clone with `helm dependency build`.
5. Install with the dependency enabled, then upgrade with it disabled and an external host configured.

**Expected outcome:** Resources named with the alias, the dependency toggled cleanly by its condition, and a reproducible build from `Chart.lock`.

<details>
<summary>Hints</summary>

In `Chart.yaml`: `dependencies: - name: postgresql, version: "16.x.x", repository: "oci://registry-1.docker.io/bitnamicharts", alias: db, condition: db.enabled`. Values for the subchart go under `db:`. A subchart sees `.Values.global` but never the parent's other values.

</details>

**Bonus:** Use `import-values` to pull a value from the subchart up into the parent.

### A15. A migration hook with weights and delete policies 🟡

**Objective:** Run a one-off task at the right moment of a release.

**Tasks**
1. Write a Job that runs a SQL or shell migration, annotated as a `pre-upgrade` hook.
2. Ship the migration script in a ConfigMap that is also a hook with a lower weight.
3. Set a delete policy that recreates the Job on the next run and keeps it when it fails.
4. Make the Job fail on purpose and observe what Helm reports and what remains in the cluster.
5. Explain why `pre-install` would be the wrong hook if the database is part of the same release.

**Expected outcome:** The ConfigMap is created before the Job, a successful Job is cleaned up, a failed Job remains for `kubectl logs`, and the upgrade fails when the Job fails.

<details>
<summary>Hints</summary>

`"helm.sh/hook": pre-upgrade`, `"helm.sh/hook-weight": "-5"` (weights are strings, sorted ascending), `"helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded`. Set `backoffLimit` and `activeDeadlineSeconds` so a broken Job does not retry forever. Non-hook ConfigMaps still hold the old content during `pre-upgrade`.

</details>

**Bonus:** Explain what `helm uninstall` does with hook resources and clean them up.

### A16. `helm test` that actually tests something 🟡

**Objective:** Add post-deploy verification that runs in the cluster.

**Tasks**
1. Write two test Pods under `templates/tests/`: one checks the Service responds, one checks a dependency (database, cache) through the app.
2. Order them with hook weights.
3. Run `helm test` with `--logs` after an install.
4. Break the app (wrong port in the Service) and run the tests again.
5. Use `--filter` to run only one test.

**Expected outcome:** Both tests pass on a healthy release, the right one fails on a broken one, and the failure output tells you which hop is broken.

<details>
<summary>Hints</summary>

Annotate with `"helm.sh/hook": test`. Use `restartPolicy: Never` and an image with `curl` (`curlimages/curl`). `helm test <release> --logs --filter name=<pod-name>`.

</details>

**Bonus:** Wire `helm test` into a CI job so a failing test triggers `helm rollback`.

### A17. Safe CI deploys with diff and atomic upgrades 🟡

**Objective:** Build the deploy step a production pipeline uses.

**Tasks**
1. Write a script that runs `helm lint --strict`, `helm template | kubeconform`, then `helm diff upgrade` for a release.
2. Deploy with `helm upgrade --install --atomic --wait --timeout 5m --history-max 10`.
3. Deploy a broken image tag and confirm the automatic rollback.
4. Make the script exit non-zero on any failure and print `helm history` on failure.
5. Run it in GitHub Actions (or any CI) against a kind cluster.

**Expected outcome:** A green run for a good change, a red run with an automatic rollback for a bad one, and the diff visible in the job log before the upgrade happens.

<details>
<summary>Hints</summary>

`set -euo pipefail`. `helm diff upgrade <rel> <chart> -n <ns> -f values.yaml --detailed-exitcode` exits 2 when there are changes. Kind in CI: `helm/kind-action`. Do not use `--reuse-values` in CI; pass the full values every time.

</details>

**Bonus:** Post the diff as a pull-request comment.

### A18. Publish to an OCI registry 🟡

**Objective:** Distribute charts the modern way.

**Tasks**
1. Run a local `registry:2` (or use GHCR / ECR).
2. Package `hello` at two versions and push both.
3. Show the chart in the registry (tag list or console).
4. Install and upgrade from `oci://` with `--version`.
5. Use an OCI chart as a dependency in another chart's `Chart.yaml`.

**Expected outcome:** Both versions listed in the registry, a release installed from OCI and upgraded between them, and a parent chart resolving an OCI dependency.

<details>
<summary>Hints</summary>

`helm push hello-0.1.0.tgz oci://localhost:5001/charts --plain-http`. For a dependency: `repository: "oci://localhost:5001/charts"` (no chart name in the URL). There is no `helm repo add` for OCI; log in with `helm registry login` for private registries.

</details>

**Bonus:** Sign the pushed artifact with cosign and verify it before install.

### A19. One chart, three environments 🟡

**Objective:** Structure environment configuration so differences are obvious in review.

**Tasks**
1. Create `values-dev.yaml`, `values-staging.yaml` and `values-prod.yaml` outside the chart, containing only overrides.
2. Enable HPA and PDB only in prod; disable `replicas` in the Deployment when HPA is on.
3. Render all three and produce a diff between staging and prod.
4. Install dev and staging as separate releases in separate namespaces.
5. Add a values-file layering example: `values-common.yaml` + `values-prod.yaml`.

**Expected outcome:** Each environment file is short, `helm template` for prod contains HPA and PDB while dev does not, and two releases of the same chart run side by side.

<details>
<summary>Hints</summary>

`{{- if not .Values.autoscaling.enabled }} replicas: ... {{- end }}`. Diff rendered output with `diff <(helm template ... -f staging) <(helm template ... -f prod)`. Lists replace rather than merge across values files, so prefer maps for things environments extend.

</details>

**Bonus:** Put the three releases in a `helmfile.yaml` with environments and run `helmfile diff`.

### A20. Config rollouts with checksums and `tpl` 🟡

**Objective:** Make config changes roll pods, and render templated values.

**Tasks**
1. Add a ConfigMap with app config generated from values.
2. Add a `checksum/config` annotation to the pod template so a config change rolls the Deployment.
3. Add a value `extraConfig` that contains template expressions (for example `{{ .Release.Name }}`) and render it with `tpl`.
4. Change only the config and upgrade; confirm pods restarted.
5. Upgrade with no changes and confirm pods did not restart.

**Expected outcome:** Config changes trigger a rollout, no-op upgrades do not, and `tpl` renders release-specific strings from values.

<details>
<summary>Hints</summary>

`checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}`. `{{ tpl .Values.extraConfig . }}`; `tpl` needs the context as its second argument. Watch with `kubectl rollout history`.

</details>

**Bonus:** Explain why `randAlphaNum` or `now` in a template makes every upgrade roll the pods, and how `lookup` can keep a generated password stable.

---

## Advanced

### A21. A library chart shared by two services 🔴

**Objective:** Remove copy-paste between application charts.

**Tasks**
1. Create `common-lib` with `type: library` containing helpers for names, labels, a Deployment and a Service.
2. Make the Deployment template accept overrides through a values contract (image, port, probes, resources, env).
3. Build two application charts (`orders`, `payments`) whose templates are one-line `include` calls.
4. Version the library, push it to OCI and consume it as a dependency.
5. Change a label in the library, bump its version, and roll it out to both charts.

**Expected outcome:** Two application charts with almost no templates of their own, rendering correct and distinct resources, and a library change propagated by bumping one dependency version.

<details>
<summary>Hints</summary>

A library chart cannot be installed and its templates are never rendered on their own. Define templates like `{{- define "common-lib.deployment" -}}` and call them as `{{ include "common-lib.deployment" . }}`. A common trick is a `merge`-based override pattern: the app chart passes its own partial template that the library merges into the base.

</details>

**Bonus:** Add unit tests for the library with the helm-unittest plugin.

### A22. Recover stuck and orphaned releases 🔴

**Objective:** Fix the two classic production Helm incidents without losing history.

**Tasks**
1. Start an upgrade with a long `--timeout` and kill the Helm process with `kill -9` to leave the release in `pending-upgrade`.
2. Reproduce `another operation (install/upgrade/rollback) is in progress`.
3. Recover with `helm rollback`, and separately (in another attempt) by removing only the pending revision Secret.
4. Create a Service with `kubectl`, then try to install a chart that renders the same Service.
5. Adopt the existing Service into the release using the ownership label and annotations.

**Expected outcome:** A written runbook for both incidents with the exact commands, and a release history that remains intact after recovery.

<details>
<summary>Hints</summary>

`kubectl get secret -l owner=helm,status=pending-upgrade -n <ns>`. Adoption needs `app.kubernetes.io/managed-by=Helm`, `meta.helm.sh/release-name=<release>` and `meta.helm.sh/release-namespace=<ns>`. `kill -9` is what leaves the release pending; a graceful Ctrl-C may let Helm record the failure.

</details>

**Bonus:** Write a CI step that detects a pending release before deploying and fails with a clear message instead of hanging.

### A23. Helmfile plus helm-secrets for a whole platform 🔴

**Objective:** Manage many releases and environments declaratively with secrets in Git.

**Tasks**
1. Write a `helmfile.yaml` with three releases (ingress-nginx, your app, a database) and `dev` and `prod` environments.
2. Use `needs` to order the releases.
3. Encrypt prod secrets with SOPS and age, and reference them from Helmfile via helm-secrets.
4. Run `helmfile diff` and `helmfile apply` for dev, then prod.
5. Show that the decrypted values never land on disk in plain text in the repo and are not visible in `git diff`.

**Expected outcome:** One command converges each environment, secrets are encrypted at rest in Git, and `helmfile diff` shows only real changes.

<details>
<summary>Hints</summary>

`.sops.yaml` with `creation_rules` and an age recipient; `sops -e -i secrets.prod.yaml`. In Helmfile, reference encrypted files under `secrets:` for a release. Install the plugin with `helm plugin install https://github.com/jkroepke/helm-secrets`.

</details>

**Bonus:** Replace age with AWS KMS and restrict decrypt permissions to the CI role.

### A24. Your chart under Argo CD 🔴

**Objective:** Understand what changes when Helm is only the renderer.

**Tasks**
1. Deploy your chart (from Git or OCI) with an Argo CD `Application` using `helm.valueFiles` and `helm.parameters`.
2. Confirm the release is missing from `helm list` and explain why.
3. Observe how your `pre-upgrade` and `post-install` hooks are mapped to Argo CD phases, and when each runs.
4. Find one template that behaves differently under Argo CD (`lookup`, `randAlphaNum`, `.Release.IsUpgrade`) and fix it.
5. Convert the migration hook to native Argo CD hooks and sync waves.

**Expected outcome:** The chart syncs cleanly without permanent `OutOfSync` drift, hooks run at the intended moment, and you can explain each behavioral difference.

<details>
<summary>Hints</summary>

Argo CD runs the equivalent of `helm template` and applies the result itself. `pre-install`/`pre-upgrade` map to `PreSync`, `post-install`/`post-upgrade` to `PostSync`, and they run on every sync. `lookup` returns empty during rendering, and random functions produce a new value every render, which causes perpetual diffs.

</details>

**Bonus:** Use an ApplicationSet to deploy the chart to dev and prod with different value files.

### A25. Survive a Kubernetes API removal 🔴

**Objective:** Upgrade a release whose stored manifest uses APIs the cluster no longer serves.

**Tasks**
1. Explain why a release that was installed with a removed API version (for example `policy/v1beta1` PodDisruptionBudget) cannot be upgraded even after the chart is fixed.
2. On an older kind cluster, install a chart using a deprecated API, then upgrade the cluster to a version that removed it (or simulate by editing a stored release in a test namespace).
3. Use the mapkubeapis plugin to rewrite the stored release manifest.
4. Upgrade the release with the fixed chart.
5. Add a CI check that fails when a rendered chart uses deprecated or removed APIs.

**Expected outcome:** The release upgrades successfully after the mapping, history is preserved, and CI catches the next deprecated API before it reaches a cluster.

<details>
<summary>Hints</summary>

Helm diffs the new manifest against the one stored in the last release Secret; if that stored manifest references a kind the API server no longer knows, the upgrade fails with `unable to build kubernetes objects from current release manifest ... no matches for kind`. `helm plugin install https://github.com/helm/helm-mapkubeapis`, then `helm mapkubeapis <release> -n <ns> --dry-run` first. For CI, `pluto detect-helm` or `kubeconform` with a target `-kubernetes-version`.

</details>

**Bonus:** Use `.Capabilities.APIVersions.Has` to make the chart support both old and new API versions during a migration window.

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
