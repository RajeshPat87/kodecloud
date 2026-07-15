You’ve completed the lab correctly and you now have a solid transcript you can turn into a reusable run book. Here’s a clean run book for *“SSH into nautilus-vm as root using the client’s root key”* based on exactly what you did.

***

## Objective

Set up passwordless SSH access from the Azure client host to the VM `nautilus-vm` as the `root` user by:

1. Confirming the VM and its public IP.  
2. Logging in as `azureuser`.  
3. Copying the lab’s public key into root’s `authorized_keys` on the VM.  
4. Verifying direct SSH as `root` from the client without a password.

***

## 1. Discover resource group, VM, and public IP

Run from the Azure client host:

```bash
# List resource groups
az group list -o table
```

Confirm:

```text
Name                          Location    Status
----------------------------  ----------  ---------
kml_rg_main-ae7ca7c413fc4713  eastus      Succeeded
```

List VMs in the resource group:

```bash
az vm list --resource-group kml_rg_main-ae7ca7c413fc4713 -o table
```

You saw:

```text
Name         ResourceGroup                 Location    Zones
-----------  ----------------------------  ----------  -------
nautilus-vm  kml_rg_main-ae7ca7c413fc4713  eastus
```

Check VM details (OS, SSH config, etc.):

```bash
az vm show --name nautilus-vm --resource-group kml_rg_main-ae7ca7c413fc4713
```

Confirm power state and IP:

```bash
az vm show -d \
  --name nautilus-vm \
  --resource-group kml_rg_main-ae7ca7c413fc4713 \
  --query powerState -o tsv

az vm show -d \
  --name nautilus-vm \
  --resource-group kml_rg_main-ae7ca7c413fc4713 \
  --query publicIps -o tsv
```

You got:

```text
VM running
23.96.1.26
```

So `<NAUTILUS_VM_PUBLIC_IP>` = `23.96.1.26`.

***

## 2. SSH into the VM as azureuser

From the Azure client host:

```bash
ssh azureuser@23.96.1.26
```

On first connection you saw the host key prompt:

```text
The authenticity of host '23.96.1.26 (23.96.1.26)' can't be established.
ECDSA key fingerprint is SHA256:JMoxW0VxwXE+rBldgmqmQW3sjnDgXejxBNtObnKMqH8.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '23.96.1.26' (ECDSA) to the list of known hosts.
```

Then you were logged in and saw the Ubuntu MOTD and prompt:

```bash
azureuser@nautilus-vm:~$ whoami
azureuser
```

This confirms SSH as `azureuser` works and the VM is reachable.

***

## 3. Become root and configure root’s .ssh on the VM

From inside `nautilus-vm`:

```bash
sudo -i
whoami
```

You saw:

```text
root
```

Create and secure root’s `.ssh` directory:

```bash
mkdir -p /root/.ssh
chmod 700 /root/.ssh
ls -ltr
```

Output showed `.ssh` exists and permissions are correct.

***

## 4. Add the lab key to root’s authorized_keys

The VM’s `osProfile.ssh.publicKeys` shows the public key for `root@azure-client` already in `/home/azureuser/.ssh/authorized_keys`, so copying that to root is equivalent to adding `/root/.ssh/id_rsa.pub` from the client. [learn.microsoft](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-cli)

As root on `nautilus-vm`:

```bash
cp /home/azureuser/.ssh/authorized_keys /root/.ssh/authorized_keys
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys
```

Verify content and permissions:

```bash
ls -ld /root/.ssh
ls -l /root/.ssh/authorized_keys
```

You saw:

```text
drwx------ 2 root root 4096 Jul  8 13:43 /root/.ssh
-rw------- 1 root root 399 Jul  8 13:56 /root/.ssh/authorized_keys
```

This is exactly what SSH expects:

- Directory: `700` (owner only).  
- File: `600` (owner read/write).

If needed, you can also inspect the key:

```bash
cat /root/.ssh/authorized_keys
```

and confirm it contains the `ssh-rsa ... root@azure-client` key.

***

## 5. Verify passwordless root SSH from Azure client host

Logout from the VM back to the Azure client host:

```bash
root@nautilus-vm:~# logout
azureuser@nautilus-vm:~$ logout
Connection to 23.96.1.26 closed.
```

Now, from the Azure client host:

```bash
ssh root@23.96.1.26
```

On first connect, you saw the host key prompt for root and then:

```text
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 6.8.0-1059-azure x86_64)
...
root@nautilus-vm:~# whoami
root
```

Critically, you did **not** get a password prompt — it used the private key `/root/.ssh/id_rsa` on the Azure client that matches the public key in `/root/.ssh/authorized_keys` on the VM. That’s the exact verification the lab requires.

***

## 6. Completion criteria checklist

You can use this checklist when doing similar tasks:

- VM `nautilus-vm` in `kml_rg_main-ae7ca7c413fc4713` is `VM running`.  
- VM has a public IP (here `23.96.1.26`).  
- `azureuser` can SSH into the VM.  
- On the VM:

  - `/root/.ssh` exists and is `drwx------`.  
  - `/root/.ssh/authorized_keys` exists and is `-rw-------`.  
  - `authorized_keys` contains the root public key from the Azure client host (or the equivalent key copied from `azureuser`).

- From the Azure client host:

  ```bash
  ssh root@<NAUTILUS_VM_PUBLIC_IP>
  ```

  connects as `root` without prompting for a password, and `whoami` inside the session returns `root`.

When all of the above are true, the “SSH into nautilus-vm as root without password” task is complete.

If you’d like, I can turn this into a ready-to-save Markdown run book that you can reuse across other SSH-key tasks (with placeholders for VM name, RG, and IP).