You successfully added the `Environment=dev` tag to `xfusion-vm` in `kml_rg_main-14ab7aa827e64185` using Azure CLI; the run book below documents the full flow and the common CLI pitfalls you hit. [azurelessons](https://azurelessons.com/az-account-list/)

***

## Objective

Add the tag `Environment=dev` to the existing VM `xfusion-vm` in resource group `kml_rg_main-14ab7aa827e64185` and verify that the tag is present on the VM. [youtube](https://www.youtube.com/watch?v=I_MsWLmOqzI)

***

## Step 1 – Confirm Azure account and subscription

1. List accounts:

   ```bash
   az account list
   ```

   You saw:

   ```json
   {
     "cloudName": "AzureCloud",
     "homeTenantId": "54c1a2d3-d100-453c-9636-3a109eb45552",
     "id": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b",
     "isDefault": true,
     "name": "Azure Free Labs",
     "state": "Enabled",
     "tenantId": "54c1a2d3-d100-453c-9636-3a109eb45552",
     "user": {
       "name": "1fd4cd48-9bd6-4c11-bff7-a9f4df551d5e",
       "type": "servicePrincipal"
     }
   }
   ```

   This confirms you’re using the `Azure Free Labs` subscription (ID `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b`) with a service principal. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/manage-azure-subscriptions-azure-cli?view=azure-cli-latest)

***

## Step 2 – Confirm resource group and VM exists

2. List resource groups:

   ```bash
   az group list
   ```

   Output included:

   ```json
   {
     "name": "kml_rg_main-14ab7aa827e64185",
     "location": "eastus",
     "properties": {
       "provisioningState": "Succeeded"
     }
   }
   ```

   So `kml_rg_main-14ab7aa827e64185` is the lab RG. [github](https://github.com/Azure/CloudShell/issues/39)

3. List resources in that RG:

   ```bash
   az resource list --resource-group kml_rg_main-14ab7aa827e64185 -o table
   ```

   Output showed:

   ```text
   Name                    ResourceGroup                 Location    Type
   ----------------------  ----------------------------  ----------  ---------------------------------------
   xfusion-vm              kml_rg_main-14ab7aa827e64185  centralus   Microsoft.Compute/virtualMachines
   ...
   ```

   Confirming the VM `xfusion-vm` exists in region `centralus`. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/resource?view=azure-cli-latest)

> Note: The bare `az resource --resource-group ...` commands failed because they were missing the `list` subcommand; Azure CLI correctly pointed you to the docs. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/use-azure-cli-successfully-troubleshooting?view=azure-cli-latest)

***

## Step 3 – Add tag Environment=dev to xfusion-vm

4. Use `az resource tag` to add the tag:

   ```bash
   az resource tag \
     --resource-group kml_rg_main-14ab7aa827e64185 \
     --name xfusion-vm \
     --resource-type "Microsoft.Compute/virtualMachines" \
     --tags Environment=dev
   ```

   The output you got included:

   ```json
   "tags": {
     "Environment": "dev"
   },
   "type": "Microsoft.Compute/virtualMachines",
   "name": "xfusion-vm",
   "resourceGroup": "kml_rg_main-14ab7aa827e64185",
   "location": "centralus"
   ```

   This confirms the tag has been applied successfully. [learn.microsoft](https://learn.microsoft.com/bs-latn-ba/previous-versions/azure/virtual-machines/tag-powershell)

If you want to *add* a tag without clearing existing tags, you can use the incremental mode (`-i`):

```bash
az resource tag \
  --resource-group kml_rg_main-14ab7aa827e64185 \
  --name xfusion-vm \
  --resource-type "Microsoft.Compute/virtualMachines" \
  --tags Environment=dev \
  -i
```

This appends/updates tags instead of replacing them. [youtube](https://www.youtube.com/watch?v=I_MsWLmOqzI)

***

## Step 4 – Verify the tag on the VM

5. Show VM details and inspect tags:

   ```bash
   az vm show \
     --name xfusion-vm \
     --resource-group kml_rg_main-14ab7aa827e64185 \
     --query "tags" \
     -o table
   ```

   Expected output:

   ```text
   Environment
   -----------
   dev
   ```

   This confirms `Environment=dev` is now attached to the VM. [learn.microsoft](https://learn.microsoft.com/bs-latn-ba/previous-versions/azure/virtual-machines/tag-powershell)

If you see an error like the last incomplete command in your log:

```bash
az vm show \
  --name xfusion-vm \
  --resource-group kml_rg_main-ac56ece0bc6d4023
```

ensure:

- The resource group name matches the one that actually contains `xfusion-vm` (`kml_rg_main-14ab7aa827e64185` in this task, not `kml_rg_main-ac56ece0bc6d4023`).  
- The command is complete (e.g., add `-o table` or `--query "tags"`), though these flags are optional. [stackoverflow](https://stackoverflow.com/questions/55480376/how-to-fix-az-cli-error-the-resource-under-resource-group-was-not-fou)

***

## Common CLI pitfalls and corrections

- `az resource --resource-group ...`  
  Must be `az resource list --resource-group ...` to list resources. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/use-azure-cli-successfully-troubleshooting?view=azure-cli-latest)

- Wrong RG for `az vm show`  
  If you use an RG that doesn’t contain the VM, you’ll get “Resource ... was not found” errors; always cross-check with `az vm list --resource-group <RG> -o table` first. [github](https://github.com/Azure/CloudShell/issues/39)

- Tagging multiple times  
  `az resource tag --tags ...` without `-i` replaces the whole tag set. Use `-i` when you want to keep existing tags. [youtube](https://www.youtube.com/watch?v=I_MsWLmOqzI)

***

## Command summary for this task

```bash
# Confirm subscription
az account list -o table

# Confirm resource group
az group list -o table

# List resources in RG
az resource list --resource-group kml_rg_main-14ab7aa827e64185 -o table

# Add tag Environment=dev to xfusion-vm
az resource tag \
  --resource-group kml_rg_main-14ab7aa827e64185 \
  --name xfusion-vm \
  --resource-type "Microsoft.Compute/virtualMachines" \
  --tags Environment=dev

# (Optional incremental tagging)
az resource tag \
  --resource-group kml_rg_main-14ab7aa827e64185 \
  --name xfusion-vm \
  --resource-type "Microsoft.Compute/virtualMachines" \
  --tags Environment=dev \
  -i

# Verify tags on VM
az vm show \
  --name xfusion-vm \
  --resource-group kml_rg_main-14ab7aa827e64185 \
  --query "tags" \
  -o table
```

Do you also want portal-based steps (Tags blade on the VM) written out in run-book style for the same Environment=dev tag task?  