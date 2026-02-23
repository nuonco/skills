---
name: nuon-app-config
description: Generate complete, working Nuon BYOC app configuration files (TOML) for deploying applications into customer cloud accounts. Use when creating a new Nuon app config, scaffolding components (Helm charts, Terraform modules, Kubernetes manifests, Docker builds, container images), defining sandboxes, inputs, dependencies, action scripts, or install configs. Triggers on requests involving Nuon, BYOC (Bring Your Own Cloud), app packaging for customer cloud deployment, or any mention of Nuon TOML config files. Also use when the user asks how to structure a Nuon app, add components, define dependencies, write action scripts, or configure installs.
---

# Nuon App Config Generator

Generate complete, validated Nuon BYOC application configuration files. Every generated config must be a working TOML file that passes `nuon apps sync` validation.

## Discovery Process

Before generating any config files, gather the following from the user through a structured interview. Ask questions in logical groups — do not dump all questions at once.

### Step 1: Application Identity

1. **What is your application?** (name, what it does, open-source or proprietary)
2. **What is the app directory name?** (lowercase, hyphens allowed, e.g. `my-app`)

### Step 2: Component Architecture

3. **How is your application deployed?** Allow multiple selections:
   - **Helm Chart** — Packaged Kubernetes application (most common for K8s apps)
   - **Terraform Module** — Infrastructure as Code for cloud resources (databases, certs, networking)
   - **Kubernetes Manifest** — Raw YAML applied directly to the cluster (secrets, configmaps, CRDs)
   - **Docker Build** — Build a container image from a Dockerfile in a repo
   - **Container Image** — Deploy a pre-built public or private container image

4. **For each component, ask:**
   - Component name (snake_case, e.g. `postgres_db`, `application_load_balancer`)
   - Source: public repo, connected (private) repo, or inline?
   - For Helm: chart name, namespace, storage driver preference
   - For Terraform: terraform version, variables needed
   - For K8s Manifest: namespace, inline or from repo?
   - For Docker Build: Dockerfile path, repo details
   - For Container Image: public image URL + tag, or ECR details

### Step 3: Dependency Chain

5. **What is the deployment order?** Which components depend on others?
   - Example: `database → secrets → app_server → load_balancer → certificate`
   - Dependencies are defined by component name in a `dependencies` array

### Step 4: Sandbox Selection

6. **What cloud platform and compute environment?**
   - AWS EKS (most common) → `nuonco/aws-eks-sandbox`
   - AWS EKS + Karpenter → `nuonco/aws-eks-karpenter-sandbox`
   - AWS Minimal (no K8s) → `nuonco/aws-min-sandbox`
   - Azure AKS → `nuonco/terraform-azure-aks-sandbox`
   - Custom sandbox from own repo

7. **Do you need Nuon DNS?** (enables `*.nuon.run` public URLs)

### Step 5: Inputs

8. **What customer-configurable inputs does your app need?**
   - Examples: `root_domain`, `api_key`, `vpc_id`, release versions, feature flags
   - For each: name, display name, description, type (`string`/`number`/`bool`/`json`), default, sensitive

### Step 6: Action Scripts

9. **Do you need action scripts for day-2 operations?**
   - Health checks (e.g. ALB health, pod status)
   - Database migrations or seed scripts
   - Init scripts (e.g. set default storage class)
   - Diagnostic scripts (e.g. kubectl logs, list pods)
   - Cleanup or teardown scripts

### Step 7: Helm Values & Terraform Variables

If using Helm or Terraform components:

10. **Do you have existing values files or tfvars?** Reference them with relative paths.
11. **What template variables do you need?** (see Template Variable Reference below)

---

## File Generation Rules

### Directory Structure

```
<app-name>/
├── inputs.toml
├── sandbox.toml
├── components/
│   ├── <N>-<component-name>.toml    # number prefix for visual ordering
│   └── ...
├── actions/                          # optional
│   ├── <action-name>.toml
│   └── ...
└── README.md
```

Numbering components (e.g. `1-postgres.toml`, `2-secrets.toml`) is a visual convention. Actual deployment order is determined by the `dependencies` array.

### Type Comment Requirement

Every TOML file **must** begin with a type comment on the first line. This is required for Nuon CLI/LSP validation:

```toml
# helm
# terraform
# kubernetes-manifest
# docker-build
# container-image
# inputs
# sandbox
# install
# action
```

---

## Component Templates

### Helm Chart

