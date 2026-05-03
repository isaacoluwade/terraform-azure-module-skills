# Terratest for Azure Modules

Terratest is the heaviest layer of the testing pyramid: real Go code that runs `terraform apply` against a live Azure subscription, asserts on its outputs, and tears the resources down. Reach for it when native `terraform test` cannot answer the question — when you need to call Azure APIs directly, hit an HTTP endpoint that the module produced, poll Azure Monitor metrics, or chain shell commands around the apply.

For the static-analysis layer, see `static-analysis.md`. For native `terraform test` (cheaper, simpler, no Go required), see `terraform-test.md`.

## When to Use Terratest Over Native `terraform test`

- **Beyond Terraform's view.** If you need to verify something that does not appear in Terraform state — for example, that a deployed Function App actually responds to HTTPS, that an Azure Monitor metric is being emitted, or that a Key Vault secret can be fetched with a different identity — Terratest gives you the Go runtime to do that.
- **Complex orchestration.** Multi-stage applies with prerequisites, polling for eventual consistency with custom backoff, parallel test fan-out across regions.
- **Existing Terratest investment.** Teams that already have a shared helper library, CI wiring, and review patterns built around Terratest should keep using it where it works rather than rewrite for marginal gain.

If none of those apply, prefer `terraform test` — it's cheaper to write, cheaper to maintain, and runs without Go.

## Directory Layout

All Terratest assets live under a single `terratest/` directory at the module root, split into the Terraform configuration under test (`plan/`) and the Go test driver (`test/`):

```
terratest/
|-- plan/                          # Terraform configuration to apply
|   |-- main.tf                    # Module call(s) with test inputs
|   |-- outputs.tf                 # Outputs the test will assert against
|   |-- variables.tf               # Auth and environment variables only
|   |-- versions.tf                # Provider/Terraform version constraints
|-- test/                          # Go test driver
    |-- go.mod
    |-- go.sum
    |-- terraform_test.go
```

The split exists so the `plan/` directory looks exactly like a real consumer's root configuration — anyone reading it can see how the module is meant to be called. The `test/` directory is purely the Go harness that runs that configuration and inspects its outputs.

## The `plan/` Directory

The `plan/` directory is a normal Terraform root configuration whose only consumer is your test. Treat it like one.

### Use `locals` for test inputs, not `auto.tfvars`

Test input data belongs in a `locals` block (or in variable `default` values). Do not use `*.auto.tfvars` files — they split the test inputs across multiple files for no benefit and make the plan less self-contained.

```hcl
# plan/main.tf
locals {
  netapp_accounts = {
    "default" = {
      resource_group_name = "shared-test-rg"
      location            = "eastus"
      # uses default constructor naming
    }
    "override" = {
      resource_group_name = "shared-test-rg"
      location            = "eastus"
      name_override       = "my-custom-netapp-account"
    }
  }
}

module "netapp" {
  source   = "../../"
  for_each = local.netapp_accounts

  resource_group_name = each.value.resource_group_name
  location            = each.value.location
  name_override       = try(each.value.name_override, null)
}
```

### Pass ARM IDs directly — no `data` source lookups

Pass resource IDs as locals; never use `data` blocks to look them up. Data lookups add API round-trips, can fail intermittently, and obscure what the test is really exercising.

```hcl
# Good — ARM ID constructed in locals
locals {
  vnet_id   = "/subscriptions/${var.azure_subscription_id}/resourceGroups/${var.resource_group_name}/providers/Microsoft.Network/virtualNetworks/${var.vnet_name}"
  subnet_id = "${local.vnet_id}/subnets/default"
}

module "example" {
  source    = "../../"
  vnet_id   = local.vnet_id
  subnet_id = local.subnet_id
}
```

### Use real modules for prerequisites, not raw resources

When a test needs a prerequisite (a key vault, a storage account, a subnet), call the corresponding existing module rather than declaring a raw `azurerm_*` resource. The point of the test is to exercise the module in conditions that mirror real consumer use; raw resources drift from how the module is actually consumed.

