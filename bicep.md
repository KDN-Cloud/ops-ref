# bicep command ref

## installation & setup

```bash
# install bicep CLI (standalone)
curl -Lo bicep https://github.com/Azure/bicep/releases/latest/download/bicep-linux-x64
chmod +x ./bicep
sudo mv ./bicep /usr/local/bin/bicep
bicep --version

# or via Azure CLI (installs/upgrades bundled bicep)
az bicep install
az bicep upgrade
az bicep version

# or via package manager (macOS)
brew install azure/bicep/bicep

# VS Code extension (strongly recommended)
# install "Bicep" extension by Microsoft for IntelliSense, linting, and visualization
```

## build & decompile

```bash
bicep build main.bicep                  # compile .bicep → ARM JSON template
bicep build main.bicep --outfile out.json   # specify output path
bicep build *.bicep                     # build all .bicep files in dir

bicep decompile main.json               # decompile ARM JSON → .bicep (best-effort)
bicep decompile main.json --force       # overwrite existing .bicep file

bicep format main.bicep                 # format .bicep file in place
bicep lint main.bicep                   # lint for issues (also runs during build)
```

## deploy (via azure cli)

```bash
# resource group scope (most common)
az deployment group create \
  --resource-group myRG \
  --template-file main.bicep

az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters @prod.bicepparam            # use param file

az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters key=value                   # inline parameter

az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --what-if                                # preview changes (like terraform plan)

az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --confirm-with-what-if                   # preview then confirm

# subscription scope
az deployment sub create \
  --location eastus \
  --template-file main.bicep

# management group scope
az deployment mg create \
  --management-group-id myMG \
  --location eastus \
  --template-file main.bicep

# tenant scope
az deployment tenant create \
  --location eastus \
  --template-file main.bicep
```

## what-if (plan equivalent)

```bash
# show what would change — equivalent to terraform plan
az deployment group what-if \
  --resource-group myRG \
  --template-file main.bicep

az deployment group what-if \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters @prod.bicepparam

# result codes: Create, Delete, Modify, Deploy, Ignore, NoChange
# use --result-format FullResourcePayloads for verbose output
az deployment group what-if \
  --resource-group myRG \
  --template-file main.bicep \
  --result-format FullResourcePayloads
```

## state & deployments

```bash
# bicep has no local state — state is in Azure. use deployment history:
az deployment group list --resource-group myRG           # list deployments
az deployment group show \
  --resource-group myRG \
  --name myDeploymentName                                # show deployment detail

az deployment group delete \
  --resource-group myRG \
  --name myDeploymentName                                # delete deployment record

# export current resource state as ARM template
az group export --name myRG                              # export whole RG
az group export --name myRG --include-parameter-default-value

# list resources in a group (terraform state list equivalent)
az resource list --resource-group myRG -o table
az resource show \
  --resource-group myRG \
  --name myResource \
  --resource-type "Microsoft.Compute/virtualMachines"
```

## output & inspect

```bash
# show outputs from a completed deployment
az deployment group show \
  --resource-group myRG \
  --name myDeployment \
  --query properties.outputs

# show a specific output value
az deployment group show \
  --resource-group myRG \
  --name myDeployment \
  --query "properties.outputs.vmPublicIp.value" \
  -o tsv

# generate ARM JSON without deploying (for inspection)
bicep build main.bicep --outfile /tmp/arm.json
cat /tmp/arm.json | jq .resources
```

## validate

```bash
# validate template without deploying
az deployment group validate \
  --resource-group myRG \
  --template-file main.bicep

az deployment group validate \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters @prod.bicepparam

# lint only (no Azure connection needed)
bicep lint main.bicep

# build compiles and reports errors
bicep build main.bicep
```

## file structure (recommended layout)

```
project/
├── main.bicep            # primary resources
├── main.bicepparam       # parameter values for an environment (replaces .tfvars)
├── variables.bicep       # shared param/type definitions (optional pattern)
├── outputs.bicep         # not standard — outputs live in main.bicep
├── modules/
│   └── vm/
│       ├── main.bicep    # module resources
│       └── README.md
└── .bicepconfig          # bicep linter/alias config (like .terraform.lock.hcl)
```

