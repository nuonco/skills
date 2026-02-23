# Template Variable Reference

All variables use Go template syntax `{{ .nuon.<path> }}`.

## General

| Variable | Description |
|---|---|
| `{{ .nuon.install.id }}` | Unique install ID |
| `{{ .nuon.inputs.inputs.<input_name> }}` | Customer-provided input value |
| `{{ .nuon.components.<component_name>.outputs.<output_name> }}` | Cross-component output reference |

## AWS

| Variable | Description |
|---|---|
| `{{ .nuon.install.sandbox.account.id }}` | AWS account ID |
| `{{ .nuon.install.sandbox.account.region }}` | AWS region |
| `{{ .nuon.install.sandbox.outputs.cluster_name }}` | EKS cluster name |
| `{{ .nuon.install.sandbox.outputs.nuon_dns.public_domain.name }}` | Public domain (Nuon DNS) |
| `{{ .nuon.install.sandbox.outputs.nuon_dns.public_domain.zone_id }}` | Route53 zone ID |
| `{{ .nuon.install.sandbox.outputs.cluster_platform_version }}` | Cluster platform version |

## Azure

| Variable | Description |
|---|---|
| `{{ .nuon.cloud_account.azure.location }}` | Azure region/location |
| `{{ .nuon.install_stack.outputs.network_name }}` | VNet name (from install stack) |
| `{{ .nuon.install_stack.outputs.resource_group_name }}` | Resource group name |
| `{{ .nuon.install_stack.outputs.private_subnet_names }}` | Private subnet names |
| `{{ .nuon.install_stack.outputs.public_subnet_names }}` | Public subnet names |

> **GCP**: Not yet supported.
