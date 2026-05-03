---
name: terraform-module-fix-advisory
description: After a module review produces findings, classify each finding by breaking-change risk and produce a prioritised migration plan. Pairs with `terraform-azure-module-review`. Sorts findings into four classes — 🔴 Breaking, 🟡 State-Sensitive, 🟢 Safe, ⚪ Advisory — independent of the reviewer's severity (Critical/Major/Minor). Emits the safe order to apply fixes (safe first, state-sensitive next, breaking last in a versioned release), the exact `moved`/`removed` blocks needed, the version bump level, and the CHANGELOG entries. Use this skill whenever you ask "what's the safe order to fix these review findings", "is this fix breaking", "produce a migration plan from this review", "classify these review findings", "what version bump do these changes need", or when planning a major version cut. Skip this skill at your peril — applying a Critical-but-Breaking fix without a `moved` block destroys infrastructure.
---

# Terraform Module Fix Advisory

A module review tells you *what's wrong*. This skill tells you *the safe order to fix it*. Severity (how bad the problem is) and Class (how risky the fix is) are independent axes. A Critical-severity finding can be a Safe fix; a Minor-severity finding can be a Breaking fix. The agent's natural instinct after a review is to start with the highest-severity items — for a production module, that order is wrong. Apply low-risk fixes first to clean the baseline, state-sensitive fixes next with their migration steps, and breaking fixes last in a coordinated versioned release.

For state mutation mechanics (`moved`, `removed`, `import`, `terraform state mv`), see `terraform-state-and-lifecycle`. For version bump rules and CHANGELOG format, see `terraform-module-versioning`. This skill *cites* those mechanics — it does not duplicate them.

