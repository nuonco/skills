# Component Templates

## Metadata

Required at the app root. No type comment. `nuon apps sync` will fail without it.

```toml
version      = "v2"
display_name = "<Human-Readable App Name>"
description  = "Short description of what this app does"
```

Optional fields: `slack_webhook_url`, `readme`.

The `readme` field accepts either an inline markdown string or a file path reference:
```toml
# inline (safe — no external file required)
readme = "# My App\nDescription here."

# file path (the file MUST exist or nuon apps sync will fail)
readme = "./README.md"
```

**Default: omit `readme` or use inline.** Do not use a file path reference unless the file is also being created.

---

## Runner

Required at the app root for **all** apps. `nuon apps sync` will fail without it.

**AWS:**
```toml
# runner
runner_type = "aws"
helm_driver = "configmap"
init_script_url = "https://raw.githubusercontent.com/nuonco/runner/refs/heads/main/scripts/aws/init-mng-v2.sh"
```

**Azure:**
```toml
# runner
runner_type = "azure"
helm_driver = "configmap"
```

Use `runner_type = "aws"` for all AWS sandboxes. Use `runner_type = "azure"` for Azure AKS.

---

## Helm Chart

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

# OR for a Helm repository (e.g. Artifact Hub, Grafana, Prometheus Community):
[helm_repo]
repo_url = "https://grafana.github.io/helm-charts"
chart    = "<chart-name>"

# Inline values:
[values]
"some.nested.key" = "{{ .nuon.install.sandbox.outputs.nuon_dns.public_domain.name }}"

# OR reference a values file:
[[values_file]]
contents = "./values/<chart-name>.yaml"
```

## Terraform Module

```toml
# terraform
name              = "<component_name>"
type              = "terraform_module"
terraform_version = "1.11.3"
dependencies      = ["<dep1>"]

# Optional: cron schedule for periodic drift detection
drift_schedule = "0 0 * * *"

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
AWS_REGION = "{{ .nuon.install_stack.outputs.region }}"

[vars]
install_id = "{{ .nuon.install.id }}"
region     = "{{ .nuon.install_stack.outputs.region }}"
vpc_id     = "{{ .nuon.install_stack.outputs.vpc_id }}"

# OR reference a var file:
[[var_file]]
contents = "./terraform/<name>.tfvars"
```

## Kubernetes Manifest

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

From a repo:

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

## Docker Build

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

## Container Image (public)

```toml
# container-image
name = "<component_name>"
type = "container_image"

[public]
image_url = "<registry>/<image>"
tag       = "latest"
```

## Container Image (ECR)

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

## Sandbox

No type comment. Uses `{{ .nuon.inputs.inputs.* }}` for input references (not `install.inputs.*`).

**AWS EKS — Nuon DNS (*.nuon.run):**
```toml
terraform_version = "1.11.3"

[public_repo]
directory = "."
repo      = "nuonco/aws-eks-sandbox"
branch    = "main"

[vars]
cluster_name          = "n-{{ .nuon.install.id }}"
enable_nuon_dns       = "true"
public_root_domain    = "{{ .nuon.install.id }}.nuon.run"
internal_root_domain  = "internal.{{ .nuon.install.id }}.nuon.run"
default_instance_type = "{{ .nuon.inputs.inputs.instance_type }}"

# Optional: reference a tfvars file for additional vars
[[var_file]]
contents = "./sandbox.tfvars"
```

**AWS EKS — Custom domain (via input):**
```toml
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

| Sandbox | Repo | Cloud |
|---|---|---|
| AWS EKS | `nuonco/aws-eks-sandbox` | AWS |
| AWS EKS + Karpenter | `nuonco/aws-eks-karpenter-sandbox` | AWS |
| AWS Minimal (no K8s) | `nuonco/aws-min-sandbox` | AWS |
| Azure AKS | `nuonco/azure-aks-sandbox` | Azure |

> **GCP**: not yet supported. Contact Nuon if interested.

### Azure Sandbox

```toml
terraform_version = "1.11.3"

[public_repo]
directory = "."
repo      = "nuonco/azure-aks-sandbox"
branch    = "main"

[vars]
public_root_domain   = "{{ .nuon.inputs.inputs.root_domain }}"
internal_root_domain = "internal.{{ .nuon.inputs.inputs.root_domain }}"
location             = "{{ .nuon.cloud_account.azure.location }}"
vnet_name            = "{{ .nuon.install_stack.outputs.network_name }}"
resource_group_name  = "{{ .nuon.install_stack.outputs.resource_group_name }}"
private_subnet_names = "{{ .nuon.install_stack.outputs.private_subnet_names }}"
public_subnet_names  = "{{ .nuon.install_stack.outputs.public_subnet_names }}"
```

