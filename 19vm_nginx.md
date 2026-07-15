You ended up with a **perfect, policy‑compliant Nginx VM** after a lot of wrestling with the disk policy and NSG rules. Here’s a clear **documentation + run book** that captures exactly what you did and how to repeat it.

***

## Overview

Goal:

- Create an Azure VM named `datacenter-vm` in **Central US**.  
- Use an Ubuntu image (`Ubuntu2204`, `Standard_B1s`).  
- Run a script to install and start Nginx.  
- Allow HTTP traffic on port 80 from the internet.

Constraints:

- Azure Policy: OS disk must be **≤128 GB** and **non‑Premium** (Standard_LRS or Standard_RAGRS). [stellium](https://stellium.consulting/articles/insights/azure-policy-for-managed-disks/)
- Lab identity has limited RBAC (you can’t list all disks, but you can delete a disk by known name). [learn.microsoft](https://learn.microsoft.com/en-us/answers/questions/1697979/authorizationfailed-the-client-with-object-id-does)

***

## Part 1 – Cleaning failed / non‑compliant resources

### 1. Set base variables

```bash
RG_NAME="kml_rg_main-bac46401c99e4d4b"
VM_NAME="datacenter-vm"
LOCATION="centralus"
```

### 2. Observe failed, non‑compliant VM

You ran:

```bash
az vm show \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  -o json
```

Which showed:

- `"provisioningState": "Failed"`  
- OS disk `diskSizeGb: 30` but `storageAccountType: "Premium_LRS"`. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/vm?view=azure-cli-latest)

This is exactly what the policy error referenced:

> Resource 'datacenter-vm_OsDisk_1_8061d395af0b422b8f67f2faa87165d8' was disallowed by policy. Reasons: ‘Ensure the disk size is 128 GB or less and the SKU is not Premium for Compute disks.’ [kodekloud](https://kodekloud.com/community/t/unable-to-create-the-vm-in-azure/474778)

### 3. Delete the failed VM

```bash
az vm delete \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --yes
```

This removed the VM resource so you could start clean.

### 4. Delete the non‑compliant OS disk by name

From the policy error, you already had the disk name:

```bash
az disk delete \
  --resource-group "$RG_NAME" \
  --name "datacenter-vm_OsDisk_1_8061d395af0b422b8f67f2faa87165d8" \
  --yes
```

This works even though `az disk list` is blocked by RBAC; delete by known name bypasses the listing permission requirement. [learn.microsoft](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/find-unattached-disks)

### 5. Confirm the VM is gone

```bash
az vm show \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  -o json
```

Result:

```text
(ResourceNotFound) The Resource 'Microsoft.Compute/virtualMachines/datacenter-vm' ... was not found.
```

So you now had a clean slate for `datacenter-vm`. [learn.microsoft](https://learn.microsoft.com/en-us/azure/azure-resource-manager/troubleshooting/error-not-found)

***

## Part 2 – Create the Nginx VM with compliant disk

### 6. Create `datacenter-vm` with Standard_LRS OS disk

You set:

```bash
VM_NAME="datacenter-vm"
IMAGE="Ubuntu2204"
VM_SIZE="Standard_B1s"
ADMIN_USER="azureuser"
```

Then ran:

```bash
az vm create \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --location "$LOCATION" \
  --image "$IMAGE" \
  --size "$VM_SIZE" \
  --admin-username "$ADMIN_USER" \
  --authentication-type password \
  --admin-password "P@ssw0rd1234!" \
  --os-disk-size-gb 30 \
  --storage-sku Standard_LRS
```

The result:

```json
{
  "powerState": "VM running",
  "privateIpAddress": "10.0.0.4",
  "publicIpAddress": "130.131.197.226",
  "resourceGroup": "kml_rg_main-bac46401c99e4d4b"
}
```

Now the new OS disk is 30 GB and `Standard_LRS`, satisfying the policy. [learn.microsoft](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types)

### 7. Run the Nginx install/start script with Run Command

You used Azure’s Run Command feature:

```bash
az vm run-command invoke \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --command-id RunShellScript \
  --scripts "sudo apt-get update -y && sudo apt-get install -y nginx && sudo systemctl start nginx && sudo systemctl enable nginx"
```

The output showed `ProvisioningState/succeeded` and detailed package installation logs, confirming Nginx was installed and enabled. [learn.microsoft](https://learn.microsoft.com/en-us/azure/virtual-machines/run-command-overview)

### 8. Open HTTP (port 80) in the NSG

First attempt (priority 1000) failed with:

> Security rule default-allow-ssh conflicts with rule open-port-80. Rules cannot have the same Priority and Direction. [youtube](https://www.youtube.com/watch?v=G7G8o1b0aSc)

So you correctly reran with a different priority:

```bash
az vm open-port \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --port 80 \
  --priority 1001
```

This created NSG `datacenter-vmNSG` with:

- `default-allow-ssh` (Inbound, port 22, priority 1000).  
- `open-port-80` (Inbound, port 80, priority 1001). [dev](https://dev.to/darkenstein/configuring-azure-vm-s-security-groups-to-allow-web-access-via-your-web-browser-22e1)

Both rules co-exist without conflict and allow SSH + HTTP.

### 9. Get public IP and test Nginx

You captured the public IP:

```bash
VM_PUBLIC_IP="$(az vm show -d \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --query publicIps \
  -o tsv)"

echo "$VM_PUBLIC_IP"
# 130.131.197.226
```

Then:

```bash
curl http://"$VM_PUBLIC_IP"
```

Output:

- Full HTML of the default Nginx “Welcome to nginx!” page, proving:
  - Nginx is installed and running.  
  - Port 80 is open in NSG.  
  - The VM is reachable from the internet. [medium](https://medium.com/@cloud.devops.enthusiast/azure-virtual-machines-demystified-installing-nginx-web-server-4af7d1019471)

***

## Final Run Book: `datacenter-vm.sh`

Here is a consolidated script that matches what you achieved, including cleanup, VM creation, Nginx install, and NSG rule creation.

```bash
#!/usr/bin/env bash
set -euo pipefail

RG_NAME="kml_rg_main-bac46401c99e4d4b"
VM_NAME="datacenter-vm"
LOCATION="centralus"
IMAGE="Ubuntu2204"
VM_SIZE="Standard_B1s"
ADMIN_USER="azureuser"

echo "Resource Group : $RG_NAME"
echo "VM Name        : $VM_NAME"
echo "Location       : $LOCATION"
echo "Image          : $IMAGE"
echo "Size           : $VM_SIZE"
echo

# ====== Cleanup: delete existing VM and blocked OS disk ======
echo "Checking for existing VM..."
if az vm show --resource-group "$RG_NAME" --name "$VM_NAME" -o none 2>/dev/null; then
  echo "Existing VM found, deleting..."
  az vm delete \
    --resource-group "$RG_NAME" \
    --name "$VM_NAME" \
    --yes

  echo "Deleting blocked OS disk (if present)..."
  az disk delete \
    --resource-group "$RG_NAME" \
    --name "datacenter-vm_OsDisk_1_8061d395af0b422b8f67f2faa87165d8" \
    --yes || true
else
  echo "No existing VM with name $VM_NAME"
fi

echo
echo "Creating VM with Standard_LRS OS disk..."
az vm create \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --location "$LOCATION" \
  --image "$IMAGE" \
  --size "$VM_SIZE" \
  --admin-username "$ADMIN_USER" \
  --authentication-type password \
  --admin-password "P@ssw0rd1234!" \
  --os-disk-size-gb 30 \
  --storage-sku Standard_LRS

# ====== Install and start Nginx via Run Command ======
echo
echo "Installing and starting Nginx..."
az vm run-command invoke \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --command-id RunShellScript \
  --scripts "sudo apt-get update -y && sudo apt-get install -y nginx && sudo systemctl start nginx && sudo systemctl enable nginx"

# ====== Open HTTP port 80 (NSG rule with unique priority) ======
echo
echo "Opening HTTP (port 80) on NSG..."
az vm open-port \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --port 80 \
  --priority 1001

# ====== Get public IP and test ======
echo
VM_PUBLIC_IP="$(az vm show -d \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --query publicIps \
  -o tsv)"

echo "VM Public IP: $VM_PUBLIC_IP"
echo
echo "Test from azure-client:"
echo "curl http://$VM_PUBLIC_IP"
echo
echo "Open in browser: http://$VM_PUBLIC_IP"
echo "You should see the default 'Welcome to nginx!' page."
echo
echo "Run book completed successfully."
```

This script:

- Cleans up any previous failed `datacenter-vm` and its blocked OS disk.  
- Creates a new, policy-compliant Ubuntu VM (`Standard_B1s`, Standard_LRS OS disk).  
- Installs and starts Nginx via Run Command.  
- Opens port 80 with a non‑conflicting NSG rule.  
- Prints the public IP and how to verify Nginx.

You’ve already proven all these steps work end‑to‑end by getting the Nginx welcome page over HTTP, so this run book is now your reusable template for similar labs.