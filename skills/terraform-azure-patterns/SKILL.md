---
name: terraform-azure-patterns
description: Cross-cutting Azure-specific patterns for Terraform modules — the no-data-source principle (modules accept ARM IDs as inputs and never look them up), Key Vault integration patterns (read vs write vs delegate), centralized RBAC delegation through a dedicated authorization module, managed identity and service principal handling, private endpoints with public access disabled by default, diagnostic settings routed to a caller-supplied destination, AzAPI provider for resources not yet covered by azurerm, and lifecycle `ignore_changes` for tag drift caused by Azure Policy. Use this skill when designing or reviewing any Azure module pattern for Terraform — Key Vault in Terraform module, RBAC role assignment in module, private endpoint, managed identity, diagnostic setting, AzAPI fallback, or whenever a module is tempted to reach for a `data` source. This is the heart of the Azure-specific guidance and applies to every Azure resource your module wraps.
---

# Terraform Azure Patterns

Cross-cutting patterns that apply to every Azure Terraform module. The most important rule is **modules accept resolved values as inputs, they don't look them up at runtime.** This single principle drives most of the other patterns here — Key Vault integration, RBAC delegation, identity handling, and networking all assume the caller has already resolved the IDs and is passing them in.

For variable shape, see `terraform-variable-design`. For module structure and file layout, see `terraform-module-structure`. For outputs, see `terraform-module-outputs`.

