# Run Book: Create Azure Storage Account and Private Blob Container

## Objective
Create a new Azure storage account named `datacenterst28172` and a private blob container named `datacenter-blob-9333` in the resource group `kml_rg_main-ca3ef9a6a48f49f8`, then verify both resources were created successfully. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest)

## Environment details
- Subscription: `Azure Free Labs` with subscription ID `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b` was active in the Azure CLI session. [learn.microsoft](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create)
- Resource group: `kml_rg_main-ca3ef9a6a48f49f8` in region `eastus` was available and in `Succeeded` state. [learn.microsoft](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create)
- The task uses Azure CLI commands executed from the Azure client host. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest)

## Prerequisites
- Azure CLI is installed and authenticated.
- The target resource group already exists.
- Storage account names must be globally unique, 3 to 24 characters long, and contain only lowercase letters and numbers. [datacamp](https://www.datacamp.com/tutorial/azure-storage-accounts)
- Blob container names must be lowercase and may contain letters, numbers, and hyphens. [learn.microsoft](https://learn.microsoft.com/en-us/azure/storage/blobs/blob-containers-cli)

## Step 1: Verify subscription and resource group
Check the active subscription:

```bash
az account list --output table
```

Expected result:

```text
Name             CloudName    SubscriptionId                        TenantId                              State    IsDefault
---------------  -----------  ------------------------------------  ------------------------------------  -------  -----------
Azure Free Labs  AzureCloud   f0c3bcdd-5ce2-4fa0-8cf3-41559747512b  54c1a2d3-d100-453c-9636-3a109eb45552  Enabled  True
```

Check the resource group:

```bash
az group list -o table
```

Expected result:

```text
Name                          Location    Status
----------------------------  ----------  ---------
kml_rg_main-ca3ef9a6a48f49f8  eastus      Succeeded
```

These checks confirm the subscription context and target resource group before creating storage resources. [learn.microsoft](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create)

## Step 2: Understand the correct storage account listing command
The command `az storage list` is invalid because `storage` is a command group, not a directly listable resource type. To list storage accounts, use `az storage account list`. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest)

Incorrect command:

```bash
az storage list --resource-group kml_rg_main-ca3ef9a6a48f49f8 -o table
```

Correct command:

```bash
az storage account list --resource-group kml_rg_main-ca3ef9a6a48f49f8 -o table
```

Initially, the list was empty, confirming there was no storage account yet in that resource group. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest)

## Step 3: Create the storage account
Create the storage account using StorageV2 and Standard_LRS:

```bash
az storage account create   --name datacenterst28172   --resource-group kml_rg_main-ca3ef9a6a48f49f8   --location eastus   --sku Standard_LRS   --kind StorageV2
```

This command creates a general-purpose v2 storage account in `eastus` using locally redundant standard storage, which is a common default for lab tasks and general blob storage use cases. [learn.microsoft](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create)

Key successful output indicators included:
- `name`: `datacenterst28172`
- `location`: `eastus`
- `kind`: `StorageV2`
- `sku.name`: `Standard_LRS`
- `provisioningState`: `Succeeded`
- `primaryEndpoints.blob`: `https://datacenterst28172.blob.core.windows.net/` [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest)

## Step 4: Verify the storage account
List storage accounts again:

```bash
az storage account list   --resource-group kml_rg_main-ca3ef9a6a48f49f8   -o table
```

Expected output included the new storage account:

```text
AccessTier    AllowBlobPublicAccess    AllowCrossTenantReplication    CreationTime                      EnableHttpsTrafficOnly    Kind       Location    MinimumTlsVersion    Name               PrimaryLocation    ProvisioningState    ResourceGroup                 StatusOfPrimary
------------  -----------------------  -----------------------------  --------------------------------  ------------------------  ---------  ----------  -------------------  -----------------  -----------------  -------------------  ----------------------------  -----------------
Hot           False                    False                          2026-07-13T06:34:11.285727+00:00  True                      StorageV2  eastus      TLS1_0               datacenterst28172  eastus             Succeeded            kml_rg_main-ca3ef9a6a48f49f8  available
```

