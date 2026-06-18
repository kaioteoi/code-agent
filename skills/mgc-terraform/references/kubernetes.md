# MGC Kubernetes Resources

## mgc_kubernetes_cluster

```hcl
resource "mgc_kubernetes_cluster" "prod" {
  name                 = "prod-cluster"
  version              = "v1.29.0"
  enabled_server_group = true
  description          = "Production Kubernetes cluster"
}
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | ✅ | Cluster name |
| `version` | string | ✅ | Kubernetes version (e.g. `v1.29.0`) |
| `enabled_server_group` | bool | | Enable server group for HA |
| `description` | string | | Optional description |

Exported: `id`, `name`, `version`, `status`, `created_at`

---

## mgc_kubernetes_nodepool

```hcl
resource "mgc_kubernetes_nodepool" "workers" {
  name        = "workers"
  cluster_id  = mgc_kubernetes_cluster.prod.id
  flavor_name = "cloud-k8s.gp2.small"
  replicas    = 3

  auto_scale = {
    min_replicas = 2
    max_replicas = 10
  }
}
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | ✅ | Nodepool name |
| `cluster_id` | string | ✅ | Parent cluster ID |
| `flavor_name` | string | ✅ | Node flavor (see below) |
| `replicas` | number | ✅ | Initial node count |
| `auto_scale` | block | | Auto-scaling config |
| `availability_zones` | list(string) | | AZs to distribute nodes across |

### `auto_scale` Block
```hcl
auto_scale = {
  min_replicas = 2
  max_replicas = 10
}
```

### Node Flavors
| Flavor | vCPU | RAM |
|--------|------|-----|
| `cloud-k8s.gp2.small` | 2 | 4 GB |
| `cloud-k8s.gp2.medium` | 4 | 8 GB |
| `cloud-k8s.gp2.large` | 8 | 16 GB |
| `cloud-k8s.gp2.xlarge` | 16 | 32 GB |

---

## Data Sources

### mgc_kubernetes_cluster_kubeconfig

```hcl
data "mgc_kubernetes_cluster_kubeconfig" "prod" {
  cluster_id = mgc_kubernetes_cluster.prod.id
}

# Save kubeconfig to file
resource "local_file" "kubeconfig" {
  content  = data.mgc_kubernetes_cluster_kubeconfig.prod.kubeconfig
  filename = "${path.module}/kubeconfig.yaml"
}
```

---

## Full Production Cluster Example

```hcl
resource "mgc_kubernetes_cluster" "prod" {
  name                 = "prod-k8s"
  version              = "v1.29.0"
  enabled_server_group = true
  description          = "Production workload cluster"
}

# General worker pool
resource "mgc_kubernetes_nodepool" "workers" {
  name        = "workers"
  cluster_id  = mgc_kubernetes_cluster.prod.id
  flavor_name = "cloud-k8s.gp2.medium"
  replicas    = 3

  auto_scale = {
    min_replicas = 2
    max_replicas = 20
  }
}

# Dedicated monitoring pool
resource "mgc_kubernetes_nodepool" "monitoring" {
  name        = "monitoring"
  cluster_id  = mgc_kubernetes_cluster.prod.id
  flavor_name = "cloud-k8s.gp2.large"
  replicas    = 2
}

# Get kubeconfig
data "mgc_kubernetes_cluster_kubeconfig" "prod" {
  cluster_id = mgc_kubernetes_cluster.prod.id
}

output "kubeconfig" {
  value     = data.mgc_kubernetes_cluster_kubeconfig.prod.kubeconfig
  sensitive = true
}
```

---

## Import

```bash
terraform import mgc_kubernetes_cluster.prod <cluster-uuid>
terraform import mgc_kubernetes_nodepool.workers <cluster-uuid>/<nodepool-name>
```
