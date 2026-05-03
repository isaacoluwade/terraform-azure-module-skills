---
name: terraform-module-documentation
description: Documentation conventions for Terraform Azure modules — covers terraform-docs README auto-generation (the `terraform-docs.yml` config and the README marker comments in `main.tf`), `CHANGELOG.md` in Keep a Changelog format, ServiceDoc / architecture documentation under `docs/`, the standard `.pre-commit-config.yaml` (terraform fmt, terraform-docs, validate, tflint, yamlfmt, shellcheck), and Azure DevOps pipeline integration with the gates that run on PR. Use this skill when you need to "generate README for module", configure or debug `terraform-docs`, draft a `CHANGELOG entry`, set up a `pre-commit config`, write a ServiceDoc, or wire up an `Azure DevOps pipeline for Terraform module` builds. This is the right skill any time module docs, README headers, the changelog, or the CI gates are in scope — even for small changes, since stale docs are the most common review reject.
---

# Terraform Module Documentation

Module documentation is part of the public API. The README is auto-generated, the CHANGELOG is consumer-facing, and the ServiceDoc is the certification artifact — all three must stay in sync with the code in every PR. Stale docs are the single most common reason a Terraform module PR gets bounced.

This skill covers the *documentation* conventions. For version bumps and release flow, see `terraform-module-versioning`. For directory layout and required files, see `terraform-module-structure`. For HCL formatting rules enforced by `terraform fmt`, see `terraform-style-and-formatting`.