```hcl
# Good — call the existing storage-account module
module "storage" {
  source              = "git::https://example.com/_git/terraform-azurerm-storage-account?ref=2.0.0"
  resource_group_name = var.resource_group_name
  location            = var.location
}
```

### Tag test resources with the module VERSION

Reading the `VERSION` file into a tag gives you traceability from a leftover Azure resource back to the exact module version that created it:

```hcl
locals {
  tags = {
    deployed_for = "Terraform - Terratest build terraform.azurerm.acme-virtual-network ${trimspace(file("../../VERSION"))}"
  }
}
```

### `variables.tf`: only the standard auth/environment set

Module-specific test values live in `locals`. The `variables.tf` of a test plan should only declare the standard auth and environment variables your test runner injects. Module-specific values like `vnet_name` or `enable_diagnostics` do **not** belong here — they go in `locals`.

```hcl
variable "azure_subscription_id" {}
variable "azure_client_id" {}
variable "azure_client_secret" {}
variable "azure_tenant_id" {}
variable "location"    { default = "eastus" }
variable "projectcode" { default = "acme-001" }
```

Why: every variable declared here is something the test runner has to know about. Keeping the surface tiny means the runner config never has to change when test inputs do.

### Provider version constraints: `>=`, not pinned

Use `>=` constraints in `versions.tf` so tests run against both the module's declared minimum provider version and whatever is currently latest. A pinned `= 3.74.0` hides breakage with newer providers.

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 3.50.0"
    }
  }
}
```

### Expose outputs the test will assert on

The `plan/outputs.tf` is what gives the Go test something to read. For modules deployed with `for_each`, output a map keyed by the iteration key:

```hcl
# plan/outputs.tf
output "account_id" {
  description = "The ID of the Azure NetApp Account."
  value       = { for k, v in module.netapp : k => v.account_id }
}

output "account_name" {
  description = "The name of the Azure NetApp Account."
  value       = { for k, v in module.netapp : k => v.account_name }
}

output "pool_id" {
  description = "The ID of the NetApp Capacity Pool."
  value       = { for k, v in module.netapp : k => v.pool_id }
}
```

File is `outputs.tf` — plural, never `output.tf`.

## The `test/` Directory

The Go test driver lives in `terratest/test/`.

### `go.mod`

```go
module github.com/acme/terraform-azurerm-netapp/terratest/test

go 1.21

require (
    github.com/gruntwork-io/terratest v0.46.0
    github.com/stretchr/testify v1.8.4
)
```

`go.sum` is generated by `go mod tidy` and committed alongside `go.mod` so test runs are reproducible.

### `terraform_test.go` — naming and package

- File name ends in `_test.go` so `go test` picks it up.
- Package is `test` (matches the directory).
- Each test function is `TestXxx(t *testing.T)`.
- Call `t.Parallel()` at the top of each test so multiple modules can be tested concurrently in CI.

### Optional: shared Terratest helper

If your team maintains a shared Terratest helper module (referenced here as `<your-terratest-helpers>`), import it for `configureTerraformOptions`, `getOutputStringMap`, `assertNotEmpty`, etc. The helper is a placeholder for whatever utilities your team has standardised on — it is **not** required. Plain `github.com/gruntwork-io/terratest/modules/terraform` works without any helper at all; the examples below show both styles.

## The Apply / Assert / Destroy Lifecycle

Every Terratest follows the same four phases, in this order:

1. **Setup** — build `terraform.Options` pointing at `../plan`, with auth env vars set.
2. **Apply** — `terraform.InitAndApply(t, opts)`.
3. **Assert** — read outputs, compare expectations.
4. **Teardown** — `defer terraform.Destroy(t, opts)` *registered before apply* so it runs even on failure.

Registering `defer Destroy` **before** `InitAndApply` is critical: if apply fails halfway, you still want destroy to clean up whatever resources were created. If you defer after apply, a panic during apply leaves orphan resources behind.

```go
package test

