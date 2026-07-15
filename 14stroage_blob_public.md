# Run Book: Create Public Storage Account and Container

## Objective
Create a storage account named `datacenterst215` and a public blob container named `datacenter-blob-4732`, with anonymous read access enabled for both container listing and blob reads. [learn.microsoft](https://learn.microsoft.com/en-us/azure/storage/blobs/anonymous-read-access-configure)

## Why `container` access is required
Azure supports multiple container public access levels. `blob` allows anonymous read access only to blob data, while `container` allows anonymous read access to both blobs and the full container listing, which matches the requirement “anonymous read access for containers and blobs.” [learn.microsoft](https://learn.microsoft.com/en-us/powershell/module/az.storage/set-azstoragecontaineracl?view=azps-13.1.0)

## Step 1: Store task values in variables
```bash
STORAGE_ACCOUNT_NAME="datacenterst215"
CONTAINER_NAME="datacenter-blob-4732"
RESOURCE_GROUP=$(az group list --query '[0].name' -o tsv)
LOCATION=$(az group show --name "$RESOURCE_GROUP" --query 'location' -o tsv)
```

## Step 2: Create the storage account with public blob access allowed
```bash
az storage account create   --name "$STORAGE_ACCOUNT_NAME"   --resource-group "$RESOURCE_GROUP"   --location "$LOCATION"   --sku Standard_LRS   --kind StorageV2   --allow-blob-public-access true
```

This account-level flag is important because new storage accounts may block anonymous public access by default unless `allowBlobPublicAccess` is explicitly enabled. [github](https://github.com/Azure/azure-powershell/issues/22360)

## Step 3: Get a storage account key
```bash
ACCOUNT_KEY=$(az storage account keys list   --account-name "$STORAGE_ACCOUNT_NAME"   --resource-group "$RESOURCE_GROUP"   --query '[0].value'   -o tsv)
```

The `--query '[0].value'` syntax is used because the keys command returns an array of key objects. [stackoverflow](https://stackoverflow.com/questions/56894664/retrieve-azure-storage-account-key-using-azure-cli)

## Step 4: Create the public blob container
```bash
az storage container create   --name "$CONTAINER_NAME"   --account-name "$STORAGE_ACCOUNT_NAME"   --account-key "$ACCOUNT_KEY"   --public-access container
```

Using `--public-access container` enables anonymous read access for both containers and blobs, which is the expected requirement here. [learn.microsoft](https://learn.microsoft.com/en-us/azure/storage/blobs/anonymous-read-access-configure)

## Step 5: Verify the result
Verify the storage account:

```bash
az storage account show   --name "$STORAGE_ACCOUNT_NAME"   --resource-group "$RESOURCE_GROUP"   --query '{name:name,allowBlobPublicAccess:allowBlobPublicAccess,provisioningState:provisioningState}'   -o table
```

Verify the container:

```bash
az storage container show   --name "$CONTAINER_NAME"   --account-name "$STORAGE_ACCOUNT_NAME"   --account-key "$ACCOUNT_KEY"   --query '{name:name,publicAccess:properties.publicAccess}'   -o table
```

Expected `publicAccess` value is `container`. [learn.microsoft](https://learn.microsoft.com/en-us/powershell/module/az.storage/set-azstoragecontaineracl?view=azps-13.1.0)

## One-time script
Use the attached shell script to execute the entire workflow in one shot. [github](https://github.com/Azure/azure-powershell/issues/22360)