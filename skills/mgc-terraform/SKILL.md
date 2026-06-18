---
name: mgc-terraform
description: >
  Expert guidance for writing, reviewing, and debugging Terraform configurations
  using the official Magalu Cloud (MGC) provider. Use this skill whenever the
  user mentions Magalu Cloud, MGC, magalucloud/mgc, terraform-provider-mgc, or
  is working with any MGC resource types (mgc_virtual_machine_instances,
  mgc_network_vpcs, mgc_kubernetes_cluster, mgc_block_storage_volume,
  mgc_dbaas_*, mgc_object_storage_*, etc.). Also trigger for questions about
  MGC regions, API keys, provider configuration, or IaC on Magalu Cloud.
  Always use this skill before answering any MGC Terraform question — even
  seemingly simple ones — to avoid giving wrong resource names or attributes.
---

# Magalu Cloud (MGC) Terraform Provider Skill

Official provider: `magalucloud/mgc`  
Registry: https://registry.terraform.io/providers/MagaluCloud/mgc/latest/docs  
GitHub: https://github.com/MagaluCloud/terraform-provider-mgc  
Latest stable: **v0.48.0** (May 2026)

---

## Provider Configuration

```hcl
terraform {
  required_providers {
    mgc = {
      source  = "magalucloud/mgc"
      version = "~> 0.48.0"
    }
  }
}

provider "mgc" {
  api_key = var.api_key   # or env var: MGC_API_KEY
  region  = var.region    # or env var: MGC_REGION

  # Required for Object Storage operations (omit if not using Object Storage):
  key_pair_id     = var.mgc_key_pair_id     # distinct from api_key; only issued when OS scopes selected
  key_pair_secret = var.mgc_key_pair_secret
}
```

**Regions:** `br-ne1` (Nordeste 1) | `br-se1` (Sudeste 1, default) | `br-mgl1` | `br-mc1`

**Multi-region pattern:**
```hcl
provider "mgc" { alias = "se"; region = "br-se1" }
provider "mgc" { alias = "ne"; region = "br-ne1" }

resource "mgc_virtual_machine_instances" "se_vm" {
  provider = mgc.se
  # ...
}
```

**Auth via environment variables (recommended for CI/CD):**
```bash
export MGC_API_KEY="your-api-key"
export MGC_REGION="br-se1"
```

---

## Resource Reference

For detailed per-resource docs, read the relevant reference file:

- **Compute (VMs, SSH keys)** → `references/compute.md`
- **Networking (VPCs, subnets, security groups, public IPs)** → `references/networking.md`
- **Storage (block storage, object storage)** → `references/storage.md`
- **Kubernetes** → `references/kubernetes.md`
- **Database as a Service (DBaaS)** → `references/dbaas.md`
- **Container Registry** → `references/container-registry.md`

Read only the file(s) relevant to the user's task.

---

## Canonical Resource Names (Quick Reference)

| Service | Resource Type |
|---------|--------------|
| Virtual Machines | `mgc_virtual_machine_instances` |
| VM Snapshots | `mgc_virtual_machine_snapshots` |
| SSH Keys | `mgc_ssh_keys` |
| Block Storage Volume | `mgc_block_storage_volume` |
| Block Storage Attachment | `mgc_block_storage_attachment` |
| Block Storage Snapshot | `mgc_block_storage_snapshots` |
| VPC | `mgc_network_vpcs` |
| Subnet Pool | `mgc_network_subnetpools` |
| Subnet | `mgc_network_vpcs_subnets` |
| Network Interface | `mgc_network_vpcs_interfaces` |
| Security Group | `mgc_network_security_groups` |
| Security Group Rule | `mgc_network_security_groups_rules` |
| Public IP | `mgc_network_public_ips` |
| Public IP Attach | `mgc_network_public_ips_attach` |
| NAT Gateway | `mgc_network_nat_gateways` |
| Kubernetes Cluster | `mgc_kubernetes_cluster` |
| Kubernetes Nodepool | `mgc_kubernetes_nodepool` |
| DBaaS Instance | `mgc_dbaas_instances` |
| DBaaS Replica | `mgc_dbaas_instances_replicas` |
| Object Storage Bucket | `mgc_object_storage_bucket` |
| Container Registry | `mgc_cr_registries` |

