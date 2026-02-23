---
name: nuon-app-config
description: Generate complete, working Nuon BYOC app configuration files (TOML) for deploying applications into customer cloud accounts. Use when creating a new Nuon app config, scaffolding components (Helm charts, Terraform modules, Kubernetes manifests, Docker builds, container images), defining sandboxes, inputs, dependencies, action scripts, or install configs. Triggers on requests involving Nuon, BYOC (Bring Your Own Cloud), app packaging for customer cloud deployment, or any mention of Nuon TOML config files. Also use when the user asks how to structure a Nuon app, add components, define dependencies, write action scripts, or configure installs.
license: Apache-2.0
metadata:
  author: nuonco
---

# Nuon App Config Generator

Generate complete, validated Nuon BYOC application configuration files. Every generated config must be a working TOML file that passes `nuon apps sync` validation.

## Discovery Process

Before generating any config files, gather the following through a structured interview. Ask questions in logical groups — do not dump all questions at once.

### Step 1: Application Identity

1. **What is your application?** (name, what it does, open-source or proprietary)
2. **What is the app directory name?** (lowercase, hyphens allowed, e.g. `my-app`)

### Step 2: Component Architecture

3. **How is your application deployed?**
   - **Helm Chart** — Packaged Kubernetes application
   - **Terraform Module** — Infrastructure as Code (databases, certs, networking)
   - **Kubernetes Manifest** — Raw YAML applied to the cluster
   - **Docker Build** — Build a container image from a Dockerfile
   - **Container Image** — Deploy a pre-built public or private container image

4. For each component: name (snake_case), source (public/connected/inline), and type-specific details (chart name, namespace, Terraform version, Dockerfile path, image URL, etc.)

### Step 3: Dependency Chain

5. **What is the deployment order?** Which components depend on others?
   - Example: `database → secrets → app_server → load_balancer → certificate`

### Step 4: Sandbox

6. **Cloud platform and compute environment?**
   - AWS EKS → `nuonco/aws-eks-sandbox` → `runner_type = "aws"`
   - AWS EKS + Karpenter → `nuonco/aws-eks-karpenter-sandbox` → `runner_type = "aws"`
   - AWS Minimal (no K8s) → `nuonco/aws-min-sandbox` → `runner_type = "aws"`
   - Azure AKS → `nuonco/azure-aks-sandbox` → `runner_type = "azure"`
   - GCP → **not yet supported**

   All apps require a `runner.toml`. The `runner_type` is determined by the cloud platform above.

7. **Do you need Nuon DNS?** (enables `*.nuon.run` public URLs; AWS only — not available on Azure)

### Step 5: Inputs

8. **What customer-configurable inputs does your app need?**
   - For each: name, display name, description, type (`string`/`number`/`bool`/`json`), default, sensitive

### Step 6: Action Scripts

9. **Do you need day-2 operation scripts?**
   - Health checks, database migrations, init scripts, diagnostics, cleanup

### Step 7: Helm Values & Terraform Variables

10. **Existing values files or tfvars?** Reference with relative paths.
11. **Template variables needed?** See `references/template-variables.md`.

---

## File Structure

```
<app-name>/
├── metadata.toml      # required
├── runner.toml        # required
├── inputs.toml
├── sandbox.toml
├── permissions/       # required
│   ├── provision.toml
│   ├── deprovision.toml
│   └── maintenance.toml
├── components/
│   ├── <N>-<component-name>.toml
│   └── ...
└── actions/           # optional
    └── <action-name>.toml
```

Number prefixes (`1-postgres.toml`) are visual only. Deployment order is determined by the `dependencies` array.

## Type Comment Convention

These files **must** begin with a type comment (the CLI uses them to determine file type):

```
# runner | # inputs | # helm | # terraform | # kubernetes-manifest | # docker-build
# container-image | # install | # action | # policy
```

These files do **not** use type comments: `metadata.toml`, `sandbox.toml`, `permissions/*.toml`.

---

## Validation Checklist

- [ ] `metadata.toml` exists at the app root with a `version` field
- [ ] `runner.toml` exists at the app root with the correct `runner_type` for the target cloud
- [ ] `permissions/` directory exists with `provision.toml`, `deprovision.toml`, and `maintenance.toml`
- [ ] Every action has `timeout`, at least one `[[triggers]]`, and at least one `[[steps]]`
- [ ] Every `[[input]]` in `inputs.toml` has a `group` field matching a declared `[[group]]`
- [ ] Files that require type comments have them (`# runner`, `# inputs`, `# helm`, `# terraform`, `# kubernetes-manifest`, `# action`, etc.) — `metadata.toml`, `sandbox.toml`, `permissions/*.toml` do NOT use type comments
- [ ] Every component has a unique `name` in snake_case
- [ ] All `dependencies` arrays reference valid, existing component names
- [ ] No circular dependencies
- [ ] Sandbox repo matches the selected cloud platform
- [ ] All `{{ .nuon... }}` variables reference valid paths (see `references/template-variables.md`)
- [ ] All input references match inputs defined in `inputs.toml`
- [ ] Helm/Terraform components have `[public_repo]` or `[connected_repo]` with `repo`, `directory`, `branch`
- [ ] Terraform components include `terraform_version`
- [ ] Kubernetes manifest components include `namespace`

## Common Gotchas

- **Input groups are required** — every `[[input]]` must have a `group` field referencing a declared `[[group]]`. Missing or empty group → `invalid group ""` error on sync
- **`readme` file reference** — if `metadata.toml` uses `readme = "./README.md"` (file path), the file must exist or `nuon apps sync` will fail with `unable to fetch field value`. Either omit `readme`, use an inline markdown string, or create the referenced file
- **Component name in cross-references** — use the exact `name` field value, not the filename
- **Sensitive inputs** — set `sensitive = true` for secrets
- **Storage driver** — always `storage_driver = "configmap"` for Helm unless you have a reason
- **Inline manifest template vars** — make sure referenced inputs/outputs exist by that point in the dependency chain
- **Actions are not K8s jobs** — they run in the Nuon runner with `kubectl` access, not as pods

---

## References

- Component templates → `references/component-templates.md`
- Template variable reference → `references/template-variables.md`
- Complete Mattermost example → `references/example-mattermost.md`
- [Nuon config docs](https://docs.nuon.co/configuration-files)
- [Example app configs](https://github.com/nuonco/example-app-configs)

## CLI Workflow

```sh
brew install nuonco/tap/nuon
nuon login
nuon apps create --name <app-name>
nuon apps sync
```