## configuration syntax

### provider / target scope

```bicep
// set deployment scope at top of file
targetScope = 'resourceGroup'    // default — can omit
// targetScope = 'subscription'
// targetScope = 'managementGroup'
// targetScope = 'tenant'
```

### parameters

```bicep
param location string = resourceGroup().location
param instanceCount int = 2
param env string                  // required — no default

@description('VM admin username')
param adminUsername string

@secure()                         // masked in logs — like sensitive = true
param adminPassword string

@allowed(['dev', 'staging', 'prod'])
param environment string = 'dev'

@minValue(1)
@maxValue(10)
param nodeCount int = 3
```

### variables (locals equivalent)

```bicep
var namePrefix = '${env}-${project}'
var commonTags = {
  env: env
  project: project
  owner: 'ak'
}
```

### outputs

```bicep
output vmPublicIp string = vm.properties.publicIPAddress
output resourceId string = storageAccount.id

@description('Connection string — treat as secret')
@secure()
output connectionString string = listKeys(storageAccount.id, storageAccount.apiVersion).keys[0].value
```

### resources

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'mystorageacct'
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  tags: commonTags
}

// reference another resource's property
resource vm 'Microsoft.Compute/virtualMachines@2023-03-01' = {
  name: 'my-vm'
  location: location
  properties: {
    // reference storageAccount defined above
    diagnosticsProfile: {
      bootDiagnostics: {
        storageUri: storageAccount.properties.primaryEndpoints.blob
      }
    }
  }
}
```

### existing resources (data source equivalent)

```bicep
// reference an existing resource without managing it
resource existingVnet 'Microsoft.Network/virtualNetworks@2023-04-01' existing = {
  name: 'my-existing-vnet'
  // scope: resourceGroup('other-rg')  // optionally scope to another RG
}

// use it like any resource
resource subnet 'Microsoft.Network/virtualNetworks/subnets@2023-04-01' existing = {
  parent: existingVnet
  name: 'default'
}
```

### loops (count / for_each equivalent)

```bicep
// array loop — like count
resource vms 'Microsoft.Compute/virtualMachines@2023-03-01' = [for i in range(0, instanceCount): {
  name: 'vm-${i}'
  location: location
  // ...
}]

// object/array loop — like for_each
var storageNames = ['logs', 'backups', 'assets']
resource buckets 'Microsoft.Storage/storageAccounts@2023-01-01' = [for name in storageNames: {
  name: '${namePrefix}${name}'
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}]
```

### conditions (lifecycle / count = 0 equivalent)

```bicep
resource diagStorage 'Microsoft.Storage/storageAccounts@2023-01-01' = if (env == 'prod') {
  name: 'diagstorageprod'
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}
```

### depends_on

```bicep
resource appService 'Microsoft.Web/sites@2023-01-01' = {
  name: 'my-app'
  location: location
  dependsOn: [
    sqlServer   // explicit dependency — usually implicit via property references
  ]
  // ...
}
```

### expressions & functions

```bicep
// conditionals
var sku = env == 'prod' ? 'Standard_LRS' : 'Standard_GRS'

// string interpolation
var fullName = '${namePrefix}-storage'

// common functions
resourceGroup().location          // current RG location
resourceGroup().id                // current RG resource ID
subscription().subscriptionId     // current subscription ID
uniqueString(resourceGroup().id)  // deterministic hash — great for unique names
concat('a', 'b', 'c')
toLower('STRING')
toUpper('string')
contains(['a','b'], 'a')          // true
length(array)
first(array)
last(array)
union(obj1, obj2)                 // merge objects — like merge() in terraform
items(obj)                        // iterate object as array of {key, value}
range(0, 5)                       // [0,1,2,3,4]
format('{0}-{1}', var1, var2)
base64(string)
base64ToString(encoded)
listKeys(resourceId, apiVersion)  // get keys/secrets from resources
```

### .bicepparam file (tfvars equivalent)

```bicep
// prod.bicepparam
using './main.bicep'

