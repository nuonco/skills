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

## How the token is issued — the public API (three steps)

Service accounts and their tokens are managed on the **public API** — the same
base URL (`NUON_API_URL`) and auth (`Authorization: Bearer` + `X-Nuon-Org-ID`)
the proxy itself uses. No admin API, no `X-Nuon-Admin-Email`, no separate base
URL. The caller must be an **org admin**.

**Bootstrap:** you mint the durable service-account token *once* using your own
org-admin API token (get it from the Nuon dashboard, or `nuon orgs api-token`).
That personal token is used only for these setup calls — it is never what the
server ships with.

**The skill does not run these for the vendor** — it displays the directions and
ready-to-paste curl commands, lists the roles, and lets the vendor pick. The
token must not pass through the skill; the vendor runs the calls and stores the
result themselves.

Assume these are set: `NUON_API_URL`, `ORG` (org id), and `TOKEN` (your personal
org-admin token, for setup only).

### Step 1 — list assignable roles and choose one

`GET /v1/roles` returns the roles that can be assigned to a service account.

```bash
curl -sS "$NUON_API_URL/v1/roles" \
  -H "Authorization: Bearer $TOKEN" -H "X-Nuon-Org-ID: $ORG"
# → [ { "role_type": "org_admin",     "title": "Admin",     "applies_to": ["user","service_account"] },
#     { "role_type": "org_read_only", "title": "Read-only", "applies_to": ["user","service_account"] },
#     { "role_type": "runner",        "title": "Runner",    "applies_to": ["service_account"] } ]
```

Have the vendor choose. **Creating installs requires write access**, so today
`org_admin` is the role that works for this integration — `org_read_only` cannot
create, and `runner` is for runners. Note the blast radius of whatever is chosen,
and re-check `GET /v1/roles` over time as finer-grained roles are added.

### Step 2 — create the service account

`POST /v1/service-accounts` with a `name` and the chosen `role`. Returns the
account object (including its `id`).

```bash
curl -sS -X POST "$NUON_API_URL/v1/service-accounts" \
  -H "Authorization: Bearer $TOKEN" -H "X-Nuon-Org-ID: $ORG" \
  -H "Content-Type: application/json" \
  -d '{"name":"customer-ui-proxy","role":"org_admin"}'
# → { "id": "acc_...", "name": "customer-ui-proxy", ... }
```

### Step 3 — mint a token for the service account

`POST /v1/service-accounts/{account_id}/tokens`. `duration` is optional
(defaults to `8760h` = 1 year). Pass `"invalidate": true` to revoke prior tokens
when rotating.

```bash
curl -sS -X POST "$NUON_API_URL/v1/service-accounts/<account_id>/tokens" \
  -H "Authorization: Bearer $TOKEN" -H "X-Nuon-Org-ID: $ORG" \
  -H "Content-Type: application/json" \
  -d '{"duration":"8760h"}'
# → { "token": "<token>" }
```

The returned `token` is what the server uses as `NUON_API_TOKEN` (see Storage
below). The dashboard equivalent (Org settings → Service accounts) also works.

## Storage

- Put the token in the project's existing secret mechanism (env var, secrets
  manager, platform secret) as `NUON_API_TOKEN`. Keep `.env` gitignored.
- The token is read server-side only and never returned to the browser
  (see `architecture-and-security.md`).
- Plan for rotation: the token can expire or be revoked; the server should fail
  clearly (surface a mapped 502) rather than silently. Rotate by minting a new
  token (`POST /v1/service-accounts/{id}/tokens`, optionally `"invalidate": true`
  to revoke the old one) and updating the secret.
