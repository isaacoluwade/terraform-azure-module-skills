---
name: terraform-module-outputs
description: Design clean, consistent `outputs.tf` files for Terraform modules. Use this skill when writing or reviewing module outputs — including naming, descriptions, attribute ordering, sensitivity flags, conditional/guarded outputs, map outputs for `for_each` resources, deprecated outputs, and CHANGELOG documentation for output changes. Triggers on: writing or modifying `outputs.tf`, adding outputs to a Terraform module, reviewing a PR that changes module outputs, deciding what to expose to consumers, designing outputs for Terratest assertions, or auditing existing outputs against AVM (Azure Verified Modules) rules. This is the right skill even for small output additions — output design directly affects every consumer of the module.
---

# Terraform Module Outputs

Design `outputs.tf` so every output is discoverable, well-described, and doesn't leak inputs or expose unnecessary internal state. Outputs are the public API of your module — once consumers depend on them, removing or renaming is a breaking change.

For overall module structure, see `terraform-module-structure`. For naming conventions on the output identifiers themselves, see `terraform-resource-naming`.

## Contents
- [File Conventions](#file-conventions)
- [Naming Rules](#naming-rules)
- [Attribute Ordering](#attribute-ordering)
- [Description Format](#description-format)
- [Sensitive Outputs](#sensitive-outputs)
- [Optional and Conditional Outputs](#optional-and-conditional-outputs)
- [Map Outputs for Iterated Resources](#map-outputs-for-iterated-resources)
- [Anti-Corruption Layer (AVM TFFR2)](#anti-corruption-layer-avm-tffr2)
- [Deprecated Outputs](#deprecated-outputs)
- [CHANGELOG for Output Changes](#changelog-for-output-changes)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)

## File Conventions

- All outputs go in `outputs.tf` — **plural**, not `output.tf`. This applies to the root module, submodules, examples, and test plan directories.
- Each output has a meaningful name and a description ending with a period.
- No comments or commented-out code in `outputs.tf` — keep it clean.
- End the file with a single empty line.

## Naming Rules

- Use `snake_case`.
- Names should be self-describing.
- **Do not prefix output names with the module name.** The module is already namespaced by its caller (e.g. `module.aro.cluster_id`). Prefixing creates `module.aro.aro_cluster_id`, which reads awkwardly and bakes the module name into every consumer.

```hcl
# Bad — redundant module-name prefix
output "aro_cluster_id" { ... }
output "aro_console_url" { ... }

# Good — clean, unprefixed names
output "cluster_id" {
  description = "The ID of the ARO cluster."
  value       = azurerm_redhat_openshift_cluster.main.id
}

output "console_url" {
  description = "The OpenShift console URL."
  value       = azurerm_redhat_openshift_cluster.main.console_url
}
```

## Attribute Ordering

Within an output block, attributes appear in this order: `description`, `value`, `sensitive`.

```hcl
output "function_app_id" {
  description = "The ID of the Function App."
  value       = local.is_linux ? azurerm_linux_function_app.main[0].id : azurerm_windows_function_app.main[0].id
}

output "function_app_key" {
  description = "The default host key of the Function App."
  value       = local.is_linux ? azurerm_linux_function_app.main[0].default_hostname : azurerm_windows_function_app.main[0].default_hostname
  sensitive   = true
}
```

The order matters for human readability — readers see *what* the output is before *how* it's computed, and any sensitivity warning is the last thing they see, where it stands out.

## Description Format

### Simple outputs — one-line description ending with a period

```hcl
output "account_id" {
  description = "The ID of the Azure NetApp Account."
  value       = azurerm_netapp_account.main.id
}
```

### Complex outputs — use heredoc to describe the shape

```hcl
output "circuit" {
  description = <<-EOT
    A `circuit` block exports the following:
      `express_route_id` - The ID of the ExpressRoute Circuit.
      `express_route_private_peering_id` - The ID of the ExpressRoute Circuit private peering.
      `primary_subnet_cidr` - The CIDR of the primary subnet.
      `secondary_subnet_cidr` - The CIDR of the secondary subnet.
  EOT
  value       = azurerm_vmware_private_cloud.main.circuit
}
```

## Sensitive Outputs

Mark outputs containing secrets with `sensitive = true`:

```hcl
output "storage_account_primary_access_key" {
  description = "The primary access key for the storage account."
  value       = azurerm_storage_account.main.primary_access_key
  sensitive   = true
}
```

**Preferred:** store the secret in Key Vault and output only the secret reference. Outputting raw secrets — even marked sensitive — puts them in state and in any logs that capture the output map.

## Optional and Conditional Outputs

Use `try()` for outputs that may not exist:

```hcl
output "api_server_a_record" {
  description = "The DNS A record for the API server."
  value       = try(infoblox_a_record.api_server[0].fqdn, null)
}
```

Don't add a redundant conditional check around `try()`:

```hcl
# Bad — redundant outer conditional
value = var.dns_enabled ? try(infoblox_a_record.api_server[0].fqdn, null) : null

# Good — try() alone handles the missing-resource case
value = try(infoblox_a_record.api_server[0].fqdn, null)
```

### Guard Outputs for Conditionally-Created Resources

When a resource is created conditionally (`count`, `for_each`), unguarded outputs **fail at plan time** if the resource doesn't exist:

```hcl
# Bad — fails when the waf_policy module isn't created
output "waf_policy_id" {
  description = "The ID of the WAF policy."
  value       = module.waf_policy[0].waf_policy_id
}

# OK — guarded with length check
output "waf_policy_id" {
  description = "The ID of the WAF policy."
  value       = length(module.waf_policy) > 0 ? module.waf_policy[0].waf_policy_id : null
}

# Better — try() is cleaner and Terraform evaluates it more efficiently
output "waf_policy_id" {
  description = "The ID of the WAF policy."
  value       = try(module.waf_policy[0].waf_policy_id, null)
}
```

Prefer `try()` over `length()` checks — it's more concise and short-circuits at evaluation time.

## Map Outputs for Iterated Resources

When using `for_each`, return a map keyed by the iteration key:

```hcl
output "storage_account_ids" {
  description = "The IDs of the Storage Accounts."
  value       = { for k, v in module.storage_accounts : k => v.storage_account_id }
}
```

### Merge Default and Dynamic Map Outputs

When a resource has both a default instance and dynamic instances created via `for_each`, combine them with `merge()`:

```hcl
output "secondary_connection_strings" {
  description = "The secondary connection strings for the Service Bus namespace."
  value = merge(
    { "default" = azurerm_servicebus_namespace.main.default_secondary_connection_string },
    { for k, v in azurerm_servicebus_namespace_authorization_rule.main : k => v.secondary_connection_string }
  )
  sensitive = true
}
```

## Anti-Corruption Layer (AVM TFFR2)

**Do not output entire resource objects.** They may contain sensitive data, and the schema can change between provider versions — silently breaking consumers.

```hcl
# Bad — entire resource object
output "storage_account" {
  description = "The storage account resource."
  value       = azurerm_storage_account.main
}

# Good — discrete computed attributes the caller actually needs
output "storage_account_id" {
  description = "The ID of the storage account."
  value       = azurerm_storage_account.main.id
}

output "storage_account_primary_endpoint" {
  description = "The primary blob endpoint of the storage account."
  value       = azurerm_storage_account.main.primary_blob_endpoint
}
```

**Additional rules:**
- **Do not re-output inputs** (except `name`). The caller already has them.
- For resources deployed with `for_each`, output computed attributes in a map structure.

### Use bracket notation for nested list access

```hcl
# Bad — dot notation for list index
value = azurerm_api_management.main.identity.0.principal_id

# Good — bracket notation
value = azurerm_api_management.main.identity[0].principal_id
```

### Skip outputs with no downstream value

If a resource exposes only a single trivial attribute (just an `id`) and that ID has no use to consumers, omit the output. Outputs should provide value to the caller.

### Add outputs for sub-resources

When a module creates sub-resources (replicas, pools, slots), expose them too — don't only surface the primary resource:

```hcl
output "server_id" {
  description = "The ID of the MySQL Flexible Server."
  value       = azurerm_mysql_flexible_server.main.id
}

output "replica_ids" {
  description = "The IDs of the MySQL Flexible Server replicas."
  value       = { for k, v in azurerm_mysql_flexible_server.replica : k => v.id }
}
```

### Separate Outputs by Resource Scope

When a module manages resources at different scopes (e.g. subscription-level vs resource-group-level), use separate outputs with scope-descriptive names rather than a single combined output:

```hcl
# Bad — one output that can't represent every scope
output "policy_exemption_id" {
  description = "The ID of the policy exemption."
  value       = azurerm_consumption_budget.main.id
}

# Good — one output per scope
output "subscription_policy_exemption_id" {
  description = "The ID of the subscription-level policy exemption."
  value       = try(azurerm_subscription_policy_exemption.main[0].id, null)
}

output "rg_policy_exemption_id" {
  description = "The ID of the resource-group-level policy exemption."
  value       = try(azurerm_resource_group_policy_exemption.main[0].id, null)
}
```

## Outputs for Testing

Expose every output Terratest (or your test harness) needs to assert against. Tests should consume output values, never hardcoded expectations — this is what proves the module produces what it claims to produce.

When iterating modules with `for_each`, the map-output pattern above is what makes tests scale.

## Deprecated Outputs (AVM TFNFR30)

When an output is deprecated:
1. Move it to `deprecated_outputs.tf`
2. Define the replacement output in `outputs.tf`
3. Remove the deprecated output during the next major version release

```hcl
# deprecated_outputs.tf
output "aro_cluster_id" {
  description = "DEPRECATED: Use `cluster_id` instead. The ID of the ARO cluster."
  value       = azurerm_redhat_openshift_cluster.main.id
}
```

## CHANGELOG for Output Changes

Every output addition, removal, or rename must be documented in `CHANGELOG.md`. Use `Added`, `Removed`, or `Changed`, and call out breaking changes:

```markdown
### Added
- Add output `function_app_slot_id`.

### Removed
- **Breaking:** remove output `primary_connection_key`.
- **Breaking:** remove output `secondary_connection_key`.

### Changed
- Rename output `vmss_id` to `id`.
```

## Common Mistakes

1. **File named `output.tf` (singular).** Always plural.
2. **Module-name prefix on output names** — `aro_cluster_id` instead of `cluster_id`.
3. **Missing description, or description without a trailing period.**
4. **Wrong attribute order** — `value` before `description`, or `sensitive` before `value`.
5. **Outputting entire resource objects** instead of discrete computed attributes.
6. **Re-outputting inputs** the caller already has.
7. **Unguarded outputs for conditional resources** — fail at plan time when the count is 0.
8. **Using `length()` for conditional guards** when `try()` is shorter and faster.
9. **Dot notation `.0` for nested list access** instead of bracket notation `[0]`.
10. **Outputting trivial single-attribute resources** with no downstream use.
11. **Not documenting output changes in CHANGELOG.md** — silent breaking changes.
12. **Combining outputs across different resource scopes** into one ambiguous output.
13. **Exposing secrets as outputs** instead of storing them in Key Vault.

## Review Checklist

- [ ] File is named `outputs.tf` (plural).
- [ ] Every output has a `description` ending with a period.
- [ ] Attribute order is `description`, `value`, `sensitive`.
- [ ] No output prefixes with the module name.
- [ ] No outputs that simply re-emit input variables (except `name`).
- [ ] No outputs returning entire resource objects.
- [ ] Conditional/`count`/`for_each` outputs are guarded with `try()`.
- [ ] `for_each` outputs return maps keyed by the iteration key.
- [ ] Outputs needed for tests are exposed.
- [ ] Sub-resource outputs (replicas, pools, slots) are present where they exist.
- [ ] Sensitive outputs are marked `sensitive = true` (or stored in Key Vault).
- [ ] Bracket notation `[0]` is used for nested list access.
- [ ] Deprecated outputs are in `deprecated_outputs.tf` with a clear deprecation notice.
- [ ] Every output addition, removal, or rename is reflected in `CHANGELOG.md`.
- [ ] No comments or commented-out code in `outputs.tf`.
