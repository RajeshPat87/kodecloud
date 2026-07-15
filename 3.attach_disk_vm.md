# Azure Run Book: Attach Existing Managed Disk to VM

## Objective
Attach the existing managed disk `xfusion-disk` to the existing virtual machine `xfusion-vm` as a data disk in the resource group `kml_rg_main-2b857f94c79b44cd`, then verify that the attachment completed successfully.[1][2]

## Existing resources
The resource group listing shows that both the target VM and the target managed disk already exist in `southcentralus`.[3]

| Resource name | Type | Location |
|---|---|---|
| `xfusion-vm` | `Microsoft.Compute/virtualMachines` | `southcentralus` |
| `xfusion-disk` | `Microsoft.Compute/disks` | `southcentralus` |

## Preconditions
- Azure login is already active on the `azure-client` host.[2]
- The resource group `kml_rg_main-2b857f94c79b44cd` exists and contains both `xfusion-vm` and `xfusion-disk`.[3]
- The VM initialization should be complete before the task is submitted, so the VM should be in a stable running state before and after disk attachment.[1]

## Pre-checks
### Verify the VM exists and is running
Run:

```bash
az vm show \
  --resource-group kml_rg_main-2b857f94c79b44cd \
  --name xfusion-vm \
  --show-details \
  --query "{name:name,powerState:powerState,location:location}" \
  --output table
```

Expected result: the VM `xfusion-vm` should be returned, and the power state should normally be `VM running` before proceeding.[2]

### Verify the disk exists and is currently unattached
Run:

```bash
az disk show \
  --resource-group kml_rg_main-2b857f94c79b44cd \
  --name xfusion-disk \
  --query "{name:name,location:location,managedBy:managedBy}" \
  --output table
```

Expected result: the disk should exist in `southcentralus`, and `managedBy` should be empty if it is not already attached to a VM.[1][4]

### Verify current data disks on the VM
Run:

```bash
az vm show \
  --resource-group kml_rg_main-2b857f94c79b44cd \
  --name xfusion-vm \
  --query "storageProfile.dataDisks[].name" \
  --output table
```

Expected result: `xfusion-disk` should not already appear in the data disk list before attachment.[4]

## Procedure
### Step 1: Get the managed disk resource ID
Run:

```bash
DISK_ID=$(az disk show \
  --resource-group kml_rg_main-2b857f94c79b44cd \
  --name xfusion-disk \
  --query id \
  --output tsv)
```

This retrieves the Azure resource ID of the existing managed disk, which can then be attached to the VM.[1]

Optional confirmation:

```bash
echo "$DISK_ID"
```

### Step 2: Attach the existing disk to the VM
Run:

```bash
az vm disk attach \
  --resource-group kml_rg_main-2b857f94c79b44cd \
  --vm-name xfusion-vm \
  --name "$DISK_ID"
```

Microsoft documents that existing managed disks can be attached by first retrieving the disk ID and then passing that ID to `az vm disk attach`.[1][2]

## Verification
### Verify the disk is attached from the VM view
Run:

```bash
az vm show \
  --resource-group kml_rg_main-2b857f94c79b44cd \
  --name xfusion-vm \
  --query "storageProfile.dataDisks[].name" \
  --output table
```

Expected result: `xfusion-disk` should now appear in the list of data disks attached to `xfusion-vm`.[4]

### Verify the disk is attached from the disk view
Run:

```bash
az disk show \
  --resource-group kml_rg_main-2b857f94c79b44cd \
  --name xfusion-disk \
  --query managedBy \
  --output tsv
```

Expected result: this should return the Azure resource ID of `xfusion-vm`, confirming the disk is attached to that VM.[4]

### Verify VM power state after attachment
Run:

```bash
az vm show \
  --resource-group kml_rg_main-2b857f94c79b44cd \
  --name xfusion-vm \
  --show-details \
  --query "{name:name,powerState:powerState}" \
  --output table
```

Expected result: the VM should still be accessible and normally remain in `VM running` state after the disk attachment operation.[2]

## Optional guest OS validation
Azure confirms attachment at the platform level, but actual use inside the VM depends on the guest operating system. For Linux VMs, commands such as `lsblk` can be used after logging into the VM to confirm the disk is visible inside the OS.[1][4]

Example:

```bash
lsblk
```

If the task validator only checks Azure-side attachment, the Azure CLI verification commands above are usually sufficient.[4]

## Troubleshooting
### Disk already attached
If `managedBy` already returns a VM resource ID before the attach step, the disk is already in use and should not be attached again until detached from the current VM.[4]

### Authorization errors
If the command returns `AuthorizationFailed`, the current identity lacks permission to update VM or disk resources. In Azure RBAC, attaching a disk requires permission to modify the VM and reference the disk resource.[2]

### Region mismatch
The listed resources already show both `xfusion-vm` and `xfusion-disk` in `southcentralus`, which is the correct placement for attachment compatibility in this task.[3]

## Completion criteria
The task is complete when all of the following are true:[1][4]
- `xfusion-disk` appears in `storageProfile.dataDisks[]` for `xfusion-vm`.
- `az disk show --name xfusion-disk --query managedBy -o tsv` returns the VM resource ID.
- The VM remains initialized and in a healthy running state before submission.

## Command summary
```bash
az vm show --resource-group kml_rg_main-2b857f94c79b44cd --name xfusion-vm --show-details --query "{name:name,powerState:powerState,location:location}" -o table
az disk show --resource-group kml_rg_main-2b857f94c79b44cd --name xfusion-disk --query "{name:name,location:location,managedBy:managedBy}" -o table
az vm show --resource-group kml_rg_main-2b857f94c79b44cd --name xfusion-vm --query "storageProfile.dataDisks[].name" -o table
DISK_ID=$(az disk show --resource-group kml_rg_main-2b857f94c79b44cd --name xfusion-disk --query id -o tsv)
az vm disk attach --resource-group kml_rg_main-2b857f94c79b44cd --vm-name xfusion-vm --name "$DISK_ID"
az vm show --resource-group kml_rg_main-2b857f94c79b44cd --name xfusion-vm --query "storageProfile.dataDisks[].name" -o table
az disk show --resource-group kml_rg_main-2b857f94c79b44cd --name xfusion-disk --query managedBy -o tsv
```