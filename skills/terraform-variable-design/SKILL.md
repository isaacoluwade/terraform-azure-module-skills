---
name: terraform-variable-design
description: Comprehensive `variables.tf` design for Terraform Azure modules. Use this skill when designing or reviewing module inputs — including typing, defaults (`null` vs `""` vs `{}`), `nullable = false`, `optional()` for object types, `validation` blocks, descriptions (heredoc for complex shapes), grouping by purpose (not alphabetical), complex object/map types, `name_override` patterns for import/migration, and sensitive variables. Triggers on: "design a variable", "validation block", "optional fields in an object", "default = null vs default = empty", "variables.tf for module", writing a new `variables.tf`, adding a `validation` block, deciding what to expose as a variable, or auditing variable design in a PR. This is the right skill any time `variables.tf` is being touched — variable design is the public API of the module and the largest topic in the module-development domain.
---

# Terraform Variable Design

Design `variables.tf` so every input is well-typed, well-described, validated at plan time, and prevents incorrect values. Variables are the public API of your module — every default, type narrowing, and validation is a contract with consumers.

For sibling concerns: `terraform-locals-patterns` (locals.tf), `terraform-module-outputs` (outputs.tf), `terraform-resource-naming` (the conventions applied to variable names).

## Contents
- [Design Principles](#design-principles)
- [Typing Rules](#typing-rules)
- [Defaults: `null` vs `""` vs `{}`](#defaults-null-vs--vs-)
- [`nullable = false` Semantics](#nullable--false-semantics)
- [Descriptions](#descriptions)
- [Validation Blocks (Quickstart)](#validation-blocks-quickstart)
- [Grouping by Purpose](#grouping-by-purpose)
- [Sensitive Variables](#sensitive-variables)
- [Variable Ordering](#variable-ordering)
- [`name_override` Pattern](#name_override-pattern)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Design Principles

Every variable should satisfy these rules:

1. **Concise description** — explain what the variable controls without restating what validation already enforces.
2. **Explicit type** — always declare `type`. Never let it default to `any` unless the shape is genuinely heterogeneous.
3. **Validation at plan time** — use `validation` blocks to reject bad input before any provider call.
4. **Validation over transformation** — reject bad input rather than silently fixing it with `lower()`, `replace()`, etc. Transformation hides user mistakes; the caller never learns the value was wrong.
5. **No redundancy** — don't repeat in the description what validation already enforces.
6. **Group by purpose** — combine related inputs into a single object variable.
7. **Sensible defaults** — define `default` for non-required variables. Use empty defaults (`{}`, `[]`) only when the underlying API treats empty as valid.
8. **Prevent incorrect values** — variables should be well-described, easy to understand, and impossible to misuse.

## Typing Rules

- Always declare `type` — `string`, `number`, `bool`, `list(...)`, `map(...)`, `set(...)`, or an `object({...})`.
- Prefer narrow types over `any`. `any` opts out of validation and shape inference; only use it for genuinely heterogeneous values (rare).
- For collections of structured data, use `list(object({...}))` or `map(object({...}))` rather than `list(any)`.
- Use `optional(T, default)` inside `object()` types to make attributes optional with a baked-in default. See `references/complex-types.md`.

```hcl
# Good — narrow type, validation possible
variable "subnet_ids" {
  description = "List of subnet resource IDs."
  type        = list(string)
  default     = []
  nullable    = false
}

# Bad — list(any) defeats the type system
variable "subnet_ids" {
  type    = list(any)
  default = []
}
```

## Defaults: `null` vs `""` vs `{}`

The choice of default is semantic, not stylistic. Pick the one that matches what the value *means*.

### Optional string — `default = null`, NOT `default = ""`

```hcl
# Good — null means "not provided"
variable "custom_domain" {
  description = "Custom domain name for the service."
  type        = string
  default     = null
}

# Bad — "" is a valid value, not the same as "not provided"
variable "custom_domain" {
  type    = string
  default = ""
}
```

**Why?** `null` and `""` are semantically different. Most Azure APIs treat `null` as "omit this parameter" and `""` as "set to empty string" — which often errors or produces drift. Use `null` to mean "not specified."

### Complex variable — `default = {}` with `nullable = false`, NOT `default = null`

```hcl
# Good
variable "network_config" {
  description = "Network configuration for the cluster."
  type = object({
    vnet_id   = optional(string)
    subnet_id = optional(string)
  })
  default  = {}
  nullable = false
}

# Bad — default = null forces null guards everywhere
variable "network_config" {
  type = object({
    vnet_id   = optional(string)
    subnet_id = optional(string)
  })
  default = null
}
```

**Why?** `nullable = false` prevents callers from passing `null`, which would force every reference to guard with `try()` or conditionals. `default = {}` gives a safe empty object that still satisfies the type constraint and lets `optional()` defaults apply.

### Quick reference

| Variable kind     | Default          | `nullable` |
|-------------------|------------------|------------|
| Complex object    | `{}`             | `false`    |
| Complex list      | `[]`             | `false`    |
| Map               | `{}`             | `false`    |
| Optional string   | `null`           | (omit)     |
| Optional number   | `null`           | (omit)     |
| Optional bool     | `null` or value  | (omit)     |
| Required variable | (no default)     | (omit)     |

## `nullable = false` Semantics

`nullable = false` says: the caller cannot pass `null`. If they omit the value, the `default` is used; if they pass `null`, Terraform errors at plan time.

Use it on:
- Every complex variable (object, map, list) with a `default` of `{}`/`[]`.
- Every `tags` map (`default = {}`).
- Every boolean toggle (`default = false` or `true`).

Do **not** use it on optional scalar strings/numbers where `null` is the legitimate "not set" sentinel.

```hcl
variable "tags" {
  description = "Tags applied to all resources."
  type        = map(string)
  default     = {}
  nullable    = false
}
```

> `nullable = false` requires Terraform `>= 1.3.0`. If you adopt it in an existing module, bump `required_version` and document it as a breaking change in `CHANGELOG.md`.

## Descriptions

Every variable has a `description`, ending with a period.

### Simple variables — one sentence

```hcl
variable "resource_group_name" {
  description = "Name of the Azure Resource Group where resources will be created."
  type        = string
}
```

### Complex variables — heredoc with type block and per-attribute docs

```hcl
variable "data_protection" {
  description = <<-EOT
    Data protection settings for the NetApp Volume.
    ```
    type = object({
      vault_id        = string
      snapshot_policy = optional(string)
      replication     = optional(object({
        endpoint_type    = string
        remote_volume_id = string
      }))
    })
    ```

    `vault_id` - (Required) Resource ID of the Backup Vault.
    `snapshot_policy` - (Optional) Resource ID of the snapshot policy.
    `replication` - (Optional) Cross-region replication settings.
      `endpoint_type` - (Required) `src` or `dst`.
      `remote_volume_id` - (Required) Resource ID of the remote volume.
  EOT

  type = object({
    vault_id        = string
    snapshot_policy = optional(string)
    replication     = optional(object({
      endpoint_type    = string
      remote_volume_id = string
    }))
  })
  default  = {}
  nullable = false
}
```

### Heredoc formatting rules

1. Start with a one-line summary.
2. Include a fenced code block showing the type shape.
3. Blank line between the type block and per-attribute descriptions.
4. Prefix each attribute with `` `attribute_name` - (Required|Optional) ``.
5. Use backticks for values, attribute names, and types.
6. Keep descriptions concise — never restate what validation already enforces.

## Validation Blocks (Quickstart)

Use `validation` blocks to enforce allowed values and syntax at plan time. Most variables get one.

```hcl
# Regex for string format
variable "storage_account_name" {
  description = "Name of the Storage Account. Must be 3-24 lowercase alphanumeric characters."
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9]{3,24}$", var.storage_account_name))
    error_message = "The `storage_account_name` must be 3-24 lowercase alphanumeric characters."
  }
}

# contains() for enum-like values
variable "sku" {
  description = "SKU tier. Allowed values: `Basic`, `Standard`, `Premium`."
  type        = string

  validation {
    condition     = contains(["Basic", "Standard", "Premium"], var.sku)
    error_message = "Allowed values for `sku` are: `Basic`, `Standard`, `Premium`."
  }
}

# Conditional validation — only validate when value is provided
variable "log_analytics_workspace_id" {
  description = "Resource ID of the Log Analytics Workspace."
  type        = string
  default     = null

  validation {
    condition     = var.log_analytics_workspace_id == null || can(regex("^/subscriptions/[a-f0-9-]+/resourceGroups/.+/providers/Microsoft\\.OperationalInsights/workspaces/.+$", var.log_analytics_workspace_id))
    error_message = "The `log_analytics_workspace_id` must be a valid Azure Log Analytics Workspace resource ID."
  }
}
```

Multiple validation blocks stack — Terraform evaluates them in order and stops at the first failure. For `alltrue()` over lists, cross-field validation, multi-condition validation (Terraform 1.9+), and full error-message conventions, see `references/validation-patterns.md`.

## Grouping by Purpose

If two or more variables share a logical relationship or a common naming prefix, group them into a single object variable.

### Good — grouped

```hcl
variable "identity" {
  description = <<-EOT
    Managed identity configuration for the resource.
    ```
    type = object({
      type         = string
      identity_ids = optional(list(string), [])
    })
    ```

    `type` - (Required) Allowed values: `SystemAssigned`, `UserAssigned`, `SystemAssigned, UserAssigned`.
    `identity_ids` - (Optional) User Assigned Identity resource IDs. Required when `type` includes `UserAssigned`.
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

  validation {
    condition     = var.identity.type == "SystemAssigned" || length(var.identity.identity_ids) > 0
    error_message = "The `identity_ids` must contain at least one ID when `identity.type` includes `UserAssigned`."
  }
}
```

### Bad — flat siblings

```hcl
variable "identity_type" { type = string }
variable "identity_ids"  { type = list(string)  default = [] }
```

For `optional()` defaults, map-of-object patterns, list-of-object patterns, and nested validation, see `references/complex-types.md`.

## Sensitive Variables

Mark inputs containing secrets with `sensitive = true`. Sensitive variables **must not** have a default — a default password is a security risk.

```hcl
variable "admin_password" {
  description = "Administrator password. Must be supplied by the caller; do not hardcode."
  type        = string
  sensitive   = true
}
```

Group related credentials into a single object:

```hcl
variable "service_principal" {
  description = <<-EOT
    Service principal credentials.
    ```
    type = object({
      object_id     = string
      client_id     = string
      client_secret = string
    })
    ```

    `object_id` - (Required) Object ID of the service principal.
    `client_id` - (Required) Application (client) ID.
    `client_secret` - (Required) Client secret.
  EOT

  type = object({
    object_id     = string
    client_id     = string
    client_secret = string
  })
  nullable  = false
  sensitive = true
}
```

**Best practices:**
- **Prefer Key Vault** — store secrets in Key Vault and reference them via `data "azurerm_key_vault_secret"` (or `ephemeral "azurerm_key_vault_secret"` for Terraform 1.10+) rather than passing them as variables.
- **Never set `sensitive = false` explicitly** — it's the default. Writing it adds noise (AVM TFNFR22).
- **No defaults on sensitive inputs** (AVM TFNFR23).

## Variable Ordering

Group variables by purpose, not alphabetically. Standard variables first (the org-wide set every module declares), then module-specific variables grouped logically by feature.

```hcl
# ──────────────────────────────────────────────
# Standard Variables
# ──────────────────────────────────────────────
variable "brand"               { ... }
variable "environment"         { ... }
variable "project_name"        { ... }
variable "costcenter"          { ... }
variable "projectcode"         { ... }
variable "location"            { ... }
variable "resource_group_name" { ... }
variable "tags"                { ... }

# ──────────────────────────────────────────────
# Module-Specific Variables
# ──────────────────────────────────────────────
variable "sku"            { ... }
variable "identity"       { ... }
variable "network_config" { ... }
variable "name_override"  { ... }
```

Grouping by purpose takes precedence over alphabetical ordering. Readers scanning the file see related concepts together; alphabetical ordering would split `network_subnet_id` from `network_private_endpoint_enabled`.

For the full set of standard variables every module declares, see `references/standard-variables.md`.

## `name_override` Pattern

Every module that creates a named Azure resource should include a `name_override` variable so consumers can import existing resources or migrate without being forced into the module's default naming.

```hcl
variable "name_override" {
  description = "This variable will override the default naming convention when required. Must be used for migration or importing existing resources."
  type        = string
  default     = null
}
```

```hcl
# Module main.tf — fallback to generated name when null
resource "azurerm_data_factory" "main" {
  name = var.name_override != null ? var.name_override : local.generated_name
  # ...
}
```

**Rules:**
1. The variable **must** be named `name_override` — not `custom_name`, `resource_name`.
2. Use the standard description shown above.
3. Default to `null` — when `null`, the module uses its generated name.
4. `name_override` must **NOT** appear in examples — examples should always demonstrate the default naming convention.

For the map-keyed `name_override` pattern (when a module creates several named resources), see `references/complex-types.md`.

## Common Mistakes

1. **`default = ""` for optional strings.** Empty string is a valid value, not the same as "not provided." Use `default = null`.
2. **`default = null` on complex variables.** Forces null-guards in every reference. Use `default = {}` (or `[]`) with `nullable = false`.
3. **Missing `nullable = false` on object/map/list variables with default `{}`/`[]`.** Lets callers pass `null` and break code that assumes the default applies.
4. **Transforming input instead of validating it** — `lower(var.environment)` silently fixes a typo. Reject bad input with a `validation` block.
5. **Description duplicates what validation enforces** — both the description and the error message restating the allowed values. Keep the description short; let the validation message carry the spec.
6. **`type = any` everywhere** — opts out of typing. Use a real `object({...})` or `list(object({...}))`.
7. **Wrapping `optional()` defaults in `locals`** — when `optional(bool, false)` is set, `try(var.x.enabled, false)` in locals just adds indirection. Reference `var.x.enabled` directly.
8. **Sensitive variable with a default value** — a default password is a security incident. Sensitive inputs must be required (AVM TFNFR23).
9. **Module-level kill switches** like `variable "enabled" { default = true }` — anti-pattern (AVM TFNFR14). Use resource-specific toggles like `create_route_table`.
10. **Flat related variables** instead of one object — `master_vm_size`, `master_subnet_id`, `master_encryption_at_host`. Group into `master_profile`.

## Review Checklist

- [ ] Every variable has an explicit `type` (no implicit `any`).
- [ ] Every variable has a `description` ending with a period.
- [ ] Optional strings/numbers default to `null`, never `""`.
- [ ] Object/map/list variables with empty defaults use `default = {}`/`[]` and `nullable = false`.
- [ ] Each `optional()` attribute provides a default where one is sensible.
- [ ] Standard variables (brand, environment, project_name, costcenter, projectcode, location, resource_group_name, tags) all have `validation` blocks.
- [ ] Related variables are grouped into object variables, not left as flat siblings.
- [ ] Variables are grouped by purpose (standard variables first), not alphabetical.
- [ ] Sensitive variables are marked `sensitive = true` and have **no** default.
- [ ] No `sensitive = false` declarations (it's the default).
- [ ] `name_override` exists for any module that creates a named resource and is **not** used in examples.
- [ ] Validation rejects bad input rather than transformation silently fixing it.
- [ ] `required_version >= 1.3.0` if `nullable = false` is used; >= 1.9.0 if multi-condition validation is used.
- [ ] Variable additions, removals, renames, and default changes are documented in `CHANGELOG.md`.
- [ ] No unused variables remain.

## References

- `references/standard-variables.md` — the org-wide standard variable set every module declares (brand, environment, project_name, costcenter, projectcode, location, resource_group_name, tags), AVM rules (deprecated variables, feature toggles, no module-level kill switches), example/Terratest variable conventions, and CHANGELOG requirements.
- `references/complex-types.md` — object types with `optional()`, map-of-object, list-of-object, nested validation, when to use which container, full `nullable = false` semantics, and `name_override` map patterns for multi-resource modules.
- `references/validation-patterns.md` — comprehensive validation patterns including regex, length checks, allowed-value lists, cross-field validation, multi-condition validation (Terraform 1.9+), and error message conventions.
