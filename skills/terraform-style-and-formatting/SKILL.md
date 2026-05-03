---
name: terraform-style-and-formatting
description: Apply and review HashiCorp's Terraform style guide as it applies to module code — indentation (2 spaces, no tabs), `=` alignment in blocks, comment style (`#` only), blank-line conventions, attribute ordering inside `resource`/`module`/`variable`/`output` blocks, file-end newline rules, and module-specific formatting anti-patterns. Use when asked to "format my Terraform", "style this HCL", "review this for style", "fix Terraform formatting", "align these attributes", "order these blocks", or before running `terraform fmt`. Different from `terraform-style-guide` (generation): this skill targets *applying* style to existing module code and catching anti-patterns that `terraform fmt` does not flag.
---

# Terraform Style and Formatting

Apply HashiCorp's Terraform style conventions to module code so every file looks the same regardless of who wrote it. Style consistency is not aesthetic — it directly reduces diff noise on PRs, removes friction in code review, and lets `terraform fmt` CI gates pass on the first push instead of bouncing back.

`terraform fmt` handles indentation and `=` alignment automatically. This skill covers the rules `fmt` does *not* enforce: attribute ordering inside blocks, blank-line conventions, comment style, single-element collections, and module-specific anti-patterns that show up only in review.

For naming conventions on identifiers (resources, variables, outputs), see `terraform-resource-naming`. For per-file content rules, see `terraform-variable-design`, `terraform-locals-patterns`, and `terraform-module-outputs`.

