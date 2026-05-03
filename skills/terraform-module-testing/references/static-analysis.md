# Static Analysis for Terraform Modules

Static analysis is the cheapest layer of the testing pyramid: no state, no plan, no Azure calls, runs in milliseconds. It catches syntax errors, formatting drift, deprecated patterns, security misconfigurations, and lint violations before any apply ever runs. Every module should run the full static suite on every commit and every PR — these checks are mandatory and cheap.

For plan-time and integration tests, see `terraform-test.md` and `terratest.md`.

## The Static Suite

| Tool | What it catches | When |
|---|---|---|
| `terraform fmt` | Formatting drift | Every save, gating CI |
| `terraform validate` | Syntax, schema, type errors | Every save, gating CI |
| `tflint` | Provider-specific lint, deprecations, naming | Every save, gating CI |
| `checkov` | Security misconfigurations, compliance | Gating CI |
| `tfsec` / `trivy` | Security misconfigurations (Aqua) | Gating CI |
| `terrascan` | OPA-based policy scanning | Optional |

Run them in this order — cheaper checks first so a failure short-circuits before the expensive ones run.

## `terraform fmt`

Canonical formatting. There is one correct format and `fmt` enforces it.

```bash
# Verify formatting (gating CI step — exits non-zero on drift)
terraform fmt -check -recursive

# Apply formatting (local development)
terraform fmt -recursive
```

Use `-check` in CI; never run `fmt` without `-check` in CI because it would silently rewrite the working tree.

## `terraform validate`

Checks that the configuration is internally consistent — variable types match, references resolve, required attributes are present, provider blocks parse. It does not call any provider API and does not need backend credentials.

```bash
# Initialize without contacting any backend
terraform init -backend=false

# Validate
terraform validate
```

`-backend=false` is essential: without it, `init` tries to configure the configured backend (S3, AzureRM, etc.), which requires credentials and network access. For a static check, you don't need that.

In CI:

```bash
terraform init -backend=false
terraform validate || exit 1
```

Always check the exit code. A common bug is piping `terraform validate` into something else and losing the non-zero exit.

## `tflint`

`tflint` extends `terraform validate` with provider-specific deep checks: deprecated arguments, invalid Azure region names, naming convention rules, version constraints. Configure it via `.tflint.hcl` at the repo root.

### `.tflint.hcl`

```hcl
plugin "terraform" {
  enabled = true
  preset  = "recommended"
}

plugin "azurerm" {
  enabled = true
  version = "0.27.0"
  source  = "github.com/terraform-linters/tflint-ruleset-azurerm"
}

config {
  call_module_type = "all"
  force            = false
}

rule "terraform_required_version" {
  enabled = true
}

rule "terraform_required_providers" {
  enabled = true
}

rule "terraform_unused_declarations" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}

rule "terraform_documented_variables" {
  enabled = true
}

rule "terraform_naming_convention" {
  enabled = true
  format  = "snake_case"
}
```

### Running

```bash
# One-time install of plugins
tflint --init

# Lint the current directory
tflint

# Lint recursively (every subdirectory with Terraform files)
tflint --recursive

# JSON output for CI parsing
tflint --format json
```

### Useful flags

