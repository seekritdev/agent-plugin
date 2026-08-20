---
name: seekrit-secrets
description: Store and use end-to-end encrypted secrets with seekrit instead of putting values in .env files, shell commands, or source. Use whenever a task involves an API key, token, password, connection string, or .env file — storing a credential a human just handed over, running a command that needs one, wiring up an app's configuration, or when a value is missing and the temptation is to hardcode it or go hunting for it. Also covers setting seekrit up from nothing (signup, org, app, environment) and granting an environment to a service token.
license: MIT
compatibility: Needs network access to seekrit's API. The local server runs through npx (Node 20+); the hosted server needs no install.
---

# Secrets with seekrit

seekrit is an end-to-end encrypted secrets manager: the server stores only
ciphertext, and values are decrypted in *this* process, next to the credential.
That single fact produces one hard rule and one preferred verb.

## The rule

**A secret value must not come to rest anywhere it can be read again.**

- Never write a value into `.env`, a config file, a Dockerfile, a Kubernetes
  manifest, or source — even temporarily, even in a file you plan to delete.
- Never put a value in a command line (`export KEY=…`, `curl -H "Authorization:
  Bearer sk-…"`). It lands in shell history, in process listings, and in this
  transcript.
- Never print a value to show your work, and never quote one back to the user
  who pasted it.
- Never route around a missing value. If a `.env` read is denied or a file is
  absent, do **not** try `docker compose config`, `git log -p`, a lockfile, a CI
  log, a cloud console, or another secrets CLI to reconstruct it. Ask, or use the
  tools below.

If you have already broken this rule in this session — a value in a file, a
command, or your output — say so plainly and treat that credential as needing
rotation.

## The preferred verb: run, don't read

Almost every task that "needs a secret" actually needs a *process* that has the
secret. Inject it and never see it:

| Where you are | Do this |
| --- | --- |
| MCP client | `run_command { command: "pnpm", args: ["dev"] }` |
| A shell | `seekrit run -- pnpm dev` |
| Dockerfile / CI / container | `seekrit-run -- ./server` (static binary) |
| Kubernetes | the seekrit ESO chart, not a copied Secret |

`run_command` resolves the environment, injects every granted secret as an
environment variable in the child process only, and returns the exit code and
captured output. **It never returns values.** Precedence, highest first: process
env → `.env` overlay → app environment → composed groups.

Read a value with `get_secret { name, reveal: true }` only when a human
explicitly asked to see it, or when it must be pasted into a system with no
other way in. Say out loud that you are doing it, and prefer
`get_secret { name }` (which confirms existence without the value) for checks.

`export_env` writes a resolved `.env`. It exists for runtimes that genuinely
cannot be wrapped. Treat it as the last resort it is, and never as a
convenience.

## Decision table

| Situation | Tool |
| --- | --- |
| Run something that needs secrets | `run_command` / `seekrit run` |
| A human pasted a new credential | `set_secret { name, value }` — then don't echo it |
| Rotate a value | `set_secret` again; history is kept, `list_secret_versions` shows it |
| "What config does this app have?" | `list_secrets` — names and metadata, never values |
| "Is `STRIPE_KEY` set?" | `list_secrets`, or `get_secret { name }` without `reveal` |
| Wire a new app or service | `create_app` → `create_env` → `set_secret` |
| Give a deployment read access | `create_token` then `grant_env` |
| Share config across apps | groups — see [references/model.md](references/model.md) |
| Try a risky config change | `create_branch` on the environment, then `run_command { branch }` |
| Needs a *third-party API key it must not read* | the **seekrit-agent-keys** skill |

## Storing a value you were handed

```
set_secret { app: "storefront", env: "production", name: "STRIPE_SECRET_KEY", value: "<the value>" }
```

Then confirm by name only: "stored `STRIPE_SECRET_KEY` in storefront/production."
A value can reference another with `${OTHER_SECRET}`, so build connection
strings from parts rather than duplicating a password.

Importing an existing `.env` is one call — `seekrit secrets import .env` — after
which **delete the file** and tell the user you did.

## If you have no credential yet

The hosted server needs no account to connect. Call `signup` on it:

```
signup { orgName: "Acme Storefront", orgSlug: "acme-storefront" }
→ { credential: { clientId, clientSecret } }
```

Name the org after the real project or company — a human claims it by that name
later, so `test` makes it unmanageable. The `clientSecret` is shown once.
Persist it so both servers keep working:

```bash
seekrit login --client-id <id> --client-secret <secret>
```

That writes `~/.config/seekrit/config.json`, which the local server reads with
no environment plumbing. (`SEEKRIT_CLIENT_ID` / `SEEKRIT_CLIENT_SECRET` in the
environment work too, if the client passes its environment through.) Machine
credentials auto-mint an admin token, so one credential drives both servers.

## Two servers, one credential

This plugin installs both. They are split along the encryption boundary, and
the split is enforced, not conventional:

- **`seekrit`** (local, stdio) — anything touching a *value*: `set_secret`,
  `get_secret`, `run_command`, `export_env`, `create_env`, `create_token`,
  `grant_env`, KMS operations, database leases. Decryption happens here because
  this is where the key is.
- **`seekrit-cloud`** (hosted, `mcp.seekrit.dev`) — metadata and management with
  no install: orgs, apps, environments, groups, composition, members, audit,
  billing, secret *names*. It cannot decrypt anything and will refuse a service
  token outright.

Start on the local server; it can finish a secrets task end to end. Use the
hosted one when you cannot install anything, or for management and audit. If you
are on the hosted server and hit a wall, `local_tool_for { operation }` names the
local tool to use instead.

## Further reading

- [references/model.md](references/model.md) — orgs, apps, environments, groups,
  composition, branches, references, and how to model a tenant.
- [references/cli.md](references/cli.md) — the CLI equivalent of every tool
  here, for shells, CI, and containers.
