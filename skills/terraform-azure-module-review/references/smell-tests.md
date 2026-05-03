# Smell Tests: Patterns That Demand Investigation

Every pattern in this document should trigger deeper analysis when encountered during a review. These aren't automatically wrong — but each one has a high prior probability of being wrong in Terraform Azure modules based on accumulated review experience.

The catalog is organized into three tiers:

- **Tier 1: Almost Always Wrong** — `>90%` wrong. Flag as Major or Critical unless there is a documented, compelling reason.
- **Tier 2: Often Wrong** — `>50%` wrong. Has legitimate use cases; investigate before flagging.
- **Tier 3: Context-Dependent** — May or may not be issues; requires deeper investigation.

---

## Tier 1: Almost Always Wrong

### Data Sources for Values That Should Be Inputs

```hcl
# SMELL — Looking up a value that the consumer already knows
data "azurerm_resource_group" "main" {
  name = var.resource_group_name
}

# SMELL — Looking up identity info that should be passed explicitly
data "azuread_service_principal" "cluster" {
  client_id = var.client_id
}
```

**Question:** "Why is the module looking this up instead of accepting it as an input?"

**Why it matters:** Data sources create runtime dependencies, require additional API permissions, can fail due to timing issues, and make the module slower. The consumer already has these values — they should pass them in.

