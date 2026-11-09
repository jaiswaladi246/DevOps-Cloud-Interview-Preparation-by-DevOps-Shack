# Azure for DevOps — Assignments

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

> 25 hands-on assignments in three levels: 10 🟢 Beginner, 10 🟡 Intermediate, 5 🔴 Advanced. Do them in a sandbox subscription, tag everything, and delete what you create.

[⬅ Back to Azure for DevOps](./README.md) · [🏠 Home](../README.md)

## How to use these

Use the Azure CLI (and Terraform or Bicep where asked) for every task, even where the portal is quicker. The CLI is what you will script and put in pipelines. Before you start, set a budget on the subscription. Work in one region (`centralindia` in our examples) and tag every resource group with `project`, `env` and `owner`, so cleanup is `az group delete` on a known list. Hints are collapsed so you can try first.

## Table of contents

- 🟢 [Beginner](#-beginner) — Assignments 1–10
- 🟡 [Intermediate](#-intermediate) — Assignments 11–20
- 🔴 [Advanced](#-advanced) — Assignments 21–25

---

## 🟢 Beginner

### Assignment 1. Set up a safe working subscription 🟢

**Objective:** Get a subscription ready for practice with cost guardrails and correct CLI context.

**Tasks**
1. Log in with `az login`, list subscriptions and set the one you will use as default.
2. Create a monthly budget of 20 USD on the subscription that emails you at 50%, 80% and 100% of forecast.
3. Create a resource group `rg-practice-cin` in `centralindia` with tags `project=practice env=lab owner=<you>`.
4. Print your own object ID and confirm your role at subscription scope.

**Expected outcome / acceptance criteria:** `az account show` shows the right subscription; the budget is visible in Cost Management; `az group show -n rg-practice-cin --query tags` returns all three tags; `az role assignment list --assignee <your-oid> --include-inherited` shows Owner or Contributor.

<details>
<summary>Hints</summary>

`az account set --subscription`, `az consumption budget create --category cost --time-grain Monthly`, `az group create --tags k=v k2=v2`, `az ad signed-in-user show --query id -o tsv`. Notifications on budgets are easier in the portal the first time; then export the JSON and recreate it with the CLI.

</details>

**Bonus:** Create a subscription diagnostic setting that sends the Activity Log to a Log Analytics workspace.

### Assignment 2. Explore the resource hierarchy 🟢

**Objective:** Understand tenants, management groups, subscriptions and resource groups from the CLI.

**Tasks**
1. Show your tenant ID and the tenants you can access.
2. List management groups you can see (it may be just the root).
3. List all resource groups in the subscription with their locations and tags in a table.
4. Use Azure Resource Graph to count resources by type across the subscription.

**Expected outcome / acceptance criteria:** A short note in your own words explaining tenant → management group → subscription → resource group, with the actual IDs from your environment.

<details>
<summary>Hints</summary>

`az account tenant list`, `az account management-group list`, `az group list --query "[].{name:name, loc:location, tags:tags}" -o table`, `az graph query -q "Resources | summarize count() by type | order by count_ desc"` (needs `az extension add -n resource-graph`).

</details>

**Bonus:** Create a management group `mg-practice` and move your subscription under it (only in a personal tenant).

### Assignment 3. Build a VNet with NSGs 🟢

**Objective:** Create a VNet with two subnets and control traffic between them with NSGs.

**Tasks**
1. Create `vnet-practice` with address space `10.50.0.0/16` and subnets `snet-web` (`10.50.1.0/24`) and `snet-app` (`10.50.2.0/24`).
2. Create NSG `nsg-web` allowing inbound 80 and 443 from `Internet`, and NSG `nsg-app` allowing inbound 8080 only from `10.50.1.0/24`.
3. Associate each NSG with its subnet.
4. List the rules of both NSGs including default rules.

**Expected outcome / acceptance criteria:** `az network vnet subnet list` shows both subnets with their NSGs; `nsg-app` has no rule that allows traffic from `Internet`.

<details>
<summary>Hints</summary>

`az network vnet create --subnet-name --subnet-prefixes`, `az network vnet subnet create`, `az network nsg rule create --priority 100 --source-address-prefixes`, `az network vnet subnet update --network-security-group`, `az network nsg rule list --include-default -o table`.

</details>

**Bonus:** Replace the source CIDR in `nsg-app` with an Application Security Group `asg-web` attached to the web VM NICs.

### Assignment 4. Launch a Linux VM with no public IP and connect safely 🟢

**Objective:** Run a VM you can manage without exposing SSH to the internet.

**Tasks**
1. Create an Ubuntu VM in `snet-app` with `--public-ip-address ""` and a system-assigned managed identity.
2. Install nginx with cloud-init (`--custom-data`).
3. Run a command on it with `az vm run-command invoke` to check nginx is active.
4. Enable auto-shutdown at 20:00 local time.

**Expected outcome / acceptance criteria:** The VM has no public IP; `run-command` returns `active`; `az vm show -d --query publicIps` is empty; the auto-shutdown schedule exists.

<details>
<summary>Hints</summary>

`az vm create --image Ubuntu2204 --size Standard_B2s --assign-identity --custom-data cloud-init.yaml --nsg ""`. `az vm run-command invoke --command-id RunShellScript --scripts "systemctl is-active nginx"`. `az vm auto-shutdown --time 1430` uses UTC.

</details>

**Bonus:** Deploy Azure Bastion (Developer SKU if available in your region) and connect with `az network bastion ssh`.

### Assignment 5. Storage account done right 🟢

**Objective:** Create a locked-down storage account and access it with Entra ID, not keys.

**Tasks**
1. Create a StorageV2 account with ZRS, TLS 1.2 minimum, blob public access disabled.
2. Give yourself `Storage Blob Data Contributor` on the account.
3. Create a container `reports` and upload a file using `--auth-mode login`.
4. Generate a user delegation SAS for that blob valid for 1 hour, and download it with `curl`.

**Expected outcome / acceptance criteria:** Upload works with `--auth-mode login`; the SAS URL downloads the file; after the expiry, the same URL returns `AuthenticationFailed`.

<details>
<summary>Hints</summary>

`az storage account create --sku Standard_ZRS --min-tls-version TLS1_2 --allow-blob-public-access false`. `az storage blob generate-sas --as-user --auth-mode login --permissions r --expiry <UTC> --full-uri`.

</details>

**Bonus:** Add a lifecycle rule that moves blobs under `reports/` to Cool after 30 days and deletes them after 365.

### Assignment 6. Key Vault with RBAC 🟢

**Objective:** Store and read secrets with Azure RBAC data-plane roles.

**Tasks**
1. Create a Key Vault with RBAC authorization enabled and soft-delete retention of 7 days.
2. Try `az keyvault secret set` before giving yourself any data role, and record the error.
3. Assign yourself `Key Vault Secrets Officer` and set a secret `api-key`.
4. Give the VM from Assignment 4 `Key Vault Secrets User`, then read the secret from inside the VM with its managed identity.

**Expected outcome / acceptance criteria:** Step 2 fails with `ForbiddenByRbac`; step 4 prints the secret from the VM via `az login --identity` or a curl to IMDS plus the Key Vault REST API.

<details>
<summary>Hints</summary>

`az keyvault create --enable-rbac-authorization true --retention-days 7`. On the VM: get a token from `http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://vault.azure.net` with header `Metadata:true`, then `GET https://<vault>.vault.azure.net/secrets/api-key?api-version=7.4`.

</details>

**Bonus:** Delete the vault, show it in `az keyvault list-deleted`, then recover it.

### Assignment 7. Push your first image to ACR 🟢

**Objective:** Build and store container images in Azure Container Registry.

**Tasks**
1. Create an ACR (Basic SKU) with the admin user disabled.
2. Write a tiny Dockerfile (any web server) and build it in the cloud with `az acr build`.
3. Import `nginx:1.27` from Docker Hub into `base/nginx:1.27`.
4. List repositories and tags.

**Expected outcome / acceptance criteria:** Two repositories exist, with tags visible in `az acr repository show-tags`; `az acr show --query adminUserEnabled` returns `false`.

<details>
<summary>Hints</summary>

`az acr create --sku Basic --admin-enabled false`, `az acr build -r <acr> -t hello:v1 .`, `az acr import --source docker.io/library/nginx:1.27 --image base/nginx:1.27`.

</details>

**Bonus:** Run the image as an Azure Container Instance pulling from ACR with a user-assigned identity.

### Assignment 8. Deploy a web app to App Service with slots 🟢

**Objective:** Use deployment slots for zero-downtime releases.

**Tasks**
1. Create a Linux App Service plan (S1) and a web app running a container image or a sample code app.
2. Create a `staging` slot and deploy a visibly different version to it.
3. Add an app setting `ENV_NAME` as a slot setting with different values per slot.
4. Swap staging into production and confirm the sticky setting stayed with its slot.

**Expected outcome / acceptance criteria:** After the swap, production serves the new version and still reports `ENV_NAME=production`.

<details>
<summary>Hints</summary>

`az appservice plan create --is-linux --sku S1`, `az webapp create --deployment-container-image-name`, `az webapp deployment slot create --slot staging`, `az webapp config appsettings set --slot-settings ENV_NAME=staging --slot staging`, `az webapp deployment slot swap`.

</details>

**Bonus:** Turn on the `@Microsoft.KeyVault(...)` reference for one setting using the app's managed identity.

### Assignment 9. Your first AKS cluster 🟢

**Objective:** Create a small AKS cluster and run a workload pulled from your ACR.

**Tasks**
1. Create a 2-node AKS cluster with managed identity and Azure CNI Overlay.
2. Attach the ACR from Assignment 7 to the cluster.
3. Get credentials, deploy your `hello:v1` image with 2 replicas and a LoadBalancer Service.
4. Run `az aks check-acr` and record the output.

**Expected outcome / acceptance criteria:** `curl` to the Service's external IP returns your app; `kubectl get nodes` shows 2 Ready nodes; `check-acr` reports success.

<details>
<summary>Hints</summary>

`az aks create --node-count 2 --network-plugin azure --network-plugin-mode overlay --attach-acr <acr> --generate-ssh-keys`. `kubectl create deployment hello --image=<acr>.azurecr.io/hello:v1 --replicas=2` then `kubectl expose deployment hello --type=LoadBalancer --port=80 --target-port=<port>`.

</details>

**Bonus:** Stop the cluster with `az aks stop`, then start it again and confirm the app is back.

### Assignment 10. Monitor a resource and alert on it 🟢

**Objective:** Use Azure Monitor metrics, logs and alerts.

**Tasks**
1. Create a Log Analytics workspace.
2. Send the Key Vault from Assignment 6 audit logs to it with a diagnostic setting.
3. Read the secret a few times, then query `AzureDiagnostics` (or `AZKVAuditLogs`) for `SecretGet` operations.
4. Create an action group with your email and a metric alert on the VM `Percentage CPU > 80`.

**Expected outcome / acceptance criteria:** The KQL query returns your secret reads with caller identity; the alert rule shows up in `az monitor metrics alert list`.

<details>
<summary>Hints</summary>

`az monitor diagnostic-settings create --logs '[{"categoryGroup":"audit","enabled":true}]'`. Query: `AzureDiagnostics | where ResourceProvider == "MICROSOFT.KEYVAULT" and OperationName == "SecretGet" | project TimeGenerated, CallerIPAddress, identity_claim_upn_s`.

</details>

**Bonus:** Use `stress-ng` on the VM through `run-command` to make the alert fire.

---

## 🟡 Intermediate

### Assignment 11. Least-privilege custom role 🟡

**Objective:** Write and test a custom role for a narrow job.

**Tasks**
1. Write a custom role "VM Operator" that can read, start, restart and deallocate VMs but not create or delete them.
2. Create it with assignable scope set to your subscription.
3. Assign it to a test user or a service principal on `rg-practice-cin`.
4. Prove it works: start/stop succeeds; `az vm delete` fails with `AuthorizationFailed`.

**Expected outcome / acceptance criteria:** Role JSON in your repo; screenshots or command outputs showing allowed and denied actions.

<details>
<summary>Hints</summary>

Actions: `Microsoft.Compute/virtualMachines/read`, `/start/action`, `/restart/action`, `/deallocate/action`, `/powerOff/action`, plus `Microsoft.Resources/subscriptions/resourceGroups/read`. Find names with `az provider operation show -n Microsoft.Compute`.

</details>

**Bonus:** Add an ABAC-style restriction by giving a second identity `Role Based Access Control Administrator` that can only assign your custom role.

### Assignment 12. Hub-and-spoke with forced egress 🟡

**Objective:** Build a hub-spoke network where spoke traffic to the internet and to other spokes goes through a central appliance.

**Tasks**
1. Create `vnet-hub` (`10.60.0.0/16`) and two spokes (`10.61.0.0/16`, `10.62.0.0/16`), peered hub↔spoke both ways.
2. Deploy Azure Firewall Basic (or a Linux VM with IP forwarding as a cheap NVA) in the hub.
3. Add UDRs on spoke subnets: `0.0.0.0/0` and the other spoke CIDR → firewall private IP.
4. Allow only spoke1 → spoke2 on TCP 22 and outbound HTTPS to `*.ubuntu.com`.

**Expected outcome / acceptance criteria:** A VM in spoke1 can SSH to spoke2 and run `apt update`, but `curl https://example.com` fails; `az network watcher show-next-hop` shows `VirtualAppliance`.

<details>
<summary>Hints</summary>

Peering flag `--allow-forwarded-traffic` on hub-to-spoke peerings. For an NVA VM: `az network nic update --ip-forwarding true` plus `sysctl net.ipv4.ip_forward=1` and iptables MASQUERADE. Firewall rules: application rule collection for FQDNs, network rule collection for 22.

</details>

**Bonus:** Turn on VNet flow logs and find the denied flows in Traffic Analytics.

### Assignment 13. Private Endpoint and private DNS 🟡

**Objective:** Make a storage account or Key Vault reachable only privately, and get DNS right.

**Tasks**
1. Create a Private Endpoint for your Key Vault in a spoke subnet with a DNS zone group for `privatelink.vaultcore.azure.net`.
2. Link the private DNS zone to the VNet.
3. Disable public network access on the vault.
4. From a VM in the VNet, show `nslookup <vault>.vault.azure.net` returns a `10.x` IP and the secret read works; from your laptop show it fails.

**Expected outcome / acceptance criteria:** Private resolution inside, `ForbiddenByFirewall` or public-access-disabled error outside.

<details>
<summary>Hints</summary>

`az network private-endpoint create --group-id vault --private-connection-resource-id <kvId>`, `az network private-dns zone create`, `az network private-dns link vnet create`, `az network private-endpoint dns-zone-group create`, `az keyvault update --public-network-access Disabled`.

</details>

**Bonus:** Link the zone only to the hub and make the spoke resolve it through a DNS Private Resolver inbound endpoint.

### Assignment 14. Terraform an environment with remote state 🟡

**Objective:** Build an RG, VNet, ACR and Key Vault with Terraform azurerm v4 and a secure backend.

**Tasks**
1. Create a state storage account with shared keys disabled and blob versioning on.
2. Write Terraform with `backend "azurerm"` using `use_azuread_auth = true`.
3. Create the resources with tags, a random suffix for globally unique names and outputs.
4. Change a tag in the portal, then show the drift with `terraform plan -detailed-exitcode`.

**Expected outcome / acceptance criteria:** `plan` exit code 0 after apply, 2 after the manual change; state blob has multiple versions.

<details>
<summary>Hints</summary>

Remember `subscription_id` in the provider block for v4. Give yourself `Storage Blob Data Contributor` on the state account. `az storage blob list --include v` shows versions.

</details>

**Bonus:** Import a resource group you created by hand using an `import` block and `-generate-config-out`.

### Assignment 15. The same stack in Bicep 🟡

**Objective:** Learn Bicep modules, parameters files and what-if.

**Tasks**
1. Write `main.bicep` with modules `network.bicep`, `acr.bicep`, `keyvault.bicep`.
2. Use a `.bicepparam` file for `dev` and one for `prod` with different SKUs.
3. Run `what-if`, then deploy `dev`.
4. Change a property and show the what-if diff before redeploying.

**Expected outcome / acceptance criteria:** Deployment succeeds; what-if output shows only the intended modification on the second run.

<details>
<summary>Hints</summary>

`az deployment group what-if -g rg -f main.bicep -p dev.bicepparam`. Use `uniqueString(resourceGroup().id)` for names. `az bicep lint -f main.bicep` catches common mistakes.

</details>

**Bonus:** Deploy it as a deployment stack with `--deny-settings-mode denyDelete` and try deleting a resource from the portal.

### Assignment 16. GitHub Actions to Azure with OIDC 🟡

**Objective:** Deploy from GitHub without any stored Azure secret.

**Tasks**
1. Create a user-assigned identity and a federated credential for `repo:<you>/<repo>:environment:dev`.
2. Give it Contributor on one resource group only.
3. Write a workflow with `id-token: write` that logs in with `azure/login@v2` and runs `az group show`.
4. Run it from a feature branch without the environment and capture the AADSTS error, then fix it.

**Expected outcome / acceptance criteria:** The `dev` environment job succeeds; the branch job fails with `AADSTS70021` showing the subject it presented.

<details>
<summary>Hints</summary>

`az identity federated-credential create --issuer https://token.actions.githubusercontent.com --subject ... --audiences api://AzureADTokenExchange`. Store client, tenant and subscription IDs as GitHub variables; they are not secrets.

</details>

**Bonus:** Add a Reader-only identity federated for `pull_request` and run `terraform plan` on PRs with it.

### Assignment 17. Azure DevOps YAML pipeline with environments 🟡

**Objective:** Build a multi-stage Azure Pipelines deployment with approvals and a WIF service connection.

**Tasks**
1. Create an Azure DevOps project and an ARM service connection using workload identity federation.
2. Write `azure-pipelines.yml` with `Build` and `Deploy` stages; Build produces an artifact, Deploy uses a `deployment` job.
3. Create environment `practice-prod` with an approval check and branch control for `main`.
4. Run from a non-main branch and show the deploy is blocked by the check.

**Expected outcome / acceptance criteria:** Pipeline run history shows the approval gate; the environment's deployments tab lists the run.

<details>
<summary>Hints</summary>

Use `AzureCLI@2` with `azureSubscription:`. Publish artifacts with `- publish: $(Build.ArtifactStagingDirectory)` and consume with `- download: current`. Branch control lives under the environment's Approvals and checks.

</details>

**Bonus:** Move the stages into a template repo and use `extends` with the "Required template" check.

### Assignment 18. Workload identity + Key Vault CSI on AKS 🟡

**Objective:** Mount a Key Vault secret into a pod with no secret in Kubernetes manifests.

**Tasks**
1. Enable `--enable-oidc-issuer --enable-workload-identity` and the `azure-keyvault-secrets-provider` add-on with rotation.
2. Create a user-assigned identity, grant it `Key Vault Secrets User`, federate it to `system:serviceaccount:demo:app-sa`.
3. Deploy a ServiceAccount, SecretProviderClass and a Deployment that mounts the secret at `/mnt/secrets`.
4. Rotate the secret and show the file updates without a pod restart.

**Expected outcome / acceptance criteria:** `kubectl exec ... cat /mnt/secrets/<name>` shows the new value within about 2 minutes of rotation; pod restart count is 0.

<details>
<summary>Hints</summary>

Pod label `azure.workload.identity/use: "true"`, SA annotation `azure.workload.identity/client-id`. SecretProviderClass needs `clientID`, `keyvaultName`, `tenantId`, `objects`. Rotation: `az aks addon update -a azure-keyvault-secrets-provider --enable-secret-rotation`.

</details>

**Bonus:** Add `secretObjects` to sync into a Kubernetes Secret and use it as an env var; explain why the env var doesn't rotate.

### Assignment 19. AKS node pools, taints and the autoscaler 🟡

**Objective:** Separate system and user workloads and watch the autoscaler work.

**Tasks**
1. Add a user node pool `apps` with autoscaler 1–4 across zones.
2. Taint the system pool with `CriticalAddonsOnly=true:NoSchedule` (add a new tainted system pool if needed) and confirm app pods land on `apps`.
3. Scale a deployment so pods go Pending, and record the autoscaler events.
4. Scale back and time how long scale-down takes; then change `scale-down-unneeded-time` and repeat.

**Expected outcome / acceptance criteria:** Events show `TriggeredScaleUp`; node count goes up and back down; you can explain the timing.

<details>
<summary>Hints</summary>

`az aks nodepool add --mode User --enable-cluster-autoscaler --min-count 1 --max-count 4 --zones 1 2 3`. `kubectl get events -A --field-selector reason=TriggeredScaleUp`. `az aks update --cluster-autoscaler-profile scale-down-unneeded-time=3m`.

</details>

**Bonus:** Add a Spot pool and schedule a CronJob onto it with the Spot toleration.

### Assignment 20. Azure Policy guardrails 🟡

**Objective:** Enforce governance with built-in and custom policies.

**Tasks**
1. Assign "Allowed locations" for `centralindia` and `southindia` on your practice RG.
2. Assign "Require a tag on resource groups" for `owner` at subscription scope in Audit mode, then check compliance.
3. Write a custom policy that denies storage accounts with `allowBlobPublicAccess` set to true.
4. Show a denied deployment and the `RequestDisallowedByPolicy` error, then create an exemption for one RG.

**Expected outcome / acceptance criteria:** Compliance report lists non-compliant RGs; the custom deny works; the exempted RG allows the deployment.

<details>
<summary>Hints</summary>

Field alias: `Microsoft.Storage/storageAccounts/allowBlobPublicAccess`. `az policy definition create --rules @rule.json --mode Indexed`, `az policy assignment create`, `az policy state summarize`, `az policy exemption create --exemption-category Waiver`.

</details>

**Bonus:** Assign a DeployIfNotExists policy that adds diagnostic settings to Key Vaults and run a remediation task.

---

## 🔴 Advanced

### Assignment 21. End-to-end AKS platform in Terraform with a secretless pipeline 🔴

**Objective:** Reproduce the [hands-on lab](./hands-on-lab.md) as reusable modules with two environments.

**Tasks**
1. Split the lab into modules (`network`, `aks`, `acr`, `keyvault`, `monitoring`, `identity`) with input validation.
2. Create `envs/dev` and `envs/prod` roots with separate state keys and different sizes (`Free` vs `Standard` tier, node counts).
3. Build a pipeline: PR → fmt, validate, tflint, Checkov, plan posted as a comment; merge → apply dev automatically, prod after approval using the saved plan.
4. Use separate federated identities for dev and prod applies.

**Expected outcome / acceptance criteria:** Both environments deploy from the same modules; a PR shows the plan; prod apply waits for approval and applies exactly the reviewed plan file.

<details>
<summary>Hints</summary>

Upload `tfplan` as an artifact between jobs. `terraform show -no-color tfplan` for the PR comment. In GitHub, environments `infra-dev` and `infra-prod` with separate federated subjects. Pin module versions with git tags.

</details>

**Bonus:** Add a nightly drift detection workflow that opens an issue when `plan -detailed-exitcode` returns 2.

### Assignment 22. Private AKS with controlled egress 🔴

**Objective:** Run a private cluster whose only egress is through Azure Firewall.

**Tasks**
1. Build a hub with Azure Firewall (DNS proxy on) and a spoke with the AKS subnet and a route table `0.0.0.0/0` → firewall.
2. Allow the AKS required egress using the `AzureKubernetesService` FQDN tag plus your ACR and MCR FQDNs, NTP, and the API server.
3. Create a private AKS cluster with `--outbound-type userDefinedRouting`, Premium ACR with a Private Endpoint, and Key Vault with a Private Endpoint.
4. Deploy a workload using `az aks command invoke` (no direct API access) and show a pod cannot `curl https://example.com`.

**Expected outcome / acceptance criteria:** Cluster provisions successfully; firewall logs show allowed AKS dependencies and denied `example.com`; images pull through the private endpoint.

<details>
<summary>Hints</summary>

Create the route table and firewall rules before the cluster, or creation fails while nodes try to bootstrap. Private DNS: `privatelink.<region>.azmk8s.io` (system-managed is fine), `privatelink.azurecr.io`, `privatelink.vaultcore.azure.net`. Use `--enable-private-cluster --vnet-subnet-id`.

</details>

**Bonus:** Add an internal Application Gateway for Containers or app routing with an internal load balancer, and expose the app only to the hub.

### Assignment 23. Zero-downtime AKS upgrade under load 🔴

**Objective:** Upgrade an AKS cluster by one minor version while proving no user-visible errors.

**Tasks**
1. Deploy an app with 4 replicas, readiness probes, a PDB (`maxUnavailable: 1`) and topology spread across zones.
2. Run a load generator (k6, hey or a curl loop) recording status codes for the whole upgrade.
3. Upgrade the control plane, then the node pool with `--max-surge 33%`, `--node-soak-duration`, and an explicit drain timeout.
4. Repeat with a broken PDB (`maxUnavailable: 0`) and capture the failure and recovery.

**Expected outcome / acceptance criteria:** First upgrade: zero non-2xx responses; second: `UpgradeFailed` with the PDB message, fixed by relaxing the PDB and resuming.

<details>
<summary>Hints</summary>

`az aks get-upgrades`, `az aks upgrade --control-plane-only`, `az aks nodepool upgrade --max-surge 33% --drain-timeout 30 --node-soak-duration 2`. Make sure you have quota for surge nodes first.

</details>

**Bonus:** Configure auto-upgrade channels with a weekly maintenance window and show the schedule with `az aks maintenanceconfiguration list`.

### Assignment 24. Multi-region active-passive with Front Door 🔴

**Objective:** Design and test regional failover for a web app.

**Tasks**
1. Deploy the same app (App Service or AKS) in `centralindia` and `southindia` from one IaC codebase.
2. Put Front Door Standard in front with priority routing and health probes on `/healthz`.
3. Use geo-replicated data (Azure SQL failover group, or GZRS storage for a static workload).
4. Simulate a primary region outage (stop the app or make `/healthz` return 500), measure RTO, then fail back.

**Expected outcome / acceptance criteria:** A written runbook with measured RTO, the observed RPO for the data tier, and Front Door logs showing the origin switch.

<details>
<summary>Hints</summary>

`az afd profile create`, `az afd origin-group create --probe-path /healthz`, `az afd origin create --priority 1/2`. Lock origins to Front Door with the `AzureFrontDoor.Backend` service tag and an `X-Azure-FDID` header check.

</details>

**Bonus:** Make it active-active with weighted routing and explain what changes in the data tier.

### Assignment 25. FinOps and governance review of a messy subscription 🔴

**Objective:** Act as the platform engineer brought in to clean up an existing subscription.

**Tasks**
1. Using Resource Graph, produce an inventory: untagged resources, unattached disks, unused public IPs, stopped-but-allocated VMs, empty App Service plans.
2. Query Cost Management for the top 10 cost drivers for the last 30 days and the Log Analytics ingestion by table.
3. Implement fixes: tag inheritance policy and remediation, delete waste, a daily cap on non-prod workspaces, auto-shutdown on dev VMs.
4. Write a one-page report with before/after monthly cost and the policies added.

**Expected outcome / acceptance criteria:** Report with numbers, the KQL and CLI used, and a policy compliance screenshot showing improvement.

<details>
<summary>Hints</summary>

Unattached disks: `Resources | where type =~ 'microsoft.compute/disks' and properties.diskState == 'Unattached'`. Unused public IPs: `Resources | where type =~ 'microsoft.network/publicipaddresses' and isnull(properties.ipConfiguration)`. Log Analytics: `Usage | summarize sum(Quantity) by DataType`. `az costmanagement query` or exports for cost data.

</details>

**Bonus:** Set up a scheduled Cost Management export to storage and a budget per resource group with an action group that posts to a Teams or Slack webhook.

[⬅ Back to Azure for DevOps](./README.md) · [🏠 Home](../README.md)

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
