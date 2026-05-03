# Native `terraform test`

`terraform test` is Terraform's built-in test framework, introduced as GA in Terraform 1.6 and significantly extended with `mock_provider` in 1.7. Test files live alongside the module (typically in `tests/`) as `*.tftest.hcl` files. The framework runs `run` blocks that execute `terraform plan` or `terraform apply`, then evaluates `assert` blocks against the resulting state and outputs.

It sits in the middle of the testing pyramid: cheaper than Terratest because it has no Go dependency, supports plan-only runs in seconds, and supports mocked providers for fully offline unit tests. For static analysis, see `static-analysis.md`. For Terratest (when you need to call Azure beyond Terraform's view), see `terratest.md`.

## When to Use `terraform test`

- **Default choice for module-shape and contract tests.** Variable validation, output shape, conditional resource creation — all of it is faster, simpler, and doesn't require Go.
- **Mocked unit tests.** With `mock_provider` (1.7+) you can assert module behavior without ever calling Azure.
- **Plan-time checks.** `command = plan` runs in seconds and catches most contract bugs.
- **Apply-based integration.** `command = apply` provisions real resources; use sparingly, prefer mocked or plan-only when possible.

Use Terratest instead when you need to interact with Azure beyond what Terraform sees (HTTP probes, polling Azure Monitor, REST API checks).

## File Layout

```
my-module/
|-- main.tf
|-- variables.tf
|-- outputs.tf
|-- tests/
    |-- defaults.tftest.hcl
    |-- naming.tftest.hcl
    |-- validation.tftest.hcl
    |-- integration.tftest.hcl
```

One file per scenario keeps blast radius small and lets `-filter` target specific cases.

## File Format

A `*.tftest.hcl` file contains one or more `run` blocks. Each `run` is a single `plan` or `apply` invocation. Optional top-level blocks: `variables` (defaults across all runs), `provider`, `mock_provider`.

```hcl
# tests/defaults.tftest.hcl

variables {
  resource_group_name = "test-rg"
  location            = "eastus"
}

run "default_name" {
  command = plan

  assert {
    condition     = output.account_name != null
    error_message = "account_name should not be null"
  }

  assert {
    condition     = startswith(output.account_name, "acme-")
    error_message = "account_name should start with the acme- prefix"
  }
}

run "name_override" {
  command = plan

  variables {
    name_override = "my-custom-account"
  }

  assert {
    condition     = output.account_name == "my-custom-account"
    error_message = "name_override should be honoured exactly"
  }
}
```

## `command = plan` vs `command = apply`

- **`plan`** (default) runs `terraform plan` and evaluates assertions against the planned state. No real resources are created. Fast (seconds). Use this for the vast majority of tests.
- **`apply`** runs `terraform apply` against the real provider, creating real Azure resources. Resources are destroyed when the test finishes (regardless of pass/fail). Slow (minutes). Use only for integration scenarios that require post-apply attributes the planner cannot know.

```hcl
run "plan_only" {
  command = plan
  # ...
}

run "real_deploy" {
  command = apply
  # ...
}
```

Inside a single `*.tftest.hcl` file, runs execute in declaration order and share state. A `plan` run can follow an `apply` run and assert on the resources the apply created.

## `assert` Blocks

Every `assert` has a `condition` (boolean expression) and an `error_message` (shown when the condition is false). Conditions can reference `var.*`, `output.*`, and any resource address inside the configuration under test.

```hcl
assert {
  condition     = length(output.subnet_ids) == 3
  error_message = "expected 3 subnets, got ${length(output.subnet_ids)}"
}

assert {
  condition     = can(regex("^/subscriptions/", output.account_id))
  error_message = "account_id should be a full ARM ID"
}

assert {
  condition     = output.diagnostic_setting_id != null
  error_message = "diagnostic_setting_id should be set when enable_diagnostics = true"
}
```

Common condition patterns:

- `output.x != null` — output exists.
- `length(output.x) > 0` — non-empty list/map.
- `can(regex("...", output.x))` — output matches a pattern.
- `contains(output.x, "expected")` — list contains element.
- `output.x == var.expected` — exact match (rare; usually you assert shape, not value).

## Negative Tests with `expect_failures`

When you want to assert that bad input is rejected — variable validation block fires, a `precondition` triggers — use `expect_failures`:

```hcl
run "rejects_invalid_sku" {
  command = plan

  variables {
    sku = "garbage-not-a-real-sku"
  }

  expect_failures = [
    var.sku,
  ]
}
```

`expect_failures` takes a list of references — variables, resources, or checks — that you expect to fail. The run passes if and only if every listed reference fails.

This is how you prove that:

- Variable `validation { condition = ... }` blocks fire on bad input.
- `precondition` and `postcondition` checks catch invalid combinations.
- `ConflictsWith` pairs are actually wired up.

```hcl
# In variables.tf
variable "sku" {
  type = string
  validation {
    condition     = contains(["Basic", "Standard", "Premium"], var.sku)
    error_message = "sku must be Basic, Standard, or Premium."
  }
}
```

```hcl
# In tests/validation.tftest.hcl
run "valid_sku" {
  command = plan
  variables { sku = "Standard" }
}

run "invalid_sku_rejected" {
  command = plan
  variables { sku = "Free" }
  expect_failures = [var.sku]
}
```

Without negative tests, validation blocks rot silently — somebody refactors `condition`, the validation no longer fires, and nobody notices.

## `mock_provider` (Terraform 1.7+)

Mocks the provider so apply runs without contacting Azure at all. Every resource appears to "create" but no real API calls happen, and you can override individual computed attributes for assertion purposes.

```hcl
mock_provider "azurerm" {
  mock_resource "azurerm_resource_group" {
    defaults = {
      id       = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/mock-rg"
      location = "eastus"
    }
  }

  mock_resource "azurerm_storage_account" {
    defaults = {
      id                         = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/mock-rg/providers/Microsoft.Storage/storageAccounts/mockstorage"
      primary_blob_endpoint      = "https://mockstorage.blob.core.windows.net/"
      primary_access_key         = "mock-key"
    }
  }
}

run "storage_account_id_propagates_to_output" {
  command = apply

  assert {
    condition     = output.storage_account_id == "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/mock-rg/providers/Microsoft.Storage/storageAccounts/mockstorage"
    error_message = "module did not surface the storage account id"
  }
}
```

### Override per-run

You can override mocks inside a specific `run` to test how the module behaves when an attribute returns a particular value:

```hcl
run "handles_custom_endpoint" {
  command = apply

  override_resource {
    target = azurerm_storage_account.main
    values = {
      primary_blob_endpoint = "https://custom.blob.core.windows.net/"
    }
  }

  assert {
    condition     = output.primary_endpoint == "https://custom.blob.core.windows.net/"
    error_message = "module did not pass the primary_blob_endpoint through"
  }
}
```

### When to use mocked vs real provider

- **Mocked:** module-shape tests, conditional resource creation, output computation, variable validation. No real provider calls — runs in seconds, costs nothing, deterministic.
- **Real provider (`command = apply` without mocks):** when you need attributes only Azure can compute (e.g., assigned identities, generated certificate thumbprints) AND you cannot fake them. Most tests do not need this.

`mock_provider` is the right default for the unit-test layer. Reach for the real provider only when an actual API round-trip is the thing under test.

## `setup` Runs and Sequencing

Multiple runs in one file execute in declaration order and share state. To stage prerequisite infrastructure before the module under test, use a `setup` run pointing at a different module path:

```hcl
run "setup_prereqs" {
  command = apply
  module {
    source = "./tests/setup"
  }
}

run "module_under_test" {
  command = apply

  variables {
    # Reference the previous run's outputs
    resource_group_name = run.setup_prereqs.resource_group_name
    location            = run.setup_prereqs.location
  }

  assert {
    condition     = output.account_id != null
    error_message = "account_id should be set after apply"
  }
}
```

`run.<previous_run>.<output>` reads outputs from earlier runs in the same file — the native equivalent of Terratest's staged-apply pattern.

When the file finishes, all runs are destroyed in reverse order automatically. No `defer` boilerplate.

## Filtering and Running

```bash
# Run all *.tftest.hcl files in tests/
terraform test

# Run a specific file
terraform test -filter=tests/defaults.tftest.hcl

# Verbose output (show plan details on assertion failure)
terraform test -verbose

# JSON for CI ingestion
terraform test -json
```

`terraform init` is implicit — `terraform test` initializes the configuration before running.

### Specifying variables

`-var` and `-var-file` work the same as `terraform plan`. Variables in the test file's `variables` block override the configuration's defaults; `-var` on the command line overrides those.

## CI Integration

Cheap. Runs on every PR. No Azure credentials needed for mocked tests; real-provider runs need the same SPN as Terratest.

### GitHub Actions — mocked tests on every PR

```yaml
name: terraform-test
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.8
      - run: terraform init -backend=false
      - run: terraform test -verbose
```

### Real-provider runs (gated)

```yaml
name: terraform-test-apply
on:
  pull_request:
    types: [labeled]
  schedule:
    - cron: "0 4 * * *"

jobs:
  apply:
    if: github.event.label.name == 'run-integration' || github.event_name == 'schedule'
    runs-on: ubuntu-latest
    timeout-minutes: 60
    env:
      ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      ARM_CLIENT_ID:       ${{ secrets.AZURE_CLIENT_ID }}
      ARM_CLIENT_SECRET:   ${{ secrets.AZURE_CLIENT_SECRET }}
      ARM_TENANT_ID:       ${{ secrets.AZURE_TENANT_ID }}
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform test -filter=tests/integration.tftest.hcl
```

Note the env-var names: `terraform test` uses `ARM_*` (the azurerm provider's native variable names), not the `AZURE_*` names Terratest reads from variables.

## Common Patterns

### Variable validation

```hcl
run "valid_sku" {
  command = plan
  variables { sku = "Standard_LRS" }
}

run "invalid_sku" {
  command = plan
  variables { sku = "garbage" }
  expect_failures = [var.sku]
}
```

### Output shape

```hcl
run "outputs_have_arm_ids" {
  command = plan

  assert {
    condition     = can(regex("^/subscriptions/[^/]+/resourceGroups/", output.account_id))
    error_message = "account_id is not a full ARM ID"
  }
}
```

### Conditional resource creation

```hcl
run "diagnostics_disabled_no_setting" {
  command = plan
  variables { enable_diagnostics = false }

  assert {
    condition     = output.diagnostic_setting_id == null
    error_message = "diagnostic_setting_id should be null when enable_diagnostics = false"
  }
}

run "diagnostics_enabled_setting_present" {
  command = plan
  variables { enable_diagnostics = true }

  assert {
    condition     = output.diagnostic_setting_id != null
    error_message = "diagnostic_setting_id should be set when enable_diagnostics = true"
  }
}
```

### `for_each` shape

```hcl
run "for_each_outputs_keyed_correctly" {
  command = plan

  variables {
    accounts = {
      "default"  = { name_override = null }
      "override" = { name_override = "custom-name" }
    }
  }

  assert {
    condition     = length(keys(output.account_id)) == 2
    error_message = "expected 2 keys in account_id map"
  }

  assert {
    condition     = contains(keys(output.account_id), "override")
    error_message = "override key missing from account_id map"
  }
}
```

## Common Mistakes

1. **Using `command = apply` for everything.** Default to `command = plan`; only escalate to apply when you need post-apply attributes.
2. **Hardcoding expected resource names.** Same anti-pattern as Terratest — assert on shape (regex, non-null), not exact strings.
3. **No negative tests.** A test suite without `expect_failures` cannot tell you that variable validation actually fires.
4. **Mocking the entire module away.** If every assertion just checks the mock value back, the test proves nothing about the module. Mock provider attributes; let the module compute outputs from them.
5. **One giant `*.tftest.hcl` with 30 runs.** One file per scenario; keeps `-filter` useful.
6. **`run "name with spaces"`.** Run labels become test names; pick `snake_case` identifiers so `-filter` is straightforward.
7. **No assertions in a `setup` run.** Setup runs should still assert that the prereqs they create are valid; otherwise a broken setup silently breaks every dependent run.
8. **Forgetting `expect_failures` is exclusive.** The run *only* passes if every listed reference fails. Adding it without a real failure path makes the test always fail.

## Review Checklist

- [ ] At least one `*.tftest.hcl` file exists in `tests/`.
- [ ] At least one `run "..." { command = plan }` block runs on every PR.
- [ ] Negative tests with `expect_failures` cover every variable `validation` block.
- [ ] Negative tests cover at least one `precondition` / `postcondition` if the module has any.
- [ ] Mocked tests use `mock_provider "azurerm"` (Terraform 1.7+) and override only the attributes the assertion needs.
- [ ] Apply-based runs are gated (label or schedule), not on every PR.
- [ ] Assertions check shape (`!= null`, regex, length) rather than hardcoded strings.
- [ ] One file per scenario; long files are split.
- [ ] CI runs `terraform test` after `terraform init -backend=false` for mocked runs, full `terraform init` for real-provider runs.
