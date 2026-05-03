# Diagnostics, AzAPI, and Lifecycle Patterns

Patterns for routing logs and metrics to Azure Monitor destinations, working with the AzAPI provider when `azurerm` doesn't yet cover what you need, and using `lifecycle.ignore_changes` to silence drift caused by Azure Policy or external controllers.

## Contents
- [The `diagnostics` Variable](#the-diagnostics-variable)
- [Parsing the Destination Type](#parsing-the-destination-type)
- [The Diagnostic Setting Resource](#the-diagnostic-setting-resource)
- [Log Analytics Workspace Integration](#log-analytics-workspace-integration)
- [Retention Policies](#retention-policies)
- [Multiple Destinations](#multiple-destinations)
- [AzAPI Provider — When and How](#azapi-provider--when-and-how)
- [AzAPI for Diagnostic Categories Not Yet in azurerm](#azapi-for-diagnostic-categories-not-yet-in-azurerm)
- [AzAPI Best Practices](#azapi-best-practices)
- [Migrating Off AzAPI](#migrating-off-azapi)
- [Lifecycle `ignore_changes` for Tag Drift](#lifecycle-ignore_changes-for-tag-drift)
- [Lifecycle for Externally-Managed Fields](#lifecycle-for-externally-managed-fields)
- [Anti-patterns](#anti-patterns)

## The `diagnostics` Variable

Accept a single `diagnostics` object that describes where to send telemetry and what to send. Don't ask the caller to specify the destination type separately — derive it from the resource ID.

```hcl
variable "diagnostics" {
  description = <<-EOT
    Diagnostics configuration.
    `destination` - Resource ID of the diagnostics destination
                    (Log Analytics workspace, Storage Account, or Event Hub authorization rule).
    `metrics`     - List of metric categories to enable.
    `logs`        - List of log categories to enable.
  EOT
  type = object({
    destination = string
    metrics     = optional(list(string), [])
    logs        = optional(list(string), [])
  })
  default = null
}
```

`default = null` (rather than `{}`) means the caller must opt in. When `var.diagnostics == null`, no diagnostic setting is created.

## Parsing the Destination Type

The destination resource ID encodes its type — `Microsoft.OperationalInsights/workspaces`, `Microsoft.Storage/storageAccounts`, `Microsoft.EventHub/namespaces/.../authorizationRules`. Parse the type from the ID rather than asking the caller to set it:

```hcl
locals {
  diag_resource_list = var.diagnostics != null ? split("/", var.diagnostics.destination) : []

  parsed_diag = var.diagnostics != null ? {
    log_analytics_id   = contains(local.diag_resource_list, "Microsoft.OperationalInsights") ? var.diagnostics.destination : null
    storage_account_id = contains(local.diag_resource_list, "Microsoft.Storage") ? var.diagnostics.destination : null
    event_hub_auth_id  = contains(local.diag_resource_list, "Microsoft.EventHub") ? var.diagnostics.destination : null
    metric             = var.diagnostics.metrics
    log                = var.diagnostics.logs
  } : {
    log_analytics_id   = null
    storage_account_id = null
    event_hub_auth_id  = null
    metric             = []
    log                = []
  }
}
```

The parser yields `null` for the destination types that don't match — `azurerm_monitor_diagnostic_setting` accepts `null` for the destinations you aren't using.

This pattern keeps the variable surface minimal: callers pass one resource ID and a list of categories, no fiddly `destination_type = "log_analytics"` enum.

## The Diagnostic Setting Resource

```hcl
resource "azurerm_monitor_diagnostic_setting" "main" {
  count = var.diagnostics != null ? 1 : 0

  name                           = "${var.name}-diag"
  target_resource_id             = azurerm_redhat_openshift_cluster.main.id
  log_analytics_workspace_id     = local.parsed_diag.log_analytics_id
  storage_account_id             = local.parsed_diag.storage_account_id
  eventhub_authorization_rule_id = local.parsed_diag.event_hub_auth_id

  dynamic "metric" {
    for_each = local.parsed_diag.metric
    content {
      category = metric.value
    }
  }

  dynamic "enabled_log" {
    for_each = local.parsed_diag.log
    content {
      category = enabled_log.value
    }
  }
}
```

The resource is gated on `count = var.diagnostics != null ? 1 : 0` so the entire diagnostic setting goes away when the caller doesn't configure it. The `dynamic` blocks expand to one block per category in the input list.

## Log Analytics Workspace Integration

When the destination is a Log Analytics workspace, the workspace ID is the only field the module needs. The workspace itself is created and managed elsewhere — almost certainly by a central observability or platform team. The module never creates the workspace.

```hcl
# Caller invokes the module:
module "service" {
  source = "<your-terraform-registry>/azure/<service>/azurerm"

  diagnostics = {
    destination = data.terraform_remote_state.observability.outputs.log_analytics_workspace_id
    logs        = ["AuditLogs", "OperationLogs"]
    metrics     = ["AllMetrics"]
  }
}
```

The remote state lookup happens in the **caller's** root module, not in the leaf module being deployed. The module receives a resolved ID.

## Retention Policies

`azurerm_monitor_diagnostic_setting` historically supported per-category `retention_policy` blocks. Azure has deprecated these in favor of workspace-level retention configured on the destination itself (Log Analytics retention is a workspace setting; Storage Account retention is a storage account setting). The current best practice is:

- Don't set `retention_policy` on the diagnostic setting.
- Let retention be governed by the destination (workspace settings, storage lifecycle policy).
- The diagnostic setting just routes data; the destination controls how long it lives.

If you have older modules with `retention_policy` blocks, plan a migration — newer azurerm versions may stop accepting those blocks entirely.

## Multiple Destinations

A single diagnostic setting can route to multiple destinations simultaneously (Log Analytics + Storage + Event Hub). For modules that need to fan out telemetry, extend the variable to accept a list:

```hcl
variable "diagnostics" {
  description = "List of diagnostic settings."
  type = list(object({
    name        = string
    destination = string
    metrics     = optional(list(string), [])
    logs        = optional(list(string), [])
  }))
  default = []
}

resource "azurerm_monitor_diagnostic_setting" "main" {
  for_each = { for d in var.diagnostics : d.name => d }

  name                           = each.value.name
  target_resource_id             = azurerm_redhat_openshift_cluster.main.id
  log_analytics_workspace_id     = local.parsed[each.key].log_analytics_id
  storage_account_id             = local.parsed[each.key].storage_account_id
  eventhub_authorization_rule_id = local.parsed[each.key].event_hub_auth_id

  # ... metric / enabled_log dynamic blocks
}
```

Use `for_each` keyed by the diagnostic-setting name so each entry has a stable address.

## AzAPI Provider — When and How

`azurerm` lags Azure ARM by varying amounts depending on the resource. AzAPI is the escape hatch for the gaps. Use it when:

- The resource type isn't yet supported by `azurerm`.
- A property exists on the ARM API but `azurerm` hasn't surfaced it yet.
- You need to invoke an ARM action (e.g. `listAdminCredentials`, `regenerateKey`) that has no `azurerm` data source equivalent.
- A preview API version is required and `azurerm` only supports the GA version.

Don't use AzAPI when `azurerm` already supports the resource. The two providers maintain independent state representations of the same resource — having both manage the same thing is a recipe for drift and circular updates.

```hcl
# Bad — azurerm supports this resource fully
resource "azapi_resource" "aro_cluster" {
  type      = "Microsoft.RedHatOpenShift/openShiftClusters@2023-09-04"
  name      = var.name
  location  = var.location
  parent_id = var.resource_group.id
  # azurerm_redhat_openshift_cluster handles this natively
}
```

## AzAPI for Diagnostic Categories Not Yet in azurerm

A common AzAPI use case: a service has just released new diagnostic categories that `azurerm` hasn't picked up yet. You can use AzAPI to set the diagnostic setting via the ARM API directly:

```hcl
resource "azapi_resource" "diagnostic_setting" {
  count = var.diagnostics != null ? 1 : 0

  type      = "Microsoft.Insights/diagnosticSettings@2021-05-01-preview"
  name      = "${var.name}-diag"
  parent_id = azurerm_redhat_openshift_cluster.main.id

  body = {
    properties = {
      workspaceId = local.parsed_diag.log_analytics_id
      logs = [
        for category in var.diagnostics.logs : {
          category = category
          enabled  = true
        }
      ]
      metrics = [
        for category in var.diagnostics.metrics : {
          category = category
          enabled  = true
        }
      ]
    }
  }
}
```

Comment why AzAPI is required, and track when `azurerm` adds support so you can migrate.

## AzAPI Best Practices

1. **Pin the API version.** Use a concrete version like `@2023-09-04`, never `@latest`. A silent ARM API change shouldn't become a silent module change.
2. **Narrow `response_export_values`.** Export only the fields you need from the response. Exporting `["*"]` lands the entire response in state.
3. **Document why.** Add an inline comment explaining what gap AzAPI is filling and when the gap is expected to close. `# azurerm doesn't yet support property X — track issue Y`.
4. **Use `azapi_resource_action` for ARM actions.** When you need to call an action (`listAdminCredentials`, `getKeys`), this is the right resource — don't try to model an action as `azapi_resource`.
5. **Use `azapi_update_resource` for property patches** when `azurerm` manages the resource but doesn't surface a particular property — the update resource patches the missing field without taking over the whole resource.

```hcl
# Good — concrete version, narrow export, explanatory comment
data "azapi_resource_action" "admin_credentials" {
  count = var.retrieve_admin_credentials ? 1 : 0

  # azurerm has no equivalent for listAdminCredentials on ARO clusters as of 4.65.
  type                   = "Microsoft.RedHatOpenShift/openShiftClusters@2023-09-04"
  resource_id            = azurerm_redhat_openshift_cluster.main.id
  action                 = "listAdminCredentials"
  response_export_values = ["kubeadminPassword"]
}

locals {
  kubeadmin_password = try(
    data.azapi_resource_action.admin_credentials[0].output.kubeadminPassword,
    null
  )
}
```

## Migrating Off AzAPI

AzAPI is a stopgap. When `azurerm` adds support for the field or action, migrate.

The risk of leaving AzAPI behind:

- Two providers managing the same resource creates drift potential.
- Future maintainers can't tell whether the AzAPI block is still needed or is a fossil.
- `terraform plan` may show no-op churn as the two providers disagree on representation.

Migration path:

1. Confirm `azurerm` covers the property/action.
2. Move the configuration into the `azurerm_*` resource.
3. `terraform state rm` the AzAPI resource, or delete-and-import if state can't be removed cleanly.
4. Remove the AzAPI block and the AzAPI provider requirement if it's no longer needed elsewhere.

Don't leave a `# TODO: migrate to azurerm` comment in the module forever — track the work as an issue and remove the AzAPI block when it's done.

## Lifecycle `ignore_changes` for Tag Drift

The most common legitimate use of `ignore_changes` in Azure modules: silencing tag drift caused by Azure Policy.

Many enterprises run policies that inject or normalize tags after resource creation — `createdAt`, `createdBy`, `costCenter`, environment-specific normalization. Without `ignore_changes`, every `terraform plan` shows a tag diff that Terraform can never reconcile (because the policy re-applies the tags after every apply).

```hcl
resource "azurerm_redhat_openshift_cluster" "main" {
  # ...
  tags = local.tags

  lifecycle {
    ignore_changes = [
      tags["createdAt"],   # injected by Azure Policy
      tags["createdBy"],   # injected by Azure Policy
    ]
  }
}
```

Two flavors:

- **Per-key ignore** — `ignore_changes = [tags["createdAt"]]` ignores specific keys but lets you still manage other tags. Preferred when only some tags drift.
- **Whole-`tags` ignore** — `ignore_changes = [tags]` is acceptable when every tag is policy-managed. Avoid when you actually want Terraform to manage tags.

Document inline why each ignored key is ignored — future readers shouldn't have to guess whether the drift is from policy, from automation, or from a long-forgotten workaround.

## Lifecycle for Externally-Managed Fields

Beyond tags, certain fields are owned by external controllers after creation:

- DNS records may have a `dns_view` or similar field set by the IPAM controller post-creation.
- Some resources have status fields the API mutates (provisioning state, last-modified timestamps) that azurerm sometimes incorrectly diffs.
- Encryption key version on customer-managed-key fields rotates outside Terraform.

```hcl
resource "infoblox_a_record" "api_server" {
  count = var.infoblox.enabled ? 1 : 0

  fqdn     = "api.${var.cluster_name}.${var.infoblox.dns_zone}"
  ip_addr  = azurerm_redhat_openshift_cluster.main.api_server_profile[0].ip
  dns_view = var.infoblox.dns_view

  lifecycle {
    ignore_changes = [
      dns_view,  # IPAM controller may switch view post-create
    ]
  }
}
```

```hcl
resource "azurerm_storage_account_customer_managed_key" "main" {
  storage_account_id = azurerm_storage_account.main.id
  key_vault_id       = var.keyvault.resource_id
  key_name           = var.cmk_key_name

  lifecycle {
    ignore_changes = [
      key_version,  # rotated externally by Key Vault
    ]
  }
}
```

## Anti-patterns

```hcl
# Bad — ignore_changes = all disables drift detection entirely
lifecycle {
  ignore_changes = all
}
```

This silences every diff, including ones you actually want to know about. Almost always wrong — list specific fields instead.

```hcl
# Bad — ignoring an input that should be a variable
resource "azurerm_storage_account" "main" {
  account_replication_type = "LRS"

  lifecycle {
    ignore_changes = [
      account_replication_type,  # If callers want to control this, expose it as a variable!
    ]
  }
}
```

If the caller has a legitimate reason to change a field, expose it as a variable rather than hard-coding a value and ignoring drift.

```hcl
# Bad — undocumented ignore_changes
lifecycle {
  ignore_changes = [
    tags,
    network_rules,
    something_else,
  ]
}
```

No comments, no reason given. Any future maintainer has to reverse-engineer why these are ignored. Always annotate inline why each field is ignored — Azure Policy, external controller, manual rotation, vendor bug, whatever the cause is.

```hcl
# Good — documented per-field ignores
lifecycle {
  ignore_changes = [
    tags["createdAt"],     # Azure Policy injection
    tags["createdBy"],     # Azure Policy injection
    network_rules[0].ip_rules,  # IP allowlist managed by central firewall team
  ]
}
```
