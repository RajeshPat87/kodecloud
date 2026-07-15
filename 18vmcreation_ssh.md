Below is a concise **run book** and **documentation-style summary** for this task, based on the exact commands you ran and the Azure policy behavior you hit. [kodekloud](https://kodekloud.com/community/t/day-21-assigning-public-ip-to-virtual-machines-azure-cloud-100-day-challenge/493204)

***

## Objective

Create an Azure VM named `xfusion-vm` in **Central US** using an Ubuntu image and size `Standard_B1s`, with:

- SSH public key generated on `azure-client`.  
- Static public IP named `xfusion-pip` attached to the VM.  
- VM accessible via SSH using the generated key.  

Additional constraint: A subscription‑level Azure Policy enforces **OS disk ≤128 GB** and **non‑Premium disk SKU (Standard_LRS/Standard_RAGRS)**. [kodekloud](https://kodekloud.com/community/t/unable-to-create-the-vm-in-azure/474778)

***

## Section 1 – Cleanup: Remove non‑compliant VM and disk

### 1. Set core variables

```bash
RG_NAME="kml_rg_main-c199ecb4185f4537"
VM_NAME="xfusion-vm"
PIP_NAME="xfusion-pip"
LOCATION="centralus"
```

### 2. Delete the VM

```bash
az vm delete \
  --resource-group "$RG_NAME" \
  --name "$VM_NAME" \
  --yes
```

Note: If the VM was created with a disallowed OS disk (Premium), `az vm delete` may itself show the `RequestDisallowedByPolicy` error referencing the OS disk. [kodekloud](https://kodekloud.com/community/t/unable-to-create-the-vm-in-azure/474778)

### 3. List and delete the OS disk

List disks:

```bash
az disk list \
  --resource-group "$RG_NAME" \
  --query "[].{name:name,sku:sku.name}" \
  -o table
```

Delete the non‑compliant OS disk:

```bash
az disk delete \
  --resource-group "$RG_NAME" \
  --name "xfusion-vm_OsDisk_1_c6df056c316449c79c22c261ce139f6f" \
  --yes
```

This removes the Premium OS disk that violated policy. [kodekloud](https://kodekloud.com/community/t/unable-to-create-the-azure-vms/82240)

### 4. Verify the public IP resource `xfusion-pip`

```bash
az network public-ip show \
  --resource-group "$RG_NAME" \
  --name xfusion-pip \
  -o json
```

Confirm:

- `name`: `xfusion-pip`  
- `location`: `centralus`  
- `publicIPAllocationMethod`: `Static`  
- `sku.name`: `Standard`  

This ensures the existing static Standard public IP is healthy and still associated with the NIC `xfusion-vmVMNic`. [notes.kodekloud](https://notes.kodekloud.com/docs/AZ-700-Designing-and-Implementing-Microsoft-Azure-Networking-Solutions/Configure-Public-IP-Addresses/Creating-Public-IP-Addresses/page)

***

## Section 2 – Build: Create VM with compliant disk and SSH access

### 1. Generate SSH key on `azure-client`

Use the default key name so the grader can find it:

```bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N ""
```

This creates:

- Private key: `/root/.ssh/id_rsa`  
- Public key: `/root/.ssh/id_rsa.pub`  

The lab validation is known to look for `/root/.ssh/id_rsa` specifically. [kodekloud](https://kodekloud.com/community/t/day-21-assigning-public-ip-to-virtual-machines-task/495205)

### 2. Create or reuse static public IP `xfusion-pip`

If not already present:

```bash
az network public-ip create \
  --resource-group "$RG_NAME" \
  --name xfusion-pip \
  --location centralus \
  --sku Standard \
  --allocation-method Static \
  --version IPv4
```

Otherwise, reuse the existing one as you did (`xfusion-pip` with `74.249.226.7`). [learn.microsoft](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/virtual-network-static-public-ip)

### 3. Create `xfusion-vm` with Ubuntu, Standard_B1s, and Standard OS disk

```bash
az vm create \
  --resource-group "$RG_NAME" \
  --name xfusion-vm \
  --location centralus \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --public-ip-address xfusion-pip \
  --os-disk-size-gb 30 \
  --storage-sku Standard_LRS
```

This ensures:

- VM name: `xfusion-vm`.  
- Region: `centralus`.  
- Image: `Ubuntu2204` (allowed in lab).  
- Size: `Standard_B1s`.  
- SSH key from `azure-client`.  
- Public IP `xfusion-pip` attached.  
- OS disk: 30 GB, `Standard_LRS`, compliant with the policy. [stellium](https://stellium.consulting/articles/insights/azure-policy-for-managed-disks/)

Your successful output:

```json
{
  "powerState": "VM running",
  "publicIpAddress": "74.249.226.7",
  "privateIpAddress": "10.0.0.4",
  "resourceGroup": "kml_rg_main-c199ecb4185f4537"
}
```

indicates the VM was created and is running. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/azure-cli-vm-tutorial-3?view=azure-cli-latest)

### 4. Open SSH port (22) via NSG

```bash
az vm open-port \
  --resource-group "$RG_NAME" \
  --name xfusion-vm \
  --port 22 \
  --priority 1001
```

This adds an NSG rule `open-port-22` in `xfusion-vmNSG` for inbound TCP port 22. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/azure-cli-vm-tutorial-3?view=azure-cli-latest)

You confirmed the NSG has:

- `default-allow-ssh` (priority 1000, TCP 22).  
- `open-port-22` (priority 1001, TCP 22).

So SSH traffic from the internet is allowed. [notes.kodekloud](https://notes.kodekloud.com/docs/AZ-700-Designing-and-Implementing-Microsoft-Azure-Networking-Solutions/Configure-Public-IP-Addresses/Creating-Public-IP-Addresses/page)

### 5. Get VM public IP and test SSH from `azure-client`

Retrieve IP:

```bash
VM_PUBLIC_IP="$(az vm show -d \
  --resource-group "$RG_NAME" \
  --name xfusion-vm \
  --query publicIps \
  -o tsv)"

echo "$VM_PUBLIC_IP"  # 74.249.226.7
```

SSH in:

```bash
ssh azureuser@"$VM_PUBLIC_IP"
```

You saw:

- Host key prompt (first connection).  
- Successful login banner (`Welcome to Ubuntu 22.04.5 LTS`).  
- `whoami` returns `azureuser`. [learn.microsoft](https://learn.microsoft.com/en-us/azure/virtual-machines/linux-vm-connect)

This final SSH test confirms:

- VM is running.  
- SSH key generated on `azure-client` is correctly associated.  
- Static public IP `xfusion-pip` is attached and reachable.  

That matches all lab requirements and passes typical KodeKloud validation (they run a similar `ssh azureuser@IP` check). [kodekloud](https://kodekloud.com/community/t/100-day-cloud-azure-day-21-assigning-public-ip-to-virtual-machines-task-validation-issue-ssh-access-failed-despite-correct-setup/493740)

***

## Section 3 – Documentation Notes

### Policy behavior

- The Azure Policy in this lab denies OS disks when:
  - `diskSizeGb > 128` **or**  
  - `sku.name` is Premium (e.g., `Premium_LRS`). [stellium](https://stellium.consulting/articles/insights/azure-policy-for-managed-disks/)
- Initial attempts without `--storage-sku Standard_LRS` caused `RequestDisallowedByPolicy` errors because Azure defaulted the OS disk to Premium.  
- Deleting the failed VM and its OS disk, then recreating with `--os-disk-size-gb 30` and `--storage-sku Standard_LRS`, resolved the policy violation.

### Grader expectations (KodeKloud)

Community threads show common pitfalls and best practices for this exact task: [kodekloud](https://kodekloud.com/community/t/assigning-public-ip-to-virtual-machines-ssh-access-to-nautilus-vm-failed-or-vm-is-not-in-the-running-state/483031)

- Use default SSH key name `/root/.ssh/id_rsa` (not a custom name).  
- Use `ssh azureuser@<PUBLIC_IP>` from `azure-client` (no `-i` in the grader’s test).  
- Keep VM name, public IP name, region, and size exactly as specified.  
- Ensure VM is in **Running** state when submitting.

Your final state (running VM, working SSH, static Standard IP) matches these expectations.

***

If you want, I can condense this into a single `.sh` run book file with comments that you can reuse for future labs with just variable changes.