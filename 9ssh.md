Here is a clean **VS Code–ready Markdown (.md)** version of your task:

***

# Azure SSH Key Creation – KodeKloud Task

## Objective
- Create an SSH key pair named `devops-kp`
- Ensure the key type is `RSA`
- Use Azure portal credentials (via `showcreds`) if creating via portal

***

## Prerequisites
- Access to `azure-client` terminal (KodeKloud Engineer UI)
- Azure portal credentials from:
  ```
  showcreds
  ```

***

## Method 1: Using Linux Terminal (azure-client)

### Step 1: Open Terminal
Connect to the `azure-client` host.

### Step 2: Navigate to Home Directory
```
cd ~
```

### Step 3: Generate RSA Key Pair
```
ssh-keygen -t rsa -b 4096 -f devops-kp
```

- `-t rsa` → Specifies RSA key type  
- `-b 4096` → Sets key length to 4096 bits  
- `-f devops-kp` → Output file name  

### Step 4: Passphrase
- Press **Enter twice** to leave it empty (unless required)

### Step 5: Verify Key Creation
```
ls -l devops-kp*
```

### Expected Output
- `devops-kp` (private key)
- `devops-kp.pub` (public key)

***

## Method 2: Using Azure Portal

### Step 1: Login
- Run `showcreds` on `azure-client`
- Use credentials to log into Azure Portal

### Step 2: Navigate to SSH Keys
- Search for **"SSH keys"** in Azure Portal
- Open the service

### Step 3: Create SSH Key
- Click **Create**
- Fill required fields:
  - Subscription
  - Resource Group
  - Region
  - Key pair name: `devops-kp`

### Step 4: Configure Key
- SSH public key source: **Generate new key pair**
- Key type: **RSA**

### Step 5: Create Resource
- Click **Review + create**
- Click **Create**

### Step 6: Download Key
- Download the private key (`.pem`) when prompted

***

## Validation

### Terminal Method
```
ls -l ~/devops-kp*
```
- Ensure both key files exist

### Portal Method
- Confirm SSH Key resource:
  - Name: `devops-kp`
  - Key type: `RSA`
  - Exists in selected resource group

***

If you want, I can also convert this into a **one-command cheat sheet** or **automated script version** for faster execution in KodeKloud.