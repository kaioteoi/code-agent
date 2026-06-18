# MGC Container Registry Resources

## mgc_cr_registries

```hcl
resource "mgc_cr_registries" "app" {
  name = "my-app-registry"
}
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | ✅ | Registry name (must be unique) |

Exported:
- `id` — Registry UUID
- `name`
- `storage_usage_bytes` — Current storage usage
- `created_at`
- `endpoint` — Docker-compatible registry endpoint URL

---

## Usage Pattern

```hcl
resource "mgc_cr_registries" "app" {
  name = "acme-app"
}

output "registry_endpoint" {
  value = mgc_cr_registries.app.endpoint
}
```

Then in CI/CD:
```bash
docker login $(terraform output -raw registry_endpoint)
docker push $(terraform output -raw registry_endpoint)/my-image:latest
```

---

## Using with Kubernetes

```hcl
resource "mgc_cr_registries" "app" {
  name = "acme-app"
}

# Use the registry endpoint in K8s deployment (via Helm or kubernetes provider)
# endpoint format: <registry-name>.cr.<region>.magalucloud.com
output "registry_endpoint" {
  value = mgc_cr_registries.app.endpoint
}
```

---

## Import

```bash
terraform import mgc_cr_registries.app <registry-name>
```
