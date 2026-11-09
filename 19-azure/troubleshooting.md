# Azure for DevOps — Troubleshooting Drills

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

> Real Azure for DevOps errors and symptoms: likely causes, diagnosis, fix and prevention · 100 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Azure for DevOps](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Identity and Authorization Errors](#identity-and-authorization-errors) — Q1–Q20
- [Quota, Deployment and IaC Errors](#quota-deployment-and-iac-errors) — Q21–Q40
- [Network Connectivity Errors](#network-connectivity-errors) — Q41–Q60
- [AKS and ACR Errors](#aks-and-acr-errors) — Q61–Q85
- [Storage, Key Vault, Monitoring and Pipeline Errors](#storage-key-vault-monitoring-and-pipeline-errors) — Q86–Q100

---

## Identity and Authorization Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. `(AuthorizationFailed) The client 'ci@contoso.com' with object id '1111...' does not have authorization to perform action 'Microsoft.Resources/subscriptions/resourcegroups/write' over scope '/subscriptions/.../resourcegroups/rg-shop-dev' or the scope is invalid.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No role with that action at that scope (or above); a newly granted role not propagated yet; wrong subscription selected so the scope is different from what you think.

**Diagnose →**

```bash
az account show --query "{sub:id, user:user.name}"
az role assignment list --assignee <objectId> --all --include-inherited -o table
```

**Fix →**

1. Confirm the subscription in the error matches your intended one; `az account set` if not
2. Grant a role that includes the action at the right scope (Contributor on the subscription to create RGs, or pre-create the RG and grant Contributor on it)
3. If just granted, wait a few minutes, then `az logout && az login` to refresh tokens

**Prevent →** Pre-create RGs in the platform pipeline and scope app identities to them.

</details>

### Q2. `AADSTS70021: No matching federated identity record found for presented assertion. Assertion Issuer: 'https://token.actions.githubusercontent.com'. Assertion Subject: 'repo:devopsshack/shop:ref:refs/heads/feature/login'.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The federated credential subject doesn't match: workflow ran on a different branch, uses an environment (subject becomes `environment:<name>`), was a PR, or org/repo name casing changed.

**Diagnose →**

```bash
az identity federated-credential list --identity-name id-shop-ci -g rg-ci \
  --query "[].{name:name, subject:subject, issuer:issuer}" -o table
```

Compare the subject in the error with the list exactly.

**Fix →**

1. Add a federated credential for the needed subject, or run the job from the expected branch/environment
2. If the job declares `environment: prod`, the subject must be `repo:devopsshack/shop:environment:prod`

**Prevent →** Standardise on environment-based subjects and document them next to the workflow.

</details>

### Q3. `AADSTS7000222: The provided client secret keys for app '2222...' are expired.` Pipelines failed overnight. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The service principal's client secret reached its end date.

**Diagnose →**

```bash
az ad app credential list --id <appId> --query "[].{keyId:keyId, end:endDateTime}" -o table
```

**Fix →**

1. Short term: `az ad app credential reset --id <appId> --append --years 1`, update the CI secret store, rerun
2. Real fix: switch the pipeline to workload identity federation and delete the secret

**Prevent →** Federated credentials (nothing expires), or alerts on upcoming secret expiry via a scheduled Graph query.

</details>

### Q4. `AADSTS50076: Due to a configuration change made by your administrator, or because you moved to a new location, you must use multi-factor authentication to access '797f4846-ba00-4fd7-ba43-dac1f8f63013'.` in a nightly script. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The script uses a human account, and a Conditional Access policy now requires MFA for Azure management (`797f4846...` is the Windows Azure Service Management API).

**Diagnose →** Check Entra sign-in logs for the account, which show the CA policy applied.

**Fix →**

1. Move the script to a managed identity (if it runs in Azure) or a service principal with federated credentials
2. Grant that identity the needed roles; remove the human account from automation

**Prevent →** Policy: no automation with user accounts. Conditional Access will keep tightening.

</details>

### Q5. `Code: Forbidden. Message: Caller is not authorized to perform action on resource. Action: 'Microsoft.KeyVault/vaults/secrets/getSecret/action' ... Inner error: {"code":"ForbiddenByRbac"}` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The vault uses RBAC and the caller has no data-plane role; being Owner or Contributor of the vault doesn't grant secret read.

**Diagnose →**

```bash
az keyvault show -n kv-shop --query properties.enableRbacAuthorization
az role assignment list --scope $(az keyvault show -n kv-shop --query id -o tsv) -o table
```

**Fix →**

1. Grant `Key Vault Secrets User` (read) or `Key Vault Secrets Officer` (manage) to the caller at vault scope
2. Wait for propagation and retry

**Prevent →** Put Key Vault data roles in the same IaC that creates the identity.

</details>

### Q6. `The user, group or application 'appid=...;oid=...' does not have secrets get permission on key vault 'kv-legacy;location=centralindia'.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The vault uses the legacy access policy model and the identity has no access policy (or lacks `get`).

**Diagnose →**

```bash
az keyvault show -n kv-legacy --query "properties.accessPolicies[].{oid:objectId, secrets:permissions.secrets}"
```

**Fix →**

1. Short term: `az keyvault set-policy -n kv-legacy --object-id <oid> --secret-permissions get list`
2. Plan migration to RBAC (assign equivalent roles, then switch `enableRbacAuthorization`)

**Prevent →** Create new vaults with RBAC only.

</details>

### Q7. A user was added to an Entra group with access, but still gets `AuthorizationFailed`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Group membership is in the token; the old token doesn't have it. Or propagation delay.

**Diagnose →**

```bash
az ad group member check --group grp-shop-dev --member-id <userObjectId>
```

**Fix →**

1. `az logout && az login` (or sign out of the portal) to get a new token
2. Wait up to around 10 minutes for RBAC propagation

**Prevent →** Tell users to re-login after group changes; build it into onboarding docs.

</details>

### Q8. `PrincipalNotFound: Principal 3333... does not exist in the directory <tenantId>.` when Terraform creates a role assignment for a new identity. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Entra replication lag: the identity was just created and ARM's lookup can't see it yet.

**Diagnose →** Re-run after a minute; it usually succeeds, which confirms replication delay.

**Fix →**

1. In Terraform, set `skip_service_principal_aad_check = true` on `azurerm_role_assignment` for service principals and managed identities
2. In CLI, pass `--assignee-object-id <id> --assignee-principal-type ServicePrincipal`

**Prevent →** Always specify principal type for newly created identities.

</details>

### Q9. `RoleAssignmentExists: The role assignment already exists.` breaks a Terraform apply. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The same principal/role/scope assignment was created outside Terraform (portal, `--attach-acr`, another module).

**Diagnose →**

```bash
az role assignment list --assignee <principalId> --scope <scope> --query "[].id" -o tsv
```

**Fix →**

1. Import the existing assignment into state with an `import` block using its ID
2. Or remove the duplicate from code if another module owns it

**Prevent →** One owner per role assignment; avoid mixing `az aks update --attach-acr` with Terraform role assignments for the same pair.

</details>

### Q10. `azure/login` fails with `Error: Unable to get ACTIONS_ID_TOKEN_REQUEST_URL env variable`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The workflow or job lacks `id-token: write` permission, or it runs from a fork PR where OIDC tokens are not issued.

**Diagnose →** Check the workflow's `permissions:` block at workflow and job level.

**Fix →**

```yaml
permissions:
  id-token: write
  contents: read
```

**Prevent →** Use a reusable workflow template that already includes the permissions.

</details>

### Q11. `Error: AADSTS700016: Application with identifier '4444...' was not found in the directory 'Contoso'.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong client ID or tenant ID in config, the app was deleted, or the app lives in another tenant and has no service principal here.

**Diagnose →**

```bash
az ad sp show --id <clientId>
az ad app show --id <clientId>
```

**Fix →**

1. Correct the client ID / tenant ID pair (common when copying between environments)
2. Recreate the service principal: `az ad sp create --id <appId>` for multi-tenant apps
3. Restore deleted apps from Entra's deleted items within 30 days

**Prevent →** Store client/tenant IDs per environment, not hard-coded.

</details>

### Q12. A managed identity on a VM gets `ManagedIdentityCredential authentication unavailable. No identity has been assigned to this resource.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No identity is enabled, or only a user-assigned identity exists and the code doesn't specify its client ID.

**Diagnose →**

```bash
az vm identity show -g rg -n vm-app
```

**Fix →**

1. Enable a system-assigned identity, or
2. For user-assigned, set `AZURE_CLIENT_ID=<clientId>` or pass `ManagedIdentityCredential(client_id=...)`

**Prevent →** Always configure the client ID explicitly when using user-assigned identities.

</details>

### Q13. `LinkedAuthorizationFailed: The client has permission to perform action 'Microsoft.Network/networkInterfaces/write' ... however, it does not have permission to perform action 'Microsoft.Network/virtualNetworks/subnets/join/action' on the linked scope(s)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The identity can create resources in its RG but the subnet is in another RG (often the network team's) where it has no rights.

**Diagnose →** Read the linked scope in the error; check assignments there.

**Fix →**

1. Grant a role with `subnets/join/action` on the subnet (Network Contributor, or a custom "Subnet Joiner" role)
2. Retry the deployment

**Prevent →** Provide a custom least-privilege subnet join role for app teams in hub-spoke designs.

</details>

### Q14. AKS `kubectl get pods` returns `Error from server (Forbidden): pods is forbidden: User "dev@devopsshack.com" cannot list resource "pods" in API group "" in the namespace "shop": User does not have access to the resource in Azure. Update role assignment to allow access.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Azure RBAC for Kubernetes is enabled and the user has no AKS RBAC role at the cluster or namespace scope.

**Diagnose →**

```bash
az role assignment list --assignee dev@devopsshack.com --scope $(az aks show -g rg -n aks --query id -o tsv) --include-inherited -o table
```

**Fix →**

```bash
az role assignment create --assignee dev@devopsshack.com --role "Azure Kubernetes Service RBAC Reader" \
  --scope "$(az aks show -g rg -n aks --query id -o tsv)/namespaces/shop"
```

**Prevent →** Grant namespace roles to groups as part of namespace onboarding.

</details>

### Q15. `kubelogin is not installed which is required to connect to AAD enabled cluster.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The kubeconfig uses the kubelogin exec plugin and it isn't installed or not on PATH.

**Diagnose →** `which kubelogin`; `kubectl config view --minify` shows `exec: command: kubelogin`.

**Fix →**

```bash
az aks install-cli          # installs kubectl and kubelogin (may need sudo)
kubelogin convert-kubeconfig -l azurecli
```

**Prevent →** Include kubelogin in CI images and developer setup scripts.

</details>

### Q16. `ERROR: (AuthorizationFailed) ... does not have authorization to perform action 'Microsoft.ContainerService/managedClusters/listClusterUserCredential/action'` running `az aks get-credentials`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The identity has Kubernetes RBAC roles but not the Azure role to download kubeconfig.

**Diagnose →** Check roles at cluster scope.

**Fix →**

1. Grant `Azure Kubernetes Service Cluster User Role` on the cluster
2. Admin credentials (`--admin`) need `Azure Kubernetes Service Cluster Admin Role` and local accounts enabled

**Prevent →** Pair Cluster User Role with every Kubernetes RBAC role grant.

</details>

### Q17. Azure DevOps fails with `AADSTS700213: No matching federated identity record found for presented assertion subject 'sc://contoso/shop/arm-shop-prod'.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The service connection was renamed, or the project/org name differs from when the federated credential was created; the subject includes them.

**Diagnose →** Compare the subject in the error with the federated credential on the app/identity.

**Fix →**

1. Update the federated credential subject to `sc://contoso/shop/arm-shop-prod`
2. Or use the service connection's "convert/verify" flow to regenerate it

**Prevent →** Don't rename service connections casually; document that the name is part of the trust.

</details>

### Q18. `RequestDisallowedByPolicy: Resource 'stshopdev' was disallowed by policy. Reasons: 'Storage accounts should disable public network access'.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A deny policy is assigned at a scope above; the request doesn't meet it.

**Diagnose →**

```bash
az policy assignment show --name <assignmentName> --scope <scope>
az policy definition show -n <definitionName> --query policyRule
```

**Fix →**

1. Change the deployment to comply (`--public-network-access Disabled` plus Private Endpoint)
2. If justified, request an expiring exemption from the policy owners

**Prevent →** Run policy-aware checks in CI (PSRule for Azure, or a what-if against a test subscription with the same policies).

</details>

### Q19. A Terraform pipeline fails with `Error: building account: could not acquire access token to parse claims: clientCredentialsToken: received HTTP status 400 with response: {"error":"invalid_client","error_description":"AADSTS7000215: Invalid client secret provided..."}`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The client secret value is wrong (secret ID pasted instead of value), rotated, or contains an escaped character mangled by CI.

**Diagnose →** Check the CI variable; confirm the secret's key ID and expiry with `az ad app credential list --id <appId>`.

**Fix →**

1. Reset the secret and paste the **value**
2. Better, switch to `ARM_USE_OIDC=true` and remove `ARM_CLIENT_SECRET`

**Prevent →** OIDC everywhere.

</details>

### Q20. After moving a subscription to another tenant, apps fail with token errors and all role assignments are gone. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tenant transfer deletes RBAC assignments, custom roles and managed identities' trust; Key Vaults keep the old tenant ID.

**Diagnose →** `az role assignment list --all` returns almost nothing; Key Vault `properties.tenantId` shows the old tenant.

**Fix →**

1. Recreate role assignments from IaC or a pre-move export
2. Disable and re-enable system-assigned identities; recreate user-assigned identities and reattach
3. Update Key Vault tenant ID and access policies/roles
4. Recreate AKS-related identity bindings (or the cluster)

**Prevent →** Export assignments before transfer (`az role assignment list --all > backup.json`) and follow Microsoft's transfer checklist.

</details>

## Quota, Deployment and IaC Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q21. `(QuotaExceeded) Operation could not be completed as it results in exceeding approved standardDSv5Family Cores quota. Location: centralindia, Current Limit: 10, Current Usage: 8, Additional Required: 4, (Minimum) New Limit Required: 12.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Family vCPU quota is too low for the scale-out or new pool (including surge nodes during AKS upgrades).

**Diagnose →**

```bash
az vm list-usage -l centralindia -o table | grep -Ei "DSv5|Total Regional"
```

**Fix →**

1. Request an increase (portal Quotas, or `az quota update`) for the family and total regional vCPUs
2. Short term: use another VM family with headroom or lower surge

**Prevent →** Track quota usage in a dashboard and alert at 80%; pre-raise before launches.

</details>

### Q22. `SkuNotAvailable: The requested VM size Standard_D4s_v5 is currently not available in location 'centralindia' zones '3'.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Capacity restriction for that SKU/zone in your subscription, or SKU not offered in that zone.

**Diagnose →**

```bash
az vm list-skus -l centralindia --size Standard_D4s_v5 --query "[].{name:name, zones:locationInfo[0].zones, restrictions:restrictions}" -o json
```

**Fix →**

1. Pick a different size (D4as_v5, D4s_v6) or drop the restricted zone
2. Open a support request to lift subscription-level restrictions

**Prevent →** Keep two or three acceptable SKUs per workload; check availability in IaC pipelines.

</details>

### Q23. `MissingSubscriptionRegistration: The subscription is not registered to use namespace 'Microsoft.ContainerService'.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The resource provider was never registered in this subscription.

**Diagnose →** `az provider show -n Microsoft.ContainerService --query registrationState`

**Fix →**

```bash
az provider register -n Microsoft.ContainerService --wait
```

**Prevent →** Register required providers in the subscription vending process.

</details>

### Q24. `StorageAccountAlreadyTaken: The storage account named sttfstate is already taken.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Storage account names are globally unique across all of Azure.

**Diagnose →** `az storage account check-name -n sttfstate`

**Fix →** Use a unique suffix: `sttfstate${RANDOM}` or a hash of subscription ID (`uniqueString(subscription().id)` in Bicep, `random_string` in Terraform).

**Prevent →** Naming convention with a unique suffix for globally unique names (storage, Key Vault, ACR).

</details>

### Q25. `(VaultAlreadyExists)` or `A vault with the same name already exists in deleted state. You need to either recover or purge existing key vault.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You deleted the vault (often via `terraform destroy`) and soft delete keeps the name reserved.

**Diagnose →** `az keyvault list-deleted --query "[].{name:name, purge:properties.scheduledPurgeDate}" -o table`

**Fix →**

1. Recover it: `az keyvault recover -n kv-shop-dev`
2. Or purge (only if purge protection is off): `az keyvault purge -n kv-shop-dev`
3. In Terraform labs, `features { key_vault { purge_soft_delete_on_destroy = true } }`

**Prevent →** Unique names per lab run, or the purge feature flag in non-prod.

</details>

### Q26. `ScopeLocked: The scope '/subscriptions/.../resourceGroups/rg-shop-prod' cannot perform delete operation because following scope(s) are locked: '/subscriptions/.../resourceGroups/rg-shop-prod'.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A CanNotDelete or ReadOnly lock on the RG or a parent.

**Diagnose →** `az lock list -g rg-shop-prod -o table` and at subscription level.

**Fix →**

1. If the delete is intended, remove the lock (needs `Microsoft.Authorization/locks/delete`, Owner or User Access Administrator): `az lock delete -n no-delete -g rg-shop-prod`
2. Re-add after the change if appropriate

**Prevent →** Manage locks in IaC so their presence is intentional and visible.

</details>

### Q27. `DeploymentFailed: At least one resource deployment operation failed. Please list deployment operations for details.` How do you find the real error? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The top-level message is generic; the real error is in a nested operation.

**Diagnose →**

```bash
az deployment operation group list -g rg -n main \
  --query "[?properties.provisioningState=='Failed'].{res:properties.targetResource.resourceName, err:properties.statusMessage.error}" -o json
```

For nested modules, list operations of the nested deployment name too.

**Fix →** Fix the specific error (quota, name conflict, policy, invalid property).

**Prevent →** Pipelines should print failed operations automatically on failure.

</details>

### Q28. `InvalidTemplateDeployment: The template deployment 'main' is not valid according to the validation procedure.` with a policy violation inside. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Preflight validation found a policy deny or an invalid property before deploying anything.

**Diagnose →** Run `az deployment group validate -g rg -f main.bicep -p @prod.bicepparam` and read the `details` array.

**Fix →** Adjust the resource properties to comply or fix the invalid value.

**Prevent →** Run `validate` in PR pipelines against an environment with the same policies.

</details>

### Q29. Terraform: `Error: A resource with the ID "/subscriptions/.../resourceGroups/rg-shop-dev" already exists - to be managed via Terraform this resource needs to be imported into the State.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The resource exists in Azure (created manually or by a previous run whose state was lost).

**Diagnose →** `terraform state list | grep rg-shop-dev` returns nothing.

**Fix →**

```hcl
import {
  to = azurerm_resource_group.main
  id = "/subscriptions/<sub>/resourceGroups/rg-shop-dev"
}
```

Then `terraform plan` and `apply`.

**Prevent →** Remote state with locking and versioning so state isn't lost.

</details>

### Q30. Terraform: `Error: Error acquiring the state lock ... state blob is already locked`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another run holds the blob lease, or a crashed run left the lease.

**Diagnose →** The error shows the lock ID, who and when. Check running pipelines.

**Fix →**

1. Wait for the other run, or if it's dead: `terraform force-unlock <LOCK_ID>`
2. Or break the lease: `az storage blob lease break --container-name tfstate --blob-name shop/prod.tfstate --account-name sttfstate --auth-mode login`

**Prevent →** Pipeline concurrency limits per environment.

</details>

### Q31. Terraform azurerm v4: `Error: subscription ID could not be determined and was not specified`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** azurerm v4 requires an explicit subscription ID; v3 inferred it from the CLI.

**Diagnose →** Check the provider block and `ARM_SUBSCRIPTION_ID`.

**Fix →**

```hcl
provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}
```

or `export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)`.

**Prevent →** Pin provider versions and read upgrade guides before bumping majors.

</details>

### Q32. `OperationNotAllowed: Operation is not allowed because there's an in progress update managed cluster operation` on an AKS change. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another operation (upgrade, scale, add-on change, auto-upgrade) is in progress; AKS serialises operations.

**Diagnose →**

```bash
az aks show -g rg -n aks --query provisioningState
az aks operation show-latest -g rg -n aks
```

**Fix →** Wait for it to finish; if stuck for hours, check AKS diagnostics or open a support case. Avoid parallel pipelines touching the same cluster.

**Prevent →** Serialise cluster changes; schedule auto-upgrades away from deploy windows.

</details>

### Q33. `InvalidResourceLocation: The resource 'vnet-shop' already exists in location 'southindia' in resource group 'rg-shop'. A resource with the same name cannot be created in location 'centralindia'.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A resource with that name and type already exists in the RG in another region; names are unique per RG and type.

**Diagnose →** `az resource list -g rg-shop -n vnet-shop -o table`

**Fix →** Use region in names (`vnet-shop-cin`), or delete/move the old resource.

**Prevent →** Naming convention including region abbreviation.

</details>

### Q34. `InUseSubnetCannotBeDeleted: Subnet snet-app is in use by /subscriptions/.../networkInterfaces/... and cannot be deleted.` during destroy. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Something still has a NIC or IP config in the subnet: a VM, private endpoint, AKS LB frontend, App Service integration, or a leaked NIC.

**Diagnose →**

```bash
az network vnet subnet show -g rg --vnet-name vnet-shop -n snet-app --query "ipConfigurations[].id" -o tsv
```

**Fix →** Delete the referencing resources first (or disconnect App Service VNet integration), then the subnet.

**Prevent →** Model dependencies correctly in IaC; delete the whole RG for labs.

</details>

### Q35. `AllocationFailed` or `ZonalAllocationFailed: Allocation failed. We do not have sufficient capacity for the requested VM size in this zone.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Temporary regional/zonal capacity shortage for that size; not a quota issue.

**Diagnose →** The error code says allocation, not quota. Check other sizes/zones.

**Fix →**

1. Retry later, or choose a different size or zone
2. For VMSS/AKS, allow multiple sizes (Karpenter/NAP, or multiple node pools)
3. Use **on-demand capacity reservations** for critical capacity

**Prevent →** Capacity reservations for must-have capacity; flexible SKU lists.

</details>

### Q36. Bicep build error `BCP037: The property "enableRbacAuthorisation" is not allowed on objects of type "VaultProperties".` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Typo or wrong property for the API version (here British spelling instead of `enableRbacAuthorization`).

**Diagnose →** Hover in VS Code, or check the API reference for the resource type and API version.

**Fix →** Correct the property name; update API versions if the property is newer.

**Prevent →** Bicep VS Code extension and `az bicep lint` in CI.

</details>

### Q37. `Conflict: Cannot delete resource group 'rg-shop-dev' while it contains resources protected by Azure Backup` or Recovery Services vault deletion fails. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Recovery Services vault has protected items or soft-deleted backup items.

**Diagnose →** `az backup item list --vault-name rsv-shop -g rg-shop-dev -o table`

**Fix →**

1. Stop protection and delete backup data for each item
2. Undelete and delete soft-deleted items, or wait for soft delete period
3. Then delete the vault and the RG

**Prevent →** Lab cleanup scripts that disable soft delete in non-prod vaults (if policy allows).

</details>

### Q38. Terraform plan always shows a change on `tags` or a property even after apply. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Azure Policy Modify adds tags after apply, or Azure normalises a value (casing, defaults) differently from config.

**Diagnose →** Compare plan diff with Activity Log for policy-initiated writes.

**Fix →**

1. Add the policy-added tags to config, or `lifecycle { ignore_changes = [tags["created_by"]] }`
2. Match the exact casing Azure returns

**Prevent →** Coordinate tag policies with IaC defaults.

</details>

### Q39. `The resource operation completed with terminal provisioning state 'Failed'` for a VM extension (`VMExtensionProvisioningError`). 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Custom Script or other extension's script failed, couldn't download, or the VM agent is unhealthy.

**Diagnose →**

```bash
az vm get-instance-view -g rg -n vm-app --query "instanceView.extensions[].statuses"
# On the VM:
sudo tail -n 100 /var/log/azure/custom-script/handler.log
sudo ls /var/lib/waagent/custom-script/download/0/
```

**Fix →** Fix the script (exit codes, network access to the script URL), then redeploy with a new `timestamp` setting.

**Prevent →** Prefer cloud-init or baked images; test scripts locally.

</details>

### Q40. `DeploymentQuotaExceeded: Creating the deployment 'shop-…' would exceed the quota of '800'.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Resource group deployment history limit; usually auto-cleaned, but locks on the RG block automatic deletion of old deployment records.

**Diagnose →** `az deployment group list -g rg --query "length(@)"`; check for CanNotDelete locks.

**Fix →**

```bash
az deployment group list -g rg --query "[?properties.timestamp<'2026-09-01'].name" -o tsv \
  | xargs -I{} az deployment group delete -g rg -n {}
```

**Prevent →** Reuse deployment names per component, or remove locks that block history cleanup.

</details>

## Network Connectivity Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q41. A VM with a public IP times out on SSH (`ssh: connect to host 20.x.x.x port 22: Connection timed out`). 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** NSG doesn't allow 22 from your IP (Standard public IPs are closed by default), a second NSG on subnet or NIC, guest firewall, or sshd not running.

**Diagnose →**

```bash
az network nic list-effective-nsg -g rg -n vm-app-nic
az network watcher test-ip-flow -g rg --vm vm-app --direction Inbound --protocol TCP \
  --local 10.20.1.4:22 --remote <yourIP>:50000
az vm boot-diagnostics get-boot-log -g rg -n vm-app | tail
```

**Fix →** Add an allow rule for your IP on port 22 (or use Bastion); check `sudo ufw status` via Run Command: `az vm run-command invoke -g rg -n vm-app --command-id RunShellScript --scripts "systemctl status sshd"`.

**Prevent →** No SSH from the internet; Bastion or JIT access.

</details>

### Q42. `nslookup kv-shop.vault.azure.net` from inside the VNet returns a public IP even though a Private Endpoint exists. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `privatelink.vaultcore.azure.net` Private DNS zone is missing, not linked to the VNet the client uses, missing the A record, or the VNet uses custom DNS that doesn't forward to Azure DNS.

**Diagnose →**

```bash
az network private-dns zone list -o table
az network private-dns link vnet list -g rg-dns -z privatelink.vaultcore.azure.net -o table
az network private-dns record-set a list -g rg-dns -z privatelink.vaultcore.azure.net -o table
az network vnet show -g rg -n vnet-shop --query dhcpOptions.dnsServers
```

**Fix →** Create the zone and DNS zone group on the endpoint, link the zone to the VNet (or the hub resolving VNet), or configure the custom DNS server to forward to `168.63.129.16` / the DNS Private Resolver.

**Prevent →** Policy-driven DNS zone group creation and central zones.

</details>

### Q43. Key Vault returns `Client address is not authorized and caller is not a trusted service.` (ForbiddenByFirewall). 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Vault firewall is on (default action Deny or public access disabled), and the caller comes from a public IP not in the allow list or not through the Private Endpoint.

**Diagnose →** The error includes the client IP; `az keyvault show -n kv-shop --query properties.networkAcls`

**Fix →**

1. Access via the Private Endpoint (fix DNS so the client uses it)
2. Or add the caller's IP/VNet rule: `az keyvault network-rule add -n kv-shop --ip-address <ip>/32`

**Prevent →** Private access by design; avoid IP allow lists for CI (hosted runner IPs change).

</details>

### Q44. Application Gateway returns `502 Bad Gateway` and backend health shows Unhealthy. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Probe path returns non-2xx/3xx, NSG on backend blocks App Gateway subnet, TLS mismatch (backend cert CN doesn't match host name), wrong port, or backend actually down.

**Diagnose →**

```bash
az network application-gateway show-backend-health -g rg -n agw-shop \
  --query "backendAddressPools[].backendHttpSettingsCollection[].servers[].{ip:address, health:health, log:healthProbeLog}" -o table
```

**Fix →**

1. Point the custom probe at a real health endpoint with correct host header
2. Allow traffic from the App Gateway subnet on the backend port
3. For HTTPS backends, set the correct host name or trusted root certificate

**Prevent →** Dedicated `/healthz` endpoints and probes in IaC.

</details>

### Q45. App Gateway v2 deployment fails or goes unhealthy after a UDR `0.0.0.0/0 → firewall` was added to its subnet. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** App Gateway v2 (public frontend) requires direct internet for control-plane traffic; sending default route to an NVA breaks it.

**Diagnose →** Check route table on `snet-agw`; App Gateway operational state and health.

**Fix →**

1. Remove the `0.0.0.0/0` route from the App Gateway subnet (keep specific routes to spokes if needed)
2. Or use the private-only App Gateway deployment, which supports forced tunnelling

**Prevent →** Exclude `snet-agw` from the "force-tunnel all subnets" policy.

</details>

### Q46. Two VNets peered but VMs can't communicate; peering state shows `Initiated`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Peering created on one side only.

**Diagnose →** `az network vnet peering list -g rg --vnet-name vnet-a -o table` shows `PeeringState: Initiated`.

**Fix →**

```bash
az network vnet peering create -g rg-b -n b-to-a --vnet-name vnet-b \
  --remote-vnet <vnetAId> --allow-vnet-access
```

State should become `Connected` on both sides.

**Prevent →** Create both sides in the same IaC module.

</details>

### Q47. Peering creation fails because address spaces overlap. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both VNets use overlapping CIDRs (often both `10.0.0.0/16` defaults).

**Diagnose →** Compare `addressSpace` of both VNets.

**Fix →**

1. Re-IP one VNet (add a new address space, migrate resources, remove old)
2. Or use Private Link Service / NAT-based connectivity if re-IP is impossible

**Prevent →** IPAM from day one; never use portal default ranges.

</details>

### Q48. VMs in a spoke lost internet access right after the network team added a route table. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** UDR `0.0.0.0/0` → firewall, but the firewall has no rule allowing the traffic (or the next-hop IP is wrong).

**Diagnose →**

```bash
az network watcher show-next-hop -g rg --vm vm-app --source-ip 10.21.2.4 --dest-ip 1.1.1.1
```

Check firewall logs (`AZFWApplicationRule`, `AZFWNetworkRule`) for denies.

**Fix →** Add firewall rules for required destinations, or correct the next-hop IP.

**Prevent →** Change routing and firewall rules together in one change with tests.

</details>

### Q49. Azure Firewall logs show allowed DNAT but clients still time out to the service behind it. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Asymmetric routing: return traffic from the backend goes directly to the internet (or via a different path) instead of back through the firewall.

**Diagnose →** Effective routes on the backend subnet: is there a route for the client's return path to the firewall?

**Fix →**

1. Ensure the backend subnet's default route points to the firewall (so return traffic goes back through it)
2. If the backend uses a public LB, don't mix with DNAT on the firewall

**Prevent →** Draw the full in/out path when designing NVAs.

</details>

### Q50. Front Door returns `Our services aren't available right now` (error 503 / origin errors). 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Origin health probes failing, origin rejecting the Host header, TLS certificate name mismatch on the origin, origin lock-down blocking Front Door IPs.

**Diagnose →** Front Door health probe metrics and access logs (`FrontDoorAccessLog` / `AzureDiagnostics` with `errorInfo_s`); curl the origin with `-H "Host: ..."`.

**Fix →**

1. Correct origin host header and probe path
2. Allow `AzureFrontDoor.Backend` service tag on the origin NSG
3. Use a valid certificate matching the origin host name

**Prevent →** Synthetic availability tests against both Front Door and origin.

</details>

### Q51. Outbound calls from AKS intermittently fail with timeouts under load. Metrics show SNAT port usage at 100%. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SNAT port exhaustion on the Standard LB outbound rule (default allocation per node is limited), especially with high connection churn to few destinations.

**Diagnose →** Load balancer metrics "SNAT Connection Count" with failed state, "Used SNAT Ports"; AKS diagnostics SNAT detector.

**Fix →**

1. Switch to NAT Gateway outbound (`az aks update --outbound-type managedNATGateway` where supported)
2. Or increase outbound IPs/ports: `az aks update --load-balancer-managed-outbound-ip-count 4 --load-balancer-outbound-ports 4000`
3. Fix connection reuse in apps

**Prevent →** NAT Gateway by default for egress-heavy clusters.

</details>

### Q52. App Service can't reach a database through its Private Endpoint even though VNet integration is configured. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** App Service isn't using the private DNS zone (needs zone linked to the integration VNet or custom DNS), route all disabled, or NSG on PE subnet/DB firewall blocks it.

**Diagnose →** From Kudu console: `nameresolver sql-shop.database.windows.net` and `tcpping sql-shop.database.windows.net:1433`.

**Fix →**

1. Link `privatelink.database.windows.net` to the integration VNet (or set `WEBSITE_DNS_SERVER` to your resolver)
2. Enable outbound internet traffic through the VNet (route all)
3. Allow the integration subnet in NSGs

**Prevent →** Standard App Service network template that includes DNS linking.

</details>

### Q53. Pods resolve external names slowly and some lookups fail with `i/o timeout`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CoreDNS overloaded or throttled, conntrack race on UDP DNS, custom VNet DNS servers slow/unreachable, or `ndots:5` causing many search-domain lookups.

**Diagnose →**

```bash
kubectl -n kube-system top pods -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns --tail 50
kubectl run dnstest --rm -it --image=busybox:1.36 -- nslookup api.github.com
```

**Fix →**

1. Scale CoreDNS (autoscaler config in `coredns-autoscaler` configmap) or enable NodeLocal DNSCache-style caching where supported
2. Set `dnsConfig.options: ndots: "2"` for apps calling many external names
3. Fix custom DNS server reachability or forwarders

**Prevent →** Monitor CoreDNS latency and errors in Prometheus.

</details>

### Q54. VPN site-to-site tunnel is `Connected`, but no traffic flows from on-prem to Azure. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Local network gateway address prefixes missing on-prem ranges, on-prem routing not pointing Azure ranges to the tunnel, NSGs blocking, or traffic selectors mismatch with policy-based devices.

**Diagnose →**

```bash
az network vnet-gateway list-learned-routes -g rg-hub -n vgw-hub -o table
az network local-gateway show -g rg-hub -n lgw-office --query localNetworkAddressSpace
```

**Fix →** Correct prefixes or BGP config; enable `usePolicyBasedTrafficSelectors` for policy-based devices; allow on-prem ranges in NSGs.

**Prevent →** Use BGP to avoid static prefix drift.

</details>

### Q55. Private Endpoint connection shows `Pending` and clients can't connect. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The target resource is in another subscription/tenant or you lack approval rights; manual approval required.

**Diagnose →** `az network private-endpoint-connection list --id <resourceId> -o table`

**Fix →**

```bash
az network private-endpoint-connection approve --id <connectionId> --description "Approved for shop"
```

**Prevent →** Grant `privateEndpointConnectionsApproval/action` to the requester's identity for auto-approval within your org.

</details>

### Q56. Storage account calls from a VM fail with `AuthorizationFailure: This request is not authorized to perform this operation.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Storage firewall: default action Deny, VM's subnet not allowed (no service endpoint or PE), or public access disabled and client not using the private endpoint.

**Diagnose →** `az storage account show -n stshop --query networkRuleSet`; check DNS resolution of `stshop.blob.core.windows.net` from the VM.

**Fix →** Add a VNet rule (with `Microsoft.Storage` service endpoint on the subnet) or use Private Endpoint and fix DNS.

**Prevent →** Distinguish network errors (`AuthorizationFailure`) from RBAC errors (`AuthorizationPermissionMismatch`) in runbooks.

</details>

### Q57. Internal Load Balancer works from other VNets but not from VMs in its own backend pool. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Hairpinning is not supported: a backend VM calling the ILB frontend can be load-balanced to itself and the flow fails.

**Diagnose →** Calls from non-backend VMs succeed; from backend VMs fail intermittently.

**Fix →** Call the service locally or via another tier, or put a proxy in front; for AKS, call the ClusterIP Service instead of the ILB.

**Prevent →** Know the ILB hairpin limitation when designing.

</details>

### Q58. A VM can't reach `169.254.169.254` (IMDS) and managed identity calls fail. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A proxy setting intercepts it (missing `NO_PROXY=169.254.169.254`), guest firewall, or a custom route in the guest OS.

**Diagnose →**

```bash
curl -s -H Metadata:true --noproxy "*" "http://169.254.169.254/metadata/instance?api-version=2021-02-01" | head -c 200
env | grep -i proxy
```

**Fix →** Add IMDS to `NO_PROXY`, fix guest firewall/routes.

**Prevent →** Include IMDS exclusions in proxy configuration standards.

</details>

### Q59. `Connection refused` when an App Service app calls an API on a VM in a peered VNet over port 8080. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Refused (not timeout) means the packet reached the host and nothing listens: wrong port, app bound to `127.0.0.1`, service down.

**Diagnose →** On the VM: `ss -tlnp | grep 8080`.

**Fix →** Bind the app to `0.0.0.0:8080`, start the service, correct the port.

**Prevent →** Readiness checks and documented ports.

</details>

### Q60. A Container Apps app with internal ingress can't be reached from a peered VNet. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No private DNS for the environment's default domain, so `<app>.<env-domain>` doesn't resolve to the environment's static IP from other VNets.

**Diagnose →** `az containerapp env show -n cae-shop -g rg --query "{domain:properties.defaultDomain, ip:properties.staticIp}"`; `nslookup` from the peered VNet.

**Fix →** Create a Private DNS zone for the default domain with a wildcard `*` A record to the static IP, linked to the peered VNets.

**Prevent →** Include this zone in the Container Apps landing template.

</details>

## AKS and ACR Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q61. Pods show `ImagePullBackOff` with `failed to authorize: failed to fetch anonymous token: unexpected status from GET request to https://acrshop.azurecr.io/oauth2/token?...: 401 Unauthorized`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The kubelet identity has no `AcrPull` on the registry (ACR not attached, attached to a different cluster, or role assigned to the cluster identity instead of the kubelet identity).

**Diagnose →**

```bash
az aks check-acr -g rg -n aks --acr acrshop.azurecr.io
KUBELET=$(az aks show -g rg -n aks --query identityProfile.kubeletidentity.objectId -o tsv)
az role assignment list --assignee $KUBELET --all -o table
```

**Fix →**

```bash
az aks update -g rg -n aks --attach-acr acrshop
```

**Prevent →** Manage the `AcrPull` role assignment for the kubelet identity in IaC.

</details>

### Q62. `Failed to pull image "acrshop.azurecr.io/shop/web:v3": ... manifest unknown` or `not found`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tag doesn't exist (typo, push failed, retention policy purged it), wrong repository path.

**Diagnose →**

```bash
az acr repository show-tags -n acrshop --repository shop/web -o table
az acr manifest list-metadata -r acrshop -n shop/web -o table
```

**Fix →** Push the image or correct the tag in the manifest.

**Prevent →** Deploy by digest from the pipeline output; purge policies that keep deployed tags.

</details>

### Q63. `ImagePullBackOff` with `dial tcp: lookup acrshop.azurecr.io on 168.63.129.16:53: no such host` or connection timeouts to the registry. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ACR has public access disabled and the cluster can't resolve the private endpoint (`privatelink.azurecr.io` zone not linked), or firewall egress blocks `*.azurecr.io` and the data endpoint `<registry>.<region>.data.azurecr.io`.

**Diagnose →**

```bash
az aks check-acr -g rg -n aks --acr acrshop.azurecr.io
kubectl run t --rm -it --image=busybox:1.36 -- nslookup acrshop.azurecr.io
```

**Fix →** Link the `privatelink.azurecr.io` zone to the AKS VNet, ensure the PE has DNS records for both login and data endpoints, or allow ACR FQDNs on the firewall.

**Prevent →** Include ACR in the private DNS and firewall baseline for AKS spokes.

</details>

### Q64. `exec /usr/local/bin/python: exec format error` in pod logs right after deploying a new image. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Image built for `arm64` (on an Apple Silicon Mac) running on `amd64` nodes, or the reverse.

**Diagnose →** `docker buildx imagetools inspect acrshop.azurecr.io/shop/web:v3` shows the platforms.

**Fix →** Build for the node architecture: `docker buildx build --platform linux/amd64 -t ... --push .` or use `az acr build` (amd64 by default, `--platform linux/arm64` for Arm pools).

**Prevent →** Build images in CI, not on laptops.

</details>

### Q65. Pods are `Pending`: `0/3 nodes are available: 3 Insufficient cpu. preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Requests exceed free allocatable capacity, and the autoscaler is off, at max, or can't scale (quota).

**Diagnose →**

```bash
kubectl describe pod <pod> | sed -n '/Events/,$p'
kubectl describe nodes | grep -A5 "Allocated resources"
az aks nodepool show -g rg --cluster-name aks -n apps --query "{auto:enableAutoScaling, min:minCount, max:maxCount, count:count}"
```

**Fix →** Enable/raise autoscaler max, right-size requests, add a pool; check quota if scale-out fails.

**Prevent →** Realistic requests and autoscaler headroom alerts.

</details>

### Q66. Cluster autoscaler events show `pod didn't trigger scale-up: 1 max node group size reached` or `NotTriggerScaleUp`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pool at max count, the pod's constraints (node selector, taints, zone with an LRS disk) don't match any pool, or the requested size doesn't fit any node size.

**Diagnose →**

```bash
kubectl get events -A --field-selector reason=NotTriggerScaleUp
kubectl -n kube-system get configmap cluster-autoscaler-status -o yaml
```

**Fix →** Raise max count, add a pool matching the pod's constraints, or adjust requests/affinity.

**Prevent →** Alert when pools hit max; review constraints in manifests.

</details>

### Q67. AKS upgrade fails: `(UpgradeFailed) Drain node aks-apps-12345-vmss000002 failed when evicting pod web-7d9... failed with Too Many Requests error. This is often caused by a restrictive Pod Disruption Budget (PDB) policy.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A PDB with `maxUnavailable: 0` or `minAvailable` equal to replicas, or a single-replica deployment with a PDB.

**Diagnose →**

```bash
kubectl get pdb -A
kubectl get pdb web -n shop -o yaml
```

**Fix →**

1. Scale the deployment up or relax the PDB to allow 1 disruption
2. Re-run: `az aks nodepool upgrade ...` (it resumes)
3. Or use undrainable node behaviour (`--undrainable-node-behavior Cordon`) to quarantine blocked nodes and continue

**Prevent →** Policy that PDBs must allow at least one disruption; at least 2 replicas for PDB-protected apps.

</details>

### Q68. `(InsufficientSubnetSize) Pre-allocated IPs 682 exceeds IPs available 251 in Subnet Cidr 10.20.1.0/24` when scaling or upgrading AKS. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Flat Azure CNI reserves `maxPods + 1` IPs per node including surge nodes, and the subnet is too small.

**Diagnose →** `az aks nodepool show ... --query "{count:count, maxPods:maxPods, surge:upgradeSettings.maxSurge}"`; compute `(count + surge) × (maxPods + 1)`.

**Fix →**

1. Lower max surge temporarily or use `--max-unavailable`
2. Add a new pool on a new bigger subnet and migrate
3. Long term: migrate to Azure CNI Overlay

**Prevent →** IP math during design; Overlay by default.

</details>

### Q69. Service of type LoadBalancer stuck with `EXTERNAL-IP <pending>`; events show `Error syncing load balancer: failed to ensure load balancer: ... AuthorizationFailed ... 'Microsoft.Network/virtualNetworks/subnets/read'`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The cluster's control-plane identity lacks rights on a custom VNet subnet (in another RG), or on the RG of a pre-created public IP.

**Diagnose →**

```bash
kubectl describe svc web -n shop | sed -n '/Events/,$p'
az aks show -g rg -n aks --query identity.principalId -o tsv
```

**Fix →** Grant `Network Contributor` on the subnet (or VNet) to the cluster identity; for static IPs in another RG, grant it on that RG and use the `azure-load-balancer-resource-group` annotation.

**Prevent →** Include cluster identity network roles in the cluster IaC.

</details>

### Q70. Pods with workload identity fail: `WorkloadIdentityCredential authentication failed ... AADSTS70021: No matching federated identity record found for presented assertion. Assertion Subject: 'system:serviceaccount:shop:default'.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The pod uses the `default` ServiceAccount instead of the federated one, or the namespace/SA name differs from the credential subject.

**Diagnose →**

```bash
kubectl get pod <pod> -n shop -o jsonpath='{.spec.serviceAccountName}'
az identity federated-credential list --identity-name id-orders -g rg --query "[].subject"
```

**Fix →** Set `serviceAccountName: orders-sa` in the pod spec, or create a federated credential for the actual subject.

**Prevent →** Helm chart values that tie SA name and federation subject together.

</details>

### Q71. Workload identity env vars (`AZURE_CLIENT_ID`, `AZURE_FEDERATED_TOKEN_FILE`) are missing in the pod. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pod lacks label `azure.workload.identity/use: "true"` (it's a pod label, not a SA annotation, on current versions), workload identity not enabled on the cluster, or the webhook isn't running.

**Diagnose →**

```bash
az aks show -g rg -n aks --query "{oidc:oidcIssuerProfile.enabled, wi:securityProfile.workloadIdentity.enabled}"
kubectl get pod <pod> -n shop --show-labels
kubectl -n kube-system get pods | grep azure-wi-webhook
```

**Fix →** Add the label to the pod template, enable the feature (`az aks update --enable-oidc-issuer --enable-workload-identity`), restart pods.

**Prevent →** Template the label into base Helm charts.

</details>

### Q72. Pod stuck in `ContainerCreating` with `MountVolume.SetUp failed for volume "secrets-store-inline" : rpc error: code = Unknown desc = failed to mount secrets store objects for pod shop/web-..., err: ... failed to get objectType:secret, objectName:db-password ... StatusCode=403`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The identity in the SecretProviderClass lacks `Key Vault Secrets User`, the vault firewall blocks the cluster, or the wrong `clientID`/`tenantId`.

**Diagnose →**

```bash
kubectl describe pod <pod> -n shop | sed -n '/Events/,$p'
kubectl -n kube-system logs -l app=secrets-store-provider-azure --tail 50
kubectl get secretproviderclass kv-shop -n shop -o yaml
```

**Fix →** Assign the role on the vault to the identity in `clientID`, allow network access (PE + DNS), correct IDs.

**Prevent →** Deploy identity, role and SecretProviderClass from the same IaC/Helm release.

</details>

### Q73. SecretProviderClass mount fails with `keyvault.BaseClient#GetSecret: Failure responding to request: StatusCode=404 ... SecretNotFound`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Secret name typo, wrong vault, or the secret was deleted (soft-deleted).

**Diagnose →** `az keyvault secret list --vault-name kv-shop -o table`; `az keyvault secret list-deleted --vault-name kv-shop`

**Fix →** Correct `objectName`/`keyvaultName`, or recover the secret: `az keyvault secret recover --vault-name kv-shop -n db-password`.

**Prevent →** Pipeline check that every object referenced by SecretProviderClasses exists.

</details>

### Q74. Rotated a Key Vault secret, but the app still uses the old value. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CSI rotation not enabled, or the app reads the value from an env var (synced Kubernetes Secret) only at startup.

**Diagnose →** `az aks show -g rg -n aks --query addonProfiles.azureKeyvaultSecretsProvider.config`; compare the mounted file contents with Key Vault.

**Fix →**

1. `az aks addon update -g rg -n aks -a azure-keyvault-secrets-provider --enable-secret-rotation --rotation-poll-interval 2m`
2. Read from the mounted file on each use, or trigger a rollout restart (Reloader or a pipeline step)

**Prevent →** Design apps to reload secrets from files.

</details>

### Q75. `az aks get-credentials` works but `kubectl` hangs then fails with `Unable to connect to the server: dial tcp 10.x.x.x:443: i/o timeout`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Private cluster; your machine has no network path to the private API endpoint (or DNS resolves to the private IP from outside).

**Diagnose →** `az aks show -g rg -n aks --query apiServerAccessProfile`

**Fix →** Connect from a VNet-connected machine (VPN, Bastion jump box, private agent) or use `az aks command invoke -g rg -n aks --command "kubectl get nodes"`.

**Prevent →** Document access paths for private clusters; private CI agents.

</details>

### Q76. `kubectl` returns `Unable to connect to the server: dial tcp 20.x.x.x:443: i/o timeout` on a public cluster from a new office network. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** API server authorized IP ranges don't include the new egress IP.

**Diagnose →** `az aks show -g rg -n aks --query apiServerAccessProfile.authorizedIpRanges`; `curl ifconfig.me`.

**Fix →** `az aks update -g rg -n aks --api-server-authorized-ip-ranges <existing>,<newIP>/32`

**Prevent →** Use stable egress IPs (VPN or NAT) for admin access.

</details>

### Q77. Nodes show `NotReady` after someone modified the node resource group (`MC_...`). 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Manual changes in the node RG: deleted NSG rules or route table entries, changed VMSS settings, locks, or policy denying AKS operations there.

**Diagnose →** Activity Log on `MC_` RG for recent changes; `kubectl describe node`; AKS Diagnose and Solve.

**Fix →**

1. Revert the changes
2. Reconcile the cluster: `az aks update -g rg -n aks` (a no-op update triggers reconciliation)
3. Reimage problematic nodes: `az aks nodepool upgrade --node-image-only` or delete the VMSS instance to get a fresh node

**Prevent →** Node resource group lockdown (`--nrg-lockdown-restriction-level ReadOnly`).

</details>

### Q78. A node pool upgrade fails with `QuotaExceeded` for surge nodes even though the cluster itself fits. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Surge nodes need extra vCPU quota temporarily.

**Diagnose →** Error shows required additional cores; `az vm list-usage`.

**Fix →** Lower `--max-surge` (for example `1`) or use `--max-unavailable 1`, or raise quota.

**Prevent →** Account for surge in quota planning.

</details>

### Q79. Pods are `OOMKilled` repeatedly (`Last State: Terminated, Reason: OOMKilled, Exit Code: 137`). 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Memory limit too low, memory leak, or JVM/.NET heap not container-aware.

**Diagnose →**

```bash
kubectl describe pod <pod> -n shop | grep -A3 "Last State"
kubectl top pod -n shop
```

Container Insights memory working set charts over time.

**Fix →** Raise limits to observed peak plus headroom, set runtime heap flags (`-XX:MaxRAMPercentage=75`), fix leaks.

**Prevent →** Alerts on OOMKilled events; VPA recommendations.

</details>

### Q80. After enabling Entra integration and disabling local accounts, the CI pipeline fails with `error: You must be logged in to the server (Unauthorized)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CI was using `az aks get-credentials --admin` (local admin cert), which no longer works, or kubeconfig still uses interactive kubelogin.

**Diagnose →** Check the pipeline's credential step and kubeconfig `exec` args.

**Fix →**

```bash
az aks get-credentials -g rg -n aks --overwrite-existing
kubelogin convert-kubeconfig -l azurecli
```

Grant the CI identity an AKS RBAC role.

**Prevent →** Use Entra-based CI auth before disabling local accounts.

</details>

### Q81. Pods can't reach each other across namespaces after enabling network policy. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A default-deny policy without explicit allow rules for the needed cross-namespace flows (and DNS egress).

**Diagnose →** `kubectl get networkpolicy -A`; test with `kubectl exec ... -- wget -qO- http://api.backend.svc.cluster.local`; Hubble flows if ACNS is on.

**Fix →** Add policies allowing ingress from the caller namespace (`namespaceSelector` with `kubernetes.io/metadata.name`) and egress to kube-dns on 53.

**Prevent →** Template baseline policies (allow DNS, allow ingress controller) with default-deny.

</details>

### Q82. `az acr build` fails with `denied: client with IP '20.x.x.x' is not allowed access`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ACR firewall/public access restrictions; ACR Tasks agents run outside your network.

**Diagnose →** `az acr show -n acrshop --query "{public:publicNetworkAccess, rules:networkRuleSet, bypass:networkRuleBypassOptions}"`

**Fix →**

1. Allow trusted services (`--allow-trusted-services true`) and run tasks with a system identity, or use a dedicated **agent pool** in your VNet (`az acr agentpool create --subnet-id ...`, Premium)
2. Or build on private agents with `docker build` and push via the private endpoint

**Prevent →** Decide build location when locking down registries.

</details>

### Q83. `docker push` fails with `unauthorized: authentication required` after `az acr login` succeeded an hour ago. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ACR refresh token from `az acr login` expired (about 3 hours), or the identity has only `AcrPull`.

**Diagnose →** Re-run `az acr login -n acrshop`; check roles: `az role assignment list --assignee <id> --scope <acrId>`.

**Fix →** Log in again right before pushing; grant `AcrPush` for push.

**Prevent →** Log in within the same job step sequence as the push.

</details>

### Q84. Node pool stuck in `Failed` provisioning state after a scale operation. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A previous operation failed (quota, allocation, policy blocking VMSS changes, extension failure) and left the pool in Failed.

**Diagnose →** `az aks nodepool show ... --query provisioningState`; Activity Log error; AKS diagnostics.

**Fix →**

1. Fix the cause (quota, policy exemption for the node RG)
2. Re-run a no-op update to reconcile: `az aks nodepool update -g rg --cluster-name aks -n apps`
3. If still failed, scale to the previous count or reimage

**Prevent →** Avoid policies that target node RG resources; monitor operation failures.

</details>

### Q85. Container Insights shows no data for a new cluster. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Monitoring add-on not enabled or DCR/DCR association missing (common with Terraform when `msi_auth_for_monitoring_enabled = true` but no DCR), `ama-logs` pods failing, or network blocked to Azure Monitor endpoints.

**Diagnose →**

```bash
kubectl -n kube-system get pods | grep ama-logs
kubectl -n kube-system logs -l rsName=ama-logs-rs --tail 50
az monitor data-collection rule association list --resource <aksId> -o table
```

**Fix →** Create the DCR with `Microsoft-ContainerInsights-Group-Default` stream and associate it with the cluster; allow egress to `*.ods.opinsights.azure.com`, `*.ingest.monitor.azure.com` or use AMPLS.

**Prevent →** Include DCR + association in the AKS module.

</details>

## Storage, Key Vault, Monitoring and Pipeline Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q86. `AuthorizationPermissionMismatch: This request is not authorized to perform this operation using this permission.` reading blobs with `--auth-mode login`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The identity has control-plane roles (Owner/Contributor) but no Storage Blob Data role.

**Diagnose →** `az role assignment list --assignee <id> --scope <storageId> --include-inherited -o table`

**Fix →** Grant `Storage Blob Data Reader` or `Contributor` on the account or container; wait for propagation.

**Prevent →** Teach the control vs data plane difference; include data roles in IaC.

</details>

### Q87. A SAS URL returns `AuthenticationFailed: Server failed to authenticate the request ... Signature not valid in the specified time frame`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SAS expired, start time in the future (clock skew), or the user delegation key expired.

**Diagnose →** Inspect `st` and `se` parameters in the URL; compare with UTC now.

**Fix →** Generate a new SAS with start time a few minutes in the past and appropriate expiry.

**Prevent →** Short-lived SAS generated on demand by a service, not long-lived ones pasted into config.

</details>

### Q88. `PublicAccessNotPermitted: Public access is not permitted on this storage account.` when setting a container to public. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `allowBlobPublicAccess` is false on the account (often enforced by policy).

**Diagnose →** `az storage account show -n stshop --query allowBlobPublicAccess`

**Fix →** Don't make it public: serve through Front Door/CDN with private origin or use SAS. Only enable public access if it's truly intended and allowed.

**Prevent →** Keep the policy; document the CDN pattern for static content.

</details>

### Q89. `KeyBasedAuthenticationNotPermitted: Key based authentication is not permitted on this storage account.` in an older tool. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Shared key access disabled; the tool uses account keys or a service SAS.

**Diagnose →** `az storage account show -n stshop --query allowSharedKeyAccess`

**Fix →** Configure the tool for Entra auth (`--auth-mode login`, `DefaultAzureCredential`, AzCopy with `azcopy login --identity`), or user delegation SAS.

**Prevent →** Inventory key-based consumers before disabling shared keys.

</details>

### Q90. Application Insights shows no telemetry after deploying an app. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing or wrong `APPLICATIONINSIGHTS_CONNECTION_STRING`, SDK/distro not initialised early, egress blocked to ingestion endpoints, or local auth disabled on App Insights without Entra auth configured.

**Diagnose →** Check app settings/env; app logs for exporter errors; Live Metrics.

**Fix →** Set the connection string, initialise OpenTelemetry distro at startup, allow egress or AMPLS, configure managed identity auth if local auth is disabled.

**Prevent →** Health check in pipeline that queries `AppRequests` after deploy.

</details>

### Q91. A log search alert didn't fire during an incident even though the errors are visible in logs now. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Ingestion latency longer than the evaluation window, the query's time range too short, the threshold/aggregation wrong, or the alert disabled/muted by an alert processing rule.

**Diagnose →** Alert rule history; run the query with the same time range; check `ingestion_time() - TimeGenerated`.

**Fix →** Lengthen the window and frequency appropriately, fix the query, review processing rules.

**Prevent →** Use metric alerts for paging signals; test alerts with fault injection.

</details>

### Q92. Log Analytics costs spiked overnight. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A chatty deployment (debug logs), a new diagnostic setting on a verbose resource, a crash loop flooding logs.

**Diagnose →**

```kusto
Usage
| where TimeGenerated > ago(3d) and IsBillable
| summarize GB = sum(Quantity) / 1000 by DataType, bin(TimeGenerated, 1h)
| render timechart
```

Then `ContainerLogV2 | summarize count() by PodNamespace, ContainerName | top 10 by count_`.

**Fix →** Fix the noisy app, filter in the DCR, move table to Basic.

**Prevent →** Daily cap in non-prod, ingestion anomaly alerts.

</details>

### Q93. Azure DevOps: `##[error]No hosted parallelism has been purchased or granted.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** New organisation without free Microsoft-hosted parallel job grant, or private projects without purchased parallel jobs.

**Diagnose →** Organization settings → Parallel jobs.

**Fix →** Request the free grant via Microsoft's form, buy parallel jobs, or use a self-hosted agent pool.

**Prevent →** Plan agent capacity when creating new orgs.

</details>

### Q94. Azure DevOps: `There was a resource authorization issue: "The pipeline is not valid. Job Deploy: Step AzureCLI input connectedServiceNameARM references service connection arm-shop-prod which could not be found. The service connection does not exist, has been disabled or has not been authorized for use."` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pipeline not authorized to use the service connection, wrong name, or connection in another project.

**Diagnose →** Project settings → Service connections → the connection's Security tab and pipeline permissions.

**Fix →** Authorize the pipeline on the connection (click "Authorize resources" on the run, or add the pipeline in pipeline permissions), correct the name.

**Prevent →** Grant per pipeline rather than "all pipelines"; keep names consistent.

</details>

### Q95. An Azure Pipelines deployment job waits forever at `Waiting for approvals and checks`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pending approval, branch control check failing (run from a non-allowed branch), business hours check, or exclusive lock held by another run.

**Diagnose →** Open the run's checks panel; it lists each check status.

**Fix →** Approve, run from the allowed branch, or cancel the run holding the lock.

**Prevent →** Clear approval owners and timeouts on environment checks.

</details>

### Q96. `azure/webapps-deploy` fails with `Deployment Failed, Error: Failed to deploy web package to App Service. Conflict (CODE: 409)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another deployment in progress on the same app/slot, or the app is configured for Run From Package with a locked file system.

**Diagnose →** Kudu deployment logs (`https://<app>.scm.azurewebsites.net/api/deployments`).

**Fix →** Serialise deployments (concurrency group), retry; ensure consistent `WEBSITE_RUN_FROM_PACKAGE` settings.

**Prevent →** One deployment at a time per slot.

</details>

### Q97. App Service container fails: `Container shop-web_0_ab12 didn't respond to HTTP pings on port: 80, failing site start.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** App listens on another port (8080, 3000) or on `127.0.0.1`, or startup takes longer than the timeout.

**Diagnose →** `az webapp log tail -g rg -n app-shop`; check Dockerfile `EXPOSE` and app listen address.

**Fix →**

```bash
az webapp config appsettings set -g rg -n app-shop --settings WEBSITES_PORT=8080 WEBSITES_CONTAINER_START_TIME_LIMIT=600
```

Bind to `0.0.0.0`.

**Prevent →** Standardise container ports and health endpoints.

</details>

### Q98. Azure Functions app shows `Azure Functions runtime is unreachable` after locking down the storage account. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Functions host can't reach `AzureWebJobsStorage` (storage firewall or disabled shared key) because the app isn't VNet-integrated or DNS for storage private endpoints (blob, queue, table, file) isn't resolving.

**Diagnose →** Check storage network rules, app's VNet integration, `WEBSITE_CONTENTOVERVNET`/content share settings, and DNS for all four storage sub-services.

**Fix →**

1. VNet integration with route all, Private Endpoints for blob/queue/table (and file if using content share), Private DNS zones linked
2. Use identity-based connection (`AzureWebJobsStorage__accountName`) with `Storage Blob Data Owner`, `Storage Queue Data Contributor` and related roles if shared keys are disabled

**Prevent →** Use the Functions private networking template; Flex Consumption handles identity-based storage cleanly.

</details>

### Q99. AKS metric alert on `node_cpu_usage_percentage` fires constantly for one node pool. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Real saturation (too few nodes, autoscaler max reached), a hot pod without limits, or system pods crowding the pool.

**Diagnose →**

```bash
kubectl top nodes
kubectl top pods -A --sort-by=cpu | head
az aks nodepool show -g rg --cluster-name aks -n apps --query "{count:count, max:maxCount}"
```

**Fix →** Raise autoscaler max, set requests/limits on hot pods, taint system pool, or scale vertically.

**Prevent →** Alert on symptoms (latency, errors) plus capacity headroom, not only CPU.

</details>

### Q100. `az keyvault secret set` fails with `(Forbidden) ... Caller is not authorized ... ForbiddenByRbac` for the engineer who just created the vault with Terraform. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Creating the vault (control plane) gives no data-plane rights on an RBAC vault; the engineer needs a Key Vault data role.

**Diagnose →** `az role assignment list --assignee $(az ad signed-in-user show --query id -o tsv) --scope <kvId>`

**Fix →**

```bash
az role assignment create --role "Key Vault Secrets Officer" \
  --assignee $(az ad signed-in-user show --query id -o tsv) --scope <kvId>
```

Wait a few minutes, then retry.

**Prevent →** Terraform grants `Key Vault Secrets Officer` to `data.azurerm_client_config.current.object_id` when it creates the vault.

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
