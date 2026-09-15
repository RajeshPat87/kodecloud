# Senior Platform Engineer (Azure IDP) — Coding Questions & Solutions

Mapped to the JD. Each question is the kind of hands-on task asked in a senior platform interview, followed by a working solution and the talking points that score points.

---

## A. Terraform / IaC

### Q1. Write a reusable Terraform module for an AKS cluster following platform best practices.

**What they're testing:** module hygiene, managed identity, RBAC, Key Vault CSI, sane defaults, outputs.

```hcl
# modules/aks/variables.tf
variable "name"                { type = string }
variable "location"            { type = string }
variable "resource_group_name" { type = string }
variable "dns_prefix"          { type = string }
variable "kubernetes_version"  { type = string  default = null }
variable "node_count"          { type = number  default = 3 }
variable "vm_size"             { type = string  default = "Standard_D4s_v5" }
variable "vnet_subnet_id"      { type = string }
variable "log_analytics_workspace_id" { type = string }
variable "admin_group_object_ids"     { type = list(string) }
variable "tags"                { type = map(string) default = {} }

# modules/aks/main.tf
resource "azurerm_kubernetes_cluster" "this" {
  name                = var.name
  location            = var.location
  resource_group_name = var.resource_group_name
  dns_prefix          = var.dns_prefix
  kubernetes_version  = var.kubernetes_version
  local_account_disabled = true            # force Entra ID auth
  oidc_issuer_enabled    = true
  workload_identity_enabled = true

  default_node_pool {
    name                 = "system"
    node_count           = var.node_count
    vm_size              = var.vm_size
    vnet_subnet_id       = var.vnet_subnet_id
    only_critical_addons_enabled = true    # taint system pool
    upgrade_settings { max_surge = "33%" }
  }

  identity { type = "SystemAssigned" }

  network_profile {
    network_plugin      = "azure"
    network_plugin_mode = "overlay"
    network_policy      = "cilium"
  }

  azure_active_directory_role_based_access_control {
    azure_rbac_enabled     = true
    admin_group_object_ids = var.admin_group_object_ids
  }

  key_vault_secrets_provider { secret_rotation_enabled = true }

  oms_agent {
    log_analytics_workspace_id = var.log_analytics_workspace_id
  }

  tags = var.tags
}

# modules/aks/outputs.tf
output "cluster_id"       { value = azurerm_kubernetes_cluster.this.id }
output "oidc_issuer_url"  { value = azurerm_kubernetes_cluster.this.oidc_issuer_url }
output "kubelet_identity" { value = azurerm_kubernetes_cluster.this.kubelet_identity[0].object_id }
output "node_resource_group" { value = azurerm_kubernetes_cluster.this.node_resource_group }
```

**Talking points:** `local_account_disabled + azure_rbac_enabled` kills the shared admin kubeconfig — every access is via Entra ID (segregation of duties, from the JD). `workload_identity + oidc_issuer` is the modern replacement for pod-managed-identity. Separate system node pool with a taint keeps app workloads off it.

---

### Q2. Provision a Key Vault that an AKS workload can read secrets from — no secrets in code, no access policies.

**What they're testing:** RBAC-based Key Vault (not the legacy access-policy model) + workload identity federation.

```hcl
resource "azurerm_key_vault" "this" {
  name                        = var.kv_name
  location                    = var.location
  resource_group_name         = var.rg_name
  tenant_id                   = data.azurerm_client_config.current.tenant_id
  sku_name                    = "standard"
  enable_rbac_authorization   = true        # RBAC, not access policies
  purge_protection_enabled    = true
  public_network_access_enabled = false     # private endpoint only
}

# User-assigned identity the workload will federate to
resource "azurerm_user_assigned_identity" "workload" {
  name                = "id-${var.app_name}"
  location            = var.location
  resource_group_name = var.rg_name
}

resource "azurerm_role_assignment" "kv_read" {
  scope                = azurerm_key_vault.this.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_user_assigned_identity.workload.principal_id
}

# Federate the identity to the AKS OIDC issuer + service account
resource "azurerm_federated_identity_credential" "wi" {
  name                = "fic-${var.app_name}"
  resource_group_name = var.rg_name
  parent_id           = azurerm_user_assigned_identity.workload.id
  audience            = ["api://AzureADTokenExchange"]
  issuer              = var.aks_oidc_issuer_url
  subject             = "system:serviceaccount:${var.namespace}:${var.app_name}"
}
```