---

## Common Patterns

### 1. VM with Public IP (most common)

```hcl
resource "mgc_virtual_machine_instances" "app" {
  name              = "my-app"
  machine_type      = "BV1-1-40"          # or use machine_type_id
  image             = "cloud-ubuntu-24.04 LTS"  # or use image_id
  ssh_key_name      = mgc_ssh_keys.key.name
  availability_zone = "br-se1-a"

  network {
    associate_public_ip = true
  }
}
```

### 2. VM + Persistent Block Storage

```hcl
resource "mgc_block_storage_volume" "data" {
  name              = "data-vol"
  size              = 50            # GB
  availability_zone = "br-se1-a"   # MUST match VM's AZ
  type = { name = "cloud_nvme20k" }
}

resource "mgc_block_storage_attachment" "attach" {
  volume_id   = mgc_block_storage_volume.data.id
  instance_id = mgc_virtual_machine_instances.app.id
}
```

⚠️ **Critical:** Block Storage volumes MUST be in the same Availability Zone as the VM. Mismatched AZ = provisioning failure.

### 3. S3-Compatible Remote State Backend

```hcl
terraform {
  backend "s3" {
    bucket                      = "my-tfstate-bucket"
    key                         = "prod/terraform.tfstate"
    region                      = "br-se1"
    endpoint                    = "https://br-se1.magaluobjects.com"
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    force_path_style            = true
  }
}
```

### 4. Kubernetes Cluster + Nodepool

```hcl
resource "mgc_kubernetes_cluster" "main" {
  name                 = "prod-cluster"
  version              = "v1.29.0"
  enabled_server_group = true
  description          = "Production cluster"
}

resource "mgc_kubernetes_nodepool" "workers" {
  name       = "workers"
  cluster_id = mgc_kubernetes_cluster.main.id
  flavor_name = "cloud-k8s.gp2.small"
  replicas    = 3

  auto_scale = {
    min_replicas = 2
    max_replicas = 10
  }
}
```

---

## Key Pitfalls to Warn About

1. **Wrong resource names** — The old `mgc_block-storage_volumes` syntax (with hyphens) was deprecated. Current: `mgc_block_storage_volume`.
2. **AZ mismatch** — Block storage volumes fail to attach if they're in a different AZ from the VM.
3. **Object Storage auth** — Object Storage operations require `key_pair_id` + `key_pair_secret` in the provider config, in addition to `api_key`. This key pair is a separate credential (not derived from `api_key`) and is only issued by Magalu Cloud when the API key is created with Object Storage scopes selected.
4. **Credentials in code** — Warn users to use env vars (`MGC_API_KEY`, `MGC_REGION`) or variables with `sensitive = true`, never hardcode.
5. **VPC networking order** — Interface creation depends on subnet existing; always use `depends_on = [mgc_network_vpcs_subnets.xxx]` or implicit references.
6. **State file** — Always recommend remote state (MGC Object Storage with S3 backend) for team use.

---

## Terraform Workflow (standard)

```bash
terraform init      # Download provider, set up backend
terraform plan      # Preview changes (always before apply)
terraform apply     # Apply changes
terraform destroy   # Tear down resources
```

---

## Data Sources

```hcl
# List available images
data "mgc_virtual_machine_images" "available" {}

# List machine types
data "mgc_virtual_machine_machine_types" "available" {}

# Get kubernetes kubeconfig
data "mgc_kubernetes_cluster_kubeconfig" "kube" {
  cluster_id = mgc_kubernetes_cluster.main.id
}

# List SSH keys
data "mgc_ssh_keys" "all" {}
```

---

## When to Read Reference Files

Read the specific reference file when the user is:
- Writing or debugging a configuration for that service
- Asking about specific attributes, constraints, or valid values
- Getting errors related to that resource type

The reference files contain full attribute lists, valid values, and additional examples for each service category.
