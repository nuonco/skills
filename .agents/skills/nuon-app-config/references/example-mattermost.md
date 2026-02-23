# Complete Example: Mattermost

Reference implementation from `nuonco/example-app-configs/mattermost`.

**metadata.toml:** (no type comment)
```toml
version      = "v2"
display_name = "Mattermost"
description  = "Open-source team collaboration platform"
```

**runner.toml:**
```toml
# runner
runner_type = "aws"
helm_driver = "configmap"
init_script_url = "https://raw.githubusercontent.com/nuonco/runner/refs/heads/main/scripts/aws/init-mng-v2.sh"
```

**inputs.toml:**
```toml
# inputs
[[group]]
name         = "mattermost"
display_name = "Application Configurations"
description  = "Mattermost operator and app release versions"

[[group]]
name         = "compute"
display_name = "Kubernetes Nodes"
description  = "Cluster sizing configuration"

[[input]]
name         = "root_domain"
description  = "Root domain for the install"
default      = ""
sensitive    = false
display_name = "Root Domain"
type         = "string"
group        = "mattermost"

[[input]]
name         = "operator_release"
description  = "Mattermost operator Helm chart release version"
default      = "v1.22.0"
sensitive    = false
display_name = "Operator Release"
type         = "string"
group        = "mattermost"

[[input]]
name         = "app_release"
description  = "Mattermost application version"
default      = "10.2.0"
sensitive    = false
display_name = "App Release"
type         = "string"
group        = "mattermost"
```

**sandbox.toml:**
```toml
# sandbox
terraform_version = "1.11.3"

[public_repo]
directory = "."
repo      = "nuonco/aws-eks-sandbox"
branch    = "main"

[vars]
cluster_name         = "n-{{ .nuon.install.id }}"
enable_nuon_dns      = "true"
public_root_domain   = "{{ .nuon.inputs.inputs.root_domain }}"
internal_root_domain = "internal.{{ .nuon.inputs.inputs.root_domain }}"
```

**components/1-postgres.toml:**
```toml
# helm
name           = "postgres_db"
type           = "helm_chart"
chart_name     = "postgresql"
namespace      = "mattermost"
storage_driver = "configmap"

[public_repo]
repo      = "nuonco/example-app-configs"
directory = "mattermost/src/components/postgresql"
branch    = "main"
```

**components/2-secrets.toml:**
```toml
# kubernetes-manifest
name         = "mattermost_manifest_db_secret"
type         = "kubernetes_manifest"
dependencies = ["postgres_db"]
namespace    = "mattermost"

manifest = """
apiVersion: v1
kind: Secret
metadata:
  name: mm-postgres-connection
  namespace: mattermost
type: Opaque
stringData:
  DB_CONNECTION_STRING: "postgres://mmuser:mmuser_password@postgresql.mattermost.svc.cluster.local:5432/mattermost?sslmode=disable&connect_timeout=10"
"""
```

**components/3-operator.toml:**
```toml
# helm
name           = "mattermost_operator"
type           = "helm_chart"
chart_name     = "mattermost-operator"
namespace      = "mattermost-operator"
storage_driver = "configmap"
dependencies   = ["postgres_db"]

[public_repo]
repo      = "nuonco/example-app-configs"
directory = "mattermost/src/components/mattermost-operator"
branch    = "main"
```

**components/4-app.toml:**
```toml
# kubernetes-manifest
name         = "mattermost_manifest_installation"
type         = "kubernetes_manifest"
dependencies = ["postgres_db", "mattermost_manifest_db_secret", "s3_buckets"]
namespace    = "mattermost"

manifest = """
apiVersion: installation.mattermost.com/v1beta1
kind: Mattermost
metadata:
  name: mm
  namespace: mattermost
spec:
  image: mattermost/mattermost-enterprise-edition
  version: "{{ .nuon.inputs.inputs.app_release }}"
  ingress:
    enabled: false
  database:
    external:
      secret: mm-postgres-connection
"""
```

**components/5-certificate.toml:**
```toml
# terraform
name              = "certificate"
type              = "terraform_module"
terraform_version = "1.11.3"
dependencies      = ["mattermost_manifest_installation"]

[public_repo]
directory = "mattermost/src/components/certificate"
repo      = "nuonco/example-app-configs"
branch    = "main"

[vars]
public_domain = "{{ .nuon.install.sandbox.outputs.nuon_dns.public_domain.name }}"
zone_id       = "{{ .nuon.install.sandbox.outputs.nuon_dns.public_domain.zone_id }}"
```

**components/6-alb.toml:**
```toml
# helm
name           = "application_load_balancer"
type           = "helm_chart"
chart_name     = "application-load-balancer"
namespace      = "mattermost"
storage_driver = "configmap"
dependencies   = ["certificate", "mattermost_manifest_installation"]

[public_repo]
repo      = "nuonco/example-app-configs"
directory = "mattermost/src/components/application-load-balancer"
branch    = "main"

[[values_file]]
contents = "./values/alb.yaml"
```

**actions/alb-health-check.toml:**
```toml
# action
name        = "alb_health_check"
description = "Check ALB and pod health for Mattermost"
run_mode    = "day2"

script = """
#!/bin/sh
set -e
echo "=== Ingress Status ==="
kubectl get ingress -n mattermost
echo ""
echo "=== Pod Status ==="
kubectl get pods -n mattermost
echo ""
echo "=== Mattermost Service ==="
kubectl get svc -n mattermost
"""
```
