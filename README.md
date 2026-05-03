# Terraform Azure Module Skills

Agent Skills for authoring, reviewing, upgrading, and refactoring production-grade Azure Terraform modules. Built for use with Claude Code, Cursor, GitHub Copilot, and any other agent runtime that supports the [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills) format.

The collection encodes hard-won patterns from real Terraform module development — variable design, naming, state safety, Azure-specific gotchas, multi-layer testing, a five-pass PR review methodology, schema-driven authoring, and provider-version migration. Each skill is self-contained and focused on one concern.

## Installation

### Claude Code

Add the marketplace, then install the bundle of your choice:

```bash
/plugin marketplace add isaacoluwade/terraform-azure-module-skills
/plugin install terraform-azure-module-skills@terraform-azure-module-skills
```

For a smaller install, use one of the focused bundles:

```bash
# Cloud-agnostic Terraform module patterns (no Azure-specific content)
/plugin install terraform-module-core-skills@terraform-azure-module-skills

# Azure-specific subset (orchestrator + Azure patterns + review)
/plugin install terraform-azure-skills@terraform-azure-module-skills

# Agent-essentials subset: schema lookup, upgrade, lifecycle, testing, fix advisory
/plugin install terraform-agent-essentials@terraform-azure-module-skills
```

### Other agents

The `skills/` directory follows the standard Agent Skills layout. Each skill is a directory containing a `SKILL.md` (with YAML frontmatter) and an optional `references/` subdirectory. Most agent runtimes that support Agent Skills can ingest these directly — copy the relevant skill folder into your agent's skills location, or point your agent at this repo.

## Bundles

| Bundle | Skills | Use when |
|---|---|---|
| `terraform-azure-module-skills` | 18 (all) | You author or review Azure Terraform modules end to end. |
| `terraform-module-core-skills` | 15 | You author Terraform modules for any cloud — cross-cutting authoring, testing, refactoring, and migration patterns without Azure-specific content. |
| `terraform-azure-skills` | 3 | You only want the Azure-specific guidance (orchestrator + Azure patterns + review). |
| `terraform-agent-essentials` | 6 | You want the agent-value-first subset: schema lookup, provider upgrade, dynamic/conditionals decisions, state and lifecycle judgment, multi-layer testing, fix advisory. |

## Skills

### Orchestrator

| Skill | Purpose |
|---|---|
| [`terraform-azure-module`](skills/terraform-azure-module) | The "start here" skill: module scaffolding workflow, file layout, creation checklist, routing to the topical skills below. |

### Per-file authoring

| Skill | Owns |
|---|---|
| [`terraform-variable-design`](skills/terraform-variable-design) | `variables.tf` — typing, defaults, validation, `optional()`, descriptions, grouping, `name_override` |
| [`terraform-locals-patterns`](skills/terraform-locals-patterns) | `locals.tf` — location maps, `primary_name`, tags merging, `module_version` |
| [`terraform-module-outputs`](skills/terraform-module-outputs) | `outputs.tf` — descriptions, no module-name prefix, sensitivity, output composition |
| [`terraform-provider-versioning`](skills/terraform-provider-versioning) | `versions.tf` — provider constraints (`>=` vs `~>`), `required_providers` |

### Cross-cutting authoring

| Skill | Owns |
|---|---|
| [`terraform-resource-naming`](skills/terraform-resource-naming) | Azure resource naming and HCL identifier naming conventions |
| [`terraform-module-structure`](skills/terraform-module-structure) | Directory layout, `examples/main/`, file naming for logical groupings, pre-commit |
| [`terraform-module-documentation`](skills/terraform-module-documentation) | `terraform-docs` README generation, CHANGELOG, ServiceDoc, pipeline integration |
| [`terraform-style-and-formatting`](skills/terraform-style-and-formatting) | HashiCorp style guide application — alignment, indentation, comment style |
| [`terraform-module-versioning`](skills/terraform-module-versioning) | Semantic versioning for modules, branching, release process |

### Agent-native (schema and version)

| Skill | Owns |
|---|---|
| [`terraform-schema-lookup`](skills/terraform-schema-lookup) | Verify resource arguments against the live provider schema (`terraform providers schema -json`) before authoring HCL — multi-version aware, ground-truth over training-data recall |
| [`terraform-provider-upgrade`](skills/terraform-provider-upgrade) | Migrate a module across provider versions via schema-diff: three-tier confidence (mechanical / likely-safe / needs-review) with CHANGELOG-aware decisions |

### Decision and judgment

