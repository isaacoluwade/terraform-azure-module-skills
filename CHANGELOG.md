# Changelog

All notable changes to this skill collection are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] — Initial public release

### Added

#### Orchestrator
- `terraform-azure-module` — scaffolding workflow, module file layout, creation checklist, routing index to topical skills

#### Per-file authoring
- `terraform-variable-design` — `variables.tf` design (typing, validation, optionals, defaults, `name_override`)
- `terraform-locals-patterns` — `locals.tf` patterns (location maps, `primary_name`, tags merge)
- `terraform-module-outputs` — `outputs.tf` conventions
- `terraform-provider-versioning` — `versions.tf`, provider constraints

#### Cross-cutting authoring
- `terraform-resource-naming` — Azure and HCL identifier naming conventions
- `terraform-module-structure` — directory layout, examples, pre-commit
- `terraform-module-documentation` — terraform-docs README, CHANGELOG, ServiceDoc
- `terraform-style-and-formatting` — HashiCorp style guide application
- `terraform-module-versioning` — semantic versioning, branching, releases

#### Agent-native (schema and version)
- `terraform-schema-lookup` — verify resource arguments against the live provider schema (`terraform providers schema -json`) before authoring HCL; multi-version aware
- `terraform-provider-upgrade` — migrate modules across provider versions via schema-diff with three-tier confidence scoring (mechanical / likely-safe / needs-review) and CHANGELOG-aware decisions

#### Decision and judgment
- `terraform-dynamic-and-conditionals` — when to use `dynamic` vs inline, `count` vs `for_each`, conditional resource patterns
- `terraform-state-and-lifecycle` — `moved` / `import` / `removed` blocks, `lifecycle.ignore_changes` vs `prevent_destroy` (use Azure Resource Locks), `replace_triggered_by`, `terraform state mv`, `name_override` refactor pattern

#### Testing
- `terraform-module-testing` — full testing pyramid: static analysis (tflint, checkov, tfsec), plan-time tests (native `terraform test`, `mock_provider` 1.7+), integration (Terratest), negative tests; with `references/static-analysis.md`, `references/terraform-test.md`, `references/terratest.md`

#### Azure-specific
- `terraform-azure-patterns` — Azure-specific cross-cutting patterns (Key Vault, RBAC, networking, diagnostics, AzAPI), with `references/key-vault.md`, `references/rbac-and-identity.md`, `references/networking.md`, `references/diagnostics.md`

#### Review and remediation
- `terraform-azure-module-review` — five-pass PR review methodology, severity calibration, smell-tests catalog, with `references/review-methodology.md`, `references/review-format.md`, `references/smell-tests.md`
- `terraform-module-fix-advisory` — classify review findings by breaking-change risk and produce an ordered migration plan with `moved` blocks and version-bump guidance

#### Plugin bundles in `.claude-plugin/marketplace.json`
- `terraform-azure-module-skills` — all 18 skills
- `terraform-module-core-skills` — 15 cloud-agnostic skills
- `terraform-azure-skills` — 3 Azure-specific skills
- `terraform-agent-essentials` — 6-skill agent-value-first subset

#### Repository
- `AGENTS.md` with maintainer guidance and authoring rules
- 3 GitHub issue templates (skill content bug, enhancement, new skill request)
- GitHub Actions workflow validating `marketplace.json` paths and `SKILL.md` frontmatter
- MIT LICENSE
