You’ve confirmed the current subscription, RG, VM, its state, and that the resize from `Standard_B1s` to `Standard_B2s` succeeded. Here’s a concise documentation/run book that captures what you did and how to repeat or verify it. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/get-started-tutorial-1-prepare-environment?view=azure-cli-latest)

***

## Subscription and account context

1. Check which Azure subscription and identity you’re using:

   ```bash
   az account show -o table
   ```

   Example output you saw:

   ```text
   EnvironmentName    HomeTenantId                          IsDefault    Name             State    TenantId
   -----------------  ------------------------------------  -----------  ---------------  -------  ------------------------------------
   AzureCloud         54c1a2d3-d100-453c-9636-3a109eb45552  True         Azure Free Labs  Enabled  54c1a2d3-d100-453c-9636-3a109eb45552
   ```

   This tells you: [azurelessons](https://azurelessons.com/az-account-show/)
   - Subscription ID: `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b`.  
   - Tenant ID: `54c1a2d3-d100-453c-9636-3a109eb45552`.  
   - Name: `Azure Free Labs`.  
   - Identity type: `servicePrincipal`.

   If you ever need to switch subscriptions, use:

   ```bash
   az account list -o table
   az account set --subscription <SUBSCRIPTION_ID>
   az account show -o table
   ```

***

## Discover resource group and VM

2. List resource groups:

   ```bash
   az group list -o table
   ```

   You saw:

   ```text
   Name                          Location    Status
   ----------------------------  ----------  ---------
   kml_rg_main-ac56ece0bc6d4023  eastus      Succeeded
   ```

   So your lab RG is `kml_rg_main-ac56ece0bc6d4023`. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/manage-azure-groups-azure-cli?view=azure-cli-latest)

3. List VMs in that RG:

   ```bash
   az vm list --resource-group kml_rg_main-ac56ece0bc6d4023 -o table
   ```

   Output:

   ```text
   Name       ResourceGroup                 Location        Zones
   ---------  ----------------------------  --------------  -------
   devops-vm  kml_rg_main-ac56ece0bc6d4023  southcentralus
   ```

   This confirms the VM name `devops-vm` and region `southcentralus`. [docs.azure](https://docs.azure.cn/en-us/virtual-machines/linux/cli-manage)

***

## Check VM state and size before resize

4. Check whether the VM is running (single-line status check):

   ```bash
   az vm show -d --name devops-vm --resource-group kml_rg_main-ac56ece0bc6d4023 --query powerState -o tsv
   ```

   You got:

   ```text
   VM running
   ```

   So the VM is up and running. [superuser](https://superuser.com/questions/1331966/azure-cli-to-check-vm-status)

5. Inspect full VM details, including size:

   ```bash
   az vm show -d --name devops-vm --resource-group kml_rg_main-ac56ece0bc6d4023 -o yaml
   ```

   In the `hardwareProfile` you saw:

   ```yaml
   hardwareProfile:
     vmSize: Standard_B1s
   ```

   That shows the current VM size before the change. [learn.microsoft](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/tutorial-manage-vm)

   The JSON variant:

   ```bash
   az vm show -d --name devops-vm --resource-group kml_rg_main-ac56ece0bc6d4023 -o json
   ```

   provides the same data programmatically. [finisky.github](https://finisky.github.io/en/use-az-cli-to-query-multiple-fields/)

> Note: The failed command `az vm show name devops-vm ...` was due to wrong syntax; `--name` must be a flag, not an inline token. Azure CLI correctly reported “unrecognized arguments: name devops-vm”. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/use-azure-cli-successfully-troubleshooting?view=azure-cli-latest)

***

## Resize the VM from Standard_B1s to Standard_B2s

6. Resize the VM:

   ```bash
   az vm resize \
     --name devops-vm \
     --resource-group kml_rg_main-ac56ece0bc6d4023 \
     --size Standard_B2s
   ```

   The output showed:

   ```json
   "hardwareProfile": {
     "vmSize": "Standard_B2s",
     "vmSizeProperties": null
   },
   "provisioningState": "Succeeded",
   ...
   ```

   This confirms the resize operation completed successfully and the VM size is now `Standard_B2s`. [tutorialspoint](https://www.tutorialspoint.com/how-to-resize-the-azure-vm-using-azure-cli-in-powershell)

   In practice, if you’re unsure whether `Standard_B2s` is available in the region, you can first run:

   ```bash
   az vm list-vm-resize-options \
     --name devops-vm \
     --resource-group kml_rg_main-ac56ece0bc6d4023 \
     -o table
   ```

   to list all sizes this VM can be resized to. [oneuptime](https://oneuptime.com/blog/post/2026-02-16-how-to-resize-an-azure-virtual-machine-without-losing-data/view)

***

## Verify VM state and size after resize

7. Confirm the VM is still running:

   ```bash
   az vm show -d --name devops-vm --resource-group kml_rg_main-ac56ece0bc6d4023 --query powerState -o tsv
   ```

   Output:

   ```text
   VM running
   ```

   So the VM is up after the resize. [stackoverflow](https://stackoverflow.com/questions/73784172/azure-cli-query-power-state-of-a-virtual-machine)

8. Confirm the VM size is now `Standard_B2s`:

   Quick query:

   ```bash
   az vm show \
     --name devops-vm \
     --resource-group kml_rg_main-ac56ece0bc6d4023 \
     --query "hardwareProfile.vmSize" \
     -o tsv
   ```

   Expected result:

   ```text
   Standard_B2s
   ```

   Or inspect again with:

   ```bash
   az vm show -d --name devops-vm --resource-group kml_rg_main-ac56ece0bc6d4023 -o yaml
   ```

   and check:

   ```yaml
   hardwareProfile:
     vmSize: Standard_B2s
   ```  

   which matches the JSON response you already saw. [tutorialspoint](https://www.tutorialspoint.com/how-to-resize-the-azure-vm-using-azure-cli-in-powershell)

***

## Summary checklist for this scenario

For future similar tasks (resize VM and verify):

1. Confirm subscription and account context:

   ```bash
   az account show -o table
   ```

2. Confirm RG and VM exist:

   ```bash
   az group list -o table
   az vm list --resource-group <RG> -o table
   ```

3. Check VM status and size:

   ```bash
   az vm show -d --name <VM> --resource-group <RG> --query "{powerState:powerState,vmSize:hardwareProfile.vmSize}" -o table
   ```

4. Resize the VM:

   ```bash
   az vm resize --name <VM> --resource-group <RG> --size <NEW_SIZE>
   ```

5. Re-check status and size:

   ```bash
   az vm show -d --name <VM> --resource-group <RG> --query "{powerState:powerState,vmSize:hardwareProfile.vmSize}" -o table
   ```

Would you like this documentation packaged as a downloadable Markdown run book file (like your previous tasks), or is the inline text enough?  