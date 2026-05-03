# RBAC and Identity Patterns

Patterns for handling Azure AD identities (service principals and managed identities) and role-based access control inside a Terraform module. The headline rules:

1. Modules **never create** service principals — they accept them.
2. Modules **never create** role assignments directly — they delegate to a centralized authorization module.
3. Modules **never create** managed identities for resources outside the module's primary scope — they accept user-assigned identity IDs as inputs.

## Contents
- [The Three Identities](#the-three-identities)
- [Service Principal Variable Shape](#service-principal-variable-shape)
- [Never Create Service Principals in a Module](#never-create-service-principals-in-a-module)
- [Centralized Authorization Module](#centralized-authorization-module)
- [Building the Role-Assignment Map](#building-the-role-assignment-map)
- [Conditional Role Assignments](#conditional-role-assignments)
- [Managed Identity vs Service Principal](#managed-identity-vs-service-principal)
- [System-Assigned vs User-Assigned](#system-assigned-vs-user-assigned)
- [Accepting an Identity ID Rather Than Creating One](#accepting-an-identity-id-rather-than-creating-one)
- [Dynamic `identity {}` Block Pattern](#dynamic-identity--block-pattern)
- [RBAC vs Local Credentials](#rbac-vs-local-credentials)
- [Principal IDs as Inputs](#principal-ids-as-inputs)

## The Three Identities

Most Azure modules touch up to three distinct identities. Understand which is which before designing the variable surface.

1. **Terraform deployment identity.** The credential running `terraform apply`. This lives in the provider block. The module never references it; it never appears in module variables.
2. **Resource provider service principal.** Some Azure services run under a first-party SPN that needs role assignments on the dependencies you pass in (e.g. ARO RP needing Network Contributor on the worker subnet). The module receives the SPN's `object_id` as input and uses it as the `principal_id` in role assignments.
3. **Service-specific service principal.** Used by the deployed service itself for its own operations (e.g. an OpenShift cluster authenticating outbound to Azure). The module receives `client_id`/`client_secret` (or, preferably, Key Vault secret names) and wires them into the resource configuration.

Each role serves a different purpose. Don't conflate them in a single variable.

## Service Principal Variable Shape

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

`sensitive = true` on the outer variable hides every field in plan output, including `object_id` (which is technically not a secret but is treated as one to keep the surface uniform).

When the module also accepts a separate resource-provider SPN object ID, give it its own field — don't reuse `object_id` for two different identities:

```hcl
variable "identity" {
  type = object({
    object_id        = string  # the service SPN's object ID
    client_id        = string
    client_secret    = string
    aro_rp_object_id = optional(string, "724ac4fb-80db-493b-a905-7f1a9d2622b9")  # the resource provider SPN
  })
  nullable  = false
  sensitive = true
}
```

The default value for `aro_rp_object_id` here is the well-known resource provider object ID, which doesn't change across tenants (it's the same first-party identity Microsoft publishes). Default it; don't make every caller look it up.

## Never Create Service Principals in a Module

```hcl
# Bad — never do this
resource "azuread_application" "cluster" {
  display_name = "aro-cluster-${var.name}"
}

resource "azuread_service_principal" "cluster" {
  client_id = azuread_application.cluster.client_id
}

resource "azuread_service_principal_password" "cluster" {
  service_principal_id = azuread_service_principal.cluster.id
}
```

Reasons:

- **Lifecycle mismatch.** SPN credentials need to outlive any single Terraform run; baking creation into a leaf module ties credential rotation to module reapplies.
- **Permissions sprawl.** Creating SPNs requires directory-level permissions; granting that to every module's runner gives leaf-module pipelines tenant-wide identity-creation power.
- **Audit trail.** Centralized SPN management means there's one place to see every identity that exists, who owns it, and when it was rotated.
- **Recreation risk.** A `terraform destroy` of the module shouldn't delete an identity that other systems depend on.

Treat SPN provisioning as an upstream concern owned by an identity-management repository or process. The module consumes the identity in its final form.

Equally bad — looking up an SPN you didn't create:

```hcl
# Bad — hidden dependency on AAD state
data "azuread_service_principal" "cluster" {
  client_id = var.service_principal_client_id
}
# Then using data.azuread_service_principal.cluster.object_id
```

Accept the `object_id` directly as input.

## Centralized Authorization Module

A common organizational pattern is to publish a dedicated authorization module that every other module calls for its role assignments. This is a recommended approach rather than a hard requirement — what matters is that role assignments aren't scattered across leaf modules.

```hcl
module "role_assignments" {
  source  = "<your-authorization-module>"
  version = "~> x.x"

  role_assignments = local.role_assignments
}
```

The authorization module typically handles:

- Naming the role assignment with a deterministic UUID derived from `(scope, role, principal)`.
- Tolerating "already exists" errors when an upstream process pre-created the assignment.
- Optional principal-type enforcement (`principal_type = "ServicePrincipal"`).
- Idempotency on transient AAD lookup failures.

If your organization doesn't yet have such a module, the same patterns apply to a thin internal wrapper around `azurerm_role_assignment`. The point is the **consolidation**, not the specific module name.

## Building the Role-Assignment Map

```hcl
locals {
  role_assignments = {
    "worker-subnet-network-contributor" = {
      role_name    = "Network Contributor"
      scope        = var.network.worker_subnet_id
      principal_id = var.identity.object_id
    }
    "master-subnet-network-contributor" = {
      role_name    = "Network Contributor"
      scope        = var.network.master_subnet_id
      principal_id = var.identity.object_id
    }
    "route-table-network-contributor" = {
      role_name    = "Network Contributor"
      scope        = var.network.route_table_id
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
  source  = "<your-authorization-module>"
  version = "~> x.x"

  role_assignments = local.role_assignments
}
```

Keys are descriptive — `"worker-subnet-network-contributor"` reads better than `"ra1"` when the plan shows `module.role_assignments.azurerm_role_assignment.main["worker-subnet-network-contributor"]`. Static keys also keep `for_each` happy: Terraform must know the keys at plan time.

## Conditional Role Assignments

Optional features generate optional assignments. Build them as separate maps and `merge()`:

```hcl
locals {
  base_role_assignments = {
    "subnet-contributor" = {
      role_name    = "Network Contributor"
      scope        = var.network.subnet_id
      principal_id = var.identity.object_id
    }
  }

  keyvault_role_assignments = var.keyvault.enabled ? {
    "keyvault-secrets-officer" = {
      role_name    = "Key Vault Secrets Officer"
      scope        = var.keyvault.resource_id
      principal_id = var.identity.object_id
    }
  } : {}

  diagnostics_role_assignments = var.diagnostics != null ? {
    "diagnostics-monitoring-contributor" = {
      role_name    = "Monitoring Contributor"
      scope        = var.diagnostics.destination
      principal_id = var.identity.object_id
    }
  } : {}

  role_assignments = merge(
    local.base_role_assignments,
    local.keyvault_role_assignments,
    local.diagnostics_role_assignments,
  )
}
```

The empty-map-when-disabled trick keeps `for_each` happy without `count` ternaries. Keys must be unique across the merged maps; namespace them by feature (`keyvault-*`, `diagnostics-*`) to avoid collisions.

## Managed Identity vs Service Principal

Where the Azure resource supports it, prefer **managed identity**. Managed identities have no credentials to rotate, can't leak via `.tfvars` or CI logs, and integrate with Azure RBAC without an AAD round-trip on the module's part.

Use a service principal when:

- The resource doesn't support managed identity (rare in modern azurerm).
- The identity needs to authenticate from outside Azure.
- An external service requires an SPN-shaped credential and won't accept federated identity.

Otherwise, managed identity wins.

## System-Assigned vs User-Assigned

Two flavors:

- **System-assigned** — created and destroyed alongside the resource. Use when the identity's lifecycle is precisely the resource's lifecycle.
- **User-assigned** — created independently, attached to one or more resources. Use when:
  - You want to grant the identity RBAC **before** the resource exists (so the resource doesn't fail on first apply trying to access dependencies).
  - The identity should outlive a recreate-and-replace of the resource.
  - Multiple resources should share an identity.

User-assigned is the more common enterprise default — it decouples identity provisioning from resource provisioning, which is exactly the property that makes modules portable.

## Accepting an Identity ID Rather Than Creating One

A leaf module **does not** create user-assigned managed identities. It accepts one as input.

```hcl
variable "user_assigned_identity_id" {
  description = "Resource ID of an existing user-assigned managed identity."
  type        = string
  default     = null
}
```

The reasoning is the same as for service principals — identity provisioning is an upstream concern, RBAC on the identity needs to be granted before the consuming resource exists, and a `terraform destroy` shouldn't delete an identity other systems may share.

A dedicated identity module (or a centralized identity provisioning workflow) creates identities and exposes their IDs; consumer modules accept the ID.

## Dynamic `identity {}` Block Pattern

Most Azure resources expose an optional `identity {}` block. Use a `dynamic` block keyed on input nullity so the block disappears entirely when no identity is configured:

```hcl
resource "azurerm_storage_account" "main" {
  # ...

  dynamic "identity" {
    for_each = var.identity_type != null ? [1] : []
    content {
      type         = var.identity_type
      identity_ids = var.identity_type == "UserAssigned" ? [var.user_assigned_identity_id] : null
    }
  }
}
```

A static block is wrong because it forces a value even when the caller doesn't want an identity at all:

```hcl
# Bad — fails when var.identity_type is null
identity {
  type         = var.identity_type
  identity_ids = var.identity_ids
}
```

For resources that support both system- and user-assigned simultaneously (`SystemAssigned, UserAssigned`), gate `identity_ids` on whether the type string contains `UserAssigned`:

```hcl
dynamic "identity" {
  for_each = var.identity_type != null ? [1] : []
  content {
    type = var.identity_type
    identity_ids = (
      can(regex("UserAssigned", var.identity_type)) ? var.user_assigned_identity_ids : null
    )
  }
}
```

## RBAC vs Local Credentials

Prefer Entra RBAC over admin keys, local accounts, or static tokens. AKS is the canonical example:

```hcl
# Good — RBAC on, local accounts off
resource "azurerm_kubernetes_cluster" "main" {
  # ...
  role_based_access_control_enabled = true
  local_account_disabled            = true

  azure_active_directory_role_based_access_control {
    azure_rbac_enabled = true
    tenant_id          = var.tenant_id
  }
}
```

```hcl
# Bad — local admin path left enabled
resource "azurerm_kubernetes_cluster" "main" {
  # ...
  local_account_disabled = false
}

output "kube_admin_config" {
  value     = azurerm_kubernetes_cluster.main.kube_admin_config
  sensitive = true
}
```

Outputting `kube_admin_config` exposes a long-lived admin credential into state. With RBAC and local accounts disabled, callers authenticate with their own AAD identity — no shared credential exists to leak.

The same principle applies to:

- Storage account keys (prefer RBAC + `shared_access_key_enabled = false`).
- SQL Server admin passwords (prefer Azure AD admin, disable SQL auth).
- Service Bus / Event Hub SAS keys (prefer RBAC + Azure AD).
- Cosmos DB master keys (prefer RBAC where available).

## Principal IDs as Inputs

When the module needs to grant access to an external principal — a deployment SPN, a developer group, a managed identity from another component — accept the principal ID as input:

```hcl
variable "additional_role_assignments" {
  description = <<-EOT
    Additional role assignments to grant on the deployed resource.
    Map key is the assignment label; values are the role and principal.
  EOT
  type = map(object({
    role_name    = string
    principal_id = string
  }))
  default  = {}
  nullable = false
}
```

The module then merges these into the role-assignment map:

```hcl
locals {
  caller_role_assignments = {
    for k, v in var.additional_role_assignments : k => {
      role_name    = v.role_name
      scope        = azurerm_storage_account.main.id
      principal_id = v.principal_id
    }
  }

  role_assignments = merge(
    local.base_role_assignments,
    local.caller_role_assignments,
  )
}
```

This lets the caller grant access without forking the module — they pass in `{ "data-team-reader" = { role_name = "Storage Blob Data Reader", principal_id = "..." } }` and the assignment lands on the right scope.

Don't accept a principal name and look it up — that's a data source, with all the problems described in the main skill.