import (
    "testing"

    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestNetAppModule(t *testing.T) {
    t.Parallel()

    opts := &terraform.Options{
        TerraformDir: "../plan",
        NoColor:      true,
    }

    // Register destroy FIRST, so it runs even if apply panics
    defer terraform.Destroy(t, opts)

    terraform.InitAndApply(t, opts)

    accountID := terraform.OutputMap(t, opts, "account_id")
    assert.NotEmpty(t, accountID["default"], "default account_id should not be empty")
}
```

## Assert Against Outputs, Not Hardcoded Values

The single most important rule of module testing: **assertions read module outputs**. They never hardcode the names or IDs the module is expected to produce.

Why: a test that hardcodes `assert.Equal("acme-prod-eastus-netapp", name)` is testing that the test wrote that string, not that the module produced it. The moment the naming convention changes, every test breaks for the wrong reason. Reading the actual output and asserting on its *shape* (non-empty, contains a substring, matches a pattern) tests the contract, not the implementation.

```go
// Good — assert on shape of the actual output
accountName := terraform.OutputMap(t, opts, "account_name")
assert.NotEmpty(t, accountName["default"], "default account_name should not be empty")
assert.Contains(t, accountName["override"], "my-custom-netapp-account",
    "override should contain the custom name")

// Bad — hardcoded expected value
assert.Equal(t, "acme-prod-eastus-netapp-default", accountName["default"])
```

The one place hardcoded strings *are* legitimate is when you explicitly set `name_override = "my-custom-netapp-account"` in the plan and want to confirm the module respected it — there, the hardcoded value is the test fixture, not an assumption about the module's behavior.

## Testing Modules with `for_each`

When the module under test uses `for_each`, structure the test plan to instantiate the module multiple times — one instance per behavior you want to cover. Two instances (default naming and one override) is usually plenty; do not provision five copies of the same default config.

```hcl
# plan/main.tf
locals {
  instances = {
    "default"  = { name_override = null }
    "override" = { name_override = "my-custom-name" }
  }
}

module "thing" {
  source   = "../../"
  for_each = local.instances

  resource_group_name = var.resource_group_name
  location            = var.location
  name_override       = each.value.name_override
}
```

```hcl
# plan/outputs.tf
output "thing_id" {
  value = { for k, v in module.thing : k => v.id }
}

output "thing_name" {
  value = { for k, v in module.thing : k => v.name }
}
```

```go
func TestThing(t *testing.T) {
    t.Parallel()

    opts := &terraform.Options{TerraformDir: "../plan", NoColor: true}
    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)

    ids   := terraform.OutputMap(t, opts, "thing_id")
    names := terraform.OutputMap(t, opts, "thing_name")

    // Every instance should have a non-empty id
    for key, id := range ids {
        assert.NotEmpty(t, id, "thing_id for %s should not be empty", key)
    }

    // The override instance should reflect its custom name
    assert.Contains(t, names["override"], "my-custom-name")
}
```

Iterating the output map (`for key, id := range ids`) ensures every `for_each` instance is checked, not just the first one.

## Staged Apply Flows

Some modules need prerequisite infrastructure that must exist *before* the module under test can apply (e.g. a private DNS zone before a private endpoint). For these, run the apply in two stages, with separate `terraform.Options` and separate `defer Destroy` calls — registered in reverse order of apply so teardown unwinds correctly.

```go
func TestComplexModule(t *testing.T) {
    t.Parallel()

    // Stage 1: prereqs
    prereqOpts := &terraform.Options{TerraformDir: "../plan/prereqs", NoColor: true}
    defer terraform.Destroy(t, prereqOpts)
    terraform.InitAndApply(t, prereqOpts)

    // Stage 2: module under test
    moduleOpts := &terraform.Options{TerraformDir: "../plan", NoColor: true}
    defer terraform.Destroy(t, moduleOpts)
    terraform.InitAndApply(t, moduleOpts)

    ids := terraform.OutputMap(t, moduleOpts, "resource_id")
    assert.NotEmpty(t, ids["default"])
}
```

Go runs deferred calls LIFO, so the stage-2 destroy runs before the stage-1 destroy — exactly what you want when stage 2 depends on stage 1.

## Retries and Timeouts

Azure provisioning is sometimes flaky in ways that have nothing to do with your module — propagation delays, eventual consistency, transient throttling. Terratest supports retrying terraform commands for known-transient errors via `RetryableTerraformErrors`:

```go
opts := &terraform.Options{
    TerraformDir: "../plan",
    NoColor:      true,
    RetryableTerraformErrors: map[string]string{
        ".*ResourceGroupNotFound.*":  "RG eventual consistency",
        ".*RetryableError.*":         "Generic retryable Azure error",
    },
    MaxRetries:         3,
    TimeBetweenRetries: 30 * time.Second,
}
```

For long-running modules (AKS, ARO, anything with cluster bootstrapping), set `go test -timeout 90m` on the command line — the default 10-minute Go test timeout will kill apply mid-flight and leave orphan resources.

```bash
go test -v -timeout 90m ./...
```

### Polling Azure for eventual consistency

When a resource appears created in Terraform but is not yet usable (DNS propagation, role-assignment latency, certificate provisioning), wrap the post-apply assertion in `retry.DoWithRetry`:

```go
import "github.com/gruntwork-io/terratest/modules/retry"

