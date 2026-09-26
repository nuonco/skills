# Nuon Skills for Claude Code

Claude Code skills for building and managing [Nuon](https://nuon.co) BYOC (Bring Your Own Cloud) applications.

## Install

```sh
npx skills add nuonco/skills
```

Or install a single skill:

```sh
npx skills add nuonco/skills/write-rego-policy
npx skills add nuonco/skills/write-readme
```

---

## Skills

### `write-rego-policy` — Policy Writer

Writes or updates an OPA/Rego policy for a Nuon app's `policies/` directory — guardrails on terraform, helm, pulumi, kubernetes, or sandbox components.

> "add a policy that blocks public S3 buckets" / "write a rego policy for this component" / "guardrail this component"

---

### `write-readme` — README Writer

Writes or updates the README for a Nuon app or runbook (the `readme` field in `metadata.toml` / `README.md`).

> "write a README for this app" / "update the runbook README"

---

## References

- [Nuon docs](https://docs.nuon.co)
- [Example app configs](https://github.com/nuonco/example-app-configs)
- [Nuon OSS](https://github.com/nuonco/nuon)
- [skills.sh listing](https://skills.sh/nuonco/skills)
