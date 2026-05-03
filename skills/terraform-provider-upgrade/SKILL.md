---
name: terraform-provider-upgrade
description: Migrate a Terraform module from one provider version to another (e.g. azurerm 3.85 to 4.20) by computing a schema diff between the two versions and producing a deterministic migration plan. Use this skill whenever the user asks to "upgrade azurerm 3.x to 4.x", "migrate this module to provider X.Y", "bump the provider version", "is this module compatible with version Z", "what changed between provider versions", or when CI flags an unsupported provider constraint, or when modernizing a legacy module. This is the right skill the moment a provider bump is on the table — do not guess-and-check. Pairs with `terraform-schema-lookup` (single-version schema fetching) and adds the diff layer plus three-tier confidence scoring (mechanical, likely-safe, needs-review) backed by CHANGELOG context for risky items. Defers `moved` block syntax to `terraform-state-and-lifecycle`.
---

# Terraform Provider Upgrade

Upgrading a module's provider version is not a code-generation problem — it's a transformation problem. The schema for both versions tells you exactly what changed; the module source tells you exactly which of those changes you care about. Compose the two and the LLM only has to reason about the genuine judgment calls.

This skill turns "upgrade this module to azurerm 4.x" into a deterministic pipeline: snapshot both schemas, diff, intersect with actual usage, classify each change as mechanical / likely-safe / needs-review, fetch CHANGELOG context for the risky ones, and emit a structured migration plan.

For fetching a single version's schema, see `terraform-schema-lookup`. For state-mutation moves (e.g. `moved` blocks when a resource address changes), see `terraform-state-and-lifecycle` — this skill does not duplicate that guidance.

