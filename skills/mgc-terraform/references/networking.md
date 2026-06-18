# MGC Networking Resources

## Networking Hierarchy

```
VPC
 └── Subnet Pool (defines IP range)
      └── Subnet (carves out CIDR from pool)
           └── Interface (NIC attached to a VM)
                └── Security Group (virtual firewall)
Public IP (attached to an Interface)
NAT Gateway (outbound internet for private resources)
```

---

## mgc_network_vpcs

```hcl
resource "mgc_network_vpcs" "main" {
  name        = "main-vpc"
  description = "Main production network"
}
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | ✅ | VPC name |
| `description` | string | | Human-readable description |

Exported: `id`, `name`, `description`, `created_at`

---

## mgc_network_subnetpools

```hcl
resource "mgc_network_subnetpools" "main" {
  name        = "main-pool"
  description = "IP pool for main VPC"
  cidr        = "10.0.0.0/16"
  ip_version  = "IPv4"
}
```

| Argument | Type | Description |
|----------|------|-------------|
| `name` | string | Pool name |
| `cidr` | string | Pool CIDR block |
| `ip_version` | string | `IPv4` or `IPv6` |
| `description` | string | Optional |

---

## mgc_network_vpcs_subnets

```hcl
resource "mgc_network_vpcs_subnets" "web" {
  name             = "web-subnet"
  description      = "Web tier subnet"
  vpc_id           = mgc_network_vpcs.main.id
  subnetpool_id    = mgc_network_subnetpools.main.id
  cidr_block       = "10.0.1.0/24"
  ip_version       = "IPv4"
  dns_nameservers  = ["8.8.8.8", "1.1.1.1"]
}
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | ✅ | Subnet name |
| `vpc_id` | string | ✅ | Parent VPC ID |
| `subnetpool_id` | string | ✅ | Subnet pool to allocate from |
| `cidr_block` | string | ✅ | Must be a subset of the pool CIDR |
| `ip_version` | string | | `IPv4` (default) or `IPv6` |
| `dns_nameservers` | list(string) | | DNS servers |
| `description` | string | | Optional |

---

## mgc_network_vpcs_interfaces

Network interfaces are the connection points between VMs and a VPC.

```hcl
resource "mgc_network_vpcs_interfaces" "web_nic" {
  name   = "web-nic"
  vpc_id = mgc_network_vpcs.main.id

  # IMPORTANT: subnet must exist first
  depends_on = [mgc_network_vpcs_subnets.web]
}
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | ✅ | Interface name |
| `vpc_id` | string | ✅ | Parent VPC |

Exported: `id`, `name`, `vpc_id`, `security_group_id`

Then attach to VM via:
```hcl
resource "mgc_virtual_machine_instances" "vm" {
  # ...
  network {
    interface_id = mgc_network_vpcs_interfaces.web_nic.id
  }
}
```

---

## mgc_network_security_groups

```hcl
resource "mgc_network_security_groups" "web_sg" {
  name                  = "web-sg"
  description           = "Allow HTTP/HTTPS"
  disable_default_rules = false  # keeps default allow-all outbound
}
```

| Argument | Type | Description |
|----------|------|-------------|
| `name` | string | Group name |
| `description` | string | Optional |
| `disable_default_rules` | bool | If true, removes default allow-all egress |

---

## mgc_network_security_groups_rules

```hcl
# Allow inbound HTTP
resource "mgc_network_security_groups_rules" "http_in" {
  security_group_id = mgc_network_security_groups.web_sg.id
  description       = "Allow HTTP"
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 80
  port_range_max    = 80
  remote_ip_prefix  = "0.0.0.0/0"
}

# Allow SSH from specific IP
resource "mgc_network_security_groups_rules" "ssh_in" {
  security_group_id = mgc_network_security_groups.web_sg.id
  description       = "Allow SSH"
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 22
  port_range_max    = 22
  remote_ip_prefix  = "203.0.113.0/24"
}

# Allow all outbound
resource "mgc_network_security_groups_rules" "all_out" {
  security_group_id = mgc_network_security_groups.web_sg.id
  description       = "Allow all outbound"
  direction         = "egress"
  ethertype         = "IPv4"
  protocol          = "any"
  remote_ip_prefix  = "0.0.0.0/0"
}
```

| Argument | Type | Required | Values |
|----------|------|----------|--------|
| `security_group_id` | string | ✅ | |
| `direction` | string | ✅ | `ingress` \| `egress` |
| `ethertype` | string | ✅ | `IPv4` \| `IPv6` |
| `protocol` | string | | `tcp` \| `udp` \| `icmp` \| `any` |
| `port_range_min` | number | | 0–65535 |
| `port_range_max` | number | | 0–65535 |
| `remote_ip_prefix` | string | | CIDR notation |

---

## mgc_network_public_ips

```hcl
resource "mgc_network_public_ips" "web_ip" {
  description = "Web server public IP"
}
```

Exported: `id`, `public_ip` (the actual IP address), `description`

---

## mgc_network_public_ips_attach

```hcl
resource "mgc_network_public_ips_attach" "web" {
  public_ip_id = mgc_network_public_ips.web_ip.id
  interface_id = mgc_network_vpcs_interfaces.web_nic.id
}
```

---

## mgc_network_nat_gateways

For providing outbound internet to private subnets:

```hcl
resource "mgc_network_nat_gateways" "main" {
  name        = "main-nat"
  description = "NAT for private subnet"
  vpc_id      = mgc_network_vpcs.main.id
  zone        = "br-se1-a"
}
```

---

## Full Three-Tier Network Example

```hcl
# VPC
resource "mgc_network_vpcs" "app_vpc" {
  name = "app-vpc"
}

# Subnet Pool
resource "mgc_network_subnetpools" "app_pool" {
  name       = "app-pool"
  cidr       = "10.0.0.0/16"
  ip_version = "IPv4"
}

# Subnets
resource "mgc_network_vpcs_subnets" "web" {
  name          = "web"
  vpc_id        = mgc_network_vpcs.app_vpc.id
  subnetpool_id = mgc_network_subnetpools.app_pool.id
  cidr_block    = "10.0.1.0/24"
  ip_version    = "IPv4"
}

resource "mgc_network_vpcs_subnets" "db" {
  name          = "db"
  vpc_id        = mgc_network_vpcs.app_vpc.id
  subnetpool_id = mgc_network_subnetpools.app_pool.id
  cidr_block    = "10.0.2.0/24"
  ip_version    = "IPv4"
}

# Security Group
resource "mgc_network_security_groups" "web_sg" {
  name = "web-sg"
}

resource "mgc_network_security_groups_rules" "http" {
  security_group_id = mgc_network_security_groups.web_sg.id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 80
  port_range_max    = 80
  remote_ip_prefix  = "0.0.0.0/0"
}
```
