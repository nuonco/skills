---
name: nuon-actions
description: Create Nuon action scripts for day-2 operations, health checks, database migrations, diagnostics, secret management, and install-time automation. Use when a user needs to add actions to an existing Nuon app config, write operational scripts that run against a customer install, or automate tasks during provisioning. Triggers on requests involving Nuon actions, day-2 ops, health checks, kubectl scripts, migrations, diagnostics, or triggers/steps.
license: Apache-2.0
metadata:
  author: nuonco
---

# Nuon Actions Creator

Generate production-ready Nuon action scripts. Actions are shell scripts wrapped in TOML that run in the Nuon runner context with `kubectl` access to the install cluster.

## Key Concepts

- Actions live in the `actions/` directory of your app config
- When an action runs is controlled by its `[[triggers]]` — not a `run_mode` field (`run_mode` does not exist)
- Use `type = "manual"` for on-demand day-2 operations
- Use `type = "post-provision"` to run automatically after the sandbox is provisioned
- Actions have access to `kubectl`, standard shell tools, and Nuon template variables

## Discovery Process

Ask these before generating any action:

1. **What does this action do?**
2. **When should it run?** On-demand (`manual`), after provisioning (`post-provision`), after deploy (`post-deploy-all-components`), etc.
3. **What namespace(s) does it target?**
4. **Does it need customer inputs or component outputs?**

---

## Action Template

Every action file must start with `# action`. Required fields: `name`, `timeout`, `triggers`, `steps`.

- `[[triggers]]` is a TOML array-of-tables — **not** `triggers = []`
- Inline scripts use `inline_contents`, not `script`

```toml
# action
name        = "<action_name>"
description = "What this action does"
timeout     = "5m"

[[triggers]]
type = "manual"

[[steps]]
name            = "<step_name>"
inline_contents = """
#!/bin/sh
set -e
# your script here
"""
```

## Trigger Types

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

Multiple triggers are supported — just add more `[[triggers]]` blocks.

---

## Common Patterns

**Health check (manual day-2):**
```toml
# action
name        = "health_check"
description = "Check pod and service health"
timeout     = "5m"

[[triggers]]
type = "manual"

[[steps]]
name            = "check_health"
inline_contents = """
#!/bin/sh
set -e
echo "=== Pods ===" && kubectl get pods -n <namespace>
echo "=== Services ===" && kubectl get svc -n <namespace>
echo "=== Ingress ===" && kubectl get ingress -n <namespace>
"""
```

**Database migration (runs after every deploy):**
```toml
# action
name        = "run_migrations"
description = "Run database migrations"
timeout     = "10m"

[[triggers]]
type = "post-deploy-all-components"

[[triggers]]
type = "manual"

[[steps]]
name            = "migrate"
inline_contents = """
#!/bin/sh
set -e
kubectl exec -n <namespace> deploy/<deployment> -- /app/migrate up
"""
```

**Idempotent secret creation (post-provision):**
```toml
# action
name        = "create_secret"
description = "Create application secret"
timeout     = "5m"

[[triggers]]
type = "post-provision"

[[triggers]]
type = "manual"

[[steps]]
name            = "create_secret"
inline_contents = """
#!/bin/sh
set -e
kubectl create secret generic <secret-name> \
  --from-literal=KEY="{{ .nuon.inputs.inputs.<input_name> }}" \
  -n <namespace> \
  --dry-run=client -o yaml | kubectl apply -f -
"""
```

**Set default storage class (post-provision):**
```toml
# action
name        = "set_default_storage_class"
description = "Configure default storage class for persistent volumes"
timeout     = "5m"

[[triggers]]
type = "post-provision"

[[steps]]
name            = "patch_storage_class"
inline_contents = """
#!/bin/sh
set -e
kubectl patch storageclass gp2 \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
"""
```

---

## Template Variables

| Variable | Description |
|---|---|
| `{{ .nuon.install.id }}` | Unique install ID |
| `{{ .nuon.install.sandbox.account.region }}` | Cloud region |
| `{{ .nuon.inputs.inputs.<name> }}` | Customer input value |
| `{{ .nuon.components.<name>.outputs.<key> }}` | Cross-component output |

---

## Rules

- Always `set -e` — fail fast
- Install-time actions block provisioning — keep them fast and idempotent
- Actions run in the Nuon runner, not as pods in the cluster
- Use `--dry-run=client -o yaml | kubectl apply -f -` for idempotent resource creation
- File naming: `<action-name>.toml` (kebab-case files, snake_case `name` field)

## Validation Checklist

- [ ] File starts with `# action`
- [ ] `name` is snake_case and unique across the app
- [ ] `timeout` is set (e.g. `"5m"`, max `"30m"`)
- [ ] At least one `[[triggers]]` block with a valid `type` value
- [ ] At least one `[[steps]]` block with a `name` field
- [ ] Step script uses `inline_contents` key — **not** `script`
- [ ] Script starts with `#!/bin/sh` and `set -e`

---

## References

- [Nuon action docs](https://docs.nuon.co/configuration-files)
- [Using variables](https://docs.nuon.co/guides/using-variables)
