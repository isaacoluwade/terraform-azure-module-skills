# Review Output Format

Use this template for all Terraform Azure module PR reviews. The structure ensures consistency, traceability to standards, and actionability.

---

## Template

````markdown
# Module Review: terraform-azurerm-<service> (<branch/PR>)

**Review Date:** YYYY-MM-DD
**Module:** `terraform-azurerm-<service>`
**Branch/PR:** `<branch-name>` or PR #<number>
**Version:** `X.Y.Z` (Initial Certification / Minor Update / Major Update / Patch)
**Reviewer:** <name or agent identifier>
**Review Type:** Certification Review / Change Review / Pre-merge Review

---

## Verdict: [Approve | Approve with Comments | Request Changes | Needs Major Revision]

[1-3 sentence summary explaining the verdict. State the most important finding that drove the decision. If requesting changes, state what must be fixed before re-review.]

---

## Findings Summary

| Severity | Count |
|----------|-------|
| Critical | N |
| Major | N |
| Minor | N |
| Informational | N |
| **Total** | **N** |

---

## Findings by File

### `<filename>.tf`

#### [SEVERITY-ID] [Short title]

**Location:** `<filename>.tf` — `<resource/variable/local/output name>` (line ~N)

**Current code:**
```hcl
<the problematic code, 3-8 lines of context>
```