## Contents
- [Run `terraform fmt` First](#run-terraform-fmt-first)
- [Indentation and Alignment](#indentation-and-alignment)
- [Blank Lines](#blank-lines)
- [Comments](#comments)
- [Attribute Ordering in `resource` and `data` Blocks](#attribute-ordering-in-resource-and-data-blocks)
- [Attribute Ordering in `module` Blocks](#attribute-ordering-in-module-blocks)
- [Attribute Ordering in `variable` Blocks](#attribute-ordering-in-variable-blocks)
- [Attribute Ordering in `output` Blocks](#attribute-ordering-in-output-blocks)
- [Single-Element Collections](#single-element-collections)
- [Lifecycle Block Formatting](#lifecycle-block-formatting)
- [Consistent Locals Formatting](#consistent-locals-formatting)
- [File-End Conventions](#file-end-conventions)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)

## Run `terraform fmt` First

Before reviewing style by hand, run:

```bash
terraform fmt -recursive
```

This fixes indentation, `=` alignment, and most whitespace issues automatically. CI almost always runs `terraform fmt -check` and fails the build if output differs — running `fmt` locally before pushing keeps the feedback loop tight.

The rest of this skill covers what `fmt` *cannot* fix on its own: ordering, blank-line semantics, comment style, and structural anti-patterns.

## Indentation and Alignment

- **Two spaces** per nesting level. Never tabs.
- Align `=` for consecutive single-line arguments at the same nesting level.

```hcl
# Good
resource "azurerm_storage_account" "main" {
  resource_group_name = var.resource_group_name
  location            = local.location
  account_tier        = "Standard"
}

# Bad — unaligned, harder to scan
resource "azurerm_storage_account" "main" {
  resource_group_name = var.resource_group_name
  location = local.location
  account_tier = "Standard"
}
```

Alignment groups break across blank lines. If you want a fresh alignment column, separate the groups with a blank line — `terraform fmt` aligns each contiguous group independently.

## Blank Lines

| Context                                            | Rule                                                       |
|----------------------------------------------------|------------------------------------------------------------|
| Between top-level blocks                           | One blank line                                             |
| Between arguments and the first nested block       | One blank line                                             |
| Between nested blocks                              | One blank line (except when grouping same-type blocks)     |
| Multiple consecutive blank lines                   | Never                                                      |
| Trailing whitespace at end of line                 | Never                                                      |
| Last line of every file                            | Single newline (no double-blank, no missing newline)       |

Use blank lines to separate **logical groups** of arguments inside a block — for example, identity attributes vs. networking attributes.

```hcl
# Good — arguments precede nested blocks, separated by one blank line
resource "azurerm_some_resource" "main" {
  resource_group_name = var.resource_group_name
  location            = local.location

  identity {
    type = "SystemAssigned"
  }

  lifecycle {
    ignore_changes = [tags]
  }
}
```

Avoid interleaving blocks of different types unless they form a semantic family (e.g. multiple `dynamic "ip_configuration"` blocks belong together).

## Comments

- **Single-line:** `#` only. Never `//`.
- **Multi-line:** stack `#` comments. Never `/* */` — Terraform's parser accepts it, but the style guide forbids it and reviewers will flag it.
- Comments explain **why**, not **what**. The code already shows what.

```hcl
# Good — explains the why
# Kept count = 1 for backward compatibility — switching to a bare resource
# would force destroy/recreate for existing consumers.
resource "azurerm_private_endpoint" "main" {
  count = 1
  # ...
}

# Bad — // comment, plus restating what the code already says
// Set the location
location = var.location
```

Avoid commented-out code in committed modules. If something is unused, delete it; git history retains the previous version.

## Attribute Ordering in `resource` and `data` Blocks

Inside a `resource` or `data` block, follow this order — this matches AVM TFNFR8 and HashiCorp's own examples:

1. **Meta-arguments first:** `provider`, `count`, `for_each`
2. **Required arguments**, then **optional arguments**
3. **Nested configuration blocks** (e.g. `identity`, `network_rules`)
4. **Meta-argument blocks last:** `depends_on`, `lifecycle`

Separate each group with one blank line.

```hcl
# Good
resource "azurerm_some_resource" "main" {
  for_each = { for hook in var.webhooks : hook.name => hook }

  resource_group_name = var.resource_group_name
  location            = local.location

  identity {
    type = "SystemAssigned"
  }

  lifecycle {
    ignore_changes = [tags]
  }
}

# Bad — for_each in the middle, lifecycle before identity
resource "azurerm_some_resource" "main" {
  resource_group_name = var.resource_group_name
  for_each            = local.webhooks
  location            = local.location

  lifecycle {
    ignore_changes = [tags]
  }

  identity {
    type = "SystemAssigned"
  }
}
```

This order matters because reviewers scan top-down: meta-args first tell them how many instances exist; required args show what is being created; nested blocks add detail; lifecycle/depends_on at the bottom answer "how does this resource interact with state changes."

## Attribute Ordering in `module` Blocks

Inside a `module` block (AVM TFNFR9):

1. `source`, `version` (always first, in that order)
2. `count` / `for_each`
3. Required arguments (alphabetical)
4. Optional arguments (alphabetical)
5. `depends_on`, `providers` (always last)

```hcl
# Good
module "storage_accounts" {
  source  = "registry.example.com/acme/storage-account/azurerm"
  version = "1.2.3"

  for_each = local.storage_accounts

  brand               = var.brand
  environment         = var.environment
  location            = var.location
  project_name        = var.project_name
  resource_group_name = lookup(each.value, "resource_group_name", var.resource_group_name)
}
```

Alphabetical ordering of required and optional arguments removes the "where do I put this new arg" debate at review time.

## Attribute Ordering in `variable` Blocks

Every `variable` block must use this internal order:

1. `description`
2. `type`
3. `default` (if applicable)
4. `validation` (if applicable)

Separate each variable declaration with one blank line.

```hcl
# Good
variable "environment" {
  description = "The deployment environment name."
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "stg", "prd"], var.environment)
    error_message = "Must be dev, stg, or prd."
  }
}

variable "location" {
  description = "The Azure region for resource deployment."
  type        = string
}

# Bad — wrong order, missing blank line between declarations
variable "environment" {
  type        = string
  default     = "dev"
  description = "The deployment environment name."
}
variable "location" {
  type        = string
  description = "The Azure region."
}
```

## Attribute Ordering in `output` Blocks

Every `output` block must use this internal order:

1. `description`
2. `value`
3. `sensitive` (if applicable)

```hcl
# Good
output "resource_id" {
  description = "The ID of the provisioned resource."
  value       = azurerm_storage_account.main.id
}

output "primary_access_key" {
  description = "The primary access key for the storage account."
  value       = azurerm_storage_account.main.primary_access_key
  sensitive   = true
}

# Bad — value before description
output "resource_id" {
  value       = azurerm_storage_account.main.id
  description = "The ID of the provisioned resource."
}
```

## Single-Element Collections

If a list or map has exactly one element, write it on one line. `terraform fmt` does not collapse multi-line single-element collections — this is a manual rule.

```hcl
# Bad — vertical noise for a single value
metric = [
  "AllMetrics"
]

variable_map = {
  "key" = "value"
}

# Good
metric       = ["AllMetrics"]
variable_map = { "key" = "value" }
```

The rule reverses for two or more elements: prefer multi-line so that adding a third element is a one-line diff instead of restructuring the whole expression.

## Lifecycle Block Formatting

Inside `lifecycle`, the `ignore_changes` attribute references must **not** be quoted — they are HCL identifiers, not strings (AVM TFNFR10).

```hcl
# Good
lifecycle {
  ignore_changes = [tags, labels]
}

# Bad — strings instead of identifiers
lifecycle {
  ignore_changes = ["tags", "labels"]
}
```

The quoted form silently does nothing because Terraform compares string literals against attribute paths and never matches.

## Consistent Locals Formatting

Within a single `locals` block, use one consistent format for all assignments. If one value spans multiple lines, either compress it to one line or expand the others — do not mix styles.

```hcl
# Good — all single-line
locals {
  location_code  = local.location_code_map[var.location]
  primary_name   = format("%s%s%s%s", var.brand, var.environment, var.project_name, local.location_code)
  module_version = trimspace(file("${path.module}/VERSION"))
}

# Bad — inconsistent mix of styles
locals {
  location_code = local.location_code_map[var.location]
  primary_name = format(
    "%s%s%s%s",
    var.brand,
    var.environment,
    var.project_name,
    local.location_code,
  )
  module_version = trimspace(file("${path.module}/VERSION"))
}
```

If a single expression genuinely needs to wrap (long `format()`, multi-line `merge()`), pull it into its own local with a clarifying name and keep the main `locals` block tidy.

## File-End Conventions

- Every `.tf` and `.md` file ends with **one** newline character — no double-blank, no missing newline.
- No trailing whitespace on any line.

These are easy to enforce automatically. A pre-commit config that includes `end-of-file-fixer` and `trailing-whitespace` hooks catches both:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: end-of-file-fixer
      - id: trailing-whitespace

  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.92.0
    hooks:
      - id: terraform_fmt
      - id: terraform_tflint
```

CI should run `terraform fmt -check -recursive` and fail the build if formatting drifts. Catching style violations on the developer's machine via pre-commit is dramatically cheaper than catching them in CI after a PR is open.

## Module-Specific Anti-Patterns

A few formatting issues show up specifically in module code that consumers will copy or extend. Catch these in review even if `terraform fmt` doesn't:

- **Variable descriptions without a trailing period.** Module READMEs are auto-generated from descriptions — inconsistent punctuation reads as sloppy in the published docs.
- **Mixing single-line and heredoc descriptions** for similar variables in the same file. Pick one style per shape: simple scalars use one-line strings; complex `object`/`map` types use `<<-EOT` heredocs with a fenced type block.
- **Re-aligning `=` across blank lines in the same block.** Each blank-line-separated group gets its own alignment column; do not pad with extra spaces trying to align across the whole block.
- **Quoted `ignore_changes` entries** (covered above) — silently broken, looks fine to a casual reader.
- **Commented-out resources or attributes** left in the module. Modules are reused as-is; consumers should not have to interpret which commented sections are "the right way."
- **Single-element lists in `for_each` / dynamic blocks** that should be a guarded scalar — produces extra noise in plan output and unnecessarily indents the call site.

## Common Mistakes

1. **Tabs instead of spaces** — paste from another editor or AI output. `terraform fmt` fixes it; commit hooks should enforce it.
2. **`//` for single-line comments** — works, but violates the style guide. Use `#`.
3. **`/* */` for multi-line comments** — same: parser accepts it, style forbids it.
4. **`for_each` or `count` placed in the middle of a `resource` block** instead of the first line.
5. **`lifecycle` block placed before nested config blocks** — it must be the last block, after `identity`, `network_rules`, etc.
6. **Wrong `variable` attribute order** — `type` before `description`, or `validation` before `default`.
7. **Wrong `output` attribute order** — `value` before `description`.
8. **Multi-line single-element collections** — `metric = ["AllMetrics"]` collapsed to three lines.
9. **Quoted `ignore_changes` identifiers** — silent no-op.
10. **Missing or doubled trailing newline** — caught by `end-of-file-fixer`.
11. **Trailing whitespace** — caught by `trailing-whitespace`.
12. **Variable descriptions missing a trailing period** — appears in generated `README.md`.
13. **Inconsistent `locals` formatting** — some single-line, some wrapped, in the same block.

## Review Checklist

- [ ] `terraform fmt -recursive` produces no diff.
- [ ] Indentation is 2 spaces, no tabs.
- [ ] `=` is aligned within each blank-line-separated group of arguments.
- [ ] `resource`/`data` blocks order: meta-args, args, nested blocks, `lifecycle`/`depends_on`.
- [ ] `module` blocks order: `source`, `version`, `count`/`for_each`, required args, optional args, `depends_on`/`providers`.
- [ ] `variable` blocks order: `description`, `type`, `default`, `validation`.
- [ ] `output` blocks order: `description`, `value`, `sensitive`.
- [ ] One blank line between top-level blocks; never two or more.
- [ ] Comments use `#` only — no `//`, no `/* */`.
- [ ] Single-element lists and maps are written on one line.
- [ ] `lifecycle.ignore_changes` entries are bare identifiers, not quoted strings.
- [ ] No trailing whitespace; every file ends with exactly one newline.
- [ ] `locals` block uses a consistent formatting style throughout.
- [ ] No commented-out code committed to the module.
- [ ] Variable descriptions end with a period.