param location = 'eastus'
param env = 'prod'
param instanceCount = 3
param adminPassword = readEnvironmentVariable('ADMIN_PASSWORD')  // from env var
```

### .bicepconfig (linter config)

```json
{
  "analyzers": {
    "core": {
      "rules": {
        "no-hardcoded-env-urls": { "level": "warning" },
        "no-unused-params": { "level": "warning" },
        "secure-secrets-in-params": { "level": "error" }
      }
    }
  },
  "moduleAliases": {
    "br": {
      "myregistry": {
        "registry": "myacr.azurecr.io"
      }
    }
  }
}
```

## modules

```bicep
// inline module reference (local)
module myVm './modules/vm/main.bicep' = {
  name: 'vmDeployment'
  params: {
    location: location
    env: env
  }
}

// public registry module (like terraform registry)
module network 'br/public:network/virtual-network:1.1.3' = {
  name: 'vnetDeploy'
  params: {
    name: 'my-vnet'
    location: location
    addressPrefixes: ['10.0.0.0/16']
  }
}

// private ACR registry module
module vm 'br:myacr.azurecr.io/bicep/modules/vm:v1.0.0' = {
  name: 'vmModule'
  params: {
    vmName: 'prod-vm'
    location: location
  }
}

// use module output
output networkId string = network.outputs.resourceId
```

## useful patterns

```bash
# what-if then apply in one step
az deployment group what-if \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters @prod.bicepparam && \
az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters @prod.bicepparam

# target a specific module output for inspection
az deployment group show \
  --resource-group myRG \
  --name myDeployment \
  --query "properties.outputs" -o json

# generate unique storage name (common pattern)
# in bicep: name: 'sa${uniqueString(resourceGroup().id)}'

# pass secret from env var via bicepparam
# prod.bicepparam: param adminPassword = readEnvironmentVariable('ADMIN_PASSWORD')
# then: ADMIN_PASSWORD=hunter2 az deployment group create ...

# delete all resources in a RG (destroy equivalent)
az group delete --name myRG --yes --no-wait

# export existing RG as ARM then decompile to bicep
az group export --name myRG > exported.json
bicep decompile exported.json

# check deployment status
az deployment group list --resource-group myRG -o table
```

## terraform → bicep concept map

| Terraform | Bicep |
|---|---|
| `terraform plan` | `az deployment group what-if` |
| `terraform apply` | `az deployment group create` |
| `terraform destroy` | `az group delete` or `az resource delete` |
| `terraform validate` | `bicep lint` / `az deployment group validate` |
| `terraform fmt` | `bicep format` |
| `terraform state list` | `az resource list --resource-group` |
| `terraform import` | `resource ... existing =` |
| `terraform output` | `az deployment group show --query properties.outputs` |
| `.tfvars` / `.tfvars.json` | `.bicepparam` |
| `variable {}` | `param` |
| `locals {}` | `var` |
| `output {}` | `output` |
| `data "..." {}` | `resource ... existing =` |
| `count` | `[for i in range(0, n):]` |
| `for_each` | `[for item in array:]` |
| `sensitive = true` | `@secure()` |
| `depends_on` | `dependsOn:` |
| `lifecycle { prevent_destroy }` | resource locks via `Microsoft.Authorization/locks` |
| `terraform.tfstate` | Azure Resource Manager (no local state) |
| Registry module | `br/public:...` or ACR `br:...` |
| Local module | `./modules/name/main.bicep` |

## bicep → terraform conversion examples

Side-by-side translations for common patterns. Bicep on the left, equivalent Terraform on the right.

---

### parameters / variables

```bicep
// bicep
param location string = resourceGroup().location
param env string = 'prod'
param nodeCount int = 3

@secure()
param adminPassword string

@allowed(['dev', 'staging', 'prod'])
param tier string = 'dev'
```

```hcl
# terraform
variable "location" {
  type    = string
  default = "eastus"   # no resourceGroup() equivalent at var level — set in tfvars
}

variable "env" {
  type    = string
  default = "prod"
}

variable "node_count" {
  type    = number
  default = 3
}

variable "admin_password" {
  type      = string
  sensitive = true
}

