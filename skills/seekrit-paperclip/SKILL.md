---
name: seekrit-paperclip
description: Handle credentials correctly while working inside Paperclip, the AI agent management platform. Use when a Paperclip run needs an API key, when you are about to put a value in an adapter env map, a project env map, an approval payload, or an issue comment, when a human hands you a credential in a Paperclip issue, or when asked to wire a Paperclip agent up to a third-party API. Also covers why a seekrit secret cannot be a Paperclip secret_ref, and which of the two paths — inject-and-exec or the egress proxy — a given run should use.
license: MIT
compatibility: Needs the seekrit CLI (or the seekrit Paperclip plugin) on the host running the agent, plus network access to seekrit's API.
---

# Secrets inside Paperclip

You are running as a Paperclip agent. Paperclip has its own secret store, and
seekrit is not one of its providers — the provider list is closed
(`local_encrypted`, `aws_secrets_manager`, `gcp_secret_manager`, `vault`). So
**a seekrit secret can never be a Paperclip `secret_ref`.** Do not try to make
one; do not ask an operator to paste a seekrit value into Paperclip's secret
store to work around it. That would create a second plaintext copy of a
credential seekrit exists to avoid copying.

Values reach a run one of two other ways, and picking between them is the whole
job.

## The two paths

| Path | Use it when | What the run sees |
| --- | --- | --- |
| **Inject and exec** — `seekrit:run_command`, or `seekrit run -- …` in a terminal | You are running a build, a test suite, a migration, a dev server — a process *you* wrote the command for | The real value, in that child process's environment only |
| **Egress proxy** — `{{seekrit:NAME}}` placeholders behind an allowlist | The thing holding the credential is a model, a sandbox, generated code, or a subagent whose output you cannot predict | Never the value — only a placeholder the proxy swaps on the way out |

Default to the first for your own tooling and the second for anything you are
about to hand to another agent or to code you just generated. When both would
work, prefer the proxy: it also bounds *where* the credential can go, which
Paperclip's execution policy does not — that policy governs review and approval
stages, not network egress.

## The rule, in Paperclip's terms

**A value must not come to rest anywhere it can be read again.** In this
environment that means, specifically, never write a credential into:

- an agent's **adapter env** map, or a project or routine **env** map — these
  persist in Paperclip's database and render in its UI. (Under
  `PAPERCLIP_SECRETS_STRICT_MODE` the server rejects it outright; assume strict
  mode even where it is off.)
- an **approval payload**, an **issue comment**, an **issue document**, or a run
  transcript. A comment is the most common accident: it feels like talking to a
  person, and it is a durable row.
- a **company export**, a workspace file, `.env`, or a commit.

A placeholder is not a value. `{{seekrit:OPENAI_API_KEY}}` is safe to put in an
adapter env map, in a config file, and in a commit — that is the point of it.

If you have already broken this rule in this run, say so plainly in your comment
on the issue and treat the credential as needing rotation. Do not quietly move
on.

## When a human hands you a credential

Someone will paste a key into an issue comment. When that happens:

1. Store it in seekrit immediately — `seekrit:store_secret`, or
   `set_secret { app, env, name, value }` on the local MCP server.
2. Reply naming the secret, never the value: "stored `STRIPE_SECRET_KEY` in
   storefront/production."
3. Tell them the comment still holds the plaintext and should be edited or
   deleted, and that the credential is worth rotating if the issue is visible
   beyond the two of you.

Never echo it back to confirm you received it.

## When a value is missing

Do not go looking. A denied read is not a puzzle to solve: reconstructing a
credential from `docker compose config`, `git log -p`, a lockfile, a CI log, or
another secrets CLI is the failure mode this skill exists to prevent, and it
looks like diligence in a transcript.

Instead, in order:

1. `seekrit:list_secrets` — confirm whether the name exists in the environment
   you are pointed at, and whether you are pointed at the right one.
2. If it genuinely is not there, ask on the issue. Name what you need and what
   it is for.
3. If the operator would rather grant than paste, point them at
   `seekrit access grant` for a service token, not at Paperclip's secret store.

Paperclip also has its own **secret proposal** flow — agents propose, people
decide, and a proposal expires after 14 days. Use it for credentials that
genuinely belong to Paperclip (a connector's own token). Do not use it to
smuggle a seekrit value into Paperclip config.

## Wiring an agent to a third-party API

The proxy path, end to end, is in
[references/paperclip-wiring.md](references/paperclip-wiring.md): the adapter
env map to paste, the `seekrit-proxy.toml` to generate, and the two lines that
make the runtime trust the proxy's CA. The short version is that the agent's
adapter env holds placeholders and an `HTTPS_PROXY`, and the real keys stay in
the proxy — so a compromised or merely creative run cannot exfiltrate them,
because it never had them.

## Tools

If the seekrit Paperclip plugin is installed, these are available in a run:

| Tool | Does |
| --- | --- |
| `seekrit:list_secrets` | Names and metadata for an environment. Never values. |
| `seekrit:run_command` | Runs a command with secrets injected into the child only; returns exit code and output. |
| `seekrit:store_secret` | Stores a value. Write-only — there is no tool that reads one back. |
| `seekrit:write_proxy_config` | Writes a `seekrit-proxy.toml` and prints the env the workload needs. |

If it is not installed, everything here is also a CLI invocation — see the
**seekrit-secrets** skill's CLI reference. If neither is available, say so on the
issue rather than falling back to a plaintext workaround.

## Further reading

- [references/paperclip-wiring.md](references/paperclip-wiring.md) — adapter env
  maps, proxy config, plugin install, and the MCP wiring per adapter.
- The **seekrit-secrets** skill — the resource model, bootstrapping from
  nothing, and the full tool surface.
- The **seekrit-agent-keys** skill — the proxy in depth, including forward mode
  and operation-level allowlists.
