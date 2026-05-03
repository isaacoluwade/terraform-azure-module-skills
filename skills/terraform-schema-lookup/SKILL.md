---
name: terraform-schema-lookup
description: Verify Terraform resource, data source, and provider arguments against the live provider schema before authoring or editing any HCL — instead of relying on training-data recall, which goes stale every provider release. Use this skill any time you are about to write or edit a `resource`, `data`, or `provider` block, or are unsure about an argument's exact name, type, optionality, deprecation status, or `ForceNew` behavior. Workflow: read `versions.tf`, run `terraform init`, dump `terraform providers schema -json` to a version-keyed cache file, then query specific resource types with `jq`. Multi-version aware — re-extract the constraint from `required_providers` per module; never share a cache across versions. Triggers on "what arguments does azurerm_X support", "is this attribute deprecated", "did this rename in 4.x", "before I write this resource block", "is this attribute ForceNew". When the schema can't answer behavioral questions, fall back to the registry docs page via WebFetch.
---

# Terraform Schema Lookup

Before authoring or editing any Terraform `resource`, `data`, or `provider` block, verify the argument shape against the live provider schema. Training-data recall goes stale every provider release — `azurerm` ships a major version roughly yearly and minor versions every two weeks, and renames, deprecations, and `ForceNew` flips happen constantly. The schema is the only source of truth that matches what the binary will actually accept.

The cost is one `terraform init` per session per module (10–30s) plus a one-line `jq` query per lookup. The payback is not writing HCL that fails at `plan` time, not silently triggering recreations, and not leaving deprecated attributes in modules that ship to consumers.

This skill assumes no external MCP servers. Everything is plain `terraform`, `jq`, and (as a fallback) `WebFetch` against the public registry.

