---
name: terraform-locals-patterns
description: Patterns for `locals.tf` in Azure Terraform modules — location maps (region name to display name and short code), `primary_name` construction, default + user tag merging, `module_version` from a `VERSION` file, safe defaults with `try()`/`lookup()`, diagnostics parsing, DRY role-assignment maps, and `for_each` name-collision prevention. Use this skill whenever you are writing or reviewing `locals.tf`, constructing resource names from brand/env/project/region, merging tags, deriving module version from a `VERSION` file, or transforming complex variable objects. Triggers on phrases like "writing locals", "tag merging", "module version from VERSION file", "primary_name", "location code map". Sibling skills own variables, outputs, and the resource-naming convention itself — keep `locals.tf` content here.
---

# Terraform Locals Patterns

Design `locals.tf` so every entry either transforms, computes, or combines values — never as a plain alias for a variable. Locals are the workshop where module inputs become resource-ready strings, maps, and flags; keep them organised, justify each one, and avoid defensive over-wrapping.

For the variable schemas these locals consume, see `terraform-variable-design`. For the output side, see `terraform-module-outputs`. For the naming convention itself (suffixes, hyphens vs underscores, casing), see `terraform-resource-naming`.

## Contents
- [Purpose of Local Values](#purpose-of-local-values)
- [Standard Locals Every Module Includes](#standard-locals-every-module-includes)
- [Location Maps](#location-maps)
- [The `primary_name` Pattern](#the-primary_name-pattern)
- [`module_version` From a `VERSION` File](#module_version-from-a-version-file)
- [Tag Merging](#tag-merging)
- [Safe Defaults with `try()` and `lookup()`](#safe-defaults-with-try-and-lookup)
- [Diagnostics Parsing](#diagnostics-parsing)
- [DRY Role Assignments with Maps](#dry-role-assignments-with-maps)
- [`for_each` Name Collision Prevention](#for_each-name-collision-prevention)
- [Static Diagnostic Categories Over Data Sources](#static-diagnostic-categories-over-data-sources)
- [Example and Terratest Locals](#example-and-terratest-locals)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)

## Purpose of Local Values

A local is justified only when it does at least one of:

- **Compute a derived value** — resource names, domain names, secret prefixes.
- **Normalise input** — map a user-friendly location string to a canonical form and short code.
- **Assemble tags** — merge user tags with the module's mandatory defaults.
- **Transform configuration** — reshape a complex variable object into the structure resources expect.

Anything else (`local.x = var.x`) is a pass-through and should be deleted. Locals exist to reduce cognitive load, not to add a layer of indirection.

## Standard Locals Every Module Includes

Most Azure modules need the same five locals: a normalised location, a location short code, a `primary_name` base, a `module_version`, and a merged `tags` map.

```hcl
locals {
  # Location normalisation
  location_map = {
    "East US"  = "East US"
    "West US"  = "West US"
    "eastus"   = "East US"
    "westus"   = "West US"
  }
  location_code_map = {
    "East US" = "eus"
    "West US" = "wus"
    "eastus"  = "eus"
    "westus"  = "wus"
  }
  location      = local.location_map[var.location]
  location_code = local.location_code_map[var.location]

  # Primary naming base — see "The primary_name Pattern"
  primary_name = format("%s%s%s%s", var.brand, var.environment, var.project_name, local.location_code)

  # Module version, read from VERSION file at the module root
  module_version = trimspace(file("${path.module}/VERSION"))

  # Default tags — merged with var.tags so defaults always win
  default_tags = {
    brand          = var.brand
    costcenter     = var.costcenter
    projectcode    = var.projectcode
    environment    = var.environment
    location       = local.location
    resource_group = var.resource_group_name
    deployed_using = "Terraform - terraform.azurerm.<module-name> ${local.module_version}"
  }
  tags = merge(var.tags, local.default_tags)
}
```

| Local | Purpose |
|---|---|
| `location_map` | Accept both human-friendly (`"East US"`) and API (`"eastus"`) inputs; emit canonical display name. |
| `location_code_map` | Map locations to short codes (`eus`, `wus`, `neu`) used in resource names. |
| `location` | Canonical location string for the deployment. |
| `location_code` | Short code for the deployment. |
| `primary_name` | Shared naming base for every computed resource name. |
| `module_version` | Module version read from `VERSION` for the `deployed_using` tag. |
| `default_tags` | Mandatory tags applied to every resource. |
| `tags` | Final merged tag map — user tags plus defaults. |

## Location Maps

Two maps keyed identically — one returns the display name, one returns the short code. Both accept the human-friendly form *and* the API form so callers can pass whichever they have without surprise.

```hcl
location_map = {
  "East US"   = "East US"
  "West US"   = "West US"
  "North EU"  = "North Europe"
  "eastus"    = "East US"
  "westus"    = "West US"
  "northeu"   = "North Europe"
}
location_code_map = {
  "East US"   = "eus"
  "West US"   = "wus"
  "North EU"  = "neu"
  "eastus"    = "eus"
  "westus"    = "wus"
  "northeu"   = "neu"
}
```

Why two maps and not one nested map: direct map lookup (`local.map[var.location]`) gives a clean error if the caller passes an unsupported region, and the structure stays readable. Pair with a `validation` block on `var.location` so the failure is descriptive at plan time rather than as a missing-key error.

## The `primary_name` Pattern

`primary_name` is the shared naming base for every computed resource name in a module. Concatenating brand, environment, project, and location code produces a predictable prefix that encodes a resource's origin.

```hcl
primary_name = format("%s%s%s%s", var.brand, var.environment, var.project_name, local.location_code)
# Example: "acmeprodmyappeus"
```

### Deriving Resource Names

Derive every internal resource name from `primary_name`. Use `coalesce()` so callers can override when needed:

```hcl
locals {
  cluster_name         = coalesce(var.cluster.name, "${local.primary_name}-aro")
  cluster_domain       = coalesce(var.cluster.domain, "${local.cluster_name}.example.com")
  vnet_name            = coalesce(var.network.vnet_name, "${local.primary_name}-vnet")
  subnet_name          = coalesce(var.network.subnet_name, "${local.primary_name}-subnet")
  nsg_name             = coalesce(var.network.nsg_name, "${local.primary_name}-nsg")
  storage_account_name = coalesce(var.storage.name, "${local.primary_name}sa")
  secret_prefix        = coalesce(var.keyvault.secret_prefix, local.cluster_name)
}
```

Why this works:

- **Consistent naming.** Every resource follows the same brand+env+project+region scheme.
- **Caller override.** Explicit names are accepted when the convention does not fit (e.g., importing existing resources).
- **Traceability.** A resource name encodes where it came from.

## `module_version` From a `VERSION` File

Every module ships with a plain-text `VERSION` file at its root containing one line, e.g. `1.4.2`. Read it once into a local and use it in the `deployed_using` tag.

```hcl
module_version = trimspace(file("${path.module}/VERSION"))
```

`trimspace` is required — without it, a trailing newline ends up in your tag value. `path.module` ensures the path resolves correctly when the module is consumed as a child module.

The `deployed_using` tag uses hyphens between words in multi-word module names to match the repository name. Do **not** use dots:

```hcl
# Bad — dots between words
deployed_using = "Terraform - terraform.azurerm.network.security.group ${local.module_version}"

# Good — hyphens between words
deployed_using = "Terraform - terraform.azurerm.network-security-group ${local.module_version}"
```

## Tag Merging

Three rules:

1. Always merge user tags with default tags: `tags = merge(var.tags, local.default_tags)`.
2. Default tags appear **second** in `merge()` so they overwrite any conflicting user keys. This guarantees mandatory tags are always present and correct.
3. Every resource uses `local.tags` — never pass `var.tags` directly to a resource.

```hcl
resource "azurerm_resource_group" "this" {
  name     = var.resource_group_name
  location = local.location
  tags     = local.tags
}

resource "azurerm_virtual_network" "this" {
  name                = local.vnet_name
  location            = local.location
  resource_group_name = var.resource_group_name
  tags                = local.tags
}
```

If you find yourself wanting `merge(local.default_tags, var.tags)` (user tags winning), step back — you almost certainly want a separate variable for instance-specific tags rather than letting users override mandatory ones.

## Safe Defaults with `try()` and `lookup()`

Complex variable objects often have optional keys. Direct attribute access on a missing key fails at plan time. Use `try()` for nested attribute access and `lookup()` for map-key access.

| Scenario | Use |
|---|---|
| Map with optional keys | `lookup(map, "key", default)` |
| Nested object attribute that may not exist | `try(var.obj.nested.attr, default)` |
| Either-or fallback chain | `try(var.cluster.domain, local.custom_domain)` |
| Variable is `nullable = false` with `optional()` defaults | Direct access — Terraform already filled in defaults |

```hcl
# lookup() — flat map with optional keys
license_type            = lookup(each.value, "license_type", null)
threat_detection_policy = lookup(each.value, "threat_detection_policy", {})

# try() — nested attribute access
worker_vm_size     = try(var.cluster.worker_vm_size, "Standard_D4s_v3")
worker_count       = try(var.cluster.worker_count, 3)
api_visibility     = try(var.cluster.api_server_visibility, "Private")
ingress_visibility = try(var.cluster.ingress_visibility, "Private")
```

### Do Not Stack Defensive Wrappers

Pick one fallback mechanism per expression. Stacking `coalesce(try(...), ...)` is noise.

```hcl
# Bad — redundant wrapping
domain = coalesce(try(var.cluster.domain, null), local.custom_domain)

# Good — try() handles missing-attribute and the fallback in one call
domain = try(var.cluster.domain, local.custom_domain)
```

### Prefer `optional()` Defaults Over Locals Fallbacks

When a variable already uses `nullable = false` with `optional(type, default)`, Terraform fills in the default. Do not duplicate it in locals.

```hcl
# variables.tf
variable "cluster" {
  nullable = false
  type = object({
    fips_enabled = optional(bool, null)
    index        = optional(number, 1)
  })
}

# Bad — redundant; optional() already supplies the defaults
locals {
  fips_enabled  = try(var.cluster.fips_enabled, null)
  cluster_index = coalesce(try(var.cluster.index, null), 1)
}

# Good — access directly
resource "azurerm_redhat_openshift_cluster" "this" {
  fips_validated_modules = var.cluster.fips_enabled
}
```

This eliminates an entire class of unnecessary locals.

## Diagnostics Parsing

Modules that accept a `var.diagnostics` object (Log Analytics + Storage Account + Event Hub destinations) usually parse it into a flat structure with boolean enable flags.

```hcl
locals {
  parsed_diagnostics = {
    log_analytics = {
      id   = try(var.diagnostics.log_analytics.id, null)
      name = try(var.diagnostics.log_analytics.name, null)
    }
    storage_account = {
      id   = try(var.diagnostics.storage_account.id, null)
      name = try(var.diagnostics.storage_account.name, null)
    }
    event_hub = {
      id                    = try(var.diagnostics.event_hub.id, null)
      name                  = try(var.diagnostics.event_hub.name, null)
      authorization_rule_id = try(var.diagnostics.event_hub.authorization_rule_id, null)
    }
  }

  # Boolean flags for conditional resource creation
  enable_log_analytics = local.parsed_diagnostics.log_analytics.id != null
  enable_storage       = local.parsed_diagnostics.storage_account.id != null
  enable_event_hub     = local.parsed_diagnostics.event_hub.id != null

  logs_retention_days    = try(var.diagnostics.logs_retention, 365)
  metrics_retention_days = try(var.diagnostics.metrics_retention, 365)
}
```

Then the diagnostic resource reads from the parsed structure rather than poking into `var.diagnostics` repeatedly:

```hcl
resource "azurerm_monitor_diagnostic_setting" "this" {
  count = local.enable_log_analytics || local.enable_storage || local.enable_event_hub ? 1 : 0

  name               = "${local.primary_name}-diag"
  target_resource_id = azurerm_some_resource.this.id

  log_analytics_workspace_id     = local.parsed_diagnostics.log_analytics.id
  storage_account_id             = local.parsed_diagnostics.storage_account.id
  eventhub_name                  = local.parsed_diagnostics.event_hub.name
  eventhub_authorization_rule_id = local.parsed_diagnostics.event_hub.authorization_rule_id
}
```

## DRY Role Assignments with Maps

When a module creates multiple role assignments across the same scopes, build a map in locals and let `for` expressions and `concat` produce the final list. Beats N nearly-identical resource blocks.

```hcl
locals {
  resource_provider_role_scopes = {
    vnet               = var.network.vnet_id
    master_subnet      = var.network.master_subnet_id
    worker_subnet      = var.network.worker_subnet_id
    master_route_table = try(var.network.master_route_table_id, null)
    worker_route_table = try(var.network.worker_route_table_id, null)
  }

  authorization_role_assignments = concat(
    [
      for name, scope in local.resource_provider_role_scopes : {
        name            = "${local.cluster_name}-rp-${replace(name, "_", "-")}-network-contributor"
        scope           = scope
        definition_name = "Network Contributor"
        principal_id    = var.identity.rp_object_id
      } if scope != null
    ],
    [
      for name, scope in local.resource_provider_role_scopes : {
        name            = "${local.cluster_name}-cluster-sp-${replace(name, "_", "-")}-network-contributor"
        scope           = scope
        definition_name = "Network Contributor"
        principal_id    = var.identity.object_id
      } if scope != null
    ]
  )
}
```

`replace(name, "_", "-")` converts underscore-keyed map entries to hyphen-delimited Azure name fragments. The `if scope != null` filter quietly drops optional scopes the caller didn't supply.

## `for_each` Name Collision Prevention

When a resource uses `for_each` over a map, always incorporate `each.key` into the Azure resource name. A static name from `local.primary_name` alone causes every iteration to attempt the same name.

```hcl
# Bad — every database tries to claim the same name
resource "azurerm_mssql_managed_database" "this" {
  for_each = var.databases
  name     = "${local.primary_name}-db"
}

# Good — names are unique per iteration
resource "azurerm_mssql_managed_database" "this" {
  for_each = var.databases
  name     = format("%s-%s-db", local.primary_name, each.key)
}
```

## Static Diagnostic Categories Over Data Sources

Prefer a static list in locals to `data.azurerm_monitor_diagnostic_categories`. Static lists make plans deterministic, eliminate per-plan API calls, and avoid noisy diff output when Azure adds or removes categories.

```hcl
# Bad — data source queries the Azure API every plan
data "azurerm_monitor_diagnostic_categories" "this" {
  resource_id = azurerm_some_resource.this.id
}

# Good — static list, extensible via a variable if a caller needs more
locals {
  log_category_types = [
    "ApplicationGatewayAccessLog",
    "ApplicationGatewayFirewallLog",
    "ApplicationGatewayPerformanceLog",
  ]
  metrics = ["AllMetrics"]
}

resource "azurerm_monitor_diagnostic_setting" "this" {
  dynamic "enabled_log" {
    for_each = toset(local.log_category_types)
    content {
      category = enabled_log.key
    }
  }
}
```

Within a module, do not prefix these locals with the module name — the surrounding module already provides scope. `local.log_category_types` reads better than `local.app_gateway_log_category_types`.

## Example and Terratest Locals

Examples and Terratest plans both need their own `locals.tf`. Keep them realistic and clean.

### Example File Values

1. Use `"example-*"` names, never `"test-*"` — examples are documentation.
2. Put the resource type suffix at the end (`"example-01-ag"`, `"example-webhook"`).
3. Use variables for standard inputs (`var.resource_group_name`), not hardcoded strings.
4. Use `{placeholder}` syntax for non-constant ARM IDs (`/subscriptions/{subscriptionID}/...`).
5. Never include `name_override` in examples — that knob is for imports/migration only.
6. Annotate magic GUIDs and IDs with an inline comment explaining what they are.

```hcl
locals {
  action_groups = {
    example_01 = {
      short_name          = "example-01-ag"
      resource_group_name = var.resource_group_name
      role_id             = "acdd72a7-3385-48ef-bd42-f606fba81ae7" # Reader role definition ID
      subnet_id           = "/subscriptions/{subscriptionID}/resourceGroups/{resourceGroupName}/providers/Microsoft.Network/virtualNetworks/{vnetName}/subnets/{subnetName}"
    }
  }
}
```

### Terratest `locals.tf` vs `variables.tf`

In `terratest/plan/`, enforce a strict split:

- **`variables.tf`** — only standard pipeline variables (`azure_subscription_id`, `resource_group_name`, `location`). No module-specific values.
- **`locals.tf`** — all test-specific configuration, static values, and computed references.
- **No `main.auto.tfvars`** committed — use environment variables for pipeline-controlled switches; if you need a tfvars file for local testing, add it to `.gitignore`.

```hcl
# terratest/plan/variables.tf — only standard pipeline vars
variable "azure_subscription_id" {}
variable "resource_group_name" {}
variable "location" {}

# terratest/plan/locals.tf — all test configuration
locals {
  tags              = { deployed_for = "Terraform - Terratest build terraform.azurerm.aro ${trimspace(file("../../VERSION"))}" }
  resource_group_id = "/subscriptions/${var.azure_subscription_id}/resourceGroups/${var.resource_group_name}/"

  aro = {
    example_01 = {
      cluster = { /* ... */ }
      network = { /* ... */ }
    }
  }
}
```

## Common Mistakes

1. **Pass-through locals** — `local.resource_group_name = var.resource_group_name`. No transformation, no value; reference `var.x` directly.
2. **Single-use locals** — a local consumed in exactly one resource. Inline the expression instead; locals exist for reuse.
3. **Stacked defensive wrappers** — `coalesce(try(var.x.y, null), default)`. Pick one: `try(var.x.y, default)` is enough.
4. **Re-implementing `optional()` defaults in locals** — when the variable schema already declares `optional(type, default)`, `try()` in locals is redundant. Access the variable directly.
5. **Forgetting `trimspace()` on `file("${path.module}/VERSION")`** — the trailing newline leaks into the `deployed_using` tag.
6. **`merge(local.default_tags, var.tags)` (wrong order)** — lets users overwrite mandatory tags. Correct order is `merge(var.tags, local.default_tags)`.
7. **Using `var.tags` directly on a resource** — bypasses the merged defaults. Always pass `local.tags`.
8. **Static name in `for_each`** — `name = "${local.primary_name}-db"` collides on every iteration. Include `each.key`.
9. **Module-name prefix on internal locals** — inside the `app-gateway` module, `local.log_category_types` is clearer than `local.app_gateway_log_category_types`.
10. **Dots instead of hyphens in `deployed_using`** — `terraform.azurerm.network.security.group` should be `terraform.azurerm.network-security-group` to match the repo name.
11. **`data.azurerm_monitor_diagnostic_categories` for a static list** — adds an API call and noisy diffs. Use a `local` list.
12. **`name_override` in example `locals.tf`** — examples should demonstrate the default convention; `name_override` is for imports.

## Review Checklist

- [ ] Every local transforms, computes, or combines values — no pass-throughs.
- [ ] No single-use locals; inline-once expressions live at the call site.
- [ ] `location_map` and `location_code_map` accept both human and API forms of every supported region.
- [ ] `primary_name` is built with `format()` from brand + env + project + location code.
- [ ] Resource names are derived from `primary_name` and wrapped in `coalesce()` for caller overrides.
- [ ] `module_version = trimspace(file("${path.module}/VERSION"))` is present.
- [ ] `default_tags` includes `brand`, `costcenter`, `projectcode`, `environment`, `location`, `resource_group`, `deployed_using`.
- [ ] `tags = merge(var.tags, local.default_tags)` (defaults win).
- [ ] Every resource uses `local.tags`, not `var.tags` directly.
- [ ] `try()` is used for nested attribute access, `lookup()` for optional map keys; no stacked `coalesce(try(...), ...)`.
- [ ] No locals duplicate defaults already supplied by `optional()` in the variable schema.
- [ ] `for_each` resource names include `each.key` to avoid collisions.
- [ ] Diagnostic categories are a static list, not a `data` source.
- [ ] `deployed_using` uses hyphens between words, matching the repo name.
- [ ] Example `locals.tf` uses `example-*` names, variable references, and annotated GUIDs; no `name_override`.
- [ ] Terratest `variables.tf` holds only pipeline vars; everything else lives in `terratest/plan/locals.tf`.
