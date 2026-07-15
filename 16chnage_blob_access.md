# Run Book: Convert Public Azure Blob Container to Private

## Objective
Convert the Azure Blob container `devops-container-179` from public to private in storage account `devopsst10016`, while leaving `devops-priv-18610` unchanged.[1][2]

## Environment details
- Storage account: `devopsst10016`.[3]
- Region: `westus`.[3]
- Resource group: `kml_rg_main-1fa79cd33ab14989`.[3]
- Public container to change: `devops-container-179`.
- Private container to leave unchanged: `devops-priv-18610`.

## Requirement summary
The task requires changing only `devops-container-179` so that it has no anonymous/public access, while ensuring `devops-priv-18610` remains private.[1][4]

## Prerequisites
- Access to the `azure-client` host.
- Azure CLI is installed and authenticated using the provided lab credentials.
- The storage account `devopsst10016` already exists in `westus`.[3]

## Step 1: Verify the resource group and storage account
Check the resource group:

```bash
az group list -o table
```

Expected resource group:

```text
kml_rg_main-1fa79cd33ab14989
```

Check the storage account:

```bash
az storage account list --resource-group kml_rg_main-1fa79cd33ab14989 -o table
```

Expected storage account details include:
- Name: `devopsst10016`
- Location: `westus`
- `AllowBlobPublicAccess`: `True`[3][4]

## Step 2: Retrieve a storage account key
Use the account key to manage the blob containers.[5][1]

```bash
ACCOUNT_KEY=$(az storage account keys list   --account-name devopsst10016   --resource-group kml_rg_main-1fa79cd33ab14989   --query '[0].value'   -o tsv)
```

The query `[0].value` is used because the keys command returns an array of key objects.[1]

## Step 3: List containers and inspect current public access
The correct command to list blob containers is `az storage container list`. The earlier `conatiner` spelling is invalid.[5][2]

```bash
az storage container list   --account-name devopsst10016   --account-key "$ACCOUNT_KEY"   --query "[].{name:name,publicAccess:properties.publicAccess}"   -o table
```

This command displays container names and their current access level, such as `blob`, `container`, or no value for private.[5][1]

## Step 4: Convert the public container to private
Use `az storage container set-permission` to remove anonymous access from `devops-container-179`.[1][4]

```bash
az storage container set-permission   --account-name devopsst10016   --account-key "$ACCOUNT_KEY"   --name devops-container-179   --public-access off
```

The value `off` sets the container to private, meaning no anonymous/public access is allowed.[1][2]

## Step 5: Verify the updated access level
Check all containers again:

```bash
az storage container list   --account-name devopsst10016   --account-key "$ACCOUNT_KEY"   --query "[].{name:name,publicAccess:properties.publicAccess}"   -o table
```

Expected result:
- `devops-container-179` shows no public access (`None` or blank).
- `devops-priv-18610` remains unchanged and private.[1][2]

You can also verify only the target container:

```bash
az storage container show   --account-name devopsst10016   --account-key "$ACCOUNT_KEY"   --name devops-container-179   --query '{name:name,publicAccess:properties.publicAccess}'   -o table
```

The `publicAccess` field should confirm private access.[5][1]

## One-time script
Use the following script to complete the task in one run.[1][4]

```bash
#!/usr/bin/env bash
set -euo pipefail

STORAGE_ACCOUNT_NAME="devopsst10016"
RESOURCE_GROUP="kml_rg_main-1fa79cd33ab14989"
TARGET_CONTAINER="devops-container-179"

echo "Storage Account : $STORAGE_ACCOUNT_NAME"
echo "Resource Group  : $RESOURCE_GROUP"
echo "Target Container: $TARGET_CONTAINER"

ACCOUNT_KEY="$(az storage account keys list   --account-name "$STORAGE_ACCOUNT_NAME"   --resource-group "$RESOURCE_GROUP"   --query '[0].value'   -o tsv)"

echo "Updating container to private..."
az storage container set-permission   --account-name "$STORAGE_ACCOUNT_NAME"   --account-key "$ACCOUNT_KEY"   --name "$TARGET_CONTAINER"   --public-access off

echo
echo "All container access levels:"
az storage container list   --account-name "$STORAGE_ACCOUNT_NAME"   --account-key "$ACCOUNT_KEY"   --query "[].{name:name,publicAccess:properties.publicAccess}"   -o table

echo
echo "Verification for target container:"
az storage container show   --account-name "$STORAGE_ACCOUNT_NAME"   --account-key "$ACCOUNT_KEY"   --name "$TARGET_CONTAINER"   --query '{name:name,publicAccess:properties.publicAccess}'   -o table
```

## Expected outcome
The task is complete when:
- `devops-container-179` is private with no public access.
- `devops-priv-18610` remains private and unchanged.[1][2]