**Exception:** Only acceptable when the module genuinely needs a computed attribute that the consumer cannot provide (e.g., a resource ID that's only available after querying). Even then, document why.

**Standard response:** *"Avoid data calls as much as possible. Pass values explicitly as variables instead of looking them up at runtime."*

---

### Service Principal Creation

```hcl
# SMELL — Module creating an SPN
resource "azuread_application" "main" { ... }
resource "azuread_service_principal" "main" { ... }
module "service_principal" { ... }
```

**Question:** "Why is this module creating a service principal?"

**Why it matters:** SPNs are centrally managed in a dedicated repository. Individual modules must never create, rotate, or manage service principals.

**Standard response:** *"You are not supposed to manage SPNs at all. There are centralized repos for this."*

**No exceptions for certified modules.** Test infrastructure may create SPNs for CI/CD, but the certified module itself must not.

---

### Role Assignments Inside the Module

```hcl
# SMELL — Direct role assignment
resource "azurerm_role_assignment" "contributor" {
  scope                = var.resource_group_id
  role_definition_name = "Contributor"
  principal_id         = var.principal_id
}
```

**Question:** "Why isn't this delegated to the authorization module?"

**Standard response:** *"This should be done by the authorization module."*

**Pattern for delegation:**

```hcl
# CORRECT — Delegate to the authorization module
module "role_assignments" {
  source   = "<your-terraform-registry>/azure/authorization/azurerm//modules/role_assignment"
  for_each = local.role_assignments
  # ...
}
```

---

### `lower()` / `upper()` / `title()` Wrapping

```hcl
# SMELL — Transforming input instead of validating it
primary_name = lower("${var.brand}-${var.project_name}")
```

**Question:** "Any reason `lower()` is required here? Don't overload the code for no reason."

**Why it matters:** If the input needs to be lowercase, add a regex validation to the variable. Wrapping in `lower()` masks bad input instead of rejecting it, and adds unnecessary complexity.

**Fix:**

```hcl
# Add validation to the variable
variable "brand" {
  validation {
    condition     = can(regex("^[a-z0-9]+$", var.brand))
    error_message = "Brand must be lowercase alphanumeric."
  }
}

# Use directly — no lower() needed
primary_name = "${var.brand}-${var.project_name}"
```

---

### `default = ""` on String Variables

```hcl
# SMELL — Empty string default
variable "custom_domain" {
  type    = string
  default = ""
}
```

**Question:** "Should this default to `null` instead of `\"\"`? They have different semantics."

**Why it matters:** In Terraform, `null` means "not set — use the provider default." `""` means "set to an empty string," which is almost never what you want. Empty strings cause unexpected behavior in `coalesce()`, `format()`, and provider APIs.

**Fix:**

```hcl
variable "custom_domain" {
  type    = string
  default = null
}
```

---

### `default = null` on Complex Objects

```hcl
# SMELL — Null default on a complex variable
variable "network_config" {
  type = object({
    vnet_id   = string
    subnet_id = optional(string)
  })
  default = null  # WRONG
}
```

**Question:** "Complex variables shouldn't be nullable. Use `default = {}` with `nullable = false`."

**Why it matters:** A null complex object means every access requires null checking. `default = {}` with `nullable = false` ensures the caller always gets a predictable structure, and `optional()` attributes handle individual missing fields.

**Fix:**

```hcl
variable "network_config" {
  type = object({
    vnet_id   = optional(string)
    subnet_id = optional(string)
  })
  default  = {}
  nullable = false
}
```

---

### Plaintext Credentials as Variable Inputs

```hcl
# SMELL — Passing credentials as direct string variables
variable "identity" {
  type = object({
    client_id     = string  # Raw credential value passed in
    client_secret = string  # Raw credential value passed in
  })
  sensitive = true
}

# Used directly in resource:
service_principal {
  client_id     = var.identity.client_id
  client_secret = var.identity.client_secret
}
```

**Question:** "Why are credentials passed as plaintext strings instead of being read from Key Vault? They must be referenced as Key Vault objects."

**Why it matters:** Plaintext credential variables appear in Terraform state, plan files, CI/CD logs, and `.tfvars` files. Modules should accept **Key Vault secret names** and resolve credentials at apply time via `data "azurerm_key_vault_secret"`, or accept the Key Vault resource ID and secret names so the module can look them up securely.

**Correct pattern — accept KV references, resolve at apply time:**

```hcl
variable "identity" {
  type = object({
    client_id_secret_name     = optional(string)  # KV secret name containing client ID
    client_secret_secret_name = optional(string)  # KV secret name containing client secret
    rp_object_id              = optional(string)
  })
  default  = {}
  nullable = false
}

# Resolve from Key Vault
data "azurerm_key_vault_secret" "client_id" {
  count        = var.keyvault.enabled ? 1 : 0
  name         = var.identity.client_id_secret_name
  key_vault_id = var.keyvault.resource_id
}
```

**When plaintext is acceptable:** Test infrastructure (`terratest/plan/`) may pass credentials directly for CI/CD convenience. The certified module itself must not expose raw credentials in its variable interface.

---

### `sensitive = false` Declarations

```hcl
# SMELL — Declaring the default value
variable "location" {
  type      = string
  sensitive = false  # Remove this line
}
```

**Question:** "`sensitive = false` is the default. Why is it declared?"

**Fix:** Remove the line entirely.

---

## Tier 2: Often Wrong

These patterns are wrong more often than not, but have legitimate use cases.

### `depends_on` Usage

```hcl
# SMELL — Explicit dependency
resource "azurerm_redhat_openshift_cluster" "main" {
  depends_on = [module.role_assignments]
  # ...
}
```

**Question:** "Why `depends_on` here? Use it as a last resort. Is there no attribute reference that establishes the dependency?"

**When it's acceptable:**

- Module outputs can't create implicit dependencies (module-to-module).
- Timing-sensitive operations (RBAC propagation before resource creation).
- Provider-specific ordering requirements with no attribute link.

**When it's wrong:**

- A simple attribute reference would work: `subnet_id = azurerm_subnet.main.id`.
- Used "just in case" without understanding the actual dependency.

---

### `dynamic` Blocks

```hcl
# SMELL — Dynamic block where a direct block would work
dynamic "timeouts" {
  for_each = length(keys(var.timeouts)) > 0 ? [var.timeouts] : []
  content {
    create = try(timeouts.value.create, null)
  }
}
```

**Question:** "Is the dynamic block necessary? Can a direct block work instead?"

**When `dynamic` is justified:**

- Repeating nested blocks (multiple `ingress` rules, multiple `ip_configuration`).
- Truly optional nested blocks where the block's presence changes behavior.

**When `dynamic` is overkill:**

- `timeouts` blocks (null values use provider defaults anyway).
- Single nested blocks with optional attributes.
- Blocks that are always present but with variable content.

---

### `coalesce(try(..., null), ...)`

```hcl
# SMELL — Layered fallback
domain = coalesce(try(var.cluster_profile.domain, null), local.custom_domain)
```

**Question:** "Don't overload the code. Can this be simplified?"

**Simplifications:**

```hcl
# Option 1: try() with fallback
domain = try(var.cluster_profile.domain, local.custom_domain)

# Option 2: lookup() with fallback
domain = lookup(var.cluster_profile, "domain", local.custom_domain)

# Option 3: With nullable = false on the variable, direct access
domain = var.cluster_profile.domain != null ? var.cluster_profile.domain : local.custom_domain
```

**Note:** `coalesce()` treats `""` as falsy, which may not be intended. `try()` only catches errors, not empty strings. Know the difference.

---

### `lifecycle { ignore_changes = [tags] }`

```hcl
# SMELL — Ignoring ALL tag changes
lifecycle {
  ignore_changes = [tags]
}
```

**Question:** "Don't ignore tags completely. This hides drift from meaningful tag changes."

**Fix:** Target specific externally-managed tags:

```hcl
lifecycle {
  ignore_changes = [
    tags["costcenter"],
    tags["projectcode"],
    tags["environment"]
  ]
}
```

---

### Resource Names That Aren't `main`

```hcl
# SMELL — Non-standard resource name
resource "azurerm_redhat_openshift_cluster" "cluster01" { ... }
resource "azurerm_key_vault" "kv" { ... }
```

**Question:** "Is there more than one of this resource type in the module?"

- If YES: use meaningful names (`primary`, `secondary`, or descriptive names).
- If NO: use `main`.

**Standard response:** Rename to `main` for single resources. Never use numbered suffixes.

---

### `for_each` with Hardcoded Keys

```hcl
# SMELL — Hardcoded map keys
locals {
  role_assignments = {
    "cluster01-vnet-contributor" = {
      scope = var.vnet_id
      role  = "Network Contributor"
    }
    "cluster01-subnet-contributor" = {
      scope = var.subnet_id
      role  = "Network Contributor"
    }
  }
}
```

**Question:** "Why are the keys hardcoded with `cluster01`? Derive them from variables."

**Why it matters:** Hardcoded keys create problems when:

- The module is used with different cluster names (keys collide in state).
- Keys need to change (forces state move).
- They embed assumptions about the deployment.

**Fix:**

```hcl
locals {
  role_assignments = {
    "vnet-contributor" = {
      scope = var.network.vnet_id
      role  = "Network Contributor"
    }
    "master-subnet-contributor" = {
      scope = var.network.master_subnet_id
      role  = "Network Contributor"
    }
  }
}
```

---

### `lifecycle { prevent_destroy = true }`

```hcl
# SMELL — Lifecycle prevention in a module
lifecycle {
  prevent_destroy = true
}
```

**Question:** "You can't use variables to control `lifecycle`. Use a management lock instead."

**Why it matters:** `prevent_destroy` in a module can't be overridden by the consumer. Use Azure management locks at the workspace level instead, which can be managed independently of the module.

---

## Tier 3: Context-Dependent

These patterns may or may not be issues — they require deeper investigation.

### Entire Object Marked `sensitive = true`

```hcl
variable "identity" {
  type = object({
    object_id     = string  # Not sensitive — Azure AD object ID
    client_id     = string  # Sensitive — credential
    client_secret = string  # Sensitive — credential
  })
  sensitive = true  # Marks EVERYTHING sensitive
}
```

**Question:** "Only `client_id` and `client_secret` are sensitive. Marking the whole object obscures plan output for non-sensitive fields."

**Trade-off:** Restructuring may complicate the variable interface. Sometimes accepting over-sensitivity is the pragmatic choice. Flag as Major but note the trade-off.

---

### Variables Without Validation

```hcl
# SMELL — No validation
variable "vm_size" {
  description = "VM size for worker nodes."
  type        = string
  default     = "Standard_D4s_v3"
}
```

**Question:** "Can this input be constrained? Would bad input cause a confusing error?"

**When validation is essential:**

- Inputs with known valid values (environment, location, brand).
- Inputs with format constraints (resource names, CIDRs, GUIDs).
- Inputs where bad values cause cryptic Azure API errors.

**When validation is optional:**

- Free-form string inputs where any value could be valid.
- Values that Azure validates with clear error messages.

---

### Output Prefixed with Module Name

```hcl
# SMELL — Redundant prefix
output "aro_cluster_id" { ... }
output "aro_console_url" { ... }
```

**Question:** "The module is already named `aro` — no reason to add it to every output."

**Fix:**

```hcl
output "cluster_id" { ... }
output "console_url" { ... }
```

---

### Missing `count` Guard on Feature-Dependent Resource

```hcl
# The toggle
variable "keyvault" {
  type = object({
    enabled     = optional(bool, false)
    resource_id = optional(string)
  })
}

# SMELL — No count guard
data "azurerm_key_vault_secret" "pull_secret" {
  name         = var.keyvault.pull_secret_name
  key_vault_id = var.keyvault.resource_id  # null when disabled!
}
```

**This is Critical, not Minor.** Trace the entire conditional chain:

1. Find the toggle: `var.keyvault.enabled`.
2. List every resource/data source that uses `var.keyvault.*`.
3. Verify each has `count = var.keyvault.enabled ? 1 : 0`.
4. Verify references use `[0]` indexing or `try()`.
5. Verify null propagation doesn't cause failures.

---

### Pass-Through Locals

```hcl
# SMELL — Local that just echoes a variable
locals {
  resource_group_name = var.resource_group_name  # No transformation
}
```

**Question:** "What is the reason for this?"

**When acceptable:** If the local is used to compute a value in some cases and pass through in others (e.g., `coalesce(var.override, var.default)`).

**When to flag:** If the local literally just assigns `var.x` to `local.x` with no transformation, remove it and use the variable directly.

---

## Review Application

When reviewing a PR, scan the code for every pattern listed above. For each match:

1. Identify which tier (1, 2, or 3) it falls into.
2. Ask the corresponding question.
3. If the developer has a good answer, document it as an Informational note.
4. If there's no good answer, flag at the appropriate severity:
   - Tier 1: Major or Critical.
   - Tier 2: Major (usually) or Minor.
   - Tier 3: Major, Minor, or Informational depending on context.

A smell is not a verdict. A smell is an invitation to investigate. The investigation produces the finding.