| Skill | Owns |
|---|---|
| [`terraform-dynamic-and-conditionals`](skills/terraform-dynamic-and-conditionals) | When to use `dynamic` vs inline blocks, `count` vs `for_each`, conditional resource patterns — pure decision rules with paired bad/good examples |
| [`terraform-state-and-lifecycle`](skills/terraform-state-and-lifecycle) | `moved`, `import`, `removed` blocks; `lifecycle.ignore_changes` vs `prevent_destroy` (use Azure Resource Locks); `replace_triggered_by`; `terraform state mv`; `name_override` refactor pattern |

### Testing

| Skill | Owns |
|---|---|
| [`terraform-module-testing`](skills/terraform-module-testing) | The full testing pyramid: static analysis (tflint, checkov, tfsec), plan-time tests (native `terraform test`, `mock_provider` 1.7+), integration (Terratest), negative tests |

### Azure-specific

| Skill | Owns |
|---|---|
| [`terraform-azure-patterns`](skills/terraform-azure-patterns) | Azure-specific cross-cutting patterns: data-source avoidance, Key Vault integration, RBAC delegation, identity, diagnostics, AzAPI, private endpoints, lifecycle |

### Review and remediation

| Skill | Owns |
|---|---|
| [`terraform-azure-module-review`](skills/terraform-azure-module-review) | Five-pass PR review methodology, severity calibration, smell-tests catalog, review output format |
| [`terraform-module-fix-advisory`](skills/terraform-module-fix-advisory) | After a review, classify findings by breaking-change risk (🔴 Breaking / 🟡 State-Sensitive / 🟢 Safe / ⚪ Advisory) and produce an ordered migration plan with `moved` blocks and version-bump guidance |

## Repository structure

```
terraform-azure-module-skills/
├── README.md                       # this file
├── AGENTS.md                       # maintainer guidance for editing this repo
├── CHANGELOG.md                    # public release history
├── LICENSE                         # MIT
├── .claude-plugin/
│   └── marketplace.json            # plugin bundle definitions
├── .github/
│   ├── ISSUE_TEMPLATE/             # bug, enhancement, new-skill-request
│   └── workflows/                  # marketplace.json validation
└── skills/
    ├── terraform-azure-module/
    ├── terraform-azure-module-review/
    ├── terraform-azure-patterns/
    ├── terraform-dynamic-and-conditionals/
    ├── terraform-locals-patterns/
    ├── terraform-module-documentation/
    ├── terraform-module-fix-advisory/
    ├── terraform-module-outputs/
    ├── terraform-module-structure/
    ├── terraform-module-testing/
    ├── terraform-module-versioning/
    ├── terraform-provider-upgrade/
    ├── terraform-provider-versioning/
    ├── terraform-resource-naming/
    ├── terraform-schema-lookup/
    ├── terraform-state-and-lifecycle/
    ├── terraform-style-and-formatting/
    └── terraform-variable-design/
```

Each skill directory contains a `SKILL.md` (YAML frontmatter + markdown body) and an optional `references/` directory with deeper material loaded as needed.

## Compatibility

These skills target:
- **Terraform** ≥ 1.5 (some skills note features available only in 1.6+, 1.7+)
- **AzureRM provider** ≥ 4.0
- **Go** ≥ 1.21 (for Terratest examples)

Most patterns work on older versions; provider-specific argument names assume `azurerm` 4.x. The schema-lookup and provider-upgrade skills are version-aware and work against whatever version the target module pins.

## Design philosophy

These skills are written for AI agents writing Terraform modules, not for human reference. Two principles drive the content:

1. **Route the agent to ground-truth tools** instead of caching facts that go stale (provider arguments, deprecated attributes, resource shape). The `terraform-schema-lookup` skill makes `terraform providers schema -json` the canonical source. Don't trust training-data recall when the provider binary can answer authoritatively.
2. **Encode judgment the agent can't derive from code alone.** When to use `dynamic` vs inline. When `prevent_destroy` is wrong (almost always — use Azure Resource Locks). What "Critical severity" means vs "Breaking change". These are the decisions agents get wrong by default; the skills correct them.

Skills that just restate facts the agent already half-knows are deliberately absent from this collection.

## Contributing

Issues and PRs welcome. See [`.github/ISSUE_TEMPLATE/`](.github/ISSUE_TEMPLATE) for guided forms.

When proposing changes to a skill:
- Read the skill's `SKILL.md` and any `references/` first
- Read [`AGENTS.md`](AGENTS.md) for the repo's authoring conventions
- Keep `SKILL.md` under ~500 lines; push deep recipes into `references/`

## License

[MIT](LICENSE) © Isaac Oluwade
