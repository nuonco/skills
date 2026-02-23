---
name: nuon-policy
description: Create Nuon deployment policies, approval workflows, and install governance configs. Use when a user needs to control how their app gets deployed — approval gates, automatic vs manual promotion, install constraints, or runner policies. Triggers on requests involving Nuon policies, approval flows, deployment gates, install approval, promotion rules, or governance.
license: Apache-2.0
metadata:
  author: nuonco
---

# Nuon Policy Creator

Generate Nuon policy and governance configuration. Policies control how installs are created, promoted, and approved — giving you guardrails over deployments into customer cloud accounts.

## Key Concepts

- **Approval options** — control whether installs require human sign-off before proceeding
- **Install constraints** — enforce rules on customer-provided inputs before an install can proceed
- **Multi-environment configs** — separate staging (auto) and production (prompt) install configs

## Discovery Process

1. **Who can trigger new installs?**
2. **Should installs require manual approval before provisioning?** (`auto` vs `prompt`)
3. **Should upgrades also require approval?**
4. **Any input validation rules?** (e.g. domain patterns, version formats, allowed regions)
5. **Any inputs that should be locked after first install?**

---

## Install Config Template

Lives at `<app-name>/installs/<install-name>.toml`.

```toml
# install
name            = "<install-name>"
approval_option = "auto" | "prompt"

[aws_account]
region = "us-east-1"

[[inputs]]
root_domain = "example.com"

[[inputs]]
app_release = "1.0.0"
```

| `approval_option` | Behavior |
|---|---|
| `"auto"` | Install starts immediately after creation |
| `"prompt"` | Install waits for human approval before provisioning |

---

## Multi-Environment Setup

**installs/staging.toml:**
```toml
# install
name            = "staging"
approval_option = "auto"

[aws_account]
region = "us-west-2"

[[inputs]]
root_domain = "staging.example.com"
app_release = "latest"
```

**installs/production.toml:**
```toml
# install
name            = "production"
approval_option = "prompt"

[aws_account]
region = "us-east-1"

[[inputs]]
root_domain = "example.com"
app_release = "1.0.0"
```

---

## Input Constraints

Enforce guardrails via the `description` field in `inputs.toml`:

```toml
# inputs
[[input]]
name         = "app_release"
description  = "Semantic version (e.g. 1.2.3). Must match a published release tag."
default      = "1.0.0"
sensitive    = false
display_name = "App Release"
type         = "string"
```

---

## Validation Checklist

- [ ] Install files start with `# install`
- [ ] `approval_option` is intentional — `"auto"` for staging, `"prompt"` for production
- [ ] All `[[inputs]]` keys match names defined in `inputs.toml`
- [ ] Sensitive inputs are NOT hardcoded in install files

---

## References

- [Nuon install config docs](https://docs.nuon.co/configuration-files)
- [Nuon dashboard](https://app.nuon.co)
