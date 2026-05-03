---
name: terraform-resource-naming
description: Apply consistent naming conventions across Azure Terraform modules — both Azure resource names (the `name = ...` argument: lowercase, hyphens, resource-type suffix) and HCL identifier names (variables, outputs, locals, resource labels: snake_case). Use this skill whenever you are choosing or reviewing a name in a module: naming an Azure resource (Key Vault, storage account, subnet, NSG, gateway, ARO cluster, etc.), picking a resource-type suffix (kv, st, snet, nsg, gw), constructing the `primary_name` local, validating naming inputs, naming a `for_each` instance, or auditing existing names against the convention. Triggers on phrases like "name an Azure resource", "what suffix should I use", "how to name a Key Vault / storage account / subnet", "Azure naming convention", "primary_name", "name_override", "resource label". Sibling skills own `locals.tf`, `variables.tf`, and `outputs.tf` — this skill owns the conventions applied across all three.
---

# Terraform Resource Naming

Naming touches every file in a module. Get it right and the module is grep-friendly and produces Azure resources that fit the platform-wide scheme. Get it wrong and the module name leaks into every consumer call, Azure's per-resource validation rules break, and a later cleanup forces destroy-and-recreate.

This skill covers two distinct namespaces:

1. **HCL identifier names** — variables, outputs, locals, resource labels, module names. Convention: `snake_case`.
2. **Azure resource names** — the `name = "..."` argument Azure sees. Convention: lowercase, hyphen-separated, ending in a resource-type suffix.

For `locals.tf` patterns see `terraform-locals-patterns`; for `variables.tf` see `terraform-variable-design`; for `outputs.tf` see `terraform-module-outputs`.

