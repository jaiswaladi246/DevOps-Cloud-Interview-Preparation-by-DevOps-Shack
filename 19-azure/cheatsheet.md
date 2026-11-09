# Azure for DevOps — Cheatsheet

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

> Dense CLI, IaC and config reference for day-to-day Azure work and interviews. Azure CLI 2.6x syntax, Terraform azurerm v4, Bicep.

[⬅ Back to Azure for DevOps](./README.md) · [🏠 Home](../README.md)

## Table of contents

- [CLI setup, context and output](#cli-setup-context-and-output)
- [Resource groups, tags, locks and Resource Graph](#resource-groups-tags-locks-and-resource-graph)
- [Entra ID, RBAC and managed identities](#entra-id-rbac-and-managed-identities)
- [Networking](#networking)
- [VMs and VMSS](#vms-and-vmss)
- [AKS](#aks)
- [ACR](#acr)
- [App Service, Functions and Container Apps](#app-service-functions-and-container-apps)
- [Storage](#storage)
- [Key Vault](#key-vault)
- [Azure Monitor and KQL](#azure-monitor-and-kql)
- [Policy and cost](#policy-and-cost)
- [Bicep and ARM](#bicep-and-arm)
- [Terraform azurerm](#terraform-azurerm)
- [Pipelines: GitHub Actions and Azure DevOps](#pipelines-github-actions-and-azure-devops)
- [Common errors decoder](#common-errors-decoder)
- [AWS to Azure quick map](#aws-to-azure-quick-map)
- [Numbers worth remembering](#numbers-worth-remembering)
- [Top 15 interview one-liners](#top-15-interview-one-liners)

---

## CLI setup, context and output

| Task | Command |
|------|---------|
| Log in (browser / device code) | `az login` / `az login --use-device-code` |
| Log in as managed identity | `az login --identity` (user-assigned: add `--client-id <id>`) |
| Log in as service principal with OIDC token | `az login --service-principal -u <clientId> -t <tenantId> --federated-token "$TOKEN"` |
| List / set subscription | `az account list -o table` / `az account set --subscription <id>` |
| Who am I | `az account show --query "{sub:name, user:user.name, tenant:tenantId}" -o table` |
| My object ID | `az ad signed-in-user show --query id -o tsv` |
| Get an access token | `az account get-access-token --resource https://management.azure.com` |
| Set defaults | `az config set defaults.group=rg-shop defaults.location=centralindia` |
| Output formats | `-o json\|jsonc\|table\|tsv\|yaml\|none` |
| Filter (JMESPath) | `--query "[?location=='centralindia'].{n:name, id:id}"` |
| Call any ARM / Graph API | `az rest --method get --url <url>` |
| Extensions | `az extension add -n <name>` / `az extension list -o table` |
| Upgrade CLI | `az upgrade` |
| Debug a call | `az group list --debug 2>&1 \| grep -i "request url"` |

## Resource groups, tags, locks and Resource Graph

| Task | Command |
|------|---------|
| Create RG with tags | `az group create -n rg-shop-dev -l centralindia --tags env=dev owner=platform` |
| Delete RG (async) | `az group delete -n rg-shop-dev --yes --no-wait` |
| Merge tags | `az tag update --resource-id <id> --operation merge --tags costcenter=cc-104` |
| Resources by tag | `az resource list --tag env=dev -o table` |
| Lock / list locks | `az lock create -n no-delete -g rg --lock-type CanNotDelete` / `az lock list -o table` |
| Move resources | `az resource move --destination-group rg-new --ids <id1> <id2>` |
| Provider registration | `az provider register -n Microsoft.ContainerService --wait` |
| Activity Log, last day | `az monitor activity-log list -g rg --offset 1d -o table` |
| Resource Graph count by type | `az graph query -q "Resources \| summarize count() by type"` |
| Unattached disks | `az graph query -q "Resources \| where type =~ 'microsoft.compute/disks' and properties.diskState == 'Unattached' \| project name, resourceGroup"` |

## Entra ID, RBAC and managed identities

| Task | Command |
|------|---------|
| Assign role (user/group) | `az role assignment create --assignee <upn-or-objectId> --role Reader --scope <scope>` |
| Assign role (new identity, skip lookup) | `az role assignment create --assignee-object-id <oid> --assignee-principal-type ServicePrincipal --role AcrPull --scope <acrId>` |
| List assignments (all scopes, inherited, groups) | `az role assignment list --assignee <id> --all --include-inherited --include-groups -o table` |
| Who has Owner here | `az role assignment list --role Owner --scope <scope> --include-inherited -o table` |
| Show a role's actions | `az role definition list -n "AcrPush" --query "[].permissions"` |
| Create custom role | `az role definition create --role-definition @role.json` |
| Create user-assigned identity | `az identity create -g rg -n id-shop-web` |
| Identity IDs | `az identity show -g rg -n id-shop-web --query "{clientId:clientId, principalId:principalId}"` |
| Enable system identity on VM | `az vm identity assign -g rg -n vm-app` |
| Federated credential (GitHub) | `az identity federated-credential create -g rg --identity-name id-ci -n gh-prod --issuer https://token.actions.githubusercontent.com --subject repo:org/repo:environment:prod --audiences api://AzureADTokenExchange` |
| Federated credential (AKS pod) | `... --issuer <aks oidc url> --subject system:serviceaccount:<ns>:<sa>` |
| App registration + SP | `az ad app create --display-name app-ci` then `az ad sp create --id <appId>` |
| SP with secret (avoid) | `az ad sp create-for-rbac -n sp-legacy --role AcrPush --scopes <acrId>` |
| List / reset secret | `az ad app credential list --id <appId>` / `az ad app credential reset --id <appId> --append` |
| Group membership check | `az ad group member check --group <group> --member-id <oid>` |

Federated subjects GitHub sends: `repo:O/R:ref:refs/heads/main`, `repo:O/R:environment:prod`, `repo:O/R:pull_request`, `repo:O/R:ref:refs/tags/v1.0.0`. Azure DevOps: issuer `https://vstoken.dev.azure.com/<org-id>`, subject `sc://<org>/<project>/<service-connection>`.

**Key roles:** Owner, Contributor, Reader, User Access Administrator, Role Based Access Control Administrator, Network Contributor, AcrPull, AcrPush, Key Vault Secrets User / Officer, Storage Blob Data Reader / Contributor, Azure Kubernetes Service Cluster User Role, Azure Kubernetes Service RBAC Reader / Writer / Admin / Cluster Admin, Monitoring Contributor, Log Analytics Reader.

## Networking

| Task | Command |
|------|---------|
| VNet + subnet | `az network vnet create -g rg -n vnet-shop --address-prefixes 10.20.0.0/16 --subnet-name snet-app --subnet-prefixes 10.20.1.0/24` |
| Add subnet | `az network vnet subnet create -g rg --vnet-name vnet-shop -n snet-pe --address-prefixes 10.20.2.0/24` |
| NSG + rule | `az network nsg create -g rg -n nsg-app` then `az network nsg rule create -g rg --nsg-name nsg-app -n allow-8080 --priority 100 --direction Inbound --access Allow --protocol Tcp --source-address-prefixes 10.20.0.0/24 --destination-port-ranges 8080` |
| Attach NSG to subnet | `az network vnet subnet update -g rg --vnet-name vnet-shop -n snet-app --network-security-group nsg-app` |
| ASG | `az network asg create -g rg -n asg-web`; use `--source-asgs asg-web` in rules |
| Route table + UDR | `az network route-table create -g rg -n rt-spoke` then `az network route-table route create -g rg --route-table-name rt-spoke -n default --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.1.4` |
| Peering (run on both sides) | `az network vnet peering create -g rg -n a-to-b --vnet-name vnet-a --remote-vnet <vnetBId> --allow-vnet-access --allow-forwarded-traffic` |
| NAT Gateway | `az network nat gateway create -g rg -n nat-shop --public-ip-addresses pip-nat` then `az network vnet subnet update ... --nat-gateway nat-shop` |
| Private Endpoint | `az network private-endpoint create -g rg -n pe-kv --vnet-name vnet-shop --subnet snet-pe --private-connection-resource-id <kvId> --group-id vault --connection-name kv-conn` |
| Private DNS zone + link | `az network private-dns zone create -g rg -n privatelink.vaultcore.azure.net` / `az network private-dns link vnet create -g rg -z privatelink.vaultcore.azure.net -n link -v <vnetId> -e false` |
| Zone group on PE | `az network private-endpoint dns-zone-group create -g rg --endpoint-name pe-kv -n default --private-dns-zone privatelink.vaultcore.azure.net --zone-name kv` |
| Effective NSG / routes | `az network nic list-effective-nsg -g rg -n <nic>` / `az network nic show-effective-route-table -g rg -n <nic> -o table` |
| IP flow verify | `az network watcher test-ip-flow -g rg --vm vm --direction Inbound --protocol TCP --local 10.20.1.4:443 --remote 1.2.3.4:50000` |
| Next hop | `az network watcher show-next-hop -g rg --vm vm --source-ip 10.20.1.4 --dest-ip 8.8.8.8` |
| App Gateway backend health | `az network application-gateway show-backend-health -g rg -n agw` |

**Private DNS zone names:** Key Vault `privatelink.vaultcore.azure.net` · Blob `privatelink.blob.core.windows.net` · ACR `privatelink.azurecr.io` · Azure SQL `privatelink.database.windows.net` · PostgreSQL Flexible `privatelink.postgres.database.azure.com` · AKS API `privatelink.<region>.azmk8s.io` · App Service `privatelink.azurewebsites.net`.

**Special subnets:** `GatewaySubnet` (/27+), `AzureFirewallSubnet` (/26), `AzureFirewallManagementSubnet` (/26), `AzureBastionSubnet` (/26+), App Gateway v2 dedicated (/24 recommended), Container Apps workload profiles (/27+, delegated).

## VMs and VMSS

| Task | Command |
|------|---------|
| Create VM, no public IP | `az vm create -g rg -n vm-app --image Ubuntu2204 --size Standard_B2s --vnet-name vnet-shop --subnet snet-app --public-ip-address "" --nsg "" --assign-identity --custom-data cloud-init.yaml --generate-ssh-keys` |
| Sizes and availability | `az vm list-skus -l centralindia --size Standard_D4 --resource-type virtualMachines -o table` |
| Quota usage | `az vm list-usage -l centralindia -o table` |
| Run a command inside | `az vm run-command invoke -g rg -n vm-app --command-id RunShellScript --scripts "uptime"` |
| Deallocate (stop billing compute) | `az vm deallocate -g rg -n vm-app` |
| Resize | `az vm resize -g rg -n vm-app --size Standard_D4s_v5` |
| Boot log | `az vm boot-diagnostics get-boot-log -g rg -n vm-app` |
| Auto-shutdown (UTC) | `az vm auto-shutdown -g rg -n vm-app --time 1430` |
| VMSS create (Flexible) | `az vmss create -g rg -n vmss-web --image Ubuntu2204 --orchestration-mode Flexible --instance-count 2 --zones 1 2 3` |
| Scale VMSS | `az vmss scale -g rg -n vmss-web --new-capacity 4` |
| Bastion SSH with Entra | `az network bastion ssh -n bas -g rg-hub --target-resource-id <vmId> --auth-type AAD` |

## AKS

| Task | Command |
|------|---------|
| Create (production-style) | `az aks create -g rg -n aks --tier standard --zones 1 2 3 --node-count 3 --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium --enable-aad --enable-azure-rbac --disable-local-accounts --enable-oidc-issuer --enable-workload-identity --enable-addons azure-keyvault-secrets-provider,monitoring --attach-acr acrshop --os-sku AzureLinux --generate-ssh-keys` |
| Credentials (Entra) | `az aks get-credentials -g rg -n aks --overwrite-existing && kubelogin convert-kubeconfig -l azurecli` |
| Install kubectl + kubelogin | `az aks install-cli` |
| Show key facts | `az aks show -g rg -n aks --query "{v:kubernetesVersion, state:provisioningState, power:powerState.code, oidc:oidcIssuerProfile.issuerUrl}"` |
| Kubelet / cluster identity | `az aks show -g rg -n aks --query "{kubelet:identityProfile.kubeletidentity.objectId, cluster:identity.principalId}"` |
| Add user pool with autoscaler | `az aks nodepool add -g rg --cluster-name aks -n apps --mode User --enable-cluster-autoscaler --min-count 2 --max-count 10 --zones 1 2 3` |
| Add Spot pool | `az aks nodepool add -g rg --cluster-name aks -n spot --priority Spot --eviction-policy Delete --spot-max-price -1 --enable-cluster-autoscaler --min-count 0 --max-count 5` |
| Scale pool | `az aks nodepool scale -g rg --cluster-name aks -n apps --node-count 5` |
| Available upgrades | `az aks get-upgrades -g rg -n aks -o table` |
| Upgrade control plane, then pool | `az aks upgrade -g rg -n aks -k 1.33.2 --control-plane-only` / `az aks nodepool upgrade -g rg --cluster-name aks -n apps -k 1.33.2 --max-surge 33%` |
| Node image only | `az aks nodepool upgrade -g rg --cluster-name aks -n apps --node-image-only` |
| Auto-upgrade channels | `az aks update -g rg -n aks --auto-upgrade-channel patch --node-os-upgrade-channel NodeImage` |
| Attach ACR / check | `az aks update -g rg -n aks --attach-acr acrshop` / `az aks check-acr -g rg -n aks --acr acrshop.azurecr.io` |
| Authorized IPs | `az aks update -g rg -n aks --api-server-authorized-ip-ranges 203.0.113.0/24` |
| Run kubectl on private cluster | `az aks command invoke -g rg -n aks --command "kubectl get pods -A"` |
| Stop / start | `az aks stop -g rg -n aks` / `az aks start -g rg -n aks` |
| Autoscaler profile | `az aks update -g rg -n aks --cluster-autoscaler-profile scale-down-unneeded-time=5m expander=least-waste` |
| Enable KEDA / app routing | `az aks update -g rg -n aks --enable-keda` / `az aks approuting enable -g rg -n aks` |
| Secret rotation | `az aks addon update -g rg -n aks -a azure-keyvault-secrets-provider --enable-secret-rotation --rotation-poll-interval 2m` |
| Namespace-scoped Azure RBAC | `az role assignment create --assignee <id> --role "Azure Kubernetes Service RBAC Writer" --scope <aksId>/namespaces/shop` |

Workload identity checklist: OIDC issuer + workload identity enabled → user-assigned identity with data-plane role → federated credential `system:serviceaccount:<ns>:<sa>` → SA annotation `azure.workload.identity/client-id` → pod label `azure.workload.identity/use: "true"` → SDK `DefaultAzureCredential`.

## ACR

| Task | Command |
|------|---------|
| Create | `az acr create -g rg -n acrshop --sku Standard --admin-enabled false` |
| Login (Docker) / token only | `az acr login -n acrshop` / `az acr login -n acrshop --expose-token` |
| Cloud build | `az acr build -r acrshop -t shop/web:{{.Run.ID}} .` |
| Import public image | `az acr import -n acrshop --source docker.io/library/nginx:1.27 --image base/nginx:1.27` |
| Repos / tags | `az acr repository list -n acrshop -o table` / `az acr repository show-tags -n acrshop --repository shop/web --orderby time_desc -o table` |
| Lock a tag | `az acr repository update -n acrshop --image shop/web:1.4.2 --write-enabled false` |
| Purge old images | `az acr run -r acrshop --cmd "acr purge --filter 'shop/web:.*' --ago 30d --keep 10 --untagged" /dev/null` |
| Disable public access | `az acr update -n acrshop --public-network-enabled false` (Premium + Private Endpoint) |
| Geo-replicate | `az acr replication create -r acrshop -l southindia` (Premium) |

## App Service, Functions and Container Apps

| Task | Command |
|------|---------|
| Plan + web app (container) | `az appservice plan create -g rg -n asp-shop --is-linux --sku P1v3` / `az webapp create -g rg -p asp-shop -n app-shop --deployment-container-image-name acrshop.azurecr.io/shop/web:v1` |
| App settings / port | `az webapp config appsettings set -g rg -n app-shop --settings WEBSITES_PORT=8080` |
| Key Vault reference | `DB_PASSWORD=@Microsoft.KeyVault(SecretUri=https://kv-shop.vault.azure.net/secrets/db-password/)` |
| Slot + swap | `az webapp deployment slot create -g rg -n app-shop --slot staging` / `az webapp deployment slot swap -g rg -n app-shop --slot staging` |
| Stream logs | `az webapp log tail -g rg -n app-shop` |
| VNet integration | `az webapp vnet-integration add -g rg -n app-shop --vnet vnet-shop --subnet snet-integration` |
| Function app (Flex Consumption) | `az functionapp create -g rg -n fn-shop --storage-account stshopfn --flexconsumption-location centralindia --runtime python --runtime-version 3.12` |
| Container Apps env + app | `az containerapp env create -g rg -n cae-shop -l centralindia` / `az containerapp create -g rg -n api --environment cae-shop --image acrshop.azurecr.io/shop/api:v1 --ingress external --target-port 8080 --registry-server acrshop.azurecr.io --registry-identity system` |
| Traffic split | `az containerapp ingress traffic set -g rg -n api --revision-weight api--v1=90 api--v2=10` |

## Storage

| Task | Command |
|------|---------|
| Secure account | `az storage account create -g rg -n stshop --sku Standard_ZRS --kind StorageV2 --min-tls-version TLS1_2 --allow-blob-public-access false --https-only true` |
| Disable shared keys | `az storage account update -g rg -n stshop --allow-shared-key-access false` |
| Container / upload (Entra auth) | `az storage container create -n reports --account-name stshop --auth-mode login` / `az storage blob upload --account-name stshop -c reports -f q3.pdf -n q3.pdf --auth-mode login` |
| User delegation SAS | `az storage blob generate-sas --account-name stshop -c reports -n q3.pdf --permissions r --expiry 2026-12-01T00:00Z --auth-mode login --as-user --full-uri` |
| Versioning + soft delete | `az storage account blob-service-properties update -g rg --account-name stshop --enable-versioning true --enable-delete-retention true --delete-retention-days 14` |
| Lifecycle policy | `az storage account management-policy create -g rg --account-name stshop --policy @policy.json` |
| Network deny by default | `az storage account update -g rg -n stshop --default-action Deny --bypass AzureServices` |
| Copy at scale | `azcopy login --identity` then `azcopy copy "https://src.blob.core.windows.net/c/*" "https://dst.blob.core.windows.net/c" --recursive` |

Tiers: Hot → Cool (30 days min) → Cold (90) → Archive (180, offline, hours to rehydrate). Redundancy: LRS, ZRS, GRS, GZRS, RA-GRS, RA-GZRS.

## Key Vault

| Task | Command |
|------|---------|
| Create (RBAC) | `az keyvault create -g rg -n kv-shop -l centralindia --enable-rbac-authorization true --retention-days 90 --enable-purge-protection true` |
| Set / get secret | `az keyvault secret set --vault-name kv-shop -n db-password --value "..."` / `az keyvault secret show --vault-name kv-shop -n db-password --query value -o tsv` |
| Versions | `az keyvault secret list-versions --vault-name kv-shop -n db-password -o table` |
| Secret expiry | `az keyvault secret set-attributes --vault-name kv-shop -n db-password --expires 2027-01-01T00:00:00Z` |
| Deleted vaults / recover / purge | `az keyvault list-deleted` / `az keyvault recover -n kv-shop` / `az keyvault purge -n kv-shop` |
| Firewall | `az keyvault update -n kv-shop --default-action Deny --bypass AzureServices` / `az keyvault network-rule add -n kv-shop --ip-address 1.2.3.4/32` |
| Key rotation policy | `az keyvault key rotation-policy update --vault-name kv-shop -n cmk --value @rotation.json` |

SecretProviderClass skeleton (CSI driver, workload identity):

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata: { name: kv-shop, namespace: shop }
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "<uami-client-id>"
    keyvaultName: kv-shop
    tenantId: "<tenant-id>"
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
```

## Azure Monitor and KQL

| Task | Command |
|------|---------|
| Workspace | `az monitor log-analytics workspace create -g rg -n law-shop --retention-time 30` |
| Diagnostic setting | `az monitor diagnostic-settings create -n to-law --resource <id> --workspace <lawId> --logs '[{"categoryGroup":"allLogs","enabled":true}]' --metrics '[{"category":"AllMetrics","enabled":true}]'` |
| List metric names | `az monitor metrics list-definitions --resource <id> -o table` |
| Query metrics | `az monitor metrics list --resource <aksId> --metric node_cpu_usage_percentage --interval PT5M -o table` |
| Query logs | `az monitor log-analytics query -w <workspaceGuid> --analytics-query "<KQL>" -o table` |
| Action group | `az monitor action-group create -g rg -n ag-oncall --short-name oncall --action email ops ops@example.com` |
| Metric alert | `az monitor metrics alert create -g rg -n cpu-high --scopes <id> --condition "avg Percentage CPU > 80" --window-size 5m --action ag-oncall` |
| Enable Container Insights | `az aks enable-addons -g rg -n aks -a monitoring --workspace-resource-id <lawId>` |
| Managed Prometheus | `az aks update -g rg -n aks --enable-azure-monitor-metrics --azure-monitor-workspace-resource-id <amwId>` |

```kusto
// Error logs per pod, last hour
ContainerLogV2
| where TimeGenerated > ago(1h) and LogLevel in ("error", "critical")
| summarize errors = count() by PodNamespace, PodName
| top 10 by errors

// Kubernetes warning events
KubeEvents
| where TimeGenerated > ago(1h) and KubeEventType == "Warning"
| project TimeGenerated, Namespace, Name, Reason, Message

// Failed requests from App Insights (workspace-based)
AppRequests
| where TimeGenerated > ago(1h) and Success == false
| summarize count() by Name, ResultCode, bin(TimeGenerated, 5m)

// What changed (control plane)
AzureActivity
| where TimeGenerated > ago(1d) and ActivityStatusValue == "Success" and OperationNameValue endswith "write"
| project TimeGenerated, Caller, OperationNameValue, ResourceGroup, _ResourceId

// Who costs the most in Log Analytics
Usage
| where TimeGenerated > ago(30d) and IsBillable
| summarize GB = sum(Quantity) / 1000 by DataType
| top 10 by GB
```

## Policy and cost

| Task | Command |
|------|---------|
| Find a built-in policy | `az policy definition list --query "[?contains(displayName,'Allowed locations')].{name:name, d:displayName}" -o table` |
| Assign with parameters | `az policy assignment create -n allowed-locs --policy e56962a6-4747-49cd-b67b-bf8b01975c4c --scope <scope> --params '{"listOfAllowedLocations":{"value":["centralindia","southindia"]}}'` |
| Assign DINE / Modify (needs identity) | `az policy assignment create ... --mi-system-assigned --location centralindia --role Contributor --identity-scope <scope>` |
| Compliance summary | `az policy state summarize --subscription <subId>` |
| Non-compliant resources | `az policy state list --filter "complianceState eq 'NonCompliant'" --query "[].resourceId" -o tsv` |
| Trigger scan | `az policy state trigger-scan -g rg` |
| Remediate | `az policy remediation create -n fix --policy-assignment <assignmentId>` |
| Exemption | `az policy exemption create -n waiver --policy-assignment <id> --exemption-category Waiver --scope <scope> --expires-on 2026-12-31` |
| Budget | `az consumption budget create --budget-name dev-monthly --amount 200 --time-grain Monthly --category cost --start-date 2026-11-01 --end-date 2027-10-31 -g rg-shop-dev` |
| Advisor cost tips | `az advisor recommendation list --category Cost -o table` |

Effects: `Deny`, `Audit`, `AuditIfNotExists`, `Append`, `Modify`, `DeployIfNotExists`, `DenyAction`, `Disabled`, `Manual`. Savings levers: right-size → deallocate idle → Spot → Savings Plan / Reservations → Azure Hybrid Benefit → Log Analytics plans and caps.

## Bicep and ARM

| Task | Command |
|------|---------|
| Install / upgrade | `az bicep install` / `az bicep upgrade` |
| Build / decompile / lint | `az bicep build -f main.bicep` / `az bicep decompile -f azuredeploy.json` / `az bicep lint -f main.bicep` |
| Validate / what-if / deploy (RG) | `az deployment group validate\|what-if\|create -g rg -f main.bicep -p prod.bicepparam` |
| Subscription scope | `az deployment sub create -l centralindia -f sub.bicep` |
| Failed operations | `az deployment operation group list -g rg -n <deployment> --query "[?properties.provisioningState=='Failed']"` |
| Deployment stack | `az stack group create -n shop -g rg -f main.bicep --action-on-unmanage deleteResources --deny-settings-mode denyDelete` |
| Export existing RG | `az group export -n rg` (then decompile) |

```bicep
targetScope = 'resourceGroup'
param location string = resourceGroup().location
param env string
var suffix = uniqueString(resourceGroup().id)

resource kv 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'kv-shop-${env}-${take(suffix, 5)}'
  location: location
  properties: {
    tenantId: subscription().tenantId
    sku: { family: 'A', name: 'standard' }
    enableRbacAuthorization: true
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
  }
}

module acr 'br/public:avm/res/container-registry/registry:0.6.0' = {
  name: 'acr'
  params: { name: 'acrshop${env}${take(suffix, 5)}', acrSku: 'Standard' }
}
```

Check the AVM module's current version in the Bicep registry before using it; the version above is an example.

## Terraform azurerm

```hcl
terraform {
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 4.0" }
  }
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "sttfstateshop"
    container_name       = "tfstate"
    key                  = "shop/prod.tfstate"
    use_azuread_auth     = true
  }
}
provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}
```

| Task | Command / snippet |
|------|---------|
| Auth from CLI (laptop) | `az login`, provider uses Azure CLI automatically |
| Auth OIDC (CI) | `ARM_CLIENT_ID`, `ARM_TENANT_ID`, `ARM_SUBSCRIPTION_ID`, `ARM_USE_OIDC=true` |
| Auth managed identity | `ARM_USE_MSI=true` (+ `ARM_CLIENT_ID` for user-assigned) |
| Init with backend file | `terraform init -backend-config=prod.backend.hcl` |
| Saved plan / apply | `terraform plan -out tfplan` / `terraform apply tfplan` |
| Drift check | `terraform plan -detailed-exitcode` (0 none, 1 error, 2 changes) |
| Import existing | `import { to = azurerm_resource_group.rg  id = "/subscriptions/<sub>/resourceGroups/rg" }` + `terraform plan -generate-config-out=gen.tf` |
| Break lease | `az storage blob lease break --container-name tfstate --blob-name shop/prod.tfstate --account-name sttfstateshop --auth-mode login` |
| Bulk export | `aztfexport resource-group rg-shop` |
| AcrPull for kubelet | `principal_id = azurerm_kubernetes_cluster.aks.kubelet_identity[0].object_id` |
| Avoid PrincipalNotFound | `skip_service_principal_aad_check = true` on `azurerm_role_assignment` |
| Autoscaler-owned count | `lifecycle { ignore_changes = [default_node_pool[0].node_count] }` |
| Rotate default pool safely | `temporary_name_for_rotation = "systemtmp"` in `default_node_pool` |
| Unsupported feature | `azapi_resource` / `azapi_update_resource` with an ARM API version |

v4 renames worth knowing: `enable_auto_scaling` → `auto_scaling_enabled`, `enable_https_traffic_only` → `https_traffic_only_enabled`, `subscription_id` required, `resource_provider_registrations` replaces auto-registration.

## Pipelines: GitHub Actions and Azure DevOps

```yaml
# GitHub Actions: OIDC login
permissions:
  id-token: write
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: prod
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - run: az acr build -r ${{ vars.ACR_NAME }} -t shop/web:${{ github.sha }} .
```

```yaml
# Azure Pipelines: WIF service connection + environment
stages:
- stage: Deploy
  jobs:
  - deployment: aks
    environment: shop-prod          # approvals and checks live here
    pool: { vmImage: ubuntu-latest }
    strategy:
      runOnce:
        deploy:
          steps:
          - checkout: self
          - task: AzureCLI@2
            inputs:
              azureSubscription: arm-shop-prod
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                az aks get-credentials -g rg-shop -n aks-shop --overwrite-existing
                kubelogin convert-kubeconfig -l azurecli
                kubectl apply -f k8s/
```

| Azure DevOps item | Use |
|---|---|
| `##vso[task.setvariable variable=x;isOutput=true]v` | Set a variable from a script |
| `$(var)` / `${{ variables.var }}` / `$[ variables.var ]` | Macro (runtime) / template (compile) / runtime expression |
| Variable group linked to Key Vault | Secrets fetched at runtime |
| `extends: template: x.yml@templates` | Enforced pipeline skeleton |
| Environment checks | Approvals, branch control, business hours, required template, exclusive lock |
| `az pipelines run --name shop-ci --branch main` | Queue a run from CLI (`azure-devops` extension) |

## Common errors decoder

| Error text | First thing to check |
|---|---|
| `AuthorizationFailed ... does not have authorization to perform action` | Role at that scope; right subscription; re-login after grant |
| `LinkedAuthorizationFailed ... subnets/join/action` | Rights on the subnet in the network RG |
| `AADSTS70021 No matching federated identity record` | Subject string vs federated credential (branch / environment / PR) |
| `AADSTS7000222 client secret keys ... are expired` | Rotate, then move to OIDC |
| `AADSTS50076 ... must use multi-factor authentication` | Automation running as a human account |
| `ForbiddenByRbac` (Key Vault) | Key Vault data role (Secrets User / Officer) |
| `ForbiddenByFirewall` / `Client address is not authorized` | Vault firewall, Private Endpoint DNS |
| `AuthorizationPermissionMismatch` (Storage) | Missing Storage Blob Data role |
| `AuthorizationFailure` (Storage) | Storage network rules / private access |
| `QuotaExceeded ... Cores quota` | `az vm list-usage`, raise quota or change family |
| `SkuNotAvailable` / `ZonalAllocationFailed` | Other size or zone; capacity reservation |
| `MissingSubscriptionRegistration` | `az provider register -n <namespace>` |
| `RequestDisallowedByPolicy` | Policy assignment ID in the error; comply or exempt |
| `ScopeLocked` | `az lock list` |
| `ImagePullBackOff ... 401 Unauthorized` (ACR) | Kubelet identity AcrPull; `az aks check-acr` |
| `ImagePullBackOff ... no such host` (ACR) | `privatelink.azurecr.io` zone link / firewall FQDNs |
| `MountVolume.SetUp failed ... secrets-store ... 403` | Workload identity role on Key Vault, `clientID`, network |
| `User does not have access to the resource in Azure` (kubectl) | AKS RBAC role at cluster or namespace scope |
| `kubelogin is not installed` | `az aks install-cli`, `kubelogin convert-kubeconfig` |
| `UpgradeFailed ... Pod Disruption Budget` | Relax PDB, add replicas, resume upgrade |
| `InsufficientSubnetSize` | Flat CNI IP math; Overlay; smaller surge |
| `EXTERNAL-IP <pending>` + `AuthorizationFailed` | Cluster identity Network Contributor on subnet |

## AWS to Azure quick map

| AWS | Azure |
|---|---|
| Organizations, OU, SCP | Management groups, Azure Policy (deny) |
| Account | Subscription (+ resource groups inside) |
| IAM role for EC2 / IRSA / Pod Identity | Managed identity / AKS workload identity |
| IAM Identity Center | Entra ID + PIM |
| VPC, subnet (per AZ) | VNet, subnet (spans zones) |
| Security group / NACL | NSG on NIC / NSG on subnet (+ ASG) |
| Transit Gateway | Virtual WAN or hub-spoke with firewall |
| Interface / gateway endpoint | Private Endpoint / Service Endpoint |
| NAT gateway | NAT Gateway |
| ALB / NLB / CloudFront / Global Accelerator | Application Gateway / Load Balancer / Front Door / Front Door or cross-region LB |
| Route 53 | Azure DNS + Traffic Manager |
| EC2, ASG, Launch template | VM, VMSS, VMSS model / Compute Gallery image |
| EKS, ECR, Fargate, Lambda | AKS, ACR, Container Apps / ACI, Functions |
| S3, EBS, EFS | Blob Storage, Managed Disks, Azure Files |
| Secrets Manager, KMS, CloudHSM | Key Vault, Key Vault keys, Managed HSM |
| CloudWatch, X-Ray, CloudTrail | Azure Monitor, Application Insights, Activity Log |
| Config, Security Hub, GuardDuty | Azure Policy, Defender for Cloud |
| CloudFormation, CDK | ARM templates, Bicep |
| CodeCommit, CodeBuild, CodePipeline | Azure Repos, Azure Pipelines (or GitHub) |

## Numbers worth remembering

| Item | Value |
|------|-------|
| Management group depth | 6 levels below root |
| Role assignments per subscription | 4,000 |
| Custom roles per tenant | 5,000 |
| Tags per resource | 50 |
| Reserved IPs per subnet | 5 |
| Smallest subnet | /29 |
| NSG rule priority range | 100 to 4096 (lower wins) |
| SNAT ports per NAT Gateway public IP | 64,512 |
| RBAC propagation | usually minutes, up to about 10 (longer at management group scope) |
| Activity Log retention | 90 days (export for longer) |
| Platform metrics retention | 93 days |
| Key Vault soft delete | 7 to 90 days, default 90 |
| Spot eviction notice | 30 seconds |
| AKS supported versions | N-2 minor versions (GA) |
| AKS Standard tier SLA | 99.95% with zones, 99.9% without |
| AKS default max pods per node (Azure CNI) | 30 (Overlay default 250) |
| Flat Azure CNI IPs per node | maxPods + 1 |
| Blob min retention for early-deletion fees | Cool 30, Cold 90, Archive 180 days |
| Storage account name | 3–24 chars, lowercase letters and numbers, globally unique |
| Key Vault name | 3–24 chars, globally unique |
| `az acr login` token lifetime | about 3 hours |

Limits change. Confirm current values in the Azure subscription and service limits documentation when it matters.

## Top 15 interview one-liners

1. Tenant is identity, subscription is billing and quota, resource group is lifecycle, management group is governance.
2. RBAC decides who can act; Azure Policy decides what resources may look like. You need both.
3. RBAC is additive. Owner has no data-plane access, so reading blobs or secrets needs data roles like Storage Blob Data Reader or Key Vault Secrets User.
4. Prefer managed identities, then federated credentials. Client secrets are the last resort.
5. GitHub and Azure DevOps log in with OIDC: the federated subject (branch, environment or service connection) must match exactly.
6. Azure subnets span availability zones. Peering is not transitive; spoke-to-spoke traffic goes through a hub firewall or Virtual WAN.
7. NSGs are stateful with priorities, and inbound traffic must pass the subnet NSG first, then the NIC NSG.
8. A Private Endpoint is only half the job; the other half is private DNS (`privatelink` zones linked to the VNets that resolve them).
9. The kubelet identity pulls from ACR (AcrPull). The cluster identity manages Azure resources such as LBs and subnets.
10. New AKS clusters: Azure CNI Overlay with Cilium, Entra + Azure RBAC, local accounts off, workload identity, Standard tier in prod.
11. AKS upgrades go one minor version at a time, control plane first, with surge nodes and PDBs that allow at least one disruption.
12. Key Vault CSI mounts secrets at pod start and can rotate files. Env vars from synced Secrets don't rotate without a restart.
13. Container Insights with managed identity auth needs a Data Collection Rule. Log Analytics ingestion is the AKS bill surprise.
14. Bicep has no state file because ARM is the state. Terraform has explicit state, so protect it with Entra auth, versioning and locks.
15. Biggest cost wins: deallocate idle compute, right-size before you reserve, Spot for interruptible work, and control log ingestion.

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
