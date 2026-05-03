---
name: terraform-state-and-lifecycle
description: Apply Terraform's state-mutation and lifecycle features safely — `moved` blocks, `import` blocks, `removed` blocks, and `lifecycle { ignore_changes / prevent_destroy / create_before_destroy / replace_triggered_by }`, plus the legacy `terraform state mv` CLI. Use this skill when refactoring a module without recreating resources, renaming a resource label, switching `count` ↔ `for_each`, importing an existing Azure resource into Terraform (brownfield ingestion), removing a resource from state without destroying it, ignoring tag drift caused by Azure Policy, suppressing rotating attributes, or deciding whether to use `prevent_destroy` (almost never) versus an Azure Resource Lock (the right answer). Triggers on phrases like "rename this resource", "import existing Azure resource", "module refactor without recreating", "ignore tag drift", "state mv", "moved block", "destroy then recreate", or any plan output showing destroy/create when a rename was intended. These constructs are judgment-heavy — applied wrong they delete production resources or hide drift.
---

# Terraform State and Lifecycle

Terraform's state-mutation features (`moved`, `import`, `removed`) and `lifecycle` meta-arguments are the difference between a safe refactor and a destroyed production resource. Default agent behavior — write the new HCL, run apply — will happily recreate resources that should have moved, or destroy resources that should have been imported. This skill makes these constructs part of the workflow when applicable, and explains the version requirements, the correct patterns, and the traps.

For variable shape (especially `name_override`), see `terraform-variable-design`. For `required_version` bumps, see `terraform-module-versioning`.

