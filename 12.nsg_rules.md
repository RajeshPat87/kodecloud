
***

## Objective

- Resource group: `kml_rg_main-20348f81eb4b497b` (location: `eastus`).  
- Create NSG: `nautilus-nsg`.  
- Add inbound rule `Allow-HTTP`:  
  - Service: HTTP  
  - Port: 80  
  - Source: `0.0.0.0/0`  
  - Action: Allow  
- Add inbound rule `Allow-SSH`:  
  - Service: SSH  
  - Port: 22  
  - Source: `0.0.0.0/0`  
  - Action: Allow. [learn.microsoft](https://learn.microsoft.com/en-us/azure/virtual-network/manage-network-security-group)

***

## 1. Pre-checks

1. Verify active subscription and account:

   ```bash
   az account show -o table
   ```

   Confirms subscription `Azure Free Labs` and the current service principal. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/get-started-tutorial-1-prepare-environment?view=azure-cli-latest)

2. Confirm the target resource group:

   ```bash
   az group list -o table
   ```

   You already saw:

   ```text
   Name                          Location    Status
   ----------------------------  ----------  ---------
   kml_rg_main-20348f81eb4b497b  eastus      Succeeded
   ```

   So this is the group where the NSG must be created. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/manage-azure-groups-azure-cli?view=azure-cli-latest)

3. Check existing NSGs (none yet):

   ```bash
   az network nsg list --resource-group kml_rg_main-20348f81eb4b497b -o table
   ```

   No output means there’s no NSG currently in that RG. [learn.microsoft](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)

***

## 2. Create the NSG

Use `az network nsg create`:

```bash
az network nsg create \
  --name nautilus-nsg \
  --resource-group kml_rg_main-20348f81eb4b497b \
  --location eastus
```

- `--name`: `nautilus-nsg`.  
- `--resource-group`: `kml_rg_main-20348f81eb4b497b`.  
- `--location`: match the RG’s region (`eastus`) for consistency. [learn.microsoft](https://learn.microsoft.com/en-us/azure/virtual-network/manage-network-security-group)

Verify:

```bash
az network nsg list --resource-group kml_rg_main-20348f81eb4b497b -o table
```

You should see `nautilus-nsg` listed with `Location` `eastus`. [learn.microsoft](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)

***

## 3. Add inbound rule Allow-HTTP (port 80)

Use `az network nsg rule create`: [oneuptime](https://oneuptime.com/blog/post/2026-02-16-how-to-configure-network-security-group-rules-to-allow-specific-traffic-in-azure/view)

```bash
az network nsg rule create \
  --resource-group kml_rg_main-20348f81eb4b497b \
  --nsg-name nautilus-nsg \
  --name Allow-HTTP \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes 0.0.0.0/0 \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 80
```

Key parameters:

- `--name Allow-HTTP`: rule name as required.  
- `--priority 100`: lower number = higher priority; keep it unique.  
- `--direction Inbound`: inbound traffic.  
- `--access Allow`: allow traffic.  
- `--protocol Tcp`: HTTP uses TCP.  
- `--source-address-prefixes 0.0.0.0/0`: any source IP.  
- `--destination-port-ranges 80`: HTTP port. [learn.microsoft](https://learn.microsoft.com/en-us/cli/azure/network/nsg/rule?view=azure-cli-latest)

***

## 4. Add inbound rule Allow-SSH (port 22)

Create another rule:

```bash
az network nsg rule create \
  --resource-group kml_rg_main-20348f81eb4b497b \
  --nsg-name nautilus-nsg \
  --name Allow-SSH \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes 0.0.0.0/0 \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 22
```

Differences:

- `--name Allow-SSH`.  
- `--priority 110` (different from 100 to avoid conflicts).  
- `--destination-port-ranges 22`: SSH port. [youtube](https://www.youtube.com/watch?v=9Pnux6zyTGk)

***

## 5. Verify NSG and rules

### 5.1 List NSGs

```bash
az network nsg list \
  --resource-group kml_rg_main-20348f81eb4b497b \
  -o table
```

Confirm `nautilus-nsg` exists.

### 5.2 Show NSG with rules

```bash
az network nsg show \
  --resource-group kml_rg_main-20348f81eb4b497b \
  --name nautilus-nsg \
  --query "securityRules[].{name:name,priority:priority,direction:direction,access:access,src:sourceAddressPrefix,dstPort:destinationPortRange}" \
  -o table
```

Expected table:

```text
Name        Priority  Direction  Access  Src         DstPort
----------  --------  ---------  ------  ----------  -------
Allow-HTTP  100       Inbound    Allow   0.0.0.0/0   80
Allow-SSH   110       Inbound    Allow   0.0.0.0/0   22
```

This confirms both rules are present and configured correctly. [oneuptime](https://oneuptime.com/blog/post/2026-02-16-how-to-configure-network-security-group-rules-to-allow-specific-traffic-in-azure/view)

***

## 6. Notes and best practices

- Opening SSH and HTTP to `0.0.0.0/0` is fine for labs, but in real environments you should restrict source IP ranges whenever possible to follow least-privilege principles. [medium](https://medium.com/@sadoksmine8/implementing-azure-virtual-network-and-network-security-groups-aaf69560e3c7)
- You can later associate `nautilus-nsg` with a subnet or NIC via:

  ```bash
  az network nic update \
    --name <NIC_NAME> \
    --resource-group kml_rg_main-20348f81eb4b497b \
    --network-security-group nautilus-nsg
  ```

  or by associating it at subnet level. [medium](https://medium.com/@sadoksmine8/implementing-azure-virtual-network-and-network-security-groups-aaf69560e3c7)

