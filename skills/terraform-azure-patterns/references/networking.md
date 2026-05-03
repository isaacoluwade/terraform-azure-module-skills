# Networking Patterns

Patterns for handling network resources — subnets, VNets, NSGs, private endpoints, DNS — inside an Azure Terraform module. The dominant theme: **the module never resolves network resources at runtime.** Subnet IDs, VNet IDs, route table IDs, and DNS zone IDs are inputs, period.

## Contents
- [Network Resource IDs as Inputs](#network-resource-ids-as-inputs)
- [The `network` Variable Pattern](#the-network-variable-pattern)
- [Public Access Disabled by Default](#public-access-disabled-by-default)
- [Private Endpoints](#private-endpoints)
- [Private Endpoint vs VNet Integration](#private-endpoint-vs-vnet-integration)
- [Private Endpoint Multiplicity](#private-endpoint-multiplicity)
- [Private DNS Integration](#private-dns-integration)
- [External DNS Provider Integration](#external-dns-provider-integration)
- [NSG Rules vs Delegating to a Network Module](#nsg-rules-vs-delegating-to-a-network-module)
- [Firewall and IP Rules](#firewall-and-ip-rules)
- [Azure Policy Enforcement](#azure-policy-enforcement)

## Network Resource IDs as Inputs

The caller's network team owns subnets, VNets, route tables, NSGs, and DNS zones. They have the IDs already. Accept them — don't look them up.

```hcl
# Bad — every one of these is a hidden API call
data "azurerm_subnet" "worker" {
  name                 = var.worker_subnet_name
  virtual_network_name = var.vnet_name
  resource_group_name  = var.vnet_rg_name
}

data "azurerm_virtual_network" "main" {
  name                = var.vnet_name
  resource_group_name = var.vnet_rg_name
}

data "azurerm_private_dns_zone" "blob" {
  name                = "privatelink.blob.core.windows.net"
  resource_group_name = var.dns_rg_name
}
```

```hcl
# Good — the caller resolves and passes IDs
variable "network" {
  description = "Network configuration with pre-resolved resource IDs."
  type = object({
    worker_subnet_id     = string
    master_subnet_id     = string
    route_table_id       = string
    private_dns_zone_id  = string
  })
  nullable = false
}
```

The reasoning is identical to the main "no data sources" rule: explicit inputs are deterministic, testable, fast at plan time, and don't bake hidden subscription dependencies into the module.

## The `network` Variable Pattern

Group every network field into a single `network` variable. Don't scatter `worker_subnet_id`, `master_subnet_id`, `route_table_id`, `dns_zone_id` across the variable surface.

```hcl
variable "network" {
  description = <<-EOT
    Network configuration. All IDs must be resolved by the caller.
    `worker_subnet_id`            - Subnet ID for worker nodes.
    `master_subnet_id`            - Subnet ID for master nodes.
    `route_table_id`              - Route table associated with the cluster subnets.
    `private_dns_zone_id`         - Private DNS zone for cluster service records.
    `pod_cidr`                    - Pod CIDR block.
    `service_cidr`                - Service CIDR block.
    `outbound_type`               - Outbound traffic type (`Loadbalancer` or `UserDefinedRouting`).
    `preconfigured_network_security_group_enabled` - Whether NSGs on the subnets are managed externally.
  EOT
  type = object({
    worker_subnet_id     = string
    master_subnet_id     = string
    route_table_id       = string
    private_dns_zone_id  = string
    pod_cidr             = optional(string, "10.128.0.0/14")
    service_cidr         = optional(string, "172.30.0.0/16")
    outbound_type        = optional(string, "Loadbalancer")
    preconfigured_network_security_group_enabled = optional(bool, false)
  })
  nullable = false
}
```

Reference fields directly — don't unpack into locals when the variable already uses `nullable = false` and `optional()` defaults:

```hcl
# Good — direct reference
network_profile {
  pod_cidr     = var.network.pod_cidr
  service_cidr = var.network.service_cidr
  outbound_type = var.network.outbound_type
}
```

```hcl
# Bad — unnecessary locals layer
locals {
  pod_cidr      = var.network.pod_cidr
  service_cidr  = var.network.service_cidr
  outbound_type = var.network.outbound_type
}

network_profile {
  pod_cidr     = local.pod_cidr
  service_cidr = local.service_cidr
  outbound_type = local.outbound_type
}
```

The locals dance adds noise without clarity when the variable already has a strong shape.

## Public Access Disabled by Default

Every Azure resource that supports a `public_network_access_enabled` (or equivalent toggle) defaults to `false` in your module. The toggle exists for the rare exception, not the common case.

```hcl
variable "public_access_enabled" {
  description = "Whether public network access is enabled. Defaults to false (private only)."
  type        = bool
  default     = false
}

resource "azurerm_storage_account" "main" {
  # ...
  public_network_access_enabled = var.public_access_enabled
}
```

For services with split public/private settings (e.g. ARO with separate `api_server_profile.visibility` and `ingress_profile.visibility`), apply the same default to each:

```hcl
api_server_profile {
  visibility = var.public_access_enabled ? "Public" : "Private"
}

ingress_profile {
  visibility = var.public_access_enabled ? "Public" : "Private"
}
```

Anti-pattern:

```hcl
# Bad — public-by-default
variable "api_visibility" {
  type    = string
  default = "Public"
}
```

## Private Endpoints

When a module owns a single private endpoint for its primary resource, it can create the PE itself and accept the subnet ID as input:

```hcl
variable "privatelink" {
  description = <<-EOT
    Private link configuration.
    `enabled`       - Whether a private endpoint is created.
    `subnet_id`     - Subnet to place the private endpoint in.
    `dns_zone_id`   - Private DNS zone to register the PE in.
  EOT
  type = object({
    enabled     = optional(bool, true)
    subnet_id   = optional(string)
    dns_zone_id = optional(string)
  })
  default  = {}
  nullable = false
}

resource "azurerm_private_endpoint" "main" {
  count = var.privatelink.enabled ? 1 : 0

  name                = "${var.name}-pe"
  location            = var.resource_group.location
  resource_group_name = var.resource_group.name
  subnet_id           = var.privatelink.subnet_id

  private_service_connection {
    name                           = "${var.name}-psc"
    private_connection_resource_id = azurerm_storage_account.main.id
    subresource_names              = ["blob"]
    is_manual_connection           = false
  }

  dynamic "private_dns_zone_group" {
    for_each = var.privatelink.dns_zone_id != null ? [1] : []
    content {
      name                 = "default"
      private_dns_zone_ids = [var.privatelink.dns_zone_id]
    }
  }
}
```

When the module passes through to a child module that creates the resource, surface the same toggle:

```hcl
module "storage_account" {
  source  = "<your-terraform-registry>/azure/storage-account/azurerm"
  version = "~> x.x"

  name                = local.storage_account_name
  resource_group      = var.resource_group
  enable_privatelink  = true

  privatelink = {
    subnet_id = var.network.privatelink_subnet_id
    dns_zone  = var.network.storage_dns_zone_id
  }
}
```

## Private Endpoint vs VNet Integration

These are different features with different use cases. Don't conflate them.

- **Private Endpoint** — a private IP allocated in your VNet that maps to the PaaS service. Traffic to the service flows through that IP. The service itself stays in Microsoft's network. Use `azurerm_private_endpoint`.
- **VNet Integration / Subnet Delegation** — the service is injected into a delegated subnet you own. The service is genuinely "inside" your VNet. Use the resource's `delegated_subnet_id` (or equivalent) field.

Most enterprise patterns standardize on Private Endpoints for ingress and avoid VNet Integration unless the service architecture genuinely requires injection (e.g. App Service plan needing outbound from your VNet, or PostgreSQL Flexible Server in private-access mode).

If your module's documentation references the wrong document — pointing callers at VNet Integration when the module actually uses Private Endpoint — that's a clarity bug. Reference `azurerm_private_endpoint` examples, not subnet-delegation examples.

## Private Endpoint Multiplicity

When a service supports **multiple** private endpoints (different sub-resources, different regions, different consumer subnets), don't embed the PE inside the module. The module creates the service; an example shows the caller how to create PEs against the service ID using a dedicated PE module.

```hcl
# In examples/with-private-endpoint/main.tf:

module "sql_managed_instance" {
  source  = "<your-terraform-registry>/azure/database-sql-managed-instance/azurerm"
  version = "~> x.x"
  # ... no PE config here
}

module "private_endpoint_app" {
  source  = "<your-terraform-registry>/azure/private-endpoint/azurerm"
  version = "~> x.x"

  target_resource_id = module.sql_managed_instance.id
  subnet_id          = var.app_subnet_id
  subresource_names  = ["managedInstance"]
}

module "private_endpoint_data" {
  source  = "<your-terraform-registry>/azure/private-endpoint/azurerm"
  version = "~> x.x"

  target_resource_id = module.sql_managed_instance.id
  subnet_id          = var.data_subnet_id
  subresource_names  = ["managedInstance"]
}
```

The service module exposes the resource ID via output; the PE module is called once per endpoint.

## Private DNS Integration

When a module creates a private endpoint, it usually needs to register the PE's IP in a private DNS zone so that consumers resolve the service's hostname to the private IP rather than the public one.

The DNS zone ID is an input — not looked up:

```hcl
variable "privatelink" {
  type = object({
    subnet_id   = optional(string)
    dns_zone_id = optional(string)
  })
  default  = {}
  nullable = false
}

resource "azurerm_private_endpoint" "main" {
  # ...
  dynamic "private_dns_zone_group" {
    for_each = var.privatelink.dns_zone_id != null ? [1] : []
    content {
      name                 = "default"
      private_dns_zone_ids = [var.privatelink.dns_zone_id]
    }
  }
}
```

The dynamic block lets the caller skip DNS registration (if their environment uses a custom resolver instead of Azure private DNS) by passing `null`.

## External DNS Provider Integration

Some environments register service hostnames in an external DNS provider (e.g. an enterprise IPAM/DNS solution like Infoblox) instead of Azure DNS. When a module integrates with such a provider, the integration is **optional** and **isolated**.

Three rules:

1. **Optional by default.** Use `count` or `for_each` so DNS resources don't appear when the caller doesn't enable the integration.
2. **Separate file.** Move all external-DNS resources to a dedicated file (e.g. `infoblox.tf`) so they're easy to locate and easy to disable.
3. **No PTR records unless asked.** Reverse DNS is usually owned by IPAM; creating PTR records from the module collides with the IPAM-owned record.

```hcl
variable "infoblox" {
  description = <<-EOT
    External DNS provider configuration.
    `enabled`      - Whether DNS records should be created.
    `dns_view`     - The DNS view to create records in.
    `dns_zone`     - The DNS zone for record creation.
    `network_view` - Network view for IP allocation.
  EOT
  type = object({
    enabled      = optional(bool, false)
    dns_view     = optional(string, "default")
    dns_zone     = optional(string)
    network_view = optional(string, "default")
  })
  default  = {}
  nullable = false
}
```

In `infoblox.tf`:

```hcl
resource "infoblox_a_record" "api_server" {
  count = var.infoblox.enabled ? 1 : 0

  fqdn     = "api.${var.cluster_name}.${var.infoblox.dns_zone}"
  ip_addr  = azurerm_redhat_openshift_cluster.main.api_server_profile[0].ip
  dns_view = var.infoblox.dns_view

  lifecycle {
    ignore_changes = [dns_view]
  }
}

resource "infoblox_a_record" "ingress" {
  count = var.infoblox.enabled ? 1 : 0

  fqdn     = "*.apps.${var.cluster_name}.${var.infoblox.dns_zone}"
  ip_addr  = azurerm_redhat_openshift_cluster.main.ingress_profile[0].ip
  dns_view = var.infoblox.dns_view

  lifecycle {
    ignore_changes = [dns_view]
  }
}
```

Resource names describe what they record (`api_server`, `ingress`), not their type (`infoblox_a_record_1`). The `lifecycle.ignore_changes = [dns_view]` accommodates the IPAM controller adjusting `dns_view` post-creation.

(`infoblox_*` is shown above because Infoblox is a real, published Terraform provider — it's a concrete example of an external DNS integration. The same pattern applies to any non-Azure DNS provider.)

## NSG Rules vs Delegating to a Network Module

Two valid approaches:

- **Module manages NSG rules** — fine when the module fully owns a subnet and the NSG attached to it (e.g. a dedicated bastion or jumpbox module).
- **Module delegates NSG rules** — better when the subnet is shared, externally managed, or owned by a network team. In this case the module accepts a flag indicating that NSGs are preconfigured externally:

```hcl
network_profile {
  preconfigured_network_security_group_enabled = var.network.preconfigured_network_security_group_enabled
}
```

When the network team owns NSGs, the module shouldn't try to create or modify rules; it just declares its dependency on a properly-configured subnet.

## Firewall and IP Rules

Resources like Storage Account, Key Vault, and Cosmos DB expose IP-based firewall rules even when private endpoints are the primary access path. Accept these as a list:

```hcl
variable "keyvault" {
  type = object({
    resource_id = optional(string)
    ip_rules    = optional(list(string), [])
  })
  default  = {}
  nullable = false
}
```

An empty list means "no IP exceptions" — combined with `public_network_access_enabled = false`, the resource is reachable only via private endpoint, which is the intended default.

Don't expose a separate `enable_ip_rules` boolean alongside the list — an empty list is sufficient signal.

## Azure Policy Enforcement

Many enterprises enforce private-network deployment via Azure Policy, in addition to module defaults. Document this where appropriate:

```hcl
# Public access is disabled by default. Public access is also enforced by Azure Policy and
# will remain disabled, or service rollout will be denied. In exceptional cases, network
# rules can be configured; however, you must obtain an Azure Policy exemption first.

resource "azurerm_servicebus_namespace" "main" {
  # ...
  public_network_access_enabled = false
}
```

When a module's defaults align with policy, drift between Terraform and policy is impossible. When a caller requests an exemption (passes `public_access_enabled = true`), the module honors the request but the policy still gates the deployment — so the caller must obtain a policy exemption out-of-band.