- `--call-module-type all` — also lint modules referenced by `source = "../shared"` etc.
- `--force` — return zero exit even on warnings (don't use this in CI).
- `--disable-rule=<name>` — disable a single rule on the command line.

### Common findings

- `azurerm_resource_invalid_name` — region or SKU name typo.
- `terraform_deprecated_interpolation` — old `${var.x}` syntax in non-string contexts.
- `terraform_unused_declarations` — variable or output declared but never referenced.
- `terraform_required_version` — `required_version` missing from `versions.tf`.

## `checkov`

`checkov` scans for security and compliance misconfigurations: public storage accounts, missing encryption at rest, public network access on PaaS services, missing diagnostic settings. It ships with hundreds of Azure-specific rules.

### Config (`checkov.yml` or `.checkov.yaml`)

```yaml
framework:
  - terraform
soft-fail: false
download-external-modules: true
skip-check:
  - CKV_AZURE_109   # example: Key Vault firewall — handled by network module instead
  - CKV2_AZURE_38   # example: Soft delete — managed centrally, not per-resource
output: cli
quiet: true
compact: true
```

### Running

```bash
# Scan the current directory
checkov -d . --config-file checkov.yml

# Scan a single file
checkov -f main.tf

# JSON for CI ingestion
checkov -d . --output json --output-file-path checkov-report.json

# Hard fail (exit non-zero on any finding)
checkov -d . --soft-fail false

# Soft fail (always exit zero — only use for ramp-up, never for production CI)
checkov -d . --soft-fail
```

### Severity gating

`checkov` itself does not have severity flags directly on the CLI in older versions; for modern versions:

```bash
checkov -d . --check HIGH,CRITICAL --soft-fail-on LOW,MEDIUM
```

Treat HIGH and CRITICAL as gating; treat LOW and MEDIUM as warnings until you've burned them down.

### Suppressing in code

Inline suppressions are explicit and reviewable:

```hcl
# checkov:skip=CKV_AZURE_33: Diagnostic settings managed by central monitoring module
resource "azurerm_storage_account" "main" {
  # ...
}
```

Prefer inline suppressions with a justification over global skip lists — global skips hide context.

### Common findings on Azure modules

- `CKV_AZURE_33` — Storage logging not enabled.
- `CKV_AZURE_59` — Storage public network access not disabled.
- `CKV_AZURE_109` — Key Vault firewall not configured.
- `CKV_AZURE_206` — Storage account replication is LRS (consider GRS/ZRS).
- `CKV2_AZURE_1` — CMK not used for storage.

## `tfsec` / `trivy`

`tfsec` was the original Aqua tool; it is now bundled into `trivy config`. Either works. Both are faster than `checkov` and have overlapping but not identical rule sets — running both catches more.

### Running `tfsec`

```bash
# Scan the current directory
tfsec .

# JSON output
tfsec . --format json --out tfsec.json

# Fail on a minimum severity
tfsec . --minimum-severity HIGH

# Exclude specific checks
tfsec . --exclude azure-storage-default-action-deny
```

### Running `trivy` for IaC

```bash
# Equivalent to tfsec
trivy config .

# Filter by severity
trivy config . --severity HIGH,CRITICAL

# JSON output
trivy config . --format json --output trivy.json

# Fail with exit code 1 on findings
trivy config . --exit-code 1 --severity HIGH,CRITICAL
```

### Inline suppression

```hcl
#tfsec:ignore:azure-storage-default-action-deny
resource "azurerm_storage_account" "main" {
  # ...
}
```

## `terrascan`

OPA-based scanner. Useful when your organization already writes Rego policies for other things and wants to reuse the policy language for Terraform.

```bash
# Scan
terrascan scan -i terraform -d .

# Specific policy pack
terrascan scan -i terraform -d . -p ~/policies/

# JSON output
terrascan scan -i terraform -d . -o json
```

If you don't already use Rego, `checkov` and `tfsec` cover the same ground with no policy language to learn.

## Pre-commit Integration

Run the cheap checks every commit so drift is caught locally before it reaches CI.

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.96.1
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
        args:
          - --hook-config=--retry-once-with-cleanup=true
      - id: terraform_tflint
        args:
          - --args=--config=__GIT_WORKING_DIR__/.tflint.hcl
      - id: terraform_checkov
        args:
          - --args=--config-file __GIT_WORKING_DIR__/checkov.yml
          - --args=--quiet
          - --args=--compact

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: end-of-file-fixer
      - id: trailing-whitespace
```

Install:

```bash
pre-commit install
pre-commit run --all-files
```

## CI Examples

### GitHub Actions

```yaml
name: static
on:
  pull_request:
  push:
    branches: [main]

jobs:
  static:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.8

      - name: fmt
        run: terraform fmt -check -recursive

      - name: init (no backend)
        run: terraform init -backend=false

      - name: validate
        run: terraform validate

      - uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: v0.53.0

      - name: tflint
        run: |
          tflint --init
          tflint --recursive --format compact

      - name: checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          config_file: checkov.yml
          framework: terraform
          soft_fail: false

      - name: trivy config
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: .
          severity: HIGH,CRITICAL
          exit-code: 1
```

### Azure DevOps Pipeline

```yaml
trigger:
  branches: { include: [main] }
pr:
  branches: { include: ["*"] }

pool:
  vmImage: ubuntu-latest

steps:
  - task: TerraformInstaller@1
    inputs:
      terraformVersion: 1.9.8

  - script: terraform fmt -check -recursive
    displayName: terraform fmt

  - script: |
      terraform init -backend=false
      terraform validate
    displayName: terraform validate

  - script: |
      curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
      tflint --init
      tflint --recursive --format compact
    displayName: tflint

  - script: |
      pip install checkov
      checkov -d . --config-file checkov.yml
    displayName: checkov

  - script: |
      curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
      trivy config . --severity HIGH,CRITICAL --exit-code 1
    displayName: trivy
```

## Decision Rules

- **`tflint` vs `checkov`** — `tflint` is for HCL correctness and provider-specific lint rules (typo'd region, deprecated argument, missing `required_version`). `checkov` is for security/compliance (public storage account, missing encryption). Run both.
- **`checkov` vs `tfsec`/`trivy`** — significant overlap, but rule coverage differs. If you have time for one, `checkov` has the broader Azure rule set. Running both catches a few extra findings each.
- **`terrascan`** — only adds value if your org already writes OPA/Rego policies for other things.

## Common Mistakes

1. **`terraform fmt` without `-check` in CI.** Silently rewrites the tree; defeats the gate.
2. **`terraform init` without `-backend=false` in CI.** Wastes time and may fail on credentials.
3. **Ignoring `terraform validate` exit code.** Always check it explicitly in shell scripts.
4. **`--soft-fail` on every checkov run.** Never finds anything because nothing fails. Use it during burn-down only.
5. **Global skip-check list with no comments.** Future readers can't tell which suppressions are still valid.
6. **Pinning every linter inline with no upgrade path.** Pin in `pre-commit-config.yaml` and bump on a schedule.
7. **Running tflint without `--init`.** The plugins aren't downloaded; many rules silently skip.
8. **Treating linter warnings as informational.** If they don't gate, they get ignored.

## Review Checklist

- [ ] `terraform fmt -check -recursive` runs in CI and gates the merge.
- [ ] `terraform init -backend=false && terraform validate` runs in CI.
- [ ] `.tflint.hcl` exists, enables the azurerm plugin, and `terraform_required_version` is on.
- [ ] `tflint --init && tflint --recursive` runs in CI.
- [ ] `checkov.yml` exists and is referenced by CI; severity gating is set.
- [ ] At least one of `tfsec`/`trivy config`/`terrascan` runs in CI.
- [ ] Inline `checkov:skip=` and `tfsec:ignore:` suppressions include a justification comment.
- [ ] `pre-commit-config.yaml` exists and contributors are encouraged to install it.
- [ ] CI step exit codes are checked — no silent failures.
