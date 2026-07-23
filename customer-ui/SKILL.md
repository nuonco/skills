---
name: customer-ui
description: Integrate your application with a Nuon BYOC control plane so your customers can create and manage Nuon installs directly from your own UI. Adds a server-side proxy (any language/framework) that holds your Nuon API token and forwards requests to the ctl-api, then wires up frontend components that call the new proxy endpoints. Use when you want your app or your customers to create installs, drive installs, or otherwise talk to a Nuon BYOC control plane from your product. Triggers on "integrate my app with Nuon", "let my customers create installs from my UI", "proxy the ctl-api", "add a create install button".
license: Apache-2.0
metadata:
  author: nuonco
---

# Nuon BYOC Integration

Build an integration that lets your customers create and manage Nuon
installs from your own product, without ever exposing Nuon credentials
to the browser.

## The architecture (always)

```
┌────────────┐   /api/... (vendor)   ┌─────────────────┐   /v1/... (ctl-api)   ┌──────────┐
│ Vendor UI  │ ────────────────────▶ │  Vendor server  │ ────────────────────▶ │  Nuon    │
│ (customer) │                       │  (proxy)        │  Bearer + X-Nuon-Org  │  ctl-api │
└────────────┘                       └─────────────────┘                       └──────────┘
```

Three tiers, three properties that MUST hold:

1. The Nuon API token and org ID live **only** on your server. Never in
   the browser bundle, never returned to the client.
2. The proxy **authorizes the customer** (your own tenant model) before
   forwarding, and owns the sensitive fields (`app_id`, cloud account block).
3. The customer supplies only a **whitelisted** subset of the ctl-api payload
   (`name` + declared inputs).

Read `references/architecture-and-security.md` before generating any code — it
is the contract the whole integration is built against and is fully
language-neutral.

## This skill is language- and framework-agnostic

Do NOT assume Node, TypeScript, Express, or React. The `references/` directory
is pure protocol (HTTP, auth, payloads, lifecycle, security) and applies to any
stack. The `adapters/` directory holds idioms for specific stacks and is
additive — Node/Express and React ship as the reference implementations, but you
can and should generate a correct integration for an unknown stack from the
references + the repo's own conventions, marking anything uncertain with
explicit `TODO` comments.

## Procedure

Follow these steps in order. Track them with tasks if the integration is large.

Everything that talks to Nuon goes through the **`nuon` CLI**, not raw curl.
Drive it with `--output agent` for a stable `{ok,data,error}` JSON envelope, and
`--read-only` for any read you don't intend to be a write.

### 1. Check the `nuon` CLI is installed

