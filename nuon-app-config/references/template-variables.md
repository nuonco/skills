# Template Variable Reference

All variables use Go template syntax `{{ .nuon.<path> }}`.

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
