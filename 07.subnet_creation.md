Here’s a run book–style, copy-paste-ready procedure to create the `xfusion-vnet` VNet and `xfusion-subnet` subnet in `centralus` using Azure CLI, based on the resource group you already have (`kml_rg_main-4cdb6de2d0ba44da`). [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/azure-cli-vm-tutorial-2?view=azure-cli-latest)

***

## Purpose

Create an Azure Virtual Network and subnet for the Nautilus DevOps migration pilot:

- VNet: `xfusion-vnet`  
- Subnet: `xfusion-subnet`  
- Region: `centralus`  
- VNet IPv4 range: `10.0.0.0/16`  
- Subnet IPv4 range: `10.0.0.0/24`  
- Resource group: `kml_rg_main-4cdb6de2d0ba44da` (already exists) [learn.microsoft](https://learn.microsoft.com/en-us/azure/virtual-network/manage-virtual-network)

***

## Preconditions

- You have shell access to the `azure-client` host.  
- Azure CLI (`az`) is installed and available in `$PATH`. [learn.microsoft](https://learn.microsoft.com/id-id/cli/azure/network/vnet?view=azure-cli-latest)
- Valid Azure credentials are provided via the `showcreds` command on `azure-client`.  
- Resource group `kml_rg_main-4cdb6de2d0ba44da` exists (you already confirmed with `az group list -o table`). [tutorialspoint](https://www.tutorialspoint.com/how-to-get-the-azure-resource-group-using-azure-cli-in-powershell)

If any of these are missing or fail, stop and escalate to the cloud admin.

***

## Step 1 – Retrieve Azure credentials

1. SSH or log in to the `azure-client` host.  
2. Run:

   ```bash
   showcreds
   ```

3. Note the output (typically includes tenant ID, subscription ID, client ID, secret, or login command). Use whatever format the lab provides to authenticate in the next step. [scribd](https://www.scribd.com/document/988037657/1-SSH)

***

## Step 2 – Log in to Azure

Depending on how `showcreds` exposes credentials, use one of these patterns.

### Option A: Interactive user login

If `showcreds` gives you a URL/device code or indicates user-based login:

```bash
az login
```

- Follow the browser/device-code instructions and ensure login succeeds. [learn.microsoft](https://learn.microsoft.com/nl-nl/Azure/virtual-network/quick-create-cli)

### Option B: Service principal login (common in labs)

If `showcreds` prints a service principal’s app ID, secret, and tenant ID, use:

```bash
az login \
  --service-principal \
  --username <APP_ID_FROM_showcreds> \
  --password <PASSWORD_OR_SECRET_FROM_showcreds> \
  --tenant <TENANT_ID_FROM_showcreds>
```

- Replace the placeholders with values from `showcreds` output. [learn.microsoft](https://learn.microsoft.com/id-id/cli/azure/network/vnet?view=azure-cli-latest)

***

## Step 3 – Select the correct subscription

1. List subscriptions:

   ```bash
   az account list -o table
   ```

2. Identify the subscription that contains the resource group `kml_rg_main-4cdb6de2d0ba44da`.  
3. Set that subscription (replace `<SUBSCRIPTION_ID>`):

   ```bash
   az account set --subscription <SUBSCRIPTION_ID>
   ```

This ensures all subsequent commands run against the right subscription. [learn.microsoft](https://learn.microsoft.com/nl-nl/Azure/virtual-network/quick-create-cli)

***

## Step 4 – Confirm the resource group

Verify the existing resource group:

```bash
az group list -o table
```

You should see:

```text
Name                          Location    Status
----------------------------  ----------  ---------
kml_rg_main-4cdb6de2d0ba44da  eastus      Succeeded
```

If the resource group is missing or in an unexpected state, do not proceed; raise an issue. [tutorialspoint](https://www.tutorialspoint.com/how-to-get-the-azure-resource-group-using-azure-cli-in-powershell)

***

## Step 5 – Plan the address space

For this task, the address plan is:

- VNet address space: `10.0.0.0/16`  
- Subnet address prefix: `10.0.0.0/24`  

A `/16` contains multiple `/24` networks; using `10.0.0.0/24` for the first subnet is acceptable and commonly used. [learn.microsoft](https://learn.microsoft.com/fil-ph/Azure/virtual-network/quick-create-cli)

Ensure this does not overlap with any existing VNets used for VPN/peering (not usually a concern in a standalone lab).

***

## Step 6 – Create the VNet and subnet

Use a single `az network vnet create` command, which can both create the VNet and the initial subnet. [techlabs](https://techlabs.blog/categories/azure/create-virtual-network-and-subnet-using-azure-cli)

Run:

```bash
az network vnet create \
  --name xfusion-vnet \
  --resource-group kml_rg_main-4cdb6de2d0ba44da \
  --location centralus \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name xfusion-subnet \
  --subnet-prefixes 10.0.0.0/24
```

Expected behavior: [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/azure-cli-vm-tutorial-2?view=azure-cli-latest)
- Command completes with `provisioningState: Succeeded`.  
- `xfusion-vnet` is created in `centralus` with address space `10.0.0.0/16`.  
- `xfusion-subnet` is created inside `xfusion-vnet` with prefix `10.0.0.0/24`.

If the command fails, capture error output and move to the troubleshooting section.

***

## Step 7 – Verify VNet configuration

Check the VNet:

```bash
az network vnet show \
  --name xfusion-vnet \
  --resource-group kml_rg_main-4cdb6de2d0ba44da \
  -o json
```

Verify in the JSON: [docs.azure](https://docs.azure.cn/en-us/virtual-machines/linux/tutorial-virtual-network)

- `location` equals `centralus`.  
- `addressSpace.addressPrefixes` includes `10.0.0.0/16`.  
- `subnets` array contains an entry with:
  - `"name": "xfusion-subnet"`  
  - `"addressPrefix": "10.0.0.0/24"`

You can use table output for a quick view:

```bash
az network vnet show \
  --name xfusion-vnet \
  --resource-group kml_rg_main-4cdb6de2d0ba44da \
  -o table
```

***

## Step 8 – Verify subnet configuration

List subnets for the VNet:

```bash
az network vnet subnet list \
  --vnet-name xfusion-vnet \
  --resource-group kml_rg_main-4cdb6de2d0ba44da \
  -o table
```

You should see one row:

- Name: `xfusion-subnet`  
- Address prefix: `10.0.0.0/24` [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/network/vnet/subnet?view=azure-cli-latest)

If the subnet is missing, continue with the manual creation step below.

***

## Optional – Manually create the subnet (if needed)

If you created the VNet without a subnet or the initial command failed to create the subnet, run: [techlabs](https://techlabs.blog/categories/azure/create-virtual-network-and-subnet-using-azure-cli)

```bash
az network vnet subnet create \
  --name xfusion-subnet \
  --resource-group kml_rg_main-4cdb6de2d0ba44da \
  --vnet-name xfusion-vnet \
  --address-prefixes 10.0.0.0/24
```

Then re-run the subnet verification step to confirm. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/network/vnet/subnet?view=azure-cli-latest)

***

## Troubleshooting

- **Login issues**  
  - Symptom: `AADSTS` errors, “Authentication failed”.  
  - Action: Re-run `showcreds` and ensure you use the correct login method (user vs service principal). [learn.microsoft](https://learn.microsoft.com/id-id/cli/azure/network/vnet?view=azure-cli-latest)

- **Subscription mismatch**  
  - Symptom: `Resource group ... could not be found`.  
  - Action: Ensure `az account set --subscription <ID>` matches the subscription where `kml_rg_main-4cdb6de2d0ba44da` exists. [tutorialspoint](https://www.tutorialspoint.com/how-to-get-the-azure-resource-group-using-azure-cli-in-powershell)

- **Address space conflicts**  
  - Symptom: Errors about overlapping address spaces.  
  - Action: Check existing VNets in the subscription:

    ```bash
    az network vnet list -o table
    ```

    Confirm that `10.0.0.0/16` is not already used by a VNet that will be peered or connected via VPN. [oneuptime](https://oneuptime.com/blog/post/2026-01-24-azure-virtual-network/view)

If you cannot resolve the error, capture:

- Full command used  
- Full error message  
- Subscription ID and resource group name  

and escalate to the platform/cloud engineering team.

***

## Completion criteria

This run book is considered successfully executed when: [learn.microsoft](https://learn.microsoft.com/en-us/azure/virtual-network/manage-virtual-network)

- `xfusion-vnet` exists in `centralus` with IPv4 range `10.0.0.0/16`.  
- `xfusion-subnet` exists inside `xfusion-vnet` with IPv4 range `10.0.0.0/24`.  
- Both resources are in resource group `kml_rg_main-4cdb6de2d0ba44da`.  
- Verification commands (`az network vnet show`, `az network vnet subnet list`) match the expected configuration.

Would you like a similar run book to attach a test VM into `xfusion-subnet` to validate connectivity end-to-end?  