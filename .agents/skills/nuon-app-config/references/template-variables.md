# Template Variable Reference

All variables use Go template syntax `{{ .nuon.<path> }}`.

## Install

| Variable | Description |
|---|---|
| `{{ .nuon.install.id }}` | Unique install ID |

## Inputs

Two valid paths depending on context:

| Variable | Context | Description |
|---|---|---|
| `{{ .nuon.inputs.inputs.<name> }}` | Sandbox vars | Input value in sandbox.toml `[vars]` |
| `{{ .nuon.install.inputs.<name> }}` | Component vars | Input value in component `[vars]` |

Both paths refer to the same customer-configured input. Use `inputs.inputs.*` in sandbox context, `install.inputs.*` in component context.

## Cross-Component Outputs

| Variable | Description |
|---|---|
| `{{ .nuon.components.<component_name>.outputs.<output_name> }}` | Output from another component |

## AWS — Sandbox Outputs

| Variable | Description |
|---|---|
| `{{ .nuon.install.sandbox.outputs.nuon_dns.public_domain.name }}` | Public domain (Nuon DNS) |
| `{{ .nuon.install.sandbox.outputs.nuon_dns.public_domain.zone_id }}` | Route53 zone ID |
| `{{ .nuon.install.sandbox.outputs.cluster_name }}` | EKS cluster name |
| `{{ .nuon.sandbox.outputs.account.region }}` | AWS region (from sandbox outputs) |
| `{{ .nuon.sandbox.outputs.vpc.private_subnet_ids }}` | List of private subnet IDs |
| `{{ index .nuon.sandbox.outputs.vpc.private_subnet_ids 0 }}` | First private subnet ID (use 0, 1, 2 for index) |

## AWS — Install Stack Outputs

These outputs come from the CloudFormation install stack (runner/VPC provisioning):

| Variable | Description |
|---|---|
| `{{ .nuon.install_stack.outputs.region }}` | AWS region |
| `{{ .nuon.install_stack.outputs.vpc_id }}` | VPC ID |
| `{{ .nuon.install_stack.outputs.resource_group_name }}` | Resource group name (Azure) |
| `{{ .nuon.install_stack.outputs.private_subnet_names }}` | Private subnet names (Azure) |
| `{{ .nuon.install_stack.outputs.public_subnet_names }}` | Public subnet names (Azure) |
| `{{ .nuon.install_stack.outputs.network_name }}` | VNet name (Azure) |

## AWS — Cloud Account

| Variable | Description |
|---|---|
| `{{ .nuon.cloud_account.aws.region }}` | AWS region of target cloud account |
| `{{ .nuon.install.sandbox.account.id }}` | AWS account ID |

## Azure

| Variable | Description |
|---|---|
| `{{ .nuon.cloud_account.azure.location }}` | Azure region/location |

## Nuon DNS

For Nuon-managed DNS (`*.nuon.run`), the sandbox vars pattern is:

```toml
public_root_domain   = "{{ .nuon.install.id }}.nuon.run"
internal_root_domain = "internal.{{ .nuon.install.id }}.nuon.run"
```

For custom domain via an input:
```toml
public_root_domain   = "{{ .nuon.inputs.inputs.root_domain }}"
internal_root_domain = "internal.{{ .nuon.inputs.inputs.root_domain }}"
```

> **GCP**: Not yet supported.