Confirm `nuon` is on PATH (`nuon version`). If it is not, stop and tell the
vendor to install it (`brew install nuonco/tap/nuon`, or see
https://docs.nuon.co) — the whole flow depends on it. Do not fall back to curl.

### 2. Check authentication; log in if needed

Check whether the vendor is authenticated — e.g. `nuon orgs list --output agent`
(an `unauthorized` error code, or a "session has expired" banner, means not
logged in). If not authenticated, have them run **`nuon auth login`** (interactive
browser flow — the vendor completes it; you cannot). Whichever control plane the
CLI is logged into is the one this integration targets (BYOC — it is not always
`api.nuon.co`); its `api_url` in `~/.nuon` becomes `NUON_API_URL`.

### 3. Confirm the org and app

- **Org:** show the current org (`nuon orgs get`) and confirm it with the vendor;
  if wrong, `nuon orgs list` then `nuon orgs select`. Capture the org id
  (`nuon orgs id`) as `NUON_ORG_ID`.
- **App:** `nuon apps list`, confirm the target app with the vendor, and
  `nuon apps select` (or capture its id) as `NUON_APP_ID`. Note its
  **cloud platform** (aws / azure / gcp) — it determines the account block in the
  create payload.

### 4. Create a service account and token

Create a dedicated **service account** for the server and mint its token via the
CLI. Use the **`org_admin`** role (creating installs requires write access;
`nuon roles list` shows the alternatives, but `org_admin` is what this integration
needs). See `references/service-account-token.md`.

```bash
# 1) Create the service account (org_admin):
nuon service-accounts create --name customer-ui-proxy --role org_admin --output agent
#   → data.id

# 2) Mint a long-lived token (duration defaults to 8760h = 1 year):
nuon service-accounts tokens create --id <account_id> --duration 8760h --output agent
#   → data.token
```

Write the returned token into the project's secret mechanism as `NUON_API_TOKEN`
(keep `.env` gitignored) — **do not print it** or commit it. Tell the vendor the
service account is **org-admin** (note the blast radius). Rotate later with
`nuon service-accounts tokens create --id <id> --invalidate`.

### 5. Inspect the vendor's app and detect the stack

Now understand the app you are integrating into. Inspect the repo to determine:

- **Language**: `package.json` (Node), `go.mod` (Go), `pyproject.toml` /
  `requirements.txt` (Python), `Gemfile` (Ruby), `pom.xml` / `build.gradle`
  (Java/Kotlin), `composer.json` (PHP), etc.
- **Server framework**: e.g. Express/Fastify/Nest/Next (Node); Gin/Echo/net-http
  (Go); FastAPI/Flask/Django (Python); Rails/Sinatra (Ruby).
- **Frontend framework**: React/Vue/Svelte/Angular, or none.
- **Conventions**: how routes are declared, how config/secrets are read, what
  validation library exists, what HTTP client the frontend already uses, TS vs
  JS. Also decide where the customer→app authorization hooks into the app's
  existing auth/session model.

If an `adapters/<lang>-<framework>.md` exists for the detected stack, load it.
Otherwise generate from `references/` + observed conventions.

### 6. Fetch the install-input schema

Install form fields are rendered **dynamically from the app's input schema**, not
hardcoded. Retrieve the schema (see `references/contracts/install-inputs-schema.md`)
with `nuon apps input-config --output agent`, or plan the proxy's schema endpoint
to fetch it live. Use it to drive both the generated form and the server-side
input whitelist.

### 7. Generate the server proxy

Per `references/contracts/install-create.md` and the relevant adapter:

- A small Nuon HTTP client that injects `Authorization: Bearer <token>` and
  `X-Nuon-Org-ID: <org>` and normalizes ctl-api errors.
- `POST /api/installs` — validate body, authorize caller, whitelist inputs,
  inject server-owned `app_id` + cloud account, call ctl-api, map errors.
- `GET /api/install-inputs` — proxy the app input schema so the UI can render
  fields (keeps the token server-side).
- `GET /api/installs/:id` — proxy install status for the async lifecycle.
- A tenant-authorization step, clearly marked where the human must enforce their
  own tenant→app mapping. Never generate a pass-through-everything proxy.

### 8. Generate the frontend

Per the frontend adapter:

- A typed API layer calling your proxy (not ctl-api directly).
- A create-install form whose fields are rendered dynamically from the input
  schema (types, required, defaults, sensitive→masked).
- Create action with pending/disabled state, then success → navigate to a status
  view that polls `GET /api/installs/:id` (creation is async).
- Error surfacing from the mapped proxy errors.

### 9. Verify

Build/typecheck the project. Optionally dry-run the proxy against Nuon using the
`nuon` CLI `--read-only` guardrail or a test org before wiring the write path.

### 10. Report

List generated files, the security TODOs the human must complete (tenant auth,
secret storage), and how to extend to more operations (deploy, teardown) by
adding contract files.

## Extending

- **New language/framework**: copy `adapters/_template.md`, fill in the idioms.
  References are reused unchanged.
- **New operation** (deploy, teardown, status detail): add a file under
  `references/contracts/` following `install-create.md`, then generate against it.
