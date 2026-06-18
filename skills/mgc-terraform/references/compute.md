# MGC Compute Resources

## mgc_virtual_machine_instances

### Required Arguments
| Argument | Type | Description |
|----------|------|-------------|
| `name` | string | VM name |
| `machine_type` | string | Machine type name (e.g. `BV1-1-40`) — OR use `machine_type_id` |
| `image` | string | Image name (e.g. `cloud-ubuntu-24.04 LTS`) — OR use `image_id` |

### Optional Arguments
| Argument | Type | Description |
|----------|------|-------------|
| `ssh_key_name` | string | SSH key name for Linux access |
| `availability_zone` | string | e.g. `br-se1-a`, `br-ne1-a` |
| `machine_type_id` | string | Use instead of `machine_type` for exact ID |
| `image_id` | string | Use instead of `image` for exact ID |
| `user_data` | string | Cloud-init script |
| `network` | block | Network configuration (see below) |

### `network` Block
```hcl
network {
  associate_public_ip  = true   # assign a public IP
  vpc_id               = mgc_network_vpcs.main.id   # optional, specific VPC
  interface_id         = mgc_network_vpcs_interfaces.iface.id  # optional
  delete_public_ip_on_deletion = true  # optional, default false
}
```

### Exported Attributes
- `id` — VM instance UUID
- `status` — VM state (running, stopped, etc.)
- `created_at`
- `network.public_address` — public IP if associated
- `network.interface.id` — interface ID

### Machine Type Naming Convention
- Format: `BV{vCPU}-{RAM_GB}-{Disk_GB}`
- Examples: `BV1-1-40`, `BV2-2-80`, `BV4-8-160`, `BV8-16-320`

### Common Images
- `cloud-ubuntu-24.04 LTS`
- `cloud-ubuntu-22.04 LTS`
- `cloud-debian-12`
- `cloud-windows-server-2022`

### Full Example
```hcl
resource "mgc_virtual_machine_instances" "web" {
  name              = "web-server"
  machine_type      = "BV2-2-80"
  image             = "cloud-ubuntu-24.04 LTS"
  ssh_key_name      = mgc_ssh_keys.deployer.name
  availability_zone = "br-se1-a"
  user_data         = base64encode(file("cloud-init.yaml"))

  network {
    associate_public_ip = true
  }
}
```

---

## mgc_virtual_machine_snapshots

```hcl
resource "mgc_virtual_machine_snapshots" "backup" {
  name        = "web-server-backup"
  instance_id = mgc_virtual_machine_instances.web.id
}
```

### Attributes
- `id` — snapshot UUID
- `instance_id` — source VM UUID
- `name`
- `status`
- `created_at`

---

## mgc_ssh_keys

SSH keys are **global** (not region-specific).

```hcl
resource "mgc_ssh_keys" "deployer" {
  name = "deployer-key"
  key  = "ssh-ed25519 AAAA... user@host"
}
```

### Arguments
| Argument | Type | Description |
|----------|------|-------------|
| `name` | string | Key name |
| `key` | string | Public key content |

### Data Source
```hcl
data "mgc_ssh_keys" "all" {}

output "keys" {
  value = data.mgc_ssh_keys.all.ssh_keys
}
```

---

## Data Sources

### mgc_virtual_machine_images
```hcl
data "mgc_virtual_machine_images" "all" {
  # optional filters
}

# Use in a VM:
resource "mgc_virtual_machine_instances" "vm" {
  # ...
  image = "cloud-ubuntu-24.04 LTS"
  # OR:
  image_id = data.mgc_virtual_machine_images.all.images[0].id
}
```

### mgc_virtual_machine_machine_types
```hcl
data "mgc_virtual_machine_machine_types" "available" {}
```

---

## Import

```bash
terraform import mgc_virtual_machine_instances.web <instance-uuid>
terraform import mgc_ssh_keys.deployer <key-name>
```
