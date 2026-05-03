---
name: terraform-provider-versioning
description: Declare provider and Terraform version constraints in `versions.tf` for reusable Terraform modules — covering file naming (`versions.tf`, not `terraform.tf`), the `terraform { required_version = ... }` block, the `required_providers` block, and the `>=` vs `~>` constraint philosophy that lets modules compose without lock-fights. Covers AzureRM and AzAPI minimums, configuration_aliases, why modules must not declare `provider {}` blocks, what to do when a consumer pins an older version, and how provider bumps must be reflected in CHANGELOG.md. Triggers on: writing or reviewing `versions.tf`, "pin provider version", "required_providers", "Terraform version constraint", "AzureRM version", picking a minimum version, resolving cross-module provider conflicts, adding the AzAPI provider, or auditing a module against AVM TFNFR27. Use this whenever provider versions are touched — a wrong constraint silently breaks every consumer.
---

# Terraform Provider Versioning

Get `versions.tf` right and your module composes cleanly with any other module the consumer pulls in. Get it wrong — pin a static version, forget the file, or declare a `provider {}` block — and you turn one module into a blocker for the entire workspace.

This skill covers the *provider* and *Terraform* version constraints that live inside the module. For the module's own semver and changelog (a different kind of versioning), see `terraform-module-versioning`. For overall file layout, see `terraform-module-structure`.