**Talking point:** the federated credential is the whole trick — no client secret ever exists. The pod's service-account token is exchanged for an Entra token scoped to that identity.

---

### Q3. Enforce a governance rule at the module level: reject any resource group name that doesn't match the corporate naming standard.

**What they're testing:** input validation as a governance control (JD: "governed infrastructure patterns").

```hcl
variable "resource_group_name" {
  type = string
  validation {
    condition     = can(regex("^rg-(dev|test|prod)-[a-z0-9]{3,10}-[a-z]{3}$", var.resource_group_name))
    error_message = "Name must be rg-<env>-<app>-<region>, e.g. rg-prod-orders-eus."
  }
}
```

Follow-up they'll ask: *"How do you enforce this across teams who don't use your module?"* → Azure Policy with a `deny` effect, applied at the management-group scope, so it holds regardless of the deployment path.

---

## B. Bicep

### Q4. Write a Bicep module for an Azure Container Registry locked down with a private endpoint.

```bicep
param name string
param location string = resourceGroup().location
param subnetId string
param privateDnsZoneId string

resource acr 'Microsoft.ContainerRegistry/registries@2023-11-01-preview' = {
  name: name
  location: location
  sku: { name: 'Premium' }            // private endpoints require Premium
  properties: {
    adminUserEnabled: false
    publicNetworkAccess: 'Disabled'
    networkRuleBypassOptions: 'AzureServices'
  }
}

resource pe 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: 'pe-${name}'
  location: location
  properties: {
    subnet: { id: subnetId }
    privateLinkServiceConnections: [{
      name: 'plsc-${name}'
      properties: {
        privateLinkServiceId: acr.id
        groupIds: [ 'registry' ]
      }
    }]
  }
}

resource dnsGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-11-01' = {
  parent: pe
  name: 'default'
  properties: {
    privateDnsZoneConfigs: [{
      name: 'acr'
      properties: { privateDnsZoneId: privateDnsZoneId }
    }]
  }
}

output loginServer string = acr.properties.loginServer
```

**Talking point:** Premium SKU is mandatory for private endpoints; `adminUserEnabled: false` forces token/identity-based pushes (governance).

---

## C. Azure DevOps Pipelines (YAML)

### Q5. Build a reusable "golden path" CI template: build → unit test → SAST → dependency scan → container scan → push. Teams consume it with one `extends`.

**What they're testing:** template design, security-shifted-left, parameterization.

```yaml
# templates/ci-golden-path.yml
parameters:
  - name: appName        type: string
  - name: dockerfile     type: string  default: Dockerfile
  - name: registry       type: string
  - name: buildContext   type: string  default: '.'

stages:
- stage: Build
  jobs:
  - job: build_scan_push
    pool: { vmImage: 'ubuntu-latest' }
    steps:
      - script: dotnet test --collect:"XPlat Code Coverage"
        displayName: Unit tests

      - task: SonarCloudPrepare@2          # code quality gate
        inputs: { SonarCloud: 'sonar-sc', organization: 'org', scannerMode: 'MSBuild' }
      - task: SonarCloudAnalyze@2
      - task: SonarCloudPublish@2          # fails build if quality gate red

      - script: |                          # dependency scanning
          dotnet list package --vulnerable --include-transitive | tee vuln.txt
          ! grep -q "Critical\|High" vuln.txt
        displayName: Dependency scan (fail on High/Critical)

      - task: Docker@2
        inputs:
          command: build
          repository: ${{ parameters.appName }}
          dockerfile: ${{ parameters.dockerfile }}
          buildContext: ${{ parameters.buildContext }}
          tags: $(Build.BuildId)

      - script: |                          # container image scan
          trivy image --exit-code 1 --severity HIGH,CRITICAL \
            ${{ parameters.appName }}:$(Build.BuildId)
        displayName: Trivy image scan

      - task: Docker@2
        inputs:
          command: push
          containerRegistry: ${{ parameters.registry }}
          repository: ${{ parameters.appName }}
          tags: $(Build.BuildId)
```

