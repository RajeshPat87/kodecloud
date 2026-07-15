# Azure Run Book: Attach Existing Public IP to VM NIC

## Objective
Attach the existing Public IP address `datacenter-pip` to the network interface of the virtual machine `datacenter-vm-pip` in the resource group `kml_rg_main-a2166ef8bf0c4c5f`, and verify that the VM is properly assigned the Public IP before task submission.[1][2][3]

## Environment
- Subscription: `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b`
- Resource group: `kml_rg_main-a2166ef8bf0c4c5f`
- VM: `datacenter-vm-pip`
- NIC: `datacenter-vm-pipVMNic`
- Existing IP configuration on NIC: `ipconfigdatacenter-vm-pip`
- Public IP: `datacenter-pip`
- Region for resources: `southcentralus`
- Execution host: `azure-client`[4][5]

## Background
Azure assigns a Public IP to a VM by associating the Public IP resource with an IP configuration on the VM’s network interface card. In Azure CLI, this is typically done with `az network nic ip-config update`.[1][6][7]

The Azure CLI errors seen earlier were caused by command typos such as `az rg`, `az resources`, and `az network public-ips`, which are not valid command groups. Azure CLI suggests corrected command names when the command group is misspelled.[8]

The failed `ResourceNotFoundError` during NIC update occurred because the actual NIC IP configuration name was `ipconfigdatacenter-vm-pip`, not `ipconfig1`. Azure requires the exact IP configuration name when updating the NIC.[9][6][10]

## Preconditions
- You are logged in to Azure from the `azure-client` host.[8]
- The resource group exists and is in `Succeeded` state.[4]
- The existing resources `datacenter-vm-pip`, `datacenter-pip`, and `datacenter-vm-pipVMNic` are present in the target resource group.[5]
- The VM has completed initialization and is visible through `az vm show`.[1][2]

## Pre-checks
### Verify the resource group

```bash
az group show \
  --name kml_rg_main-a2166ef8bf0c4c5f \
  --output table
```

Expected result: `kml_rg_main-a2166ef8bf0c4c5f` is returned with provisioning state `Succeeded`.[4]

### Verify the VM exists

```bash
az vm show \
  --resource-group kml_rg_main-a2166ef8bf0c4c5f \
  --name datacenter-vm-pip \
  --show-details \
  --query "{name:name,powerState:powerState,location:location}" \
  --output table
```

Expected result: the VM exists in `southcentralus` and shows a valid power state, typically `VM running`.[11][3]

### Verify the Public IP exists

```bash
az network public-ip show \
  --resource-group kml_rg_main-a2166ef8bf0c4c5f \
  --name datacenter-pip \
  --output table
```

Expected result: `datacenter-pip` exists, has a valid address, and shows provisioning state `Succeeded`.[12][3]

### Verify the NIC exists

```bash
az network nic show \
  --resource-group kml_rg_main-a2166ef8bf0c4c5f \
  --name datacenter-vm-pipVMNic \
  --query "{name:name,location:location,provisioningState:provisioningState}" \
  --output table
```

Expected result: the NIC exists and is healthy.[13]

### Verify the actual NIC IP configuration name

```bash
az network nic show \
  --resource-group kml_rg_main-a2166ef8bf0c4c5f \
  --name datacenter-vm-pipVMNic \
  --query "ipConfigurations[].{name:name,privateIp:privateIpAddress,publicIpId:publicIPAddress.id}" \
  --output table
```

Expected result: the IP configuration name appears as `ipconfigdatacenter-vm-pip`. This value must be used exactly in the update command.[6][10]

## Procedure
### Step 1: Attach the Public IP to the NIC
Run the command below using the actual IP configuration name returned by the NIC query.[9][6]

```bash
az network nic ip-config update \
  --resource-group kml_rg_main-a2166ef8bf0c4c5f \
  --nic-name datacenter-vm-pipVMNic \
  --name ipconfigdatacenter-vm-pip \
  --public-ip-address datacenter-pip
```

This command updates the NIC IP configuration so that it references the existing Public IP resource `datacenter-pip`.[9][6][7]

### Step 2: Wait for completion
Allow the CLI command to finish and confirm it does not return an error. A successful response indicates the NIC was updated in Azure Resource Manager.[1][6]

## Verification
### Verify the NIC now references the Public IP

```bash
az network nic show \
  --resource-group kml_rg_main-a2166ef8bf0c4c5f \
  --name datacenter-vm-pipVMNic \
  --query "ipConfigurations[].{name:name,publicIpId:publicIPAddress.id}" \
  --output table
```