## Contents
- [File Naming](#file-naming)
- [versions.tf Structure](#versionstf-structure)
- [Constraint Philosophy: `>=` vs `~>`](#constraint-philosophy--vs-)
- [Why Static Versions Break Composition](#why-static-versions-break-composition)
- [Terraform Version](#terraform-version)
- [Common Providers](#common-providers)
- [AzAPI Provider](#azapi-provider)
- [Modules Must Not Configure Providers](#modules-must-not-configure-providers)
- [configuration_aliases for Multi-Provider Modules](#configuration_aliases-for-multi-provider-modules)
- [Modules vs Examples vs Test Plans](#modules-vs-examples-vs-test-plans)
- [Minimum Version Must Match Feature Requirements](#minimum-version-must-match-feature-requirements)
- [When a Consumer Locks an Older Version](#when-a-consumer-locks-an-older-version)
- [CHANGELOG for Provider Bumps](#changelog-for-provider-bumps)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)

## File Naming

Provider and Terraform version constraints go in **`versions.tf`** — not `terraform.tf`, not `providers.tf`, not at the top of `main.tf`. One file, one purpose, predictable location.

You will see some style guides (including AVM examples) use `terraform.tf`. We standardize on `versions.tf` because the file's job is to declare versions, and reviewers searching for "where is the provider pinned" find it immediately.

Every module — root, submodule, example, test plan — must have a `versions.tf`.

## versions.tf Structure

A minimal, well-formed `versions.tf` for a module:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 4.10.0"
    }
  }
}
```

Three things, in this order:

1. `required_version` — minimum Terraform CLI version.
2. `required_providers` — every provider the module *actually uses*. Don't list providers you might use later; don't list providers used only by your test harness.
3. Each provider entry has both `source` and `version`. Source is mandatory since Terraform 0.13 and there is no excuse to omit it.

## Constraint Philosophy: `>=` vs `~>`

This is the most important rule in the file. A module is a library. It will be composed with other modules the consumer picks. **The constraint you write decides whether composition is possible.**

| Constraint | Use it for |
|---|---|
| `>= X.Y.Z` | Default for **certified modules**. Floor only — lets the consumer or another sibling module pick the actual version. |
| `~> X.Y.Z` | Acceptable for ancillary providers (`random`, `null`, `time`) where the API is stable and you want patch-only drift. |
| `>= X.Y.Z, < A.0.0` | Use when a known major release breaks something you depend on. Document why in a comment. |
| `>= X.Y.Z, < X.A.B` | Use when a specific later version has a known bug. Pin the upper bound to the broken version, not arbitrarily. |
| `"X.Y.Z"` (exact) | **Never** in a reusable module. Pinning an exact version means your module cannot coexist with any other module that pins anything else. Acceptable only in *examples* (see below). |
| `latest` | Never. Not a real constraint — Terraform doesn't honor it the way Docker does. |

### Good — minimum constraint for the primary provider

```hcl
azurerm = {
  source  = "hashicorp/azurerm"
  version = ">= 4.10.0"
}
```

The consumer's workspace will resolve to whatever satisfies *all* modules' minimums. That's the whole point.

### Good — pessimistic constraint for an ancillary provider

```hcl
random = {
  source  = "hashicorp/random"
  version = "~> 3.4.0"
}
```

`~> 3.4.0` allows `3.4.x` but not `3.5.0`. Fine for `random` and `null` because you don't need new features and patch updates rarely break.

### Good — bounded range avoiding a known-bad release

```hcl
azurerm = {
  source  = "hashicorp/azurerm"
  version = ">= 4.10.0, < 4.15.0"
  # 4.15.0 reverted the diagnostic settings schema we depend on; remove upper bound when 4.16.0 ships.
}
```

### Bad — static version

```hcl
# Do NOT do this in a module — it cannot compose with sibling modules.
azurerm = {
  source  = "hashicorp/azurerm"
  version = "4.9.0"
}
```

## Why Static Versions Break Composition

Concretely, here is what happens when two modules in the same workspace both pin exact versions:

```text
Module A: azurerm version = "4.9.0"
Module B: azurerm version = "4.12.0"

Error: no available version of hashicorp/azurerm satisfies the
       constraints "4.9.0, 4.12.0".
```

Terraform has no version that is simultaneously `4.9.0` and `4.12.0`. The deployment fails. The fix is not negotiable — one or both modules must be edited.

With minimum constraints:

```text
Module A: azurerm version = ">= 4.9.0"
Module B: azurerm version = ">= 4.12.0"

Resolved: 4.12.0 (or any later available version)
```

Both modules' floors are satisfied. The workspace's `.terraform.lock.hcl` records the chosen version. Everyone's happy.

This is why `>=` is the default for modules — it expresses "I need *at least* this much" instead of "I need *exactly* this and nothing else."

## Terraform Version

Set the Terraform CLI floor to the lowest version that supports every language feature the module uses:

```hcl
terraform {
  required_version = ">= 1.5.0"
}
```

Common floors and what they unlock:

| Floor | What you get |
|---|---|
| `>= 1.0.0` | Stable release line. |
| `>= 1.3.0` | `optional()` attributes in `object` types. |
| `>= 1.5.0` | `import` blocks, `check` blocks. |
| `>= 1.6.0` | `terraform test` framework. |
| `>= 1.8.0` | Provider-defined functions. |

`~> 1.x.y` (e.g. `~> 1.5.0`) is acceptable but rarely useful — Terraform's CLI minor releases are backward-compatible, so a hard upper bound just creates churn.

Pick the lowest version that supports the syntax you use, not the latest. Like provider versions, bumping the Terraform floor is a breaking change for consumers.

## Common Providers

A module that uses several providers might look like:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 4.10.0"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = ">= 2.50.0"
    }
    azapi = {
      source  = "Azure/azapi"
      version = ">= 2.0.1"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.4.0"
    }
    null = {
      source  = "hashicorp/null"
      version = "~> 3.2.0"
    }
  }
}
```

**Only declare providers the module actually uses.** If the module only manages Azure resources, do not list `random` "just in case" — it forces the consumer's workspace to download a provider for nothing.

If a provider is used **only by the test harness** (e.g. `random` for generating test resource names), it belongs in `test/.../versions.tf`, not in the module's root `versions.tf`.

## AzAPI Provider

When the AzureRM provider doesn't yet support an Azure feature, fall back to AzAPI. Target the 2.x line:

```hcl
azapi = {
  source  = "Azure/azapi"
  version = ">= 2.0.1"
}
```

The 2.x release introduced typed resources, better drift detection, and is meaningfully more pleasant to use than 1.x. If you're starting fresh, there is no reason to pin below 2.0.1.

Use AzAPI surgically — for the specific resource AzureRM doesn't cover — rather than wholesale. AzureRM gives you better diff output and richer defaults for everything it supports.

## Modules Must Not Configure Providers

This is enforced by AVM (`TFNFR27`) and is the rule that keeps modules reusable.

A reusable module **declares** what providers it requires (`required_providers`). It must **not configure** them with a `provider "azurerm" { ... }` block. Configuration — subscription IDs, tenant IDs, `features {}` — belongs in the *root* module (the workspace) that calls your module.

```hcl
# versions.tf in a reusable module
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 4.10.0"
    }
  }
  required_version = ">= 1.5.0"
}

# No `provider "azurerm" { ... }` block anywhere in the module.
```

```hcl
# providers.tf in a root module / workspace (NOT inside the reusable module)
provider "azurerm" {
  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id
  features {}
}
```

Why this matters: if the module configures a provider with a hardcoded subscription, every consumer is forced into that subscription. Multi-tenant, multi-environment, and aliased-provider scenarios all break.

```hcl
# Anti-pattern — do NOT do this in a reusable module.
provider "azurerm" {
  features {}
  subscription_id = "hardcoded-sub-id"
}
```

## configuration_aliases for Multi-Provider Modules

The one exception to "no provider config in modules" is `configuration_aliases`. Use it when the module needs to receive *more than one* configuration of the same provider — e.g. a module that creates resources in both a hub and a spoke subscription.

```hcl
terraform {
  required_providers {
    azurerm = {
      source                = "hashicorp/azurerm"
      version               = ">= 4.10.0"
      configuration_aliases = [azurerm.hub]
    }
  }
}
```

```hcl
# The consumer passes both providers explicitly.
module "peering" {
  source = "..."

  providers = {
    azurerm     = azurerm.spoke
    azurerm.hub = azurerm.hub
  }
}
```

`configuration_aliases` declares the contract; the consumer fulfills it. The module still has zero `provider {}` blocks of its own.

## Modules vs Examples vs Test Plans

The constraint strategy differs by context. Same `versions.tf` shape, different version strings.

| Context | Strategy | Why |
|---|---|---|
| Reusable module (root) | `>= X.Y.Z` (flexible minimum) | Allows composition with other modules. |
| `examples/<name>/versions.tf` | `"X.Y.Z"` (pinned exact) | Gives the operator a reproducible, known-good baseline. |
| `test/.../versions.tf` | `>= X.Y.Z` (flexible minimum) | Tests against both the floor and the latest. |

### Examples — pin exact versions

Examples are *consumer-facing*. They are what someone copies into a workspace to deploy the module for the first time. Reproducibility matters more than composition flexibility, because they're standalone.

```hcl
# examples/sample/versions.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "4.65.0" # pinned for reproducible example deployments
    }
    azapi = {
      source  = "Azure/azapi"
      version = "~> 2.0"
    }
  }
}
```

Add a comment when the pin is non-obvious. Don't be afraid to explain *why* this version.

```hcl
# Bad — examples should not use module-style open-ended constraints.
required_providers {
  azurerm = {
    source  = "hashicorp/azurerm"
    version = ">= 4.10.0"
  }
}
```

### Test plans — flexible constraints, no default-only `provider {}` blocks

Test plans should use `>=` so CI can verify the module works against both the declared floor and the current latest.

```hcl
# test/integration/terraform/plan/versions.tf
azurerm = {
  source  = "hashicorp/azurerm"
  version = ">= 4.10.0"
}
```

If a test plan needs a `provider "azurerm" { features {} }` block but the block contains *only* default values, drop the file entirely — defaults already apply.

```hcl
# Unnecessary — `features {}` is default behavior.
provider "azurerm" {
  features {}
}
```

And: **a provider used only in tests does not belong in the module's `versions.tf`.** If `random_pet` is used by Terratest to generate a unique resource group name but is never used in the module's resources, declare `random` only in the test plan's `versions.tf`.

## Minimum Version Must Match Feature Requirements

The minimum version is not "whatever was current when I wrote this." It is the *earliest* provider release that supports every feature the module actually uses.

- **Don't bump without reason.** If no resource or property in the module needs a higher version, don't increase it. A bump forces every consumer to upgrade — a real cost paid for nothing.
- **Don't lower without reason.** If you drop the minimum, reviewers will assume you removed functionality. Always explain in the PR description.
- **Bumping for a bug fix is fine** — but say so. Pin to the bug-fix release, not arbitrarily higher.

```hcl
# Good — minimum matches when the feature was introduced.
azurerm = {
  source  = "hashicorp/azurerm"
  version = ">= 4.13.0" # required by azurerm_virtual_network_flow_log
}
```

```hcl
# Bad — bumped to "latest" with no documented reason.
azurerm = {
  source  = "hashicorp/azurerm"
  version = ">= 4.65.0" # nothing in this module needs 4.65
}
```

When a refresh fixes bugs *and* adopts a feature requiring a newer provider, split into two PRs: bug fixes with no version bump first, then the new feature plus the provider bump it requires. Consumers who only want the bug fix don't get pulled into a forced provider upgrade.

## When a Consumer Locks an Older Version

If a consumer's `.terraform.lock.hcl` pins, say, AzureRM 4.5.0 and you need 4.10.0 minimum, the consumer runs `terraform init -upgrade` and Terraform regenerates the lock file with the highest version that satisfies every module's minimum. If they can't upgrade (org policy, in-flight migration), they pin the consuming module to an older version of *yours* that still allows AzureRM 4.5.0 — or push back on whether the higher minimum is genuinely required.

This is why bumping the floor is a breaking change. There is no provider-level "support both 4.5 and 4.10 with feature flags" — provider versions are workspace-wide, so any constraint you publish becomes the floor for every consumer.

## CHANGELOG for Provider Bumps

A provider version bump is a **breaking change**. It must be documented in `CHANGELOG.md` with the reason — which property or feature requires it.

```markdown
## [1.5.0] - 2026-01-09

### Changed

- **Breaking:** Update minimum required AzureRM provider version to `4.31.0`, required by the `enabled_metric` property on `azurerm_monitor_diagnostic_setting`.
- **Breaking:** Update Terraform required version to `>= 1.3.0` for `optional()` attribute support in `variables.tf`.
- Replace deprecated `metric` block with `enabled_metric` for `azurerm_monitor_diagnostic_setting`.
```

Rules:

1. Mark the line with `**Breaking:**`.
2. Name the *specific* property or feature that forced the bump.
3. Put breaking changes at the top of the entry.
4. The module version bump is at minimum a **minor** release — never just a patch.

```markdown
# Bad — no reason, patch version used for a breaking change.
## [1.4.3] - 2026-01-09

### Changed

- Change provider version to 4.31.0
```

## Common Mistakes

1. **File named `terraform.tf` instead of `versions.tf`.** Use `versions.tf`. Reviewers expect to find provider versions there.
2. **Static (exact) version pin in a module.** `version = "4.9.0"` cannot compose with any sibling module that pins anything else.
3. **Using `latest` or omitting `version` entirely.** Both produce non-reproducible behavior; `latest` is not even a valid Terraform constraint.
4. **A `provider "azurerm" { ... }` block inside a reusable module.** Breaks reusability across subscriptions and tenants. Allowed only via `configuration_aliases`.
5. **Listing providers the module doesn't use.** Forces consumers to download providers for no reason and clutters dependency graphs.
6. **Listing test-only providers in the module's `versions.tf`.** They belong in the test plan's `versions.tf`.
7. **Bumping the minimum version without a documented feature reason.** Forces consumer upgrades for no gain.
8. **Hardcoding a version inside `examples/` open-endedly OR inside a module exact-pinned.** It's the inverse for each context — exact in examples, floor in modules.
9. **Provider bump landed as a patch release with no CHANGELOG note.** A floor change is breaking; treat it that way.
10. **Empty `provider "azurerm" { features {} }` in a test plan.** `features {}` is default — delete the block.

## Review Checklist

- [ ] File is named `versions.tf` (not `terraform.tf`, not `providers.tf`).
- [ ] `terraform { required_version = ... }` is set with a `>=` floor.
- [ ] `required_providers` lists every provider the module uses — and *only* those.
- [ ] Every provider entry has both `source` and `version`.
- [ ] AzureRM (and any primary provider) uses a `>=` floor, not an exact pin.
- [ ] No `provider "<name>" { ... }` configuration blocks in the module (apart from `configuration_aliases` declarations).
- [ ] Multi-provider modules use `configuration_aliases` and the consumer passes providers via `providers = { ... }`.
- [ ] AzAPI, if used, is `>= 2.0.1`.
- [ ] Examples pin exact versions; test plans use `>=` floors.
- [ ] Minimum versions match the earliest release that supports the features actually used.
- [ ] Empty default-only `provider {}` blocks have been removed from test plans.
- [ ] Test-only providers are not declared in the module's `versions.tf`.
- [ ] Any provider/Terraform floor change is in `CHANGELOG.md` with `**Breaking:**` and a stated reason.
- [ ] The module version bump for a floor change is at least a minor release.
- [ ] No use of `latest` or implicit/missing version constraints anywhere.