Consumer pipeline (this is what a stream-aligned team writes — nothing else):

```yaml
# azure-pipelines.yml in the app repo
trigger: [ main ]
resources:
  repositories:
    - repository: templates
      type: git
      name: platform/pipeline-templates
extends:
  template: templates/ci-golden-path.yml@templates
  parameters:
    appName: orders-api
    registry: acr-prod
```

**Key point (JD explicitly says this):** security/compliance are *embedded in the pipeline as failing gates*, not manual approvals. Trivy `--exit-code 1` and the dependency-scan `grep` both break the build automatically.

---

### Q6. Add a governed CD stage: deploy to `prod` only through an Environment with a required approval, pulling secrets from Key Vault.

```yaml
- stage: Deploy_Prod
  dependsOn: Build
  jobs:
  - deployment: deploy
    environment: 'production'      # attach approvals + checks in ADO UI
    variables:
      - group: prod-secrets        # variable group linked to Key Vault
    strategy:
      runOnce:
        deploy:
          steps:
            - task: AzureKeyVault@2
              inputs:
                azureSubscription: 'sc-prod'   # workload-identity service connection
                KeyVaultName: 'kv-prod-orders'
                SecretsFilter: 'db-conn,api-key'
            - task: HelmDeploy@1
              inputs:
                connectionType: 'Azure Resource Manager'
                azureSubscription: 'sc-prod'
                command: upgrade
                chartType: FilePath
                chartPath: ./chart
                overrideValues: image.tag=$(Build.BuildId)
```

**Talking points:** the *Environment* object (not a stage condition) is where you attach approvals, business-hours checks, and branch-control checks — that's ADO's segregation-of-duties primitive. The service connection uses **workload identity federation**, so there's no stored service-principal secret to rotate.

---

### Q7. Write a Terraform pipeline template that runs `plan` on PR and gates `apply` behind approval on merge.

```yaml
# templates/terraform.yml
parameters:
  - name: workingDir  type: string
  - name: environment type: string

stages:
- stage: Plan
  jobs:
  - job: plan
    steps:
      - script: |
          terraform init -backend-config=env/${{ parameters.environment }}.tfbackend
          terraform plan -out=tfplan -var-file=env/${{ parameters.environment }}.tfvars
        workingDirectory: ${{ parameters.workingDir }}
      - publish: ${{ parameters.workingDir }}/tfplan
        artifact: tfplan
      - task: Checkov@1                   # IaC policy scan on the plan
        inputs: { directory: ${{ parameters.workingDir }} }

- stage: Apply
  dependsOn: Plan
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: apply
    environment: 'infra-${{ parameters.environment }}'   # approval gate here
    strategy:
      runOnce:
        deploy:
          steps:
            - download: current
              artifact: tfplan
            - script: terraform apply -auto-approve tfplan
              workingDirectory: ${{ parameters.workingDir }}
```

**Talking point:** apply the *saved plan artifact*, never re-plan at apply time — that guarantees what was reviewed is what gets applied. Checkov shifts IaC policy compliance left.

---

## D. Kubernetes / AKS

### Q8. Write production-grade manifests for a stateless service: Deployment + Service + HPA + PDB, hardened.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders, labels: { app: orders } }
spec:
  replicas: 3
  selector: { matchLabels: { app: orders } }
  template:
    metadata: { labels: { app: orders } }
    spec:
      serviceAccountName: orders           # federated to a workload identity
      securityContext:
        runAsNonRoot: true
        seccompProfile: { type: RuntimeDefault }
      containers:
        - name: orders
          image: acrprod.azurecr.io/orders:1.4.2
          ports: [ { containerPort: 8080 } ]
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits:   { cpu: "500m", memory: "256Mi" }
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: [ "ALL" ] }
          readinessProbe:
            httpGet: { path: /healthz/ready, port: 8080 }
          livenessProbe:
            httpGet: { path: /healthz/live, port: 8080 }
          topologySpreadConstraints:
            - maxSkew: 1
              topologyKey: kubernetes.io/hostname
              whenUnsatisfiable: DoNotSchedule
