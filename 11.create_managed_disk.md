You created an empty managed disk `devops-disk` correctly using `az disk create`; the main issues were using the wrong size flag (`--size` instead of `--size-gb`) and the wrong parameter name (`--type` instead of `--sku`). [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/disk?view=azure-cli-latest)

Here’s a run book you can reuse for “Create a managed data disk with Azure CLI”.

***

## Objective

Create an empty managed disk named `devops-disk` in resource group `kml_rg_main-073f9f59cb014f62` with size 2 GB and SKU `Standard_LRS`, then verify that it exists and is in `Succeeded` state. [azure.microsoft](https://azure.microsoft.com/fr-fr/blog/azure-cli-managed-disks/)

***

## 1. Confirm resource group

From the Azure client host:

```bash
az group list -o table
```

You saw:

```text
Name                          Location    Status
----------------------------  ----------  ---------
kml_rg_main-073f9f59cb014f62  eastus      Succeeded
```

So the target RG exists and is ready. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/manage-azure-groups-azure-cli?view=azure-cli-latest)

***

## 2. Review az disk create syntax

Help output for `az disk create` shows: [linuxcommandlibrary](https://linuxcommandlibrary.com/man/az-disk)

- Name: `--name` or `-n`.  
- Resource group: `--resource-group` or `-g`.  
- Size in **GB**: `--size-gb` or `-z` (integer, no units).  
- Storage SKU: `--sku` (values like `Standard_LRS`, `Premium_LRS`, etc.).

The examples include:

```bash
az disk create -g MyResourceGroup -n MyDisk --size-gb 10
```

So the correct basic pattern for an empty disk is:

```bash
az disk create \
  --name <DISK_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --size-gb <INTEGER_GB> \
  --sku <SKU_NAME>
```

***

## 3. Initial attempts and errors

You first tried:

```bash
az disk create --name devops-disk --resource-group kml_rg_main-073f9f59cb014f62 --size 2GiB --type Standard_LRS
```

Errors:

- `--size` is not a valid parameter; it should be `--size-gb` (`-z`).  
- `2GiB` is invalid; `--size-gb` expects a plain integer (e.g., `2`).  
- `--type` is not the parameter name for storage SKU in `az disk create`; the correct flag is `--sku`. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/disk?view=azure-cli-latest)

Azure CLI reported:

```text
argument --size-gb/-z: invalid int value: '2GiB'
unrecognized arguments: --type Standard_LRS
```

which aligns with the documentation.

***

## 4. Correct disk creation command

You then used the correct command:

```bash
az disk create \
  --name devops-disk \
  --resource-group kml_rg_main-073f9f59cb014f62 \
  --size-gb 2 \
  --sku Standard_LRS
```

This matches the official examples: `--size-gb` for size and `--sku` for disk type. [azure.microsoft](https://azure.microsoft.com/fr-fr/blog/azure-cli-managed-disks/)

The resulting JSON showed:

```json
"diskSizeGb": 2,
"diskSizeBytes": 2147483648,
"diskState": "Unattached",
"location": "eastus",
"name": "devops-disk",
"sku": {
  "name": "Standard_LRS",
  "tier": "Standard"
},
"provisioningState": "Succeeded",
"type": "Microsoft.Compute/disks",
"creationData": {
  "createOption": "Empty"
}
```

This confirms an empty 2 GB Standard_LRS managed disk was created successfully in the specified RG and region. [learn.microsoft](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/tutorial-manage-disks)

***

## 5. Verify the disk exists

You checked:

```bash
az disk list --resource-group kml_rg_main-073f9f59cb014f62
```

Output included:

```json
{
  "name": "devops-disk",
  "diskSizeGB": 2,
  "diskState": "Unattached",
  "location": "eastus",
  "sku": {
    "name": "Standard_LRS",
    "tier": "Standard"
  },
  "provisioningState": "Succeeded",
  "type": "Microsoft.Compute/disks"
}
```

So the disk:

- Exists in the correct RG.  
- Has the expected size and SKU.  
- Is `Unattached` (ready to be attached to a VM later).  
- Is in `Succeeded` provisioning state. [learn.microsoft](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/tutorial-manage-disks)

***

## 6. Reusable steps for future disk creation tasks

You can reuse this pattern whenever you need a simple data disk:

1. Confirm RG:

   ```bash
   az group list -o table
   ```

2. Create an empty managed disk:

   ```bash
   az disk create \
     --name <DISK_NAME> \
     --resource-group <RESOURCE_GROUP> \
     --size-gb <SIZE_IN_GB> \
     --sku Standard_LRS
   ```

   Examples:

   ```bash
   az disk create -g kml_rg_main-073f9f59cb014f62 -n devops-disk --size-gb 2 --sku Standard_LRS
   ```

3. Verify:

   ```bash
   az disk list --resource-group <RESOURCE_GROUP> -o table
   ```

For more advanced scenarios (import from blob, copy existing disk, or create from snapshot), use the `--source` flag as shown in the official examples. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/disk?view=azure-cli-latest)

If you’d like, next I can add a short section to this run book on **attaching `devops-disk` to a VM** (using `az vm disk attach`), so you have the full lifecycle from create to attach.