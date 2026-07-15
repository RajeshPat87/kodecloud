# Azure Run Book: Attach Existing NIC to VM

## Objective
Attach the existing network interface `xfusion-nic` to the existing virtual machine `xfusion-vm` in the resource group `kml_rg_main-045b4bdee4654a6b`, and verify that the NIC status is attached before submitting the task.[1][2]

## Existing resources
The resource listing provided for the resource group confirms that both `xfusion-vm` and `xfusion-nic` already exist in `southcentralus`.[3]

| Resource name | Type | Location |
|---|---|---|
| `xfusion-vm` | `Microsoft.Compute/virtualMachines` | `southcentralus` |
| `xfusion-nic` | `Microsoft.Network/networkInterfaces` | `southcentralus` |
| `xfusion-vmVMNic` | `Microsoft.Network/networkInterfaces` | `southcentralus` |

## Preconditions
- Azure login is already available on the `azure-client` host using the provided credentials.[2]
- Resource group `kml_rg_main-045b4bdee4654a6b` exists and contains the VM and NIC needed for the task.[3]
- The VM should be fully initialized before submission, and NIC changes on an existing Azure VM generally require the VM to be stopped or deallocated before the network interface is added.[1][4]
- The VM size must support multiple NICs if the VM already has one primary NIC attached.[5][1]

## Pre-checks
### Verify the VM exists and check power state
Run:

```bash
az vm show \
  --resource-group kml_rg_main-045b4bdee4654a6b \
  --name xfusion-vm \
  --show-details \
  --query "{name:name,powerState:powerState,location:location,vmSize:hardwareProfile.vmSize}" \
  --output table
```

Expected result: the VM should exist in `southcentralus`, and the current power state should be visible before proceeding.[2][6]

### Verify the NIC exists and is not already attached elsewhere
Run:

```bash
az network nic show \
  --resource-group kml_rg_main-045b4bdee4654a6b \
  --name xfusion-nic \
  --query "{name:name,location:location,virtualMachine:id,provisioningState:provisioningState}" \
  --output table
```

Expected result: `xfusion-nic` should exist in `southcentralus`. If the NIC is unattached, the `virtualMachine` field should be empty. If a VM ID already appears, the NIC is already attached.[1][7]

### Verify current NICs attached to the VM
Run:

```bash
az vm nic list \
  --resource-group kml_rg_main-045b4bdee4654a6b \
  --vm-name xfusion-vm \
  --output table
```

Expected result: this lists the NICs already attached to `xfusion-vm`, which helps confirm whether `xfusion-nic` still needs to be added.[2][8]

## Procedure
### Step 1: Deallocate the VM
Azure documentation and examples show that adding a NIC to an existing VM should be done while the VM is stopped or deallocated.[1][4]

Run:

```bash
az vm deallocate \
  --resource-group kml_rg_main-045b4bdee4654a6b \
  --name xfusion-vm
```

Wait until the command completes successfully.[4]

### Step 2: Attach the NIC to the VM
Run:

```bash
az vm nic add \
  --resource-group kml_rg_main-045b4bdee4654a6b \
  --vm-name xfusion-vm \
  --nics xfusion-nic
```

This adds the existing NIC `xfusion-nic` to the VM `xfusion-vm`.[2][4]

### Step 3: Start the VM again
Run:

```bash
az vm start \
  --resource-group kml_rg_main-045b4bdee4654a6b \
  --name xfusion-vm
```

This returns the VM to running state so that initialization is complete before task submission.[6][4]

## Verification
### Verify the NIC is attached from the VM side
Run:

```bash
az vm nic list \
  --resource-group kml_rg_main-045b4bdee4654a6b \
  --vm-name xfusion-vm \
  --output table
```

Expected result: `xfusion-nic` should appear in the VM NIC list, confirming attachment.[2][8]

### Verify the NIC is attached from the NIC side
Run:

```bash
az network nic show \
  --resource-group kml_rg_main-045b4bdee4654a6b \
  --name xfusion-nic \
  --query "{name:name,attachedVm:virtualMachine.id,provisioningState:provisioningState}" \
  --output table
```

Expected result: `attachedVm` should contain the Azure resource ID of `xfusion-vm`, which confirms the NIC status is attached.[1][7]

### Verify the VM is back in running state
Run:

```bash
az vm show \
  --resource-group kml_rg_main-045b4bdee4654a6b \
  --name xfusion-vm \
  --show-details \
  --query "{name:name,powerState:powerState}" \
  --output table
```

Expected result: the VM should show `VM running` before the task is submitted.[2][4]

## Troubleshooting
### NIC already attached
If `az network nic show` returns a non-empty `virtualMachine.id` before the attach step, `xfusion-nic` is already attached to a VM and cannot be attached elsewhere until detached first.[1][7]

### VM size does not support more NICs
Azure VM sizes support different maximum NIC counts. If `xfusion-vm` already has a primary NIC and the size supports only one NIC, the add operation will fail until a suitable VM size is used.[5][1]

### Authorization errors
If `AuthorizationFailed` appears, the current identity lacks permission to update the VM or NIC resources in that resource group.[2][7]

### Region mismatch
The provided resource list shows both the VM and NIC in `southcentralus`, which is correct for this task.[3]

## Completion criteria
The task is complete when all of the following are true:[1][2]
- `xfusion-nic` appears in the NIC list for `xfusion-vm`.
- `az network nic show` for `xfusion-nic` returns the VM resource ID in the `virtualMachine.id` field.
- The VM is back in `VM running` state after the NIC attachment.
- The VM initialization has completed before task submission.[4]

## Command summary
```bash
az vm show --resource-group kml_rg_main-045b4bdee4654a6b --name xfusion-vm --show-details --query "{name:name,powerState:powerState,location:location,vmSize:hardwareProfile.vmSize}" -o table
az network nic show --resource-group kml_rg_main-045b4bdee4654a6b --name xfusion-nic --query "{name:name,location:location,virtualMachine:id,provisioningState:provisioningState}" -o table
az vm nic list --resource-group kml_rg_main-045b4bdee4654a6b --vm-name xfusion-vm -o table
az vm deallocate --resource-group kml_rg_main-045b4bdee4654a6b --name xfusion-vm
az vm nic add --resource-group kml_rg_main-045b4bdee4654a6b --vm-name xfusion-vm --nics xfusion-nic
az vm start --resource-group kml_rg_main-045b4bdee4654a6b --name xfusion-vm
az vm nic list --resource-group kml_rg_main-045b4bdee4654a6b --vm-name xfusion-vm -o table
az network nic show --resource-group kml_rg_main-045b4bdee4654a6b --name xfusion-nic --query "{name:name,attachedVm:virtualMachine.id,provisioningState:provisioningState}" -o table
az vm show --resource-group kml_rg_main-045b4bdee4654a6b --name xfusion-vm --show-details --query "{name:name,powerState:powerState}" -o table
```