retry.DoWithRetry(t, "GET https endpoint", 10, 30*time.Second, func() (string, error) {
    resp, err := http.Get(endpoint)
    if err != nil { return "", err }
    if resp.StatusCode != 200 {
        return "", fmt.Errorf("got %d", resp.StatusCode)
    }
    return "ok", nil
})
```

This is the single best reason to use Terratest over `terraform test` — Terraform itself does not poll, so you cannot assert "the deployed thing actually responds" without a Go runtime.

## Azure Authentication

Tests authenticate using a service principal via environment variables. Do not embed credentials in code or in `*.tfvars` files committed to the repo:

| Variable                | Purpose                          |
|-------------------------|----------------------------------|
| `AZURE_SUBSCRIPTION_ID` | Subscription the test runs in    |
| `AZURE_CLIENT_ID`       | Service principal application ID |
| `AZURE_CLIENT_SECRET`   | Service principal secret         |
| `AZURE_TENANT_ID`       | Azure AD tenant ID               |

The test plan reads these via `var.azure_subscription_id` etc. and the test runner exports them. Local runs use a `.env` file that is gitignored.

## CI Integration

Your CI pipeline runs `go test` from `terratest/test/` after pulling the module. The test runner needs:

- Go (matching the version in `go.mod`).
- Terraform (matching `required_version` in `versions.tf`).
- The four `AZURE_*` env vars exported into the job.
- A long enough timeout (`-timeout 90m` for heavy modules).

### Gating: do not run integration on every PR

Terratest applies real Azure resources and costs real money. Don't gate every PR on it. Instead:

- **PRs:** static + `terraform test` (plan-time + mocked) only.
- **Integration:** triggered by a label like `run-integration`, by a nightly schedule, or as a pre-release gate.

GitHub Actions example:

```yaml
name: terratest
on:
  pull_request:
    types: [labeled]
  schedule:
    - cron: "0 3 * * *"
jobs:
  terratest:
    if: github.event.label.name == 'run-integration' || github.event_name == 'schedule'
    runs-on: ubuntu-latest
    timeout-minutes: 120
    env:
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      AZURE_CLIENT_ID:       ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_CLIENT_SECRET:   ${{ secrets.AZURE_CLIENT_SECRET }}
      AZURE_TENANT_ID:       ${{ secrets.AZURE_TENANT_ID }}
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: 1.9.8 }
      - uses: actions/setup-go@v5
        with: { go-version: '1.21' }
      - name: terratest
        working-directory: terratest/test
        run: go test -v -timeout 90m ./...
