# AGENTS.md

This file provides guidance to coding agents (Claude Code, Cursor, GitHub Copilot, etc.) working with content in this repository.

## Communication Style

- **No fluff.** Skip phrases like "Thanks for the thoughtful feedback!", "Happy to help!", "I appreciate...", etc.
- **No soliciting opinions.** Don't end responses with "Would you like me to...", "Let me know if...", "What do you think?"
- **Direct and terse.** State facts, provide options when relevant, then stop.

## What this repo is

A maintained collection of Agent Skills for authoring and reviewing production-grade Terraform modules, with a focus on Azure (`azurerm` provider) but with most patterns applicable to any cloud.

This is **not a Terraform module itself** and not a buildable artifact. The work here is **authoring, refining, and reorganizing skill content** so the skills are accurate, discoverable, and useful to future agents.

## Core repo shape

Treat this as a content + metadata repository:
- `skills/<skill>/SKILL.md` — canonical skill instructions
- `skills/<skill>/references/*.md` — deeper examples, recipes, and reference material
- `.claude-plugin/marketplace.json` — source of truth for plugin bundle membership and metadata
- `README.md` — public catalog and distribution document
- `CHANGELOG.md` — public-surface history for new skills, restructures, and bundle changes

There is no normal app-style build or test loop here. Validation is mostly editorial and structural.

## Required context before any skill change

Before editing a skill, creating a new one, or revising repo guidance, load and follow Anthropic's Agent Skills best practices:
- <https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices.md>

Treat that document as required context for work in this repository, not optional background reading.

The rules below are repo-specific reminders, not a replacement for the source document.

## Repo-specific authoring rules

- Keep `SKILL.md` concise — target under 500 lines. Push long examples and deep detail into local `references/` files.
- Use progressive disclosure. Important reference files should be linked directly from `SKILL.md`.
- Keep skills self-contained. A skill may *mention* a sibling skill, but it should not *require* another skill's files to be useful.
- Preserve topic boundaries between sibling skills instead of recombining broad topics.
- Prefer one strong modern default over presenting many competing approaches.
- Keep guidance focused on current Terraform (≥ 1.5) and current `azurerm` provider (≥ 4.0). Avoid deprecated patterns unless the skill is explicitly about migration.
- When a skill changes substantially, sanity-check it with realistic prompts to make sure the description, scope, and guidance still work well in practice.

## Ground claims in official sources

This repo is about Terraform and Azure, so skill content should be grounded in official sources whenever possible:

- HashiCorp Terraform documentation: <https://developer.hashicorp.com/terraform>
- HashiCorp Style Guide: <https://developer.hashicorp.com/terraform/language/style>
- AzureRM Provider Reference: <https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs>
- Azure Verified Modules: <https://azure.github.io/Azure-Verified-Modules/>
- Microsoft Azure documentation: <https://learn.microsoft.com/azure>

Re-check official docs when updating resource arguments, provider behavior, or best-practice recommendations. Provider arguments and Azure resource shapes change between releases.

## Skill metadata matters

Frontmatter on each `SKILL.md` is important for discovery and routing.

- Keep `name` aligned with the folder name.
- Make `description` specific about what the skill covers and when it should be used.
- Favor descriptions that help an agent choose the right skill among nearby alternatives. Mention the file (`variables.tf`, `versions.tf`, etc.) or concept the skill owns.

## Structure patterns to preserve

Most skills in this repo follow a stable pattern. Preserve it unless there is a strong reason not to:

- clear opening scope statement (1–3 sentences, including scope boundary vs sibling skills)
- `## Contents`
- main guidance sections, each with HCL examples
- `## Common Mistakes`
- `## Review Checklist`
- `## References` when supporting material exists

## Important boundaries

Do not casually collapse existing splits between sibling skills. For example:
- `terraform-variable-design` owns `variables.tf` patterns (typing, defaults, validation, optionals)
- `terraform-locals-patterns` owns `locals.tf` (location maps, primary_name, tags merging)
- `terraform-module-outputs` owns `outputs.tf` (descriptions, no module-name prefix, sensitivity)
- `terraform-resource-naming` owns Azure resource and HCL identifier naming
- `terraform-azure-patterns` owns Azure-specific cross-cutting patterns (Key Vault, RBAC, private endpoints, diagnostics)
- `terraform-azure-module-review` owns review *methodology*; the topical skills own the *standards* being checked

Some overlap between skills is intentional so each skill remains useful on its own.

## Maintainer sync checklist

If you add, remove, rename, or significantly retarget a skill, update the related repo surfaces together:
- the skill folder under `skills/`
- `.claude-plugin/marketplace.json` (every bundle that references the skill)
- `README.md` when the public catalog, descriptions, or counts change
- `CHANGELOG.md` when the public surface changes

When reviewing changes, check:
- frontmatter is still valid and `name` matches the folder
- referenced paths still resolve
- bundle membership still matches the intended taxonomy
- public docs stay consistent with repo contents

## HCL examples

- Use real, runnable HCL where possible — modulo placeholder values like `<your-resource-group>`.
- Show the modern `optional()` form for object-typed variables; do not use `try()`-based defaulting where `optional()` is available.
- Prefer current `azurerm` resource argument names; provider 4.x renamed and removed several arguments from 3.x.
- Avoid examples that depend on internal/private modules. Where a delegating module is referenced (e.g. an authorization module), use a generic placeholder like `module "authorization"` and explain the pattern.

## Placeholder conventions

This repo's content was authored generically. When examples need a stand-in, use:
- `acme` — example organization or brand code
- `<your-terraform-registry>` — private Terraform registry
- `<your-azure-devops-org>` / `<your-ado-project>` — Azure DevOps references
- `<your-pipeline-templates>` — shared pipeline template repository
- `<your-terratest-helpers>` — shared Terratest helper module
- `eastus`, `westus`, `northeurope` — example Azure regions (avoid hardcoding regional defaults that imply a specific business)

If you find a real organization name, internal URL, or person's name in a skill file, treat it as a bug and replace it with the appropriate placeholder.