Expected result: the row for `ipconfigdatacenter-vm-pip` now contains the resource ID of `datacenter-pip` in `publicIpId`.[6][10]

### Verify the Public IP details

```bash
az network public-ip show \
  --resource-group kml_rg_main-a2166ef8bf0c4c5f \
  --name datacenter-pip \
  --query "{name:name,ip:ipAddress,provisioningState:provisioningState}" \
  --output table
```

Expected result: `datacenter-pip` shows the assigned address and provisioning state `Succeeded`.[12][6]

### Verify the VM shows the Public IP

```bash
az vm show \
  --resource-group kml_rg_main-a2166ef8bf0c4c5f \
  --name datacenter-vm-pip \
  --show-details \
  --query "{name:name,publicIps:publicIps}" \
  --output table
```

Expected result: the VM output includes the Public IP address assigned through the NIC.[11][3]

### Verify the VM is using the updated NIC

```bash
az vm nic list \
  --resource-group kml_rg_main-a2166ef8bf0c4c5f \
  --vm-name datacenter-vm-pip \
  --output table
```

Expected result: `datacenter-vm-pipVMNic` appears in the NIC list for the VM.[14][2]

## Optional portal steps
The same association can also be performed in the Azure portal by opening the VM, going to **Networking** or **IP configurations**, selecting the NIC IP configuration, and associating the existing Public IP `datacenter-pip` with it.[1][3]

## Troubleshooting
### Misspelled Azure CLI command groups
The following are invalid and should be corrected as shown below:[8]

| Incorrect command | Correct command |
|---|---|
| `az rg show` | `az group show` |
| `az rg list` | `az group list` |
| `az group lisy` | `az group list` |
| `az resources list` | `az resource list` |
| `az network public-ips list` | `az network public-ip list` |

### ResourceNotFoundError during NIC update
This error usually means the NIC IP configuration name in `--name` is wrong. In this case, `ipconfig1` failed because the real IP configuration name is `ipconfigdatacenter-vm-pip`.[15][6][10]

### Public IP not found
If Azure reports that `datacenter-pip` was not found, confirm the name and resource group with:

```bash
az network public-ip list \
  --resource-group kml_rg_main-a2166ef8bf0c4c5f \
  --output table
```

This confirms whether the Public IP exists in the expected scope.[12]

### Authorization errors
If the update command fails with `AuthorizationFailed`, the current identity lacks permission to modify the NIC or Public IP resources, and RBAC changes will be required by an administrator.[8]

## Completion criteria
This task is complete when all of the following are true:[1][2][3]
- The NIC IP configuration `ipconfigdatacenter-vm-pip` references `datacenter-pip` in `publicIpId`.
- `az network public-ip show` returns the Public IP successfully with an address and `Succeeded` state.
- `az vm show --show-details --query publicIps` returns the VM’s public IP.
- The VM `datacenter-vm-pip` is properly assigned the Public IP and ready for any downstream connectivity checks.

## Command summary
```bash
az group show --name kml_rg_main-a2166ef8bf0c4c5f -o table
az vm show --resource-group kml_rg_main-a2166ef8bf0c4c5f --name datacenter-vm-pip --show-details --query "{name:name,powerState:powerState,location:location}" -o table
az network public-ip show --resource-group kml_rg_main-a2166ef8bf0c4c5f --name datacenter-pip -o table
az network nic show --resource-group kml_rg_main-a2166ef8bf0c4c5f --name datacenter-vm-pipVMNic --query "ipConfigurations[].{name:name,privateIp:privateIpAddress,publicIpId:publicIPAddress.id}" -o table
az network nic ip-config update --resource-group kml_rg_main-a2166ef8bf0c4c5f --nic-name datacenter-vm-pipVMNic --name ipconfigdatacenter-vm-pip --public-ip-address datacenter-pip
az network nic show --resource-group kml_rg_main-a2166ef8bf0c4c5f --name datacenter-vm-pipVMNic --query "ipConfigurations[].{name:name,publicIpId:publicIPAddress.id}" -o table
az network public-ip show --resource-group kml_rg_main-a2166ef8bf0c4c5f --name datacenter-pip --query "{name:name,ip:ipAddress,provisioningState:provisioningState}" -o table
az vm show --resource-group kml_rg_main-a2166ef8bf0c4c5f --name datacenter-vm-pip --show-details --query "{name:name,publicIps:publicIps}" -o table
az vm nic list --resource-group kml_rg_main-a2166ef8bf0c4c5f --vm-name datacenter-vm-pip -o table
```