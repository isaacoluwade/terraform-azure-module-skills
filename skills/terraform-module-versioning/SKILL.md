---
name: terraform-module-versioning
description: Apply semantic versioning, branching strategy, and release discipline to Terraform modules. Use this skill when bumping a module's version, deciding whether a change is MAJOR/MINOR/PATCH, writing or reviewing a CHANGELOG entry, updating the VERSION file, cutting a release branch, tagging a release, or backporting a fix. Triggers on: "bump version", "is this a major change", "release this module", "CHANGELOG entry for", "VERSION file", "what version should this be", "breaking change", "cut a release", or any PR that touches `VERSION`, `CHANGELOG.md`, or release branches. This skill is for *modules*, not Terraform providers — for provider versioning rules, see `terraform-provider-versioning`. Apply this even for tiny PRs: a missing CHANGELOG entry or an unbumped VERSION file blocks publishing.
---

# Terraform Module Versioning

Modules are consumed by other teams. Every change you ship is a public API change for somebody. Semantic versioning, a disciplined CHANGELOG, and a predictable branching model are how consumers know whether `terraform init -upgrade` is safe.

This skill covers versioning rules for Terraform *modules*. Provider versioning has different rules — see `terraform-provider-versioning`.

For output-change documentation specifically, see `terraform-module-outputs`.