## Contents
- [Decision Matrix](#decision-matrix)
- [`moved` Blocks (Terraform 1.5+)](#moved-blocks-terraform-15)
- [`import` Blocks (Terraform 1.5+)](#import-blocks-terraform-15)
- [`removed` Blocks (Terraform 1.7+)](#removed-blocks-terraform-17)
- [`lifecycle { ignore_changes }`](#lifecycle--ignore_changes-)
- [`lifecycle { prevent_destroy }` — Almost Always Wrong](#lifecycle--prevent_destroy--almost-always-wrong)
- [`lifecycle { create_before_destroy }`](#lifecycle--create_before_destroy-)
- [`lifecycle { replace_triggered_by }`](#lifecycle--replace_triggered_by-)
- [`terraform state mv` — Legacy Path](#terraform-state-mv--legacy-path)
- [`name_override` Pattern](#name_override-pattern)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)

## Decision Matrix

Use this table to pick the right construct before you write any HCL.

| Need | Use |
|---|---|
| Rename a resource label | `moved` block |
| Move a resource between modules / submodules | `moved` block |
| Switch `count` ↔ `for_each` | `moved` block keyed appropriately |
| Adopt an existing Azure resource into state | `import` block (1.5+) |
| Remove from state without destroying the underlying resource | `removed { lifecycle { destroy = false } }` (1.7+) |
| Suppress legitimate external drift on specific attributes | `lifecycle.ignore_changes` (specific attrs only) |
| Protect against accidental destroy in production | Azure Resource Lock (NOT `prevent_destroy`) |
| Replace child when parent changes (graph can't see the dep) | `lifecycle.replace_triggered_by` |
| Avoid downtime on resource recreation | `lifecycle.create_before_destroy` |
| Refactor a name formula without renaming the underlying Azure resource | `name_override` variable + `coalesce(try(...))` |
| Pre-1.1 module that needs to refactor | `terraform state mv` (CLI) — and bump `required_version` |

If the agent reaches for `lifecycle.prevent_destroy` or `ignore_changes = all`, stop and re-read this skill.

## `moved` Blocks (Terraform 1.5+)

`moved` blocks let you rename, restructure, or re-key a resource and have Terraform update its state in-place during the next apply. No CLI surgery, no destroy/create, no downtime. They are the default tool for any in-place refactor.

### When to use

- Renaming a resource label: `azurerm_storage_account.storage` → `azurerm_storage_account.main`.
- Moving a resource between modules: `azurerm_subnet.app` → `module.networking.azurerm_subnet.app`.
- Switching `count` ↔ `for_each` keying.
- Restructuring nested modules without behavioral change.

### Pattern — rename

```hcl
resource "azurerm_storage_account" "main" {
  name                = local.storage_account_name
  resource_group_name = var.resource_group_name
}

moved {
  from = azurerm_storage_account.storage
  to   = azurerm_storage_account.main
}
```

### Pattern — `count` → `for_each`

```hcl
resource "azurerm_subnet" "app" {
  for_each = toset(["primary"])
}

moved {
  from = azurerm_subnet.app[0]
  to   = azurerm_subnet.app["primary"]
}
```

### Pattern — moving into a submodule

```hcl
moved {
  from = azurerm_subnet.app
  to   = module.networking.azurerm_subnet.app
}
```

### Validation

After adding the block, run:

```sh
terraform plan
```

The plan must show `# azurerm_storage_account.storage has moved to azurerm_storage_account.main` with **no destroy or create** for that resource. If you see destroy/create, the `from`/`to` addresses don't match — fix them before applying.

### Lifetime

Keep the `moved` block in the codebase **for at least one release cycle** so consumers upgrading from older versions are also covered. After every consumer has applied with the new block, you can delete it — leaving it longer is harmless.

### Versions

`moved` blocks require Terraform `>= 1.1.0` (stable in 1.5+). For `required_version < 1.1.0`, fall through to `terraform state mv`.

## `import` Blocks (Terraform 1.5+)

`import` blocks are the declarative way to bring an existing Azure resource into Terraform state. Strongly prefer them over the legacy `terraform import` CLI — the block is reviewable in PR, version-controlled, and removable after the import is committed.

### When to use

- A resource exists in Azure (created manually, by ARM, by another team) but isn't in this Terraform state.
- Brownfield ingestion: adopting a pre-existing Azure environment into a new module.
- Recovering after state loss where the resource still exists in Azure.

### Pattern

```hcl
import {
  to = azurerm_storage_account.main
  id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-prod/providers/Microsoft.Storage/storageAccounts/stprodfoo"
}

resource "azurerm_storage_account" "main" {
  name                     = "stprodfoo"
  resource_group_name      = "rg-prod"
  location                 = "eastus2"
  account_tier             = "Standard"
  account_replication_type = "GRS"
}
```

### Generating the resource block

In Terraform 1.5+, you can scaffold the resource block from the live resource:

```sh
terraform plan -generate-config-out=generated.tf
```

This writes a starter resource block matching the live shape. Treat it as a draft — clean it up, move it into your normal file layout, replace literals with variable/local references, then iterate `terraform plan` until the diff is empty.

### Workflow

1. Add the `import` block to a `.tf` file.
2. Run `terraform plan -generate-config-out=generated.tf` (1.5+) **or** hand-write the resource block.
3. Run `terraform plan` and iterate on the resource block until **no diff** appears.
4. Run `terraform apply` to commit the import.
5. After the import is in state, **delete the `import` block** from configuration.

### Why blocks beat the CLI

Reviewable in PR (not a runbook sentence), reproducible (re-running `apply` is a no-op), pairs with config generation, and the same code imports identically across environments. The CLI is only appropriate when `required_version < 1.5.0` — in which case bump the constraint.

## `removed` Blocks (Terraform 1.7+)

`removed` blocks let you delete a resource from configuration and choose what happens: destroy it (default) or remove it from state and leave it in Azure.

### When to use

- A resource is being deleted from configuration but should remain in Azure (e.g. ownership is moving to another module/team).
- Cleaning up after an `import` decision that turned out to be wrong.
- Decommissioning Terraform management of a resource without affecting the running workload.

### Pattern — orphan from state, keep the resource in Azure

```hcl
removed {
  from = azurerm_storage_account.legacy

  lifecycle {
    destroy = false
  }
}
```

After apply, `azurerm_storage_account.legacy` is no longer in state, but the storage account still exists in Azure.

### Pattern — destroy and remove (default)

```hcl
removed {
  from = azurerm_storage_account.legacy
}
```

Equivalent to deleting the `resource` block in pre-1.7 Terraform. Usually you can simply delete the resource block; `removed` is most valuable for the `destroy = false` case.

### Trap

`destroy = false` is a one-way action — once the resource leaves state, Terraform no longer manages it. If you later want it back, you need an `import` block. Don't use this block as a "soft delete." Use it only when the resource genuinely shouldn't be touched and is moving to other management.

### Versions

`removed` blocks require Terraform `>= 1.7.0`. For older versions, the equivalent is `terraform state rm <address>` followed by deleting the resource from configuration.

## `lifecycle { ignore_changes }`

`ignore_changes` tells Terraform not to plan changes for specific attributes even when state differs from configuration. It's a powerful escape hatch for legitimate external mutation — and a footgun when used to silence symptoms of a broken module.

### Justified uses

#### Tag drift caused by Azure Policy

Azure Policy can apply tags out-of-band (e.g. cost-center tags applied centrally). Ignore the specific keys, never the whole `tags` map:

```hcl
resource "azurerm_storage_account" "main" {
  # ...

  lifecycle {
    # Azure Policy applies these tags centrally. Ignoring them prevents
    # every plan from showing noise. Other tags are still managed by Terraform.
    ignore_changes = [
      tags["costcenter"],
      tags["environment"],
    ]
  }
}
```

#### Externally-rotated secrets

If a Key Vault automation rotates a database password, ignore that attribute so apply doesn't fight the rotator:

```hcl
resource "azurerm_mssql_server" "main" {
  # ...

  lifecycle {
    # Password is rotated by Key Vault automation; Terraform sets the initial
    # value only. See key-vault-rotation-policy in the platform repo.
    ignore_changes = [administrator_login_password]
  }
}
```

#### Numeric attributes managed by autoscaling

When the AKS cluster autoscaler manages `node_count`, ignore it:

```hcl
resource "azurerm_kubernetes_cluster" "main" {
  default_node_pool {
    name       = "system"
    vm_size    = var.system_pool_vm_size
    node_count = var.system_pool_min_count
  }

  lifecycle {
    # Autoscaler controls node_count. Terraform sets the initial value only.
    ignore_changes = [default_node_pool[0].node_count]
  }
}
```

### Not justified

#### `ignore_changes = all`

```hcl
# WRONG
lifecycle {
  ignore_changes = all
}
```

This disables all drift detection. If you really mean this, the resource shouldn't be in Terraform — move it out, or delete the configuration. There is no production scenario where ignoring all drift on a managed resource is correct.

#### `ignore_changes = [tags]` (whole map)

```hcl
# WRONG — masks adversarial tag drift
lifecycle {
  ignore_changes = [tags]
}
```

Always target specific keys. Whole-map ignore lets a malicious or accidental tag change go undetected forever. If you have many policy-applied keys, list them all:

```hcl
ignore_changes = [
  tags["costcenter"],
  tags["environment"],
  tags["owner"],
  tags["dataclassification"],
]
```

#### Hiding broken modules

If every apply produces noise on the same attribute, the module is broken (e.g. case-sensitivity normalization, default-value mismatches, computed-vs-configured collisions). Fix the module logic — don't suppress the symptom in `ignore_changes`.

### Rationale comments are mandatory

Every `ignore_changes` entry must have an inline comment explaining **why** — readers need to know what owns the attribute, not just that it's suppressed. No comment, no merge.

```hcl
lifecycle {
  ignore_changes = [
    # Rotated by `key-vault-rotation` runbook every 30 days.
    administrator_login_password,
    # Policy-applied centrally. Owned by the FinOps team.
    tags["costcenter"],
  ]
}
```

## `lifecycle { prevent_destroy }` — Almost Always Wrong

`prevent_destroy = true` causes Terraform to error if a plan would destroy the resource. It sounds like a safety net. It isn't.

### Why it's wrong

1. **It only blocks Terraform-initiated destroys.** Anyone with Azure portal/CLI access can delete the resource.
2. **It blocks legitimate operations.** Renames, provider switches, module restructures all require lifting it first — at which point it's no protection.
3. **It doesn't prevent removal from configuration.** Anyone deleting the `resource` block has to lift `prevent_destroy` to do so, defeating the guard.
4. **Production protection belongs at the platform layer, not in HCL.**

### The right answer: Azure Resource Locks

Use Azure Resource Locks for actual delete protection. They are platform-level, independent of Terraform state, and apply regardless of who or what initiates the destroy:

```hcl
resource "azurerm_management_lock" "main" {
  name       = "${azurerm_storage_account.main.name}-delete-lock"
  scope      = azurerm_storage_account.main.id
  lock_level = "CanNotDelete"
  notes      = "Production storage account — deletion requires lock removal in the portal."
}
```

`CanNotDelete` blocks delete operations from any source — Terraform, ARM, the portal, the CLI. `ReadOnly` is stricter (blocks updates too), but rarely useful for Terraform-managed resources because it blocks Terraform's legitimate updates as well.

### The only legitimate use of `prevent_destroy`

Dev-loop modules where you want a Terraform-side typo guard and the cost of false positives (annoying error messages) is acceptable:

```hcl
resource "azurerm_storage_account" "dev_loop_state" {
  # ...

  lifecycle {
    # Local dev-loop convenience guard. Production protection is the lock
    # in main.tf. Removing this block is a one-line PR.
    prevent_destroy = true
  }
}
```

Even here, an Azure Resource Lock is better. In production modules, never use `prevent_destroy`.

## `lifecycle { create_before_destroy }`

The default Terraform behavior on a forced replacement is destroy-then-create. For some resources this means downtime — or worse, dependent resources break during the gap.

### When to use

- The resource has external dependencies that break during the destroy/create gap. Common cases:
  - A network interface attached to a load balancer backend pool.
  - A DNS record pointed at the resource's IP.
  - A Front Door origin pointed at the resource's hostname.
- Replacement on every plan would otherwise cause downtime.

### Pattern

```hcl
resource "azurerm_network_interface" "main" {
  name                = "${local.name}-nic"
  location            = var.location
  resource_group_name = var.resource_group_name
  # ...

  lifecycle {
    create_before_destroy = true
  }
}
```

### Trap — name collisions

`create_before_destroy` requires the new resource to coexist with the old one for the duration of the apply. Azure won't let two resources share a name in the same resource group, so you usually need to vary the name (random suffix, `name_override`):

```hcl
resource "random_string" "nic_suffix" {
  length  = 4
  upper   = false
  special = false
  numeric = true
}

resource "azurerm_network_interface" "main" {
  name = "${local.base_name}-nic-${random_string.nic_suffix.result}"

  lifecycle {
    create_before_destroy = true
  }
}
```

Without a name change, apply fails with an Azure name-conflict error.

### Trap — propagates through dependencies

If A has `create_before_destroy = true` and B depends on A, B is also forced into create-before-destroy. This cascades further than you expect — test in non-production first.

## `lifecycle { replace_triggered_by }`

Forces a resource to be replaced when a referenced value changes. Useful when Terraform's dependency graph doesn't capture a relationship that should cause replacement.

### When to use

- Parent/child relationships where Terraform sees the parent's `id` as stable but a meaningful field changed and the child needs to be rebuilt.
- Bootstrapping resources whose contents depend on an external data source the graph doesn't connect.

### Pattern

```hcl
resource "azurerm_kubernetes_cluster_node_pool" "user" {
  name                  = "user"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = var.user_pool_vm_size

  lifecycle {
    # Recreate the node pool when the cluster is replaced.
    replace_triggered_by = [azurerm_kubernetes_cluster.main.id]
  }
}
```

### Use sparingly

Usually a sign the module structure is wrong — the dependency should be modeled directly (attribute reference), or the "child" should be owned inside the parent's block. Several `replace_triggered_by` clauses in one module is a refactor signal.

## `terraform state mv` — Legacy Path

For Terraform versions earlier than 1.1.0 (where `moved` blocks aren't available), the CLI command `terraform state mv` is the only way to refactor without recreate.

### When to use

- The module's `required_version` constraint is `< 1.1.0` and you cannot bump it.
- A consumer is stuck on an older Terraform version while the module already moved on.

### Pattern

```sh
terraform state mv \
  azurerm_storage_account.storage \
  azurerm_storage_account.main
```

Run this **once, before** the apply that introduces the new code. The consumer must then apply the new HCL.

### Document in CHANGELOG

State surgery is invisible from the diff — a consumer pulling the new module without running `state mv` first gets a destroy/create. Document the command:

```markdown
### Changed
- **Manual state migration required:** rename `azurerm_storage_account.storage` → `azurerm_storage_account.main`. Before applying, run:
  `terraform state mv azurerm_storage_account.storage azurerm_storage_account.main`
```

### Graduate when possible

The right long-term answer is bumping `required_version` to `>= 1.5.0` and using `moved` blocks — reviewable, idempotent, no consumer-side CLI step. Push for the version bump.

## `name_override` Pattern

When a module's name formula changes (e.g. `local.primary_name` evolves), existing consumers who already deployed under the old formula will see their resources recreated. The fix is to expose a `name_override` variable so consumers can opt to keep their existing names.

### Pattern

```hcl
variable "name_override" {
  description = <<-EOT
    Map of resource names to override the module's computed name. Use this when
    upgrading from an older version of the module that produced different names —
    set the override to the existing name so the resource is not renamed/recreated.
  EOT
  type = object({
    storage_account = optional(string)
    key_vault       = optional(string)
  })
  default = {}
}

locals {
  storage_account_name = coalesce(
    try(var.name_override.storage_account, null),
    "st${var.environment}${var.application}${random_string.suffix.result}",
  )
}

resource "azurerm_storage_account" "main" {
  name = local.storage_account_name
  # ...
}
```

### Why `coalesce(try(...))`

- `try(var.name_override.storage_account, null)` resolves to the override if set, `null` otherwise.
- `coalesce(..., computed_name)` falls back to the computed name when the override is `null`.
- Existing consumers set `name_override.storage_account = "stprodapp123"` to pin their existing name; new consumers leave it unset and get the computed name.

Variable-shape conventions (object types, `optional()`, defaults) live in `terraform-variable-design`. This skill covers when to introduce the override; that skill covers how to shape it.

## Common Mistakes

1. **`prevent_destroy = true` as a "safety net"** — doesn't protect against config deletion or non-Terraform deletes. Use Azure Resource Locks instead.
2. **`ignore_changes = all`** — disables all drift detection. The resource shouldn't be in Terraform if you mean this.
3. **`ignore_changes = [tags]` whole-map** — masks adversarial tag drift. Always target specific keys.
4. **`ignore_changes` without a rationale comment** — the reader can't tell intentional from forgotten.
5. **Missing `moved` block on a rename** — plan shows destroy/create instead of move; production resources get recreated.
6. **Using `terraform state mv` when a `moved` block would be cleaner** — invisible state surgery; consumers miss it.
7. **Importing without `-generate-config-out`** — hand-written resource block doesn't match the live resource; first apply produces a 200-line diff.
8. **`replace_triggered_by` masking a missing dependency** — the right fix is usually a direct attribute reference.
9. **`create_before_destroy` without varying the resource name** — fails at apply with an Azure name-conflict error.
10. **Removing a `moved` block too soon** — consumers upgrading from older versions see destroy/create. Keep it for at least one release cycle.

## Review Checklist

- [ ] Any rename of a resource label is paired with a `moved` block.
- [ ] Any `count` ↔ `for_each` switch is paired with a correctly-keyed `moved` block.
- [ ] `terraform plan` after a refactor shows "moved" with no destroy/create.
- [ ] `import` blocks (not `terraform import` CLI) are used for brownfield ingestion.
- [ ] `terraform plan -generate-config-out` was used to scaffold imported resource blocks where possible.
- [ ] Plan diff is empty before applying an import.
- [ ] `removed { lifecycle { destroy = false } }` is used only where the resource genuinely shouldn't be touched.
- [ ] `ignore_changes` targets specific attributes/keys, never `all` or whole `tags`.
- [ ] Every `ignore_changes` entry has an inline comment explaining why.
- [ ] `prevent_destroy` is not used in production modules; Azure Resource Locks are used instead.
- [ ] `create_before_destroy` resources have a name-varying mechanism (suffix, override) to avoid Azure name conflicts.
- [ ] `replace_triggered_by` is justified — not a workaround for a missing direct dependency.
- [ ] `terraform state mv` is documented in CHANGELOG when used.
- [ ] `name_override` variable is exposed for any module whose name formula has changed since release.
- [ ] `moved` / `import` / `removed` version requirements (1.1+, 1.5+, 1.7+) are reflected in `required_version`.
