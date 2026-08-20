# The resource model

```
organization
└── application            (a codebase / service)
    └── environment        (production, staging, dev …)
        └── secret         (name → encrypted value, versioned)
groups                     (org-scoped reusable secret bags, also per-environment)
```

An application environment resolves to **its own secrets plus the groups it
composes**. Values layer, lowest precedence first:

```
group secrets  <  app-env secrets  <  .env overlay  <  process env
```

After merging, `${OTHER_SECRET}` references inside values are expanded — so a
`DATABASE_URL` can be assembled from a group's host and the app's password
without duplicating either.

## Groups

Use a group when two or more apps need the same value (a shared database, an
observability key, an internal CA). A group has its own environments keyed by
slug; at resolve time each composed group is matched to the environment whose
slug equals the app environment's, so composing `platform` into
`storefront/production` pulls `platform/production`.

- `create_group` / `create_group_env` / `compose_group` / `uncompose_group`
- `list_env_groups` shows what an environment pulls in; `list_group_envs` shows
  a group's slices.

Prefer a group over copying a value into five apps. Copies drift and rotate
badly.

## Branches

A branch is an ephemeral fork of an environment: it inherits everything and
overrides only what differs, then deletes itself. Use one for a preview deploy,
a pull request, or a risky config experiment — never a real environment named
`pr-1234`.

- `create_branch { app, from: "production", slug: "pr-142" }`, then `run_command { branch }`
- `delete_branch` when done; `list_branches` to find strays.

## Service tokens and grants

A token is a principal with a keypair, not a password. Creating one does not
grant it anything; `grant_env` wraps that environment's data key to the token's
public key, which is what lets it decrypt.

- `create_token` (add `admin: true` only for a management credential)
- `grant_env { token, app, env }` — one grant per environment the token reads
- `revoke_token` immediately; `list_tokens` to audit

A deployment should hold a token scoped to exactly one environment. An admin
token belongs on a developer machine or a management job, never in a running
service.

## Temporary access (leases)

For a human or agent that needs a database *now* and should not keep it, mint a
lease instead of handing over a stored credential: `create_pg_lease` /
`create_mysql_lease` provision a scoped, expiring database user through a
customer-hosted provisioner, and `revoke_pg_lease` / `revoke_mysql_lease` end it
early. `list_pg_targets` / `list_mysql_targets` show what can be leased.

Prefer a lease over `get_secret reveal:true` on a database password. It expires
on its own, and the audit trail says who had it and when.

## Modelling a multi-tenant agent

If one agent serves many customers and each needs its own credentials, pick by
how the tenants differ:

| Shape | Model it as | Why |
| --- | --- | --- |
| Tenants share the code, differ in credentials | one **environment per tenant** under one app | independent grants, independent audit, independent rotation |
| Most config is shared, a few keys differ | one group composed by every tenant environment | rotate the shared parts once |
| Access should expire after a task | a **lease** | nothing to revoke later |
| Tenant credential must never be readable by the agent | the **seekrit-agent-keys** skill (proxy) | the agent holds a placeholder, not a key |

Resolve per run, not per process: fetch for the tenant you are serving now and
let it go afterwards. Do not build a process-wide map of every tenant's secrets
— that turns one compromised request into a full breach.
