---
name: terraform-azure-module-review
description: Expert-level pull request review for Terraform Azure modules. Use this skill whenever reviewing a Terraform module PR, diff, branch, or code change in any `terraform-azurerm-*` repository — even small ones, since small Terraform changes can produce large state and runtime impacts. Defines a five-pass methodology (Structural Scan, Standards Compliance, Logic & Correctness, Architecture & Design, Cross-Cutting Concerns), severity calibration (Critical / Major / Minor / Informational), how to write Critical findings via causal-chain tracing from bad inputs to crash points, the review output format with finding IDs and code-suggestion blocks, and a smell-tests catalog. Triggers on "review this Terraform PR", "review this module", "check this before merge", "is this Terraform code ready", PR URLs for `terraform-azurerm-*` repos, or any diff containing `.tf` files. The sibling authoring skills define **what** the standards are; this skill defines **how to review** against them.
---

# Terraform Azure Module Review

Expert-level review of Terraform Azure module pull requests. The sibling authoring skills define **what** good Terraform looks like. This skill defines **how to review** — the methodology, the questioning approach, the severity calibration, and the adversarial thinking that separates a rubber-stamp review from one that catches real problems.

For the standards being checked, follow the cross-references in the [Standards Reference](#standards-reference) table — each topical skill owns its own rules.

## Contents
- [How This Skill Differs from the Topical Authoring Skills](#how-this-skill-differs-from-the-topical-authoring-skills)
- [Expert Reviewer Mindset](#expert-reviewer-mindset)
- [Review Workflow Overview](#review-workflow-overview)
- [Severity Calibration](#severity-calibration)
- [How to Write Critical Findings](#how-to-write-critical-findings)
- [Review Output Format](#review-output-format)
- [Smell Tests Overview](#smell-tests-overview)
- [Standards Reference](#standards-reference)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [Getting Started](#getting-started)
- [References](#references)

## How This Skill Differs from the Topical Authoring Skills

The topical skills (`terraform-variable-design`, `terraform-module-outputs`, `terraform-locals-patterns`, etc.) define **what** the standards are. They are the rulebook.

This skill defines **how to apply that rulebook** — the multi-pass methodology, the questioning approach, the severity calibration, the causal-chain tracing for Critical findings, and the adversarial thinking that catches real bugs rather than rubber-stamping a diff.

When reviewing, always read the relevant topical skill for the standards being checked. This skill teaches you how to apply them with the depth and skepticism of a senior Terraform reviewer.

## Expert Reviewer Mindset

A senior reviewer brings these traits to every PR. Internalize them:

1. **Question everything.** Don't accept code at face value. Ask "why?" about every design decision. If you can't articulate a good reason for a pattern, it's probably wrong.

2. **Zero tolerance for dead code.** Every variable, local, output, data source, and resource must be referenced. "Declared but not used" is the most common review comment for a reason.

3. **No deferred fixes on new modules.** If the module is new (version `1.0.0`), there is no excuse for TODOs, known issues, or "I'll fix it later" comments. Get it right the first time.

4. **Demand exact patterns.** When a standard pattern exists in this skill collection, the module must use it exactly. Don't approximate, don't reinvent, don't "improve" the standard.

5. **Simplify relentlessly.** Every layer of indirection, every defensive wrapper, every clever trick must justify its existence. Prefer `try()` over `coalesce(try())`, direct blocks over unnecessary `dynamic`, simple attribute access over layered fallbacks.

6. **Think about the consumer.** Every PR change affects downstream workspaces. Will this break existing `terraform plan`? Will state need to be moved? Does the interface make sense to someone who hasn't read the source code?

7. **Cross-module responsibility.** Each module owns exactly one concern. Authorization belongs in the authorization module. Identity belongs in centralized SPN management. Data lookups should become explicit variable inputs. When you see scope creep, call it out.

8. **Suggest with code.** Don't just say "this is wrong" — provide the exact corrected code in a `Suggested change` block. This eliminates ambiguity and accelerates the fix.

## Review Workflow Overview

The review is structured as **five sequential passes**, each building on the previous:

```
Pass 1: Structural Scan          Files, layout, naming, completeness
Pass 2: Standards Compliance     Variable design, locals, outputs, providers
Pass 3: Logic & Correctness      Resource behavior, conditional guards, state safety
Pass 4: Architecture & Design    Responsibility boundaries, consumer experience, patterns
Pass 5: Cross-Cutting Concerns   Security, testing, documentation, pipeline, examples
```

Each pass has specific questions to ask and patterns to look for. Read [`references/review-methodology.md`](references/review-methodology.md) before starting any review — that document is the working procedure.

Do not skip passes. Cross-cutting issues in Pass 4 and Pass 5 often reveal problems that file-level passes miss.

## Severity Calibration

Use these severity levels precisely. Over-severity erodes trust. Under-severity misses real bugs.

| Severity | Criteria | Examples |
|----------|----------|----------|
| Critical | Will cause runtime failure, data loss, or security vulnerability. Blocks merge. | Null reference in required argument, missing count guard on conditional resource, secrets in plaintext |
| Major | Violates a documented standard in a way that creates technical debt or confuses consumers. Should block merge. | Wrong `primary_name` format, missing standard variables, incorrect tags merge order, data source instead of variable input |
| Minor | Convention deviation or minor improvement. Fix recommended but doesn't block merge. | Description missing period, `deployed_using` format, example directory naming |
| Informational | Positive observation or optional suggestion. No action needed. | Good pattern usage, clean naming, well-structured conditionals |

### Severity Rules

- **When in doubt, go one level higher.** It's better to flag a Major issue as Critical than to miss a real problem by underrating it.
- **Count guards on conditional resources are always Critical.** A missing `count` on a conditional data source or resource means the module fails for any consumer who doesn't enable that feature.
- **Standard variable/local deviations are Major**, not Minor. They affect every module consumer and break consistency across the organization.
- **State-breaking changes without migration paths are Critical.** Renaming resources without `moved` blocks, changing `count` to `for_each`, or changing variable types.
- **Formatting-only issues are always Minor or Informational.** Description missing a trailing period, heredoc style preferences, blank line placement — these never rise above Minor. They do not affect runtime behavior or consumer experience. Do not inflate them.
- **Separate distinct problems into distinct findings.** A finding that bundles "standard variables are wrong AND VERSION file is missing AND CHANGELOG says 0.1.0" is three separate issues at three different severities. Keep findings atomic — one problem, one finding, one severity.

## How to Write Critical Findings

Critical findings are the most important part of your review. They predict runtime failures the developer hasn't seen yet.

**The right approach: trace bad inputs to their crash point.**

Don't just say "this variable has a bad default." Follow the data:

1. The variable `vnet_id` defaults to `""`.
2. This flows into `local.role_assignments` as a map value.
3. Which feeds `for_each` on `azurerm_role_assignment.network_contributor`.
4. Azure API receives `scope = ""` and returns a cryptic 400 error.
5. The consumer has no idea what went wrong.

This is what a senior reviewer does — they trace the **causal chain** from bad input to runtime failure. The finding isn't "variable has wrong default" (that's Major). The finding is "this default **will cause** an Azure API failure with an unhelpful error message when the consumer omits the variable" (that's Critical).

**Signs you have a real Critical:**
- You can describe a specific consumer action that triggers a failure.
- The failure happens at `apply` time (not `plan` time) — making it expensive.
- The error message doesn't point to the actual cause.
- Data loss or security exposure is possible.

If you can't trace from a bad input to a specific crash point, you probably don't have a Critical — you have a Major.

## Review Output Format

Read [`references/review-format.md`](references/review-format.md) for the exact output template.

### Key Principles for Review Output

1. **Every finding must cite the exact file and line/block** where the issue occurs.
2. **Every finding must reference the specific standard** being violated, with a quote and a pointer to the topical skill that owns it.
3. **Every finding above Informational must include a code fix** — either inline or as a `Suggested change` block.
4. **Group findings by file**, not by severity — this makes it easier for the developer to fix things file by file. Within a file, order by severity (Critical first).
5. **Include positive observations** — acknowledge what's done well; this builds trust and helps the developer understand which patterns to keep using.
6. **End with a clear verdict**: Approve, Approve with Comments, Request Changes, or Needs Major Revision. If there is any Critical finding, the verdict is Request Changes — no exceptions.

## Smell Tests Overview

Read [`references/smell-tests.md`](references/smell-tests.md) for the complete catalog of smells (around thirty patterns) and the questions to ask when you encounter each one.

These are patterns that should trigger deeper investigation:

| Smell | Question to Ask |
|-------|-----------------|
| `data "azurerm_*"` or `data "azuread_*"` | Why is this looking up a value instead of accepting it as an input? |
| `lower()`, `upper()`, `title()` | Why is this transforming the value instead of validating the input? |
| `coalesce(try(..., null), ...)` | Can this be simplified to just `try()` or `lookup()`? |
| `default = ""` on a string variable | Should this be `default = null`? What is the semantic difference here? |
| `default = null` on a complex variable | Should this be `default = {}` with `nullable = false`? |
| `var.identity.client_id` / `client_secret` as raw strings | Should credentials come from Key Vault instead of plaintext variables? |
| `sensitive = false` | Why is this declared? It is the default — remove it. |
| `depends_on = [...]` | Is there truly no attribute reference that creates the implicit dependency? |
| `dynamic` block | Is this necessary? Could a direct block or simple conditional work instead? |
| `for_each` with hardcoded keys | Are the keys stable across applies? Will adding/removing items cause recreation? |
| Module creating `azurerm_role_assignment` | Should this be delegated to the authorization module? |
| Module creating `azuread_service_principal` | SPNs are centrally managed — why is this here? |
| Resource named something other than `main` | Is there genuinely more than one of this resource type? |
| File named `terraform.tf` | The standard is `versions.tf` for the terraform/provider block. |
| Variable without `validation {}` | Can the input be constrained? Would bad input cause a confusing error? |
| Output prefixed with module name | The module name already provides namespace — remove the prefix. |
| `lifecycle { ignore_changes = [tags] }` | This hides ALL tag drift — should it target specific keys? |
| `lifecycle { prevent_destroy = true }` | Variables can't control lifecycle — use Azure management locks instead. |
| `count = var.something ? 1 : 0` without checking related resources | Are ALL resources/data sources that depend on this feature also guarded? |

## Standards Reference

For the actual standards being checked, refer to the topical authoring skills in this collection:

| Topic | Skill |
|-------|-------|
| Variable design rules | `terraform-variable-design` |
| Output conventions | `terraform-module-outputs` |
| Locals patterns | `terraform-locals-patterns` |
| File and folder structure | `terraform-module-structure` |
| Resource and identifier naming | `terraform-resource-naming` |
| Provider versioning | `terraform-provider-versioning` |
| Azure-specific patterns | `terraform-azure-patterns` |
| Style and formatting rules | `terraform-style-and-formatting` |
| Testing patterns | `terraform-module-testing` |
| Documentation standards | `terraform-module-documentation` |
| Module versioning rules | `terraform-module-versioning` |
| End-to-end module orchestration | `terraform-azure-module` |

When a finding cites a standard, name the topical skill explicitly — e.g. "for variable design rules see the `terraform-variable-design` skill." Don't reference file paths.

## Common Mistakes

These are anti-patterns specific to the **review process itself** — not the modules being reviewed. The topical skills cover module anti-patterns.

1. **Severity inflation.** Marking a missing trailing period as Major because you found a lot of them. Cosmetic issues stay Minor. Counting findings is not a substitute for calibrating them.

2. **Severity deflation on conditional guards.** A missing `count` on a conditional resource is always Critical. Reviewers sometimes mark this Major because "it would only break if someone disabled the feature" — that *is* the failure mode.

3. **Bundling unrelated issues into one finding.** "Standard variables are wrong AND VERSION is missing AND CHANGELOG says 0.1.0" is three findings at three severities. Each finding describes one problem with one root cause.

4. **Calling something Critical without tracing the causal chain.** A bad default on a string variable is Major on its own. It becomes Critical only when you can trace it through locals into a resource argument that produces a runtime failure with a confusing error. If you can't write the trace, downgrade.

5. **Grouping findings by severity instead of by file.** Developers fix code file by file. A review that lists all Criticals first, then all Majors across the whole module, forces the developer to jump between files. Group by file; order by severity within each file.

6. **No code-suggestion block.** A finding without a suggested fix forces the developer to re-derive the solution. Every Critical and Major finding includes a `Suggested change` block or a complete corrected snippet.

7. **No positive observations.** A review with only criticism doesn't tell the developer which patterns to keep using. Note 3-5 positive observations on every review, even on a poor PR.

8. **Skipping passes.** Pass 4 (Architecture) and Pass 5 (Cross-Cutting) are where responsibility-boundary and security findings come from. A review that stops at Pass 2 is a checklist, not an expert review.

9. **Not citing exact files and lines.** "There's a problem in `main.tf`" is unactionable. Every finding includes the file, the resource/variable/local/output name, and the approximate line.

10. **Citing standards by file path instead of skill name.** Findings should reference the owning topical skill (e.g. `terraform-variable-design`), not a file path. The topical skill is the durable reference.

11. **Approving with Critical findings present.** If there is any Critical finding, the verdict is Request Changes. There are no exceptions to this rule.

12. **Hedging language.** "Perhaps you might want to consider possibly using a variable." State the issue directly. Soften with the fix, not with weasel words.

13. **Missing the conditional chain check.** Finding one missing guard but not tracing the rest of the toggle's chain. For each boolean toggle, every dependent resource, data source, local, and output must be verified — not just the one you noticed.

14. **No verdict, or a verdict that contradicts the findings.** Every review ends with one of the four verdicts, and that verdict matches the findings. Critical present → Request Changes.

## Review Checklist

A meta-checklist for the reviewer. Before submitting any review, verify:

- [ ] Did I run all five passes (Structural, Standards, Logic, Architecture, Cross-Cutting)?
- [ ] Did I trace the causal chain for every Critical finding from bad input to runtime crash?
- [ ] Did I cite specific files and line numbers (not just "in `main.tf`")?
- [ ] Did I name the owning topical skill for each standards-based finding (not a file path)?
- [ ] Did I verify the conditional chain for every boolean toggle in the module?
- [ ] Are findings grouped by file, with severity ordering inside each file?
- [ ] Does every Critical and Major finding include a code-suggestion block?
- [ ] Are findings atomic — one problem per finding, not bundled mega-findings?
- [ ] Are formatting-only issues (periods, heredoc style) correctly rated Minor?
- [ ] Are missing-guard findings correctly rated Critical?
- [ ] Did I include 3-5 positive observations?
- [ ] Does the verdict match the findings (Critical present → Request Changes)?
- [ ] Did I question every data source and explain why it shouldn't be a variable input?
- [ ] Did I verify credentials are referenced through Key Vault, not as plaintext variable values?
- [ ] Did I check that the module doesn't create SPNs or manage role assignments directly?
- [ ] Would I be comfortable defending every severity rating I assigned?

If you can answer yes to all of these, submit the review.

## Getting Started

When asked to review a PR, module, or Terraform code change:

1. **Identify the scope.** Is this a new module (`1.0.0`)? A minor change? A major version bump? This determines how strict the review should be. New modules get zero tolerance.

2. **Read the methodology.** Open [`references/review-methodology.md`](references/review-methodology.md) and follow the five passes.

3. **Load the relevant standards.** Based on which files are changed, read the corresponding topical skills (see the [Standards Reference](#standards-reference) table).

4. **Execute each pass.** Work through systematically. Don't skip passes — cross-cutting issues in Pass 4 and Pass 5 often reveal problems that file-level passes miss.

5. **Calibrate severity.** Use the table in [Severity Calibration](#severity-calibration). Review your findings and adjust — a common mistake is marking standard variable issues as Minor when they're actually Major.

6. **Format the output.** Use the template from [`references/review-format.md`](references/review-format.md).

7. **State your verdict.** Be clear. If there are any Critical findings, the verdict is Request Changes — no exceptions.

## References

- [`references/review-methodology.md`](references/review-methodology.md) — The full five-pass methodology with file-by-file checklists, the data-flow tracing technique, and the post-review synthesis steps.
- [`references/review-format.md`](references/review-format.md) — The output template, finding ID scheme (C-/M-/m-/I-), citation rules, code-block requirements, and severity-consistency guardrails.
- [`references/smell-tests.md`](references/smell-tests.md) — The catalog of code smells organized by tier (Almost Always Wrong / Often Wrong / Context-Dependent), with the question to ask and the corrective pattern for each.
