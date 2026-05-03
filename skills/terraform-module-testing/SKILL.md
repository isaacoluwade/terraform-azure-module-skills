---
name: terraform-module-testing
description: Test Azure Terraform modules across the full pyramid — static analysis at the bottom, native `terraform test` in the middle, Terratest at the top. Use this skill when deciding which testing layer to reach for, when writing or reviewing tests at any layer, when wiring tests into CI, or when designing the test strategy for a new module. Covers static analysis (`terraform fmt`, `terraform validate`, `tflint`, `checkov`, `tfsec`/`trivy`), native `terraform test` (Terraform 1.6+, both `command = plan` and `command = apply`), `mock_provider` for unit-style assertions (Terraform 1.7+), Terratest for real Azure integration, and negative tests at every layer. Triggers on: "test this Terraform module", "what testing layers", "module test pyramid", "tflint config", "checkov", "write a terraform test", "mock_provider", "Terratest", "module integration test", "test this in CI", "negative tests for variable validation". Always start with this skill before writing any test — picking the wrong layer wastes the agent's and the user's time.
---

# Terraform Module Testing

Every Terraform module needs more than one kind of test. Static analysis catches typos and security misconfigurations in milliseconds. Plan-time tests catch contract bugs in seconds. Mocked unit tests prove module behavior without ever calling Azure. Integration tests prove the module produces real working infrastructure. Negative tests prove that bad input is rejected. Skip the cheap layers and you end up running 30-minute integration tests to catch a missing comma; skip the integration layer and you ship a module that has never actually deployed anything.

This skill teaches the agent to reach for the **cheapest layer that can answer the question**, then escalate only when the cheaper layers can't. The detail for each layer lives in the `references/` files; this page is the map.

For module structure, see `terraform-module-structure`. For output design (which determines what tests can assert against), see `terraform-module-outputs`. For documentation and CI wiring beyond what's covered here, see `terraform-module-documentation`.

