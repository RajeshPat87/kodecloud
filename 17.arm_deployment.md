The OTS output you shared shows that everything ran exactly as required: the ARM template was modified, deployed to the correct resource group, and the resulting VNet has the right name, address space, and tags.

Below is a structured run book and documentation you can keep as your reference.

***

## Objective

Modify the existing ARM template `vnet-deployment-template.json` so that it deploys a virtual network:

- Name: `arm-vnet-nautilus`  
- Address space: `192.168.0.0/16`  
- Tags:
  - `displayName = arm-vnet-nautilus`
  - `Environment = KKE-nautilus`

Then deploy it to the lab resource group using Azure CLI and verify the VNet.

***

## Environment and Files

- Host: `azure-client`
- User: `root`
- Working directory: `/root`
- Template directory: `/root/arm-templates`
- Template file: `/root/arm-templates/vnet-deployment-template.json`
- Script file: `/root/arm-vnet-nautilus-ots.sh`
- Subscription: `Azure Free Labs`
- Resource group discovered: `kml_rg_main-56e63e1adf724def`
- VNet after deployment: `arm-vnet-nautilus`

***

## High-Level Steps

1. Verify Azure CLI authentication.  
2. Discover the correct resource group.  
3. List existing VNets before any changes.  
4. Use a one-time script (OTS) to:
   - Update the ARM template JSON (name, tags, addressPrefixes).
   - Deploy the updated template to the resource group.
   - List VNets after deployment.
   - Show detailed properties of `arm-vnet-nautilus`.  
5. Confirm the expected values.

***

## Step-by-Step Run Book

### Step 1 – Verify authentication

From the shell on `azure-client`:

```bash
az account show
```

You should see output similar to:

```json
{
  "name": "Azure Free Labs",
  "state": "Enabled",
  "user": {
    "type": "servicePrincipal"
  }
}
```

This confirms the CLI is authenticated and ready for deployments.

***

### Step 2 – Locate the ARM template

Change to the ARM template directory:

```bash
cd /root/arm-templates
ls
```

Confirm `vnet-deployment-template.json` is present.

***

### Step 3 – Discover the lab resource group

From any directory (e.g. `/root`):

```bash
az group list --query '[].name' -o tsv | grep 'k'
```

In your run, this returned:

```text
kml_rg_main-56e63e1adf724def
```

This is the resource group used for the deployment.

***

### Step 4 – Prepare the One-Time Script

Create the script file:

```bash
cd /root
nano arm-vnet-nautilus-ots.sh
```

Paste:

```bash
#!/usr/bin/env bash
set -euo pipefail

TEMPLATE_DIR="/root/arm-templates"
TEMPLATE_FILE="$TEMPLATE_DIR/vnet-deployment-template.json"
TMP_FILE="$TEMPLATE_DIR/vnet-deployment-template.tmp.json"

NEW_VNET_NAME="arm-vnet-nautilus"
NEW_ADDRESS_PREFIX="192.168.0.0/16"
NEW_ENV_TAG="KKE-nautilus"

cd "$TEMPLATE_DIR"

echo "Finding resource group..."
RG_NAME="$(az group list --query '[].name' -o tsv | grep 'k' | head -n 1)"

if [ -z "$RG_NAME" ]; then
  echo "ERROR: Resource group not found."
  exit 1
fi

echo "Resource Group: $RG_NAME"
echo

echo "=== Existing VNets before deployment ==="
az network vnet list \
  --resource-group "$RG_NAME" \
  --query "[].{name:name,addressPrefixes:addressSpace.addressPrefixes,tags:tags}" \
  -o table

echo
echo "=== Updating ARM template JSON ==="

jq --arg vnetName "$NEW_VNET_NAME" \
   --arg addr "$NEW_ADDRESS_PREFIX" \
   --arg envTag "$NEW_ENV_TAG" \
   '.resources |= map(
     if .type == "Microsoft.Network/virtualNetworks" then
       .name = $vnetName
       | .tags = ((.tags // {}) + {
           "displayName": $vnetName,
           "Environment": $envTag
         })
       | .properties.addressSpace.addressPrefixes = [$addr]
     else
       .
     end
   )
   ' "$TEMPLATE_FILE" > "$TMP_FILE"

mv "$TMP_FILE" "$TEMPLATE_FILE"

echo "Updated template preview:"
cat "$TEMPLATE_FILE"

echo
echo "=== Deploying ARM template ==="
DEPLOY_NAME="arm-vnet-nautilus-deployment"

az deployment group create \
  --resource-group "$RG_NAME" \
  --name "$DEPLOY_NAME" \
  --template-file "$TEMPLATE_FILE"

echo
echo "=== VNets after deployment ==="
az network vnet list \
  --resource-group "$RG_NAME" \
  --query "[].{name:name,addressPrefixes:addressSpace.addressPrefixes,tags:tags}" \
  -o table

echo
echo "=== Verification of target VNet ==="
az network vnet show \
  --resource-group "$RG_NAME" \
  --name "$NEW_VNET_NAME" \
  --query "{name:name,addressPrefixes:addressSpace.addressPrefixes,tags:tags}" \
  -o json

echo
echo "OTS completed successfully."
```