---
apiVersion: v1
kind: Service
metadata: { name: orders }
spec:
  selector: { app: orders }
  ports: [ { port: 80, targetPort: 8080 } ]
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: orders }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: orders }
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: orders }
spec:
  maxUnavailable: 1
  selector: { matchLabels: { app: orders } }
```

**Talking points:** `requests`/`limits` are what the HPA and the cluster autoscaler key off — omit them and both break. `readOnlyRootFilesystem + drop ALL caps + runAsNonRoot` is the baseline pod-security posture. PDB `maxUnavailable: 1` keeps the app serving through node drains/upgrades (tie back to the earlier PDB discussion).

---

## E. Scripting & Automation (self-service onboarding)

### Q9. Automate onboarding a new team: create the repo, apply branch policies, add a build validation pipeline — via the Azure DevOps REST API. (JD: "automate onboarding ... repository creation, branching policies, pipeline configuration.")

Python version:

```python
import requests, base64, os

ORG   = "myorg"
PROJ  = "platform"
PAT   = os.environ["AZDO_PAT"]
BASE  = f"https://dev.azure.com/{ORG}/{PROJ}/_apis"
AUTH  = {"Authorization": "Basic " + base64.b64encode(f":{PAT}".encode()).decode()}
API   = "?api-version=7.1"

def create_repo(name):
    r = requests.post(f"{BASE}/git/repositories{API}",
                      json={"name": name}, headers=AUTH)
    r.raise_for_status()
    return r.json()["id"]

def require_min_reviewers(repo_id, refs="refs/heads/main"):
    # Policy type GUID for "Minimum number of reviewers"
    policy = {
        "isEnabled": True, "isBlocking": True,
        "type": {"id": "fa4e907d-c16b-4a4c-9dfa-4906e5d171dd"},
        "settings": {
            "minimumApproverCount": 2,
            "creatorVoteCounts": False,
            "resetOnSourcePush": True,
            "scope": [{"repositoryId": repo_id,
                       "refName": refs, "matchKind": "exact"}]
        }
    }
    requests.post(f"{BASE}/policy/configurations{API}",
                  json=policy, headers=AUTH).raise_for_status()

if __name__ == "__main__":
    app = "orders-api"
    rid = create_repo(app)
    require_min_reviewers(rid)
    print(f"Onboarded {app} -> repo {rid} with 2-reviewer branch policy")
```

**Talking points:** this is the heart of the self-service platform — a developer fills a form, a pipeline runs this, and the repo comes pre-governed (branch policy, reviewers, reset-on-push). Idempotency matters: in production I'd check-if-exists before creating so re-runs don't fail. PAT should itself come from Key Vault / a workload identity, not an env var, in the real thing.

---

### Q10. PowerShell: write a script that reports any Key Vault secret expiring within 30 days across a subscription.

```powershell
param([int]$DaysAhead = 30)

$threshold = (Get-Date).AddDays($DaysAhead)
$results = foreach ($kv in Get-AzKeyVault) {
    foreach ($s in Get-AzKeyVaultSecret -VaultName $kv.VaultName) {
        if ($s.Expires -and $s.Expires -lt $threshold) {
            [pscustomobject]@{
                Vault   = $kv.VaultName
                Secret  = $s.Name
                Expires = $s.Expires
                Days    = [int]($s.Expires - (Get-Date)).TotalDays
            }
        }
    }
}
$results | Sort-Object Days | Format-Table -AutoSize
if ($results) { exit 1 }   # non-zero so a pipeline step can fail on findings
```

**Talking point:** `exit 1` on findings lets you drop this straight into a scheduled pipeline as a compliance gate rather than a report someone has to read.

---

### Q11. Bash: given `kubectl get pods -o json`, list every pod that has a container without CPU/memory limits set.

```bash
#!/usr/bin/env bash
kubectl get pods -A -o json \
| jq -r '
  .items[]
  | . as $p
  | .spec.containers[]
  | select((.resources.limits.cpu == null) or (.resources.limits.memory == null))
  | "\($p.metadata.namespace)/\($p.metadata.name) -> \(.name)"