## Contents
- [README.md — Auto-Generated with terraform-docs](#readmemd--auto-generated-with-terraform-docs)
- [Module Header Comment in main.tf](#module-header-comment-in-maintf)
- [terraform-docs Configuration](#terraform-docs-configuration)
- [CHANGELOG.md — Keep a Changelog](#changelogmd--keep-a-changelog)
- [ServiceDoc — Architecture Documentation](#servicedoc--architecture-documentation)
- [Architecture Diagrams](#architecture-diagrams)
- [Pre-commit Hooks](#pre-commit-hooks)
- [Azure DevOps Pipeline Integration](#azure-devops-pipeline-integration)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)

## README.md — Auto-Generated with terraform-docs

The `README.md` at the module root is **auto-generated** by [`terraform-docs`](https://github.com/terraform-docs/terraform-docs). **Never hand-write it.** A hand-written README will drift from the code within one or two PRs, and reviewers cannot trust it.

The auto-generated README contains:

| Section      | Source                                              |
| ------------ | --------------------------------------------------- |
| Header       | `/** ... */` doc comment at the top of `main.tf`    |
| Requirements | `terraform { required_version, required_providers }`|
| Providers    | Declared providers and their versions               |
| Modules      | Child modules called by this module                 |
| Resources    | All resources/data sources the module manages       |
| Inputs       | Each `variable` block (name, type, default, etc.)   |
| Outputs      | Each `output` block (name, description)             |

**Rule — any PR that modifies `variables.tf`, `outputs.tf`, providers, or resources must include a regenerated `README.md`.** The pre-commit hook regenerates it automatically; the pipeline check rejects PRs where the committed README does not match what `terraform-docs` would produce.

### Generation command

```bash
terraform-docs markdown table --output-file README.md .
```

Use the latest stable `terraform-docs`. The pre-commit hook pins a specific version so every contributor produces byte-identical output — bumping the hook version is itself a CHANGELOG-worthy change, because output formatting can shift.

## Module Header Comment in main.tf

Every module **must** include a doc-comment block at the very top of `main.tf`. `terraform-docs` parses this comment and uses it as the module title and description in the generated README.

```hcl
# Good — header comment at top of main.tf
/**
 * # Azure Kubernetes Fleet Manager Terraform module
 *
 * This module provisions an Azure Kubernetes Fleet Manager instance and the
 * associated member-cluster registrations. It assumes the AKS clusters being
 * registered already exist.
 */

resource "azurerm_kubernetes_fleet_manager" "main" {
  # ...
}
```

```hcl
# Bad — no header comment, README has no title or description
resource "azurerm_kubernetes_fleet_manager" "main" {
  # ...
}
```

**Why:** without the header comment the generated README opens straight into the Requirements table — consumers landing on the registry page have no idea what the module does. The comment lives in `main.tf` (not a separate file) so it is impossible to delete the resource without also seeing the description that needs updating.

## terraform-docs Configuration

For consistent output across the module fleet, ship a `terraform-docs.yml` (or `.terraform-docs.yml`) at the module root and reference it from the pre-commit hook. The config below matches what most Azure modules use — `markdown table` formatter, `replace` mode, and a content template that strips the trailing `terraform-docs` watermark.

```yaml
# terraform-docs.yml
formatter: "markdown table"

content: |-
  {{ .Content }}

output:
  file: README.md
  mode: replace
  template: |-
    {{ .Content }}

sort:
  enabled: true
  by: name

settings:
  anchor: true
  color: true
  default: true
  description: true
  escape: true
  hide-empty: false
  html: true
  indent: 2
  lockfile: true
  read-comments: true
  required: true
  sensitive: true
  type: true
```

**Why these settings:**
- `mode: replace` — overwrite the file each run; do not append, otherwise consecutive runs accumulate duplicate sections.
- `sort.by: name` — variables and outputs sort alphabetically, so README diffs reflect real changes instead of declaration-order churn.
- `read-comments: true` — pulls the `/** */` header from `main.tf` into the rendered title.
- `required: true`, `default: true`, `type: true` — keep the inputs table fully populated; partial tables are easy to miss in review.

### README marker comments (optional)

If you need to keep some hand-authored content alongside the generated tables (e.g. a custom usage example), use `terraform-docs`' marker pattern:

```markdown
<!-- BEGIN_TF_DOCS -->
... terraform-docs writes between these markers ...
<!-- END_TF_DOCS -->
```

Anything outside the markers is preserved across regenerations. Place hand-authored sections (Usage, Examples) above `BEGIN_TF_DOCS`. The pre-commit hook honours these markers automatically when `output.mode: inject` is configured instead of `replace`.

## CHANGELOG.md — Keep a Changelog

Releases are tracked in a `CHANGELOG.md` at the module root, using [Keep a Changelog](https://keepachangelog.com/) sections (`Added`, `Changed`, `Fixed`, `Removed`, `Deprecated`, `Security`). The CHANGELOG is the consumer's authoritative source for "what changed and do I need to do anything about it".

### Format

```markdown
## [1.3.0] - 2025-04-14

### Added

- Add support for KEDA addon with configurable settings.
- Add Azure Fabric Capacity resource provisioning with configurable SKU tiers.

### Changed

- **Breaking:** replace variable `shared_image` with `source_image_id`; see the variable description for the new format.
- Update service documentation and architecture diagram.

### Fixed

- Fix variable `availability_zone` validation to accept a `null` value.
- Fix the ability to configure Azure Service Bus network rule set.

### Removed

- **Breaking:** remove variable `vault_type` (drops support for Hashicorp Vault).
- Remove unused variables `app_service` and `hostname_binding`.
```

### Rules

1. **Imperative mood, present tense.** Start each entry with `Add`, `Fix`, `Refactor`, `Bump`, `Document`, `Deprecate`, `Remove`. Not "Added" or "Adds". This matches commit-message convention and reads naturally as "this version will... add support for X".
2. **Describe consumer impact, not internal code changes.** "Merge resources for `cert` and `cert_order` into one file" is not changelog-worthy; the consumer's `terraform plan` is unaffected. "Fix `cert_order` not being recreated when `cert` rotates" is.
3. **Prefix breaking changes with `**Breaking:**`.** This is the only signal a downstream consumer has at a glance that they need to update their calling configuration before bumping.
4. **First version: squash entries.** Initial release combines everything under a single `### Added` and lists what the module provisions, not the development history.
5. **Version classification follows SemVer:**
   - **Major** — any breaking change to inputs, outputs, or required behavior.
   - **Minor** — new optional inputs, new outputs, new resources behind a flag.
   - **Patch** — bug fixes that do not change the public API (e.g. relaxing a validation rule that was too strict).

### Good vs bad entries

```markdown
# Bad — retells code refactoring, wrong mood
## [1.1.0]
- Created new file for delivery_property block
- Moved resources around

# Good — customer-focused, imperative
## [1.1.0]
### Fixed
- Fix condition for setting up Event Grid delivery properties via the `delivery_property` variable.
```

### Release notice

If the pipeline pulls a one-line release notice for notifications (Slack/Teams/email), put it as the first line under the version heading, italicised, terminated with a period:

```markdown
## [1.4.0] - 2025-05-01

_Adds support for `minimum_tls_version` on the storage account._

### Added
- Add input `minimum_tls_version` to enforce TLS 1.2 on the storage account.
```

The italic single-sentence form is what the pipeline picks up — multi-line or unformatted notices break the extractor.

## ServiceDoc — Architecture Documentation

The ServiceDoc lives at `docs/ServiceDoc-<service>.md` (e.g. `docs/ServiceDoc-aro.md`, `docs/ServiceDoc-storage.md`). It is the architecture-and-operations document for module consumers — what the README cannot capture because the README is mechanical.

**Rule — read the ServiceDoc before changing code.** When code changes, the ServiceDoc must be updated in the same PR. The two are reviewed together.

### Required sections

A complete ServiceDoc covers:

1. **Title, Audience, Scope** — who is this for and what does the module provision vs assume as input.
2. **Technical Design** — architecture, the resources the module creates, design decisions. References the architecture diagram.
3. **Networking Prerequisites** — explicit list of required network resources with variable names, formats, and example values.
4. **Security** — authentication, RBAC, network security (private endpoints, NSG expectations), encryption at rest and in transit.
5. **Identity and Authentication** — managed identity vs service principal, where credentials come from.
6. **Key Vault Integration** — variable names, required RBAC roles on the vault, working example.
7. **Optional Integrations** — step-by-step enablement for each (DNS registration, monitoring, replication, etc.).
8. **Operations and Support** — monitoring, alerting, log destinations, scaling.
9. **Troubleshooting** — concrete `az`/`oc`/`kubectl` commands for the common failure modes, not "check the logs".
10. **Validation** — post-deploy checks: which command, which variable flag, what the expected output looks like.
11. **Disaster Recovery** — failover pattern, RPO/RTO targets.
12. **Decommissioning** — drain order, manual cleanup steps, what `terraform destroy` will not handle.
13. **Appendices** — glossary, external references, doc version history.

### Be explicit, not vague

```markdown
# Bad — vague, the consumer cannot act on this
"Ensure networking is configured properly before deploying."

# Good — explicit, copy-pasteable
"Provide `master_subnet_id` and `worker_subnet_id` as full Azure resource IDs
(format: `/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Network/virtualNetworks/<vnet>/subnets/<subnet>`).
Both subnets must have the `Microsoft.ContainerService` service endpoint enabled."
```

```markdown
# Bad — generic
"If the cluster is not working, check the Azure portal for errors."

# Good — concrete commands
"Run `az aro show -n <cluster> -g <rg> --query provisioningState`. If the result is `Failed`,
run `az aro show -n <cluster> -g <rg> --query 'clusterProfile.failedProvisioningState'`
for the specific error code."
```

### Cross-link related modules

When the ServiceDoc mentions a sibling capability (Key Vault, NSG, monitoring, private endpoint), link to the corresponding Terraform module so consumers can find it without searching:

```markdown
# Bad
- SSH keys must be stored in an Azure Key Vault.

# Good
- SSH keys must be stored in an Azure Key Vault. See the [Key Vault module](<your-azure-devops-org>/_git/terraform-azurerm-key-vault).
```

### Use absolute language for security

Security statements must be unambiguous. Hedging language is the most common reason a security review comes back:

```markdown
# Bad — hedge
- Avoid embedding secrets in code when possible.

# Good — absolute
- **Never embed secrets in code**, configuration files, or any other plaintext format under any circumstances.
```

### Keep information inline, not behind paywalled links

If a consumer cannot reach the linked document (internal wiki, restricted SharePoint), include the relevant content directly in the ServiceDoc. A broken link with no fallback is worse than no link.

### Standard compliance statements

Most modules include some variation of these as a checklist consumers can verify against:

```markdown
- Naming conventions must follow your organization's published Azure naming standard.
- Only approved regions (e.g. `eastus`, `eastus2`) may be used for resource deployment.
- Public network access must be disabled by default; expose service via private endpoints. Public access is enforced by Azure Policy and exemption is required to override.
- Features of this service are certified and tested against `azurerm` provider up to version `<X.Y.Z>`.
```

### ServiceDoc version is independent of module version

The ServiceDoc has its own version line (e.g. `## Service Version - 1.6.0`). It tracks documentation content changes — bump it when ServiceDoc content changes, regardless of whether the module version bumps. Code-only changes do not bump the doc version; doc-only changes do.

## Architecture Diagrams

Diagrams live under `docs/images/`. Use **draw.io** so source and rendered output ship together:

```
docs/images/aro.drawio        # Editable source
docs/images/aro.drawio.png    # Rendered, embedded in markdown
```

Reference diagrams from the ServiceDoc with relative paths and a caption that names what the reader is looking at:

```markdown
![ARO Architecture](images/aro.drawio.png)

*Figure 1: ARO cluster topology — master and worker subnets, private API endpoint, monitoring integration.*
```

**Why both files:** the `.drawio` source is editable by anyone who later needs to update the topology; the `.png` is what GitHub/Azure DevOps renders inline. Shipping only the PNG forces the next editor to recreate from scratch. Verify both paths exist before merging — a broken image link is invisible in the editor and obvious on the registry.

## Pre-commit Hooks

Every module ships a `.pre-commit-config.yaml` at the root. The hooks run locally before commit (and again in the pipeline) so formatting, README staleness, and lint errors surface before code review.

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://gitlab.com/vojko.pribudic.foss/pre-commit-update
    rev: v0.9.0
    hooks:
      - id: pre-commit-update

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: check-yaml
        stages: [pre-commit]
      - id: end-of-file-fixer
        stages: [pre-commit]
      - id: trailing-whitespace
        stages: [pre-commit]

  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.97.4
    hooks:
      - id: terraform_fmt
      - id: terraform_validate

  - repo: https://github.com/gruntwork-io/pre-commit
    rev: v0.1.26
    hooks:
      - id: tflint

  - repo: https://github.com/terraform-docs/terraform-docs
    rev: v0.21.0
    hooks:
      - id: terraform-docs-go
        args:
          - "markdown"
          - "table"
          - "--output-mode"
          - "replace"
          - "--output-file"
          - "README.md"
          - "--output-template"
          - "{{ .Content }}\n"
          - "."

  - repo: https://github.com/jumanjihouse/pre-commit-hook-yamlfmt
    rev: 0.2.3
    hooks:
      - id: yamlfmt
        args: [--mapping, '2', --sequence, '4', --offset, '2']

  - repo: https://github.com/jumanjihouse/pre-commit-hooks
    rev: 3.0.0
    hooks:
      - id: shellcheck
        stages: [pre-commit]
      - id: shfmt
        stages: [pre-commit]
```

### What each hook does

| Hook                  | Purpose                                                  |
| --------------------- | -------------------------------------------------------- |
| `pre-commit-update`   | Auto-bumps hook revisions so the toolchain does not rot. |
| `check-yaml`          | Catches malformed YAML in pipelines and configs.         |
| `end-of-file-fixer`   | Ensures every file ends with a single newline.           |
| `trailing-whitespace` | Strips trailing spaces — keeps diffs clean.              |
| `terraform_fmt`       | Canonical HCL formatting (no opinions to argue over).    |
| `terraform_validate`  | Catches reference and type errors before push.           |
| `tflint`              | Lints for Azure-specific anti-patterns and stale syntax. |
| `terraform-docs-go`   | Regenerates `README.md` so it cannot drift from code.    |
| `yamlfmt`             | Stable YAML indentation across pipeline files.           |
| `shellcheck`          | Lints any helper shell scripts in the repo.              |
| `shfmt`               | Canonical shell formatting.                              |

### Key terraform-docs args

- `--output-mode replace` — overwrite, do not inject. Repeated `inject` runs accumulate duplicate sections if the markers ever drift.
- `--output-template "{{ .Content }}\n"` — strips the trailing footer line so the README ends cleanly with a single newline.

### Setup

```bash
pip install pre-commit
pre-commit install
pre-commit install --hook-type commit-msg

# Run all hooks against the entire repo (useful first time)
pre-commit run --all-files
```

**Run pre-commit locally before pushing.** It catches stale READMEs, formatting drift, and YAML errors that would otherwise fail CI five minutes later.

## Azure DevOps Pipeline Integration

A standard `azure-pipelines.yml` lives at the module root and extends a shared template that owns the test-plan-apply-publish flow. The module-specific file is mostly parameter wiring.

```yaml
---
trigger:
  - main

variables:
  - group: terraform-modules
  - name: varGroupId
    value: <group-id>
  - name: lVersion
    value: $[variables.lastVersion]
  - name: tfToken
    value: $[variables.terraformToken]
  - name: tfTokenProd
    value: $[variables.tfeTokenProd]

resources:
  repositories:
    - repository: pipeline_templates
      type: git
      name: <your-pipeline-templates>/<your-pipeline-templates>
      ref: refs/heads/main

extends:
  template: terraform/terraform_module.yml@<your-pipeline-templates>
  parameters:
    varGroupId: $(varGroupId)
    lastVersion: $(lVersion)
    projectName: <your-ado-project>
    azureSub: <subscription-id>
    serviceConnection: <service-connection-name>
    tfToken: $(tfToken)
    tfTokenProd: $(tfeTokenProd)
    tfOrg: azure
    modName: <module-name>
    provider: azurerm
    isServiceDoc: true
```

### Parameters

| Parameter           | Description                                           | Example                |
| ------------------- | ----------------------------------------------------- | ---------------------- |
| `varGroupId`        | Azure DevOps variable group ID                        | `142`                  |
| `projectName`       | ADO project name                                      | `<your-ado-project>`   |
| `azureSub`          | Azure subscription ID for test deployments            | `xxxxxxxx-xxxx-...`    |
| `serviceConnection` | ADO service connection with Azure deploy permissions  | `sc-terraform-modules` |
| `tfOrg`             | Terraform Cloud / Enterprise organization             | `azure`                |
| `modName`           | Module short name (without `terraform-azurerm-` prefix)| `storage-account`     |
| `provider`          | Terraform provider                                    | `azurerm`              |
| `isServiceDoc`      | Whether ServiceDoc certification gate runs            | `true`                 |

### Gates that run on PR

The shared template runs these checks; failing any one of them blocks merge:

- **`terraform fmt -check`** — formatting matches the canonical style.
- **`terraform validate`** — references resolve, types check.
- **`tflint`** — no Azure anti-patterns.
- **`terraform-docs` drift check** — committed README matches what would be regenerated. Catches "forgot to run pre-commit".
- **CHANGELOG / VERSION sync** — if the module touched code that affects consumers, both `CHANGELOG.md` and `VERSION` must have moved together.
- **Terratest plan/apply** — `examples/main` deploys end-to-end against the test subscription; output assertions run.
- **ServiceDoc presence** — when `isServiceDoc: true`, the gate refuses to publish if `docs/ServiceDoc-*.md` is missing or empty.

### Placeholders to replace when bootstrapping

When copying the pipeline to a new module, replace:

- `<group-id>` — variable group ID from your ADO project.
- `<subscription-id>` — Azure subscription used for test deployments.
- `<service-connection-name>` — ADO service connection with deploy permissions.
- `<module-name>` — the short module name (e.g. `aro`, `storage-account`).

The `<your-pipeline-templates>` and `<your-ado-project>` placeholders refer to the shared infrastructure — substitute the real values for your organization once and they should not change per-module.

## Common Mistakes

1. **Hand-editing `README.md`.** The next pre-commit run reverts your edits. Change the code or the `main.tf` header comment, then regenerate.
2. **Missing `/** */` header comment in `main.tf`.** The README opens straight into the Requirements table with no module description.
3. **Stale README in PR** — variables or outputs changed, README not regenerated. Pipeline catches it; rerunning pre-commit locally is faster.
4. **CHANGELOG entries that describe code refactors instead of consumer impact.** "Moved files around" is not actionable for downstream callers.
5. **Forgetting `**Breaking:**` prefix on a breaking change.** Consumers reading the changelog have no warning before they bump.
6. **Patch bump that includes a breaking change**, or major bump with no breaking changes — version classification doesn't match the entries.
7. **`VERSION` updated but `CHANGELOG.md` not updated** (or vice versa). Both must move together — the pipeline gate enforces it.
8. **Vague ServiceDoc networking section** — "configure networking properly" instead of explicit variable names, formats, and required service endpoints.
9. **Generic troubleshooting** — "check the logs" instead of the actual `az`/`kubectl` commands the consumer needs to run.
10. **Linking to internal docs the consumer cannot reach.** Inline the information instead.
11. **Missing architecture diagram or referencing a path that does not exist.** Verify the `docs/images/*.drawio.png` resolves before merging.
12. **Skipping pre-commit locally and discovering README drift in CI.** Costs a second push and a second pipeline run every time.
13. **Bumping `terraform-docs` hook revision without re-running it on every module.** Output formatting can change subtly; commit the regenerated README in the same PR.

## Review Checklist

- [ ] `README.md` is auto-generated by `terraform-docs`, not hand-edited.
- [ ] `main.tf` has a `/** # Module Title */` header comment that renders into the README.
- [ ] PR regenerated the README after any change to variables, outputs, providers, or resources.
- [ ] `CHANGELOG.md` has an entry for this version, in imperative mood, focused on consumer impact.
- [ ] Breaking changes are prefixed with `**Breaking:**`.
- [ ] `VERSION` and `CHANGELOG.md` were bumped together; SemVer classification matches the entries.
- [ ] `docs/ServiceDoc-<service>.md` exists, has title/audience/scope, and matches the current code.
- [ ] ServiceDoc networking section lists explicit variables, formats, and required service endpoints.
- [ ] ServiceDoc troubleshooting includes concrete commands, not generic advice.
- [ ] ServiceDoc cross-links sibling modules where relevant (Key Vault, NSG, monitoring, etc.).
- [ ] Architecture diagram exists at `docs/images/<service>.drawio.png` (and the `.drawio` source ships alongside).
- [ ] `.pre-commit-config.yaml` is present and includes `terraform_fmt`, `terraform_validate`, `tflint`, `terraform-docs-go`.
- [ ] `azure-pipelines.yml` is wired to the shared template with the correct `modName`, `projectName`, and `isServiceDoc` flag.
- [ ] Pre-commit was run locally before push; CI gates pass on the first attempt.
