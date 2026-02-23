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
- Each action has `triggers` (when it runs) and `steps` (what it does)
- `type = "manual"` trigger — on-demand via Nuon dashboard or CLI (day-2 ops)
- `type = "pre-deploy-component"` trigger — runs automatically before a component deploys (install-time)
- `timeout` is required on every action
- Actions run in the Nuon runner, not as pods in the cluster

## Discovery Process

Ask these before generating any action:

1. **What does this action do?**
2. **When should it run?** On-demand (`manual`) or automatically before a component deploys (`pre-deploy-component`)?
3. **What namespace(s) does it target?**
4. **Does it need customer inputs or component outputs?**

---

## Action Template

Every action file must start with `# action`:

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
# your script here
"""
```

---

## Trigger Types

**Manual (on-demand, day-2):**
```toml
[[triggers]]
type = "manual"
```

**Pre-deploy-component (runs before a component deploys):**
```toml
[[triggers]]
type           = "pre-deploy-component"
component_name = "<component_name>"
```

**Post-deploy-component (runs after a component deploys — common for secrets/credentials):**
```toml
[[triggers]]
type           = "post-deploy-component"
component_name = "<component_name>"
```

An action can have multiple triggers:
```toml
[[triggers]]
type           = "post-deploy-component"
component_name = "rds_cluster"

[[triggers]]
type = "manual"
```

Use `post-deploy-component` when the action depends on outputs from a completed component (e.g., copying an RDS secret into Kubernetes after the database is provisioned).

---

## Step Types

**Inline script:**
```toml
[[steps]]
name            = "<step_name>"
inline_contents = """
#!/usr/bin/env sh
set -e
kubectl get pods -n <namespace>
"""
```

**Script from a repo:**
```toml
[[steps]]
name    = "<step_name>"
command = "./script.sh"

[steps.public_repo]
repo      = "<org>/<repo>"
directory = "<path/to/scripts>"
branch    = "main"

[steps.env_vars]
MY_VAR = "{{ .nuon.install.id }}"
```

Use `[steps.connected_repo]` instead of `[steps.public_repo]` for private repos.

---

## Common Patterns

**Health check (manual/day-2):**
```toml
# action
name    = "health_check"
timeout = "30s"

[[triggers]]
type = "manual"

[[steps]]
name            = "check-pods-and-services"
inline_contents = """
#!/usr/bin/env sh
set -e
echo "=== Pods ===" && kubectl get pods -n <namespace>
echo "=== Services ===" && kubectl get svc -n <namespace>
echo "=== Ingress ===" && kubectl get ingress -n <namespace>
"""
```

**Set default storage class (pre-deploy, install-time):**
```toml
# action
name    = "default_storage_class"
timeout = "1m"

[[triggers]]
type           = "pre-deploy-component"
component_name = "postgres_db"

[[triggers]]
type = "manual"

[[steps]]
name            = "make_gp2_default_storage_class"
inline_contents = """
#!/usr/bin/env sh
set -e
kubectl patch storageclass gp2 \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
"""
```

**Copy secret after RDS/DB provisioned (post-deploy, install-time):**
```toml
# action
name    = "copy_db_secret"
timeout = "1m"

[[triggers]]
type           = "post-deploy-component"
component_name = "rds_cluster"

[[triggers]]
type = "manual"

[[steps]]
name    = "import-rds-secret"
command = "./import.sh"

[steps.public_repo]
repo      = "<org>/<repo>"
directory = "<path/to/scripts>"
branch    = "main"

[steps.env_vars]
SECRET_ARN       = "{{ .nuon.components.rds_cluster.outputs.db_instance_master_user_secret_arn }}"
REGION           = "{{ .nuon.install_stack.outputs.region }}"
TARGET_NAME      = "db-secret"
TARGET_NAMESPACE = "<namespace>"
DB_ADDRESS       = "{{ .nuon.components.rds_cluster.outputs.address }}"
DB_PORT          = "{{ .nuon.components.rds_cluster.outputs.db_instance_port }}"
DB_NAME          = "{{ .nuon.components.rds_cluster.outputs.db_instance_name }}"
```

**Multi-step action:**
```toml
# action
name    = "copy_db_secrets"
timeout = "2m"

[[triggers]]
type           = "post-deploy-component"
component_name = "rds_cluster"

[[triggers]]
type = "manual"

[[steps]]
name    = "copy-secret-to-app-namespace"
command = "./import.sh"

[steps.public_repo]
repo      = "<org>/<repo>"
directory = "<path/to/scripts>"
branch    = "main"

[steps.env_vars]
SECRET_ARN       = "{{ .nuon.components.rds_cluster.outputs.db_instance_master_user_secret_arn }}"
TARGET_NAMESPACE = "app"

[[steps]]
name    = "copy-secret-to-monitoring-namespace"
command = "./import.sh"

[steps.public_repo]
repo      = "<org>/<repo>"
directory = "<path/to/scripts>"
branch    = "main"

[steps.env_vars]
SECRET_ARN       = "{{ .nuon.components.rds_cluster.outputs.db_instance_master_user_secret_arn }}"
TARGET_NAMESPACE = "monitoring"
```

**Idempotent secret creation (pre-deploy, install-time):**
```toml
# action
name    = "create_secret"
timeout = "1m"

[[triggers]]
type           = "pre-deploy-component"
component_name = "<target_component>"

[[steps]]
name            = "create-app-secret"
inline_contents = """
#!/usr/bin/env sh
set -e
kubectl create secret generic <secret-name> \
  --from-literal=KEY="{{ .nuon.install.inputs.<input_name> }}" \
  -n <namespace> \
  --dry-run=client -o yaml | kubectl apply -f -
"""
```

**Database migration (pre-deploy, install-time):**
```toml
# action
name    = "run_migrations"
timeout = "5m"

[[triggers]]
type           = "pre-deploy-component"
component_name = "<app_component>"

[[steps]]
name            = "migrate-db"
inline_contents = """
#!/usr/bin/env sh
set -e
kubectl exec -n <namespace> deploy/<deployment> -- /app/migrate up
"""
```

---

## Template Variables

| Variable | Description |
|---|---|
| `{{ .nuon.install.id }}` | Unique install ID |
| `{{ .nuon.install_stack.outputs.region }}` | AWS region |
| `{{ .nuon.install.inputs.<name> }}` | Customer input value (use in components/actions) |
| `{{ .nuon.components.<name>.outputs.<key> }}` | Cross-component output |

---

## Rules

- Always `set -e` — fail fast
- `timeout` is required — use `"30s"`, `"1m"`, `"5m"` etc.
- Install-time actions (pre-deploy) block progression — keep them fast and idempotent
- Use `--dry-run=client -o yaml | kubectl apply -f -` for idempotent resource creation
- File naming: `<action-name>.toml` (kebab-case files, snake_case `name` field)
- Use `inline_contents` for simple scripts; use `command` + repo source for complex ones

## Validation Checklist

- [ ] File starts with `# action`
- [ ] `name` is snake_case and unique across the app
- [ ] `timeout` is present (e.g., `"30s"`, `"1m"`)
- [ ] `[[triggers]]` array is present with at least one entry
- [ ] `[[steps]]` array is present with at least one entry
- [ ] Each step has either `inline_contents` or `command` + repo source
- [ ] Install-time actions (pre-deploy) are idempotent

---

## References

- [Nuon action docs](https://docs.nuon.co/configuration-files)
- [Using variables](https://docs.nuon.co/guides/using-variables)