## Contents
- [No Data Sources (Critical)](#no-data-sources-critical)
- [Key Vault Integration](#key-vault-integration)
- [RBAC and Authorization Delegation](#rbac-and-authorization-delegation)
- [Service Principals and Managed Identity](#service-principals-and-managed-identity)
- [Networking and Private Endpoints](#networking-and-private-endpoints)
- [Diagnostic Settings](#diagnostic-settings)
- [AzAPI Provider](#azapi-provider)
- [Lifecycle and `ignore_changes`](#lifecycle-and-ignore_changes)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## No Data Sources (Critical)

**Avoid `data` sources inside modules.** Accept resolved values — names, IDs, locations, principal IDs — as inputs. The caller already knows them; re-fetching them inside the module creates more problems than it solves.

### Why it matters

- **Extra API calls on every plan/apply.** Data sources don't compare against state — they query Azure each time. In a config with many module instances this slows plans dramatically.
- **State file pollution.** Every data source result lands in state. Sensitive values (Key Vault contents, SPN metadata) end up serialized to a backend that may not have the same protections as Key Vault itself.
- **Plan-time uncertainty.** If a data source depends on a resource managed elsewhere in the same plan, Terraform may not know the value at plan time. The plan shows `(known after apply)` for downstream attributes, hiding what would otherwise be a clean diff.
- **Hidden dependencies.** A data source ties the module to whatever happens to exist in the target subscription. Two consumers run the same module against different subscriptions and get different behavior — the module isn't really portable any more.
- **Non-reproducible deployments.** A successful run today depends on the live state of resources outside the module's scope. If somebody renames the looked-up resource, your module breaks even though no input changed.
- **Testing friction.** Tests need real Azure infrastructure to exist before they can run. Pure-fixture or offline tests become impossible.
- **Consumer surprise.** Consumers can't tell from the module's variable surface what implicit lookups it will perform. They debug failures in resources they didn't think the module touched.

### The pattern

Accept resolved values. Don't look them up.

```hcl
# Good — caller passes the resource group properties
variable "resource_group" {
  description = "Resource group where resources will be created."
  type = object({
    name     = string
    location = string
    id       = string
  })
  nullable = false
}

# Good — caller passes pre-resolved subnet IDs
variable "network" {
  description = "Network configuration with pre-resolved resource IDs."
  type = object({
    worker_subnet_id = string
    master_subnet_id = string
  })
  nullable = false
}
```

```hcl
# Bad — hidden API call, hidden dependency, state pollution
data "azurerm_resource_group" "main" {
  name = var.resource_group_name
}

data "azurerm_subnet" "worker" {
  name                 = var.worker_subnet_name
  virtual_network_name = var.vnet_name
  resource_group_name  = var.vnet_rg_name
}

# Bad — looking up an SPN by client_id to get its object_id
data "azuread_service_principal" "cluster" {
  client_id = var.client_id
}
```

### When data sources are acceptable

A small set of cases legitimately need data sources:

1. The value is genuinely global to the runtime — `azurerm_client_config` for the current tenant ID, for example.
2. Reading a Key Vault secret (or, on Terraform 1.10+, an `ephemeral` resource) that the caller cannot pre-resolve.
3. There is no input-based alternative because the value isn't knowable until apply time.

Even in those cases, prefer accepting the value as input where the caller can reasonably provide it.

## Key Vault Integration

Storing secrets in Key Vault is preferred over emitting them as outputs. Modules either accept an existing Key Vault by ID, write metadata to it, read secrets from it, or do all three — but they don't manage the Key Vault itself unless that is the module's primary purpose.

```hcl
variable "keyvault" {
  description = <<-EOT
    Azure Key Vault integration configuration.
    `enabled`                - Whether Key Vault integration is enabled.
    `resource_id`            - Key Vault resource ID.
    `pull_secret_name`       - Secret name used to read the pull secret.
    `secret_prefix`          - Prefix applied to written secret names.
    `write_cluster_metadata` - Whether cluster metadata secrets are written.
  EOT
  type = object({
    enabled                = optional(bool, false)
    resource_id            = optional(string)
    pull_secret_name       = optional(string)
    secret_prefix          = optional(string)
    write_cluster_metadata = optional(bool, true)
  })
  default  = {}
  nullable = false
}
```

Three rules to internalize:

- **Don't output secrets.** Write them to Key Vault and output the secret reference. Even `sensitive = true` outputs land in state and CI logs.
- **Accept secret names, not values.** When the module needs credentials, take Key Vault secret names as inputs and resolve at apply time. Plaintext credential variables leak through `.tfvars`, CI logs, and state.
- **Prefer `ephemeral` on Terraform 1.10+.** Ephemeral resources never persist to state — strictly better than a `data "azurerm_key_vault_secret"` for reading secrets.

```hcl
ephemeral "azurerm_key_vault_secret" "main" {
  name         = var.eventhub.secret_name
  key_vault_id = var.eventhub.key_vault_id
}
```

See `references/key-vault.md` for writing secrets, reading pull secrets, lifecycle for KV secrets, and raw resource vs. Key Vault module trade-offs.

## RBAC and Authorization Delegation

Modules **do not create role assignments directly.** A common pattern is to delegate every `azurerm_role_assignment` to a centralized authorization module that the rest of your modules call. This is a recommended approach rather than a hard requirement, but the reasoning generalizes: keeping role assignments out of the leaf modules makes RBAC auditable in one place and avoids resource sprawl.

### Why centralize

- **Consistent naming and idempotency.** A shared module handles edge cases (existing assignments, transient principal-not-found errors) once.
- **Auditability.** All assignments live behind one module call and one shape. Reviewers can see every role grant in a single locals map.
- **No sprawl.** Direct `azurerm_role_assignment` resources scattered across leaf modules are difficult to inventory.

### The pattern

Build a map of role assignments in `locals`, pass it to the authorization module:

```hcl
locals {
  role_assignments = {
    "worker-subnet-network-contributor" = {
      role_name    = "Network Contributor"
      scope        = var.network.worker_subnet_id
      principal_id = var.identity.object_id
    }
    "dns-zone-contributor" = {
      role_name    = "Private DNS Zone Contributor"
      scope        = var.network.dns_zone_id
      principal_id = var.identity.object_id
    }
  }
}

module "role_assignments" {
  source  = "<your-terraform-registry>/azure/authorization/azurerm"
  version = "~> x.x"

  role_assignments = local.role_assignments
}
```

Don't:

```hcl
# Bad — direct role assignment in a leaf module
resource "azurerm_role_assignment" "network_contributor" {
  scope                = var.subnet_id
  role_definition_name = "Network Contributor"
  principal_id         = var.service_principal_object_id
}

# Bad — custom role definition in a leaf module
resource "azurerm_role_definition" "custom" { ... }
```

See `references/rbac-and-identity.md` for conditional role-assignment maps, principal-id input shapes, and when an `enabled` flag is justified versus inferring enablement from a non-null ID.

## Service Principals and Managed Identity

**Modules never create service principals.** SPNs are provisioned and rotated outside the module lifecycle — typically in a dedicated identity-management repository. Modules consume identities in their final form.

### Three identities most modules touch

1. **Terraform deployment identity** — the credential running `terraform apply`. Configured in the provider block, never managed by the module.
2. **Resource provider SPN** — the Azure resource provider's identity that needs RBAC on dependencies (e.g. an ARO RP needing Network Contributor on subnets). The module receives its `object_id` and grants that role.
3. **Service-specific SPN** — used by the deployed service itself. The module receives `client_id`/`client_secret` (typically as Key Vault secret names) to wire into resource configuration.

### Variable shape

```hcl
variable "identity" {
  description = <<-EOT
    Service principal identity for the deployed resource.
    `object_id`     - Object ID for RBAC role assignments.
    `client_id`     - Application/Client ID for authentication.
    `client_secret` - Client secret for authentication.
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

### Managed identity over service principals

Where the resource supports it, prefer managed identity over an explicit SPN. Within managed identity, prefer **user-assigned** when the lifecycle of the identity is decoupled from the resource (because you want to grant RBAC to it before the resource exists, or you want it to outlive a resource recreation), and **system-assigned** when the identity should live and die with the resource.

Accept the user-assigned identity ID as input — never create one inside a leaf module:

```hcl
variable "user_assigned_identity_id" {
  description = "Resource ID of an existing user-assigned managed identity."
  type        = string
  default     = null
}

dynamic "identity" {
  for_each = var.user_assigned_identity_id != null ? [1] : []
  content {
    type         = "UserAssigned"
    identity_ids = [var.user_assigned_identity_id]
  }
}
```

See `references/rbac-and-identity.md` for full identity patterns.

## Networking and Private Endpoints

Three rules:

1. **Public access is disabled by default.** Every Azure resource that supports a `public_network_access_enabled` (or equivalent) toggle defaults to `false` in your module.
2. **Private endpoints are first-class.** Either the module creates the PE or the example shows how to create one with the dedicated PE module.
3. **Network IDs come from the caller.** Subnet IDs, VNet IDs, DNS zone IDs, route table IDs are all inputs. The module never looks them up.

### Public access toggle

```hcl
variable "public_access_enabled" {
  description = "Whether public network access is enabled. Defaults to false (private only)."
  type        = bool
  default     = false
}
```

### Private endpoint multiplicity

When a service supports multiple private endpoints, don't embed PE creation inside the module — provide an example showing how callers create PEs against your module's output ID.

```hcl
# In examples/with-private-endpoint/main.tf:
module "service" {
  source = "<your-terraform-registry>/azure/<service>/azurerm"
  # ... no PE config
}

module "private_endpoint" {
  source  = "<your-terraform-registry>/azure/private-endpoint/azurerm"
  version = "~> x.x"

  target_resource_id = module.service.id
  subnet_id          = var.pe_subnet_id
}
```

### Private endpoint vs VNet integration

These are **different** features and shouldn't be conflated:

- **Private Endpoint** — a private IP in your VNet that maps to the service. `azurerm_private_endpoint`.
- **VNet Integration / Subnet Delegation** — the service is injected into a delegated subnet. `delegated_subnet_id` on the resource itself.

Most enterprise patterns standardize on Private Endpoints for ingress and disable VNet Integration unless the service genuinely requires it.

See `references/networking.md` for NSG rule delegation, DNS integration with an external DNS provider, firewall rules, and the pattern for accepting a network configuration object.

## Diagnostic Settings

Wire diagnostics to a caller-supplied destination — never hardcode a Log Analytics workspace ID inside a module.

### Variable

```hcl
variable "diagnostics" {
  description = <<-EOT
    Diagnostics configuration.
    `destination` - Resource ID of the destination (Log Analytics, Storage Account, or Event Hub).
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

### Parsing pattern

The module derives the destination type from the resource ID rather than asking the caller to set a `destination_type` field:

```hcl
locals {
  diag_resource_list = var.diagnostics != null ? split("/", var.diagnostics.destination) : []
  parsed_diag = var.diagnostics != null ? {
    log_analytics_id   = contains(local.diag_resource_list, "Microsoft.OperationalInsights") ? var.diagnostics.destination : null
    storage_account_id = contains(local.diag_resource_list, "Microsoft.Storage") ? var.diagnostics.destination : null
    event_hub_auth_id  = contains(local.diag_resource_list, "Microsoft.EventHub") ? var.diagnostics.destination : null
    metric             = var.diagnostics.metrics
    log                = var.diagnostics.logs
  } : null
}
```

See `references/diagnostics.md` for the diagnostic setting resource, retention policy notes, and how AzAPI fits in for resources whose diagnostic categories aren't yet exposed by azurerm.

## AzAPI Provider

Use AzAPI when:

- A resource type or property is not yet supported by `azurerm`.
- You need an ARM action (e.g. `listAdminCredentials`) with no `azurerm` equivalent.
- A preview API version is required and `azurerm` doesn't support it yet.

### Pin the API version

Always specify a concrete API version like `@2023-09-04`. Never `@latest` — a silent API change becomes a silent module change.

### Migrate back when azurerm catches up

AzAPI is a stopgap, not a destination. When `azurerm` adds support, migrate. Leaving an `azapi_update_resource` in place after the property lands in `azurerm` creates state drift between the two providers and confuses readers.

```hcl
# Good — concrete API version, narrow response export, comment explaining why
data "azapi_resource_action" "admin_credentials" {
  count = var.retrieve_admin_credentials ? 1 : 0

  type                   = "Microsoft.RedHatOpenShift/openShiftClusters@2023-09-04"
  resource_id            = azurerm_redhat_openshift_cluster.main.id
  action                 = "listAdminCredentials"
  response_export_values = ["kubeadminPassword"]
}
```

```hcl
# Bad — using AzAPI for a resource fully supported by azurerm
resource "azapi_resource" "aro_cluster" {
  type      = "Microsoft.RedHatOpenShift/openShiftClusters@2023-09-04"
  name      = var.name
  parent_id = var.resource_group.id
  # azurerm_redhat_openshift_cluster does this natively
}
```

See `references/diagnostics.md` for the AzAPI fallback pattern when a diagnostic category isn't yet exposed by `azurerm`.

## Lifecycle and `ignore_changes`

`lifecycle` is a sharp tool. Use `ignore_changes` only when there's a concrete reason — most commonly to silence drift caused by Azure Policy or other external controllers that mutate resources after Terraform creates them.

The two cases that genuinely warrant `ignore_changes`:

1. **Tag drift from Azure Policy.** Many enterprises run policies that inject or normalize tags after resource creation. Without `ignore_changes`, every `terraform plan` shows a tag diff that Terraform can never reconcile.
2. **Externally-managed fields.** DNS records sometimes have a `dns_view` or similar property that an external IPAM or DNS controller owns after creation.

```hcl
resource "azurerm_redhat_openshift_cluster" "main" {
  # ...
  lifecycle {
    ignore_changes = [
      tags,  # Azure Policy normalizes these post-create
    ]
  }
}
```

Anti-patterns:

- `ignore_changes = all` — almost never correct; it disables drift detection entirely.
- `ignore_changes` on inputs that ought to be variables — if the caller wants to control it, expose it as a variable and let the caller decide.

See `references/diagnostics.md` for the Azure Policy interaction in detail.

## Common Mistakes

1. **`data` source lookups inside modules** — accept resolved IDs and names as inputs instead. This is the single most common Azure module smell.
2. **Creating service principals in a leaf module** — SPNs are centrally provisioned; modules accept them as inputs.
3. **Direct `azurerm_role_assignment` in a leaf module** — delegate to a centralized authorization module.
4. **Outputting secrets** — write them to Key Vault and output the reference. Even `sensitive = true` outputs land in state and CI logs that capture the output map.
5. **Accepting raw credential values as variables** — accept Key Vault secret names instead, resolve at apply time.
6. **Public network access enabled by default** — flip the default to private; the toggle exists for the rare exception.
7. **Hardcoded diagnostics destinations** — accept a `destination` resource ID, parse the type from the ID.
8. **AzAPI used for resources `azurerm` already supports** — and the inverse: leaving AzAPI workarounds in place after `azurerm` catches up.
9. **Static nested blocks for optional features** (`identity {}`, `diagnostics {}`) — use `dynamic` blocks so the block disappears when the input is null.
10. **`ignore_changes = all` or speculative `ignore_changes`** — only ignore fields with a concrete reason, document the reason inline.

## Review Checklist

- [ ] No `data` sources except for genuinely global values (current tenant, KV secret reads with no input alternative).
- [ ] No service principal or service principal password resources inside the module.
- [ ] No `azurerm_role_assignment` or `azurerm_role_definition` inside the module — RBAC delegated to a centralized authorization module.
- [ ] No raw credential variables — credentials accepted as Key Vault secret names where applicable.
- [ ] No secret values exposed as module outputs — secrets written to Key Vault, references exported.
- [ ] `public_network_access_enabled` (or equivalent) defaults to `false` on every resource that supports it.
- [ ] Subnet IDs, VNet IDs, DNS zone IDs, and route table IDs are all inputs, never looked up.
- [ ] Diagnostic settings accept a destination resource ID; type is parsed from the ID.
- [ ] AzAPI usages pin an exact API version, narrow `response_export_values`, and include a comment explaining why AzAPI was needed.
- [ ] Optional nested blocks (`identity`, `diagnostics`, `customer_managed_key`) use `dynamic` blocks driven by input nullity.
- [ ] `lifecycle.ignore_changes` lists are minimal and have an inline comment explaining the reason.
- [ ] Managed identity (user-assigned) is preferred over service principal where supported; the identity ID is an input, not created inside the module.
- [ ] Private endpoints either created by the module or demonstrated in `examples/with-private-endpoint`.
- [ ] No hidden `data "azurerm_resource_group"` / `data "azurerm_subnet"` calls in submodules either.

## References

- `references/key-vault.md` — Key Vault integration patterns: accepting Key Vault by ID, writing metadata, reading secrets, ephemeral resources, lifecycle on KV secrets, and when a raw resource beats a Key Vault module.
- `references/rbac-and-identity.md` — RBAC delegation through a centralized authorization module, conditional role-assignment maps, accepting principal IDs and identity IDs as inputs, system-assigned vs user-assigned managed identity, the three identities most modules consume.
- `references/networking.md` — Private endpoints vs VNet integration, accepting subnet/VNet/DNS zone IDs, NSG rule delegation, public access disabled by default, firewall rules, integration with an external DNS provider.
- `references/diagnostics.md` — Diagnostic settings, destination parsing, retention policy notes, AzAPI fallback for unsupported diagnostic categories, and `ignore_changes` for tag drift from Azure Policy.