**Issue:** [Clear explanation of what's wrong and why it matters. Not just "violates standard" — explain the runtime impact, consumer confusion, or technical debt it creates.]

**Standard:** *"[Exact quote of the rule being violated]"*
— see the `<topical-skill-name>` skill (e.g. `terraform-variable-design`).

**Fix:**
```hcl
<exact corrected code, ready to copy-paste>
```

**Impact:** [What happens if this isn't fixed? Runtime failure? Consumer confusion? State issues?]

---

[Repeat for each finding in this file]

### `<next-filename>.tf`

[Continue with findings for the next file...]

---

## Cross-Cutting Findings

[Findings that span multiple files or are architectural in nature]

### [SEVERITY-ID] [Short title]

**Affected files:** `file1.tf`, `file2.tf`, `file3.tf`

**Issue:** [Description]

**Fix:** [Description with code examples for each affected file]

---

## Positive Observations

| # | Area | Observation |
|---|------|-------------|
| 1 | [Area] | [What was done well and why it matters] |
| 2 | [Area] | [What was done well and why it matters] |

---

## Action Items

### Must Fix (Blocks Merge)

| ID | Severity | File | Summary |
|----|----------|------|---------|
| C-01 | Critical | `file.tf` | [One-line summary] |

### Should Fix (Before Merge)

| ID | Severity | File | Summary |
|----|----------|------|---------|
| M-01 | Major | `file.tf` | [One-line summary] |

### Recommended

| ID | Severity | File | Summary |
|----|----------|------|---------|
| m-01 | Minor | `file.tf` | [One-line summary] |

---

## Conditional Chain Verification

[For each boolean toggle in the module, document the complete trace]

### Toggle: `var.<feature>.enabled`

| Resource/Data Source | Guard Present | References Use [0]/try() |
|----------------------|---------------|--------------------------|
| `data.azurerm_*.resource` | Yes — `count = var.feature.enabled ? 1 : 0` | Yes — `[0].value` |
| `resource.azurerm_*.main` | Yes — `count = var.feature.enabled ? 1 : 0` | Yes — `try(..., null)` |
| `local.feature_value` | **MISSING** | N/A |

---

## Breaking Change Assessment

[For updates to existing modules — skip for 1.0.0 initial certification]

| Change | Breaking? | Migration Path |
|--------|-----------|----------------|
| [Description of change] | Yes/No | [moved block / variable deprecation / etc.] |
````

---

## Format Rules

### Finding IDs

Use a consistent ID scheme:

- **C-01, C-02, ...** — Critical findings
- **M-01, M-02, ...** — Major findings
- **m-01, m-02, ...** — Minor findings (lowercase `m`)
- **I-01, I-02, ...** — Informational observations

IDs are referenced in the Action Items tables and in any cross-references between findings.

### Code Blocks

Always include:

1. **Current code** — what's in the PR now (3-8 lines of context, not the entire file).
2. **Fix** — the exact corrected code, ready to copy-paste.

For suggested changes that mirror ADO/GitHub format:

```
Suggested change:
- old_line
+ new_line
```

A finding above Informational that lacks a code-fix block is incomplete. Add one.

### Citation Rules

Always cite the specific standard being violated:

- A direct quote of the rule: *"exact quote"*.
- A pointer to the topical skill that owns the rule, by skill name (not by file path). Examples: `terraform-variable-design`, `terraform-locals-patterns`, `terraform-module-outputs`.
- Where applicable, reference the AVM rule code (e.g. AVM TFNFR35, AVM TFFR2).

A finding without a standard reference is just an opinion — give it a citation.

### Grouping

- **Primary grouping: by file.** Developers fix code file by file.
- **Secondary: cross-cutting.** Issues that span files go in their own section.
- **Within a file: by severity.** Critical first, then Major, Minor, Informational.

Do not group findings by severity across the whole module — that forces the developer to jump between files.

### Tone

Model a senior reviewer's tone:

- **Direct but not harsh.** State the issue clearly, don't soften it with excessive qualifiers.
- **Question-driven.** Frame issues as questions when the intent is ambiguous: "Why is this looking up the value instead of accepting it as input?"
- **Provide solutions.** Every criticism comes with a fix.
- **Acknowledge good work.** The Positive Observations section is not optional.

Examples of good review tone:

- "This data source should be replaced with a variable input. Pass the resource group ID explicitly instead of looking it up at runtime."
- "Why `lower()` here? The variable already has regex validation enforcing lowercase. Remove the wrapper."
- "Clean use of the authorization module — correctly delegates RBAC management."

Examples of bad review tone:

- "Perhaps you might want to consider possibly using a variable instead of this data source?" (too hedging — state it directly).
- "This is completely wrong and shows a lack of understanding." (too harsh — explain why it's wrong and show the fix).
- "LGTM" (never — even approvals should note what was done well).

### Completeness Checks

Before submitting the review, verify:

1. Every finding has: ID, severity, location, current code, issue description, standard reference, fix.
2. The Conditional Chain Verification section is complete for every toggle.
3. Positive Observations has at least 3-5 entries (there's always something good to note).
4. The Action Items summary matches the findings (no orphaned IDs).
5. The Verdict matches the findings (Critical -> Request Changes, always).
6. Word count on each finding is adequate — not one-liners, but not essays either. Aim for 50-150 words per finding explanation.

### Severity Consistency Checks

Before submitting, verify these severity rules are followed:

| Pattern | Minimum Severity |
|---------|------------------|
| Missing count guard on conditional resource/data source | Critical |
| Null scope in role assignment or `for_each` | Critical |
| Bad default value causes runtime API failure (trace the data flow!) | Critical |
| Resource rename without `moved` block (on updates) | Critical |
| Plaintext credentials that should go through Key Vault | Major |
| Missing standard variable (`brand`, `environment`, etc.) | Major |
| Standard variable with wrong validation or default | Major |
| Wrong `primary_name` format | Major |
| Wrong tags merge order | Major |
| Data source that should be a variable | Major |
| SPN creation in module | Major |
| Role assignment not delegated to auth module | Major |
| `lower()`/`upper()` on validated input | Major |
| `default = ""` instead of `null` for strings | Major |
| `default = null` instead of `{}` for objects | Major |
| Example directory naming | Minor |
| Description missing period | Minor — **never Major** |
| Description missing heredoc format | Minor — **never Major** |
| Output description missing period | Minor — **never Major** |
| `deployed_using` format deviation | Minor |
| Missing architecture diagram | Minor |
| Commented-out code | Minor |
| Correct use of standard patterns | Informational |
| Clean naming conventions | Informational |
| Good separation of concerns | Informational |

**Severity guardrails — do NOT violate these:**

- Trailing periods on descriptions are **always Minor** — they are cosmetic.
- Heredoc description formatting is **always Minor** — it improves readability but doesn't affect runtime.
- Missing `.gitignore` entries are **always Minor** — they affect developer workflow, not consumers.
- Redundant conditional wrappers (e.g., `var.x ? try(...) : null` where `try()` suffices) are **always Minor** — the code works correctly, it's just verbose.

### Keep Findings Atomic

Do not bundle unrelated issues into a single finding. Each finding should describe ONE problem with ONE root cause. If standard variables are wrong AND the VERSION file is missing AND the CHANGELOG says 0.1.0, that is THREE separate findings at potentially different severity levels — not one combined Critical.