---

## Inputs

**Every `[[input]]` must belong to a `[[group]]`. Groups must be declared before the inputs that reference them. Inputs without a valid `group` will fail with `invalid group ""`.**

```toml
# inputs
[[group]]
name         = "<group_name>"
display_name = "<Human-Readable Group Name>"
description  = "Description of this group of inputs"

[[input]]
name         = "root_domain"
description  = "Root domain for the application"
default      = ""
sensitive    = false
display_name = "Root Domain"
type         = "string"
group        = "<group_name>"
# required   = true   # uncomment to make this field mandatory at install time

[[input]]
name         = "api_key"
description  = "API key for external service"
default      = ""
sensitive    = true
display_name = "API Key"
type         = "string"
group        = "<group_name>"
```

Multiple groups are supported — declare each `[[group]]` then add inputs with matching `group` fields:

```toml
# inputs
[[group]]
name         = "app"
display_name = "Application"
description  = "Core application settings"

[[group]]
name         = "compute"
display_name = "Kubernetes Nodes"
description  = "Cluster sizing and instance configuration"

[[input]]
name         = "app_release"
display_name = "App Release"
description  = "Version to deploy"
default      = "1.0.0"
sensitive    = false
type         = "string"
group        = "app"

[[input]]
name         = "instance_type"
display_name = "Node Instance Size"
description  = "AWS EC2 instance type for EKS nodes"
default      = "t3a.medium"
sensitive    = false
type         = "string"
group        = "compute"
```

Input types: `string`, `number`, `bool`, `json`

## Action

```toml
# action
name    = "<action_name>"
timeout = "30s"

[[triggers]]
type = "manual"

[[steps]]
name            = "<step_name>"
inline_contents = """
#!/usr/bin/env sh
set -e
kubectl get pods -n <namespace> --no-headers
"""
```

**Trigger types:**

| Type | When it runs |
|---|---|
| `manual` | On-demand via dashboard or CLI |
| `post-provision` | After sandbox is provisioned |
| `pre-provision` | Before sandbox provisioning |
| `post-deploy-all-components` | After all components deploy |
| `pre-deploy-all-components` | Before all components deploy |
| `post-deprovision` | After teardown |
| `pre-deprovision` | Before teardown |
| `post-update-inputs` | After inputs are updated |
| `post-reprovision` | After sandbox reprovision |
| `pre-deploy-component` | Before a specific component deploys — add `component_name = "<name>"` |

Actions can have multiple `[[triggers]]` entries. Each action must have at least one `[[triggers]]` and one `[[steps]]` entry, plus a `timeout`.

---

## Permissions

Required at the app root. `nuon apps sync` will fail without a permissions config. Create a `permissions/` directory with three files:

**permissions/provision.toml:**
```toml
type         = "provision"
name         = "{{ .nuon.install.id }}-provision"
description  = "Provision the sandbox and components"
display_name = "provision role"

[[policies]]
managed_policy_name = "AdministratorAccess"
```

**permissions/deprovision.toml:**
```toml
type         = "deprovision"
name         = "{{ .nuon.install.id }}-deprovision"
description  = "Deprovision sandbox and components"
display_name = "deprovision role"

[[policies]]
managed_policy_name = "AdministratorAccess"
```

**permissions/maintenance.toml:**
```toml
type         = "maintenance"
name         = "{{ .nuon.install.id }}-maintenance"
description  = "Operate and maintain the app components"
display_name = "maintenance role"

[[policies]]
managed_policy_name = "AdministratorAccess"
```

> The `AdministratorAccess` managed policy is suitable for most apps. Scope down with inline `[[policies]]` entries (using `name` + `contents` as a JSON IAM policy document) when tighter permissions are needed. Optionally add `permissions_boundary = "./boundary.json"` for a boundary policy file.

## Install Config

> **Note**: Install config files (`install.toml`) are only supported for **AWS** installs. Azure installs are provisioned via the Nuon dashboard + Azure CLI — there is no `install.toml` for Azure.

### AWS Install Config

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
