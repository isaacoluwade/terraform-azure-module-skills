---
name: terraform-dynamic-and-conditionals
description: Choose the right Terraform construct for conditional and iterative HCL — `dynamic` blocks, `count`, `for_each`, conditional resource creation, and plain inline blocks. Use this skill when deciding "should this be a `dynamic` block", "`count` vs `for_each`", "how do I make this resource optional", "how do I model an optional nested block", "iterate over a list of objects", or when reviewing HCL that uses any of these constructs. Encodes decision rules so you stop reaching for `dynamic` whenever a block *might* not be needed and stop using `count` over a list when `for_each` would give stable keys. Triggers on writing or reviewing modules with optional blocks, repeating nested blocks, identity blocks, diagnostic settings, NSG rules, subnet delegations, or any HCL that branches on a variable. Pick the simplest form that works.
---

# Terraform `dynamic`, `count`, and `for_each`

Choose the simplest construct that expresses the intent. A static inline block beats `dynamic`. `for_each` beats `count` for collections. `count = var.x ? 1 : 0` on the resource beats `dynamic` with 0-or-1 cardinality. Most "looks more flexible" rewrites are actively worse — they read poorly, produce noisier plan output, and create fragile state addresses.

For variable type design (including `optional()`), see `terraform-variables`. For naming the resulting resource addresses, see `terraform-resource-naming`. For outputs that read from `count`/`for_each` resources, see `terraform-module-outputs`.