```toml
# helm
name           = "<component_name>"
type           = "helm_chart"
chart_name     = "<chart-name>"
namespace      = "<namespace>"
storage_driver = "configmap"
dependencies   = ["<dep1>", "<dep2>"]

[public_repo]
repo      = "<org>/<repo>"
directory = "<path/to/chart>"
branch    = "main"

# OR for private repos:
[connected_repo]
repo      = "<org>/<repo>"
directory = "<path/to/chart>"
branch    = "main"

# Inline values:
[values]
"some.nested.key" = "{{ .nuon.install.sandbox.outputs.nuon_dns.public_domain.name }}"

# OR reference a values file:
[[values_file]]
contents = "./values/<chart-name>.yaml"
```

### Terraform Module

```toml
# terraform
name              = "<component_name>"
type              = "terraform_module"
terraform_version = "1.11.3"
dependencies      = ["<dep1>"]

[connected_repo]
directory = "<path/to/module>"
repo      = "<org>/<repo>"
branch    = "main"

# OR public:
[public_repo]
directory = "<path/to/module>"
repo      = "<org>/<repo>"
branch    = "main"

[env_vars]
AWS_REGION = "{{ .nuon.install.sandbox.account.region }}"

[vars]
account_id = "{{ .nuon.install.sandbox.account.id }}"
install_id = "{{ .nuon.install.id }}"

# OR reference a var file:
[[var_file]]
contents = "./terraform/<name>.tfvars"
```

### Kubernetes Manifest

```toml
# kubernetes-manifest
name         = "<component_name>"
type         = "kubernetes_manifest"
dependencies = ["<dep1>"]
namespace    = "<namespace>"

manifest = """
apiVersion: v1
kind: Secret
metadata:
  name: <secret-name>
  namespace: <namespace>
type: Opaque
stringData:
  KEY: "{{ .nuon.inputs.inputs.some_input }}"
"""
```

Manifests can also reference a repo:

```toml
# kubernetes-manifest
name         = "<component_name>"
type         = "kubernetes_manifest"
dependencies = ["<dep1>"]
namespace    = "<namespace>"

[public_repo]
repo      = "<org>/<repo>"
directory = "<path/to/manifests>"
branch    = "main"
```

### Docker Build

```toml
# docker-build
name       = "<component_name>"
type       = "docker_build"
dockerfile = "Dockerfile"

[connected_repo]
directory = "<path/to/context>"
repo      = "<org>/<repo>"
branch    = "main"
```

### Container Image (public)

```toml
# container-image
name = "<component_name>"
type = "container_image"

[public]
image_url = "<registry>/<image>"
tag       = "latest"
```

### Container Image (ECR)

```toml
# container-image
name = "<component_name>"
type = "container_image"

[aws_ecr]
iam_role_arn = "<role_arn>"
image_url    = "<ecr-url>"
tag          = "latest"
region       = "us-west-2"
```

---

## Sandbox Configuration

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

Available managed sandboxes:
| Sandbox | Repo |
|---|---|
| AWS EKS | `nuonco/aws-eks-sandbox` |
| AWS EKS + Karpenter | `nuonco/aws-eks-karpenter-sandbox` |
| AWS Minimal (no K8s) | `nuonco/aws-min-sandbox` |
| Azure AKS | `nuonco/terraform-azure-aks-sandbox` |

---

## Inputs Configuration

```toml
# inputs
[[input]]
name         = "root_domain"
description  = "Root domain for the application"
default      = ""
sensitive    = false
display_name = "Root Domain"
type         = "string"

[[input]]
name         = "api_key"
description  = "API key for external service"
default      = ""
sensitive    = true
display_name = "API Key"
type         = "string"
```

Input types: `string`, `number`, `bool`, `json`

---

## Action Scripts

Actions live in the `actions/` directory. They are shell scripts wrapped in TOML.

`run_mode`:
- `"install"` — runs automatically during provisioning
- `"day2"` — runs on-demand after install

```toml
# action
name        = "<action_name>"
description = "What this action does"
run_mode    = "day2"

script = """
#!/bin/sh
set -e
kubectl get pods -n <namespace> --no-headers
"""
```

### Common Action Patterns

**Set Default Storage Class (install-time):**
```toml
# action
name        = "set_default_storage_class"
description = "Configure the default storage class for persistent volumes"
run_mode    = "install"

script = """
#!/bin/sh
set -e
kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
"""
```

**ALB Health Check (day-2):**
```toml
# action
name        = "alb_health_check"
description = "Check ALB and pod health"
run_mode    = "day2"

script = """
#!/bin/sh
set -e
echo "=== Ingress Status ==="
kubectl get ingress -A
echo "=== Pod Status ==="
kubectl get pods -n <namespace>
echo "=== Services ==="
kubectl get svc -n <namespace>
"""
```

**Database Migration (install-time):**
```toml
# action
name        = "init_db"
description = "Run database migrations on first install"
run_mode    = "install"

script = """
#!/bin/sh
set -e
kubectl exec -n <namespace> deploy/<deployment> -- /app/migrate up
"""
```