## Contents
- [When to Invoke](#when-to-invoke)
- [Workflow](#workflow)
- [Querying the Schema with jq](#querying-the-schema-with-jq)
- [What the Schema Tells You](#what-the-schema-tells-you)
- [What the Schema Does Not Tell You](#what-the-schema-does-not-tell-you)
- [Multi-Version Handling](#multi-version-handling)
- [Cost-Awareness and Caching](#cost-awareness-and-caching)
- [Fallback to Registry Docs](#fallback-to-registry-docs)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)

## When to Invoke

Invoke this skill **before** you write or edit:
- A `resource` block — to confirm every argument exists, its type, whether it is required, and whether changing it triggers `ForceNew`.
- A `data` block — to confirm filter arguments and the exported attribute set.
- A `provider` block — to confirm provider-level arguments (e.g. `features {}` shape, `subscription_id`, auth attributes) for the constrained version.
- A nested block inside any of the above — to confirm the block name, `min_items`/`max_items`, and the inner attributes.

Invoke this skill **whenever** you are uncertain about:
- An argument's exact name (`storage_account_name` vs `account_name`, `resource_group_name` vs `resource_group`).
- Whether an attribute was renamed, removed, or deprecated in the version the module pins.
- Whether an attribute is `Required`, `Optional`, `Computed`, or some combination.
- Whether changing an attribute forces resource recreation (`ForceNew`).
- The shape of a nested block — what fields it contains, how many instances are allowed.
- Whether two attributes are mutually exclusive (`ConflictsWith`) or co-required (`RequiredWith`, `ExactlyOneOf`, `AtLeastOneOf`).

If you catch yourself thinking "I'm pretty sure the argument is called X" — stop and check the schema. The cost of a single `jq` query is much lower than the cost of pushing HCL that fails plan or quietly does the wrong thing.

## Workflow

### Step 1: Read `versions.tf` to extract the provider constraint

The first thing to do for any module-scoped task is to read the module's `versions.tf` and pull the version constraint for every provider you will touch.

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.20.0"
    }
  }
}
```

From this, the relevant pin is `~> 4.20.0` — any 4.20.x patch is acceptable. After `terraform init`, the lockfile (`.terraform.lock.hcl`) will resolve this to a concrete version (e.g. `4.20.0`) — use that exact version as the cache key.

If a module pins multiple providers (`azurerm`, `azuread`, `azapi`, `random`, `null`), extract them all. Each provider gets its own schema dump.

### Step 2: Run `terraform init` once per session per module

In the module's directory:

```bash
cd <module-root>
terraform init -backend=false -upgrade=false
```

- `-backend=false` skips backend config — you only need the provider plugins.
- `-upgrade=false` respects the lockfile — important if the module pins a specific patch you must match.

If the module is not initializable in place (e.g. it has unresolved variables in a backend block, or relies on a wrapper at a parent path), use a temp directory:

```bash
mkdir -p /tmp/tf-schema-shim && cd /tmp/tf-schema-shim
cat > versions.tf <<'EOF'
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "= 4.20.0"
    }
  }
}
EOF
terraform init -backend=false
```

Pin the exact version in the shim — the goal is to match the resolved version from the real module's lockfile.

`terraform init` typically takes 10–30 seconds because it downloads the provider binary (azurerm is roughly 100MB unpacked). You only do this once per session per module.

### Step 3: Dump the schema to a version-keyed cache file

```bash
terraform providers schema -json > /tmp/tf-schema-azurerm-4.20.0.json
```

The cache key includes both the provider name and the resolved version. This is non-negotiable — if you have a second module open in the same session that pins `azurerm 3.85.0`, that one needs `/tmp/tf-schema-azurerm-3.85.0.json`. Sharing a cache across versions is how you write HCL that's correct for the wrong provider version.

Confirm the file size — a fresh azurerm 4.x schema is about 30–50 MB of JSON. **Do not** read this file with `Read` or pipe it into your context. Always query with `jq`.

### Step 4: Query specific resource types with `jq`

The provider schema is shaped like this:

```
.provider_schemas
  └── "registry.terraform.io/hashicorp/azurerm"
       ├── provider.block.attributes        # provider-level args
       ├── resource_schemas
       │    └── azurerm_storage_account
       │         └── block
       │              ├── attributes        # flat args
       │              └── block_types       # nested blocks
       └── data_source_schemas
            └── azurerm_storage_account
```

Use targeted queries — never dump the whole tree. See [Querying the Schema with jq](#querying-the-schema-with-jq) for the exact commands.

### Step 5: Use the result to verify before authoring

For each argument you plan to write, confirm from the schema:
- The argument name exists at the level you're writing it (top-level vs nested in a block).
- The type matches what you intend to pass (`string`, `number`, `bool`, `set(string)`, `list(object({...}))`).
- If `required` is true, the value is non-null and non-empty.
- If `force_new` is true, you have a deliberate reason to set it on a resource that may be updated in place.
- If `deprecated` is true, you have an explicit migration plan — do not author new HCL using a deprecated attribute.
- For nested blocks, `min_items` and `max_items` are honored.
- Any `validators` field hints (some `ConflictsWith` / `ExactlyOneOf` / `RequiredWith` rules surface here, but coverage is incomplete — see the honesty note in [What the Schema Does Not Tell You](#what-the-schema-does-not-tell-you)).

## Querying the Schema with jq

### List every resource type the provider supports

```bash
jq -r '.provider_schemas["registry.terraform.io/hashicorp/azurerm"].resource_schemas | keys[]' \
  /tmp/tf-schema-azurerm-4.20.0.json | head -50
```

### Get the flat attributes of a resource

```bash
jq '.provider_schemas["registry.terraform.io/hashicorp/azurerm"].resource_schemas.azurerm_storage_account.block.attributes' \
  /tmp/tf-schema-azurerm-4.20.0.json
```

Each attribute looks roughly like:

```json
{
  "name": {
    "type": "string",
    "description_kind": "plain",
    "required": true,
    "force_new": true
  }
}
```

### Get only required attributes

```bash
jq '[.provider_schemas["registry.terraform.io/hashicorp/azurerm"].resource_schemas.azurerm_storage_account.block.attributes | to_entries[] | select(.value.required == true) | .key]' \
  /tmp/tf-schema-azurerm-4.20.0.json
```

### Get only deprecated attributes (with their deprecation messages)

```bash
jq '[.provider_schemas["registry.terraform.io/hashicorp/azurerm"].resource_schemas.azurerm_storage_account.block.attributes | to_entries[] | select(.value.deprecated != null) | {name: .key, message: .value.deprecated}]' \
  /tmp/tf-schema-azurerm-4.20.0.json
```

### Get only ForceNew attributes

```bash
jq '[.provider_schemas["registry.terraform.io/hashicorp/azurerm"].resource_schemas.azurerm_storage_account.block.attributes | to_entries[] | select(.value.force_new == true) | .key]' \
  /tmp/tf-schema-azurerm-4.20.0.json
```

### Inspect a single attribute

```bash
jq '.provider_schemas["registry.terraform.io/hashicorp/azurerm"].resource_schemas.azurerm_storage_account.block.attributes.account_replication_type' \
  /tmp/tf-schema-azurerm-4.20.0.json
```

### List nested blocks and their cardinality

```bash
jq '.provider_schemas["registry.terraform.io/hashicorp/azurerm"].resource_schemas.azurerm_storage_account.block.block_types | to_entries | map({name: .key, nesting_mode: .value.nesting_mode, min_items: .value.min_items, max_items: .value.max_items})' \
  /tmp/tf-schema-azurerm-4.20.0.json
```

### Inspect attributes inside a specific nested block

```bash
jq '.provider_schemas["registry.terraform.io/hashicorp/azurerm"].resource_schemas.azurerm_storage_account.block.block_types.network_rules.block.attributes' \
  /tmp/tf-schema-azurerm-4.20.0.json
```

### Drill into a doubly-nested block

```bash
jq '.provider_schemas["registry.terraform.io/hashicorp/azurerm"].resource_schemas.azurerm_storage_account.block.block_types.blob_properties.block.block_types.cors_rule.block.attributes' \
  /tmp/tf-schema-azurerm-4.20.0.json
```

### Same queries for a data source

Replace `resource_schemas` with `data_source_schemas`:

```bash
jq '.provider_schemas["registry.terraform.io/hashicorp/azurerm"].data_source_schemas.azurerm_storage_account.block.attributes' \
  /tmp/tf-schema-azurerm-4.20.0.json
```

### Inspect provider-level arguments

```bash
jq '.provider_schemas["registry.terraform.io/hashicorp/azurerm"].provider.block.attributes' \
  /tmp/tf-schema-azurerm-4.20.0.json

jq '.provider_schemas["registry.terraform.io/hashicorp/azurerm"].provider.block.block_types.features.block.block_types | keys' \
  /tmp/tf-schema-azurerm-4.20.0.json
```

### Search for a partial attribute name across a resource

When you suspect an attribute exists but aren't sure of the exact name:

```bash
jq -r '.provider_schemas["registry.terraform.io/hashicorp/azurerm"].resource_schemas.azurerm_storage_account.block.attributes | keys[] | select(test("identity"; "i"))' \
  /tmp/tf-schema-azurerm-4.20.0.json
```

## What the Schema Tells You

The provider schema is authoritative for these properties:

- **Argument names** — every flat attribute and every nested block, at every depth.
- **Types** — primitive (`string`, `number`, `bool`) and complex (`list(string)`, `set(object({...}))`, `map(string)`).
- **`required`** — must be set; absence is a plan-time error.
- **`optional`** — may be set; defaults are applied if not.
- **`computed`** — exported; cannot be set as input (or, if both `optional` and `computed`, may be set or read).
- **`force_new`** — changing the value triggers resource recreation (destroy + create).
- **`deprecated`** — string message describing the deprecation. If non-null, do not author new HCL using this attribute.
- **`sensitive`** — the value is treated as sensitive in plan output and state.
- **Nested-block shape** — `nesting_mode` (`single`, `list`, `set`, `map`), `min_items`, `max_items`, and all inner attributes recursively.
- **Some** `ConflictsWith` / `RequiredWith` / `ExactlyOneOf` / `AtLeastOneOf` rules — these sometimes surface in a `validators` field, but coverage in the JSON output is incomplete.

## What the Schema Does Not Tell You

Be honest about the gaps. The schema cannot tell you:

- **Operational consequences** — "changing this triggers a 30-minute downtime", "this attribute can only be increased, never decreased", "modifying this drains the connection pool".
- **Allowed value sets** — for many string attributes (SKU names, pricing tiers, region names), the schema just says `"type": "string"` and the actual allowed values are enforced by the Azure API at apply time.
- **Why an attribute exists** — the schema gives you the shape, not the design rationale.
- **Runtime interactions between attributes** — two attributes may interact at runtime (e.g. one only applies when another is set to a specific value) without that relationship being declared in the schema.
- **Examples** — the schema has no example HCL.
- **Migration guidance** — when an attribute is deprecated, the schema may say "use `X` instead" in the deprecation message, but the actual migration steps live in the registry docs and provider CHANGELOG.
- **Some validation rules** — `ConflictsWith`, `RequiredWith`, `ExactlyOneOf`, `AtLeastOneOf` are sometimes present as a `validators` field, but coverage is uneven across providers and resources. If the schema doesn't mention a relationship you suspect exists, check the docs.

When the schema doesn't answer your question, fall back to the registry docs (see [Fallback to Registry Docs](#fallback-to-registry-docs)) — not training-data recall.

## Multi-Version Handling

A single conversation may touch multiple modules with different provider pins. For example:
- `terraform-azurerm-aks` pins `azurerm ~> 4.12.0`.
- `terraform-azurerm-keyvault` pins `azurerm ~> 3.85.0`.

The two providers have meaningfully different schemas — `azurerm 4.x` introduced renames, removed legacy attributes, and changed defaults. **Never share a schema cache across versions.**

Rules:
1. Always re-extract the version from the target module's `required_providers` before querying.
2. Resolve the constraint to a concrete version (the resolved version from `.terraform.lock.hcl`, or the highest version compatible with the constraint if you're using a shim).
3. Use that concrete version in the cache filename: `/tmp/tf-schema-<provider-short-name>-<version>.json`.
4. If you switch from one module to another mid-session, switch cache files.
5. Never carry a recollection like "I just looked up `azurerm_storage_account` for the AKS module" into work on the keyvault module — the keyvault module pins a different version and the answer may differ.

When in doubt, re-query. The query itself is a single `jq` invocation against an already-cached JSON file — it costs nothing.

## Cost-Awareness and Caching

`terraform init` is the expensive step. It:
- Downloads the provider binary (azurerm 4.x is ~100MB).
- Verifies checksums.
- Sets up `.terraform/` in the module directory.

This takes 10–30 seconds on a fast connection, longer on a slow one. Avoid running it more than once per module per session.

`terraform providers schema -json` is fast (1–3 seconds) once the provider is initialized. The 30–50 MB JSON dump is also fast to write to disk.

`jq` queries against a cached JSON file are essentially instant (<100ms).

**Caching rules:**
- Cache the JSON to `/tmp/tf-schema-<provider>-<version>.json`. Reuse across many edits in the same session.
- Re-init only if the version constraint changed (e.g. you bumped the pin in `versions.tf`).
- If you finish a long session and start a new one, the cache may still be on disk and reusable — but verify the file's age and the resolved version still matches before trusting it.
- Do not commit cache files. They belong in `/tmp` or a gitignored scratch directory.

**Anti-patterns:**
- Re-running `terraform init` between every `jq` query.
- Reading the full JSON file with `Read` instead of querying with `jq` — that puts 30–50MB of provider schema into your context window for no benefit.
- Caching by module path instead of by provider version — two modules pinning the same provider version can share a cache; two modules pinning different versions cannot.

## Fallback to Registry Docs

When the schema doesn't answer your question (allowed values, operational consequences, runtime interactions, examples), fetch the resource doc page from the Terraform Registry. The URL pattern is:

```
https://registry.terraform.io/providers/hashicorp/<provider>/<version>/docs/resources/<resource_short_name>
https://registry.terraform.io/providers/hashicorp/<provider>/<version>/docs/data-sources/<data_source_short_name>
```

For example, for `azurerm_storage_account` at version `4.20.0`:

```
https://registry.terraform.io/providers/hashicorp/azurerm/4.20.0/docs/resources/storage_account
```

Note the resource short name drops the `azurerm_` prefix in the URL.

Use `WebFetch` to retrieve the page. The registry renders these from Markdown files in the provider's GitHub repository, so the content is authoritative and version-specific.

**Prefer the registry docs over training-data recall** for any of:
- Allowed values for SKU / tier / region / kind attributes.
- Examples of how a nested block is typically composed.
- Migration notes for deprecated attributes.
- ConflictsWith / RequiredWith pairs that the schema doesn't surface.
- Operational caveats called out in the docs (`NOTE`, `IMPORTANT` callouts).

If the registry doesn't have the version you need (rare), fall back to the GitHub source: `https://github.com/hashicorp/terraform-provider-azurerm/blob/v<version>/website/docs/r/<resource_name>.html.markdown`.

## Common Mistakes

1. **Authoring HCL from memory without checking the schema.** The argument you remember as `storage_account_name` may have been renamed, the type may have changed, or the attribute may now be deprecated. Always verify before writing.
2. **Ignoring `force_new` and getting surprised by recreation in plan.** A change you thought would update the resource in place actually destroys and recreates it — visible in the plan only after you've already authored the change. Check `force_new` first.
3. **Not noticing a `deprecated` attribute** and writing new HCL that uses it. New code should never use deprecated attributes; existing code that uses them should be migrated, not extended.
4. **Mixing schemas across versions.** Querying the `4.20.0` cache while editing a module pinned to `3.85.0` produces wrong answers. Always re-extract the version from `required_providers` and switch cache files.
5. **Dumping the entire 30–50 MB schema into context** instead of querying with targeted `jq` expressions. Wastes context and surfaces nothing useful.
6. **Falling back to `WebFetch` when the schema would have answered the question.** The schema is faster, more precise, and version-exact. Reach for the docs only when you've confirmed the schema doesn't have what you need (operational consequences, allowed values, etc).
7. **Re-running `terraform init` between every query.** It's a 10–30 second operation. Run it once per session per module and cache the JSON.
8. **Trusting a stale cache.** If the module's `versions.tf` changed mid-session (a constraint bump, a provider added or removed), invalidate and re-init.

## Review Checklist

- [ ] `versions.tf` was read and the provider constraint resolved to a concrete version before any HCL was authored.
- [ ] `terraform init` ran successfully for the target module (or a version-matched shim).
- [ ] The schema JSON is cached at `/tmp/tf-schema-<provider>-<version>.json` keyed by the resolved version.
- [ ] Every argument written exists in the schema at the correct level (top-level vs nested block).
- [ ] Every argument's type in HCL matches the type in the schema.
- [ ] Every `required` attribute on the resource is present in the HCL.
- [ ] No deprecated attributes are used in new code (or each use is intentional and documented as a migration step).
- [ ] All `force_new` attributes set on the resource were chosen deliberately, with awareness that changing them recreates the resource.
- [ ] Nested blocks honor `min_items` / `max_items` from the schema.
- [ ] No `ConflictsWith` / `ExactlyOneOf` / `RequiredWith` rules surfaced in the schema's `validators` field are violated.
- [ ] When working across multiple modules, each module's lookups used its own version-keyed cache.
- [ ] Where the schema had no answer (allowed values, operational consequences), the registry docs were fetched via `WebFetch` for the matching version — not answered from training-data recall.
