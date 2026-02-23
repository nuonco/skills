---
name: nuon-policy
description: Create Nuon compliance and security policies using OPA Rego. Use when a user needs to validate container images, Terraform plans, or Helm charts before builds or deployments. Triggers on requests involving Nuon policies, OPA, Rego, image signing, resource limits, encryption requirements, policy violations, or build/deploy governance.
license: Apache-2.0
metadata:
  author: nuonco
---

# Nuon Policy Creator

Generate Nuon compliance and security policies using OPA Rego. Policies validate components at build time (container images) or deploy time (Terraform, Helm) and can block or warn on violations.

## Key Concepts

- **Policy types** — `container_image` (build-time), `terraform_module` (deploy-time), `helm_chart` (deploy-time)
- **Policy engine** — `opa` (Open Policy Agent using Rego)
- **`deny` rules** — block the build or deployment step
- **`warn` rules** — log a warning but allow the step to continue
- **`components`** — list of component names to apply the policy to, or `["*"]` for all

## Discovery Process

1. **What component types are in scope?** (container images, Terraform modules, Helm charts)
2. **What should be enforced vs warned?** (hard blocks vs advisory)
3. **Which specific components, or should it apply to all?**
4. **What are the compliance requirements?** (image signing, encryption, resource limits, tags, etc.)

---

## Directory Structure

```
myapp/
├── policies/
│   ├── require-signed-images.rego
│   ├── require-encryption.rego
│   └── require-resource-limits.rego
├── policies.toml
├── components/
│   ├── api_image.toml
│   ├── database.toml
│   └── api.toml
└── metadata.toml
```

---

## policies.toml

Register all policies at the app root:

```toml
# Container image policy — evaluated at build time
[[policy]]
type       = "container_image"
engine     = "opa"
components = ["api_image"]
contents   = "./policies/require-signed-images.rego"

# Terraform policy — evaluated at deploy time
[[policy]]
type       = "terraform_module"
engine     = "opa"
components = ["*"]
contents   = "./policies/require-encryption.rego"

# Helm chart policy — evaluated at deploy time
[[policy]]
type       = "helm_chart"
engine     = "opa"
components = ["api"]
contents   = "./policies/require-resource-limits.rego"
```

| Field        | Description                                                     |
|--------------|-----------------------------------------------------------------|
| `type`       | `container_image`, `terraform_module`, or `helm_chart`          |
| `engine`     | `opa` for Rego policies                                         |
| `components` | Component names to apply to, or `["*"]` for all                 |
| `contents`   | Path to policy file (relative to app root)                      |

---

## Container Image Policy (Build-time)

Evaluates `input.metadata` fields on the image. Use `input.image` and `input.tag` for identity.

```rego
# policies/require-signed-images.rego
package nuon

default allow := false

allow if {
    input.metadata.signed == true
}

deny contains msg if {
    not input.metadata.signed
    msg := sprintf("Image %s:%s must be signed", [input.image, input.tag])
}
```

---

## Terraform Policy (Deploy-time)

Evaluates `input.plan.resource_changes`. Each entry has `.type`, `.address`, `.change.actions`, and `.change.after`.

```rego
# policies/require-encryption.rego
package nuon

# Deny unencrypted S3 buckets
deny contains msg if {
    some resource in input.plan.resource_changes
    resource.type == "aws_s3_bucket"
    resource.change.actions[_] in ["create", "update"]
    not resource.change.after.server_side_encryption_configuration
    msg := sprintf("S3 bucket '%s' must have encryption enabled", [resource.address])
}

# Warn about missing Environment tag
warn contains msg if {
    some resource in input.plan.resource_changes
    resource.change.actions[_] in ["create", "update"]
    not resource.change.after.tags.Environment
    msg := sprintf("Resource '%s' is missing Environment tag", [resource.address])
}
```

---

## Helm Chart Policy (Deploy-time)

Evaluates `input.review.kind.kind` and `input.review.object.spec.containers`.

```rego
# policies/require-resource-limits.rego
package nuon

# Deny containers without CPU limits
deny contains msg if {
    input.review.kind.kind == "Pod"
    some container in input.review.object.spec.containers
    not container.resources.limits.cpu
    msg := sprintf("Container '%s' must have CPU limits defined", [container.name])
}

# Deny containers without memory limits
deny contains msg if {
    input.review.kind.kind == "Pod"
    some container in input.review.object.spec.containers
    not container.resources.limits.memory
    msg := sprintf("Container '%s' must have memory limits defined", [container.name])
}

# Warn about missing resource requests
warn contains msg if {
    input.review.kind.kind == "Pod"
    some container in input.review.object.spec.containers
    not container.resources.requests
    msg := sprintf("Container '%s' should have resource requests defined", [container.name])
}
```

---

## Syncing and Observing Results

After creating policy files and `policies.toml`, sync the app:

```sh
nuon apps sync
```

**Build-time** (container image): policies are checked when running `nuon builds create -c <component>`. A failed policy produces `policy_failed` build status.

**Deploy-time** (Terraform, Helm): policies are evaluated per workflow step during `nuon installs deploy`. View results:

```sh
nuon installs workflows steps list -w <workflow-id>
nuon installs workflows steps get -w <workflow-id> -s <step-id>
```

Step output legend:
- `✗` — deny violation, step blocked
- `⚠` — warning, step continued
- `✓` — all policies passed

---

## Validation Checklist

- [ ] All `.rego` files use `package nuon`
- [ ] `deny` rules use `deny contains msg if { ... }` pattern
- [ ] `warn` rules use `warn contains msg if { ... }` pattern
- [ ] `policies.toml` `type` matches the component type
- [ ] `contents` paths are correct relative to app root
- [ ] Component names in `components` match names in `components/` directory

---

## References

- [Configuring Policies guide](https://docs.nuon.co/guides/configuring-policies)
- [External Image Policies](https://docs.nuon.co/guides/external-image-policies)
- [Policy Configuration Reference](https://docs.nuon.co/config-ref/policy)
- [Example Policies Repository](https://github.com/nuonco/policies)
- [OPA Rego language docs](https://www.openpolicyagent.org/docs/latest/policy-language/)