**Idempotent Secret Creation (install-time):**
```toml
# action
name        = "create_app_secret"
description = "Create app connection secret"
run_mode    = "install"

script = """
#!/bin/sh
set -e
kubectl create secret generic <secret-name> \
  --from-literal=CONNECTION_STRING="<value>" \
  -n <namespace> \
  --dry-run=client -o yaml | kubectl apply -f -
"""
```

---

## Install Configuration (optional)

Use for defining named installs with pre-set input values — useful for testing or demo environments.

```toml
# install
name            = "<install-name>"
approval_option = "prompt"

[aws_account]
region = "us-east-1"

[[inputs]]
root_domain = "example.com"

[[inputs]]
app_release = "9.5.1"
```

---

## Template Variable Reference

All variables use Go template syntax `{{ .nuon.<path> }}`:

| Variable | Description |
|---|---|
| `{{ .nuon.install.id }}` | Unique install ID |
| `{{ .nuon.install.sandbox.account.id }}` | Cloud account ID |
| `{{ .nuon.install.sandbox.account.region }}` | Cloud region |
| `{{ .nuon.install.sandbox.outputs.cluster_name }}` | EKS cluster name |
| `{{ .nuon.install.sandbox.outputs.nuon_dns.public_domain.name }}` | Public domain (Nuon DNS) |
| `{{ .nuon.install.sandbox.outputs.nuon_dns.public_domain.zone_id }}` | Route53 zone ID |
| `{{ .nuon.install.sandbox.outputs.cluster_platform_version }}` | Cluster platform version |
| `{{ .nuon.inputs.inputs.<input_name> }}` | Customer-provided input value |
| `{{ .nuon.components.<component_name>.outputs.<output_name> }}` | Cross-component output reference |

---

## Validation Checklist

Before presenting the final config, verify:

- [ ] Every `.toml` file starts with the correct type comment (`# helm`, `# terraform`, `# action`, etc.)
- [ ] Every component has a unique `name` in snake_case
- [ ] All `dependencies` arrays reference valid, existing component names
- [ ] No circular dependencies exist
- [ ] Sandbox repo matches the selected cloud platform
- [ ] All `{{ .nuon... }}` template variables reference valid paths
- [ ] All input references in components match inputs defined in `inputs.toml`
- [ ] Helm/Terraform components have either `[public_repo]` or `[connected_repo]` with `repo`, `directory`, `branch`
- [ ] Terraform components include `terraform_version`
- [ ] Kubernetes manifest components include `namespace`
- [ ] File naming follows `<N>-<name>.toml` convention for components

---

## Common Gotchas

- **Component name slugs in cross-references** — when referencing `{{ .nuon.components.<name> }}`, use the exact `name` field value from the component TOML, not the filename
- **Sensitive inputs** — set `sensitive = true` for secrets; they will be masked in the Nuon dashboard and logs
- **Storage driver** — always use `storage_driver = "configmap"` for Helm components unless you have a specific reason to use `secret`
- **Inline manifests with template vars** — template vars inside `manifest = """..."""` blocks are evaluated at deploy time; make sure the referenced inputs/outputs exist by that point in the dependency chain
- **Action scripts are not K8s jobs** — they run in the Nuon runner context with `kubectl` access to the install cluster; do not use them as application init containers
- **`run_mode = "install"` actions** — these block install progression, so keep them fast and idempotent

---

## Complete Working Example: Mattermost

Reference implementation from `nuonco/example-app-configs/mattermost`:

**inputs.toml:**
```toml
# inputs
[[input]]
name         = "root_domain"
description  = "Root domain for the install"
default      = ""
sensitive    = false
display_name = "Root Domain"
type         = "string"

[[input]]
name         = "operator_release"
description  = "Mattermost operator Helm chart release version"
default      = "v1.22.0"
sensitive    = false
display_name = "Operator Release"
type         = "string"

[[input]]
name         = "app_release"
description  = "Mattermost application version"
default      = "10.2.0"
sensitive    = false
display_name = "App Release"
type         = "string"
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

---

## CLI Workflow

```sh
# Install Nuon CLI
brew install nuonco/tap/nuon

# Login
nuon login

# Create the app (name must match directory)
nuon apps create --name <app-name>

# Sync config files to Nuon
nuon apps sync

# Verify in dashboard
# Visit https://app.nuon.co
```

---

## References

- Nuon docs: https://docs.nuon.co/configuration-files
- Example app configs: https://github.com/nuonco/example-app-configs
- Using variables: https://docs.nuon.co/guides/using-variables
- Component dependencies: https://docs.nuon.co/guides/component-dependencies
- Nuon OSS repo: https://github.com/nuonco/nuon
- Managed sandboxes: https://docs.nuon.co/concepts/sandboxes
