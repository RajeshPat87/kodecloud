Here’s a concise run book–style documentation for the blob upload task you just did.

***

## Objective

Copy the local file `/tmp/xfusion.txt` from the Azure client host into an existing Blob container:

- Storage account: `xfusionst6291`  
- Blob container: `xfusion-blob-25964`  
- Target blob name: `xfusion.txt`

***

## Preconditions

- You are logged in to Azure CLI (lab `showcreds` + `az login` already done).  
- Storage account `xfusionst6291` and container `xfusion-blob-25964` already exist (created by the lab).  
- The file `/tmp/xfusion.txt` exists on the `azure-client` VM.

***

## Step 1 – Set task variables

From the `azure-client` shell:

```bash
STORAGE_ACCOUNT_NAME="xfusionst6291"
CONTAINER_NAME="xfusion-blob-25964"
LOCAL_FILE="/tmp/xfusion.txt"
BLOB_NAME="xfusion.txt"
```

Optionally, discover the resource group and subscription the lab is using:

```bash
RESOURCE_GROUP=$(az group list --query '[0].name' -o tsv)
SUBSCRIPTION_NAME=$(az account show --query 'name' -o tsv)

echo "Subscription   : $SUBSCRIPTION_NAME"
echo "Resource Group : $RESOURCE_GROUP"
echo "Storage Account: $STORAGE_ACCOUNT_NAME"
echo "Container      : $CONTAINER_NAME"
echo "Local File     : $LOCAL_FILE"
echo "Blob Name      : $BLOB_NAME"
```

***

## Step 2 – Check the local file

Verify that `/tmp/xfusion.txt` exists before attempting upload:

```bash
if [ ! -f "$LOCAL_FILE" ]; then
  echo "ERROR: File $LOCAL_FILE not found"
  exit 1
fi
```

If this prints the error, re-check the lab instructions and ensure the file is present at the expected path.

***

## Step 3 – Retrieve a storage account key

Use the Azure CLI to fetch one of the access keys for `xfusionst6291`:

```bash
ACCOUNT_KEY=$(az storage account keys list \
  --account-name "$STORAGE_ACCOUNT_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query '[0].value' \
  -o tsv)
```

- `az storage account keys list` returns an array of key objects.  
- `[0].value` selects the first key’s `value` field.  
- `-o tsv` prints only the raw key value, suitable for scripting.

You can confirm:

```bash
echo "${ACCOUNT_KEY:0:6}..."
```

***

## Step 4 – Upload the file as a blob

Use `az storage blob upload` to copy the local file into the container:

```bash
az storage blob upload \
  --account-name "$STORAGE_ACCOUNT_NAME" \
  --account-key "$ACCOUNT_KEY" \
  --container-name "$CONTAINER_NAME" \
  --name "$BLOB_NAME" \
  --file "$LOCAL_FILE" \
  --overwrite true
```

Notes:

- `--name "$BLOB_NAME"` is the blob’s name inside the container; here we use `xfusion.txt`.  
- `--overwrite true` ensures the command succeeds even if the blob already exists, by replacing it.

A successful run prints JSON including fields like `lastModified`, `etag`, and `request_server_encrypted`.

***

## Step 5 – Verify the upload

List blobs in the container and confirm `xfusion.txt` appears:

```bash
az storage blob list \
  --account-name "$STORAGE_ACCOUNT_NAME" \
  --account-key "$ACCOUNT_KEY" \
  --container-name "$CONTAINER_NAME" \
  --query "[].name" \
  -o table
```

Expected output:

```text
Result
-----------
xfusion.txt
```

At this point, the run book requirement “Copy the file `/tmp/xfusion.txt` to the Blob container `xfusion-blob-25964`” is met.

***

## Full OTS script

Here is the complete script in one block, matching what you ran but cleaned up:

```bash
#!/usr/bin/env bash
set -eo pipefail

STORAGE_ACCOUNT_NAME="xfusionst6291"
CONTAINER_NAME="xfusion-blob-25964"
LOCAL_FILE="/tmp/xfusion.txt"
BLOB_NAME="xfusion.txt"

RESOURCE_GROUP="$(az group list --query '[0].name' -o tsv)"
SUBSCRIPTION_NAME="$(az account show --query 'name' -o tsv)"

echo "Subscription   : $SUBSCRIPTION_NAME"
echo "Resource Group : $RESOURCE_GROUP"
echo "Storage Account: $STORAGE_ACCOUNT_NAME"
echo "Container      : $CONTAINER_NAME"
echo "Local File     : $LOCAL_FILE"
echo "Blob Name      : $BLOB_NAME"

if [ ! -f "$LOCAL_FILE" ]; then
  echo "ERROR: File $LOCAL_FILE not found"
  exit 1
fi

ACCOUNT_KEY="$(az storage account keys list \
  --account-name "$STORAGE_ACCOUNT_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query '[0].value' \
  -o tsv)"

az storage blob upload \
  --account-name "$STORAGE_ACCOUNT_NAME" \
  --account-key "$ACCOUNT_KEY" \
  --container-name "$CONTAINER_NAME" \
  --name "$BLOB_NAME" \
  --file "$LOCAL_FILE" \
  --overwrite true

echo
echo "Verifying blobs in container:"
az storage blob list \
  --account-name "$STORAGE_ACCOUNT_NAME" \
  --account-key "$ACCOUNT_KEY" \
  --container-name "$CONTAINER_NAME" \
  --query "[].name" \
  -o table
```