variable "tier" {
  type    = string
  default = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.tier)
    error_message = "tier must be dev, staging, or prod"
  }
}
```

---

### locals / var

```bicep
// bicep
var namePrefix = '${env}-${project}'
var commonTags = {
  env: env
  project: project
  owner: 'ak'
}
```

```hcl
# terraform
locals {
  name_prefix = "${var.env}-${var.project}"
  common_tags = {
    env     = var.env
    project = var.project
    owner   = "ak"
  }
}
```

---

### storage account

```bicep
// bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name:     'mystorageacct'
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  tags: commonTags
}
```

```hcl
# terraform (azurerm provider)
resource "azurerm_storage_account" "this" {
  name                     = "mystorageacct"
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  account_kind             = "StorageV2"
  tags                     = local.common_tags
}
```

---

### virtual machine

```bicep
// bicep
resource vm 'Microsoft.Compute/virtualMachines@2023-03-01' = {
  name:     'my-vm'
  location: location
  properties: {
    hardwareProfile: { vmSize: 'Standard_B2s' }
    osProfile: {
      computerName:  'my-vm'
      adminUsername: adminUsername
      adminPassword: adminPassword
    }
    storageProfile: {
      imageReference: {
        publisher: 'Canonical'
        offer:     '0001-com-ubuntu-server-jammy'
        sku:       '22_04-lts'
        version:   'latest'
      }
      osDisk: {
        createOption: 'FromImage'
        managedDisk: { storageAccountType: 'Standard_LRS' }
      }
    }
    networkProfile: {
      networkInterfaces: [{
        id: nic.id
      }]
    }
  }
  tags: commonTags
}
```

```hcl
# terraform
resource "azurerm_linux_virtual_machine" "this" {
  name                = "my-vm"
  resource_group_name = var.resource_group_name
  location            = var.location
  size                = "Standard_B2s"
  admin_username      = var.admin_username
  admin_password      = var.admin_password

  disable_password_authentication = false

  network_interface_ids = [azurerm_network_interface.this.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }

  tags = local.common_tags
}
```

---

### existing resource (data source)

```bicep
// bicep — reference an existing resource without managing it
resource existingVnet 'Microsoft.Network/virtualNetworks@2023-04-01' existing = {
  name: 'my-existing-vnet'
}

resource subnet 'Microsoft.Network/virtualNetworks/subnets@2023-04-01' existing = {
  parent: existingVnet
  name:   'default'
}

// use it
resource nic 'Microsoft.Network/networkInterfaces@2023-04-01' = {
  name:     'my-nic'
  location: location
  properties: {
    ipConfigurations: [{
      name: 'ipconfig1'
      properties: {
        subnet: { id: subnet.id }
      }
    }]
  }
}
```

```hcl
# terraform — data source for existing resource
data "azurerm_virtual_network" "existing" {
  name                = "my-existing-vnet"
  resource_group_name = var.resource_group_name
}

data "azurerm_subnet" "default" {
  name                 = "default"
  virtual_network_name = data.azurerm_virtual_network.existing.name
  resource_group_name  = var.resource_group_name
}

resource "azurerm_network_interface" "this" {
  name                = "my-nic"
  location            = var.location
  resource_group_name = var.resource_group_name

  ip_configuration {
    name                          = "ipconfig1"
    subnet_id                     = data.azurerm_subnet.default.id
    private_ip_address_allocation = "Dynamic"
  }
}
```

---

### loops — array (count equivalent)

```bicep
// bicep
resource vms 'Microsoft.Compute/virtualMachines@2023-03-01' = [for i in range(0, nodeCount): {
  name:     'vm-${i}'
  location: location
  // ...
}]
```

```hcl
# terraform
resource "azurerm_linux_virtual_machine" "this" {
  count               = var.node_count
  name                = "vm-${count.index}"
  location            = var.location
  resource_group_name = var.resource_group_name
  # ...
}
```

---

### loops — named set (for_each equivalent)

```bicep
// bicep
var storageNames = ['logs', 'backups', 'assets']