'
```

**Talking point:** unbounded pods are the classic noisy-neighbour and autoscaler-failure cause; this is the kind of one-liner audit you build into a platform health dashboard. In a real setup I'd enforce it up front with an OPA/Gatekeeper or Azure Policy constraint rather than detect it after the fact.

---

## F. Crossplane (self-service abstraction)

### Q12. Design a Crossplane abstraction so a developer can request a "Postgres database" with a tiny YAML, and the platform composes the real Azure resources. (JD: "platform provisioning and abstraction layers using Crossplane.")

The developer-facing claim:

```yaml
apiVersion: platform.acme.io/v1alpha1
kind: PostgresInstance
metadata: { name: orders-db }
spec:
  parameters: { size: small, region: eastus }
  compositionSelector:
    matchLabels: { provider: azure }
```

The XRD (defines the API):

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata: { name: xpostgresinstances.platform.acme.io }
spec:
  group: platform.acme.io
  names: { kind: XPostgresInstance, plural: xpostgresinstances }
  claimNames: { kind: PostgresInstance, plural: postgresinstances }
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    size:   { type: string, enum: [small, medium, large] }
                    region: { type: string }
```

The Composition (maps the abstraction to real Azure resources — flexible server + firewall):

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: azure-postgres
  labels: { provider: azure }
spec:
  compositeTypeRef:
    apiVersion: platform.acme.io/v1alpha1
    kind: XPostgresInstance
  resources:
    - name: flexible-server
      base:
        apiVersion: dbforpostgresql.azure.upbound.io/v1beta1
        kind: FlexibleServer
        spec:
          forProvider:
            location: eastus
            skuName: B_Standard_B1ms
            storageMb: 32768
            version: "16"
      patches:
        - fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.location
        - fromFieldPath: spec.parameters.size
          toFieldPath: spec.forProvider.skuName
          transforms:
            - type: map
              map: { small: B_Standard_B1ms, medium: GP_Standard_D2s_v3, large: GP_Standard_D4s_v3 }
```

**Talking points:** the developer never sees `skuName` or Azure resource types — they ask for `size: small`. The platform team owns the Composition, so guardrails (region allow-list, SKU mapping, backup policy) live in one governed place. This is "platform-as-a-product" made concrete: a self-service API with the complexity abstracted away. Compare/contrast with Terraform: Terraform is imperative-run-to-completion; Crossplane is a *continuously reconciling control plane* — it drift-corrects the database forever, not just at apply time.

---

## G. Light coding round (some panels include one)

### Q13. Parse a pipeline log file and print the top 5 slowest stages by duration.

```python
import re, heapq
from collections import defaultdict

# lines like: "2024-05-01T10:00:00 STAGE Build START"
#             "2024-05-01T10:04:30 STAGE Build END"
from datetime import datetime

starts, durations = {}, {}
pat = re.compile(r"(\S+)\s+STAGE\s+(\w+)\s+(START|END)")
with open("pipeline.log") as f:
    for line in f:
        m = pat.search(line)
        if not m: continue
        ts, stage, kind = m.groups()
        t = datetime.fromisoformat(ts)
        if kind == "START":
            starts[stage] = t
        else:
            durations[stage] = (t - starts[stage]).total_seconds()

for stage, secs in heapq.nlargest(5, durations.items(), key=lambda kv: kv[1]):
    print(f"{stage:20} {secs:8.1f}s")
```

**Talking point:** keep it O(n) single-pass, use a dict keyed by stage, and `heapq.nlargest` instead of sorting the whole set. Be ready to discuss handling interleaved/parallel stages (the `starts` dict already supports that).

---

## How to prep with this

- Be fluent writing the **Terraform AKS module and the golden-path pipeline template from memory** — those two are the most likely live-coding asks for this exact JD.
- For every security/quality answer, say the phrase **"embedded as a failing gate, not a manual approval"** — the JD calls this out twice.
- Whenever asked "how do you enforce X", have a **two-layer answer**: the module/pipeline enforces it on the happy path, and **Azure Policy at management-group scope** enforces it regardless of path.
- Know the **Terraform vs Crossplane** distinction cold (run-to-completion vs continuous reconciliation) — it's the differentiator for the "abstraction layer" bullet.