## Contents
- [HCL Resource Labels](#hcl-resource-labels)
- [HCL Identifier Naming](#hcl-identifier-naming)
- [`primary_name` — the Shared Naming Base](#primary_name--the-shared-naming-base)
- [Azure Resource Naming Pattern](#azure-resource-naming-pattern)
- [Resource-Type Suffix Table](#resource-type-suffix-table)
- [`for_each` Resources Must Produce Unique Names](#for_each-resources-must-produce-unique-names)
- [`name_override` Convention](#name_override-convention)
- [Validate Naming Inputs at the Variable](#validate-naming-inputs-at-the-variable)
- [DNS Records, Map Keys, and Module Labels](#dns-records-map-keys-and-module-labels)
- [Backward Compatibility — Don't Rename Casually](#backward-compatibility--dont-rename-casually)
- [Quick Reference](#quick-reference)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)

## HCL Resource Labels

The HCL label (the string after the resource type) describes the resource's role — not its Azure name and not a number.

- **Single resource of its type → `main`.** Reviewers grep for `.main` first.
- **Multiple resources → descriptive labels** that convey purpose (`worker`, `master`, `api_server`).
- **Never use numbered labels** like `cluster01`, `vm01`, `subnet01` — they carry no meaning and lock the module to a count. If you need many of one type with dynamic keys, use `for_each`.

```hcl
# Good — only one of its type
resource "azurerm_virtual_network" "main" {}
resource "azurerm_redhat_openshift_cluster" "main" {}

# Good — multiples, distinguished by purpose
resource "azurerm_subnet" "worker" {}
resource "azurerm_subnet" "master" {}
resource "infoblox_a_record" "api_server" {}
resource "infoblox_a_record" "ingress"    {}

# Bad — numbered label is meaningless
resource "azurerm_redhat_openshift_cluster" "cluster01" {}
resource "azurerm_subnet" "subnet01" {}
```

## HCL Identifier Naming

All HCL identifiers — variables, outputs, locals, resource labels — use **snake_case**.

### Outputs: no module-name prefix

The module is already the namespace. A consumer writes `module.aro.cluster_id` — repeating the module name in the output gives them `module.aro.aro_cluster_id`, which reads awkwardly.

```hcl
# Bad — module is already called "aro"
output "aro_cluster_id" { value = azurerm_redhat_openshift_cluster.main.id }

# Good
output "cluster_id" {
  description = "The ID of the ARO cluster."
  value       = azurerm_redhat_openshift_cluster.main.id
}
```

### Variables: descriptive, standardised where common

Common cross-module variables use a fixed name so consumers wire them up consistently:

```hcl
variable "brand"                {}
variable "environment"          {}
variable "project_name"         {}
variable "location"             {}
variable "resource_group_name"  {}
variable "tags"                 {}
```

Module-specific variables should be descriptive and grouped (`worker_vm_size`, `worker_node_count`, `pod_cidr`) — not vague (`size`, `count1`, `net_block`).

### Don't repeat the service prefix in nested object properties

When a variable is named after a service, the object provides the namespace — don't prefix every property:

```hcl
# Bad — redundant "dbw_" prefix on every property
variable "databricks_config" {
  type = object({
    dbw_name         = string
    dbw_id           = string
    dbw_access_token = string
  })
}

# Good — clean property names; the variable name is the namespace
variable "databricks_config" {
  type = object({
    name          = string
    workspace_id  = string
    access_token  = string
  })
}
```

Use **dot notation** (`var.obj.attr`), not bracket notation (`var.obj["attr"]`), for object attribute access.

### Locals: only when computing or transforming

Drop pass-through locals that just copy a variable. Locals exist to compute or transform.

```hcl
# Bad — pure pass-through, adds no value
locals {
  resource_group_name = var.resource_group_name
  location            = var.location
}

# Good — actual transformation
locals {
  location_code = lookup(var.location_codes, var.location, "unk")
  vnet_name     = format("%s-vnet-01", local.primary_name)
}
```

## `primary_name` — the Shared Naming Base

Every module computes one local called `primary_name` that all Azure resource names build on. Centralising the base means renaming the prefix scheme is a one-line change and every resource shares a stable, recognisable prefix.

```hcl
locals {
  location_code = lookup(var.location_codes, var.location, "unk")
  primary_name  = format("%s%s%s%s", var.brand, var.environment, var.project_name, local.location_code)
}
```

`primary_name` is not the Azure resource name itself — individual names append a hyphen and a resource-type suffix.

## Azure Resource Naming Pattern

```
{brand}-{project_name}-{environment}-{location_code}-{resource-type-suffix}-{index?}
```

- All lowercase.
- Hyphen-separated.
- Resource-type suffix at the **end** (not a prefix or middle segment).
- Optional numeric index only when multiple instances of the same type exist.

```hcl
# Good — type suffix at the end
locals {
  vnet_name    = format("%s-vnet-01", local.primary_name)
  cluster_name = format("%s-aro",     local.primary_name)
  nsg_name     = format("%s-nsg",     local.primary_name)
  subnet_name  = format("%s-snet",    local.primary_name)
  kv_name      = format("%s-kv",      local.primary_name)
}

# Bad — number at the end instead of a type suffix
locals {
  cluster_name = format("%s-01", local.primary_name)
}
```

The type suffix is what makes a name self-describing in the Azure portal. `acme-prod-mlops-eus-kv` is obviously a Key Vault; `acme-prod-mlops-eus-01` is anyone's guess.

## Resource-Type Suffix Table

Use these suffixes consistently. When in doubt, prefer a suffix already used by sibling modules over inventing a new one.

| Resource                                  | Suffix     | Example name                      |
|-------------------------------------------|------------|-----------------------------------|
| Resource Group                            | `rg`       | `acme-prod-mlops-eus-rg`          |
| Virtual Network                           | `vnet`     | `acme-prod-mlops-eus-vnet-01`     |
| Subnet                                    | `snet`     | `acme-prod-mlops-eus-snet-01`     |
| Network Security Group                    | `nsg`      | `acme-prod-mlops-eus-nsg`         |
| Route Table                               | `rt`       | `acme-prod-mlops-eus-rt`          |
| Route Map                                 | `rmap`     | `acme-prod-mlops-eus-rmap`        |
| Virtual Network Gateway                   | `gw`       | `acme-prod-mlops-eus-gw`          |
| Public IP                                 | `ip`       | `acme-prod-mlops-eus-ip`          |
| Load Balancer                             | `lb`       | `acme-prod-mlops-eus-lb`          |
| Application Gateway                       | `agw`      | `acme-prod-mlops-eus-agw`         |
| Front Door                                | `fd`       | `acme-prod-mlops-eus-fd`          |
| Private Endpoint                          | `pe`       | `acme-prod-mlops-eus-pe`          |
| Private DNS Zone                          | `pdns`     | `acme-prod-mlops-eus-pdns`        |
| Key Vault                                 | `kv`       | `acme-prod-mlops-eus-kv`          |
| Storage Account (no hyphens, ≤ 24 chars)  | `st`       | `acmeprodmlopseusst`              |
| Azure Red Hat OpenShift cluster           | `aro`      | `acme-prod-mlops-eus-aro`         |
| Azure Kubernetes Service cluster          | `aks`      | `acme-prod-mlops-eus-aks`         |
| Kubernetes Fleet Manager                  | `fleet`    | `acme-prod-mlops-eus-fleet`       |
| Container Registry (no hyphens)           | `cr`       | `acmeprodmlopseuscr`              |
| Container Apps Environment                | `cae`      | `acme-prod-mlops-eus-cae`         |
| App Service Plan                          | `asp`      | `acme-prod-mlops-eus-asp`         |
| App Service / Web App                     | `app`      | `acme-prod-mlops-eus-app`         |
| Function App                              | `func`     | `acme-prod-mlops-eus-func`        |
| API Management                            | `apim`     | `acme-prod-mlops-eus-apim`        |
| SQL Server                                | `sql`      | `acme-prod-mlops-eus-sql`         |
| SQL Managed Instance                      | `sqlmi`    | `acme-prod-mlops-eus-sqlmi`       |
| MySQL Flexible Server                     | `mysql`    | `acme-prod-mlops-eus-mysql`       |
| PostgreSQL Flexible Server                | `psql`     | `acme-prod-mlops-eus-psql`        |
| Cosmos DB account                         | `cosmos`   | `acme-prod-mlops-eus-cosmos`      |
| Service Bus Namespace                     | `sb`       | `acme-prod-mlops-eus-sb`          |
| Event Hub Namespace                       | `evh`      | `acme-prod-mlops-eus-evh`         |
| Event Grid Topic                          | `evgt`     | `acme-prod-mlops-eus-evgt`        |
| Data Factory                              | `df`       | `acme-prod-mlops-eus-df`          |
| Synapse Workspace                         | `syn`      | `acme-prod-mlops-eus-syn`         |
| Databricks Workspace                      | `dbw`      | `acme-prod-mlops-eus-dbw`         |
| Azure Machine Learning Workspace          | `aml`      | `acme-prod-mlops-eus-aml`         |
| AI Foundry / AI Services                  | `ai`       | `acme-prod-mlops-eus-ai`          |
| Log Analytics Workspace                   | `log`      | `acme-prod-mlops-eus-log`         |
| Application Insights                      | `appi`     | `acme-prod-mlops-eus-appi`        |
| Action Group                              | `ag`       | `acme-prod-mlops-eus-ag`          |
| Recovery Services Vault                   | `rsv`      | `acme-prod-mlops-eus-rsv`         |
| Managed Identity (user-assigned)          | `id`       | `acme-prod-mlops-eus-id`          |
| Network Interface                         | `nic`      | `acme-prod-mlops-eus-nic`         |
| Virtual Machine                           | `vm`       | `acme-prod-mlops-eus-vm`          |
| Virtual Machine Scale Set                 | `vmss`     | `acme-prod-mlops-eus-vmss`        |
| Disk                                      | `disk`     | `acme-prod-mlops-eus-disk`        |

**Storage accounts and container registries** forbid hyphens and uppercase. Strip hyphens when constructing them:

```hcl
locals {
  storage_account_name = replace(format("%s-st", local.primary_name), "-", "")
}
```

## `for_each` Resources Must Produce Unique Names

When a resource is created with `for_each`, the Azure name **must** incorporate `each.key` (or another unique value from `each.value`). Otherwise every iteration produces the same Azure name and apply fails.

```hcl
# Bad — every database gets the same Azure name
resource "azurerm_mssql_database" "main" {
  for_each = var.databases
  name     = format("%s-db", local.primary_name)
}

# Good — each database gets a unique name from the map key
resource "azurerm_mssql_database" "main" {
  for_each = var.databases
  name     = format("%s-%s-db", local.primary_name, each.key)
}
```

If `each.key` would not produce a valid Azure name, sanitise it in a local rather than transforming it inline at every reference.

## `name_override` Convention

The escape hatch for importing existing resources whose names don't follow the standard pattern is a single `name_override` variable.

```hcl
# Standard shape
variable "name_override" {
  description = "This variable will override the default naming convention when required. Must be used for migration or importing existing resources."
  type        = string
  default     = null
}
```

Apply it with `coalesce` — never nest `try()` around it:

```hcl
# Good
resource "azurerm_kubernetes_fleet_manager" "main" {
  name = coalesce(var.name_override, format("%s-fleet", local.primary_name))
}

# Bad — overcomplicated nesting
name = try(coalesce(try(var.name_override, null), format("%s-fleet", local.primary_name)), null)
```

When `name_override` affects multiple sub-resources, document that suffixes are still appended:

```hcl
description = "Setting this will override the default resource naming convention. Suffixes like `-gw`, `-ip` are still appended to sub-resources."
```

**Don't use `name_override` in examples.** Examples should demonstrate the standard pattern; the override exists only for legacy imports.

```hcl
# Bad — example shows name_override
module "fleet_manager" {
  source        = "<your-terraform-registry>/azure/kubernetes-fleet-manager/azurerm"
  name_override = "my-custom-fleet"
}

# Good — example uses the standard naming inputs
module "fleet_manager" {
  source       = "<your-terraform-registry>/azure/kubernetes-fleet-manager/azurerm"
  brand        = var.brand
  project_name = var.project_name
  environment  = var.environment
  location     = var.location
}
```

**Managed/child resource groups** (AKS node RG, ARO infra RG) derive their name from the parent — no separate override variable:

```hcl
resource "azurerm_redhat_openshift_cluster" "main" {
  managed_resource_group_name = "${local.cluster_name}-infra-rg"
}
```

## Validate Naming Inputs at the Variable

Don't sprinkle `lower()` calls across resource blocks to defend against bad input. Add a `validation {}` block to the variable so bad input fails plan immediately and resource bodies stay clean.

```hcl
# Bad — lower() scattered across resource blocks
resource "azurerm_api_management" "main" {
  resource_group_name = lower(var.resource_group_name)
  location            = lower(var.location)
}

# Good — validation enforces correct input
variable "resource_group_name" {
  description = "The name of the resource group where the resources should be created."
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]+[a-z0-9]$", var.resource_group_name))
    error_message = "Only lowercase alphanumeric characters and hyphens are allowed; must not start or end with a hyphen."
  }
}
```

The regex `^[a-z0-9][a-z0-9-]+[a-z0-9]$` is a good baseline for most Azure name fields: starts and ends with an alphanumeric character; only lowercase letters, digits, and hyphens in the middle. Adjust per resource for length limits and for resources that forbid hyphens (e.g. storage accounts: `^[a-z0-9]{3,24}$`).

## DNS Records, Map Keys, and Module Labels

### DNS records — name by purpose

Don't embed the module name or resource type into a DNS record's HCL label:

```hcl
# Bad
resource "infoblox_a_record" "aro_api_server_a_record" {}

# Good
resource "infoblox_a_record" "api_server" {}
```

### Map keys — derive dynamically, never hardcode numbered names

```hcl
# Bad — hardcoded numbered key
locals {
  role_assignments = {
    "cluster01-worker-subnet-corenetwork-contributor" = { ... }
  }
}

# Good — key describes the role, not an instance number
locals {
  role_assignments = {
    "worker-subnet-network-contributor" = { ... }
    "master-subnet-network-contributor" = { ... }
  }
}
```

### Module call labels — meaningful; the label IS the namespace

```hcl
module "service_principal" { source = "./modules/service-principal" }
module "network"           { source = "./modules/network" }
```

Inside a module, do not prefix outputs or resources with the module's own name — `module.service_principal.client_id` is already namespaced. Prefer underscored module labels over hyphenated ones; hyphenated labels force quoting at every reference site.

## Backward Compatibility — Don't Rename Casually

Renaming an HCL resource label forces Terraform to **destroy and recreate** the resource. For stateful resources — storage accounts, SQL servers, Key Vaults — that is data loss.

```hcl
# DANGER — renaming "adls" to "main" destroys and recreates the storage account
# Before: resource "azurerm_storage_account" "adls" {}
# After:  resource "azurerm_storage_account" "main" {}
```

If a rename is genuinely necessary:

1. Document it in `CHANGELOG.md` under **Breaking**.
2. Provide a `terraform state mv` recipe consumers can run before applying.
3. Prefer a major version bump.

The same applies to renaming the Azure `name = "..."` argument — most Azure resource types cannot be renamed in place, so a rename is also a destroy-and-recreate at the cloud level.

## Quick Reference

| Element                  | Convention                                        | Example                                      |
|--------------------------|---------------------------------------------------|----------------------------------------------|
| Single resource label    | `main`                                            | `resource "azurerm_virtual_network" "main"`  |
| Multiple resource labels | Descriptive purpose                               | `"worker"`, `"master"`, `"api_server"`       |
| Outputs                  | `snake_case`, no module prefix                    | `output "cluster_id"`                        |
| Variables                | `snake_case`, standard names where applicable     | `var.resource_group_name`                    |
| Object properties        | No service prefix on nested fields                | `var.databricks_config.name` (not `dbw_name`)|
| Attribute access         | Dot notation, not bracket notation                | `var.network_profile.network_plugin_mode`    |
| Locals                   | Only when computing/transforming                  | `local.primary_name`                         |
| Azure resource names     | Lowercase, hyphens, type suffix at end            | `format("%s-aro", local.primary_name)`       |
| `name_override`          | Standard name + `coalesce`, never in examples     | `coalesce(var.name_override, ...)`           |
| `for_each` names         | Must include `each.key`                           | `format("%s-%s-db", primary_name, each.key)` |
| Map keys                 | Derived dynamically, no hardcoded numbers         | `"worker-subnet-network-contributor"`        |
| Input validation         | Validate format; don't `lower()` at usage         | `can(regex("^[a-z0-9-]+$", var.name))`       |
| Renames                  | Treat as breaking; document `state mv`            | bump major version                           |

## Common Mistakes

1. **Numbered HCL labels** — `cluster01`, `vm02` instead of `main` or descriptive labels.
2. **Module-name prefix on outputs** — `aro_cluster_id` instead of `cluster_id`.
3. **Resource-type suffix in the wrong position** — putting `-01` at the end and `aro` in the middle, instead of `-aro-01`.
4. **`for_each` resources without `each.key` in the name** — produces duplicate Azure names and fails apply.
5. **Hyphens in storage account / container registry names** — Azure rejects them; you must `replace(..., "-", "")`.
6. **Using `name_override` in examples** — examples should demonstrate the standard pattern.
7. **`lower()` scattered across resource blocks** — validate at the variable instead.
8. **Pass-through locals** — `local.resource_group_name = var.resource_group_name` adds no value.
9. **Service-prefix on nested object properties** — `dbw_name`, `dbw_id` inside `var.databricks_config` is redundant.
10. **Renaming HCL labels without thinking** — destroys and recreates the resource; treat as breaking.

## Review Checklist

- [ ] Single-of-its-type resources are labelled `main`; multiples have descriptive labels (no numbers).
- [ ] All HCL identifiers (variables, outputs, locals, labels) are `snake_case`.
- [ ] Outputs do not repeat the module name as a prefix.
- [ ] Nested object properties don't repeat the parent variable's service prefix.
- [ ] Object attribute access uses dot notation, not bracket notation.
- [ ] A `local.primary_name` exists and is the base for every Azure resource name.
- [ ] No pass-through locals — every local computes or transforms.
- [ ] Azure resource names are lowercase, hyphen-separated, with the type suffix at the end.
- [ ] Resource-type suffixes match the table (kv, st, snet, nsg, gw, aro, etc.).
- [ ] `for_each` resources include `each.key` (or another unique value) in the Azure name.
- [ ] `name_override` (if present) uses the standard name and `coalesce`, and is not used in examples.
- [ ] Naming inputs are validated at the variable with a regex — no `lower()` at usage sites.
- [ ] Map keys are derived dynamically, with no hardcoded `01` / `02` numbering.
- [ ] Storage account / container registry names strip hyphens (`replace(..., "-", "")`).
- [ ] No HCL labels or Azure names have been renamed without a documented breaking-change note.
