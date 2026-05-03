# Standard Variables, AVM Rules, and Examples

The set of variables every Azure module declares (the org-wide standard set), plus the AVM (Azure Verified Modules) rules that apply to variable design, plus example/Terratest variable conventions and CHANGELOG requirements. The main `SKILL.md` and `complex-types.md` cover patterns; this file covers the org-wide contract.

## Contents
- [The Standard Variable Set](#the-standard-variable-set)
- [Complete Example: Putting It All Together](#complete-example-putting-it-all-together)
- [AVM Rules That Apply to Variables](#avm-rules-that-apply-to-variables)
- [Deprecated Variables](#deprecated-variables)
- [Feature Toggles for New Resources](#feature-toggles-for-new-resources)
- [Example File Conventions](#example-file-conventions)
- [Terratest Variable Conventions](#terratest-variable-conventions)
- [Terraform Version Considerations](#terraform-version-considerations)
- [CHANGELOG Requirements for Variable Changes](#changelog-requirements-for-variable-changes)

## The Standard Variable Set

Every module **must** include these variables. Copy them verbatim and adjust only when the module has a specific reason to deviate. Brand codes and locations are examples — adapt the allowed-value list to your organization's set.

```hcl
# ---------- brand ----------
variable "brand" {
  description = "The brand abbreviation. Must be 2-10 lowercase alphanumeric characters."
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9]{2,10}$", var.brand))
    error_message = "The `brand` must be 2-10 lowercase alphanumeric characters (e.g., `acme`)."
  }
}

# ---------- environment ----------
variable "environment" {
  description = "The deployment environment. Allowed values: `dev`, `tst`, `stg`, `uat`, `prd`, `qa`."
  type        = string

  validation {
    condition     = contains(["dev", "tst", "stg", "uat", "prd", "qa"], var.environment)
    error_message = "Allowed values for `environment` are: `dev`, `tst`, `stg`, `uat`, `prd`, `qa`."
  }
}

# ---------- project_name ----------
variable "project_name" {
  description = "Short project name used in resource naming. Must be 2-10 lowercase alphanumeric characters."
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9]{2,10}$", var.project_name))
    error_message = "The `project_name` must be 2-10 lowercase alphanumeric characters."
  }
}

# ---------- costcenter ----------
variable "costcenter" {
  description = "Cost center code for billing. Must match the pattern `XXXX-XXXX`."
  type        = string

  validation {
    condition     = can(regex("^[0-9]{4}-[0-9]{4}$", var.costcenter))
    error_message = "The `costcenter` must follow the format `XXXX-XXXX` (e.g., `1234-5678`)."
  }
}

# ---------- projectcode ----------
variable "projectcode" {
  description = "Project code for cost tracking. Must match the pattern `XXXX-XXXX`."
  type        = string

  validation {
    condition     = can(regex("^[0-9]{4}-[0-9]{4}$", var.projectcode))
    error_message = "The `projectcode` must follow the format `XXXX-XXXX` (e.g., `1234-5678`)."
  }
}

# ---------- location ----------
variable "location" {
  description = "Azure region for resource deployment. Allowed values: `eastus`, `eastus2`, `westus2`."
  type        = string

  validation {
    condition     = contains(["eastus", "eastus2", "westus2"], var.location)
    error_message = "Allowed values for `location` are: `eastus`, `eastus2`, `westus2`."
  }
}

# ---------- resource_group_name ----------
variable "resource_group_name" {
  description = "Name of the Azure Resource Group where resources will be created."
  type        = string

  validation {
    condition     = length(var.resource_group_name) > 0
    error_message = "The `resource_group_name` must not be empty."
  }
}

# ---------- tags ----------
variable "tags" {
  description = "Additional tags to apply to all resources. Standard tags (`brand`, `environment`, `costcenter`, `projectcode`) are merged automatically."
  type        = map(string)
  default     = {}
  nullable    = false
}
```

### Why these variables and not others

- `brand`, `environment`, `project_name` — drive resource naming. Validating shape here means every downstream `local.resource_name = "${var.brand}-..."` is guaranteed to produce a valid Azure name.
- `costcenter`, `projectcode` — emitted as tags for billing. Validating format prevents broken cost reports.
- `location`, `resource_group_name` — required by virtually every Azure resource.
- `tags` — `default = {}` + `nullable = false` so consumers can omit it, and so the module can always merge against `local.default_tags` without null-checks.

### Tag merge order

```hcl
# Bad — caller can override mandatory default tags (brand, environment, etc.)
locals {
  tags = merge(local.default_tags, var.tags)
}

# Good — defaults last so they take precedence over user-supplied tags
locals {
  tags = merge(var.tags, local.default_tags)
}
```

## Complete Example: Putting It All Together

Below is the module-specific portion of a well-designed `variables.tf`. Place it after the standard variable set above. Comment dividers separate logical groupings.

```hcl
# ──────────────────────────────────────────────
# (Standard Variables block from above)
# ──────────────────────────────────────────────

# ──────────────────────────────────────────────
# Module-Specific Variables
# ──────────────────────────────────────────────

variable "sku" {
  description = "SKU tier. Allowed values: `Basic`, `Standard`, `Premium`."
  type        = string

  validation {
    condition     = contains(["Basic", "Standard", "Premium"], var.sku)
    error_message = "Allowed values for `sku` are: `Basic`, `Standard`, `Premium`."
  }
}

variable "identity" {
  description = <<-EOT
    Managed identity configuration.
    ```
    type = object({
      type         = string
      identity_ids = optional(list(string), [])
    })
    ```

    `type` - (Required) Allowed values: `SystemAssigned`, `UserAssigned`, `SystemAssigned, UserAssigned`.
    `identity_ids` - (Optional) User Assigned Identity resource IDs.
  EOT

  type = object({
    type         = string
    identity_ids = optional(list(string), [])
  })
  nullable = false

  validation {
    condition     = contains(["SystemAssigned", "UserAssigned", "SystemAssigned, UserAssigned"], var.identity.type)
    error_message = "Allowed values for `identity.type` are: `SystemAssigned`, `UserAssigned`, `SystemAssigned, UserAssigned`."
  }
}

variable "network_config" {
  description = <<-EOT
    Network configuration for the resource.
    ```
    type = object({
      subnet_id         = string
      private_endpoint  = optional(bool, false)
      allowed_ip_ranges = optional(list(string), [])
    })
    ```

    `subnet_id` - (Required) Resource ID of the subnet.
    `private_endpoint` - (Optional) Enable private endpoint. Defaults to `false`.
    `allowed_ip_ranges` - (Optional) Allowed IP CIDR ranges.
  EOT

  type = object({
    subnet_id         = string
    private_endpoint  = optional(bool, false)
    allowed_ip_ranges = optional(list(string), [])
  })
  nullable = false

  validation {
    condition = alltrue([
      for cidr in var.network_config.allowed_ip_ranges : can(cidrhost(cidr, 0))
    ])
    error_message = "Each entry in `allowed_ip_ranges` must be a valid CIDR block."
  }
}

variable "custom_domain" {
  description = "Custom domain name for the service."
  type        = string
  default     = null

  validation {
    condition     = var.custom_domain == null || can(regex("^[a-z0-9][a-z0-9.-]+\\.[a-z]{2,}$", var.custom_domain))
    error_message = "The `custom_domain`, when provided, must be a valid domain name."
  }
}

variable "name_override" {
  description = "This variable will override the default naming convention when required. Must be used for migration or importing existing resources."
  type        = string
  default     = null
}
```

## AVM Rules That Apply to Variables

### No `sensitive = false` declarations (AVM TFNFR22)

`sensitive = false` is the Terraform default. Explicitly writing it adds noise.

```hcl
# Bad
variable "instance_name" {
  type      = string
  sensitive = false
}

# Good
variable "instance_name" {
  type = string
}
```

### No defaults for sensitive inputs (AVM TFNFR23)

Sensitive variables (passwords, secrets, tokens) **must not** have default values. A default password is a security risk.

```hcl
# Bad — default password is a security incident waiting to happen
variable "admin_password" {
  type      = string
  sensitive = true
  default   = "P@ssw0rd123!"
}

# Good — caller must provide
variable "admin_password" {
  type      = string
  sensitive = true
}
```

### No module-level control variables (AVM TFNFR14)

Do not add variables like `enabled` or `module_depends_on` to control the entire module's operation. Boolean toggles for **specific resources** are acceptable.

```hcl
# Bad — module-level kill switch
variable "enabled" {
  type    = bool
  default = true
}

# Good — resource-specific toggle
variable "create_dns_records" {
  description = "Whether to create DNS records."
  type        = bool
  default     = false
  nullable    = false
}
```

### Variable ordering — by purpose, not alphabetical

AVM TFNFR15 recommends alphabetical ordering with required-first. Many organizations override this in favor of grouping by purpose (standard variables first, then module-specific groups). Pick one convention and apply it consistently across all modules in your registry.

## Deprecated Variables

When a variable is deprecated (AVM TFNFR24):

1. Move it to `deprecated_variables.tf`.
2. Prefix the description with `DEPRECATED`.
3. Name the replacement variable in the description.
4. Remove during the next major version release.

```hcl
# deprecated_variables.tf
variable "vnet_rg" {
  description = "DEPRECATED: Use `resource_group_name` instead. The resource group for the VNet."
  type        = string
  default     = null
}
```

## Feature Toggles for New Resources

New resources added in **minor or patch** versions **must** be guarded by a toggle variable so existing consumers don't unexpectedly get new resources on a `terraform apply` (AVM TFNFR34):

```hcl
variable "create_route_table" {
  description = "Whether to create the route table. Set to `true` to enable."
  type        = bool
  default     = false
  nullable    = false
}

resource "azurerm_route_table" "main" {
  count = var.create_route_table ? 1 : 0
  # ...
}
```

The default is **false** — opt-in only. The next major version may flip the default to `true` (a breaking change requiring `CHANGELOG.md`).

## Example File Conventions

Examples are consumer-facing documentation. They must be simple, realistic, and follow strict conventions.

### Example `variables.tf` — keep it minimal

Example variable files declare variables **without types, descriptions, or validation**. The Terraform Cloud / Enterprise workspace provides the actual values.

```hcl
# Good — examples/main/variables.tf
variable "brand" {}
variable "project_name" {}
variable "projectcode" {}
variable "costcenter" {}
variable "environment" {}
variable "location" {}
variable "resource_group_name" {}
```

```hcl
# Bad — full variable definitions in examples
variable "brand" {
  description = "The brand abbreviation."
  type        = string
  validation {
    condition     = contains(["acme"], var.brand)
    error_message = "Invalid brand."
  }
}
```

### No Azure credential variables in examples

Examples must **not** declare `subscription_id`, `client_id`, `client_secret`, or `tenant_id` variables. The workspace injects these via environment variables (`ARM_SUBSCRIPTION_ID`, `ARM_TENANT_ID`, etc.).

```hcl
# Bad — examples/main/variables.tf
variable "subscription_id" {}
variable "client_id" {}
variable "client_secret" { sensitive = true }
variable "tenant_id" {}

# Bad — examples/main/providers.tf
provider "azurerm" {
  subscription_id = var.subscription_id
  client_id       = var.client_id
  client_secret   = var.client_secret
  tenant_id       = var.tenant_id
  features {}
}
```

```hcl
# Good — examples/main/providers.tf (consumer-style)
provider "azurerm" {
  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id
  features {}
}

provider "azapi" {
  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id
}
```

### `var.` references and `{placeholder}` values

Use `var.` references everywhere possible. For hardcoded non-variable values (like ARM resource IDs), use `{placeholder}` notation to signal the value is not a constant the consumer should copy verbatim.

```hcl
# Good — reference variables
resource_group_name = var.resource_group_name

# Good — placeholder for ARM IDs
apps_subnet_id = "/subscriptions/{subscriptionID}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/virtualNetworks/{vnetName}/subnets/{subnetName}"

# Bad — hardcoded literal values
resource_group_name = "my-resource-group"
```

### Use `example` not `test` in example values

Example names and descriptions must use the prefix `example`, never `test`. Resource-type suffixes go at the end. Keys may use `_01`, `_02` numeric suffixes.

```hcl
# Good
short_name = "example-01-ag"
name       = "example-custom-role-01-name"

# Bad
short_name = "test-ag"
name       = "test-role-01"
```

### Don't use `name_override` in examples

Examples should always demonstrate the default naming convention. `name_override` is for migration and import only.

## Terratest Variable Conventions

### No `.auto.tfvars` files

Terratest plans must not use `main.auto.tfvars`. Move fixed test settings into variable defaults and locals. Runtime switches go through environment variables.

```hcl
# Good — terratest/plan/variables.tf (defaults baked in)
variable "location"    { default = "eastus" }
variable "projectcode" { default = "0000-0000" }
variable "costcenter"  { default = "9100-0021" }

# Bad — terratest/plan/main.auto.tfvars (extra file dependency)
location    = "eastus"
projectcode = "0000-0000"
costcenter  = "9100-0021"
```

### Terratest `main.tf` — `for_each` with a map local

Terratest module calls should pass configuration through a `for_each` map for consistency with how consumers iterate the module:

```hcl
# Good — terratest/plan/main.tf
locals {
  example = {
    "instance_01" = {
      sku = "Standard"
    }
  }
}

module "main" {
  source   = "../.."
  for_each = local.example

  brand               = var.brand
  costcenter          = var.costcenter
  projectcode         = var.projectcode
  environment         = var.environment
  location            = var.location
  project_name        = var.project_name
  resource_group_name = var.resource_group_name

  sku = each.value.sku
}
```

## Terraform Version Considerations

### `nullable = false` requires Terraform >= 1.3

When adopting `nullable = false`, `versions.tf` must declare `required_version = ">= 1.3.0"`. Document the bump as a breaking change in `CHANGELOG.md`.

```hcl
# versions.tf
terraform {
  required_version = ">= 1.3.0"
}
```

### `optional()` for object attributes requires Terraform >= 1.3

Same constraint — `optional(string)` and `optional(string, "default")` are 1.3+ features.

### Multi-condition validation with cross-variable references requires Terraform >= 1.9

If a `validation` block needs to reference another variable (`!var.enable_monitoring || var.workspace_id != null`), bump `required_version = ">= 1.9.0"`.

### Ephemeral resources for secrets require Terraform >= 1.10

When reading secrets from Key Vault, prefer `ephemeral` resources so secrets are not persisted in state:

```hcl
# Good — secret not stored in state (Terraform 1.10+)
ephemeral "azurerm_key_vault_secret" "main" {
  name         = var.eventhub.secret_name
  key_vault_id = var.eventhub.key_vault_id
}

# Avoid when 1.10+ is available — secret persists in state
data "azurerm_key_vault_secret" "main" {
  name         = var.eventhub.secret_name
  key_vault_id = var.eventhub.key_vault_id
}
```

## CHANGELOG Requirements for Variable Changes

Every variable change must be documented in `CHANGELOG.md`. The following are **breaking changes** that require a major version bump:

1. **Deleted variables** — always breaking.
2. **Adding `nullable = false`** — breaking if any caller was passing `null`.
3. **Changing defaults** — may affect existing deployments.
4. **Renaming variables** — breaking; use the deprecated variables pattern for migration.
5. **Tightening validation** — breaking if previously-accepted values now fail.
6. **Bumping `required_version`** — breaks consumers on older Terraform CLIs.

```markdown
## [2.0.0] - 2026-04-07

### Breaking Changes
- Removed `subscription_id` variable (unused in root module).
- Update the Terraform version to `>= 1.3.0` as required by `optional()` for attributes in `variables.tf`.
- Changed `network_config` to use `nullable = false`.
- Renamed `vnet_rg` to `resource_group_name` (old name moved to `deprecated_variables.tf`).

### Changed
- Grouped `master_*` and `worker_*` variables into `master_profile` and `worker_profile` objects.
- Changed optional string defaults from `""` to `null`.

### Added
- Added `create_route_table` toggle (defaults to `false`; opt-in for new resource).
```

Non-breaking changes (new variables with sensible defaults, looser validation, additional optional attributes on object variables) go under `### Added` or `### Changed` without the breaking flag.
