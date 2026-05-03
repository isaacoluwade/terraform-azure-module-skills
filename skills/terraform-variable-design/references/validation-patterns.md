# Validation Patterns

Comprehensive validation patterns for `variables.tf`. Validation runs at plan time, before any provider call — it's the cheapest place to reject bad input. The principle: **prefer validation over transformation**. Reject `Dev` rather than silently `lower()`-ing it; the caller learns about the typo immediately.

## Contents
- [Anatomy of a Validation Block](#anatomy-of-a-validation-block)
- [Regex Validation](#regex-validation)
- [Length Checks](#length-checks)
- [Allowed-Value Lists](#allowed-value-lists)
- [Numeric Range Checks](#numeric-range-checks)
- [List and Map Validation with `alltrue()`](#list-and-map-validation-with-alltrue)
- [Conditional Validation (Only Validate When Provided)](#conditional-validation-only-validate-when-provided)
- [Cross-Field Validation](#cross-field-validation)
- [Multi-Condition Validation (Terraform 1.9+)](#multi-condition-validation-terraform-19)
- [Stacking Multiple Validation Blocks](#stacking-multiple-validation-blocks)
- [Error Message Conventions](#error-message-conventions)
- [Validation vs Transformation](#validation-vs-transformation)

## Anatomy of a Validation Block

```hcl
variable "sku" {
  description = "SKU tier. Allowed values: `Basic`, `Standard`, `Premium`."
  type        = string

  validation {
    condition     = contains(["Basic", "Standard", "Premium"], var.sku)
    error_message = "Allowed values for `sku` are: `Basic`, `Standard`, `Premium`."
  }
}
```

- `condition` — a boolean expression. Must reference `var.<this_variable>` (Terraform allows referencing other variables only on Terraform 1.9+ in cross-field validation).
- `error_message` — a string shown when `condition` is false. End with a period. Use backticks around values and identifiers.

## Regex Validation

Use `can(regex(...))` to enforce string format. `can()` swallows the error and returns `false` on no-match.

```hcl
variable "storage_account_name" {
  description = "Name of the Storage Account. Must be 3-24 lowercase alphanumeric characters."
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9]{3,24}$", var.storage_account_name))
    error_message = "The `storage_account_name` must be 3-24 lowercase alphanumeric characters."
  }
}
```

### Common Azure regex patterns

```hcl
# Brand abbreviation (alphanumeric, lowercase, 2-10 chars)
condition = can(regex("^[a-z0-9]{2,10}$", var.brand))

# Project name (alphanumeric, lowercase, 2-10 chars)
condition = can(regex("^[a-z0-9]{2,10}$", var.project_name))

# Cost center / project code (XXXX-XXXX with digits)
condition = can(regex("^[0-9]{4}-[0-9]{4}$", var.costcenter))

# Azure resource ID (subscription-scoped Log Analytics workspace)
condition = can(regex("^/subscriptions/[a-f0-9-]+/resourceGroups/.+/providers/Microsoft\\.OperationalInsights/workspaces/.+$", var.log_analytics_workspace_id))

# Domain name
condition = can(regex("^[a-z0-9][a-z0-9.-]+\\.[a-z]{2,}$", var.custom_domain))

# AKS-style cluster name (1-63 chars, alphanumeric + hyphen, must start/end alphanumeric)
condition = can(regex("^[a-z0-9][a-z0-9-]{0,61}[a-z0-9]$", var.cluster_name))

# UUID
condition = can(regex("^[a-fA-F0-9]{8}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{12}$", var.tenant_id))

# CIDR (use cidrhost() for actual CIDR validation; regex matches shape only)
condition = can(cidrhost(var.address_prefix, 0))
```

### Anchor your regex

Always anchor with `^...$`. An unanchored regex `[a-z0-9]+` matches anything containing at least one lowercase alphanumeric character, including `My_Bad_Name`. Anchors enforce the rule against the whole string.

### Escape the period

In a regex, `.` matches any character. To match a literal dot use `\\.` in the HCL string (one `\` for the regex, escaped as `\\` for the HCL literal).

## Length Checks

```hcl
variable "resource_group_name" {
  description = "Name of the Azure Resource Group where resources will be created."
  type        = string

  validation {
    condition     = length(var.resource_group_name) > 0
    error_message = "The `resource_group_name` must not be empty."
  }
}

variable "cluster_name" {
  description = "AKS cluster name (1-63 characters)."
  type        = string

  validation {
    condition     = length(var.cluster_name) >= 1 && length(var.cluster_name) <= 63
    error_message = "The `cluster_name` must be between 1 and 63 characters."
  }
}
```

For collections:

```hcl
variable "subnet_ids" {
  description = "List of subnet IDs (at least one required)."
  type        = list(string)

  validation {
    condition     = length(var.subnet_ids) > 0
    error_message = "At least one entry in `subnet_ids` is required."
  }
}
```

Length checks compose well with regex — typically you stack them as separate `validation` blocks so error messages stay focused.

## Allowed-Value Lists

Use `contains(allowed, value)` for enum-like values. The error message should list the allowed values verbatim:

```hcl
variable "environment" {
  description = "The deployment environment. Allowed values: `dev`, `tst`, `stg`, `uat`, `prd`, `qa`."
  type        = string

  validation {
    condition     = contains(["dev", "tst", "stg", "uat", "prd", "qa"], var.environment)
    error_message = "Allowed values for `environment` are: `dev`, `tst`, `stg`, `uat`, `prd`, `qa`."
  }
}

variable "location" {
  description = "Azure region. Allowed values: `eastus`, `eastus2`, `westus2`."
  type        = string

  validation {
    condition     = contains(["eastus", "eastus2", "westus2"], var.location)
    error_message = "Allowed values for `location` are: `eastus`, `eastus2`, `westus2`."
  }
}
```

### Don't accept `case-insensitive` matches by lowercasing

```hcl
# Bad — silently accepts `Dev`, `DEV`, etc. The caller never learns the canonical form.
condition = contains(["dev", "tst", "prd"], lower(var.environment))

# Good — reject. The caller fixes their input and uses the canonical form everywhere.
condition = contains(["dev", "tst", "prd"], var.environment)
```

## Numeric Range Checks

```hcl
variable "instance_count" {
  description = "Number of instances. Must be between 1 and 100."
  type        = number

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 100
    error_message = "The `instance_count` must be between 1 and 100."
  }
}

variable "ttl" {
  description = "DNS record TTL in seconds. Allowed range: 30-86400."
  type        = number
  default     = 300

  validation {
    condition     = var.ttl >= 30 && var.ttl <= 86400
    error_message = "The `ttl` must be between 30 and 86400 seconds."
  }
}
```

## List and Map Validation with `alltrue()`

For list/map variables, validate each element using `alltrue()` over a `for` expression:

```hcl
variable "subnets" {
  description = <<-EOT
    List of subnet configurations.
    ```
    type = list(object({
      name           = string
      address_prefix = string
    }))
    ```

    `name` - (Required) Name of the subnet.
    `address_prefix` - (Required) CIDR block for the subnet.
  EOT

  type = list(object({
    name           = string
    address_prefix = string
  }))
  default  = []
  nullable = false

  validation {
    condition = alltrue([
      for s in var.subnets : can(regex("^[a-zA-Z][a-zA-Z0-9-]{0,78}[a-zA-Z0-9]$", s.name))
    ])
    error_message = "Each subnet `name` must be 1-80 characters, start with a letter, and contain only alphanumeric characters or hyphens."
  }

  validation {
    condition = alltrue([
      for s in var.subnets : can(cidrhost(s.address_prefix, 0))
    ])
    error_message = "Each `address_prefix` must be a valid CIDR block (e.g., `10.0.1.0/24`)."
  }
}
```

### `alltrue()` over a map

```hcl
validation {
  condition = alltrue([
    for v in var.subnets : alltrue([for c in v.address_prefixes : can(cidrhost(c, 0))])
  ])
  error_message = "Every CIDR in `subnets[*].address_prefixes` must be a valid CIDR block."
}
```

### `anytrue()` — at least one match

```hcl
variable "identity" {
  type = object({
    type         = string
    identity_ids = optional(list(string), [])
  })

  validation {
    condition     = anytrue([for t in ["SystemAssigned", "UserAssigned"] : strcontains(var.identity.type, t)])
    error_message = "The `identity.type` must include `SystemAssigned` or `UserAssigned`."
  }
}
```

## Conditional Validation (Only Validate When Provided)

For optional inputs, validate only when the value is non-null. This pattern is essential for defaults of `null`:

```hcl
variable "log_analytics_workspace_id" {
  description = "Resource ID of the Log Analytics Workspace. Required when `enable_monitoring` is `true`."
  type        = string
  default     = null

  validation {
    condition     = var.log_analytics_workspace_id == null || can(regex("^/subscriptions/[a-f0-9-]+/resourceGroups/.+/providers/Microsoft\\.OperationalInsights/workspaces/.+$", var.log_analytics_workspace_id))
    error_message = "The `log_analytics_workspace_id`, when provided, must be a valid Azure Log Analytics Workspace resource ID."
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
```

The pattern `var.x == null || <check>` short-circuits on `null` — when the caller doesn't provide the value, validation is bypassed. When they do provide one, it must satisfy the check.

## Cross-Field Validation

Sometimes a constraint spans two attributes of the same object variable. Stack a second `validation` block:

```hcl
variable "identity" {
  description = "Managed identity configuration."
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
    # If type includes UserAssigned, identity_ids must be non-empty.
    condition     = var.identity.type == "SystemAssigned" || length(var.identity.identity_ids) > 0
    error_message = "The `identity_ids` must contain at least one ID when `identity.type` includes `UserAssigned`."
  }
}
```

### Cross-variable validation (Terraform 1.9+)

Before Terraform 1.9, validation conditions could only reference `var.<this_variable>`. Terraform 1.9+ relaxes this — you may reference other input variables, locals, and other module objects. Document this in `versions.tf`:

```hcl
terraform {
  required_version = ">= 1.9.0"
}
```

```hcl
variable "log_analytics_workspace_id" {
  type    = string
  default = null

  validation {
    # Reference another variable: workspace_id is required when monitoring is enabled.
    condition     = !var.enable_monitoring || var.log_analytics_workspace_id != null
    error_message = "The `log_analytics_workspace_id` is required when `enable_monitoring` is `true`."
  }
}
```

## Multi-Condition Validation (Terraform 1.9+)

Terraform 1.9+ supports `condition` with multi-condition error messages — but in practice the cleanest expression is still **stacking validation blocks** (one per logical rule). Keep each block focused on one error message so the caller knows exactly what to fix.

```hcl
variable "cluster_name" {
  description = "Name of the AKS cluster."
  type        = string

  # Block 1 — length check
  validation {
    condition     = length(var.cluster_name) >= 1 && length(var.cluster_name) <= 63
    error_message = "The `cluster_name` must be between 1 and 63 characters."
  }

  # Block 2 — character set
  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]*[a-z0-9]$", var.cluster_name))
    error_message = "The `cluster_name` must start and end with a lowercase alphanumeric character and may contain hyphens."
  }
}
```

Terraform evaluates blocks in order and stops at the first failure, so the caller fixes one issue at a time. This is preferable to one mega-condition that says "name is invalid" without telling them why.

## Stacking Multiple Validation Blocks

You can attach as many `validation` blocks as you need to a single variable. Each gets its own error message.

```hcl
variable "address_space" {
  description = "VNet address space CIDRs."
  type        = list(string)

  validation {
    condition     = length(var.address_space) >= 1
    error_message = "At least one CIDR is required in `address_space`."
  }

  validation {
    condition     = alltrue([for c in var.address_space : can(cidrhost(c, 0))])
    error_message = "Every entry in `address_space` must be a valid CIDR block."
  }

  validation {
    condition     = alltrue([for c in var.address_space : tonumber(split("/", c)[1]) <= 24])
    error_message = "Every CIDR in `address_space` must have a prefix of /24 or larger (smaller number)."
  }
}
```

Order matters — stricter format checks should come after basic existence checks, so the caller doesn't see a confusing regex error when their list is empty.

## Error Message Conventions

- **End with a period.** Match the descriptive style of the rest of the file.
- **Backticks around variable names, attribute names, and values.** ``The `sku` must be `Basic`, `Standard`, or `Premium`.``
- **State the rule, not the failure.** "Allowed values for `sku` are: ..." not "Invalid sku."
- **Include examples** for format constraints. ``Must follow the format `XXXX-XXXX` (e.g., `1234-5678`).``
- **Use the variable name verbatim.** Don't shorten `log_analytics_workspace_id` to "workspace ID."
- **Describe the constraint precisely.** "must be 3-24 lowercase alphanumeric characters" beats "must be valid."
- **For object attributes, use dot-paths.** ``The `identity.type` must be...`` not "the identity type."

### Good

```hcl
error_message = "The `costcenter` must follow the format `XXXX-XXXX` (e.g., `1234-5678`)."
error_message = "Allowed values for `environment` are: `dev`, `tst`, `stg`, `uat`, `prd`, `qa`."
error_message = "Each `firewall_rules[*].priority` must be between 100 and 4096."
```

### Bad

```hcl
error_message = "Invalid costcenter"           # No period, no backticks, no detail.
error_message = "environment is wrong."        # Doesn't say what's accepted.
error_message = "Bad rule priority detected"   # Vague, no period, no detail.
```

## Validation vs Transformation

The single most important rule: **reject bad input rather than silently fixing it**. Transformation hides user mistakes and produces unpredictable behavior.

### Bad — transforming input

```hcl
# Module silently lowercases. The caller passing `Dev` never learns the
# canonical form is `dev`, and may be confused by diffs in tags, names, etc.
variable "environment" {
  description = "Deployment environment."
  type        = string
}

locals {
  environment = lower(var.environment)
}
```

### Good — validation rejects invalid input

```hcl
variable "environment" {
  description = "The deployment environment. Allowed values: `dev`, `tst`, `stg`, `uat`, `prd`, `qa`."
  type        = string

  validation {
    condition     = contains(["dev", "tst", "stg", "uat", "prd", "qa"], var.environment)
    error_message = "Allowed values for `environment` are: `dev`, `tst`, `stg`, `uat`, `prd`, `qa`."
  }
}
```

### When transformation is OK

Transformation is acceptable when it's **deriving** a value (computing a name, joining tags) — not when it's coercing user input into the form you wish they'd provided. The hint: if the transformation could change the meaning of what the caller asked for (`Dev` → `dev`), reject it instead.

```hcl
# Fine — deriving a name from validated inputs.
locals {
  resource_name = "${var.brand}-${var.environment}-${var.project_name}-rg"
}
```