```

### Cleaning up failed runs

When `go test -timeout` kills the process before `defer Destroy` runs, resources leak. Two defenses:

1. Tag every test resource with the module `VERSION` and the run ID so a sweeper job can identify and delete leftovers.
2. Schedule a nightly subscription-wide cleanup job that destroys anything tagged `deployed_for = "Terraform - Terratest build*"` older than 24 hours.

## Test Design Principles

1. **Cover different behaviors, not the same behavior multiple times.** One default-naming instance plus one `name_override` instance is enough; three default instances is wasteful.
2. **Test both enabled and disabled paths of optional features.** If the module has an `enable_diagnostics` toggle, exercise both `true` and `false` — that is where conditional logic bugs hide.
3. **Every new feature ships with both a test AND an example.** Tests prove correctness; examples teach consumers.
4. **Verify new feature inputs reach the module.** When you add a test input for a new variable, confirm the module call actually passes it — otherwise the feature is silently untested.
5. **Minimize the test resource footprint.** Cloud minutes cost money; do not provision a jumpbox you do not assert on.
6. **Document untestable scenarios explicitly.** When the test SPN cannot exercise a code path (cross-tenant assignments, marketplace-gated SKUs), call it out in the README rather than silently skipping.
7. **Do not rename resources to dodge prior failed-destroy collisions.** If a prior run left orphans, clean them up; renaming masks the real problem.

## Common Mistakes

1. **`defer Destroy` registered after apply.** A panic in apply leaves resources orphaned. Always defer destroy *before* calling `InitAndApply`.
2. **Hardcoding expected names or IDs in assertions.** The test breaks for the wrong reason every time the naming convention shifts.
3. **`data` source lookups in the test plan.** Adds latency and a new failure mode that has nothing to do with the module under test.
4. **`auto.tfvars` files in `plan/`.** Test inputs belong in `locals` or variable defaults — keep the plan self-contained.
5. **Module-specific variables in `variables.tf`.** Only standard auth/environment vars. Everything else is a `local`.
6. **Pinned (`= 3.74.0`) provider versions in `versions.tf`.** Use `>=` so the test runs against minimum-required and current latest.
7. **Default 10-minute `go test` timeout on a 30-minute apply.** Set `-timeout 90m` (or longer) for heavy modules.
8. **Forgetting `t.Parallel()`.** Your single test still works, but CI burns wall-clock running tests serially.
9. **Asserting only the first instance of a `for_each` map.** Iterate every key — that is the whole point of the map output.
10. **Raw `azurerm_*` resources for prereqs when a module exists.** Drifts the test from how the module is actually consumed.
11. **No `VERSION` reference in test resource tags.** When a leftover resource shows up next week, you cannot tell which module run created it.
12. **Running Terratest on every PR.** Static + `terraform test` should be the PR gate; Terratest belongs on a schedule or behind a label.

## Review Checklist

- [ ] `terratest/plan/` and `terratest/test/` directories both exist.
- [ ] `outputs.tf` (plural) in `plan/` exposes every value the test asserts on.
- [ ] Test inputs are in `locals` or variable defaults — no `auto.tfvars`.
- [ ] `variables.tf` only declares the standard auth/environment variables.
- [ ] Provider versions in `versions.tf` use `>=`, not pinned `=`.
- [ ] No `data` source lookups in the test plan.
- [ ] Prerequisite resources use existing modules, not raw `azurerm_*` resources.
- [ ] Test resources are tagged with the module `VERSION`.
- [ ] `go.mod` and `go.sum` are committed.
- [ ] Each test function calls `t.Parallel()`.
- [ ] `defer terraform.Destroy(...)` is registered **before** `InitAndApply`.
- [ ] Assertions read module outputs; no hardcoded expected names/IDs.
- [ ] `for_each` test cases iterate the entire output map, not just one key.
- [ ] Staged applies (if used) defer destroys in reverse order via separate `terraform.Options`.
- [ ] `go test -timeout` is large enough for the module's slowest apply.
- [ ] CI gates Terratest behind a label or schedule, not on every PR.
