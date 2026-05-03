# Complex Variable Types

Detailed treatment of object types with `optional()`, map-of-object, list-of-object, nested validation, container-type selection, full `nullable = false` semantics, and `name_override` map patterns. The main `SKILL.md` covers the high-frequency rules; this file covers the patterns you reach for when designing real-world `variables.tf`.

## Contents
- [Choosing a Container Type](#choosing-a-container-type)
- [Object Types with `optional()`](#object-types-with-optional)
- [Map-of-Object Patterns](#map-of-object-patterns)
- [List-of-Object Patterns](#list-of-object-patterns)
- [Nested Objects and Nested Validation](#nested-objects-and-nested-validation)
- [`nullable = false` — Full Semantics](#nullable--false--full-semantics)
- [`name_override` Map Patterns](#name_override-map-patterns)
- [Anti-Patterns: Wrapping `optional()` in Locals](#anti-patterns-wrapping-optional-in-locals)

## Choosing a Container Type

| Shape                                  | Use                          | Why                                                              |
|----------------------------------------|------------------------------|------------------------------------------------------------------|
| One settings bag                       | `object({...})`              | Single configuration record; attributes are heterogeneous.       |
| Several keyed instances                | `map(object({...}))`         | Caller picks meaningful keys; `for_each` over the map.           |
| Ordered, possibly duplicate            | `list(object({...}))`        | Order matters or duplicates allowed (e.g. ordered rules).        |
| Set of unique scalar values            | `set(string)` / `set(number)`| Membership semantics; no order; useful with `for_each`.          |
| Free-form key-value                    | `map(string)`                | Tags, labels, simple lookup tables.                              |

Prefer `map(object(...))` over `list(object(...))` when the caller needs stable identity (e.g. you'll use `for_each = var.thing` in the module). Lists key by index, which is fragile across reorderings.

## Object Types with `optional()`

The `optional()` modifier lets callers omit attributes from an object. Pair it with a default value to provide sensible fallbacks:

```hcl
variable "master_profile" {
  description = <<-EOT
    Configuration for the master (control-plane) nodes.
    ```
    type = object({
      vm_size             = optional(string, "Standard_D8s_v3")
      subnet_id           = string
      encryption_at_host  = optional(bool, true)
      disk_encryption_set = optional(string)
    })
    ```

    `vm_size` - (Optional) VM size for master nodes. Defaults to `Standard_D8s_v3`.
    `subnet_id` - (Required) Resource ID of the master subnet.
    `encryption_at_host` - (Optional) Enable host-level encryption. Defaults to `true`.
    `disk_encryption_set` - (Optional) Resource ID of the Disk Encryption Set.
  EOT

  type = object({
    vm_size             = optional(string, "Standard_D8s_v3")
    subnet_id           = string
    encryption_at_host  = optional(bool, true)
    disk_encryption_set = optional(string)
  })
  nullable = false
}
```

### Caller specifies only what they care about

```hcl
master_profile = {
  subnet_id = azurerm_subnet.master.id
}
# vm_size defaults to "Standard_D8s_v3", encryption_at_host defaults to true,
# disk_encryption_set defaults to null
```

### `optional()` with no default vs with a default

```hcl
type = object({
  # Caller may omit. If omitted, the attribute is null.
  disk_encryption_set = optional(string)

  # Caller may omit. If omitted, the attribute is "Standard_D8s_v3".
  vm_size = optional(string, "Standard_D8s_v3")
})
```

When you have a sensible default, **always** provide it in `optional()` — don't leave it `null` and recompute the default in `locals`. See [Anti-Patterns: Wrapping `optional()` in Locals](#anti-patterns-wrapping-optional-in-locals).

### `optional()` requires Terraform 1.3+

Modules using `optional()` for attributes inside object types must declare `required_version >= 1.3.0`. Adopting `optional()` in a previously 1.0+ module is a breaking change for the consumer's Terraform CLI requirement.

### Don't re-specify `null` for optional attributes

```hcl
# Bad — explicit = null is redundant; null is the default for optional()
master_profile = {
  subnet_id           = azurerm_subnet.master.id
  vm_size             = "Standard_D16s_v3"
  disk_encryption_set = null
}

# Good — omit the attribute entirely
master_profile = {
  subnet_id = azurerm_subnet.master.id
  vm_size   = "Standard_D16s_v3"
}
```

## Map-of-Object Patterns

When a module creates several keyed instances of the same resource, use `map(object({...}))` and `for_each` over it. The caller chooses the keys; the module preserves identity across plans.

```hcl
variable "subnets" {
  description = <<-EOT
    Subnets to create in the virtual network, keyed by subnet name.
    ```
    type = map(object({
      address_prefixes  = list(string)
      service_endpoints = optional(list(string), [])
      delegation        = optional(object({
        name         = string
        service_name = string
      }))
    }))
    ```

    `address_prefixes` - (Required) CIDR blocks for the subnet.
    `service_endpoints` - (Optional) List of service endpoints to enable.
    `delegation` - (Optional) Subnet delegation block.
      `name` - (Required) Name of the delegation.
      `service_name` - (Required) Name of the delegated service.
  EOT

  type = map(object({
    address_prefixes  = list(string)
    service_endpoints = optional(list(string), [])
    delegation        = optional(object({
      name         = string
      service_name = string
    }))
  }))
  default  = {}
  nullable = false

  validation {
    condition = alltrue([
      for v in var.subnets : alltrue([for c in v.address_prefixes : can(cidrhost(c, 0))])
    ])
    error_message = "Every CIDR in `subnets[*].address_prefixes` must be a valid CIDR block."
  }
}
```

```hcl
# Module main.tf — for_each on the map
resource "azurerm_subnet" "main" {
  for_each = var.subnets

  name                 = each.key
  address_prefixes     = each.value.address_prefixes
  service_endpoints    = each.value.service_endpoints
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name

  dynamic "delegation" {
    for_each = each.value.delegation == null ? [] : [each.value.delegation]
    content {
      name = delegation.value.name
      service_delegation {
        name = delegation.value.service_name
      }
    }
  }
}
```

### Why map over list

- `for_each` on a map gives stable instance addresses (`azurerm_subnet.main["frontend"]`).
- Reordering the caller's input doesn't move resources around.
- Deleting a key destroys exactly that one instance — adding/removing a list element shifts every subsequent index.

## List-of-Object Patterns

Use `list(object({...}))` when order matters (e.g. ordered rules) or when duplicates are valid:

```hcl
variable "firewall_rules" {
  description = <<-EOT
    Ordered firewall rules. Earlier rules take precedence.
    ```
    type = list(object({
      name             = string
      priority         = number
      action           = string
      source_addresses = list(string)
      destination_port = string
    }))
    ```

    `name` - (Required) Rule name.
    `priority` - (Required) Lower numbers evaluated first.
    `action` - (Required) Allowed values: `Allow`, `Deny`.
    `source_addresses` - (Required) Source CIDR ranges.
    `destination_port` - (Required) Destination port or range.
  EOT

  type = list(object({
    name             = string
    priority         = number
    action           = string
    source_addresses = list(string)
    destination_port = string
  }))
  default  = []
  nullable = false

  validation {
    condition = alltrue([
      for r in var.firewall_rules : contains(["Allow", "Deny"], r.action)
    ])
    error_message = "Each `firewall_rules[*].action` must be `Allow` or `Deny`."
  }

  validation {
    condition = alltrue([
      for r in var.firewall_rules : r.priority >= 100 && r.priority <= 4096
    ])
    error_message = "Each `firewall_rules[*].priority` must be between 100 and 4096."
  }
}
```

### Iterating a list with `for_each` requires re-keying

If you need stable identity, project the list into a map keyed by `name`:

```hcl
locals {
  firewall_rules_map = { for r in var.firewall_rules : r.name => r }
}

resource "azurerm_firewall_network_rule_collection" "main" {
  for_each = local.firewall_rules_map
  # ...
}
```

This pattern is documented in detail under `terraform-locals-patterns`.

## Nested Objects and Nested Validation

Nested objects let you express compound configuration shapes without inventing dozens of flat variables.

```hcl
variable "data_protection" {
  description = <<-EOT
    Data protection settings for the volume.
    ```
    type = object({
      backup = optional(object({
        vault_id        = string
        snapshot_policy = optional(string)
      }))
      replication = optional(object({
        endpoint_type    = string
        remote_volume_id = string
      }))
    })
    ```

    `backup` - (Optional) Backup configuration.
      `vault_id` - (Required) Resource ID of the Backup Vault.
      `snapshot_policy` - (Optional) Resource ID of the snapshot policy.
    `replication` - (Optional) Cross-region replication settings.
      `endpoint_type` - (Required) `src` or `dst`.
      `remote_volume_id` - (Required) Resource ID of the remote volume.
  EOT

  type = object({
    backup = optional(object({
      vault_id        = string
      snapshot_policy = optional(string)
    }))
    replication = optional(object({
      endpoint_type    = string
      remote_volume_id = string
    }))
  })
  default  = {}
  nullable = false

  validation {
    condition     = var.data_protection.replication == null || contains(["src", "dst"], var.data_protection.replication.endpoint_type)
    error_message = "When `data_protection.replication` is set, `endpoint_type` must be `src` or `dst`."
  }
}
```

### Nested validation tips

- Guard against `null` parents in your condition: `var.x.y == null || <check on var.x.y.z>`.
- Stack multiple `validation` blocks rather than chaining many conditions in one with `&&`.
- For comprehensive validation patterns (alltrue, regex, allowed-value lists, cross-field, multi-condition) see `references/validation-patterns.md`.

## `nullable = false` — Full Semantics

`nullable` controls whether the **whole variable** can be `null`.

| Setting          | Caller passes `null`     | Caller omits the value |
|------------------|---------------------------|-------------------------|
| (default)        | Accepted; value is `null` | Default applies         |
| `nullable = false` | Plan-time error          | Default applies         |

`nullable = false` does **not** affect `optional()` attributes inside an object — those can still be `null` if the caller didn't provide them and there's no `optional()` default. To prevent attribute-level `null`, use a `validation` block.

```hcl
variable "network" {
  type = object({
    vnet_id   = optional(string)
    subnet_id = optional(string)
  })
  default  = {}
  nullable = false  # var.network itself cannot be null

  validation {
    # But the caller can still pass { vnet_id = null }; this validation rejects that.
    condition     = var.network.vnet_id == null || can(regex("^/subscriptions/", var.network.vnet_id))
    error_message = "`network.vnet_id`, when provided, must be an Azure resource ID."
  }
}
```

### Adoption is a breaking change

Adding `nullable = false` to an existing variable is a breaking change for any caller that was passing `null`. Document it in `CHANGELOG.md` and bump the major version:

```markdown
### Breaking Changes
- Add `nullable = false` to `network_config`. Callers passing `null` must now omit the variable or pass `{}`.
- Bump `required_version` to `>= 1.3.0` (required by `nullable = false`).
```

## `name_override` Map Patterns

The simple `name_override` is a single string default `null`. When a module creates several named resources from one map variable, the override needs to be a map keyed the same way:

```hcl
variable "subnets" {
  description = "Subnets to create, keyed by logical name."
  type = map(object({
    address_prefixes = list(string)
  }))
  default  = {}
  nullable = false
}

variable "name_override" {
  description = "This variable will override the default naming convention when required. Map is keyed by the same logical name as `subnets`. Must be used for migration or importing existing resources."
  type        = map(string)
  default     = {}
  nullable    = false
}
```

```hcl
# Module main.tf
locals {
  subnet_names = {
    for k, v in var.subnets :
    k => lookup(var.name_override, k, "${var.project_name}-${var.environment}-${k}-snet")
  }
}

resource "azurerm_subnet" "main" {
  for_each = var.subnets
  name     = local.subnet_names[each.key]
  # ...
}
```

**Rules for `name_override` (any form):**
1. Always named `name_override`. Never `custom_name`, `resource_name`, `override_name`.
2. Use the standard description (adapted for map shape if needed).
3. Default to `null` for scalar form, `{}` with `nullable = false` for map form.
4. Do **not** use `name_override` in examples — examples should demonstrate the default naming.

## Anti-Patterns: Wrapping `optional()` in Locals

When a variable already declares `nullable = false` and uses `optional()` with defaults, wrapping the values in `locals` is pure indirection.

### Bad — locals re-specify the same defaults

```hcl
variable "network" {
  type = object({
    pod_cidr      = optional(string, "10.128.0.0/14")
    service_cidr  = optional(string, "172.30.0.0/16")
    outbound_type = optional(string, "Loadbalancer")
  })
  default  = {}
  nullable = false
}

locals {
  pod_cidr      = try(var.network.pod_cidr, "10.128.0.0/14")
  service_cidr  = try(var.network.service_cidr, "172.30.0.0/16")
  outbound_type = try(var.network.outbound_type, "Loadbalancer")
}

resource "example" "main" {
  pod_cidr      = local.pod_cidr
  service_cidr  = local.service_cidr
  outbound_type = local.outbound_type
}
```

### Good — reference `var` directly

```hcl
variable "network" {
  type = object({
    pod_cidr      = optional(string, "10.128.0.0/14")
    service_cidr  = optional(string, "172.30.0.0/16")
    outbound_type = optional(string, "Loadbalancer")
  })
  default  = {}
  nullable = false
}

resource "example" "main" {
  pod_cidr      = var.network.pod_cidr
  service_cidr  = var.network.service_cidr
  outbound_type = var.network.outbound_type
}
```

### Bad — defining the default in locals instead of `optional()`

```hcl
variable "keyvault" {
  type = object({
    enabled = optional(bool)  # No default specified
  })
  default  = {}
  nullable = false
}

locals {
  keyvault_enabled = try(var.keyvault.enabled, false)
}
```

### Good — set the default in `optional()`

```hcl
variable "keyvault" {
  type = object({
    enabled = optional(bool, false)
  })
  default  = {}
  nullable = false
}

# Reference var.keyvault.enabled directly — no locals needed.
```

### Don't double-wrap `try()`

```hcl
# Bad — try() inside try() is expensive and unreadable.
locals {
  cluster_index = try(try(var.cluster.index, null), 0)
}

# Good — single try() with the fallback.
locals {
  cluster_index = try(var.cluster.index, 0)
}
```

### Don't `coalesce(try(...), fallback)`

```hcl
# Bad
locals {
  workspace_id = coalesce(try(var.monitoring.workspace_id, null), var.default_workspace_id)
}

# Good
locals {
  workspace_id = try(var.monitoring.workspace_id, var.default_workspace_id)
}
```

`try()` already returns the fallback when the lookup fails — wrapping it in `coalesce` only adds a third case for `""` that you usually don't want.