## Contents
- [The Testing Pyramid](#the-testing-pyramid)
- [What Each Layer Catches](#what-each-layer-catches)
- [Test Layout in a Module](#test-layout-in-a-module)
- [Static Analysis](#static-analysis)
- [Plan-Time Tests with `terraform test`](#plan-time-tests-with-terraform-test)
- [Mocked Unit Tests with `mock_provider`](#mocked-unit-tests-with-mock_provider)
- [Integration Tests](#integration-tests)
- [Negative Tests](#negative-tests)
- [Picking the Right Layer (Worked Examples)](#picking-the-right-layer-worked-examples)
- [Tooling Decision Rules](#tooling-decision-rules)
- [CI Strategy](#ci-strategy)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## The Testing Pyramid

Order layers from cheapest/fastest (run on every save) to most expensive (run on a schedule or pre-release). Each layer should catch its own class of bug; layers above it should not have to.

```
                  +------------------------+
                  | Integration (minutes)  |   real Azure, real apply
                  +------------------------+
                +----------------------------+
                | Mocked unit (seconds)      | mock_provider, no Azure
                +----------------------------+
              +--------------------------------+
              | Plan-time (seconds)            | terraform test, plan
              +--------------------------------+
            +------------------------------------+
            | Static analysis (milliseconds)     | fmt, validate, tflint,
            +------------------------------------+   checkov, tfsec, trivy
```

| Layer | Speed | Cost | Runs on |
|---|---|---|---|
| Static analysis | ms | free | every save, every PR |
| Plan-time `terraform test` | seconds | free | every save, every PR |
| Mocked unit (`mock_provider`) | seconds | free | every PR |
| Integration (apply) | minutes per scenario | Azure $$ | label, schedule, pre-release |
| Negative tests | varies (any layer) | free or cheap | every PR |

**Rule of thumb:** if a question can be answered at a cheaper layer, answer it there. Don't run a 20-minute integration test to find out your variable type is wrong — `terraform validate` does that in 200ms.

## What Each Layer Catches

| Layer | Catches |
|---|---|
| Static | Syntax errors, formatting drift, security misconfigurations, deprecated patterns, lint violations, missing `required_version`, typo'd region names |
| Plan-time | Contract bugs (variable types, output shape, broken references), conditional resource logic without applying, `for_each` keys, expression errors |
| Mocked | Module behavior under controlled conditions without real provider calls — output computation, conditional creation, attribute propagation through the module |
| Integration | Actual Azure deployment, idempotency, output values from real resources, eventual-consistency behavior, post-apply attributes only Azure can compute |
| Negative | Variable `validation` blocks fire on bad input, `precondition`/`postcondition` checks catch invalid combinations, `ConflictsWith` pairs are wired up, error messages are informative |

## Test Layout in a Module

A module that uses every layer ends up with this shape:

```
my-module/
|-- main.tf
|-- variables.tf
|-- outputs.tf
|-- versions.tf
|-- VERSION
|-- .tflint.hcl                    # static layer config
|-- checkov.yml                    # static layer config
|-- .pre-commit-config.yaml        # local enforcement of static layer
|-- tests/                         # native terraform test
|   |-- defaults.tftest.hcl        #   plan-time, real provider, no apply
|   |-- naming.tftest.hcl          #   plan-time, real provider, no apply
|   |-- mocked.tftest.hcl          #   apply with mock_provider
|   |-- validation.tftest.hcl      #   negative tests with expect_failures
|   |-- integration.tftest.hcl     #   command = apply, gated in CI
|-- terratest/                     # Terratest, only if needed
    |-- plan/
    |   |-- main.tf
    |   |-- outputs.tf
    |   |-- variables.tf
    |   |-- versions.tf
    |-- test/
        |-- go.mod
        |-- go.sum
        |-- terraform_test.go
```

Not every module needs Terratest. If everything you want to assert is observable through Terraform state and outputs, stop at `tests/` — `terraform test` alone is the simpler and cheaper choice.

## Static Analysis

The cheapest layer. Runs in milliseconds, no state, no plan, no Azure calls. **Mandatory on every PR — these checks should never be skipped.**

The full suite:

```bash
# Formatting (gating CI step)
terraform fmt -check -recursive

# Schema and type validation (no backend, no credentials)
terraform init -backend=false
terraform validate

# HCL lint with provider-specific rules
tflint --init
tflint --recursive

# Security/compliance scanning
checkov -d . --config-file checkov.yml

# Aqua-family security scanning (use one or both)
trivy config . --severity HIGH,CRITICAL --exit-code 1
```

Decision rule: **`tflint` for HCL correctness and provider-specific lint** (typo'd region, deprecated argument, missing `required_version`); **`checkov` for security/compliance** (public storage, missing encryption). Run both — they catch different classes of issue.

For configuration files, suppression patterns, severity gating, pre-commit integration, and full GitHub Actions / Azure DevOps pipeline examples, see [`references/static-analysis.md`](references/static-analysis.md).

## Plan-Time Tests with `terraform test`

Native to Terraform (GA in 1.6+). Test files are `*.tftest.hcl` alongside the module. Each `run` block runs `terraform plan` (or `apply`) and evaluates `assert` blocks against state and outputs.

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

run "name_override_honoured" {
  command = plan
  variables { name_override = "my-custom-account" }

  assert {
    condition     = output.account_name == "my-custom-account"
    error_message = "name_override should be honoured exactly"
  }
}
```

Run with:

```bash
terraform test
terraform test -filter=tests/defaults.tftest.hcl
terraform test -verbose
```

Use `command = plan` by default — it's fast, free, and catches most contract bugs. Escalate to `command = apply` only when you genuinely need post-apply attributes the planner cannot know.

For the full `*.tftest.hcl` format, `setup` runs, dependent runs via `run.<previous>.<output>`, JSON output for CI, and patterns for variable validation / output shape / conditional resources / `for_each`, see [`references/terraform-test.md`](references/terraform-test.md).

## Mocked Unit Tests with `mock_provider`

Terraform 1.7+ adds `mock_provider`, which mocks the provider so apply runs without contacting Azure at all. Every resource appears to "create"; you can override individual computed attributes per assertion.

```hcl
mock_provider "azurerm" {
  mock_resource "azurerm_storage_account" {
    defaults = {
      id                    = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/mock/providers/Microsoft.Storage/storageAccounts/mockstorage"
      primary_blob_endpoint = "https://mockstorage.blob.core.windows.net/"
    }
  }
}

run "module_surfaces_endpoint" {
  command = apply

  assert {
    condition     = output.primary_endpoint == "https://mockstorage.blob.core.windows.net/"
    error_message = "module did not pass primary_blob_endpoint through"
  }
}
```

This is the right layer for proving "given X, the module produces Y" without paying Azure cents-per-minute. It runs in seconds and is fully deterministic.

**Don't mock the entire module away.** If every assertion just checks the mock value back, the test proves nothing about the module. Mock provider *attributes* and let the module *compute outputs from them*.

For per-run `override_resource`, when to mock vs use the real provider, and full examples, see [`references/terraform-test.md`](references/terraform-test.md).

## Integration Tests

The expensive layer: real Azure resources, real apply, real teardown. Two tools, used for different reasons.

### Native `terraform test` with `command = apply`

Simplest path. No Go, no extra binary. Use this when the module's apply-side behavior is fully observable through Terraform state and outputs.

```hcl
run "apply_real" {
  command = apply

  assert {
    condition     = can(regex("^/subscriptions/", output.account_id))
    error_message = "account_id should be a full ARM ID"
  }
}
```

### Terratest

Use when you need to interact with Azure beyond what Terraform sees — poll an HTTP endpoint, query Azure Monitor metrics, hit the ARM REST API directly, run shell commands around the apply, or you have an existing Terratest investment with shared helpers.

```go
func TestNetAppModule(t *testing.T) {
    t.Parallel()
    opts := &terraform.Options{TerraformDir: "../plan", NoColor: true}
    defer terraform.Destroy(t, opts)              // register destroy BEFORE apply
    terraform.InitAndApply(t, opts)
    accountID := terraform.OutputMap(t, opts, "account_id")
    assert.NotEmpty(t, accountID["default"])
}
```

**Critical rules at the integration layer (both tools):**
- **Assert on output shape, not hardcoded values.** A test that hardcodes `"acme-prod-eastus-netapp"` is testing the test, not the module. Read outputs and assert with `!= null`, regex, or substring.
- **Tag every test resource with the module `VERSION`** so leftover resources from failed runs are traceable.
- **Don't run integration on every PR** — gate it behind a label like `run-integration` or run nightly.

For the Terratest directory layout (`terratest/plan/`, `terratest/test/`), the apply/assert/destroy lifecycle, `for_each` patterns, staged applies, retries, polling for eventual consistency, Azure auth, and CI gating, see [`references/terratest.md`](references/terratest.md).

## Negative Tests

Negative tests prove that bad input is rejected. Without them, variable `validation` blocks rot silently — somebody refactors the condition, the validation no longer fires, and nobody notices.

Use `expect_failures` in `terraform test`:

```hcl
run "rejects_invalid_sku" {
  command = plan
  variables { sku = "garbage-not-a-real-sku" }
  expect_failures = [var.sku]
}
```

The run passes if and only if the listed reference fails. Cover at least:

- Every variable `validation { condition = ... }` block.
- Every `precondition` / `postcondition` the module declares.
- Every `ConflictsWith`-style pair (input combinations the module is supposed to reject).

Negative tests are nearly free — most run at plan time — and they're the only way to keep validation rules honest.

## Picking the Right Layer (Worked Examples)

When a question comes up, walk down the pyramid until you hit the cheapest layer that can answer it.

**"Did I name a variable wrong?"** — Static (`terraform validate`). Don't escalate.

**"Does my output return a non-null ARM ID?"** — Plan-time (`terraform test`, `command = plan`, `condition = can(regex("^/subscriptions/", output.x))`).

**"Does my module create the diagnostic setting only when `enable_diagnostics = true`?"** — Plan-time. Two `run` blocks, one with each toggle, asserting `output.diagnostic_setting_id` is null vs not-null.

**"Does my module reject `sku = 'garbage'`?"** — Plan-time with `expect_failures = [var.sku]`.

**"Given a particular Azure-computed value, does my module surface it correctly?"** — Mocked. `mock_provider "azurerm"` with the value as a default; assert the module's output reflects it.

**"Does the module actually deploy to Azure and produce a working storage account?"** — Integration. `terraform test` with `command = apply`, or Terratest.

**"Does the deployed Function App actually respond to HTTPS at its hostname?"** — Terratest. Terraform itself does not poll; you need Go's `http.Get` and `retry.DoWithRetry`.

**"Does the deployed module emit metrics to Azure Monitor?"** — Terratest. Outside Terraform's view; needs an SDK call.

The pattern: prefer the cheapest layer that can answer the question. Each escalation costs the team time and money — make sure it's necessary.

## Tooling Decision Rules

**`terraform test` vs Terratest** — Default to native `terraform test`. It's simpler, has no Go dependency, runs faster, and the assertion language is more direct. Reach for Terratest when:

- You need to call Azure beyond what Terraform sees (HTTP probes, REST API checks, Azure Monitor queries).
- You need to chain shell commands or external tools around the apply.
- You already have a shared Terratest helper library and CI wiring.

**`mock_provider` vs real provider** — Default to `mock_provider` for unit-style assertions about module behavior. Use the real provider only when an actual API round-trip is the thing under test, or when you need post-apply computed attributes that mocks can't usefully fake (assigned identities, certificate thumbprints).

**`tflint` vs `checkov`** — Run both. `tflint` is for HCL correctness and provider-specific lint rules (typo'd region, deprecated argument, naming conventions, missing `required_version`). `checkov` is for security and compliance (public storage account, missing encryption at rest, public network access). They overlap in maybe 5% of findings; the rest is unique to each.

**`checkov` vs `tfsec`/`trivy`** — Significant overlap, different rule coverage. `checkov` has the broadest Azure rule set; `trivy config` (which now includes `tfsec`) is faster. Running both catches a few extra findings; running just `checkov` is acceptable.

## CI Strategy

The pyramid maps directly to CI gating:

| Job | When | Layers | Time budget |
|---|---|---|---|
| `static` | Every PR | fmt, validate, tflint, checkov, trivy | < 1 min |
| `terraform-test` | Every PR | `terraform test` (plan + mocked) | < 5 min |
| `terraform-test-apply` | Label or schedule | `terraform test` (apply) | 30-60 min |
| `terratest` | Label or schedule | Terratest | 30-90+ min |

**Rules:**

1. Static + plan-time + mocked are mandatory PR gates. They're cheap and they catch most defects.
2. Integration jobs are gated by a label (`run-integration`) or a nightly schedule. Don't burn Azure dollars on every PR.
3. Pre-release: run integration before tagging.
4. Always check exit codes — silent failures defeat the gate.
5. Set explicit timeouts on integration jobs (`go test -timeout 90m` or `timeout-minutes: 60` in GitHub Actions). Default 10-minute timeouts kill long applies and leak resources.

## Common Mistakes

1. **Only writing Terratest, skipping the cheap layers.** Static analysis would have caught the typo in 200ms; instead it failed a 30-minute apply.
2. **Running integration tests on every PR.** Slow, costs money, and rate-limits the team. Gate behind a label or schedule.
3. **Asserting against hardcoded values instead of module outputs.** A test that hardcodes `"acme-prod-eastus-netapp"` is testing the test, not the module. Assert on shape (regex, non-null, contains).
4. **Ignoring `terraform validate` exit code in CI.** A test that always passes is a test that always lies. Always check the exit code.
5. **No negative tests at all.** Variable `validation` blocks rot silently; without `expect_failures` you have no way to know.
6. **Mocking everything away so tests don't catch real provider behavior.** If `mock_provider` returns the value the assertion checks, the test proves nothing about the module. Mock attributes; let the module compute outputs.
7. **`defer Destroy` registered after apply (Terratest).** A panic in apply leaves resources orphaned. Defer destroy *before* `InitAndApply`.
8. **`auto.tfvars` files in test directories.** Splits inputs across files for no benefit. Use `locals` or variable defaults.
9. **`data` source lookups in test plans.** Adds latency and a new failure mode that has nothing to do with the module under test.
10. **Pinned (`= 3.74.0`) provider versions in test `versions.tf`.** Use `>=` so tests catch breakage with newer providers.
11. **Default 10-minute `go test` timeout on a 30-minute apply.** Set `-timeout 90m` for heavy modules; otherwise Go kills apply mid-flight and leaks resources.
12. **No `VERSION` tag on test resources.** When a leftover shows up next week, you cannot tell which module run created it.

## Review Checklist

- [ ] Static analysis runs in CI (`terraform fmt -check`, `terraform validate`, `tflint`, `checkov`).
- [ ] `terraform validate` exit code is checked explicitly.
- [ ] At least one `*.tftest.hcl` file exists with one or more `command = plan` `run` blocks.
- [ ] Mocked unit tests use `mock_provider` for module-shape assertions where applicable.
- [ ] Integration tests cover the happy path (`terraform test` with `command = apply`, or Terratest).
- [ ] Negative tests with `expect_failures` cover every variable `validation` block.
- [ ] Tests assert against module outputs (`!= null`, regex, length); no hardcoded resource IDs or names.
- [ ] `for_each` outputs are tested by iterating every key, not just the first.
- [ ] Integration jobs are gated by label or schedule, not every PR.
- [ ] Test resources are tagged with the module `VERSION`.
- [ ] `defer terraform.Destroy(...)` is registered before `InitAndApply` (Terratest).
- [ ] CI timeout is large enough for the slowest apply.
- [ ] Provider versions in test plans use `>=`, not pinned `=`.
- [ ] No `data` source lookups in test plans.
- [ ] Pre-commit config exists for the static layer.

## References

- [`references/static-analysis.md`](references/static-analysis.md) — `terraform fmt`, `terraform validate`, `tflint`, `checkov`, `tfsec`/`trivy`, `terrascan`. Configuration files, suppression patterns, severity gating, pre-commit integration, full CI examples for GitHub Actions and Azure DevOps.
- [`references/terraform-test.md`](references/terraform-test.md) — Native `terraform test` end to end. `*.tftest.hcl` format, `command = plan` and `command = apply`, `mock_provider` (1.7+), `expect_failures`, `setup` runs, dependent runs, filtering, JSON output, CI integration.
- [`references/terratest.md`](references/terratest.md) — Terratest for real Azure integration. Directory layout, `go.mod`/`go.sum`/`terraform_test.go`, the apply/assert/destroy lifecycle, `for_each` patterns, staged applies, retries and timeouts, polling for eventual consistency, Azure auth, CI gating, cleanup of failed runs.