## Contents
- [When to Invoke](#when-to-invoke)
- [The Four Classes](#the-four-classes)
- [Severity vs Class — They Are Independent](#severity-vs-class--they-are-independent)
- [Classification Lookup Table](#classification-lookup-table)
- [Order of Operations](#order-of-operations)
- [Required Output for 🔴 Breaking Items](#required-output-for--breaking-items)
- [Required Output for 🟡 State-Sensitive Items](#required-output-for--state-sensitive-items)
- [The Migration Plan Output Template](#the-migration-plan-output-template)
- [Worked Example](#worked-example)
- [What This Skill Does Not Do](#what-this-skill-does-not-do)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)

## When to Invoke

Use this skill when:
- A module review (e.g. `terraform-azure-module-review`) has produced a list of findings and you need to convert them into an action plan.
- A user hands you a review report and asks "what's the safe order to apply these?"
- A user asks "is this fix breaking?" or "what version bump do these changes need?"
- You're planning a major version cut and need to gather everything that *must* land in the breaking release versus everything that can ship before it.
- A PR contains multiple fixes and you need to decide what can land now and what must wait for a coordinated release.

Do **not** invoke this skill to:
- Generate findings (that's the reviewer's job).
- Apply the fixes (this skill produces a plan, not a patch).
- Re-litigate the reviewer's severity (Critical/Major/Minor stays as the reviewer assigned it).

## The Four Classes

Every finding falls into exactly one of these four classes. The class is determined by **what happens when the fix is applied**, not by how serious the underlying issue is.

### 🔴 Breaking

Applying the fix causes a state-address change, destroys/recreates infrastructure, or breaks the module's input/output contract. Requires a `moved`/`removed` block (when the change is purely an address rename), a CHANGELOG `**Breaking:**` entry, a coordinated consumer upgrade, and almost always a **major** version bump.

Examples:
- Rename a resource label without a `moved` block (e.g. `azurerm_kubernetes_cluster.aks` → `azurerm_kubernetes_cluster.main`).
- Remove a variable.
- Remove an output.
- Change a variable's `type` incompatibly (e.g. `string` → `object({ ... })`, `list(string)` → `set(string)` when downstream code indexes positionally).
- Switch `count` → `for_each` without a `moved` block.
- Add a **required** variable (no default) to an existing module.
- Narrow a variable's `validation` allowed-values list so values previously accepted are now rejected.
- Rename an output (renaming an output is a contract break; consumers reference it by name).
- Change the default of a variable in a way that flips behaviour (e.g. `enable_diag_settings` default `false` → `true` — every existing caller relying on the old default now creates new infrastructure on the next apply).
- Increase the floor of `required_providers` or `required_version` such that existing consumers' lockfiles fail.

### 🟡 State-Sensitive

Does **not** destroy infrastructure but mutates state addresses or metadata. Safe **only** if applied with the right migration step. Wrong order of operations turns these into 🔴 Breaking by accident.

Examples:
- Rename a resource label *with* a `moved` block (safe **only if** the `moved` block lands in the **same apply** as the rename).
- Switch `count` → `for_each` paired with a `moved` block per migrated key.
- Add a new variable that is logically required but ships with a default that matches the implicit prior behaviour (so the contract doesn't break).
- Add `nullable = false` to a variable that callers may currently be passing `null` to (no state effect, but breaks callers — handle as a deprecation cycle).
- Move a resource from the root module into a submodule (or vice-versa) with `moved` blocks for each address.

### 🟢 Safe

No state impact. Can be applied in any order, in any commit, at any time. These fixes establish a clean baseline so the riskier fixes apply against a quiet `terraform plan`.

Examples:
- Add or fix a `description` on a variable, output, or resource.
- Reorder attributes within a block.
- Fix a comment, formatting, or whitespace.
- Add a missing standard variable that has a default matching the implicit prior behaviour.
- Add `lifecycle { ignore_changes = [tags] }` (no state effect on its own).
- Rename a `.tf` file (e.g. `output.tf` → `outputs.tf`) — file layout doesn't affect state.
- Refactor a `locals` block where the produced value is byte-identical.
- Add a `validation` block that matches **all** currently-accepted inputs (rejects nothing new).
- Tighten a `description` heredoc.

### ⚪ Advisory

Not a code change. An architectural or process recommendation that requires a team decision before action. Surface it; do not auto-apply.

Examples:
- "Split this module into two."
- "Add state locking to the backend."
- "Move this secret to Key Vault."
- "Introduce a CI pipeline / pre-commit hooks."
- "Adopt a release-versioning strategy."
- "Deprecate this submodule and consolidate into the parent."

## Severity vs Class — They Are Independent

The reviewer assigns **severity** (Critical / Major / Minor / Informational) — how bad the underlying issue is. This skill assigns **class** (🔴 / 🟡 / 🟢 / ⚪) — how risky the fix is. They are orthogonal.

| Finding | Reviewer Severity | Fix Class |
|---|---|---|
| "Resource label is `aks` — should be `main`." | Minor | 🔴 Breaking (or 🟡 with `moved`) |
| "Variable `location` has no default — every caller must supply it." | Critical | 🟢 Safe (add default that matches what callers all currently pass) |
| "Output `aro_cluster_id` should be `cluster_id`." | Minor | 🔴 Breaking (output rename = contract break) |
| "File is named `output.tf` — should be `outputs.tf`." | Minor | 🟢 Safe |
| "No validation on `vm_size` — accepts any string." | Major | 🟢 Safe (if the new validation accepts every value already in use) |
| "Whole module needs splitting in two." | Major | ⚪ Advisory |
| "`count` should be `for_each`." | Major | 🔴 Breaking (🟡 if paired with `moved`) |

The lesson: **never let severity drive apply order**. A Critical-but-Safe fix lands on Monday; a Minor-but-Breaking fix waits for the next major release.

## Classification Lookup Table

Apply this table mechanically. When in doubt, assume the more dangerous class.

| Review finding | Default class | Promote to 🟡/🔴 if… | Demote to 🟢 if… |
|---|---|---|---|
| Resource label is wrong (e.g. `aks` → `main`) | 🔴 Breaking | — | Paired with a `moved` block in the same apply → 🟡 |
| Output prefixed with module name (e.g. `aro_cluster_id`) | 🔴 Breaking | — | — (output rename always breaks consumers) |
| Output description missing trailing period | 🟢 Safe | — | — |
| Variable lacks a `validation` block | 🟢 Safe | New validation rejects values currently accepted → 🔴 | — |
| File named `output.tf` (singular) | 🟢 Safe | — | — (file rename has no state effect) |
| Standard variable missing (e.g. `tags`, `location`) | 🟡 State-Sensitive | Variable is required-no-default → 🔴 | Default matches what callers all implicitly use → 🟢 |
| Wrong tags merge order | 🟢 Safe | Merge produces a different output for at least one caller → 🟡 | — |
| `count` should be `for_each` | 🔴 Breaking | Paired with `moved` block per migrated key → 🟡 | — |
| Add `lifecycle { ignore_changes = [tags] }` | 🟢 Safe | — | — |
| Switch a variable's `type` (e.g. `string` → `object({...})`) | 🔴 Breaking (always) | — | — (type change is always a contract break) |
| Remove a variable | 🔴 Breaking (always) | — | — |
| Remove an output | 🔴 Breaking (always) | — | — |
| Add a new optional variable with a default | 🟢 Safe | — | — |
| Add a new **required** variable (no default) | 🔴 Breaking | — | Ship with a default that matches implicit prior behaviour → 🟡 or 🟢 |
| Rename a variable | 🔴 Breaking | Keep old name, add new name as alias, deprecate old → 🟡 | — |
| Rename an output | 🔴 Breaking | Keep old output, add new output, mark old deprecated → 🟡 | — |
| Reorder attributes within a block | 🟢 Safe | — | — |
| Move resource between `.tf` files | 🟢 Safe | — | — |
| Move resource into a submodule (state address changes) | 🔴 Breaking | Paired with `moved` blocks → 🟡 | — |
| Add `nullable = false` to existing variable | 🟡 State-Sensitive | Callers currently pass `null` → 🔴 (or do a deprecation cycle) | Variable has a non-null default and `null` was never valid → 🟢 |
| Tighten an existing `validation` rule | 🔴 Breaking | — | New rule still accepts every value currently passed → 🟢 |
| Add a `description` to a variable/output | 🟢 Safe | — | — |
| Refactor `locals` (output byte-identical) | 🟢 Safe | Output differs for any input → 🟡 | — |
| Architectural recommendation (split, restructure, adopt CI) | ⚪ Advisory | — | — |
| Backend / state-locking recommendation | ⚪ Advisory | — | — |
| Move secret to Key Vault | ⚪ Advisory | Already coded as a secret resource → 🔴 if currently outputting it | — |

## Order of Operations

This is the rule the agent follows every time. **Do not vary it.**

1. **🟢 Safe first.** Apply all safe fixes. They establish a clean baseline. Run `terraform plan` and confirm it shows **no diff** before proceeding. If plan shows a diff, one of your "safe" fixes was misclassified — stop and reclassify.
2. **🟡 State-Sensitive next.** Apply each with its migration step. For every renamed address, the `moved` block must land in the **same apply** as the rename. Verify each `terraform plan` shows the expected `# … has moved to …` lines and **zero destroys**. If a destroy appears, stop — the fix is actually 🔴 and needs a versioned release.
3. **🔴 Breaking last.** Apply in a single dedicated versioned release. The version bump, CHANGELOG entry, and `moved`/`removed` blocks all land in the same commit. Roll out to the lowest environment first (dev → test → prod). Consumers upgrade their `version = "..."` pin in lock-step.

Do not commingle. Do not bundle a 🟢 with a 🔴. Mixing classes in one commit makes the breaking change harder to revert and obscures the diff readers care about.

## Required Output for 🔴 Breaking Items

Every 🔴 finding in the migration plan **must** include all of the following — non-negotiable:

1. **The exact `moved` / `removed` block** to add. If the module's `required_version < 1.1.0`, supply a `terraform state mv` command instead and flag the version-floor problem.
   ```hcl
   moved {
     from = azurerm_kubernetes_cluster.aks
     to   = azurerm_kubernetes_cluster.main
   }
   ```
2. **The CHANGELOG entry text**, under the right section. Use `### Removed`, `### Changed`, or `### Added`, and prefix with `**Breaking:**`:
   ```markdown
   ### Changed
   - **Breaking:** rename resource `azurerm_kubernetes_cluster.aks` to `azurerm_kubernetes_cluster.main`. Consumers using `terraform_remote_state` lookups against this address must update.
   ```
3. **The version bump level**:
   - **Major** if the variable/output contract changes (variable removed/renamed, type changed, output removed/renamed, required variable added).
   - **Minor** if state migrates but the contract is intact (label rename with `moved`, `count`→`for_each` with `moved`).
   - Never **Patch** for a 🔴 item.
4. **The lowest environment to roll to first** (almost always `dev`), and the validation step (`terraform plan` shows the expected `moved` lines / zero destroys) before promoting.
5. **Any consumer-side migration step** (e.g. "consumer must bump `version = "~> 2.0"`", "consumer must update output reference from `module.x.aro_cluster_id` to `module.x.cluster_id`").

If any of these five are missing, the plan is incomplete — emit a `TODO: …` placeholder rather than silently dropping the requirement.

## Required Output for 🟡 State-Sensitive Items

Every 🟡 finding must include:

1. **A one-line "risk if applied wrong"** — what destroys/recreates if the migration step is forgotten.
2. **Ordered steps**, with the `moved` block (or equivalent) landing in the *same commit* as the change it covers — not a follow-up commit.
3. **The expected plan output** — e.g. `# azurerm_kubernetes_cluster.aks has moved to azurerm_kubernetes_cluster.main` and *zero destroy/create lines*. The agent applying the plan checks against this.
4. **Rollback note** — what to do if the plan looks wrong. Usually: revert the commit, do not apply, reclassify.

Note: 🟡 items often do **not** need a major version bump — a label rename with a `moved` block is a Minor bump because the public contract (variable names, output names, types) is unchanged. State-only changes are Minor; contract changes are Major.

## Worked Example

Here's how a small review report (4 findings) becomes a migration plan.

**Input findings (from `terraform-azure-module-review`):**

| # | Severity | Finding |
|---|---|---|
| 1 | Minor | Output `aro_cluster_id` should be `cluster_id` (drop module-name prefix). |
| 2 | Major | Resource label `azurerm_redhat_openshift_cluster.aro` should be `main`. |
| 3 | Minor | File is named `output.tf` — should be `outputs.tf`. |
| 4 | Major | Module needs to be split into `aro-cluster` and `aro-networking`. |

**Classification:**

| # | Severity | Class | Reasoning |
|---|---|---|---|
| 1 | Minor | 🔴 Breaking | Renaming an output breaks every consumer that references `module.x.aro_cluster_id`. |
| 2 | Major | 🟡 State-Sensitive | Safe with a `moved` block; without it, the cluster destroys/recreates. |
| 3 | Minor | 🟢 Safe | File rename has no state effect. |
| 4 | Major | ⚪ Advisory | Architectural decision — surface it, don't auto-apply. |

**Apply order:** 🟢 → 🟡 → 🔴 → ⚪ (decision).

**What lands when:**
- **Today (patch release):** rename `output.tf` → `outputs.tf`. `terraform plan` shows no diff. Done.
- **Next sprint (minor release):** add the `moved` block + rename the resource label, in one commit. Plan shows `# azurerm_redhat_openshift_cluster.aro has moved to azurerm_redhat_openshift_cluster.main` and zero destroys. Roll to dev → test → prod.
- **Next major release (v2.0.0):** rename the output. CHANGELOG `### Changed` with `**Breaking:**` prefix. Consumers bump `version = "~> 2.0"` and update their `module.x.aro_cluster_id` references to `module.x.cluster_id`.
- **Team decision:** module split goes to architecture review.

The lesson: a Minor-severity output rename (#1) is **strictly more dangerous to apply** than a Major-severity label rename with a migration path (#2). Class, not severity, drives the order.

## The Migration Plan Output Template

When the skill runs, emit the plan in **this exact structure**. Fill in content; keep section order and headings verbatim.

```markdown
## Summary
<N> findings classified: 🔴 <N> breaking, 🟡 <N> state-sensitive, 🟢 <N> safe, ⚪ <N> advisory.
Recommended version bump: <patch | minor | major>.
Recommended apply order: 🟢 safe → 🟡 state-sensitive → 🔴 breaking (in a versioned release).

## 🟢 Safe changes — apply freely (no state impact)
Group by file. One commit recommended. Verify `terraform plan` shows no diff after applying.

### `outputs.tf`
- [ ] <finding 1> (reviewer severity: <Major>)
- [ ] <finding 2> (reviewer severity: <Minor>)

### `variables.tf`
- [ ] <finding 3> (reviewer severity: <Minor>)

…

## 🟡 State-Sensitive changes — apply with migration steps
One commit per finding. Each must show the expected `moved` lines and zero destroys at plan time.

### 1. <finding title> (reviewer severity: <Major>)

**Risk if applied wrong:** <e.g. "Without the `moved` block, the AKS cluster will be destroyed and recreated.">

**Steps (in order):**
1. Add the `moved` block:
   ```hcl
   moved {
     from = <old.address>
     to   = <new.address>
   }
   ```
2. Rename the resource label in the same commit.
3. Run `terraform plan` — expect: `# <old> has moved to <new>` and **zero destroy/create lines**.
4. Apply in `dev`, verify state, then `test`, then `prod`.
5. (Optional) Remove the `moved` block on the next major version bump.

…

## 🔴 Breaking changes — versioned release with consumer migration
Bundle into one dedicated release. Land version bump + CHANGELOG + moved/removed blocks together.

### 1. <finding title> (reviewer severity: <Critical>)

**Why it breaks:** <e.g. "Removing the `legacy_subnet_id` variable means any caller still passing it gets a plan-time error.">

**Migration steps:**
1. <e.g. "In v1.x, mark the variable deprecated in `deprecated_variables.tf` (one minor release prior).">
2. <e.g. "In v2.0, remove the variable.">
3. Add the `removed` block:
   ```hcl
   removed {
     from = <address>
     lifecycle { destroy = false }
   }
   ```
4. Run `terraform plan` — expect: <expected output>.

**Version bump:** Major (variable removal = contract break).

**CHANGELOG entry:**
```markdown
### Removed
- **Breaking:** remove variable `legacy_subnet_id`. Consumers must remove this argument from their module call.
```

**Lowest environment first:** `dev`. Validate, then promote to `test`, then `prod`. Consumers must bump `version = "~> 2.0"` in lock-step.

…

## ⚪ Advisory items — team decision required
These are not code changes. Surface them for a decision; do not auto-apply.

### 1. <recommendation title>
- **Decision needed:** <e.g. "Should we split this module into two?">
- **Options:**
  - **A.** <option 1> — <tradeoff>
  - **B.** <option 2> — <tradeoff>
  - **C.** Defer — <consequence of doing nothing>
- **Recommended owner:** <module owner / platform team>

…
```

## What This Skill Does Not Do

- **It does not apply fixes.** It produces a plan. Apply tooling lives elsewhere.
- **It does not invent findings.** It only advises on findings that were supplied (typically from `terraform-azure-module-review`). If you have no findings, you have nothing to classify.
- **It does not change reviewer severity.** Severity (Critical / Major / Minor / Informational) and Class (🔴 / 🟡 / 🟢 / ⚪) are independent. Both surface in the plan; neither overrides the other.
- **It does not duplicate state-mutation mechanics.** For the syntax of `moved`, `removed`, `import`, and `terraform state mv`, see `terraform-state-and-lifecycle`. Cite, don't recopy.
- **It does not duplicate version-bump rules.** For SemVer guidance and CHANGELOG layout, see `terraform-module-versioning`. Cite, don't recopy.
- **It does not negotiate with consumers.** If a 🔴 needs a coordinated upgrade, the plan flags it; the team coordinates it.

## Common Mistakes

1. **Mixing severity with class.** Treating "Critical" as "apply first." A Critical-but-Breaking fix without a `moved` block destroys infrastructure. Class drives order, not severity.
2. **Applying 🔴 fixes without a `moved` / `removed` block** — silent destroy/recreate at next apply. Always pair an address change with the migration block, in the same commit.
3. **Applying everything at once and asking "why does plan show destroys?"** That's the failure mode this skill exists to prevent. Apply 🟢 first, verify clean plan, then 🟡, verify expected `moved` lines, then 🔴 in a versioned release.
4. **Forgetting to update CHANGELOG when applying a 🔴.** If the next release ships without a `**Breaking:**` entry, consumers walk into a destroy without warning. CHANGELOG and `moved`/`removed` block land in the same commit.
5. **Applying 🟢 fixes alongside 🔴 in the same commit.** Makes the breaking change harder to revert (you'd lose the safe fixes too) and obscures the diff readers care about. Keep them in separate commits — even separate releases.
6. **Promoting a 🟡 to 🟢 because "the description change is trivial."** It isn't. If the fix touches a state address, validation rule, or `nullable` flag, it's at least 🟡 — verify with a plan, don't eyeball it.
7. **Skipping the lowest-environment verification step on 🔴.** Roll to `dev` first, confirm `terraform plan` shows the expected `moved`/`removed` lines, then promote. Production is not where you discover a misclassified fix.
8. **Treating an ⚪ Advisory item as a 🟢 quick win.** "Move the secret to Key Vault" is a coordinated change with consumer-side impact, not a one-line edit. Surface it for a decision; don't sneak it into the safe-changes commit.

## Review Checklist

Run this checklist on every migration plan you emit. Every 🔴 must have a `moved`/`removed` block. Every 🟡 must have ordered steps. Every 🔴 must have a CHANGELOG entry.

- [ ] Every finding has exactly one class assigned (🔴, 🟡, 🟢, or ⚪).
- [ ] The reviewer's severity (Critical/Major/Minor/Informational) is preserved verbatim alongside the class.
- [ ] Every 🔴 item includes the exact `moved` or `removed` block (or `terraform state mv` if `required_version < 1.1.0`).
- [ ] Every 🔴 item names the version bump level (Major or Minor — never Patch).
- [ ] Every 🔴 item has a `**Breaking:**`-prefixed CHANGELOG entry under the correct section.
- [ ] Every 🟡 item has an ordered step-by-step migration plan and a stated "risk if applied wrong."
- [ ] No 🟢 item touches a state address, variable type, output name, or required-flag.
- [ ] The Summary line names total findings per class and the recommended version bump.
- [ ] The recommended apply order is 🟢 → 🟡 → 🔴 (no exceptions).
- [ ] No 🟢 fix is bundled into the same commit as a 🔴 fix.
- [ ] All ⚪ advisory items list the decision needed and at least two options with tradeoffs.
- [ ] The plan cites `terraform-state-and-lifecycle` for `moved`/`removed` mechanics and `terraform-module-versioning` for bump/CHANGELOG rules instead of duplicating them.
