# Azure VM Creation Runbook

## Purpose
Create a Linux virtual machine named `datacenter-vm` in the existing lab resource group, in `westus`, using the Ubuntu 24.04 LTS image, size `Standard_B1s`, with an NSG rule allowing SSH on port `22`, and a `30 GB` Standard HDD OS disk. Then verify SSH connectivity to the VM. 

## Prerequisites
- Logged into Azure using the lab account; verify with `az account show`. 
- `az` CLI available on the `azure-client` host. 
- Existing lab resource group already present; do not create a new resource group. 

## Step 1: Identify the existing resource group
Run the following command:

```bash
az group list -o table
```

From the output, note the existing lab resource group name and store it in a variable, for example:

```bash
RG_NAME=kml_rg_main-df8961bb49f944d8
```

Reuse this existing resource group for the VM deployment. 

## Step 2: Select the Ubuntu 24.04 LTS image
Use the following Ubuntu 24.04 LTS server image URN:

```bash
IMAGE="Canonical:ubuntu-24_04-lts:server:latest"
```

This matches the `ubuntu-24_04-lts` offer and `server` SKU used for Azure Linux VM creation. 

## Step 3: Create the VM
Run the following command:

```bash
az vm create \
  --resource-group "$RG_NAME" \
  --name datacenter-vm \
  --location westus \
  --image "$IMAGE" \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --nsg-rule SSH \
  --os-disk-size-gb 30 \
  --os-disk-name datacenter-vm-osdisk \
  --storage-sku Standard_LRS
```

### Parameter mapping
- Existing resource group: `--resource-group "$RG_NAME"` 
- VM name: `--name datacenter-vm` 
- Region: `--location westus` 
- Image: `Canonical:ubuntu-24_04-lts:server:latest` 
- Size: `--size Standard_B1s` 
- NSG allowing SSH: `--nsg-rule SSH` creates an inbound rule for port `22`. 
- OS disk: `--os-disk-size-gb 30` with `--storage-sku Standard_LRS` for Standard HDD storage. 
- SSH authentication: `--generate-ssh-keys` creates or reuses `~/.ssh/id_rsa`. 

## Step 4: Verify deployment and public IP
Check the VM details with:

```bash
az vm show \
  --resource-group "$RG_NAME" \
  --name datacenter-vm \
  -d -o table
```

Confirm the following in the output:
- `powerState` is `VM running` 
- `location` is `westus` 
- `publicIpAddress` is present 

## Step 5: Test SSH connectivity
From the `azure-client` host, connect using:

```bash
ssh -i ~/.ssh/id_rsa azureuser@<publicIpAddress>
```

Example:

```bash
ssh -i ~/.ssh/id_rsa azureuser@20.245.66.88
```

Accept the host key prompt, then verify access by running:

```bash
hostname
```

Successful shell access confirms the VM is reachable over SSH. 

## Validation checklist
- VM `datacenter-vm` exists in the existing lab resource group. 
- Region is `westus`. 
- Image is Ubuntu 24.04 LTS using `Canonical:ubuntu-24_04-lts:server:latest`. 
- Size is `Standard_B1s`. 
- NSG attached to the VM allows inbound SSH on port `22`. 
- OS disk is `30 GB` and uses `Standard_LRS` for Standard HDD. 
- SSH from `azure-client` to the VM succeeds with the `azureuser` account.