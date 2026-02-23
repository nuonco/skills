# Nuon Skills for Claude Code

Claude Code skills for building and managing [Nuon](https://nuon.co) BYOC (Bring Your Own Cloud) applications.

## Install

```sh
npx skills add nuonco/skills
```

Or install a single skill:

```sh
npx skills add nuonco/skills/nuon-creator
npx skills add nuonco/skills/nuon-app-config
npx skills add nuonco/skills/nuon-actions
npx skills add nuonco/skills/nuon-policy
```

---

## Skills

### `nuon-creator` — Main Navigator

The entry point for all Nuon work. Understands what you want to do and routes you to the right specialist. **Start here if you're not sure where to begin.**

> "help me with Nuon" / "I want to build a Nuon app" / general Nuon questions

---

### `nuon-app-config` — App Config Generator

Generates complete, validated Nuon app configuration files (TOML) for deploying applications into customer cloud accounts. Walks you through a structured interview, then produces a ready-to-sync directory of config files.

**Covers:** sandboxes, components (Helm, Terraform, Kubernetes, Docker), dependency ordering, customer inputs, install configs

> "create a Nuon app config" / "help me package my app for BYOC" / `nuon apps sync`

---

### `nuon-actions` — Actions Creator

Builds day-2 operation scripts for an existing Nuon app. Actions are shell scripts that run in the Nuon runner context with `kubectl` access to the customer's install cluster.

**Covers:** health checks, database migrations, secret creation, storage class config, install-time automation, on-demand operations

> "add a health check action" / "write a migration script for Nuon" / `run_mode`

---

### `nuon-policy` — Policy Creator

Defines compliance and security policies using OPA Rego. Validates container images at build time and Terraform/Helm components at deploy time — blocking or warning on violations.

**Covers:** image signing requirements, Terraform resource encryption, Helm resource limits, component-scoped or app-wide policy enforcement

> "require signed images" / "enforce encryption on my Terraform resources" / "add resource limits to my Helm chart"

---

## References

- [Nuon docs](https://docs.nuon.co)
- [Example app configs](https://github.com/nuonco/example-app-configs)
- [Nuon OSS](https://github.com/nuonco/nuon)
- [skills.sh listing](https://skills.sh/nuonco/skills)
