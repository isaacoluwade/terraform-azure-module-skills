# Key Vault Integration Patterns

Patterns for integrating Azure Key Vault into a Terraform module. The decisions you make here ripple through state safety, secret rotation, and consumer ergonomics. Treat Key Vault as the canonical store for sensitive values; treat module outputs as references to those values, never as the values themselves.

## Contents
- [When the Module Manages Key Vault vs Accepts One](#when-the-module-manages-key-vault-vs-accepts-one)
- [The `keyvault` Variable](#the-keyvault-variable)
- [Writing Secrets to Key Vault](#writing-secrets-to-key-vault)
- [Reading Secrets from Key Vault](#reading-secrets-from-key-vault)
- [Resolving Credentials at Apply Time](#resolving-credentials-at-apply-time)
- [Don't Output Secrets](#dont-output-secrets)
- [Ephemeral Resources for Secrets (Terraform 1.10+)](#ephemeral-resources-for-secrets-terraform-110)
- [Raw Resource vs Key Vault Module](#raw-resource-vs-key-vault-module)
- [Lifecycle for Key Vault Secrets](#lifecycle-for-key-vault-secrets)
- [Using `key_vault_secret_id` on Other Resources](#using-key_vault_secret_id-on-other-resources)
- [Conditional Enablement](#conditional-enablement)

## When the Module Manages Key Vault vs Accepts One

Three situations:

1. **The module's job is Key Vault** — e.g. a `keyvault` module. It creates and manages the vault and exposes its `id` and `vault_uri`.
2. **The module needs Key Vault for its own operation** (a database needing a TDE key, an app service needing connection strings) — accept an existing Key Vault by `resource_id`, never create one.
3. **The module is Key Vault-aware but doesn't strictly need it** — make the integration optional. Accept a `keyvault` variable; if `enabled = false` (the default), skip every Key Vault interaction.

Most leaf modules fall into category 2 or 3.

## The `keyvault` Variable

Group every Key Vault-related field into a single variable. Don't scatter `keyvault_id`, `keyvault_secret_prefix`, `enable_keyvault` across the variable surface — group them.

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

Notes on the shape:

- `nullable = false` plus `optional()` defaults means callers can pass `{}` or omit the variable entirely and the module won't crash.
- `enabled` toggles the whole feature; the other fields are only consulted when `enabled = true`.
- A separate `enabled` flag is justified here because the feature has multiple sub-capabilities (write cluster metadata, read pull secret) that callers may want to control independently. For simpler cases where presence of `resource_id` is sufficient, drop `enabled` and infer from the ID — see [Conditional Enablement](#conditional-enablement) below.

## Writing Secrets to Key Vault

When the module produces credentials or metadata other components need (cluster console URLs, generated passwords, connection strings), write them to Key Vault rather than emitting them as outputs.

```hcl
locals {
  cluster_secrets = var.keyvault.enabled && var.keyvault.write_cluster_metadata ? {
    "${var.keyvault.secret_prefix}api-server-url" = azurerm_redhat_openshift_cluster.main.console_url
    "${var.keyvault.secret_prefix}console-url"    = azurerm_redhat_openshift_cluster.main.console_url
    "${var.keyvault.secret_prefix}cluster-id"     = azurerm_redhat_openshift_cluster.main.id
  } : {}
}

resource "azurerm_key_vault_secret" "cluster_metadata" {
  for_each = local.cluster_secrets

  name         = each.key
  value        = each.value
  key_vault_id = var.keyvault.resource_id
}
```

Patterns to notice:

- The map is built in `locals` and is empty when the feature is off — `for_each` over an empty map produces zero resources, no `count` ternary needed.
- A `secret_prefix` applied to every name lets the same Key Vault host secrets from many module instances without collisions.
- Use `coalesce()` to default the prefix to the resource's own name when the caller doesn't supply one:

```hcl
secret_prefix = coalesce(var.keyvault.secret_prefix, azurerm_redhat_openshift_cluster.main.name)
```

Cleaner than `var.keyvault.secret_prefix != null ? var.keyvault.secret_prefix : azurerm_redhat_openshift_cluster.main.name`.

## Reading Secrets from Key Vault

When the module needs a secret the caller has already placed in Key Vault — pull secrets, image registry credentials, certificates — read it via a guarded data source:

```hcl
data "azurerm_key_vault_secret" "pull_secret" {
  count = var.keyvault.enabled && var.keyvault.pull_secret_name != null ? 1 : 0

  name         = var.keyvault.pull_secret_name
  key_vault_id = var.keyvault.resource_id
}

locals {
  pull_secret = try(data.azurerm_key_vault_secret.pull_secret[0].value, null)
}
```

Note this is one of the legitimate uses of a data source — the secret value isn't knowable to the caller at plan time, and the alternative is asking the caller to interpolate the secret in their own code (which lands the secret in their state too, just shifted upstream).

Where Terraform 1.10+ is available, prefer `ephemeral` — see below.

## Resolving Credentials at Apply Time

When the module needs service principal credentials (`client_id`, `client_secret`), accept **Key Vault secret names**, not the credential values themselves.

```hcl
variable "identity" {
  description = <<-EOT
    Identity configuration for the module. Credentials are stored in Key Vault
    and resolved at apply time.
    `client_id_secret_name`     - Key Vault secret name containing the client ID.
    `client_secret_secret_name` - Key Vault secret name containing the client secret.
  EOT
  type = object({
    client_id_secret_name     = optional(string)
    client_secret_secret_name = optional(string)
  })
  default  = {}
  nullable = false
}

data "azurerm_key_vault_secret" "client_id" {
  count = var.keyvault.enabled && var.identity.client_id_secret_name != null ? 1 : 0

  name         = var.identity.client_id_secret_name
  key_vault_id = var.keyvault.resource_id
}

data "azurerm_key_vault_secret" "client_secret" {
  count = var.keyvault.enabled && var.identity.client_secret_secret_name != null ? 1 : 0

  name         = var.identity.client_secret_secret_name
  key_vault_id = var.keyvault.resource_id
}

locals {
  client_id     = try(data.azurerm_key_vault_secret.client_id[0].value, null)
  client_secret = try(data.azurerm_key_vault_secret.client_secret[0].value, null)
}
```

Why secret names instead of values:

- Plaintext credential variables land in `.tfvars` files, CI logs, and any place that prints variable values during debugging.
- Secret names are not sensitive — they can be committed to source control alongside other module configuration.
- Rotation lives in Key Vault: change the secret value, re-run the module, no module-input change required.

Anti-pattern:

```hcl
# Bad — raw credentials end up in state and plan output
variable "identity" {
  type = object({
    client_id     = string
    client_secret = string
  })
  sensitive = true
}
```

## Don't Output Secrets

```hcl
# Bad
output "kubeadmin_password" {
  value     = azurerm_redhat_openshift_cluster.main.kubeadmin_password
  sensitive = true
}
```

Even `sensitive = true` outputs are written to state and to any CI/CD log that prints the output map. Write the secret to Key Vault and output the reference — `key_vault_secret_id` or just the secret name within a known vault — instead.

```hcl
# Good — write to KV
resource "azurerm_key_vault_secret" "kubeadmin_password" {
  count = var.keyvault.enabled ? 1 : 0

  name         = "${var.keyvault.secret_prefix}kubeadmin-password"
  value        = local.kubeadmin_password
  key_vault_id = var.keyvault.resource_id
}

# Output the reference, not the value
output "kubeadmin_password_secret_id" {
  description = "The Key Vault secret ID containing the kubeadmin password."
  value       = try(azurerm_key_vault_secret.kubeadmin_password[0].id, null)
}
```

## Ephemeral Resources for Secrets (Terraform 1.10+)

When a module needs to read a secret to wire it into another resource, `ephemeral` is strictly preferable to `data` — the secret value is available during apply but is never persisted to state.

```hcl
ephemeral "azurerm_key_vault_secret" "main" {
  name         = var.eventhub.secret_name
  key_vault_id = var.eventhub.key_vault_id
}

resource "azurerm_eventhub_authorization_rule" "main" {
  # ... use ephemeral.azurerm_key_vault_secret.main.value
}
```

Constraints:

- Requires Terraform >= 1.10.
- The provider must support ephemeral for that resource type.
- The ephemeral value is only available during the apply that reads it — you can't reference it from outside the same plan.

When ephemeral is unavailable, fall back to a `data "azurerm_key_vault_secret"` and document the state-residence trade-off in the module README.

## Raw Resource vs Key Vault Module

For a single secret write, prefer the raw `azurerm_key_vault_secret` resource over wrapping it in a Key Vault helper module.

```hcl
# Good — raw resource is shorter and clearer for a single secret
resource "azurerm_key_vault_secret" "cluster_metadata" {
  for_each = local.cluster_secrets

  name         = each.key
  value        = each.value
  key_vault_id = var.keyvault.resource_id
}
```

```hcl
# Bad — module wrapper for a single resource adds indirection without value
module "keyvault_secret" {
  source  = "<your-terraform-registry>/azure/keyvault-secret/azurerm"
  version = "~> x.x"

  secrets      = local.cluster_secrets
  key_vault_id = var.keyvault.resource_id
}
```

Module wrappers are worth it when there's genuine added behavior: tagging conventions, content-type detection, expiration policy, retry semantics. For a straight secret write, the raw resource wins.

## Lifecycle for Key Vault Secrets

Two cases worth knowing.

### Ignoring `value` changes for externally-rotated secrets

If a secret is rotated by an external process (an automation runbook, a key-rotation scheduler) the module shouldn't fight that rotation by re-applying the original value:

```hcl
resource "azurerm_key_vault_secret" "client_secret" {
  name         = "client-secret"
  value        = var.initial_client_secret
  key_vault_id = var.keyvault.resource_id

  lifecycle {
    ignore_changes = [
      value,
      content_type,
    ]
  }
}
```

The module seeds the initial value, then steps back. Subsequent rotations happen outside Terraform.

### Tag drift from Azure Policy

Most enterprises run Azure Policy that injects tags onto every Key Vault secret. Without `ignore_changes`, Terraform shows a permanent diff:

```hcl
resource "azurerm_key_vault_secret" "main" {
  # ...
  tags = local.tags

  lifecycle {
    ignore_changes = [
      tags["createdAt"],   # injected by policy
      tags["createdBy"],   # injected by policy
    ]
  }
}
```

Document inline why each ignored field is ignored — future readers should not have to guess.

## Using `key_vault_secret_id` on Other Resources

Many Azure resources accept a `key_vault_secret_id` argument directly — TDE keys, Customer Managed Keys, App Service secret-backed app settings, and so on. Pass the secret ID through rather than reading the secret value and writing it back into the resource as plaintext:

```hcl
# Good — the resource references the KV secret directly
resource "azurerm_mssql_server_transparent_data_encryption" "main" {
  server_id        = azurerm_mssql_server.main.id
  key_vault_key_id = var.keyvault.tde_key_id
}
```

```hcl
# Bad — reading the value and putting it in state
data "azurerm_key_vault_secret" "tde" {
  name         = var.tde_secret_name
  key_vault_id = var.keyvault.resource_id
}

resource "some_resource" "main" {
  some_field = data.azurerm_key_vault_secret.tde.value  # Now in state
}
```

The first pattern has the resource pull the value at runtime; Terraform only stores the reference.

## Conditional Enablement

Two valid shapes:

### Shape 1 — separate `enabled` flag

Use when the integration has multiple independent sub-capabilities:

```hcl
variable "keyvault" {
  type = object({
    enabled                = optional(bool, false)
    resource_id            = optional(string)
    write_cluster_metadata = optional(bool, true)
    pull_secret_name       = optional(string)
  })
  default  = {}
  nullable = false
}
```

`enabled` is the master switch. `write_cluster_metadata` and `pull_secret_name` are sub-features.

### Shape 2 — infer enablement from a non-null ID

Use when the only question is "is there a Key Vault to talk to?":

```hcl
variable "keyvault" {
  type = object({
    resource_id = optional(string)
    ip_rules    = optional(list(string), [])
  })
  default  = {}
  nullable = false
}

# Then:
resource "azurerm_key_vault_secret" "main" {
  count = var.keyvault.resource_id != null ? 1 : 0
  # ...
}
```

A separate `enable_keyvault` boolean alongside `keyvault_id` is redundant — if the ID is set, the feature is on; if it's null, it's off.