resource buckets 'Microsoft.Storage/storageAccounts@2023-01-01' = [for name in storageNames: {
  name:     '${namePrefix}${name}'
  location: location
  sku:  { name: 'Standard_LRS' }
  kind: 'StorageV2'
}]
```

```hcl
# terraform
resource "azurerm_storage_account" "this" {
  for_each = toset(["logs", "backups", "assets"])

  name                     = "${local.name_prefix}${each.key}"
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  account_kind             = "StorageV2"
}
```

---

### conditionals

```bicep
// bicep — resource only deployed when condition is true
resource diagStorage 'Microsoft.Storage/storageAccounts@2023-01-01' = if (env == 'prod') {
  name:     'diagstorageprod'
  location: location
  sku:  { name: 'Standard_LRS' }
  kind: 'StorageV2'
}
```

```hcl
# terraform — count = 0 skips creation
resource "azurerm_storage_account" "diag" {
  count = var.env == "prod" ? 1 : 0

  name                     = "diagstorageprod"
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  account_kind             = "StorageV2"
}
```

---

### outputs

```bicep
// bicep
output storageEndpoint string = storageAccount.properties.primaryEndpoints.blob
output vmId string = vm.id

@secure()
output connectionString string = listKeys(storageAccount.id, storageAccount.apiVersion).keys[0].value
```

```hcl
# terraform
output "storage_endpoint" {
  value = azurerm_storage_account.this.primary_blob_endpoint
}

output "vm_id" {
  value = azurerm_linux_virtual_machine.this.id
}

output "connection_string" {
  value     = azurerm_storage_account.this.primary_connection_string
  sensitive = true
}
```

---

### modules

```bicep
// bicep — local module
module myVm './modules/vm/main.bicep' = {
  name: 'vmDeployment'
  params: {
    location: location
    env:      env
  }
}

output vmId string = myVm.outputs.vmId
```

```hcl
# terraform — local module
module "my_vm" {
  source   = "./modules/vm"
  location = var.location
  env      = var.env
}

output "vm_id" {
  value = module.my_vm.vm_id
}
```

---

### depends_on

```bicep
// bicep — explicit dependency (usually implicit via property refs)
resource appService 'Microsoft.Web/sites@2023-01-01' = {
  name:     'my-app'
  location: location
  dependsOn: [sqlServer]
  // ...
}
```

```hcl
# terraform — explicit dependency (usually implicit via resource references)
resource "azurerm_app_service" "this" {
  name                = "my-app"
  location            = var.location
  resource_group_name = var.resource_group_name
  app_service_plan_id = azurerm_app_service_plan.this.id

  depends_on = [azurerm_sql_server.this]
}
```

---

### prevent destroy / resource locks

```bicep
// bicep — no native lifecycle; use an ARM resource lock
resource dbLock 'Microsoft.Authorization/locks@2020-05-01' = {
  name:  'db-do-not-delete'
  scope: sqlServer
  properties: {
    level: 'CanNotDelete'
    notes: 'Prevent accidental deletion of production DB'
  }
}
```

```hcl
# terraform — lifecycle block on the resource itself
resource "azurerm_mssql_server" "this" {
  name                         = "my-sql-server"
  resource_group_name          = var.resource_group_name
  location                     = var.location
  version                      = "12.0"
  administrator_login          = var.admin_username
  administrator_login_password = var.admin_password

  lifecycle {
    prevent_destroy = true
  }
}
```

---

### unique name generation

```bicep
// bicep — deterministic unique suffix from RG ID
var suffix = uniqueString(resourceGroup().id)
resource sa 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'sa${suffix}'    // e.g. "sa3x7kqm2p4n6"
  // ...
}
```

```hcl
# terraform — random_id resource or use known unique values
resource "random_id" "suffix" {
  byte_length = 6
}

resource "azurerm_storage_account" "this" {
  name = "sa${random_id.suffix.hex}"   # e.g. "sa3f9a1b2c"
  # ...
}

# or deterministic from known values
resource "azurerm_storage_account" "this" {
  name = "sa${substr(md5(var.resource_group_name), 0, 10)}"
}
```

---

### param file (.bicepparam → .tfvars)

```bicep
// prod.bicepparam
using './main.bicep'

param location      = 'eastus'
param env           = 'prod'
param nodeCount     = 3
param adminPassword = readEnvironmentVariable('ADMIN_PASSWORD')
```

```hcl
# prod.tfvars
location       = "eastus"
env            = "prod"
node_count     = 3
# admin_password not stored in file — pass via:
# TF_VAR_admin_password=hunter2 terraform apply
# or: terraform apply -var="admin_password=$ADMIN_PASSWORD"
```