## Contents
- [Semantic Versioning](#semantic-versioning)
- [VERSION File](#version-file)
- [What Counts as a Breaking Change](#what-counts-as-a-breaking-change)
- [CHANGELOG.md Format](#changelogmd-format)
- [Writing Change Entries](#writing-change-entries)
- [Breaking Changes, Removed vs Changed](#breaking-changes-removed-vs-changed)
- [Upgrade Notes](#upgrade-notes)
- [Branching, Releases, Backporting](#branching-releases-backporting)
- [Pre-1.0 vs Post-1.0](#pre-10-vs-post-10)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)

## Semantic Versioning

Modules follow [SemVer](https://semver.org/) `MAJOR.MINOR.PATCH`:

- **MAJOR** — backward-incompatible changes. Consumers must read the changelog and update their config.
- **MINOR** — backward-compatible new functionality. Safe to upgrade if pinned to `~> X.Y`.
- **PATCH** — backward-compatible bug fixes only. No new variables, no new behaviour, no new resources.

### Bump classification rules

- **New variable, new optional resource, new output, new timeouts block** → MINOR (not PATCH). New surface area is new functionality.
- **Bumping minimum required AzureRM provider** → MINOR at minimum, even if no new feature is exposed. Consumers who pin a lower provider version will be forced to upgrade.
- **Bumping `required_version` for Terraform itself** (e.g. to use `nullable`, `optional()`) → MINOR at minimum.
- **Bug fix that does not change the variable or output surface** → PATCH.
- **Renaming a variable, removing an output, changing a variable type** → MAJOR.
- **A MINOR or MAJOR bump resets PATCH to zero** — `5.1.3` becomes `5.2.0`, never `5.1.4`, when adding a feature.

```
# Good
5.1.3 -> 5.2.0   # added timeouts block + bumped provider min
5.2.0 -> 5.2.1   # fixed null handling in availability_zone validation
5.2.1 -> 6.0.0   # renamed variable, removed deprecated output

# Bad
5.1.3 -> 5.1.4   # added a new variable (PATCH is for fixes only)
5.1.3 -> 5.2.4   # any MINOR bump must reset PATCH to 0
```

Why: PATCH means "drop in safely." If a PATCH adds variables or bumps provider mins, consumers who pin `~> 5.1.0` get surprises.

## VERSION File

A plain text file at the module root containing the version string and nothing else:

```
1.2.0
```

It is referenced from HCL via:

```hcl
locals {
  module_version = trimspace(file("${path.module}/VERSION"))
}
```

Rules:

- Update `VERSION` on every release. The publishing pipeline reads this file — without a bump, the new version will not be published to `<your-terraform-registry>`.
- The string must match the latest release heading in `CHANGELOG.md` exactly.
- Never commit `unreleased` or any non-semver string into `VERSION`. Use a real version number when you cut the release.
- Do not prefix with `v` — `1.2.0`, not `v1.2.0`.

Why a file (rather than only a tag): it lets the module read its own version at plan time and surface it in tags or outputs without a separate build step.

## What Counts as a Breaking Change

For Terraform modules, a change is breaking if any *existing valid caller* could fail to plan, fail to apply, or get a destroy-and-recreate after upgrading. Treat all of the following as MAJOR:

- **Variable removed or renamed.** Even if the rename is "clearly the same thing," consumers' `module "x" { … }` blocks break.
- **Variable type changed** (e.g. `string` → `list(string)`, `object({a=string})` → `object({a=string, b=string})` without `optional()`).
- **Default removed** from a variable, making it effectively required, or **validation tightened** so previously-valid inputs now fail.
- **Output removed, renamed, or its semantics changed** (e.g. an ID output now returns a name). Downstream `module.x.foo` references break.
- **`count` ↔ `for_each` change**, or any **resource address change** (renamed in HCL, moved to a submodule), without a `moved` block. Terraform will plan a destroy + recreate.
- **Minimum AzureRM / Terraform version raised past what consumers commonly pin to.** Treat as breaking when documented expectations of the consumer base shift.
- **Required-side behaviour changes** — e.g. a resource that was optional becomes always-created, or vice versa.

### When `moved` blocks save you from a MAJOR

If you rename a resource, move it into a submodule, or split it across `for_each`, you can sometimes keep the change MINOR by adding `moved` blocks so existing state migrates cleanly:

```hcl
moved {
  from = azurerm_storage_account.this
  to   = azurerm_storage_account.main
}
```

Without `moved`, Terraform sees a destroy + create — that's breaking. With `moved`, state is rewritten in place — not breaking. Use it whenever an internal address changes.

## CHANGELOG.md Format

Follow [Keep a Changelog](https://keepachangelog.com/en/1.0.0/):

```markdown
# Changelog

All notable changes to this project will be documented in this file.

## [1.2.0] - 2026-05-02

### Added

- Add support for customer-managed encryption keys, see variable `encryption.key_vault_key_id`.
- Add output `private_endpoint_ip_addresses`.

### Fixed

- Fix variable `availability_zone` validation to accept a null value.
```

Release heading rules:

- Format: `## [<version>] - <YYYY-MM-DD>`.
- No `v` prefix.
- Version string must match the `VERSION` file.
- Newest release at the top.

### Change categories (in this order)

1. **Changed** — changes in existing functionality.
2. **Added** — new functionality.
3. **Deprecated** — features that still work but are scheduled for removal.
4. **Removed** — functionality that is gone.
5. **Fixed** — bug fixes.

Why this order: when consumers scan a release, "what might break me" (Changed, Removed) needs to be visible before "what's new."

## Writing Change Entries

- **Imperative mood, present tense.** "Add support for…", "Fix validation of…", "Remove deprecated variable…".
- **One change per bullet.** Don't pack two changes into one entry — split them.
- **End every entry with a period.**
- **Reference the full variable or output path.** Write `` `databricks_cluster.num_workers` ``, not `num_workers`. Consumers grep changelogs for variable names.
- **Every variable surface change appears in the changelog.** Every rename, type change, default change, removal, or addition.
- **Explain why, not just what.** A reader should understand the motivation.

```markdown
# Good
### Added
- Add support to configure the number of worker nodes in a cluster, see variable `databricks_cluster.num_workers`.
- Add `project_name` to default tags so cost reports can attribute spend per project.
- Add variable `keyvault` for grouping Azure Key Vault related parameters.

### Fixed
- Fix a bug that prevented changes to the VNet Integration Subnet configuration from being applied.

# Bad
### Added
- Worker nodes
- `project_name` default tag
- variable `keyvault`

### Fixed
- Fix bug
```

### Fix entries must describe the actual problem

Vague fix entries are worse than useless — readers can't tell if the fix matters to them.

```markdown
# Good
- Fix variable `availability_zone` validation to accept a null value.
- Fix a race condition where the Key Vault access policy could be applied before role assignment propagated.

# Bad
- Fix bug creating cluster with local accounts enabled.
- Fix various issues.
```

If you can't describe the problem clearly enough to write a useful entry, you probably can't be sure you fixed it.

## Breaking Changes, Removed vs Changed

Prefix breaking changes with bold **Breaking:** and list them first within their category:

```markdown
### Removed

- **Breaking:** remove deprecated variables `subnet_name`, `vnet_name`, `vnet_rg` in favor of `subnet_id`.
- Remove unused output `legacy_id` (was always null).

### Changed

- **Breaking:** set minimum required AzureRM provider to `>= 3.39.0`, required by variable `default_node_pool.node_taints`.
```

Rules:

- **Breaking labels must match code behaviour.** If the changelog says a variable is required, it must actually be required in code — no `try()` fallback or default keeping it optional. A "breaking" entry the code doesn't enforce is a documentation bug.
- **One breaking change per bullet** — never bundle.

### Removed vs Changed

- **Removed** — functionality or surface area that is *gone*.
- **Changed** — functionality that is *replaced* or *altered*. Use "Replace X with Y" phrasing.

When you rename a variable, prefer **Removed** with a **Breaking:** prefix — consumers' configs fail with `Unsupported argument`, which is a clearer signal than silently accepting the old name.

```markdown
# Good
### Removed
- **Breaking:** variable `custom_extension` has been replaced by `custom_extensions` to support multiple CustomScript extensions; see the new variable description.

# Bad
### Changed
- Remove variable `custom_extension`.
```

Why rename to break loudly: consumers ignore changelogs. A `Unsupported argument` error at plan time is a feature, not a bug.

## Upgrade Notes

When a release requires consumer action, add an italicised single-sentence upgrade note directly under the release heading. Wrap it in underscores:

```markdown
## [3.2.0] - 2026-05-02

_If you are upgrading: please be sure to provide a value for variable `resource_group_name` — the previous fallback to data source lookup has been removed._

### Removed

- **Breaking:** remove implicit data-source lookup of `resource_group_name`; the variable is now required.
```

Only add an upgrade note when there is real consumer action to take. Empty upgrade notes are noise — drop them.

## Branching, Releases, Backporting

| Branch         | Branched from   | Merges to              | Purpose                |
| -------------- | --------------- | ---------------------- | ---------------------- |
| `main`         | —               | —                      | Latest development     |
| `feature/*`    | `main`          | `main` via PR          | New features (MINOR)   |
| `bugfix/*`     | `main`          | `main` via PR          | Bug fixes (PATCH)      |
| `hotfix/*`     | `releases/X.Y`  | `releases/X.Y` via PR  | Urgent fixes on a release line, then forward-port to `main` |
| `releases/X.Y` | `main`          | —                      | Long-lived release branch (no patch number in the name) |

- Release branches omit the patch: `releases/1.2`, not `releases/1.2.0`. Patches accumulate on the same release branch.
- `feature/*` and `bugfix/*` are short-lived — delete after merge.
- `hotfix/*` is for "production is broken on `releases/1.0`, we can't wait for the next MINOR." Branch from the release branch, then forward-port to `main`.

### Release flow (MINOR example)

```bash
git checkout main && git pull
git checkout -b feature/add-keyvault-integration
# ...HCL changes, bump VERSION 1.0.0 -> 1.1.0, update CHANGELOG.md...
git add VERSION CHANGELOG.md path/to/changes
git commit -m "feat: add Key Vault integration [TICKET-5678]"
git push -u origin feature/add-keyvault-integration
# Open PR to main, merge.

# Cut the release branch:
git checkout main && git pull
git checkout -b releases/1.1
git push -u origin releases/1.1
```

MAJOR follows the same shape with `feature/v2-*` and `releases/2.0`. PATCH uses `bugfix/*` and lands on the existing release branch via backport.

### Commit messages

- Include a ticket reference, e.g. `[TICKET-1234]`.
- Use a Conventional-Commits prefix (`feat:`, `fix:`, `docs:`, `refactor:`, `chore:`).
- Pre-commit hooks must pass — never `--no-verify`.

```
feat: add optional Infoblox DNS integration for ARO [ARO-456]
fix: resolve race condition in Key Vault secret retrieval [ARO-789]
docs: document required Azure RBAC permissions for ARO [ARO-321]
```

### Tagging

After the release branch is cut and the PR is merged, tag the exact commit:

```bash
git checkout releases/1.1 && git pull
git tag -a 1.1.0 -m "Release 1.1.0"
git push origin 1.1.0
```

- Tag name = `VERSION` file contents = changelog heading. No `v` prefix.
- Annotated tags (`-a`), not lightweight — they carry the release date and tagger.
- One tag per published version. Never re-tag; to fix a release, cut a new PATCH.

`<your-terraform-registry>` and `<your-azure-devops-org>/`-hosted registries discover modules by tag. A missing or malformed tag means the version doesn't appear to consumers.

### Backporting

When a fix lands on `main` but an older release line still needs it:

1. Land the fix on `main` first via a `bugfix/*` PR. Note the merge commit SHA.
2. Branch from the affected release branch (`releases/1.0`).
3. `git cherry-pick <sha>` from `main`.
4. Resolve conflicts.
5. Bump `VERSION` for that release line (`1.0.4` → `1.0.5`, not the latest minor).
6. Update `CHANGELOG.md` with a new release heading on that line.
7. PR into the release branch, merge, tag.
8. Repeat for every affected release branch.

```bash
COMMIT_SHA=$(git --no-pager log --oneline -1 main | awk '{print $1}')

git checkout releases/1.0 && git pull
git checkout -b bugfix/fix-dns-timeout-backport-1.0
git cherry-pick "$COMMIT_SHA"
# ...bump VERSION on this line, update CHANGELOG.md...
git push -u origin bugfix/fix-dns-timeout-backport-1.0
# Open PR to releases/1.0, merge, tag.
```

Why: consumers pinned to `~> 1.0` cannot upgrade to `1.1.x` without a new MINOR's worth of validation. A backported `1.0.5` lets them take the fix without changing the surface.

## Pre-1.0 vs Post-1.0

- **`0.y.z`** — initial development. The API is unstable; anything can change in a MINOR. Use this only while the module is in active design and not yet certified.
- **`1.0.0`** — the first stable, public/certified release. Once shipped, the SemVer contract applies strictly.
- **Certified modules start at `1.0.0`**, not `0.1.0`. If a module is ready for general consumption, it gets a stable version on first publish.
- **Use `## [unreleased]`** as the changelog heading until *all* release artefacts are ready (service docs, architecture diagram, tests, examples that pass `terraform plan`). Stamp the real version only when the module is publishable.

```markdown
## [unreleased]

### Added

- Add initial Azure Bastion module implementation.
```

When you flip to `1.0.0`, merge any `unreleased` entries into a single `## [1.0.0] - <date>` heading — never keep both:

```markdown
# Good — single release entry
## [1.0.0] - 2026-05-02

### Added
- Add initial module implementation.
- Add support for configurable network settings.

# Bad — separate unreleased and version
## [unreleased]
### Added
- Add network settings.

## [1.0.0] - 2026-05-02
### Added
- Add initial module implementation.
```

If a reviewer requests additional changes during a PR (rename a variable, bump a provider min, add a validation), update the changelog *before* merge to reflect them. `VERSION` and `CHANGELOG.md` are part of the PR, not afterthoughts.

## Common Mistakes

1. **Bumping PATCH for new functionality.** Adding a variable, output, or resource is MINOR — never PATCH.
2. **MINOR bump that doesn't reset PATCH to 0.** `5.1.3` → `5.2.0`, not `5.2.4`.
3. **Forgetting the `VERSION` file.** The pipeline won't publish without it; reviewers shouldn't have to remind you.
4. **`VERSION` and CHANGELOG out of sync.** They must match exactly, including format.
5. **Changelog says "Breaking:" but the code allows the old behaviour.** A `try()` fallback or default value contradicts the breaking label — fix the code or fix the entry.
6. **Renaming a variable but burying it in `### Changed`.** Use `### Removed` with `**Breaking:**` so consumers get a loud, grep-able signal.
7. **Vague fix entries** like "Fix bug creating cluster" — describe the *actual* problem and *how* it was fixed.
8. **Splitting a single PR's changes across `## [unreleased]` and a stamped version.** Merge unreleased entries into the release on cut.
9. **`count` ↔ `for_each` flip without a `moved` block.** Plans destroy and recreate every resource — that's a MAJOR, full stop.
10. **Tagging with a `v` prefix** (`v1.2.0`). The format is `1.2.0` to match `VERSION` and the changelog heading.
11. **Bumping the AzureRM provider minimum without a MINOR bump.** Even with no new feature, this changes the consumer's required toolchain.
12. **Skipping the upgrade note** on a release that genuinely requires consumer action.

## Review Checklist

- [ ] `VERSION` file is updated and contains a valid SemVer string with no `v` prefix.
- [ ] `VERSION` matches the latest `## [<version>]` heading in `CHANGELOG.md`.
- [ ] Bump type (MAJOR/MINOR/PATCH) matches the actual scope of changes.
- [ ] MINOR or MAJOR bumps reset PATCH to `0`.
- [ ] Every variable, output, and resource surface change has a changelog entry.
- [ ] Every entry uses imperative mood, ends with a period, and references full variable/output paths.
- [ ] Breaking changes are prefixed with `**Breaking:**` and listed first in their category.
- [ ] Every `**Breaking:**` label is enforced by the code (no silent fallbacks).
- [ ] Renamed variables appear under `### Removed` with `**Breaking:**`, not under `### Changed`.
- [ ] Refactors that change resource addresses include `moved` blocks (or are tagged MAJOR).
- [ ] Provider/Terraform minimum bumps are at least MINOR and called out in the changelog.
- [ ] The release heading uses `## [<version>] - <YYYY-MM-DD>` with no `v` prefix.
- [ ] Upgrade notes are present when consumer action is required, and absent otherwise.
- [ ] Branch name follows `feature/`, `bugfix/`, `hotfix/`, or `releases/X.Y` conventions.
- [ ] Commit message includes a ticket reference and a conventional-commit prefix.
