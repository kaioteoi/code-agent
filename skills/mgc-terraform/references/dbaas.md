# MGC Database as a Service (DBaaS) Resources

## mgc_dbaas_instances

```hcl
resource "mgc_dbaas_instances" "primary" {
  name           = "prod-db"
  engine_name    = "mysql"
  engine_version = "8.0"
  instance_type  = { name = "cloud-dbaas.c1.small" }
  volume = {
    size = 50    # GB
    type = "cloud_nvme20k"
  }
  user     = "admin"
  password = var.db_password
}
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | ✅ | Instance name |
| `engine_name` | string | ✅ | Database engine |
| `engine_version` | string | ✅ | Engine version |
| `instance_type` | block | ✅ | Compute flavor |
| `volume` | block | ✅ | Storage config |
| `user` | string | ✅ | Admin username |
| `password` | string | ✅ | Admin password (mark sensitive!) |
| `backup_retention_days` | number | | Default 7 |
| `backup_start_at` | string | | e.g. `03:00` (UTC) |
| `availability_zone` | string | | AZ for the instance |

### Supported Engines
| `engine_name` | `engine_version` |
|---------------|-----------------|
| `mysql` | `8.0`, `5.7` |
| `postgresql` | `15`, `14`, `13` |

### Instance Types
| Name | vCPU | RAM |
|------|------|-----|
| `cloud-dbaas.c1.small` | 2 | 4 GB |
| `cloud-dbaas.c1.medium` | 4 | 8 GB |
| `cloud-dbaas.c1.large` | 8 | 16 GB |
| `cloud-dbaas.c1.xlarge` | 16 | 32 GB |

Exported: `id`, `status`, `address` (connection endpoint), `port`, `engine_name`, `engine_version`

---

## mgc_dbaas_instances_replicas

Read replicas for horizontal read scaling:

```hcl
resource "mgc_dbaas_instances_replicas" "replica" {
  name        = "prod-db-replica"
  source_id   = mgc_dbaas_instances.primary.id
  instance_type = { name = "cloud-dbaas.c1.small" }
  volume = {
    size = 50
    type = "cloud_nvme20k"
  }
}
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | ✅ | Replica name |
| `source_id` | string | ✅ | Primary instance ID |
| `instance_type` | block | ✅ | Can differ from primary |
| `volume` | block | ✅ | Must be >= primary volume size |

---

## Security Best Practices

Always use sensitive variables for passwords:

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

resource "mgc_dbaas_instances" "db" {
  # ...
  password = var.db_password
}
```

Set via environment variable:
```bash
export TF_VAR_db_password="your-secure-password"
```

---

## Full HA Database Example

```hcl
resource "mgc_dbaas_instances" "primary" {
  name           = "app-db-primary"
  engine_name    = "postgresql"
  engine_version = "15"
  instance_type  = { name = "cloud-dbaas.c1.medium" }
  volume = {
    size = 100
    type = "cloud_nvme20k"
  }
  user                  = "appuser"
  password              = var.db_password
  backup_retention_days = 14
  backup_start_at       = "02:00"
}

resource "mgc_dbaas_instances_replicas" "replica1" {
  name        = "app-db-replica-1"
  source_id   = mgc_dbaas_instances.primary.id
  instance_type = { name = "cloud-dbaas.c1.small" }
  volume = {
    size = 100
    type = "cloud_nvme20k"
  }
}

output "db_endpoint" {
  value     = mgc_dbaas_instances.primary.address
  sensitive = false
}

output "db_port" {
  value = mgc_dbaas_instances.primary.port
}
```

---

## Import

```bash
terraform import mgc_dbaas_instances.primary <instance-uuid>
terraform import mgc_dbaas_instances_replicas.replica <replica-uuid>
```