Save and exit.

Make the script executable and run it:

```bash
chmod +x arm-vnet-nautilus-ots.sh
./arm-vnet-nautilus-ots.sh
```

***

### Step 5 – What the script did in your run

Your actual output shows the entire flow:

1. **Resource group detection**

```text
Finding resource group...
Resource Group: kml_rg_main-56e63e1adf724def
```

2. **VNets before deployment**

No VNets were listed (empty section), meaning this VNet was the first one.

3. **ARM template JSON update**

The preview printed:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {},
  "functions": [],
  "variables": {},
  "resources": [
    {
      "name": "arm-vnet-nautilus",
      "type": "Microsoft.Network/virtualNetworks",
      "apiVersion": "2023-11-01",
      "location": "[resourceGroup().location]",
      "tags": {
        "displayName": "arm-vnet-nautilus",
        "Environment": "KKE-nautilus"
      },
      "properties": {
        "addressSpace": {
          "addressPrefixes": [
            "192.168.0.0/16"
          ]
        }
      }
    }
  ],
  "outputs": {}
}
```

This confirms the JSON logic correctly changed:

- `name` → `arm-vnet-nautilus`  
- `tags.displayName` → `arm-vnet-nautilus`  
- `tags.Environment` → `KKE-nautilus`  
- `addressSpace.addressPrefixes` → `["192.168.0.0/16"]`

4. **Deployment**

The deployment object showed:

- `provisioningState`: `"Succeeded"`  
- `mode`: `"Incremental"`  
- `outputResources` included:

  ```json
  {
    "id": "/subscriptions/.../resourceGroups/kml_rg_main-56e63e1adf724def/providers/Microsoft.Network/virtualNetworks/arm-vnet-nautilus",
    "resourceGroup": "kml_rg_main-56e63e1adf724def"
  }
  ```

5. **VNets after deployment**

```text
=== VNets after deployment ===
Name
-----------------
arm-vnet-nautilus
```

6. **Verification of target VNet**

```json
{
  "addressPrefixes": [
    "192.168.0.0/16"
  ],
  "name": "arm-vnet-nautilus",
  "tags": {
    "Environment": "KKE-nautilus",
    "displayName": "arm-vnet-nautilus"
  }
}
```

That matches the lab requirements exactly.

***

## Validation Checklist

Use this checklist to confirm the task is complete:

- [x] ARM template `vnet-deployment-template.json` has:
  - VNet `name = arm-vnet-nautilus`
  - Tag `displayName = arm-vnet-nautilus`
  - Tag `Environment = KKE-nautilus`
  - `addressSpace.addressPrefixes = ["192.168.0.0/16"]`
- [x] Azure CLI deployment succeeded with `provisioningState = "Succeeded"`.
- [x] The VNet `arm-vnet-nautilus` exists in `kml_rg_main-56e63e1adf724def`.
- [x] `az network vnet show` confirms:
  - `addressPrefixes` includes `192.168.0.0/16`
  - Tags include both `Environment` and `displayName` with correct values.

***

## Documentation Summary (OT-style)

- **Purpose:** Automate modification and deployment of a VNet ARM template in a lab environment.
- **Script name:** `arm-vnet-nautilus-ots.sh`
- **Inputs:** None (resource group auto-discovered from `az group list`, template file path fixed).
- **Outputs:** VNet `arm-vnet-nautilus` with required address space and tags.
- **Execution:** One-time, idempotent for this template and resource group (running again will re-deploy same VNet configuration).
- **Rollback:** Not required in this lab; re-running with different values would update the VNet.

If you’d like, I can also produce a short “lab answer” version (3–4 command snippets) you can paste into the KodeKloud submission box.