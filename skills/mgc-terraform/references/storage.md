# MGC Storage Resources

## Block Storage

### mgc_block_storage_volume

⚠️ **Critical:** Volume MUST be in the same Availability Zone as the VM it will attach to.

```hcl
resource "mgc_block_storage_volume" "data" {
  name              = "data-volume"
  size              = 100           # GB
  availability_zone = "br-se1-a"   # must match VM's AZ
  type = {
    name = "cloud_nvme20k"
  }
}
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | ✅ | Volume name |
| `size` | number | ✅ | Size in GB |
| `availability_zone` | string | | e.g. `br-se1-a`, `br-ne1-a` |
| `type` | block | | Volume type (see below) |

**Volume Types:**
| Name | Description |
|------|-------------|
| `cloud_nvme5k` | NVMe, 5k IOPS |
| `cloud_nvme10k` | NVMe, 10k IOPS |
| `cloud_nvme20k` | NVMe, 20k IOPS (most common) |
| `cloud_nvme40k` | NVMe, 40k IOPS |

Exported: `id`, `name`, `size`, `status`, `type.id`, `type.name`, `availability_zone`, `created_at`

---

### mgc_block_storage_attachment

```hcl
resource "mgc_block_storage_attachment" "attach" {
  volume_id   = mgc_block_storage_volume.data.id
  instance_id = mgc_virtual_machine_instances.app.id
}
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `volume_id` | string | ✅ | Block storage volume ID |
| `instance_id` | string | ✅ | VM instance ID |

---

### mgc_block_storage_snapshots

```hcl
resource "mgc_block_storage_snapshots" "snap" {
  name        = "data-vol-snapshot"
  description = "Weekly backup"
  volume_id   = mgc_block_storage_volume.data.id
}
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | ✅ | Snapshot name |
| `volume_id` | string | ✅ | Source volume ID |
| `description` | string | | Optional |

Exported: `id`, `status`, `size`, `created_at`

---

### Complete Block Storage Pattern

```hcl
locals {
  az = "br-se1-a"
}

resource "mgc_virtual_machine_instances" "app" {
  name              = "app-server"
  machine_type      = "BV4-8-160"
  image             = "cloud-ubuntu-24.04 LTS"
  ssh_key_name      = "my-key"
  availability_zone = local.az

  network {
    associate_public_ip = true
  }
}

resource "mgc_block_storage_volume" "data" {
  name              = "app-data"
  size              = 200
  availability_zone = local.az   # same as VM!
  type = { name = "cloud_nvme20k" }
}

resource "mgc_block_storage_attachment" "data" {
  volume_id   = mgc_block_storage_volume.data.id
  instance_id = mgc_virtual_machine_instances.app.id
}
```

---

## Object Storage

Object Storage is S3-compatible. Authentication requires `key_pair_id` + `key_pair_secret` in the provider (not just `api_key`).

### Provider config for Object Storage

```hcl
provider "mgc" {
  region          = "br-se1"
  api_key         = var.api_key
  key_pair_id     = var.obj_storage_key_id      # required for Object Storage
  key_pair_secret = var.obj_storage_key_secret  # required for Object Storage
}
```

### mgc_object_storage_bucket

```hcl
resource "mgc_object_storage_bucket" "assets" {
  bucket                  = "my-company-assets"
  enable_versioning       = true
  recursive_delete        = false  # set true to allow destroy with objects
}
```

| Argument | Type | Description |
|----------|------|-------------|
| `bucket` | string | Globally unique bucket name |
| `enable_versioning` | bool | Enable object versioning |
| `recursive_delete` | bool | Allow destroying bucket with objects inside |

⚠️ If `recursive_delete = false` (default), `terraform destroy` will fail if the bucket has objects.

---

### S3 Remote State Backend

Magalu Object Storage works as a Terraform remote state backend via S3 compatibility:

```hcl
terraform {
  backend "s3" {
    bucket = "my-company-tfstate"
    key    = "prod/terraform.tfstate"
    region = "br-se1"

    # MGC Object Storage endpoint
    endpoint = "https://br-se1.magaluobjects.com"

    # Required overrides for non-AWS S3
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    force_path_style            = true
  }
}
```

Auth for the backend uses `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` env vars,
which should be set to your MGC `key_pair_id` and `key_pair_secret`.

```bash
export AWS_ACCESS_KEY_ID="your-mgc-key-pair-id"
export AWS_SECRET_ACCESS_KEY="your-mgc-key-pair-secret"
```

---

## Import

```bash
terraform import mgc_block_storage_volume.data <volume-uuid>
terraform import mgc_object_storage_bucket.assets <bucket-name>
```
