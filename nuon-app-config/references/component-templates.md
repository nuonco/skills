# Component Templates

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

| Sandbox | Repo | Cloud |
|---|---|---|
| AWS EKS | `nuonco/aws-eks-sandbox` | AWS |
| AWS EKS + Karpenter | `nuonco/aws-eks-karpenter-sandbox` | AWS |
| AWS Minimal (no K8s) | `nuonco/aws-min-sandbox` | AWS |
| Azure AKS | `nuonco/azure-aks-sandbox` | Azure |

> **GCP**: not yet supported. Contact Nuon if interested.

### Azure Sandbox

```toml
# sandbox
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

### Azure Runner

Azure apps require a `runner.toml` alongside `sandbox.toml`:

```toml
# runner
runner_type = "azure"
helm_driver = "configmap"
```

---

## Inputs

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

## Action

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

`run_mode`: `"install"` (runs during provisioning) or `"day2"` (on-demand)

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