## Contents
- [The Four Constructs](#the-four-constructs)
- [Decision Rules](#decision-rules)
- [Pattern Catalog](#pattern-catalog)
- [`for_each` Keying Rules](#for_each-keying-rules)
- [`count` Antipatterns](#count-antipatterns)
- [`dynamic` Antipatterns](#dynamic-antipatterns)
- [Conditional Resource Creation](#conditional-resource-creation)
- [Conditional Submodules](#conditional-submodules)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)

## The Four Constructs

Terraform has exactly four ways to make HCL conditional or iterative. Each has one job.

### 1. Static inline block — the default
Always present, same shape every apply. Don't reach for the others unless you have a reason.

```hcl
resource "azurerm_storage_account" "main" {
  name                     = var.name
  account_replication_type = "GRS"

  blob_properties {
    versioning_enabled = true
  }
}
```

### 2. `count = var.flag ? 1 : 0` — whole resource on/off
Cardinality is 0 or 1, controlled by a single boolean. Address is `azurerm_thing.main[0]`.

```hcl
resource "azurerm_monitor_diagnostic_setting" "main" {
  count              = var.enable_diagnostics ? 1 : 0
  name               = "${var.name}-diag"
  target_resource_id = azurerm_storage_account.main.id
}
```

### 3. `for_each` — multiple instances with stable keys
Cardinality is 0+, keyed by a string. Address is `azurerm_thing.main["key"]`.

```hcl
resource "azurerm_network_security_rule" "main" {
  for_each = { for r in var.rules : r.name => r }

  name                       = each.value.name
  priority                   = each.value.priority
  direction                  = each.value.direction
  access                     = each.value.access
  protocol                   = each.value.protocol
  source_address_prefix      = each.value.source_address_prefix
  destination_address_prefix = each.value.destination_address_prefix
}
```

### 4. `dynamic "block_name"` — generated nested blocks
Use only when a nested block inside a single resource repeats (0+) or is conditional (0 or 1) based on a variable.

```hcl
dynamic "delegation" {
  for_each = var.delegations
  content {
    name = delegation.value.name
    service_delegation {
      name    = delegation.value.service_name
      actions = delegation.value.actions
    }
  }
}
```

## Decision Rules

Apply these in order. Stop at the first one that fits.

1. **Is the block always present with a fixed shape?** Use a static inline block. Don't `dynamic` something you always emit once.
2. **Is the *whole resource* optional (0 or 1)?** Use `count = var.flag ? 1 : 0` on the resource.
3. **Are there 0+ instances of the resource keyed by name?** Use `for_each` over a map. Always prefer `for_each` over `count` for collections.
4. **Does a nested block repeat (0+) inside a single resource?** Use `dynamic` with `for_each = var.things`.
5. **Is a single nested block optional (0 or 1) inside a resource?** Use `dynamic` with `for_each = var.thing != null ? [var.thing] : []`. Or, if the block is always emitted but its *attributes* are optional, use `optional()` in the variable type and keep the block inline.

### Don't `dynamic` a block with a single optional attribute

```hcl
# Bad — dynamic ceremony for one optional attribute
variable "settings" {
  type = object({
    retention_days = number
  })
}

dynamic "blob_properties" {
  for_each = var.settings != null ? [var.settings] : []
  content {
    delete_retention_policy {
      days = blob_properties.value.retention_days
    }
  }
}

# Good — model optionality in the variable, keep the block inline
variable "settings" {
  type = object({
    retention_days = optional(number, 7)
  })
  default = {}
}

blob_properties {
  delete_retention_policy {
    days = var.settings.retention_days
  }
}
```

The `dynamic` form has more lines, more indirection, and produces longer plan diffs. The `optional()` form puts the conditional logic where it belongs — in the type — and keeps the resource readable.

## Pattern Catalog

### Network rules / NSG rules — `dynamic` over a `for_each` map
Multi-instance, keyed by name, so rules can be added or removed independently.

```hcl
# Bad — list-indexed; inserting a rule re-keys everything after it
dynamic "security_rule" {
  for_each = var.rules
  content {
    name     = var.rules[security_rule.key].name
    priority = var.rules[security_rule.key].priority
  }
}

# Good — for_each over a name-keyed map; rules are independent
dynamic "security_rule" {
  for_each = { for r in var.rules : r.name => r }
  content {
    name                  = security_rule.value.name
    priority              = security_rule.value.priority
    direction             = security_rule.value.direction
    access                = security_rule.value.access
    protocol              = security_rule.value.protocol
    source_address_prefix = security_rule.value.source_address_prefix
  }
}
```

### Identity block — conditional 0-or-1
System-assigned vs user-assigned identity is a presence-or-absent decision. Two sibling `dynamic` blocks read better than one with a complex `for_each`.

```hcl
# Good — one dynamic per identity mode, each gated independently
dynamic "identity" {
  for_each = var.identity != null ? [var.identity] : []
  content {
    type         = identity.value.type
    identity_ids = identity.value.type == "UserAssigned" ? identity.value.identity_ids : null
  }
}
```

### Diagnostic settings — `count` on the resource, not `dynamic` inside it
Diagnostics are typically all-or-nothing. Gate the whole resource.

```hcl
# Bad — gating the inner block leaves an empty resource around
resource "azurerm_monitor_diagnostic_setting" "main" {
  dynamic "enabled_log" {
    for_each = var.enable_diagnostics ? var.log_categories : []
    content { category = enabled_log.value }
  }
}

# Good — gate the resource itself
resource "azurerm_monitor_diagnostic_setting" "main" {
  count              = var.enable_diagnostics ? 1 : 0
  name               = "${var.name}-diag"
  target_resource_id = azurerm_storage_account.main.id

  dynamic "enabled_log" {
    for_each = toset(var.log_categories)
    content { category = enabled_log.value }
  }
}
```

### Storage account `static_website` — 0-or-1 with non-null trigger
The block is omitted when the variable is null and present once otherwise.

```hcl
# Good
dynamic "static_website" {
  for_each = var.static_website != null ? [var.static_website] : []
  content {
    index_document     = static_website.value.index_document
    error_404_document = static_website.value.error_404_document
  }
}
```

### Subnet delegation — 0+ keyed by delegation name

```hcl
# Good
dynamic "delegation" {
  for_each = { for d in var.delegations : d.name => d }
  content {
    name = delegation.key
    service_delegation {
      name    = delegation.value.service_name
      actions = delegation.value.actions
    }
  }
}
```

### Lifecycle / timeouts — never `dynamic`
`lifecycle` and `timeouts` are meta-blocks. Terraform requires them to be statically analyzable. Write them inline, always.

```hcl
# Good
lifecycle {
  prevent_destroy = true
  ignore_changes  = [tags]
}

timeouts {
  create = "30m"
  delete = "30m"
}
```

## `for_each` Keying Rules

`for_each` is fragile if you key it wrong. The keys become the resource address, so they must be stable and known at plan time.

### Keys must be stable across applies
Adding or removing items must not re-key everything else. Index-based keys violate this — inserting `var.rules[0]` shifts every later index by one.

```hcl
# Bad — count over a list
resource "azurerm_network_security_rule" "main" {
  count                       = length(var.rules)
  name                        = var.rules[count.index].name
  priority                    = var.rules[count.index].priority
  # ...
}

# Good — for_each keyed by stable name
resource "azurerm_network_security_rule" "main" {
  for_each = { for r in var.rules : r.name => r }
  name     = each.value.name
  priority = each.value.priority
  # ...
}
```

### Keys must be known at plan time
You cannot use values that are only known after apply (resource attributes that haven't been computed yet, `random_string` results, etc.) as `for_each` keys. Terraform will refuse the plan.

```hcl
# Bad — id is computed; not known at plan time on first apply
for_each = { for sa in azurerm_storage_account.main : sa.id => sa }

# Good — name is an input, known at plan time
for_each = { for sa in azurerm_storage_account.main : sa.name => sa }
```

### Convert lists to maps with a stable key field

```hcl
# Bad — toset on objects
for_each = toset(var.rules)  # error: toset only accepts strings/numbers

# Good — explicit map projection
for_each = { for r in var.rules : r.name => r }
```

If `r.name` could collide, choose a different field, concatenate fields, or compute a stable hash:

```hcl
for_each = { for r in var.rules : "${r.direction}-${r.name}" => r }
```

### `toset(...)` for sets of strings
Fine for collections of strings, but the key *is* the value — long strings produce ugly plan output (`["resource_address[\"a-very-long-category-name\"]"]`).

```hcl
for_each = toset(var.log_categories)  # OK for short string lists
```

## `count` Antipatterns

### `count = length(var.list)` — fragile
Insertion re-keys everything after the inserted item, forcing destroy+recreate of unrelated resources.

```hcl
# Bad
resource "azurerm_subnet" "main" {
  count          = length(var.subnets)
  name           = var.subnets[count.index].name
  address_prefix = var.subnets[count.index].cidr
  # ...
}

# Good
resource "azurerm_subnet" "main" {
  for_each       = { for s in var.subnets : s.name => s }
  name           = each.value.name
  address_prefix = each.value.cidr
  # ...
}
```

### `count = var.enabled ? 1 : 0` without guarding downstream references
When count is 0, `azurerm_thing.main[0]` doesn't exist. Outputs and downstream resources break unless guarded.

```hcl
# Bad — output fails at plan time when enabled = false
output "id" { value = azurerm_thing.main[0].id }

# Good
output "id" { value = try(azurerm_thing.main[0].id, null) }
```

See `terraform-module-outputs` for the full output-guard pattern.

### `count` on a module where the parent caller already knows the answer
Let the parent gate the call rather than baking conditional inclusion into the child.

```hcl
# Bad — child module conditionalizes itself with var.enabled everywhere

# Good — parent decides
module "thing" {
  count  = var.use_thing ? 1 : 0
  source = "./modules/thing"
}
```

The child stays single-purpose; the conditional sits at the call site where it's obvious.

## `dynamic` Antipatterns

### `dynamic` for a block that's always present

```hcl
# Bad
dynamic "blob_properties" {
  for_each = [var.blob_properties]
  content {
    versioning_enabled = blob_properties.value.versioning_enabled
  }
}

# Good
blob_properties {
  versioning_enabled = var.blob_properties.versioning_enabled
}
```

The `dynamic` adds zero conditional behavior. It's pure noise that doubles the line count and obscures the resource shape.

### `dynamic` with a hardcoded `for_each`
If the iteration list is a literal, it's not dynamic — it's decoration.

```hcl
# Bad
dynamic "log" {
  for_each = ["StorageRead", "StorageWrite", "StorageDelete"]
  content {
    category = log.value
  }
}

# Good — write the three blocks, or move the list to a variable
log { category = "StorageRead" }
log { category = "StorageWrite" }
log { category = "StorageDelete" }
```

### Nested `dynamic` more than 2 levels deep
Three levels of `dynamic` is almost always a variable-shape problem. Flatten the input rather than chaining `parent.value.x.value.y` references — keep at most one level of `dynamic` and inline the rest.

### `dynamic` with 0-or-1 cardinality when `count` on the resource would do
If the entire resource exists only when the block exists, gate the resource with `count`, not the block with `dynamic`.

## Conditional Resource Creation

### One feature flag, one resource
Use `count`. Done.

```hcl
resource "azurerm_private_endpoint" "main" {
  count = var.enable_private_endpoint ? 1 : 0
  name  = "${var.name}-pe"
  # ...
}
```

### Many feature flags, many resources
Keep them as separate `count`-gated resources. Don't try to merge unrelated optional resources into a single `for_each` keyed by feature name — the resource shapes differ and you'll fight the schema.

```hcl
# Good — one count per resource, easy to read and reason about
resource "azurerm_private_endpoint" "main" {
  count = var.enable_private_endpoint ? 1 : 0
  # ...
}

resource "azurerm_monitor_diagnostic_setting" "main" {
  count = var.enable_diagnostics ? 1 : 0
  # ...
}

resource "azurerm_advanced_threat_protection" "main" {
  count = var.enable_atp ? 1 : 0
  # ...
}
```

### Multiple instances behind one flag
Combine the flag with the collection.

```hcl
resource "azurerm_role_assignment" "main" {
  for_each             = var.enable_rbac ? { for a in var.role_assignments : a.principal_id => a } : {}
  scope                = var.scope
  role_definition_name = each.value.role
  principal_id         = each.key
}
```

## Conditional Submodules

Prefer the parent caller deciding to call the submodule:

```hcl
# Good — root module decides
module "diagnostics" {
  count  = var.enable_diagnostics ? 1 : 0
  source = "./modules/diagnostics"

  target_resource_id = module.storage.id
  workspace_id       = var.log_analytics_workspace_id
}
```

Then guard outputs that read from the module:

```hcl
output "diagnostics_id" {
  value = try(module.diagnostics[0].id, null)
}
```

Avoid the inverse — a child module with `var.enabled` that gates every resource inside it. That distributes the conditional logic across many files and creates a "ghost module" that's instantiated but does nothing.

## Common Mistakes

1. **Writing `dynamic` for a single block that's always emitted.** It's pure ceremony — produces longer plan output, more lines, no benefit. Use a static inline block.
2. **Using `count` over a list when `for_each` would give stable keys.** Inserting an item into the middle of the list re-keys everything after it and forces destroy+recreate of unrelated resources.
3. **`for_each` keys that aren't known at plan time.** Computed resource attributes (like `id`) can't be `for_each` keys on the first apply.
4. **Hardcoded keys in `for_each`** — `for_each = toset(["a", "b"])` is decoration, not iteration. Either move it to a variable or write the resources literally.
5. **Forgetting `try()` guards on outputs of `count`/`for_each` resources.** When the count is 0, `aws_thing.main[0].id` fails at plan time.
6. **Deeply nested `dynamic` blocks (>2 levels).** Almost always a variable-shape problem in disguise. Flatten the input before nesting `dynamic`.
7. **`dynamic` with 0-or-1 cardinality when `count` on the resource is the right answer.** If the whole resource is optional, gate the resource — not a block inside it.
8. **`dynamic` blocks for `lifecycle` or `timeouts`** — these are meta-blocks and must be static.
9. **Modeling a single optional attribute as a `dynamic` block** instead of using `optional()` in the variable type.
10. **Conditionalizing a child module from inside itself** with `var.enabled` everywhere, instead of letting the parent decide whether to instantiate it.
11. **`toset()` on a list of objects** — `toset` only accepts strings/numbers. Project to a map with `{ for x in ... : x.key => x }`.

## Review Checklist

- [ ] Every nested block that's always emitted is written as a static inline block, not `dynamic`.
- [ ] `count` is used only for whole-resource 0-or-1 toggles, never over a list.
- [ ] `for_each` is used for any 0+ collection, keyed by a stable string field.
- [ ] `for_each` keys are known at plan time (no computed/`apply`-time values).
- [ ] No hardcoded literal lists inside `for_each` — those are decoration.
- [ ] No `dynamic` block more than 2 levels deep — flatten the variable shape instead.
- [ ] Optional single attributes are modeled with `optional()` in the variable type, not wrapped in `dynamic`.
- [ ] Outputs that read from `count`/`for_each` resources are guarded with `try()`.
- [ ] `lifecycle` and `timeouts` blocks are written inline, never `dynamic`.
- [ ] Conditional submodules are gated at the parent (`module "x" { count = ... }`), not internally.
- [ ] When converting from `count` over a list to `for_each`, a state migration plan exists (`terraform state mv`) so existing resources aren't destroyed.
- [ ] Each "should this be `dynamic`?" answered by walking the decision rules — not by reflex.
