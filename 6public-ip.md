# Azure Public IP Allocation Run Book

## Objective
Allocate an Azure Public IP address named `devops-pip` in the resource group `kml_rg_main-bd7c70c775f74d18` using the Azure portal or Azure CLI, and verify that the resource was created successfully.[cite:54][cite:55][cite:65]

## Environment
- Portal URL: [https://portal.azure.com](https://portal.azure.com/)
- Resource group: `kml_rg_main-bd7c70c775f74d18`
- Target Public IP name: `devops-pip`
- Subscription: `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b` (based on the resource group link provided in the task context)
- Azure CLI host: `azure-client`[cite:54][cite:55]

## Background
Azure portal resource creation commonly triggers an Azure Resource Manager (ARM) deployment behind the scenes. This means a resource created through the GUI is still deployed by Azure Resource Manager, using the same control plane that Azure CLI and templates use.[cite:146][cite:150][cite:154]

A resource group can exist in one region while contained resources are created in another supported region. The resource group location stores metadata for the group, while each resource keeps its own `location` property.[cite:68][cite:71][cite:75]

## Preconditions
- Valid Azure credentials are available for the lab session.[cite:54]
- Access to the Azure portal is working.[cite:65]
- The resource group `kml_rg_main-bd7c70c775f74d18` already exists.[cite:68]
- The session time window is still valid, because lab credentials are time-bound by design.[cite:54]

## Pre-checks
### Verify the resource group exists
Run the following command on `azure-client`:

```bash
az group show \
  --name kml_rg_main-bd7c70c775f74d18 \
  --output table
```

Expected result: the resource group should be returned with `Succeeded` state and a valid location.[cite:68]

### Check whether the Public IP already exists

```bash
az network public-ip list \
  --resource-group kml_rg_main-bd7c70c775f74d18 \
  --query "[?name=='devops-pip']" \
  --output table
```

Expected result: no rows should be returned before creation. If `devops-pip` already exists, do not recreate it.[cite:55][cite:130]

## Procedure using Azure Portal
1. Sign in to [Azure Portal](https://portal.azure.com/).
2. Open **Public IP addresses** from the portal search bar.[cite:65]
3. Click **Create**.[cite:65]
4. Set the following values:
   - Subscription: lab subscription
   - Resource group: `kml_rg_main-bd7c70c775f74d18`
   - Name: `devops-pip`
   - Region: use the region required by the task or the region intended for the dependent resource.[cite:65][cite:77]
   - IP version: `IPv4`
   - SKU: `Basic` unless the task requires `Standard`.[cite:65]
   - Assignment: `Dynamic` unless the task requires `Static`.[cite:65]
5. Click **Review + create** and then **Create**.[cite:65][cite:130]
6. Wait for the deployment status to show **Succeeded**.[cite:150][cite:154]

## Procedure using Azure CLI
If the lab identity has sufficient permissions, the Public IP can also be created with Azure CLI.[cite:54][cite:55]

```bash
az network public-ip create \
  --name devops-pip \
  --resource-group kml_rg_main-bd7c70c775f74d18 \
  --location centralus \
  --allocation-method Dynamic
```

If `Standard` and `Static` are required, use:

```bash
az network public-ip create \
  --name devops-pip \
  --resource-group kml_rg_main-bd7c70c775f74d18 \
  --location centralus \
  --sku Standard \
  --allocation-method Static
```

If the command returns `AuthorizationFailed`, the current identity lacks the required RBAC permission such as `Microsoft.Network/publicIPAddresses/write`.[cite:78][cite:82][cite:89]

## Verification
### Verify the specific Public IP

```bash
az network public-ip show \
  --name devops-pip \
  --resource-group kml_rg_main-bd7c70c775f74d18 \
  --output table
```

This confirms that `devops-pip` exists and shows its location, SKU, allocation method, and current address details.[cite:55][cite:130]

### List all Public IPs in the resource group

```bash
az network public-ip list \
  --resource-group kml_rg_main-bd7c70c775f74d18 \
  --output table
```

This confirms that `devops-pip` is present in the correct resource group.[cite:55][cite:129][cite:141]

### Print only the assigned IP address

```bash
az network public-ip show \
  --name devops-pip \
  --resource-group kml_rg_main-bd7c70c775f74d18 \
  --query ipAddress \
  --output tsv
```

This is useful when the lab validator or a downstream task needs only the public IPv4 value.[cite:55][cite:61]

## Troubleshooting
### AuthorizationFailed during CLI creation
If Azure CLI returns `AuthorizationFailed`, the signed-in identity does not have permission to create Public IPs in the target scope.[cite:78][cite:82][cite:89]

Typical causes:
- Restricted service principal credentials.[cite:82][cite:89]
- Missing RBAC rights such as `Contributor` or `Network Contributor` at the resource group or subscription scope.[cite:78][cite:82][cite:88]
- Missing permission to assign roles, which requires `Microsoft.Authorization/roleAssignments/write` and is normally available only to higher-privilege roles.[cite:104][cite:110][cite:112]

### Why the portal worked when CLI failed
The portal session may have been authenticated as a different identity with broader permissions. Also, portal creation usually submits an ARM deployment, which explains why the resource appeared under **Deployments** in the resource group.[cite:146][cite:150][cite:154]

## Completion criteria
This task is complete when all of the following are true:[cite:54][cite:55][cite:65]
- A Public IP named `devops-pip` exists.
- It is in resource group `kml_rg_main-bd7c70c775f74d18`.
- The deployment status is successful.
- `az network public-ip show` returns the resource successfully.
- `az network public-ip list` shows `devops-pip` in the target resource group.

## Command summary
```bash
az group show --name kml_rg_main-bd7c70c775f74d18 -o table
az network public-ip list --resource-group kml_rg_main-bd7c70c775f74d18 --query "[?name=='devops-pip']" -o table
az network public-ip create --name devops-pip --resource-group kml_rg_main-bd7c70c775f74d18 --location centralus --allocation-method Dynamic
az network public-ip show --name devops-pip --resource-group kml_rg_main-bd7c70c775f74d18 -o table
az network public-ip list --resource-group kml_rg_main-bd7c70c775f74d18 -o table
az network public-ip show --name devops-pip --resource-group kml_rg_main-bd7c70c775f74d18 --query ipAddress -o tsv
```
