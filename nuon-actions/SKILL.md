---
name: nuon-actions
description: Create Nuon action scripts for day-2 operations, health checks, database migrations, diagnostics, secret management, and install-time automation. Use when a user needs to add actions to an existing Nuon app config, write operational scripts that run against a customer install, or automate tasks during provisioning. Triggers on requests involving Nuon actions, day-2 ops, health checks, kubectl scripts, migrations, diagnostics, or run_mode.
license: Apache-2.0
metadata:
  author: nuonco
---

# Nuon Actions Creator

Generate production-ready Nuon action scripts. Actions are shell scripts wrapped in TOML that run in the Nuon runner context with `kubectl` access to the install cluster.

## Key Concepts

- Actions live in the `actions/` directory of your app config
- `run_mode = "install"` — runs automatically during provisioning (blocks progression, must be idempotent)
- `run_mode = "day2"` — runs on-demand after install via the Nuon dashboard or CLI
- Actions have access to `kubectl`, standard shell tools, and Nuon template variables

## Discovery Process

Ask these before generating any action:

1. **What does this action do?**
2. **When should it run?** Install-time (`run_mode = "install"`) or on-demand (`run_mode = "day2"`)?
3. **What namespace(s) does it target?**
4. **Does it need customer inputs or component outputs?**

---

## Action Template

Every action file must start with `# action`:

```toml
# action
name        = "<action_name>"
description = "What this action does"
run_mode    = "install" | "day2"

script = """
#!/bin/sh
set -e
# your script here
"""
```

---

## Common Patterns

**Health check (day-2):**
```toml
# action
name        = "health_check"
description = "Check pod and service health"
run_mode    = "day2"

script = """
#!/bin/sh
set -e
echo "=== Pods ===" && kubectl get pods -n <namespace>
echo "=== Services ===" && kubectl get svc -n <namespace>
echo "=== Ingress ===" && kubectl get ingress -n <namespace>
"""
```

**Database migration (install-time):**
```toml
# action
name        = "run_migrations"
description = "Run database migrations on first install"
run_mode    = "install"

script = """
#!/bin/sh
set -e
kubectl exec -n <namespace> deploy/<deployment> -- /app/migrate up
"""
```

**Idempotent secret creation (install-time):**
```toml
# action
name        = "create_secret"
description = "Create application secret"
run_mode    = "install"

script = """
#!/bin/sh
set -e
kubectl create secret generic <secret-name> \
  --from-literal=KEY="{{ .nuon.inputs.inputs.<input_name> }}" \
  -n <namespace> \
  --dry-run=client -o yaml | kubectl apply -f -
"""
```

**Set default storage class (install-time):**
```toml
# action
name        = "set_default_storage_class"
description = "Configure default storage class for persistent volumes"
run_mode    = "install"

script = """
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
- [ ] `run_mode` is `"install"` or `"day2"`
- [ ] Script starts with `#!/bin/sh` and `set -e`
- [ ] Install-mode actions are idempotent

---

## References

- [Nuon action docs](https://docs.nuon.co/configuration-files)
- [Using variables](https://docs.nuon.co/guides/using-variables)
