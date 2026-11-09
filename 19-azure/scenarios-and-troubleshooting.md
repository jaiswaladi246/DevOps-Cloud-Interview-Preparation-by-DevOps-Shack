# Azure for DevOps — Scenarios & Troubleshooting

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

> Real situations you will face running Azure in production (Q1–Q100) and real error messages with diagnosis and fix (Q101–Q200) · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Azure for DevOps](./README.md) · [🏠 Home](../README.md)

## Table of contents

**Scenarios**
- [Identity, Access and Governance](#identity-access-and-governance) — Q1–Q20
- [Networking Design](#networking-design) — Q21–Q40
- [AKS and Containers](#aks-and-containers) — Q41–Q65
- [CI/CD and IaC](#cicd-and-iac) — Q66–Q85
- [Reliability, Cost and Migration](#reliability-cost-and-migration) — Q86–Q100

**Troubleshooting**
- [Identity and Authorization Errors](#identity-and-authorization-errors) — Q101–Q120
- [Quota, Deployment and IaC Errors](#quota-deployment-and-iac-errors) — Q121–Q140
- [Network Connectivity Errors](#network-connectivity-errors) — Q141–Q160
- [AKS and ACR Errors](#aks-and-acr-errors) — Q161–Q185
- [Storage, Key Vault, Monitoring and Pipeline Errors](#storage-key-vault-monitoring-and-pipeline-errors) — Q186–Q200

---

## Identity, Access and Governance

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. A new developer joins and needs to deploy to the dev resource group only. How do you give access? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Access should come from group membership, at the smallest useful scope, and never touch prod.

**What I'd do →**

1. Add the developer to the existing Entra group `grp-shop-dev-contributors` (create it once if missing)
2. Make sure that group holds Contributor on the dev RG only:
   ```bash
   az role assignment create --assignee-object-id <groupObjectId> --assignee-principal-type Group \
     --role Contributor --scope /subscriptions/<sub>/resourceGroups/rg-shop-dev
   ```
3. Developer runs `az logout && az login`, then `az group show -n rg-shop-dev` to confirm
4. For prod, they get nothing by default; read access comes later through a separate reader group

**Why →** Group-based access survives team changes, keeps the subscription under the role assignment limit, and makes access reviews readable.

**Interview tip →** Say "groups, not users, at the lowest scope" before naming any role.

</details>

### Q2. Your GitHub Actions workflow uses a client secret stored in a repo secret. Security wants it gone. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A long-lived secret in CI is the most common Azure credential leak. Replace it with workload identity federation.

**What I'd do →**

1. Add a federated credential to the existing app registration (or switch to a user-assigned identity):
   ```bash
   az ad app federated-credential create --id <appId> --parameters '{
     "name":"gh-main","issuer":"https://token.actions.githubusercontent.com",
     "subject":"repo:devopsshack/shop:ref:refs/heads/main","audiences":["api://AzureADTokenExchange"]}'
   ```
2. Update the workflow: `permissions: id-token: write, contents: read` and `azure/login@v2` with `client-id`, `tenant-id`, `subscription-id` only
3. Run the workflow and confirm `az account show` works
4. Delete the client secret: `az ad app credential delete --id <appId> --key-id <keyId>` and remove the repo secret

**Why →** Federated tokens live minutes, are bound to a repo and branch or environment, and there is nothing to rotate or leak.

**Interview tip →** Mention binding the subject to a protected GitHub environment for prod.

</details>

### Q3. Auditors ask "who has Owner on production, and why?" How do you answer and fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Standing Owner access is the finding. You need an inventory now and PIM going forward.

**What I'd do →**

1. Inventory, including inherited assignments:
   ```bash
   az role assignment list --role Owner --scope /subscriptions/<prod-sub> --include-inherited \
     --query "[].{who:principalName, type:principalType, scope:scope}" -o table
   ```
2. Expand groups and identify service principals with Owner (pipelines rarely need it)
3. Convert human Owner assignments to PIM-eligible with approval, MFA and a 2–4 hour activation limit
4. Replace pipeline Owner with Contributor plus a constrained `Role Based Access Control Administrator` condition if it must assign roles
5. Schedule quarterly access reviews on the privileged groups

**Why →** PIM gives auditors a justification and approval trail per activation, and drops the attack surface to near zero most of the time.

**Interview tip →** "Zero standing access to prod" is the phrase to use.

</details>

### Q4. A VM application needs to read blobs from one container. How do you set up access without keys? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Managed identity plus a data-plane role scoped to the container.

**What I'd do →**

1. Enable a system-assigned identity: `az vm identity assign -g rg -n vm-app`
2. Grant the data role at container scope:
   ```bash
   az role assignment create --assignee <principalId> --role "Storage Blob Data Reader" \
     --scope "/subscriptions/<sub>/resourceGroups/rg/providers/Microsoft.Storage/storageAccounts/stshop/blobServices/default/containers/reports"
   ```
3. In code use `DefaultAzureCredential`; on the VM test with `az login --identity && az storage blob list --account-name stshop -c reports --auth-mode login`
4. Disable shared key access on the account once nothing uses keys

**Why →** No secret anywhere, access scoped to one container, and access is revoked by removing one role assignment.

**Interview tip →** Point out `--auth-mode login`; without it the CLI tries account keys.

</details>

### Q5. Teams keep creating resources in random regions and without tags. How do you stop it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Governance belongs in Azure Policy at management group scope, not in a wiki.

**What I'd do →**

1. Assign built-in "Allowed locations" (and "Allowed locations for resource groups") with `centralindia` and `southindia` on the landing zones management group
2. Assign "Require a tag on resource groups" for `owner`, `env`, `costcenter`
3. Assign "Inherit a tag from the resource group if missing" (Modify) for each tag, then remediate existing resources:
   ```bash
   az policy remediation create -n inherit-owner --policy-assignment inherit-owner-tag \
     --resource-discovery-mode ReEvaluateCompliance
   ```
4. Start new deny policies in `Audit` for a week, review compliance, then switch to `Deny`

**Why →** Policy enforces at ARM for every tool, so portal clicks, CLI and Terraform are all covered.

**Interview tip →** Rolling out deny policies via audit first shows you care about not breaking teams.

</details>

### Q6. A contractor needs temporary read access to logs for two weeks. How do you handle it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** External, time-boxed, read-only. Use B2B guest access with an expiring assignment.

**What I'd do →**

1. Invite the contractor as a guest (or via an access package in Entra ID Governance with a 14-day expiry)
2. Grant `Log Analytics Reader` on the workspace, or better, a PIM-eligible assignment with an end date
3. Require MFA for guests through Conditional Access
4. Set a calendar check and an access review on the guest's group

**Why →** Time-limited assignments clean themselves up; manual removal is what gets forgotten.

**Interview tip →** Name "access packages" if the company has Entra ID Governance.

</details>

### Q7. You need to let an app team create their own role assignments for their identities, without giving them Owner. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Delegated, constrained role assignment through an ABAC condition.

**What I'd do →**

1. Assign the team group `Role Based Access Control Administrator` on their RG with a condition limiting roles and principal types:
   ```bash
   az role assignment create --assignee-object-id <teamGroupId> --assignee-principal-type Group \
     --role "Role Based Access Control Administrator" --scope <rgId> \
     --condition "((!(ActionMatches{'Microsoft.Authorization/roleAssignments/write'})) OR (@Request[Microsoft.Authorization/roleAssignments:RoleDefinitionId] ForAnyOfAnyValues:GuidEquals {7f951dda-4ed3-4680-a7ca-43fe172d538d, 4633458b-17de-408a-b874-0445c86b69e6}))" \
     --condition-version 2.0
   ```
   (the GUIDs are AcrPull and Key Vault Secrets User)
2. Test that assigning Owner fails and AcrPull succeeds
3. Document the allowed role list in the platform docs

**Why →** Teams self-serve the common grants, and escalation to Owner or Contributor is impossible.

**Interview tip →** Explaining conditional delegation separates senior answers from "just give them Owner".

</details>

### Q8. Someone deleted a production resource group last night. How do you find out who, and prevent it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Activity Log answers who; locks and RBAC prevent the repeat.

**What I'd do →**

1. Query the Activity Log at subscription level (the RG is gone):
   ```bash
   az monitor activity-log list --offset 2d \
     --query "[?operationName.value=='Microsoft.Resources/subscriptions/resourceGroups/delete'].{who:caller, when:eventTimestamp, rg:resourceGroupName}" -o table
   ```
2. Restore from IaC plus data backups (Azure Backup, storage soft delete, SQL long-term retention)
3. Add `CanNotDelete` locks on prod RGs holding state
4. Remove standing write access in prod; move humans to PIM

**Why →** Locks stop accidents by anyone, including Owners, without blocking normal updates.

**Interview tip →** Mention that you rebuilt from code; it proves IaC was real.

</details>

### Q9. You have 40 subscriptions and need consistent diagnostic logging for Key Vaults everywhere. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** DeployIfNotExists policy at management group scope plus remediation.

**What I'd do →**

1. Assign the built-in initiative for resource logging to Log Analytics (category group `audit`) at the top landing-zone management group, parameterised with the central workspace ID
2. Give the assignment a managed identity with `Log Analytics Contributor` and `Monitoring Contributor`
3. Create remediation tasks for existing vaults
4. Check compliance: `az policy state summarize --management-group lz`

**Why →** New vaults in any subscription get logging automatically, and compliance is visible in one dashboard.

**Interview tip →** Call out the managed identity on the assignment; DINE fails silently without its roles.

</details>

### Q10. A partner company's engineers need to manage resources in your tenant regularly. B2B or Lighthouse? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Regular, multi-person, cross-tenant management. Lighthouse fits better than individual guests.

**What I'd do →**

1. Agree on the roles (Contributor on specific RGs, never Owner) and their Entra group in the partner tenant
2. Deploy a Lighthouse registration definition and assignment via ARM/Bicep in our subscription, with eligible (PIM) authorizations for elevated roles
3. Confirm actions appear in our Activity Log with the partner's identities
4. Review delegations quarterly; remove by deleting the registration assignment

**Why →** Partner engineers keep their own identities and MFA, we never manage their accounts, and access is defined as code.

**Interview tip →** Note that Lighthouse cannot delegate Owner, which auditors like.

</details>

### Q11. Your Terraform pipeline needs to create role assignments, but it only has Contributor. What do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Contributor excludes `Microsoft.Authorization/roleAssignments/write`. Give the minimum extra right.

**What I'd do →**

1. Add `Role Based Access Control Administrator` for the pipeline identity at the RG scope, with a condition restricting assignable roles (AcrPull, Key Vault Secrets User, Network Contributor)
2. Do not grant Owner or User Access Administrator unrestricted
3. Re-run the pipeline and check that the `azurerm_role_assignment` resources succeed
4. Record the exception in the platform's RBAC register

**Why →** Pipelines that can grant any role are a privilege escalation path for anyone who can edit the pipeline.

**Interview tip →** Ask "who can change the pipeline?" — that person effectively holds the pipeline's rights.

</details>

### Q12. You must prevent anyone from creating public IPs in corp subscriptions, except the hub firewall. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Deny policy with a scoped exemption.

**What I'd do →**

1. Assign a deny policy on `Microsoft.Network/publicIPAddresses` at the `corp` management group
2. Create a policy exemption for the hub connectivity RG:
   ```bash
   az policy exemption create -n hub-fw-pip --policy-assignment <assignmentId> \
     --exemption-category Waiver --scope /subscriptions/<conn-sub>/resourceGroups/rg-hub \
     --description "Azure Firewall needs a public IP"
   ```
3. Verify with a test `az network public-ip create` in a spoke; expect `RequestDisallowedByPolicy`

**Why →** Exemptions are documented and auditable, unlike carving the hub out with ad hoc exclusions.

**Interview tip →** Mention AKS and App Gateway also want public IPs; design ingress before you deny.

</details>

### Q13. A developer's laptop was stolen. They had `az login` sessions and kubeconfigs. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Revoke sessions centrally, and check whether any static credentials were on the device.

**What I'd do →**

1. Revoke the user's refresh tokens: `az rest --method post --url https://graph.microsoft.com/v1.0/users/<id>/revokeSignInSessions`
2. Reset password, re-register MFA, mark device lost in Intune
3. If AKS clusters still have local accounts, rotate cluster certificates: `az aks rotate-certs -g rg -n aks` (causes API downtime; plan it)
4. Rotate any secrets the user had locally (storage keys, SP secrets)
5. Review sign-in and Activity Logs for activity after the theft time

**Why →** Entra-backed access dies with the session; static credentials (admin kubeconfig, keys) don't. That is the argument for disabling local accounts.

**Interview tip →** Use this to argue for `--disable-local-accounts` on every cluster.

</details>

### Q14. Two teams share a subscription and keep breaking each other's resources. What do you propose? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The subscription is the isolation unit; RGs plus RBAC are a stopgap.

**What I'd do →**

1. Short term: separate RGs per team, Contributor only on their own RG, Reader on the rest
2. Medium term: vend one subscription per team per environment through the landing-zone process
3. Move resources with `az resource move` where supported, or redeploy from IaC
4. Shared things (hub network, DNS) stay in platform subscriptions

**Why →** Separate subscriptions isolate quotas, cost, RBAC and blast radius; RG-level separation still shares limits and subscription-level operations.

**Interview tip →** Say "subscription per application per environment" as the target state.

</details>

### Q15. Security wants all Azure resource changes in prod to go through code. How do you enforce it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Remove human write paths, keep a break-glass path, and detect drift.

**What I'd do →**

1. Give humans Reader in prod; write roles only via PIM with approval for emergencies
2. Pipelines deploy with federated identities from protected branches and environments with approvals
3. For Bicep, use deployment stacks with `--deny-settings-mode denyWriteAndDelete`, excluding the pipeline and break-glass identities
4. Nightly drift detection (`terraform plan -detailed-exitcode` or `what-if`) that opens a ticket on changes
5. Alert on Activity Log writes in prod by any caller that isn't the pipeline identity

**Why →** Enforcement plus detection: PIM handles emergencies, everything else is reviewed in PRs.

**Interview tip →** Always keep a break-glass path in the answer; "nobody can change prod" is not a real design.

</details>

### Q16. A managed identity needs to call Microsoft Graph to read group memberships. How do you grant it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Graph application permissions for managed identities can't be granted in the portal UI; use the Graph API.

**What I'd do →**

1. Find the Graph service principal and the `GroupMember.Read.All` app role ID:
   ```bash
   graphSp=$(az ad sp show --id 00000003-0000-0000-c000-000000000000 --query id -o tsv)
   roleId=$(az ad sp show --id 00000003-0000-0000-c000-000000000000 \
     --query "appRoles[?value=='GroupMember.Read.All'].id" -o tsv)
   ```
2. Create the app role assignment (needs a Privileged Role Administrator or equivalent):
   ```bash
   az rest --method post --url "https://graph.microsoft.com/v1.0/servicePrincipals/<miObjectId>/appRoleAssignments" \
     --body "{\"principalId\":\"<miObjectId>\",\"resourceId\":\"$graphSp\",\"appRoleId\":\"$roleId\"}"
   ```
3. Get a token for `https://graph.microsoft.com` in the app and test

**Why →** Graph rights are directory permissions, not Azure RBAC; Contributor on anything doesn't help.

**Interview tip →** Distinguishing Graph permissions from RBAC is a strong signal.

</details>

### Q17. Finance wants cost per team, but resources aren't tagged consistently. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fix tagging going forward with policy, backfill with remediation, and use RG/subscription structure as a fallback.

**What I'd do →**

1. Define the tag schema (`costcenter`, `owner`, `env`)
2. Require tags on RGs, inherit to resources with Modify policies, and remediate
3. Enable tag inheritance in Cost Management settings so usage records pick up RG tags
4. Build a Cost Management view grouped by `costcenter`, scheduled export for finance
5. Report untagged spend as its own line so owners feel the pressure

**Why →** Tags on RGs plus inheritance is far less work than tagging every resource by hand.

**Interview tip →** Mention shared costs (hub firewall) need an allocation rule, not tags.

</details>

### Q18. A Global Administrator leaves the company. What must be checked beyond disabling the account? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Admins accumulate hidden access: app ownership, secrets, elevated root access, personal subscriptions.

**What I'd do →**

1. Disable the account and revoke sessions
2. Check whether they used "elevate access" (User Access Administrator at root `/`) and remove it: `az role assignment list --scope / --role "User Access Administrator"`
3. Find app registrations they own and credentials they created; rotate those secrets
4. Check subscriptions where they are Account Admin or billing owner; transfer billing ownership
5. Review PIM eligible assignments and Conditional Access exclusions they added

**Why →** Root-scope access and owned apps survive account disablement in practice, because other systems still trust those secrets.

**Interview tip →** "Elevate access at root scope" is the detail most candidates miss.

</details>

### Q19. You want app teams to get compliant AKS clusters without writing Terraform themselves. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Platform engineering problem: golden path module plus self-service.

**What I'd do →**

1. Build an internal Terraform module wrapping AVM's AKS module with our defaults (Overlay + Cilium, Entra RBAC, workload identity, monitoring, policy add-on, maintenance windows)
2. Expose it through a self-service flow: a PR to a "clusters" repo with a small `tfvars` file, or Azure Deployment Environments catalog items
3. CI runs plan, policy checks (Checkov, Azure Policy at deploy), platform approval, apply
4. Flux extension bootstraps each new cluster with baseline platform components

**Why →** Teams get speed, the platform team keeps control, and every cluster is built the same way.

**Interview tip →** Name the golden path idea and the guardrails, not just the tool.

</details>

### Q20. An Azure Policy in Deny mode suddenly blocks an urgent production fix. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Don't delete the assignment. Create a narrow, expiring exemption and fix the root cause.

**What I'd do →**

1. Read the policy assignment ID from the `RequestDisallowedByPolicy` error
2. Create an exemption scoped to the single resource or RG with `--expires-on` a day or two out
3. Ship the fix, then decide whether the policy or the deployment needs changing
4. Feed it back: policy authors add the case to tests; app team adjusts templates

**Why →** Exemptions keep the audit trail and expire on their own; deleted assignments tend to stay deleted.

**Interview tip →** Mention `--expires-on`; it shows you've done this under pressure before.

</details>

---

## Networking Design

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q21. Design the network for a new three-tier app in Central India that must be private except for the web front end. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Spoke VNet peered to the hub, App Gateway WAF as the only public entry, private endpoints for data.

**What I'd do →**

1. Allocate `10.21.0.0/20` from IPAM; subnets: `snet-agw` (/24), `snet-app` (/23), `snet-pe` (/24), `snet-data` if needed
2. Application Gateway v2 with WAF in `snet-agw`, public frontend, backends in `snet-app`
3. App tier on VMSS or AKS in `snet-app`, NSG allowing 443 only from `snet-agw`
4. Database and Key Vault via Private Endpoints in `snet-pe`, Private DNS zones linked from the hub
5. UDR `0.0.0.0/0` to the hub firewall on `snet-app` (not on `snet-agw`, which needs direct internet for its control traffic)

**Why →** One public surface with WAF, east-west and egress inspected, PaaS never exposed publicly.

**Interview tip →** Mention that Application Gateway v2's subnet must not get a default route to an NVA unless you use its private-only deployment.

</details>

### Q22. Spoke A can't talk to spoke B, both peered to the hub. What design change makes it work? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Peering is not transitive. Route through the hub firewall.

**What I'd do →**

1. On each spoke's subnets, add a UDR for the other spoke's CIDR (or a supernet like `10.0.0.0/8`) with next hop the firewall private IP
2. Enable `allowForwardedTraffic` on the hub-to-spoke peerings
3. Add firewall network rules allowing the required flows A→B
4. Verify with `az network watcher show-next-hop` and a test connection

**Why →** The firewall becomes the transit point, and every cross-spoke flow is explicitly allowed and logged.

**Interview tip →** Offer Virtual WAN or AVNM mesh as alternatives when there are many spokes.

</details>

### Q23. Workloads must reach Azure SQL privately, including from on-prem over ExpressRoute. How do you set it up? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Private Endpoint plus hybrid DNS is the whole problem.

**What I'd do →**

1. Create a Private Endpoint for the SQL server in the spoke `snet-pe` with a DNS zone group to `privatelink.database.windows.net` (zone lives in the connectivity subscription, linked to the hub)
2. Deploy DNS Private Resolver in the hub with an inbound endpoint
3. On-prem DNS conditionally forwards `database.windows.net` to the resolver inbound IP
4. Set the SQL server `publicNetworkAccess` to Disabled
5. Test from on-prem: `nslookup sql-shop.database.windows.net` returns `10.x`

**Why →** The endpoint gives the private IP; the DNS chain makes clients use it. Forwarding the public zone (not only privatelink) keeps the CNAME chain intact.

**Interview tip →** Most real outages here are DNS, so lead with the DNS design.

</details>

### Q24. You need all outbound traffic from AKS to go through Azure Firewall. How do you do it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** UDR outbound type with firewall rules for AKS dependencies.

**What I'd do →**

1. Create the route table on the AKS subnet: `0.0.0.0/0` → firewall private IP
2. On the firewall: application rule with FQDN tag `AzureKubernetesService`, plus rules for your ACR, MCR (`mcr.microsoft.com`, `*.data.mcr.microsoft.com`), Ubuntu/Azure Linux update endpoints, monitoring endpoints, and a network rule allowing TCP 443 to the API server (service tag `AzureCloud.<region>`) plus UDP 123 for NTP
3. Create the cluster with `--outbound-type userDefinedRouting --vnet-subnet-id <subnetId>`
4. For inbound to public Services, add a DNAT rule on the firewall (asymmetric routing otherwise) or use an internal LB behind App Gateway

**Why →** Central egress control and logging, with a known egress IP for partners.

**Interview tip →** Mention asymmetric routing for inbound public LBs; it's the classic gotcha.

</details>

### Q25. A partner needs to whitelist a single egress IP from your App Service and AKS workloads. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** NAT Gateway with a static public IP (or firewall egress) on the subnets.

**What I'd do →**

1. Create a public IP prefix or single IP and a NAT Gateway; associate it with the AKS node subnet and App Service integration subnet
2. App Service: enable VNet integration and route all outbound through the VNet
3. AKS: create or update with `--outbound-type userAssignedNATGateway`
4. Confirm with `curl -s ifconfig.me` from a pod and from the App Service Kudu console

**Why →** A NAT Gateway gives a deterministic egress IP and avoids SNAT exhaustion.

**Interview tip →** Mention it is a per-subnet, per-zone resource; plan zone redundancy accordingly.

</details>

### Q26. You're asked to expose an internal API to a customer's Azure tenant without VNet peering. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Private Link Service: publish your service behind a Standard internal LB; they create a Private Endpoint to it.

**What I'd do →**

1. Put the API behind an internal Standard Load Balancer (AKS: Service annotations for internal LB plus `service.beta.kubernetes.io/azure-pls-create: "true"`)
2. Create a Private Link Service on the LB frontend with NAT IPs from a dedicated subnet
3. Share the PLS alias with the customer; set auto-approval for their subscription or approve manually
4. They create a Private Endpoint in their VNet; traffic arrives via NAT IPs

**Why →** No overlapping-CIDR problems, no peering, one-way connectivity, and the customer reaches only that one service.

**Interview tip →** Note that you see the NAT IPs as sources; use PROXY protocol v2 if you need the original client.

</details>

### Q27. Users worldwide hit your app hosted in Central India and complain about latency. What do you put in front? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Global anycast entry with edge caching: Front Door.

**What I'd do →**

1. Create Front Door Standard (Premium if you need Private Link to origin or managed WAF rule sets)
2. Origin group pointing to the App Gateway or App Service; health probe on `/health`
3. Caching rules for static paths, compression on
4. WAF policy attached, custom domain with managed certificate, then DNS cutover
5. Lock the origin to accept only Front Door: `AzureFrontDoor.Backend` service tag plus check the `X-Azure-FDID` header

**Why →** TLS terminates near users, static content comes from the edge, and dynamic traffic rides the Microsoft backbone.

**Interview tip →** Always mention origin lock-down; otherwise attackers bypass the WAF.

</details>

### Q28. You must allow developers SSH to Linux VMs without public IPs or SSH keys. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Bastion plus Entra login extension, RBAC-controlled.

**What I'd do →**

1. Deploy Azure Bastion Standard in the hub (`AzureBastionSubnet`, native client support enabled)
2. Install the extension: `az vm extension set --publisher Microsoft.Azure.ActiveDirectory --name AADSSHLoginForLinux -g rg --vm-name vm-app`
3. Grant `Virtual Machine User Login` (or `Administrator Login`) to the dev group on the RG
4. Connect: `az network bastion ssh -n bas-hub -g rg-hub --target-resource-id <vmId> --auth-type AAD`

**Why →** Access follows Entra groups and Conditional Access, nothing to rotate, nothing exposed.

**Interview tip →** Mention JIT VM access in Defender for Servers as the alternative when Bastion isn't available.

</details>

### Q29. An application needs to call Key Vault, Storage and ACR privately from a spoke. How do you structure Private DNS for many spokes? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Central private DNS zones in the connectivity subscription, linked once to the hub, resolved via the DNS resolver.

**What I'd do →**

1. Create one zone per service type centrally: `privatelink.vaultcore.azure.net`, `privatelink.blob.core.windows.net`, `privatelink.azurecr.io`
2. Link them to the hub VNet; spokes use the hub resolver inbound IP (or the firewall DNS proxy) as their DNS server
3. Use Azure Policy (DeployIfNotExists) to create DNS zone group records automatically when spokes create Private Endpoints
4. Deny creation of `privatelink.*` zones in spokes by policy to avoid split-brain

**Why →** One zone per service tenant-wide avoids duplicate zones that resolve differently in different places.

**Interview tip →** "Duplicate privatelink zones" is the root cause of most enterprise PE DNS incidents.

</details>

### Q30. You need to inspect and filter outbound HTTPS by URL, not just domain. What do you choose? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** URL filtering inside HTTPS needs TLS inspection: Azure Firewall Premium.

**What I'd do →**

1. Deploy Azure Firewall Premium with a Premium Firewall Policy
2. Create an intermediate CA certificate in Key Vault; grant the firewall's managed identity access
3. Distribute the root CA to clients (VMs, AKS node trust stores via image or daemonset)
4. Enable TLS inspection on application rules, then URL-based rules and IDPS
5. Exclude categories you must not inspect (banking, health) and pinned-cert destinations

**Why →** Without decrypting, the firewall only sees SNI and can filter by FQDN, not path.

**Interview tip →** Mention certificate-pinning breakages and privacy exclusions.

</details>

### Q31. Your App Gateway WAF in Prevention mode starts blocking a legitimate checkout POST. How do you handle it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** False positive. Find the rule, add a narrow exclusion, keep Prevention on.

**What I'd do →**

1. Query the firewall log:
   ```kusto
   AzureDiagnostics
   | where Category == "ApplicationGatewayFirewallLog" and action_s == "Blocked"
   | where requestUri_s has "/checkout"
   | project TimeGenerated, ruleId_s, details_message_s, details_data_s
   ```
2. Identify the rule (for example a SQLi rule matching a field `comments`)
3. Add an exclusion on the WAF policy for that request body field and rule, or a per-URI policy for `/checkout`
4. Replay the request; confirm blocks stop and other traffic is still inspected

**Why →** Narrow exclusions preserve protection; switching the whole WAF to Detection removes it.

**Interview tip →** Say you'd never disable an entire rule group for one field.

</details>

### Q32. How would you connect two Azure regions and on-prem with redundancy? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two hubs, ExpressRoute with diverse circuits, VPN as backup, global peering or Virtual WAN between hubs.

**What I'd do →**

1. Hub per region (Central India, South India) each with ExpressRoute gateway connected to two circuits in different peering locations
2. S2S VPN on each hub as a backup path
3. Hub-to-hub global peering with UDRs, or Virtual WAN hubs with transitive routing
4. BGP weights/AS-path prepending so each region prefers its local path
5. Test failover by disabling circuits during a maintenance window

**Why →** Every single failure (circuit, gateway, region) leaves a working path.

**Interview tip →** Mention testing; a backup path that was never exercised often fails.

</details>

### Q33. A team needs Container Apps with private access to a database. How do you set up networking? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Container Apps environment injected into a VNet subnet, internal or external ingress, DB via Private Endpoint.

**What I'd do →**

1. Create a dedicated subnet (`/27` minimum for workload profiles environments) delegated to `Microsoft.App/environments`
2. Create the environment with `--infrastructure-subnet-resource-id` and `--internal-only` if no public ingress is needed
3. Database Private Endpoint in the same VNet, Private DNS zone linked
4. Connect the app with managed identity where the DB supports Entra auth
5. If internal-only, create a Private DNS zone for the environment's default domain pointing to its static IP

**Why →** Workload profiles environments support UDRs and NAT Gateway, so egress control works like the rest of the spoke.

**Interview tip →** Mention the environment's own DNS zone for internal-only setups; it's often forgotten.

</details>

### Q34. A large VNet is running out of IPs because of AKS. What are your options? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pod IPs are eating the VNet. Move pods off VNet IPs.

**What I'd do →**

1. Migrate to Azure CNI Overlay: `az aks update -g rg -n aks --network-plugin-mode overlay --pod-cidr 192.168.0.0/16` (supported upgrade from flat Azure CNI and kubenet; nodes reimage)
2. If staying on VNet IPs, switch to dynamic pod IP allocation with separate pod subnets
3. Add another address space to the VNet (`az network vnet update --address-prefixes`) for new node pools
4. Reduce `maxPods` on pools that don't need 30+

**Why →** Overlay cuts VNet IP use to just nodes, typically freeing thousands of addresses.

**Interview tip →** Mention the IP math, nodes × (maxPods + 1), to show you understand the cause.

</details>

### Q35. You need DNS names like `api.devopsshack.com` to fail over between regions automatically. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** HTTP traffic: Front Door priority routing. Non-HTTP: Traffic Manager priority profile.

**What I'd do →**

1. Front Door origin group with primary (priority 1) and secondary (priority 2) origins, health probes every 30s
2. Custom domain `api.devopsshack.com` CNAME to the Front Door endpoint
3. For TCP services, a Traffic Manager profile with priority routing and endpoint monitoring, CNAME to `*.trafficmanager.net`
4. Test failover by stopping the primary origin and timing recovery

**Why →** Health-probed global routing removes humans from the failover path.

**Interview tip →** Mention DNS TTLs when using Traffic Manager; clients cache answers.

</details>

### Q36. You must block a known malicious IP range from all public endpoints quickly. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Block at the edge first (Front Door or App Gateway WAF custom rule), then NSGs if needed.

**What I'd do →**

1. Add a WAF custom rule with `IPMatch` and action Block at priority 1 on the WAF policy
2. For non-HTTP endpoints, add an NSG deny rule at low priority number or an AVNM security admin rule across VNets
3. Confirm blocks in WAF logs
4. Remove or review the rule after the incident

**Why →** WAF policies apply to every attached listener instantly; editing many NSGs by hand is slow.

**Interview tip →** Mention rate-limit custom rules for the next time.

</details>

### Q37. An on-prem app needs to reach a VM in a spoke, but the connection hangs. Route tables look fine. What's your design checklist? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hybrid connectivity spans gateways, peerings, UDRs, firewall and NSGs. Walk the path.

**What I'd do →**

1. Hub-spoke peering: `allowGatewayTransit` on hub side, `useRemoteGateways` on spoke side
2. On-prem routing: the spoke CIDR is advertised over BGP (check `az network vnet-gateway list-advertised-routes`)
3. If traffic goes via the firewall, the `GatewaySubnet` route table must send the spoke CIDR to the firewall, and the spoke must return via the firewall (symmetry)
4. NSG on the spoke subnet allows the on-prem source range
5. Firewall rule allows the flow; check firewall logs for drops

**Why →** Asymmetric routing (in via firewall, out direct) is the usual silent hang.

**Interview tip →** Draw the path out loud: on-prem → gateway → firewall → spoke and back.

</details>

### Q38. You need Application Gateway to serve 30 different hostnames with TLS. How do you manage certificates? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Key Vault-referenced certificates on multi-site listeners, auto-renewed.

**What I'd do →**

1. Give App Gateway a user-assigned identity with `Key Vault Secrets User` on the vault
2. Store certificates in Key Vault (auto-renew via integrated CA or import from ACME tooling)
3. Reference them on listeners with the versionless secret ID, so App Gateway polls and picks up renewals
4. Use wildcard certs where hostnames share a domain to reduce count

**Why →** Versionless Key Vault references remove manual cert uploads and expired-cert outages.

**Interview tip →** Mention App Gateway checks Key Vault periodically (every 4 hours) for new versions.

</details>

### Q39. Management asks for network traffic visibility across all VNets. What do you enable? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** VNet flow logs plus Traffic Analytics, enforced by policy.

**What I'd do →**

1. Enable VNet flow logs on each VNet to a regional storage account
2. Turn on Traffic Analytics into the central Log Analytics workspace (10-minute interval)
3. Use the built-in policy to deploy flow logs for all VNets in a region
4. Build workbooks for top talkers, denied flows and internet exposure

**Why →** Flow logs show allowed and denied flows; Traffic Analytics turns them into queryable, geo-enriched data.

**Interview tip →** Mention NSG flow logs are being retired in favour of VNet flow logs.

</details>

### Q40. A DDoS attack hit a competitor. Leadership asks what protects your public endpoints. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Layered answer: infrastructure DDoS protection is on by default; L3/L4 enhanced protection and L7 WAF are choices.

**What I'd do →**

1. Explain default infrastructure protection covers every public IP at platform scale
2. For critical apps, enable **DDoS Network Protection** on the VNet (or **DDoS IP Protection** per IP) for adaptive tuning, attack telemetry, rapid response and cost protection
3. Front Door + WAF with rate limiting and bot protection for L7 floods
4. Alert on the `IfUnderDDoSAttack` metric for protected public IPs

**Why →** Network-layer DDoS and application-layer floods need different controls.

**Interview tip →** Mention the cost: DDoS Network Protection is a significant monthly fee, so justify it per workload.

</details>

---

## AKS and Containers

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q41. You're asked to build a production AKS cluster. Walk through your key decisions. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The interviewer wants the decisions and trade-offs, not a single command.

**What I'd do →**

1. **Tier and zones**: Standard tier, node pools across zones 1, 2, 3
2. **Networking**: Azure CNI Overlay with Cilium dataplane, custom VNet subnet, UDR or NAT Gateway egress
3. **Identity**: Entra integration, Azure RBAC, local accounts disabled, workload identity + OIDC issuer
4. **Pools**: tainted system pool (3 × D4s_v5), autoscaling user pools, Spot pool for batch
5. **Add-ons**: Key Vault CSI, monitoring + Managed Prometheus, Azure Policy, app routing or App Gateway for Containers
6. **Upgrades**: auto-upgrade `patch` channel and `NodeImage` OS channel inside maintenance windows
7. **Delivery**: ACR with `AcrPull` for kubelet identity, GitOps via Flux

**Why →** Each choice is hard to change later (networking, zones) or critical for security (identity).

**Interview tip →** Call out which choices are immutable after creation; that's what interviewers probe.

</details>

### Q42. A team wants to deploy a new microservice to AKS that reads a database password. How do you wire secrets? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Key Vault + CSI driver + workload identity; no secrets in git or plain Kubernetes Secrets.

**What I'd do →**

1. Store the secret: `az keyvault secret set --vault-name kv-shop -n db-password --value ...`
2. Create a user-assigned identity, grant `Key Vault Secrets User` on the vault, add a federated credential for `system:serviceaccount:shop:orders-sa`
3. Create the ServiceAccount with `azure.workload.identity/client-id` annotation
4. Create a `SecretProviderClass` with `clientID`, `keyvaultName`, `tenantId` and the object list
5. Mount it in the Deployment (`csi` volume, driver `secrets-store.csi.k8s.io`), with pod label `azure.workload.identity/use: "true"`

**Why →** One identity per workload, revocable in Azure, auditable in Key Vault logs.

**Interview tip →** Mention Entra auth to the DB itself to avoid the password entirely.

</details>

### Q43. You need to upgrade a production AKS cluster from 1.31 to 1.33. Plan it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two minor hops, deprecated API checks, PDB review, staged rollout.

**What I'd do →**

1. Check removed APIs for 1.32 and 1.33 (use `kubent` or `pluto detect-helm`, and the AKS deprecated API detection in the portal)
2. Upgrade dev and staging clusters first and run tests
3. Prod: `az aks upgrade --control-plane-only --kubernetes-version 1.32.x`, then node pools one by one with `--max-surge 33%`
4. Watch: `kubectl get nodes -w`, events, error rates
5. Repeat for 1.33
6. Have PDBs that allow at least one disruption, and quota for surge nodes

**Why →** Control plane first, nodes after, one minor at a time is the only supported path, and staging catches API removals.

**Interview tip →** Mention quota for surge nodes; upgrades fail on it more than people expect.

</details>

### Q44. Pods need to scale on Service Bus queue length, not CPU. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** KEDA, via the AKS add-on, with workload identity to read queue metrics.

**What I'd do →**

1. Enable KEDA: `az aks update -g rg -n aks --enable-keda`
2. Give a user-assigned identity `Azure Service Bus Data Owner` (needed to read message counts) on the namespace, federate it to the KEDA operator service account (`system:serviceaccount:kube-system:keda-operator`)
3. Create a `TriggerAuthentication` with `podIdentity: provider: azure-workload` and the identity client ID
4. Create a `ScaledObject` with an `azure-servicebus` trigger, `queueName`, `messageCount: "50"`, `minReplicaCount: 0`

**Why →** Workers scale with backlog, including down to zero, which saves cost on idle queues.

**Interview tip →** Mention KEDA creates an HPA under the hood; don't define your own HPA on the same deployment.

</details>

### Q45. You need to run a GPU workload on an existing AKS cluster. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Separate GPU user node pool, tainted, scaled to zero when idle.

**What I'd do →**

1. Check quota for the GPU family (`az vm list-usage -l centralindia | grep -i NC`)
2. Add the pool: `az aks nodepool add -g rg --cluster-name aks -n gpu --node-vm-size Standard_NC4as_T4_v3 --node-taints sku=gpu:NoSchedule --enable-cluster-autoscaler --min-count 0 --max-count 3`
3. Install the NVIDIA device plugin (or GPU Operator) if the AKS GPU image doesn't handle it
4. Workloads request `nvidia.com/gpu: 1` with a matching toleration

**Why →** Taints keep regular pods off expensive GPUs; min 0 stops paying when idle.

**Interview tip →** Mention GPU quota is often zero by default and needs a request.

</details>

### Q46. A noisy application keeps starving other workloads on shared nodes. How do you fix it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Missing requests/limits and namespace controls.

**What I'd do →**

1. Set requests and limits on the noisy deployment based on observed usage (Container Insights or Prometheus)
2. Add a `ResourceQuota` and `LimitRange` on its namespace
3. Enforce "containers must have limits" with Azure Policy or Deployment Safeguards
4. If still noisy, move it to a dedicated node pool with taints and node affinity

**Why →** The scheduler and autoscaler only reason about requests; limits contain bursts.

**Interview tip →** Mention priority classes for critical workloads.

</details>

### Q47. You must move an AKS workload from one cluster to a new cluster with zero downtime. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Blue/green at cluster level, traffic switched at the edge.

**What I'd do →**

1. Build the new cluster from IaC, bootstrap with the same GitOps repo path
2. Re-create workload identities' federated credentials for the new cluster OIDC issuer (each cluster has its own issuer URL)
3. Deploy the workload; validate via a separate hostname
4. Shift traffic with Front Door weights (10% → 50% → 100%) or Traffic Manager
5. Keep the old cluster for a rollback window, then decommission

**Why →** Cluster replacement is safer than in-place upgrades for big changes (networking, OS SKU).

**Interview tip →** The OIDC issuer detail shows you've actually done a workload identity migration.

</details>

### Q48. Developers want to pull images from Docker Hub in AKS, and builds keep hitting rate limits. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Mirror public images into ACR.

**What I'd do →**

1. Import needed base images: `az acr import -n acrshop --source docker.io/library/python:3.12-slim --image base/python:3.12-slim`
2. Or set up an ACR **cache rule** (artifact cache) so `acrshop.azurecr.io/docker/library/python` pulls through
3. Update Dockerfiles and manifests to use ACR paths
4. Add Azure Policy allowing images only from `acrshop.azurecr.io`

**Why →** No rate limits, faster pulls in-region, and one place to scan images.

**Interview tip →** Mention ACR Tasks base-image triggers to keep these mirrors patched.

</details>

### Q49. You need to give a developer read-only kubectl access to one namespace. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** With Azure RBAC enabled, assign a built-in role at namespace scope.

**What I'd do →**

1. Assign the role:
   ```bash
   AKS_ID=$(az aks show -g rg -n aks --query id -o tsv)
   az role assignment create --assignee dev@devopsshack.com \
     --role "Azure Kubernetes Service RBAC Reader" --scope "$AKS_ID/namespaces/shop"
   az role assignment create --assignee dev@devopsshack.com \
     --role "Azure Kubernetes Service Cluster User Role" --scope "$AKS_ID"
   ```
2. Developer runs `az aks get-credentials` and `kubectl get pods -n shop`
3. Confirm `kubectl get pods -n kube-system` is forbidden

**Why →** The Cluster User role allows downloading kubeconfig; the RBAC Reader role defines what they can see.

**Interview tip →** Mention both roles; missing Cluster User Role is a common confusion.

</details>

### Q50. Your AKS bill doubled. Where do you look? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Usually nodes (autoscaler, oversized pools), Log Analytics ingestion, or networking.

**What I'd do →**

1. Cost analysis by meter for the node RG and the workspace
2. Check autoscaler events and node count history; look for pods with huge requests preventing scale-down (`kubectl top`, Container Insights)
3. Check Log Analytics usage: `Usage | where TimeGenerated > ago(30d) | summarize sum(Quantity) by DataType`
4. Enable AKS cost analysis add-on for per-namespace allocation
5. Fixes: right-size requests, Spot for batch, reservations for baseline, filter or Basic-plan container logs

**Why →** Data drives the fix; guessing leads to cutting the wrong thing.

**Interview tip →** Always mention Log Analytics; it's the AKS cost surprise nobody budgets for.

</details>

### Q51. You need different teams to share one AKS cluster safely. How do you set up multi-tenancy? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Soft multi-tenancy: namespaces with RBAC, quotas, network policies, and policy enforcement.

**What I'd do →**

1. Namespace per team per environment, Azure RBAC roles at namespace scope bound to team groups
2. `ResourceQuota` and `LimitRange` per namespace
3. Default-deny `NetworkPolicy` (Cilium) per namespace, explicit allow for ingress controller and shared services
4. Pod Security Admission `restricted` labels, Azure Policy for allowed registries
5. Separate workload identities per team; Key Vault per team
6. Dedicated node pools with taints for teams with special needs

**Why →** Each layer limits a different blast radius: access, resources, network, runtime.

**Interview tip →** Say when you'd give separate clusters instead: hostile tenants or strict compliance boundaries.

</details>

### Q52. A stateful service on AKS uses Azure Disks. A zone goes down. What happens and how do you design for it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** LRS disks are zonal; the pod can't reschedule to another zone with its volume.

**What I'd do →**

1. Explain: pod stays Pending (`volume node affinity conflict`) until the zone returns
2. Design fix: ZRS disks via a custom StorageClass (`skuName: Premium_ZRS`), which can attach in any zone
3. Or app-level replication (a database operator with replicas per zone) using LRS disks
4. Back up with Azure Backup for AKS or Velero

**Why →** Zone-redundant storage or replication is what makes stateful pods survive a zone loss.

**Interview tip →** "Volume node affinity conflict" is the phrase that shows you've hit it.

</details>

### Q53. Container images must be scanned and only signed images may run in prod. How do you build it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scan in CI and registry, sign after build, verify at admission.

**What I'd do →**

1. CI: Trivy scan, fail on critical CVEs with fixes available
2. Sign with Notation using a certificate in Key Vault: `notation sign --plugin azure-kv --id <keyId> acrshop.azurecr.io/shop/web@sha256:...`
3. Defender for Containers scans ACR images continuously
4. Install Ratify on AKS with Azure Policy (the signature verification policy) to deny unsigned images
5. Restrict registries to `acrshop.azurecr.io` via policy

**Why →** Signing proves an image came from your pipeline; admission control enforces it.

**Interview tip →** Sign by digest, not tag; tags are mutable.

</details>

### Q54. You need to expose three services from AKS on one public IP with TLS. What do you use? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** An ingress controller with host or path routing.

**What I'd do →**

1. Enable the app routing add-on: `az aks approuting enable -g rg -n aks`
2. Create Ingress resources with `ingressClassName: webapprouting.kubernetes.azure.com` and host rules
3. Store certs in Key Vault, integrate with `az aks approuting update --enable-kv --attach-kv <kvId>`, reference via annotation `kubernetes.azure.com/tls-cert-keyvault-uri`
4. DNS A records pointing at the ingress public IP (or Azure DNS integration via `az aks approuting zone add`)

**Why →** One LB frontend, managed controller lifecycle, certificates rotated from Key Vault.

**Interview tip →** Mention Gateway API as the forward-looking choice.

</details>

### Q55. An AKS node pool needs a different VM size. What's the safe way? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** VM size is mostly immutable on a pool; add a new pool and migrate.

**What I'd do →**

1. Add a new pool with the target size, same labels/taints, across zones
2. Cordon old nodes: `kubectl cordon -l agentpool=apps`
3. Drain gradually: `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`, respecting PDBs
4. Verify pods are healthy on the new pool, then `az aks nodepool delete -n apps`
5. For the default/system pool in Terraform, set `temporary_name_for_rotation` and let azurerm rotate it

**Why →** Workloads move gradually with PDB protection; no in-place risk.

**Interview tip →** Newer AKS supports resizing some pools in place, but the add-and-migrate pattern works everywhere.

</details>

### Q56. Your team wants to adopt GitOps on AKS. What do you set up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Flux extension or Argo CD, a repo structure per environment, CI only updates image tags.

**What I'd do →**

1. Repo layout: `apps/base`, `apps/overlays/{dev,prod}`, `clusters/{dev,prod}` with Kustomizations
2. Install Flux via `az k8s-configuration flux create` (or Argo CD via Helm)
3. CI builds and pushes, then opens a PR bumping the image tag in the overlay (or Flux image automation)
4. Prod changes require PR approval; Flux reconciles every few minutes and reports drift

**Why →** Git becomes the deployment audit log, and CI no longer needs cluster credentials.

**Interview tip →** Highlight that CI loses kube credentials, a security win.

</details>

### Q57. You need to run a nightly batch job that must not affect web pods. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** CronJob on a Spot pool, isolated by taints.

**What I'd do →**

1. Add a Spot user pool with autoscaler min 0
2. Create a `CronJob` with `tolerations` for `kubernetes.azure.com/scalesetpriority=spot:NoSchedule`, a `nodeSelector` for the pool, `concurrencyPolicy: Forbid`, `backoffLimit`
3. Make the job idempotent and checkpoint progress so evictions are survivable
4. Alert on job failures via Container Insights (`KubeEvents` or job status)

**Why →** Cheap, isolated capacity that scales to zero after the job.

**Interview tip →** Mention eviction handling; Spot without retries is a bug waiting.

</details>

### Q58. How would you migrate from Container Apps to AKS (or the reverse) for a service? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same container image; the difference is runtime config, identity, networking and scaling definitions.

**What I'd do →**

1. Inventory: env vars, secrets (Key Vault refs), scaling rules (KEDA triggers map 1:1), ingress, Dapr components
2. Translate to Kubernetes manifests or Helm: Deployment, Service, Ingress, ScaledObject, SecretProviderClass
3. Recreate the managed identity federation for AKS workload identity
4. Run both in parallel behind Front Door and shift traffic by weight
5. Decommission after a soak period

**Why →** Container images are portable; platform glue isn't, so map it deliberately.

**Interview tip →** Mention KEDA is in both platforms, which makes scaling rules portable.

</details>

### Q59. An app on AKS must call an Azure OpenAI or Storage API using its own identity. Set it up end to end. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Workload identity with a data-plane role on the target.

**What I'd do →**

1. `az aks update -g rg -n aks --enable-oidc-issuer --enable-workload-identity` (if not already)
2. `az identity create -g rg -n id-orders`; grant `Storage Blob Data Contributor` (or `Cognitive Services OpenAI User`) on the target
3. Federate: `az identity federated-credential create --identity-name id-orders -g rg --name aks-orders --issuer $(az aks show -g rg -n aks --query oidcIssuerProfile.issuerUrl -o tsv) --subject system:serviceaccount:shop:orders-sa --audiences api://AzureADTokenExchange`
4. ServiceAccount annotated with the client ID, pod labelled `azure.workload.identity/use: "true"`
5. Code uses `DefaultAzureCredential()`

**Why →** No secrets, per-workload identity, works with every Azure SDK.

**Interview tip →** Note that changing the SA name or namespace breaks the federation subject.

</details>

### Q60. Leadership asks whether you should run your database inside AKS. What's your answer? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Usually no: managed databases are cheaper to operate for most teams.

**What I'd do →**

1. Default to Azure Database for PostgreSQL Flexible Server or Azure SQL: zone-redundant HA, backups, PITR, patching
2. Consider in-cluster only with an operator (CloudNativePG), skills on the team, zone-redundant storage and tested backups
3. Compare cost including people time, not just compute

**Why →** Databases need backup, failover and patching discipline that managed services give you for free.

**Interview tip →** Ask "who gets paged when the in-cluster database fails over at 3 a.m.?"

</details>

### Q61. You need an AKS cluster with no public egress at all for compliance. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Network isolated cluster: private cluster, private ACR as the bootstrap source, no outbound internet.

**What I'd do →**

1. Premium ACR with Private Endpoint and an artifact cache rule for MCR
2. Create the cluster with `--bootstrap-artifact-source Cache --bootstrap-container-registry-resource-id <acrId> --outbound-type none --enable-private-cluster`
3. Private Endpoints for Key Vault, monitoring (Azure Monitor Private Link Scope)
4. Node OS updates via node image upgrades only (no package mirrors from the internet)

**Why →** Nodes pull all bootstrap images from your registry, so nothing reaches the internet.

**Interview tip →** Mention AMPLS for monitoring; Container Insights needs a private path too.

</details>

### Q62. A security team wants to know which pods talk to what. How do you provide it on AKS? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Network observability at pod level.

**What I'd do →**

1. Enable Advanced Container Networking Services (with Cilium): Hubble flow logs and network metrics, `az aks update --enable-acns`
2. Send flow metrics to Managed Prometheus, view in Grafana dashboards
3. Use Hubble CLI/UI for live flows per namespace and dropped flows from network policies
4. Feed this into default-deny network policy design

**Why →** You can't write good network policies without knowing current flows.

**Interview tip →** Connect observability to enforcement; that's the senior framing.

</details>

### Q63. A Windows .NET Framework app must run on AKS. What changes? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Windows node pool alongside Linux system pool.

**What I'd do →**

1. Add a Windows pool: `az aks nodepool add -g rg --cluster-name aks -n win22 --os-type Windows --os-sku Windows2022`
2. Pool names for Windows max 6 characters
3. Use `nodeSelector: kubernetes.io/os: windows` on Windows workloads; ensure Linux workloads select Linux
4. Build images on `mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2022` matching the node OS version
5. Plan for larger images and slower pulls

**Why →** Windows containers must match the host OS version and need explicit scheduling.

**Interview tip →** Mention Azure CNI is required (kubenet doesn't support Windows).

</details>

### Q64. You need to rotate the TLS certificate used by your AKS ingress without downtime. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** If the cert lives in Key Vault with CSI or app routing integration, rotation is a new version.

**What I'd do →**

1. Import or let Key Vault renew the certificate (new version)
2. The CSI driver with rotation enabled (or app routing's Key Vault sync) updates the synced Secret
3. The ingress controller reloads the cert from the updated Secret
4. Verify: `openssl s_client -connect shop.devopsshack.com:443 -servername shop.devopsshack.com | openssl x509 -noout -dates`

**Why →** Versionless references and rotation polling avoid manual secret edits.

**Interview tip →** cert-manager with Let's Encrypt is a valid alternative; mention either.

</details>

### Q65. A developer `kubectl edit`ed a production Deployment during an incident. How do you prevent drift going forward? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Humans need break-glass, not routine write access.

**What I'd do →**

1. Developers get `Azure Kubernetes Service RBAC Reader` in prod namespaces
2. Writes only through GitOps (Flux reverts manual changes on reconcile)
3. Break-glass via a PIM-eligible group with `RBAC Writer`
4. Alert on `kube-audit-admin` logs for writes by human users in prod

**Why →** GitOps makes the repo the truth; manual edits get reverted and are visible in audit.

**Interview tip →** Mention the follow-up: the incident fix must be committed to git.

</details>

---

## CI/CD and IaC

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q66. Design a GitHub Actions pipeline that builds an image, pushes to ACR and deploys to AKS with no secrets. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** OIDC to a user-assigned identity with AcrPush and namespace-scoped AKS rights.

**What I'd do →**

1. Identity `id-shop-ci` with federated credential `repo:org/shop:environment:prod`; roles: `AcrPush` on ACR, `Azure Kubernetes Service Cluster User Role` on the cluster, `Azure Kubernetes Service RBAC Writer` on the namespace
2. Workflow: `permissions: id-token: write`, `azure/login@v2`, `az acr build` or `docker build` + `az acr login` + push, tag with `${{ github.sha }}`
3. Deploy job with `environment: prod` (approval), `az aks get-credentials`, `kubelogin convert-kubeconfig -l azurecli`, `kubectl apply` or `helm upgrade --install`
4. `kubectl rollout status` as the gate

**Why →** Zero stored secrets, environment approvals gate prod, identity has only what it needs.

**Interview tip →** Use immutable tags (git SHA), never `latest`, in deployment manifests.

</details>

### Q67. Your organisation uses Azure DevOps. Set up a secure deployment pipeline to AKS. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** WIF service connections, environments with checks, templates.

**What I'd do →**

1. Create an ARM service connection with workload identity federation, scoped to the app RG, not granted to all pipelines
2. Docker registry service connection for ACR (also WIF-backed)
3. Environment `shop-prod` with approvals and branch control (`refs/heads/main`)
4. Pipeline: `Docker@2` buildAndPush, then a `deployment` job to `shop-prod` using `AzureCLI@2` or `KubernetesManifest@1`
5. Extend from a platform template that adds scanning stages

**Why →** Approvals and checks live on the resources, so a modified pipeline can't skip them.

**Interview tip →** Mention checks on the service connection itself, not only the environment.

</details>

### Q68. You need to promote the same image from dev to staging to prod. How do you design it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Build once, promote by digest.

**What I'd do →**

1. Build and push once with tag `sha-<commit>`; record the digest
2. Deploy to dev automatically; run tests
3. Promote to staging and prod by changing only the manifest/values reference (GitOps PR or pipeline stage), same digest
4. Optionally `az acr import` between dev and prod registries to keep prod registry isolated

**Why →** Rebuilding per environment can produce different artifacts; promoting the same digest means what you tested is what you ship.

**Interview tip →** Say "build once, deploy many" and "by digest".

</details>

### Q69. Terraform state for prod is in a storage account accessible with account keys. Harden it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** State contains secrets and is the keys to the kingdom. Entra auth, network isolation, recovery.

**What I'd do →**

1. Backend `use_azuread_auth = true`; grant `Storage Blob Data Contributor` on the container to the pipeline identity only
2. `az storage account update --allow-shared-key-access false`
3. Enable blob versioning and soft delete; `CanNotDelete` lock on the account
4. Private Endpoint (with private agents) or network rules limiting access
5. Separate state per environment, separate identities per environment

**Why →** Versioning recovers from corrupted state, RBAC limits who can read secrets in state, locks stop deletion.

**Interview tip →** Mention state contains secrets in plaintext; that's why it's guarded.

</details>

### Q70. Your team wants to move from ARM JSON templates to Bicep. How do you migrate? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Bicep compiles to ARM, so migration is mechanical and low risk.

**What I'd do →**

1. `az bicep decompile -f azuredeploy.json` for each template
2. Clean up: modules, parameter types, remove generated noise, use symbolic references
3. Run `az deployment group what-if` against existing environments; expect no changes
4. Switch pipelines to Bicep, keep the same deployment names and scopes

**Why →** What-if confirms the Bicep version describes the same resources before any deployment.

**Interview tip →** Mention decompile is "best effort"; review the output.

</details>

### Q71. A pipeline needs to deploy to dev, test and prod in different subscriptions. How do you handle identities? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** One identity per environment, each federated to its environment.

**What I'd do →**

1. Three user-assigned identities, each with roles only in its subscription
2. Federated credentials: `repo:org/shop:environment:dev`, `...:test`, `...:prod`
3. Each workflow job sets `environment:` and uses that environment's `AZURE_CLIENT_ID` (stored as environment variables in GitHub)
4. Prod environment has required reviewers and branch restrictions

**Why →** A compromised dev job can't touch prod, because it can never get the prod token.

**Interview tip →** Environment-scoped federation is the key phrase.

</details>

### Q72. The Terraform plan for a small change shows the AKS cluster will be replaced. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Stop. Find the ForceNew attribute before anything else.

**What I'd do →**

1. Read the plan line marked `# forces replacement`
2. Common causes: provider upgrade changing defaults, case differences in IDs, changed `default_node_pool` immutable fields, `dns_prefix`, network profile
3. Fix: revert the field, add `temporary_name_for_rotation` for node pool changes, or use `lifecycle { ignore_changes = [...] }` for externally-managed fields
4. Add `prevent_destroy = true` on the cluster resource in prod
5. Re-run plan; proceed only when it shows in-place updates

**Why →** Replacing a cluster means downtime, new OIDC issuer, new IPs and lost PVs.

**Interview tip →** Mention `prevent_destroy`; it's a cheap seatbelt.

</details>

### Q73. Portal changes keep drifting from Terraform. How do you get control back? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Detect, decide, prevent.

**What I'd do →**

1. Nightly `terraform plan -detailed-exitcode` job; exit code 2 posts the diff to the team channel
2. For each drift: codify in Terraform or revert by applying
3. Reduce human write access in prod; PIM for emergencies
4. Azure Policy deny/modify for the properties people keep changing (TLS version, public access)

**Why →** Detection alone doesn't stop drift; removing the write path does.

**Interview tip →** Mention `terraform apply -refresh-only` for accepting harmless drift into state.

</details>

### Q74. You need a reusable pipeline for 50 microservices in Azure DevOps. How do you design it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Central template repo with extends templates, versioned.

**What I'd do →**

1. `platform/pipeline-templates` repo with `build.yml`, `deploy-aks.yml`, `service.yml` (extends template), tagged releases
2. Each service's `azure-pipelines.yml` is ~15 lines: resource reference to the template at a tag, parameters (service name, chart path)
3. Environments require the template via "Required template" check
4. Roll out template changes by bumping the tag per service or with a bot PR

**Why →** One place to add security scanning or change deploy logic, enforced for prod.

**Interview tip →** Pinning to a tag avoids breaking 50 pipelines at once.

</details>

### Q75. A deployment to App Service must have zero downtime and instant rollback. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Deployment slots.

**What I'd do →**

1. Deploy to the `staging` slot (`azure/webapps-deploy@v3` with `slot-name: staging`)
2. Smoke test the staging URL
3. Swap: `az webapp deployment slot swap -g rg -n app-shop --slot staging`
4. Rollback: swap again
5. Mark environment-specific settings as slot settings

**Why →** Swap warms instances before switching traffic and keeps the previous version ready in the other slot.

**Interview tip →** Mention swap with preview for multi-phase validation.

</details>

### Q76. You must run Terraform against a private AKS cluster and private Key Vault from CI. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hosted runners can't reach private endpoints. Need private runners.

**What I'd do →**

1. Self-hosted runners/agents in a VNet with line of sight (Managed DevOps Pools for Azure DevOps, or GitHub-hosted runners with Azure private networking, or self-hosted runners on VMSS/ACA jobs)
2. DNS resolution of privatelink zones from that VNet
3. Same OIDC identities; no change to auth
4. For one-off kubectl, `az aks command invoke` from hosted runners

**Why →** Private endpoints mean network access is required in addition to identity.

**Interview tip →** Mention GitHub's Azure private networking for hosted runners; few candidates know it.

</details>

### Q77. You need to deploy infrastructure and application in one pipeline but with separate approvals. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Separate stages, environments and identities.

**What I'd do →**

1. Stage `infra-plan` (Reader identity), output plan artifact
2. Stage `infra-apply` targeting environment `infra-prod` with platform team approval (Contributor identity)
3. Stage `app-deploy` targeting environment `shop-prod` with app team approval (namespace-scoped identity)
4. Skip `infra-apply` when the plan has no changes (set an output variable from `terraform plan -detailed-exitcode`)

**Why →** Different owners approve different risks, and each stage has only the rights it needs.

**Interview tip →** Conditional infra stages keep pipelines fast.

</details>

### Q78. A PR changes an NSG to open port 22 to the internet. How do you catch it automatically? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Static analysis in PR, policy at deploy.

**What I'd do →**

1. Run Checkov or Trivy config scanning in PR builds; both flag `0.0.0.0/0` on 22/3389
2. Make the check required in branch policies
3. Azure Policy deny for management ports from Internet as a backstop
4. Defender for Cloud recommendation alerts if anything slips through

**Why →** Shift left for fast feedback, enforce at ARM for certainty.

**Interview tip →** "Defence in depth for IaC: PR scan, policy, runtime detection."

</details>

### Q79. How do you manage secrets needed by Terraform (like a DB admin password)? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Avoid passing secrets around; generate and store in Key Vault, or avoid passwords.

**What I'd do →**

1. Use `random_password` to generate, write to Key Vault with `azurerm_key_vault_secret`, consumers read from Key Vault
2. Better: Entra-only auth on Azure SQL/PostgreSQL so no admin password exists
3. Mark variables `sensitive = true`; know they still land in state
4. Use ephemeral resources/write-only arguments (Terraform 1.10+/1.11+) where the provider supports them to keep secrets out of state

**Why →** Every secret in variables ends up in logs, state or both unless you design against it.

**Interview tip →** Mention state encryption and access as the remaining risk.

</details>

### Q80. You need to apply Azure Policy definitions themselves as code. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Policy-as-code repo, deployed by pipeline, tested in a sandbox management group.

**What I'd do →**

1. Store definitions, initiatives and assignments in git (Bicep, Terraform `azurerm_policy_definition`, or the EPAC tool)
2. Pipeline deploys to a test management group first, runs compliance checks
3. Promote to production management groups with approval
4. New deny policies start in Audit (or with `enforcementMode: DoNotEnforce`)

**Why →** Policies can break every team at once; they deserve the same review and staging as code.

**Interview tip →** `enforcementMode: DoNotEnforce` lets you see impact without blocking.

</details>

### Q81. A Helm deployment to AKS fails halfway and leaves the release in `pending-upgrade`. How do you make the pipeline robust? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use atomic upgrades and a clear rollback.

**What I'd do →**

1. `helm upgrade --install shop ./chart -n shop --atomic --timeout 10m --wait` (`--rollback-on-failure` in Helm 4)
2. If stuck: `helm history shop -n shop`, then `helm rollback shop <rev> -n shop`
3. Serialise deployments per environment (concurrency group in GitHub, exclusive lock check in Azure DevOps)
4. Add readiness probes so `--wait` reflects real health

**Why →** Atomic rollbacks avoid half-deployed releases; serialisation avoids two pipelines fighting.

**Interview tip →** Concurrency control is often the actual root cause.

</details>

### Q82. Developers want ephemeral environments per PR on Azure. How do you build it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Namespace per PR on a shared AKS cluster (cheap) or RG per PR (more isolation).

**What I'd do →**

1. On PR open: create namespace `pr-123`, deploy with Helm using the PR image, ingress host `pr-123.dev.devopsshack.com`
2. Backing services: shared dev database with a per-PR schema, or containers in the namespace
3. Comment the URL on the PR
4. On PR close: `helm uninstall` and delete the namespace; nightly cleanup of orphaned namespaces by label age

**Why →** Reviewers test real deployments; namespaces keep cost low.

**Interview tip →** Mention cleanup automation; orphaned environments are the cost trap.

</details>

### Q83. Your pipeline's Azure login works on `main` but fails on pull requests. Why, and what should you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The federated subject differs on PRs (`repo:org/repo:pull_request`), and that's often intentional.

**What I'd do →**

1. Decide whether PRs should have Azure access at all (plan-only, Reader)
2. If yes, add a separate federated credential for `pull_request` on a Reader-only identity
3. Never give PR workflows deploy identities; forks can submit PRs

**Why →** PR code is untrusted until merged; its token should not deploy anything.

**Interview tip →** Saying "PRs get a read-only identity" shows security thinking.

</details>

### Q84. How do you test Bicep or Terraform modules before teams consume them? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Static checks, then real deployments in a test subscription.

**What I'd do →**

1. Static: `bicep lint` / PSRule for Azure, `terraform validate`, `tflint` with azurerm ruleset, Checkov
2. Unit-ish: `terraform test` with `command = plan` assertions
3. Integration: `terraform test` with `command = apply` in a sandbox subscription, assertions on outputs, auto-destroy
4. Publish versioned releases (Bicep registry, git tags) only after tests pass

**Why →** Module bugs multiply across every consumer; tests catch them once.

**Interview tip →** Mention PSRule for Azure; it maps to Well-Architected rules.

</details>

### Q85. A deployment must be gated on error rate after release. How do you automate rollback? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Progressive delivery with metrics analysis.

**What I'd do →**

1. Use Argo Rollouts or Flagger on AKS with a canary strategy
2. Analysis template querying Managed Prometheus (or App Insights via a web metric) for 5xx rate and p95 latency
3. Steps: 10% → analysis 5 minutes → 50% → analysis → 100%; abort and roll back on failure
4. Pipeline only updates the image; rollout controller handles promotion

**Why →** Automated, metric-driven rollback is faster than a human watching dashboards.

**Interview tip →** Define the SLO-based thresholds before the tooling.

</details>

---

## Reliability, Cost and Migration

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q86. A key customer wants a 99.99% SLA for an app currently in a single region. What do you propose? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Composite SLA math says single-region with several dependencies won't reach four nines.

**What I'd do →**

1. Calculate current composite SLA (Front Door × App Gateway × AKS × SQL)
2. Propose active-active or active-passive across two regions behind Front Door
3. Zone-redundant everything within each region
4. Data: Azure SQL failover groups or Cosmos DB multi-region
5. Prove it with failover drills and measure RTO/RPO

**Why →** Parallel regions multiply failure probabilities, raising the composite SLA above any single region.

**Interview tip →** Do the SLA math out loud; interviewers love it.

</details>

### Q87. You need to define RTO and RPO for a workload and build DR. Walk through it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Business decides RTO/RPO; you choose the pattern that meets them at acceptable cost.

**What I'd do →**

1. Agree targets with the business (for example RTO 1 hour, RPO 15 minutes)
2. Pattern: backup-restore (hours), pilot light (tens of minutes), warm standby (minutes), active-active (seconds)
3. For RTO 1h/RPO 15m: warm standby in South India, IaC ready, SQL geo-replication, GZRS storage, ACR geo-replication
4. Runbook and a scheduled DR drill

**Why →** Cost scales with tighter targets; matching pattern to target avoids overspending.

**Interview tip →** Mention Azure Site Recovery for VM-based workloads.

</details>

### Q88. Monthly Azure spend is 30% over budget. What's your 30-day plan? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Find the top spenders, take quick wins, then commit.

**What I'd do →**

1. Cost analysis grouped by service and RG; top 10 meters
2. Quick wins: delete unattached disks, idle public IPs, stopped-but-allocated VMs, old snapshots; shut down dev at night (AKS stop, VM auto-shutdown)
3. Right-size via Advisor recommendations; reduce Log Analytics ingestion
4. Commit: reservations/savings plans for stable baseline
5. Budgets with alerts per team; tag-based showback

**Why →** Cleanup gives immediate savings; commitments give the largest durable discount.

**Interview tip →** Quantify: "Removed X, saved Y per month."

</details>

### Q89. You're migrating an app from AWS to Azure. How do you plan it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Map services, rebuild IaC, migrate data, cut over with rollback.

**What I'd do →**

1. Inventory and map: EKS→AKS, ECR→ACR, RDS→Azure Database for PostgreSQL, S3→Blob, Secrets Manager→Key Vault, ALB→App Gateway, Route 53→Azure DNS
2. Rebuild IaC in Terraform azurerm; don't translate CloudFormation literally
3. Identity: IRSA → workload identity; IAM policies → RBAC data roles
4. Data: Azure Database Migration Service or logical replication for PostgreSQL; AzCopy for S3 to Blob (`azcopy copy 'https://s3.amazonaws.com/bucket' 'https://acct.blob.core.windows.net/c' --recursive`)
5. Cut over with low DNS TTLs; keep AWS running for rollback

**Why →** Services map closely; identity and networking differ most and need redesign, not translation.

**Interview tip →** Mention subnets spanning AZs and RG structure as design differences.

</details>

### Q90. Dev and test environments run 24/7. How do you cut their cost? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Schedule them off outside working hours.

**What I'd do →**

1. AKS: `az aks stop` nightly via a scheduled pipeline or Automation runbook; or scale user pools to zero
2. VMs: auto-shutdown schedules (`az vm auto-shutdown -g rg -n vm --time 2000`), deallocate not just stop
3. Smaller SKUs and Free tier AKS in dev
4. Log Analytics daily cap in dev workspaces
5. Azure Dev/Test pricing on eligible subscriptions

**Why →** Non-prod is idle about 70% of the week; scheduling is the cheapest big win.

**Interview tip →** "Stopped" in the OS still bills; "deallocated" doesn't.

</details>

### Q91. A region-wide Azure incident is affecting your primary region. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Confirm, decide on failover with data-loss trade-off, execute the runbook.

**What I'd do →**

1. Confirm via Service Health and your own probes; open the incident bridge
2. Estimate impact and Microsoft's ETA; decide against your RTO
3. Fail over: Front Door shifts automatically if probes fail; manually fail over SQL failover group and storage if needed (accept RPO)
4. Scale up the secondary region
5. After recovery, fail back deliberately, then run a post-incident review

**Why →** A clear decision point (RTO vs ETA) avoids both panic failovers and waiting too long.

**Interview tip →** Mention customer-initiated storage failover loses data written after last sync.

</details>

### Q92. You're asked to cut Log Analytics costs by half without losing important data. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Find the biggest tables, then filter, move to cheaper plans or reduce retention.

**What I'd do →**

1. `Usage | where TimeGenerated > ago(30d) | summarize GB=sum(Quantity)/1000 by DataType | top 10 by GB`
2. Container Insights: filter namespaces, disable stdout for chatty ones, lower collection interval in the DCR
3. Move verbose tables (`ContainerLogV2`) to the Basic plan
4. DCR transformations to drop debug lines
5. Commitment tier if daily volume is above 100 GB

**Why →** A few tables are usually the majority of cost.

**Interview tip →** Quote the query; it shows you've actually done it.

</details>

### Q93. An important VM keeps getting "unexpectedly rebooted". How do you make the workload resilient? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Single VMs fail. Check Resource Health, then remove single points of failure.

**What I'd do →**

1. Resource Health and Activity Log for platform-initiated events (host maintenance, hardware)
2. Short term: Scheduled Events handling in the app, maintenance configuration for controlled windows
3. Long term: two or more VMs across zones (VMSS Flexible) behind a Load Balancer
4. Azure Backup and Site Recovery for DR

**Why →** No single VM SLA covers application availability; redundancy does.

**Interview tip →** Mention Scheduled Events give 15 minutes notice for most maintenance.

</details>

### Q94. Your company is moving to Azure from on-prem VMware. What migration approach do you take? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Assess, choose a strategy per app (rehost/replatform/refactor), migrate in waves.

**What I'd do →**

1. Azure Migrate discovery and assessment (sizing, cost, dependencies)
2. Group apps into waves by dependency
3. Rehost with Azure Migrate server migration (or Azure VMware Solution for a fast lift)
4. Replatform easy wins: databases to managed services, web apps to App Service
5. Landing zone first: networking, identity, policy, monitoring

**Why →** Waves reduce risk; landing zone first avoids rework.

**Interview tip →** Mention the "6 Rs" briefly and pick per app.

</details>

### Q95. A team wants to use Azure SQL. How do you ensure it's production ready? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** HA, backups, security, monitoring.

**What I'd do →**

1. Zone-redundant configuration on Business Critical or General Purpose (where supported)
2. Failover group to a secondary region if DR is needed
3. Entra-only authentication, managed identity for apps, Private Endpoint, public access disabled
4. Backup retention and long-term retention set; test a PITR restore
5. Diagnostic settings, Defender for SQL, alerts on DTU/vCore, storage and deadlocks

**Why →** The defaults are good but not production-grade for every requirement.

**Interview tip →** "Entra-only auth" removes SQL passwords entirely.

</details>

### Q96. You need to reduce compute cost for a stable AKS workload by 40%. Options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Right-size first, then commit and use cheaper capacity.

**What I'd do →**

1. Right-size requests (VPA recommendations or Prometheus data); bin-pack with fewer, larger nodes
2. Arm (`Dpsv5`) or AMD SKUs where images support it
3. 3-year reservations or savings plan for baseline nodes
4. Spot pool for interruptible work
5. Autoscaler profile tuned for faster scale-down

**Why →** Over-requested pods waste the most; commitments then discount what remains.

**Interview tip →** Right-size before buying reservations; otherwise you lock in waste.

</details>

### Q97. A third-party audit asks how you prove encryption and access controls on Azure data. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Show policies, compliance reports and logs.

**What I'd do →**

1. Azure Policy compliance for encryption, TLS, public access settings
2. Defender for Cloud regulatory compliance dashboard (for example ISO 27001, PCI DSS)
3. Key Vault and storage diagnostic logs showing who accessed what
4. RBAC exports and PIM activation history
5. Microsoft's own audit reports from the Service Trust Portal for the platform side

**Why →** Auditors want evidence, not descriptions; policy compliance and logs are evidence.

**Interview tip →** Mention the shared responsibility split.

</details>

### Q98. You inherit an Azure estate with no IaC. How do you bring it under control? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Inventory, prioritise, import incrementally, lock down.

**What I'd do →**

1. Inventory with Azure Resource Graph: `az graph query -q "Resources | summarize count() by type, resourceGroup"`
2. Prioritise prod critical resources
3. Export with `aztfexport` per RG; clean up into modules; import blocks
4. Achieve "no changes" plan, then make the pipeline the only write path
5. Policy for guardrails; drift detection

**Why →** Importing is incremental; a clean plan proves code matches reality before you depend on it.

**Interview tip →** Mention Resource Graph; it's the fastest way to see everything.

</details>

### Q99. App Service instances keep running out of SNAT ports when calling external APIs. What do you design? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Outbound connection churn exhausts the limited SNAT allocation per instance.

**What I'd do →**

1. Fix the app: reuse HTTP clients (connection pooling, `HttpClientFactory` in .NET), keep-alive
2. VNet integration with NAT Gateway for large SNAT capacity
3. Private Endpoints for Azure services so traffic doesn't use SNAT
4. Monitor "SNAT Port Exhaustion" in App Service diagnostics

**Why →** Code fixes reduce demand; NAT Gateway raises supply.

**Interview tip →** Always mention connection reuse first.

</details>

### Q100. Leadership wants a "cloud-agnostic" setup across Azure and AWS. How do you respond? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Portability has a cost; target portability where it matters.

**What I'd do →**

1. Clarify the goal: avoid lock-in, regulatory, negotiating power or real multi-cloud resilience
2. Portable layers: containers, Kubernetes, Terraform, OpenTelemetry, GitOps
3. Accept native managed services (databases, identity) where they save the most operational effort, behind clean interfaces
4. Estimate the cost of lowest-common-denominator design vs exit cost

**Why →** Full cloud-agnosticism means not using the best features of either cloud.

**Interview tip →** Lead with the business goal, then the trade-off.

</details>

---

## Identity and Authorization Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. `(AuthorizationFailed) The client 'ci@contoso.com' with object id '1111...' does not have authorization to perform action 'Microsoft.Resources/subscriptions/resourcegroups/write' over scope '/subscriptions/.../resourcegroups/rg-shop-dev' or the scope is invalid.` 🟢

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

### Q102. `AADSTS70021: No matching federated identity record found for presented assertion. Assertion Issuer: 'https://token.actions.githubusercontent.com'. Assertion Subject: 'repo:devopsshack/shop:ref:refs/heads/feature/login'.` 🟢

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

### Q103. `AADSTS7000222: The provided client secret keys for app '2222...' are expired.` Pipelines failed overnight. 🟢

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

### Q104. `AADSTS50076: Due to a configuration change made by your administrator, or because you moved to a new location, you must use multi-factor authentication to access '797f4846-ba00-4fd7-ba43-dac1f8f63013'.` in a nightly script. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The script uses a human account, and a Conditional Access policy now requires MFA for Azure management (`797f4846...` is the Windows Azure Service Management API).

**Diagnose →** Check Entra sign-in logs for the account, which show the CA policy applied.

**Fix →**

1. Move the script to a managed identity (if it runs in Azure) or a service principal with federated credentials
2. Grant that identity the needed roles; remove the human account from automation

**Prevent →** Policy: no automation with user accounts. Conditional Access will keep tightening.

</details>

### Q105. `Code: Forbidden. Message: Caller is not authorized to perform action on resource. Action: 'Microsoft.KeyVault/vaults/secrets/getSecret/action' ... Inner error: {"code":"ForbiddenByRbac"}` 🟢

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

### Q106. `The user, group or application 'appid=...;oid=...' does not have secrets get permission on key vault 'kv-legacy;location=centralindia'.` 🟢

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

### Q107. A user was added to an Entra group with access, but still gets `AuthorizationFailed`. 🟢

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

### Q108. `PrincipalNotFound: Principal 3333... does not exist in the directory <tenantId>.` when Terraform creates a role assignment for a new identity. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Entra replication lag: the identity was just created and ARM's lookup can't see it yet.

**Diagnose →** Re-run after a minute; it usually succeeds, which confirms replication delay.

**Fix →**

1. In Terraform, set `skip_service_principal_aad_check = true` on `azurerm_role_assignment` for service principals and managed identities
2. In CLI, pass `--assignee-object-id <id> --assignee-principal-type ServicePrincipal`

**Prevent →** Always specify principal type for newly created identities.

</details>

### Q109. `RoleAssignmentExists: The role assignment already exists.` breaks a Terraform apply. 🟡

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

### Q110. `azure/login` fails with `Error: Unable to get ACTIONS_ID_TOKEN_REQUEST_URL env variable`. 🟡

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

### Q111. `Error: AADSTS700016: Application with identifier '4444...' was not found in the directory 'Contoso'.` 🟡

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

### Q112. A managed identity on a VM gets `ManagedIdentityCredential authentication unavailable. No identity has been assigned to this resource.` 🟡

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

### Q113. `LinkedAuthorizationFailed: The client has permission to perform action 'Microsoft.Network/networkInterfaces/write' ... however, it does not have permission to perform action 'Microsoft.Network/virtualNetworks/subnets/join/action' on the linked scope(s)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The identity can create resources in its RG but the subnet is in another RG (often the network team's) where it has no rights.

**Diagnose →** Read the linked scope in the error; check assignments there.

**Fix →**

1. Grant a role with `subnets/join/action` on the subnet (Network Contributor, or a custom "Subnet Joiner" role)
2. Retry the deployment

**Prevent →** Provide a custom least-privilege subnet join role for app teams in hub-spoke designs.

</details>

### Q114. AKS `kubectl get pods` returns `Error from server (Forbidden): pods is forbidden: User "dev@devopsshack.com" cannot list resource "pods" in API group "" in the namespace "shop": User does not have access to the resource in Azure. Update role assignment to allow access.` 🟢

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

### Q115. `kubelogin is not installed which is required to connect to AAD enabled cluster.` 🟢

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

### Q116. `ERROR: (AuthorizationFailed) ... does not have authorization to perform action 'Microsoft.ContainerService/managedClusters/listClusterUserCredential/action'` running `az aks get-credentials`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The identity has Kubernetes RBAC roles but not the Azure role to download kubeconfig.

**Diagnose →** Check roles at cluster scope.

**Fix →**

1. Grant `Azure Kubernetes Service Cluster User Role` on the cluster
2. Admin credentials (`--admin`) need `Azure Kubernetes Service Cluster Admin Role` and local accounts enabled

**Prevent →** Pair Cluster User Role with every Kubernetes RBAC role grant.

</details>

### Q117. Azure DevOps fails with `AADSTS700213: No matching federated identity record found for presented assertion subject 'sc://contoso/shop/arm-shop-prod'.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The service connection was renamed, or the project/org name differs from when the federated credential was created; the subject includes them.

**Diagnose →** Compare the subject in the error with the federated credential on the app/identity.

**Fix →**

1. Update the federated credential subject to `sc://contoso/shop/arm-shop-prod`
2. Or use the service connection's "convert/verify" flow to regenerate it

**Prevent →** Don't rename service connections casually; document that the name is part of the trust.

</details>

### Q118. `RequestDisallowedByPolicy: Resource 'stshopdev' was disallowed by policy. Reasons: 'Storage accounts should disable public network access'.` 🟢

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

### Q119. A Terraform pipeline fails with `Error: building account: could not acquire access token to parse claims: clientCredentialsToken: received HTTP status 400 with response: {"error":"invalid_client","error_description":"AADSTS7000215: Invalid client secret provided..."}`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The client secret value is wrong (secret ID pasted instead of value), rotated, or contains an escaped character mangled by CI.

**Diagnose →** Check the CI variable; confirm the secret's key ID and expiry with `az ad app credential list --id <appId>`.

**Fix →**

1. Reset the secret and paste the **value**
2. Better, switch to `ARM_USE_OIDC=true` and remove `ARM_CLIENT_SECRET`

**Prevent →** OIDC everywhere.

</details>

### Q120. After moving a subscription to another tenant, apps fail with token errors and all role assignments are gone. 🔴

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

---

## Quota, Deployment and IaC Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q121. `(QuotaExceeded) Operation could not be completed as it results in exceeding approved standardDSv5Family Cores quota. Location: centralindia, Current Limit: 10, Current Usage: 8, Additional Required: 4, (Minimum) New Limit Required: 12.` 🟢

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

### Q122. `SkuNotAvailable: The requested VM size Standard_D4s_v5 is currently not available in location 'centralindia' zones '3'.` 🟡

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

### Q123. `MissingSubscriptionRegistration: The subscription is not registered to use namespace 'Microsoft.ContainerService'.` 🟢

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

### Q124. `StorageAccountAlreadyTaken: The storage account named sttfstate is already taken.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Storage account names are globally unique across all of Azure.

**Diagnose →** `az storage account check-name -n sttfstate`

**Fix →** Use a unique suffix: `sttfstate${RANDOM}` or a hash of subscription ID (`uniqueString(subscription().id)` in Bicep, `random_string` in Terraform).

**Prevent →** Naming convention with a unique suffix for globally unique names (storage, Key Vault, ACR).

</details>

### Q125. `(VaultAlreadyExists)` or `A vault with the same name already exists in deleted state. You need to either recover or purge existing key vault.` 🟢

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

### Q126. `ScopeLocked: The scope '/subscriptions/.../resourceGroups/rg-shop-prod' cannot perform delete operation because following scope(s) are locked: '/subscriptions/.../resourceGroups/rg-shop-prod'.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A CanNotDelete or ReadOnly lock on the RG or a parent.

**Diagnose →** `az lock list -g rg-shop-prod -o table` and at subscription level.

**Fix →**

1. If the delete is intended, remove the lock (needs `Microsoft.Authorization/locks/delete`, Owner or User Access Administrator): `az lock delete -n no-delete -g rg-shop-prod`
2. Re-add after the change if appropriate

**Prevent →** Manage locks in IaC so their presence is intentional and visible.

</details>

### Q127. `DeploymentFailed: At least one resource deployment operation failed. Please list deployment operations for details.` How do you find the real error? 🟢

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

### Q128. `InvalidTemplateDeployment: The template deployment 'main' is not valid according to the validation procedure.` with a policy violation inside. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Preflight validation found a policy deny or an invalid property before deploying anything.

**Diagnose →** Run `az deployment group validate -g rg -f main.bicep -p @prod.bicepparam` and read the `details` array.

**Fix →** Adjust the resource properties to comply or fix the invalid value.

**Prevent →** Run `validate` in PR pipelines against an environment with the same policies.

</details>

### Q129. Terraform: `Error: A resource with the ID "/subscriptions/.../resourceGroups/rg-shop-dev" already exists - to be managed via Terraform this resource needs to be imported into the State.` 🟡

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

### Q130. Terraform: `Error: Error acquiring the state lock ... state blob is already locked`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another run holds the blob lease, or a crashed run left the lease.

**Diagnose →** The error shows the lock ID, who and when. Check running pipelines.

**Fix →**

1. Wait for the other run, or if it's dead: `terraform force-unlock <LOCK_ID>`
2. Or break the lease: `az storage blob lease break --container-name tfstate --blob-name shop/prod.tfstate --account-name sttfstate --auth-mode login`

**Prevent →** Pipeline concurrency limits per environment.

</details>

### Q131. Terraform azurerm v4: `Error: subscription ID could not be determined and was not specified`. 🟢

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

### Q132. `OperationNotAllowed: Operation is not allowed because there's an in progress update managed cluster operation` on an AKS change. 🟡

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

### Q133. `InvalidResourceLocation: The resource 'vnet-shop' already exists in location 'southindia' in resource group 'rg-shop'. A resource with the same name cannot be created in location 'centralindia'.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A resource with that name and type already exists in the RG in another region; names are unique per RG and type.

**Diagnose →** `az resource list -g rg-shop -n vnet-shop -o table`

**Fix →** Use region in names (`vnet-shop-cin`), or delete/move the old resource.

**Prevent →** Naming convention including region abbreviation.

</details>

### Q134. `InUseSubnetCannotBeDeleted: Subnet snet-app is in use by /subscriptions/.../networkInterfaces/... and cannot be deleted.` during destroy. 🟡

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

### Q135. `AllocationFailed` or `ZonalAllocationFailed: Allocation failed. We do not have sufficient capacity for the requested VM size in this zone.` 🟡

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

### Q136. Bicep build error `BCP037: The property "enableRbacAuthorisation" is not allowed on objects of type "VaultProperties".` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Typo or wrong property for the API version (here British spelling instead of `enableRbacAuthorization`).

**Diagnose →** Hover in VS Code, or check the API reference for the resource type and API version.

**Fix →** Correct the property name; update API versions if the property is newer.

**Prevent →** Bicep VS Code extension and `az bicep lint` in CI.

</details>

### Q137. `Conflict: Cannot delete resource group 'rg-shop-dev' while it contains resources protected by Azure Backup` or Recovery Services vault deletion fails. 🟡

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

### Q138. Terraform plan always shows a change on `tags` or a property even after apply. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Azure Policy Modify adds tags after apply, or Azure normalises a value (casing, defaults) differently from config.

**Diagnose →** Compare plan diff with Activity Log for policy-initiated writes.

**Fix →**

1. Add the policy-added tags to config, or `lifecycle { ignore_changes = [tags["created_by"]] }`
2. Match the exact casing Azure returns

**Prevent →** Coordinate tag policies with IaC defaults.

</details>

### Q139. `The resource operation completed with terminal provisioning state 'Failed'` for a VM extension (`VMExtensionProvisioningError`). 🟡

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

### Q140. `DeploymentQuotaExceeded: Creating the deployment 'shop-…' would exceed the quota of '800'.` 🔴

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

---

## Network Connectivity Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q141. A VM with a public IP times out on SSH (`ssh: connect to host 20.x.x.x port 22: Connection timed out`). 🟢

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

### Q142. `nslookup kv-shop.vault.azure.net` from inside the VNet returns a public IP even though a Private Endpoint exists. 🟡

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

### Q143. Key Vault returns `Client address is not authorized and caller is not a trusted service.` (ForbiddenByFirewall). 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Vault firewall is on (default action Deny or public access disabled), and the caller comes from a public IP not in the allow list or not through the Private Endpoint.

**Diagnose →** The error includes the client IP; `az keyvault show -n kv-shop --query properties.networkAcls`

**Fix →**

1. Access via the Private Endpoint (fix DNS so the client uses it)
2. Or add the caller's IP/VNet rule: `az keyvault network-rule add -n kv-shop --ip-address <ip>/32`

**Prevent →** Private access by design; avoid IP allow lists for CI (hosted runner IPs change).

</details>

### Q144. Application Gateway returns `502 Bad Gateway` and backend health shows Unhealthy. 🟡

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

### Q145. App Gateway v2 deployment fails or goes unhealthy after a UDR `0.0.0.0/0 → firewall` was added to its subnet. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** App Gateway v2 (public frontend) requires direct internet for control-plane traffic; sending default route to an NVA breaks it.

**Diagnose →** Check route table on `snet-agw`; App Gateway operational state and health.

**Fix →**

1. Remove the `0.0.0.0/0` route from the App Gateway subnet (keep specific routes to spokes if needed)
2. Or use the private-only App Gateway deployment, which supports forced tunnelling

**Prevent →** Exclude `snet-agw` from the "force-tunnel all subnets" policy.

</details>

### Q146. Two VNets peered but VMs can't communicate; peering state shows `Initiated`. 🟡

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

### Q147. Peering creation fails because address spaces overlap. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both VNets use overlapping CIDRs (often both `10.0.0.0/16` defaults).

**Diagnose →** Compare `addressSpace` of both VNets.

**Fix →**

1. Re-IP one VNet (add a new address space, migrate resources, remove old)
2. Or use Private Link Service / NAT-based connectivity if re-IP is impossible

**Prevent →** IPAM from day one; never use portal default ranges.

</details>

### Q148. VMs in a spoke lost internet access right after the network team added a route table. 🟢

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

### Q149. Azure Firewall logs show allowed DNAT but clients still time out to the service behind it. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Asymmetric routing: return traffic from the backend goes directly to the internet (or via a different path) instead of back through the firewall.

**Diagnose →** Effective routes on the backend subnet: is there a route for the client's return path to the firewall?

**Fix →**

1. Ensure the backend subnet's default route points to the firewall (so return traffic goes back through it)
2. If the backend uses a public LB, don't mix with DNAT on the firewall

**Prevent →** Draw the full in/out path when designing NVAs.

</details>

### Q150. Front Door returns `Our services aren't available right now` (error 503 / origin errors). 🟡

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

### Q151. Outbound calls from AKS intermittently fail with timeouts under load. Metrics show SNAT port usage at 100%. 🔴

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

### Q152. App Service can't reach a database through its Private Endpoint even though VNet integration is configured. 🟡

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

### Q153. Pods resolve external names slowly and some lookups fail with `i/o timeout`. 🔴

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

### Q154. VPN site-to-site tunnel is `Connected`, but no traffic flows from on-prem to Azure. 🟡

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

### Q155. Private Endpoint connection shows `Pending` and clients can't connect. 🟢

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

### Q156. Storage account calls from a VM fail with `AuthorizationFailure: This request is not authorized to perform this operation.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Storage firewall: default action Deny, VM's subnet not allowed (no service endpoint or PE), or public access disabled and client not using the private endpoint.

**Diagnose →** `az storage account show -n stshop --query networkRuleSet`; check DNS resolution of `stshop.blob.core.windows.net` from the VM.

**Fix →** Add a VNet rule (with `Microsoft.Storage` service endpoint on the subnet) or use Private Endpoint and fix DNS.

**Prevent →** Distinguish network errors (`AuthorizationFailure`) from RBAC errors (`AuthorizationPermissionMismatch`) in runbooks.

</details>

### Q157. Internal Load Balancer works from other VNets but not from VMs in its own backend pool. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Hairpinning is not supported: a backend VM calling the ILB frontend can be load-balanced to itself and the flow fails.

**Diagnose →** Calls from non-backend VMs succeed; from backend VMs fail intermittently.

**Fix →** Call the service locally or via another tier, or put a proxy in front; for AKS, call the ClusterIP Service instead of the ILB.

**Prevent →** Know the ILB hairpin limitation when designing.

</details>

### Q158. A VM can't reach `169.254.169.254` (IMDS) and managed identity calls fail. 🟡

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

### Q159. `Connection refused` when an App Service app calls an API on a VM in a peered VNet over port 8080. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Refused (not timeout) means the packet reached the host and nothing listens: wrong port, app bound to `127.0.0.1`, service down.

**Diagnose →** On the VM: `ss -tlnp | grep 8080`.

**Fix →** Bind the app to `0.0.0.0:8080`, start the service, correct the port.

**Prevent →** Readiness checks and documented ports.

</details>

### Q160. A Container Apps app with internal ingress can't be reached from a peered VNet. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No private DNS for the environment's default domain, so `<app>.<env-domain>` doesn't resolve to the environment's static IP from other VNets.

**Diagnose →** `az containerapp env show -n cae-shop -g rg --query "{domain:properties.defaultDomain, ip:properties.staticIp}"`; `nslookup` from the peered VNet.

**Fix →** Create a Private DNS zone for the default domain with a wildcard `*` A record to the static IP, linked to the peered VNets.

**Prevent →** Include this zone in the Container Apps landing template.

</details>

---

## AKS and ACR Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q161. Pods show `ImagePullBackOff` with `failed to authorize: failed to fetch anonymous token: unexpected status from GET request to https://acrshop.azurecr.io/oauth2/token?...: 401 Unauthorized`. 🟢

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

### Q162. `Failed to pull image "acrshop.azurecr.io/shop/web:v3": ... manifest unknown` or `not found`. 🟢

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

### Q163. `ImagePullBackOff` with `dial tcp: lookup acrshop.azurecr.io on 168.63.129.16:53: no such host` or connection timeouts to the registry. 🔴

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

### Q164. `exec /usr/local/bin/python: exec format error` in pod logs right after deploying a new image. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Image built for `arm64` (on an Apple Silicon Mac) running on `amd64` nodes, or the reverse.

**Diagnose →** `docker buildx imagetools inspect acrshop.azurecr.io/shop/web:v3` shows the platforms.

**Fix →** Build for the node architecture: `docker buildx build --platform linux/amd64 -t ... --push .` or use `az acr build` (amd64 by default, `--platform linux/arm64` for Arm pools).

**Prevent →** Build images in CI, not on laptops.

</details>

### Q165. Pods are `Pending`: `0/3 nodes are available: 3 Insufficient cpu. preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod.` 🟢

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

### Q166. Cluster autoscaler events show `pod didn't trigger scale-up: 1 max node group size reached` or `NotTriggerScaleUp`. 🟡

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

### Q167. AKS upgrade fails: `(UpgradeFailed) Drain node aks-apps-12345-vmss000002 failed when evicting pod web-7d9... failed with Too Many Requests error. This is often caused by a restrictive Pod Disruption Budget (PDB) policy.` 🟡

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

### Q168. `(InsufficientSubnetSize) Pre-allocated IPs 682 exceeds IPs available 251 in Subnet Cidr 10.20.1.0/24` when scaling or upgrading AKS. 🟡

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

### Q169. Service of type LoadBalancer stuck with `EXTERNAL-IP <pending>`; events show `Error syncing load balancer: failed to ensure load balancer: ... AuthorizationFailed ... 'Microsoft.Network/virtualNetworks/subnets/read'`. 🟡

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

### Q170. Pods with workload identity fail: `WorkloadIdentityCredential authentication failed ... AADSTS70021: No matching federated identity record found for presented assertion. Assertion Subject: 'system:serviceaccount:shop:default'.` 🟡

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

### Q171. Workload identity env vars (`AZURE_CLIENT_ID`, `AZURE_FEDERATED_TOKEN_FILE`) are missing in the pod. 🟡

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

### Q172. Pod stuck in `ContainerCreating` with `MountVolume.SetUp failed for volume "secrets-store-inline" : rpc error: code = Unknown desc = failed to mount secrets store objects for pod shop/web-..., err: ... failed to get objectType:secret, objectName:db-password ... StatusCode=403`. 🔴

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

### Q173. SecretProviderClass mount fails with `keyvault.BaseClient#GetSecret: Failure responding to request: StatusCode=404 ... SecretNotFound`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Secret name typo, wrong vault, or the secret was deleted (soft-deleted).

**Diagnose →** `az keyvault secret list --vault-name kv-shop -o table`; `az keyvault secret list-deleted --vault-name kv-shop`

**Fix →** Correct `objectName`/`keyvaultName`, or recover the secret: `az keyvault secret recover --vault-name kv-shop -n db-password`.

**Prevent →** Pipeline check that every object referenced by SecretProviderClasses exists.

</details>

### Q174. Rotated a Key Vault secret, but the app still uses the old value. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CSI rotation not enabled, or the app reads the value from an env var (synced Kubernetes Secret) only at startup.

**Diagnose →** `az aks show -g rg -n aks --query addonProfiles.azureKeyvaultSecretsProvider.config`; compare the mounted file contents with Key Vault.

**Fix →**

1. `az aks addon update -g rg -n aks -a azure-keyvault-secrets-provider --enable-secret-rotation --rotation-poll-interval 2m`
2. Read from the mounted file on each use, or trigger a rollout restart (Reloader or a pipeline step)

**Prevent →** Design apps to reload secrets from files.

</details>

### Q175. `az aks get-credentials` works but `kubectl` hangs then fails with `Unable to connect to the server: dial tcp 10.x.x.x:443: i/o timeout`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Private cluster; your machine has no network path to the private API endpoint (or DNS resolves to the private IP from outside).

**Diagnose →** `az aks show -g rg -n aks --query apiServerAccessProfile`

**Fix →** Connect from a VNet-connected machine (VPN, Bastion jump box, private agent) or use `az aks command invoke -g rg -n aks --command "kubectl get nodes"`.

**Prevent →** Document access paths for private clusters; private CI agents.

</details>

### Q176. `kubectl` returns `Unable to connect to the server: dial tcp 20.x.x.x:443: i/o timeout` on a public cluster from a new office network. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** API server authorized IP ranges don't include the new egress IP.

**Diagnose →** `az aks show -g rg -n aks --query apiServerAccessProfile.authorizedIpRanges`; `curl ifconfig.me`.

**Fix →** `az aks update -g rg -n aks --api-server-authorized-ip-ranges <existing>,<newIP>/32`

**Prevent →** Use stable egress IPs (VPN or NAT) for admin access.

</details>

### Q177. Nodes show `NotReady` after someone modified the node resource group (`MC_...`). 🔴

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

### Q178. A node pool upgrade fails with `QuotaExceeded` for surge nodes even though the cluster itself fits. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Surge nodes need extra vCPU quota temporarily.

**Diagnose →** Error shows required additional cores; `az vm list-usage`.

**Fix →** Lower `--max-surge` (for example `1`) or use `--max-unavailable 1`, or raise quota.

**Prevent →** Account for surge in quota planning.

</details>

### Q179. Pods are `OOMKilled` repeatedly (`Last State: Terminated, Reason: OOMKilled, Exit Code: 137`). 🟢

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

### Q180. After enabling Entra integration and disabling local accounts, the CI pipeline fails with `error: You must be logged in to the server (Unauthorized)`. 🟡

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

### Q181. Pods can't reach each other across namespaces after enabling network policy. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A default-deny policy without explicit allow rules for the needed cross-namespace flows (and DNS egress).

**Diagnose →** `kubectl get networkpolicy -A`; test with `kubectl exec ... -- wget -qO- http://api.backend.svc.cluster.local`; Hubble flows if ACNS is on.

**Fix →** Add policies allowing ingress from the caller namespace (`namespaceSelector` with `kubernetes.io/metadata.name`) and egress to kube-dns on 53.

**Prevent →** Template baseline policies (allow DNS, allow ingress controller) with default-deny.

</details>

### Q182. `az acr build` fails with `denied: client with IP '20.x.x.x' is not allowed access`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ACR firewall/public access restrictions; ACR Tasks agents run outside your network.

**Diagnose →** `az acr show -n acrshop --query "{public:publicNetworkAccess, rules:networkRuleSet, bypass:networkRuleBypassOptions}"`

**Fix →**

1. Allow trusted services (`--allow-trusted-services true`) and run tasks with a system identity, or use a dedicated **agent pool** in your VNet (`az acr agentpool create --subnet-id ...`, Premium)
2. Or build on private agents with `docker build` and push via the private endpoint

**Prevent →** Decide build location when locking down registries.

</details>

### Q183. `docker push` fails with `unauthorized: authentication required` after `az acr login` succeeded an hour ago. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ACR refresh token from `az acr login` expired (about 3 hours), or the identity has only `AcrPull`.

**Diagnose →** Re-run `az acr login -n acrshop`; check roles: `az role assignment list --assignee <id> --scope <acrId>`.

**Fix →** Log in again right before pushing; grant `AcrPush` for push.

**Prevent →** Log in within the same job step sequence as the push.

</details>

### Q184. Node pool stuck in `Failed` provisioning state after a scale operation. 🔴

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

### Q185. Container Insights shows no data for a new cluster. 🔴

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

---

## Storage, Key Vault, Monitoring and Pipeline Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q186. `AuthorizationPermissionMismatch: This request is not authorized to perform this operation using this permission.` reading blobs with `--auth-mode login`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The identity has control-plane roles (Owner/Contributor) but no Storage Blob Data role.

**Diagnose →** `az role assignment list --assignee <id> --scope <storageId> --include-inherited -o table`

**Fix →** Grant `Storage Blob Data Reader` or `Contributor` on the account or container; wait for propagation.

**Prevent →** Teach the control vs data plane difference; include data roles in IaC.

</details>

### Q187. A SAS URL returns `AuthenticationFailed: Server failed to authenticate the request ... Signature not valid in the specified time frame`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SAS expired, start time in the future (clock skew), or the user delegation key expired.

**Diagnose →** Inspect `st` and `se` parameters in the URL; compare with UTC now.

**Fix →** Generate a new SAS with start time a few minutes in the past and appropriate expiry.

**Prevent →** Short-lived SAS generated on demand by a service, not long-lived ones pasted into config.

</details>

### Q188. `PublicAccessNotPermitted: Public access is not permitted on this storage account.` when setting a container to public. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `allowBlobPublicAccess` is false on the account (often enforced by policy).

**Diagnose →** `az storage account show -n stshop --query allowBlobPublicAccess`

**Fix →** Don't make it public: serve through Front Door/CDN with private origin or use SAS. Only enable public access if it's truly intended and allowed.

**Prevent →** Keep the policy; document the CDN pattern for static content.

</details>

### Q189. `KeyBasedAuthenticationNotPermitted: Key based authentication is not permitted on this storage account.` in an older tool. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Shared key access disabled; the tool uses account keys or a service SAS.

**Diagnose →** `az storage account show -n stshop --query allowSharedKeyAccess`

**Fix →** Configure the tool for Entra auth (`--auth-mode login`, `DefaultAzureCredential`, AzCopy with `azcopy login --identity`), or user delegation SAS.

**Prevent →** Inventory key-based consumers before disabling shared keys.

</details>

### Q190. Application Insights shows no telemetry after deploying an app. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing or wrong `APPLICATIONINSIGHTS_CONNECTION_STRING`, SDK/distro not initialised early, egress blocked to ingestion endpoints, or local auth disabled on App Insights without Entra auth configured.

**Diagnose →** Check app settings/env; app logs for exporter errors; Live Metrics.

**Fix →** Set the connection string, initialise OpenTelemetry distro at startup, allow egress or AMPLS, configure managed identity auth if local auth is disabled.

**Prevent →** Health check in pipeline that queries `AppRequests` after deploy.

</details>

### Q191. A log search alert didn't fire during an incident even though the errors are visible in logs now. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Ingestion latency longer than the evaluation window, the query's time range too short, the threshold/aggregation wrong, or the alert disabled/muted by an alert processing rule.

**Diagnose →** Alert rule history; run the query with the same time range; check `ingestion_time() - TimeGenerated`.

**Fix →** Lengthen the window and frequency appropriately, fix the query, review processing rules.

**Prevent →** Use metric alerts for paging signals; test alerts with fault injection.

</details>

### Q192. Log Analytics costs spiked overnight. 🟡

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

### Q193. Azure DevOps: `##[error]No hosted parallelism has been purchased or granted.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** New organisation without free Microsoft-hosted parallel job grant, or private projects without purchased parallel jobs.

**Diagnose →** Organization settings → Parallel jobs.

**Fix →** Request the free grant via Microsoft's form, buy parallel jobs, or use a self-hosted agent pool.

**Prevent →** Plan agent capacity when creating new orgs.

</details>

### Q194. Azure DevOps: `There was a resource authorization issue: "The pipeline is not valid. Job Deploy: Step AzureCLI input connectedServiceNameARM references service connection arm-shop-prod which could not be found. The service connection does not exist, has been disabled or has not been authorized for use."` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pipeline not authorized to use the service connection, wrong name, or connection in another project.

**Diagnose →** Project settings → Service connections → the connection's Security tab and pipeline permissions.

**Fix →** Authorize the pipeline on the connection (click "Authorize resources" on the run, or add the pipeline in pipeline permissions), correct the name.

**Prevent →** Grant per pipeline rather than "all pipelines"; keep names consistent.

</details>

### Q195. An Azure Pipelines deployment job waits forever at `Waiting for approvals and checks`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pending approval, branch control check failing (run from a non-allowed branch), business hours check, or exclusive lock held by another run.

**Diagnose →** Open the run's checks panel; it lists each check status.

**Fix →** Approve, run from the allowed branch, or cancel the run holding the lock.

**Prevent →** Clear approval owners and timeouts on environment checks.

</details>

### Q196. `azure/webapps-deploy` fails with `Deployment Failed, Error: Failed to deploy web package to App Service. Conflict (CODE: 409)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another deployment in progress on the same app/slot, or the app is configured for Run From Package with a locked file system.

**Diagnose →** Kudu deployment logs (`https://<app>.scm.azurewebsites.net/api/deployments`).

**Fix →** Serialise deployments (concurrency group), retry; ensure consistent `WEBSITE_RUN_FROM_PACKAGE` settings.

**Prevent →** One deployment at a time per slot.

</details>

### Q197. App Service container fails: `Container shop-web_0_ab12 didn't respond to HTTP pings on port: 80, failing site start.` 🟢

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

### Q198. Azure Functions app shows `Azure Functions runtime is unreachable` after locking down the storage account. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Functions host can't reach `AzureWebJobsStorage` (storage firewall or disabled shared key) because the app isn't VNet-integrated or DNS for storage private endpoints (blob, queue, table, file) isn't resolving.

**Diagnose →** Check storage network rules, app's VNet integration, `WEBSITE_CONTENTOVERVNET`/content share settings, and DNS for all four storage sub-services.

**Fix →**

1. VNet integration with route all, Private Endpoints for blob/queue/table (and file if using content share), Private DNS zones linked
2. Use identity-based connection (`AzureWebJobsStorage__accountName`) with `Storage Blob Data Owner`, `Storage Queue Data Contributor` and related roles if shared keys are disabled

**Prevent →** Use the Functions private networking template; Flex Consumption handles identity-based storage cleanly.

</details>

### Q199. AKS metric alert on `node_cpu_usage_percentage` fires constantly for one node pool. 🟡

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

### Q200. `az keyvault secret set` fails with `(Forbidden) ... Caller is not authorized ... ForbiddenByRbac` for the engineer who just created the vault with Terraform. 🟢

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