## Contents
- [When To Invoke](#when-to-invoke)
- [Workflow](#workflow)
- [Three-Tier Confidence Scoring](#three-tier-confidence-scoring)
- [Schema Doesn't Tell The Whole Story](#schema-doesnt-tell-the-whole-story)
- [Migration Plan Output Format](#migration-plan-output-format)
- [Pairing With State Mutations](#pairing-with-state-mutations)
- [Performance Honesty](#performance-honesty)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)

## When To Invoke

Reach for this skill the moment a provider-version change is in scope. Specifically:

- The user asks to upgrade a module to a newer provider version ("bump azurerm to 4.20", "move this module to azurerm 4.x").
- CI flags an unsupported provider — the lockfile, AVM tooling, or a downstream consumer has rejected the current pin.
- A compatibility question lands: "is this module compatible with azurerm 4.x?", "what would break if we moved to provider Y?".
- Modernizing a legacy module that is several minor versions behind, even if no specific version is named yet.
- A module fails to plan after a `terraform init -upgrade` and the diagnostics point at deprecated or removed attributes.

Do not skip ahead and edit HCL by hand. Without the schema diff you are guessing; with it, most of the work is mechanical.

## Workflow

### Step 1 — Detect the current pinned version

Read `versions.tf` and pull the `version` constraint from the relevant `required_providers` block. Note whether the constraint is a pin (`= 3.85.0`), a pessimistic constraint (`~> 3.85`), or a range. Also capture `required_version` for the Terraform core itself — a provider bump may force a core bump too.

### Step 2 — Confirm or ask for the target version

If the user named a target ("upgrade to 4.20"), use it. Otherwise, ask once for the target. Do not infer "latest" silently — major bumps regularly carry breaking changes the user may not be ready for.

### Step 3 — Fetch schemas for both versions

You need two JSON schema dumps: one for the current pin, one for the target. Pick whichever workflow fits the workspace.

**Approach A — In-place, single working tree.** Snapshot, dump, bump, dump, restore.

```bash
# 1. Snapshot the current versions.tf
cp versions.tf /tmp/versions.tf.bak

# 2. Dump the current schema
terraform init -upgrade=false >/dev/null
terraform providers schema -json > /tmp/tf-schema-azurerm-3.85.0.json

# 3. Bump the constraint in versions.tf to the target (edit in place)
#    e.g. version = "= 4.20.0"
terraform init -upgrade >/dev/null
terraform providers schema -json > /tmp/tf-schema-azurerm-4.20.0.json

# 4. Restore the original versions.tf so the working tree is clean
cp /tmp/versions.tf.bak versions.tf
```

**Approach B — Two worktrees.** Cleaner when the module already has a branch with the new pin, or when you want to keep the current tree untouched.

```bash
git worktree add ../upgrade-from <branch-or-sha-with-old-pin>
git worktree add ../upgrade-to   <branch-or-sha-with-new-pin>

(cd ../upgrade-from && terraform init -upgrade=false >/dev/null && \
  terraform providers schema -json > /tmp/tf-schema-azurerm-3.85.0.json)

(cd ../upgrade-to   && terraform init -upgrade >/dev/null && \
  terraform providers schema -json > /tmp/tf-schema-azurerm-4.20.0.json)
```

Either way, the file naming convention is `tf-schema-<provider>-<version>.json` so later steps can reference the two artifacts unambiguously.

### Step 4 — Identify which resource and data-source types the module actually uses

Most of the schema diff is irrelevant — the module probably touches a small subset of the provider surface. Enumerate it first.

```bash
# All resource types declared in the module
grep -rhoE 'resource "[^"]+"'    *.tf | awk -F'"' '{print $2}' | sort -u

# All data sources
grep -rhoE 'data "[^"]+"'        *.tf | awk -F'"' '{print $2}' | sort -u
```

Save the union as the "in-scope types" list. Every diff step from here on is filtered by this list.

### Step 5 — Compute the diff with `jq`

For each in-scope type, compute added / removed / deprecated attributes between the two schemas. Top-level attributes first:

```bash
jq -n \
  --slurpfile o /tmp/tf-schema-azurerm-3.85.0.json \
  --slurpfile n /tmp/tf-schema-azurerm-4.20.0.json \
  --arg TYPE   azurerm_storage_account \
  '
  def attrs(s; t):
    s[0].provider_schemas["registry.terraform.io/hashicorp/azurerm"]
        .resource_schemas[t].block.attributes // {};
  {
    type:                  $TYPE,
    added:                 ((attrs($n; $TYPE) | keys) - (attrs($o; $TYPE) | keys)),
    removed:               ((attrs($o; $TYPE) | keys) - (attrs($n; $TYPE) | keys)),
    deprecated_in_target:  [ attrs($n; $TYPE) | to_entries[]
                              | select(.value.deprecated // false)
                              | .key ],
    newly_required:        [ attrs($n; $TYPE) | to_entries[]
                              | select(.value.required // false)
                              | .key ]
                            - [ attrs($o; $TYPE) | to_entries[]
                                | select(.value.required // false)
                                | .key ],
    force_new_flipped:     [ attrs($n; $TYPE) | to_entries[]
                              | .key as $k
                              | select(((attrs($o; $TYPE)[$k].force_new // false)
                                        != (.value.force_new // false)))
                              | $k ]
  }'
```

**Recurse into nested blocks too.** Top-level attribute diffs miss most of the interesting changes. Nested blocks live under `block.block_types[<name>].block.attributes`. Walk them:

```bash
jq -n \
  --slurpfile o /tmp/tf-schema-azurerm-3.85.0.json \
  --slurpfile n /tmp/tf-schema-azurerm-4.20.0.json \
  --arg TYPE azurerm_storage_account \
  '
  def block(s; t):
    s[0].provider_schemas["registry.terraform.io/hashicorp/azurerm"]
        .resource_schemas[t].block;

  def walk_blocks(b; path):
    [ { path: path, attributes: (b.attributes // {}) } ]
    + ( (b.block_types // {}) | to_entries
        | map(walk_blocks(.value.block; path + "." + .key)) | add // [] );

  {
    old: walk_blocks(block($o; $TYPE); $TYPE),
    new: walk_blocks(block($n; $TYPE); $TYPE)
  }'
```

Compare `old` vs `new` by `path` and inside each path by attribute key. A change in `azurerm_storage_account.network_rules` is what bites — the top-level attribute set may look identical.

**Type changes matter too.** An attribute that went from `string` to `list(string)`, or from `set` to `list`, is a real diff even if the key didn't move. Pull `value.type` for each attribute and compare across versions.

### Step 6 — Cross-reference the diff against actual usage

The schema may report 200 added attributes on `azurerm_storage_account`. The module probably sets twelve of them. Filter the diff down to *changes that touch arguments the module actually sets*.

```bash
# Pull every argument assignment in main.tf for a given resource type, naively.
# (Refine with hcl-aware tooling if needed.)
awk '/resource "azurerm_storage_account"/,/^}$/' main.tf \
  | grep -oE '^[[:space:]]+[a-z_]+[[:space:]]*=' \
  | awk '{print $1}' | sort -u
```

Intersect this set with the schema diff. Removed-but-unused = no-op. Added-but-unused = no-op. Removed-and-used = needs review. Now the migration plan is short and load-bearing.

## Three-Tier Confidence Scoring

Every surviving diff item gets exactly one tier. The tier dictates whether the agent applies the change, applies it with a CHANGELOG entry, or stops and asks.

### Mechanical (deterministic transform from schema diff)

The schema itself encodes the answer. No human required.

- **Plain rename**: attribute deprecated in the target with an explicit `replacement` field pointing at the new name. Apply the rename.
- **Attribute moved into a nested block**: same data, different shape, schema makes the move obvious. Restructure the HCL.
- **Block renamed** (e.g. `network_rules` → `network`) where the deprecation message names the successor. Restructure.

The agent applies these without review. Note them in the migration plan so the reviewer can scan, but don't gate on them.

### Likely safe (low-risk, deterministic intent, write a CHANGELOG entry)

The change is safe in nearly every case but worth recording.

- **New optional attribute with a default** that doesn't change behavior for callers who don't set it.
- **Newly deprecated attribute that the module doesn't use** — nothing to do today, but flag in the CHANGELOG so consumers know the field is on its way out.
- **Type widening with backwards compatibility** (`string` → `list(string)` where a single string value still parses).
- **Attribute reordering or alphabetization** in nested blocks, when behavior is unchanged.

Apply, add a CHANGELOG entry, no judgment call required.

### Needs review (stop and flag for the human)

The schema cannot answer the question. Surface the item and the CHANGELOG context you fetched.

- **`force_new` flipped**: changing this attribute now recreates the resource. Real production blast radius.
- **Removed attribute with no replacement** named in the schema. Could be a behavior removed entirely, or a renamed-but-undeclared replacement.
- **Type narrowing** (`list(string)` → `string`, or `set` → single value) — modules that were passing collections will break.
- **A required attribute that wasn't required before**: every existing caller is now invalid until they pass the new value.
- **Behavior change documented only in CHANGELOG, not in the schema** — e.g. "default for `min_tls_version` changed from `TLS1_0` to `TLS1_2`". The schema is silent; production isn't.

For every needs-review item, the migration plan must include the CHANGELOG context (next section) and the specific question the human has to answer.

## Schema Doesn't Tell The Whole Story

The schema describes the *new shape*. It does not describe *why it changed* or *whether the change is safe in production*. For 🔴 needs-review items, fetch authoritative context before producing the plan.

- **Provider CHANGELOG between the two versions** — for azurerm:
  `https://github.com/hashicorp/terraform-provider-azurerm/blob/v<target-version>/CHANGELOG.md`
  Read every entry between the current pin and the target. The relevant entries are usually under `BREAKING CHANGES` and `ENHANCEMENTS`/`BUG FIXES` sections that mention the resource by name.
- **Resource doc page** — `https://registry.terraform.io/providers/hashicorp/azurerm/<version>/docs/resources/<resource>` for the target version. Schema fields may have prose semantics ("setting this to `true` is a destructive operation").
- **Upgrade guide** — many providers ship a top-level upgrade guide for major bumps (e.g. `Upgrading to v4.0 of the AzureRM Provider`). When present, it's the single highest-signal document; skim it cover-to-cover before producing the plan.

Use WebFetch to pull these. The CHANGELOG entry is what turns "needs review" into a confident decision; without it you are passing the user a list of unanswered questions.

When CHANGELOG context contradicts the schema (or vice versa), the CHANGELOG wins for behavior questions and the schema wins for shape questions.

## Migration Plan Output Format

The skill produces a *plan*, not a code change. Apply the plan as a separate, reviewable step. The plan has five sections, in this order.

### 1. Summary

```
Module:          terraform-azurerm-storage-account
Provider:        hashicorp/azurerm
From:            3.85.0
To:              4.20.0
Terraform core:  >= 1.5  (no change required)

Mechanical:      4
Likely safe:     2
Needs review:    1

Recommended bump: MAJOR (provider major version changed; one breaking change in scope)
```

The bump level follows SemVer for the *module*, not the provider. Any 🔴 item that requires consumer action is a major. Otherwise minor.

### 2. Mechanical transformations

For each item:

- **File** + line.
- **Before** snippet, **after** snippet.
- **Source**: which schema-diff entry justifies the transformation (e.g. `azurerm_storage_account.queue_properties → azurerm_storage_account_queue_properties (deprecation message)`).

```hcl
# main.tf:42 — attribute moved into a separate resource
# Source: schema diff — `queue_properties` deprecated, replacement
# `azurerm_storage_account_queue_properties` resource introduced.

# Before
resource "azurerm_storage_account" "main" {
  ...
  queue_properties { ... }
}

# After
resource "azurerm_storage_account" "main" { ... }

resource "azurerm_storage_account_queue_properties" "main" {
  storage_account_id = azurerm_storage_account.main.id
  ...
}
```

### 3. Likely-safe changes

Same shape as mechanical, plus the exact CHANGELOG.md entry the agent will add to the module's own CHANGELOG.

```markdown
### Changed
- Bump azurerm provider constraint from `~> 3.85` to `~> 4.20`.
- Note: argument `enable_https_traffic_only` is deprecated in favor of `https_traffic_only_enabled`. The module already uses the new name; no consumer action required.
```

### 4. Needs review

One block per item.

```
[NEEDS REVIEW] azurerm_storage_account.account_replication_type — force_new behavior changed

Where used: main.tf:14
What changed: in azurerm 4.x, changing `account_replication_type` from `LRS` to
              `GRS` no longer rebuilds the resource — the provider now performs
              an in-place update.
CHANGELOG context (https://github.com/hashicorp/terraform-provider-azurerm/blob/v4.20.0/CHANGELOG.md):
  > `azurerm_storage_account` — `account_replication_type` is no longer ForceNew; updates are now performed in-place.
Question for human: existing state was created under the ForceNew semantics. Confirm
              that callers expect in-place replication changes and that any
              automation that previously assumed a destroy/create on this
              attribute has been updated.
```

### 5. Recommended apply order

Always:

1. Mechanical transformations (atomic commit, easy to review).
2. Likely-safe changes + version constraint bump + CHANGELOG (atomic commit).
3. Needs-review items (separate commit per item, only after the human signs off).

Mixing tiers in a single commit is the single most common source of failed PRs in upgrade work — keep them apart.

## Pairing With State Mutations

If any transformation requires a `moved` block (resource re-keyed, resource address changed, sub-resource extracted from a parent), do **not** write the `moved` syntax in this skill's output. Defer to `terraform-state-and-lifecycle` for the move syntax and the rules on when `moved` is sufficient vs. when a `terraform state mv` is required.

In the migration plan, mark the item as needing a `moved` block and reference the sibling skill:

```
[NEEDS MOVED BLOCK] azurerm_storage_account.queue_properties → azurerm_storage_account_queue_properties.main
See: terraform-state-and-lifecycle for the moved-block syntax and acceptance criteria.
```

## Performance Honesty

Two `terraform init` runs plus a `jq` walk is fine for a one-off upgrade — call it 60–120 seconds end-to-end on a typical module. It is not fine for batch work across dozens of modules: every `init` re-downloads the provider plugin, and `jq` walks every block type even when most aren't in scope.

If the team later builds a schema cache (provider+version → JSON, keyed by SHA), the workflow in this skill does not change. The same six steps apply; the schema-fetch step gets faster. Don't restructure the skill around a cache that doesn't exist yet, but don't pretend the current shape is what we'd run in CI on every module weekly.

For now: accept the latency, run it once per upgrade, and move on.

## Common Mistakes

1. **Doing the upgrade without consulting the schema diff at all.** Edit-and-`terraform plan` until it stops complaining is a guess-and-check loop that misses every silent behavior change — the things that don't surface as plan errors.
2. **Applying 🔴 needs-review items without CHANGELOG context.** The schema doesn't say *why*. Without the CHANGELOG you are inferring intent from a JSON shape, and the inference is often wrong.
3. **Missing nested-block changes.** Top-level attribute diffs are easy. The breaking changes hide in `block_types[*].block.attributes`. The `jq` diff has to recurse.
4. **Assuming the module uses every attribute in the schema.** A 200-line diff against a 12-attribute module is 188 lines of noise. Filter by actual usage in step 6, not at review time.
5. **Forgetting to bump `required_version` for Terraform core.** Some provider majors require a newer Terraform CLI. Read the provider's upgrade guide for the floor, and update `versions.tf` `required_version` alongside the provider constraint.
6. **Mixing tiers in a single commit.** Mechanical + needs-review in one PR forces the reviewer to context-switch on every hunk and makes a clean revert impossible if a 🔴 item turns out to be wrong.
7. **Writing the migration plan against the wrong schema dump.** Both files should be from the same module's `terraform providers schema -json` — not one from this module and one from a different workspace that happens to have the same provider. The `provider_schemas` keys are stable, but resource subsets reflect the workspace, not just the version.
8. **Skipping the upgrade guide for major bumps.** Major-version upgrade guides (e.g. `Upgrading to v4.0`) frequently document behavior changes that are not visible in the per-resource CHANGELOG entries.

## Review Checklist

- [ ] Both schema dumps were taken from the same module workspace (not unrelated trees).
- [ ] Every in-scope `resource "<type>"` and `data "<type>"` is represented in the diff.
- [ ] Nested blocks were diffed recursively, not just top-level attributes.
- [ ] Type changes (string → list, set → list) were checked, not just key sets.
- [ ] The diff was filtered to attributes the module actually sets before classification.
- [ ] Every diff item has exactly one tier: mechanical, likely-safe, or needs-review.
- [ ] Every 🔴 needs-review item carries CHANGELOG context fetched from the provider repo.
- [ ] Provider major-version upgrade guide was consulted when crossing a major boundary.
- [ ] `versions.tf` constraint is updated and `required_version` is checked against the provider's floor.
- [ ] `terraform init -upgrade` succeeds against the new constraint.
- [ ] `terraform validate` passes after mechanical transformations are applied.
- [ ] `terraform plan` against an example shows no spurious recreates (no force-new firing for unchanged inputs).
- [ ] CHANGELOG.md has an entry for the bump, with breaking changes called out.
- [ ] `moved` blocks (if any) defer to `terraform-state-and-lifecycle` and are committed before the resource rename takes effect.
- [ ] Mechanical, likely-safe, and needs-review tiers are split into separate commits.
