# Service-account token for the server (language-neutral)

Your server authenticates to ctl-api with a long-lived bearer token
(`NUON_API_TOKEN`). Getting this credential right is a security decision, not a
config detail.

## Use a dedicated service account — not a person's token

- Nuon accounts have types; automation should use an `AccountType=Service`
  account, distinct from a human (`Auth0`) account.
- A service-account token is not tied to an individual, survives people leaving,
  and can be scoped and rotated independently.
- **Never ship a developer's personal token** (e.g. the one in `~/.nuon` from
  `nuon auth login`). That token is fine ONLY for local, throwaway verification
  while building the integration — never for the committed or deployed server.

## How the token is issued — the `nuon` CLI (two steps)

Service accounts and their tokens are managed with the **`nuon` CLI**, against
whichever control plane the CLI is logged into (see SKILL steps 1–2). The
logged-in account must be an **org admin**. Drive the CLI with `--output agent`
to get a `{ok,data,error}` envelope you can parse. Do not use raw curl.

### Step 1 — create the service account (`org_admin`)

Creating installs requires write access, so use the **`org_admin`** role.
`nuon roles list` shows the assignable roles (`org_admin`, `org_read_only`,
`runner`), but `org_admin` is what this integration needs.

```bash
nuon service-accounts create --name customer-ui-proxy --role org_admin --output agent
# → {"ok":true,"data":{"id":"acc_...","name":"customer-ui-proxy",...}}
```

### Step 2 — mint a token for the service account

`duration` is optional (defaults to `8760h` = 1 year). Pass `--invalidate` to
revoke prior tokens when rotating.

```bash
nuon service-accounts tokens create --id <account_id> --duration 8760h --output agent
# → {"ok":true,"data":{"token":"<token>"}}
```

Take `data.token` and write it to the server's secret store as `NUON_API_TOKEN`
(see Storage below) — **never print or commit it**. Tell the vendor the service
account is granted **org-admin** (note the blast radius).

## Storage

- Put the token in the project's existing secret mechanism (env var, secrets
  manager, platform secret) as `NUON_API_TOKEN`. Keep `.env` gitignored.
- The token is read server-side only and never returned to the browser
  (see `architecture-and-security.md`).
- Plan for rotation: the token can expire or be revoked; the server should fail
  clearly (surface a mapped 502) rather than silently. Rotate by minting a new
  token (`nuon service-accounts tokens create --id <id> [--invalidate]`) and
  updating the secret.