This confirms the storage account was created and is available. [learn.microsoft](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create)

## Step 5: Retrieve a storage account key
To create a blob container with Azure CLI using shared key authorization, retrieve one of the storage account keys. [stackoverflow](https://stackoverflow.com/questions/56894664/retrieve-azure-storage-account-key-using-azure-cli)

Incorrect query used:

```bash
az storage account keys list   --account-name datacenterst28172   --resource-group kml_rg_main-ca3ef9a6a48f49f8   --query ".value"   -o tsv
```

This failed because the command returns an array, and `.value` is not a valid JMESPath expression for selecting a field from a list. [stackoverflow](https://stackoverflow.com/questions/56894664/retrieve-azure-storage-account-key-using-azure-cli)

Correct command:

```bash
az storage account keys list   --account-name datacenterst28172   --resource-group kml_rg_main-ca3ef9a6a48f49f8   --query "[0].value"   -o tsv
```

This selects the `value` field from the first key object in the returned array and outputs the raw key string in TSV format. [stackoverflow](https://stackoverflow.com/questions/56894664/retrieve-azure-storage-account-key-using-azure-cli)

## Step 6: Create the private blob container
Create the blob container using the storage account name and key:

```bash
az storage container create   --name datacenter-blob-9333   --account-name datacenterst28172   --account-key "<ACCOUNT_KEY>"   --public-access off
```

Replace `<ACCOUNT_KEY>` with the value returned from the previous step. Setting `--public-access off` ensures the container is private, which matches the requirement. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/storage/container?view=azure-cli-latest)

Expected output:

```json
{
  "created": true
}
```

This confirms the private container was created successfully. [learn.microsoft](https://learn.microsoft.com/en-us/azure/storage/blobs/blob-containers-cli)

## Step 7: Verify the blob container
List containers in the storage account:

```bash
az storage container list   --account-name datacenterst28172   --account-key "<ACCOUNT_KEY>"   --query "[].name"   -o table
```

Expected output:

```text
Result
--------------------
datacenter-blob-9333
```

This confirms that the blob container exists in the target storage account. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/storage/container?view=azure-cli-latest)

## Troubleshooting
### Error: `az storage list` is not recognized
Use `az storage account list` because `storage` is a command group and the list action applies to `account`. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/storage/account?view=azure-cli-latest)

### Error: `invalid jmespath_type value: '.value'`
Use `--query "[0].value"` because the output of `az storage account keys list` is an array of key objects. [stackoverflow](https://stackoverflow.com/questions/56894664/retrieve-azure-storage-account-key-using-azure-cli)

### Container is not visible
Verify the correct storage account name and account key are being used, then rerun the container list command with the same credentials. [learn.microsoft](https://learn.microsoft.com/en-us/azure/storage/blobs/blob-containers-cli)

## Completion criteria
The task is complete when all of the following are true: [learn.microsoft](https://learn.microsoft.com/en-us/azure/storage/blobs/blob-containers-cli)
- Storage account `datacenterst28172` exists in resource group `kml_rg_main-ca3ef9a6a48f49f8`.
- The storage account shows `ProvisioningState` as `Succeeded`.
- Blob container `datacenter-blob-9333` exists in the storage account.
- The container is private because `--public-access off` was used during creation.

## Command summary

```bash
az account list --output table
az group list -o table
az storage account list --resource-group kml_rg_main-ca3ef9a6a48f49f8 -o table

az storage account create   --name datacenterst28172   --resource-group kml_rg_main-ca3ef9a6a48f49f8   --location eastus   --sku Standard_LRS   --kind StorageV2

az storage account list   --resource-group kml_rg_main-ca3ef9a6a48f49f8   -o table

az storage account keys list   --account-name datacenterst28172   --resource-group kml_rg_main-ca3ef9a6a48f49f8   --query "[0]