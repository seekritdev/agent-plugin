---
name: seekrit-agent-keys
description: Let an agent, sandbox, or untrusted process call an API without ever being able to read the credential, using seekrit's egress proxy and {{seekrit:NAME}} placeholders behind a default-deny allowlist. Use when running model-generated or untrusted code that needs a third-party key (LLM providers, Stripe, GitHub, internal APIs), when a sandbox or container needs credentials, when a key must be usable for one operation but not others, or when asked to limit what an agent can do with a key rather than only where the key is stored.
license: MIT
compatibility: Needs Docker (or the seekrit-proxy binary) and network access. The proxy runs beside the workload, not inside seekrit.
---

# Keys an agent cannot read

Injecting a secret as an environment variable (see the **seekrit-secrets**
skill) is right when you trust the process. It is the wrong tool when the
process is a sandbox, model-generated code, or an agent whose output you cannot
predict: anything in the environment can be printed, logged, or POSTed
somewhere else.

The seekrit **egress proxy** removes the key from the workload entirely. The
workload sends a placeholder; the proxy — which holds the token — swaps in the
real value on the way out, and only toward destinations you allowlisted.

```
agent ──"Authorization: Bearer {{seekrit:OPENAI_API_KEY}}"──▶ proxy ──real key──▶ api.openai.com
                                                               │
                                                               └─ anything not allowlisted: refused
```

A compromised or merely creative agent cannot exfiltrate the key, because it
never had it, and cannot redirect it, because the destination is fixed by
config the agent does not control.

## Start it

The proxy is a single static binary on `scratch`; the container is the fastest
path. Write a config, then run it beside the workload:

```toml
# seekrit-proxy.toml
listen = "127.0.0.1:8080"

[[route]]
prefix = "/openai"
upstream = "https://api.openai.com"
allow = ["OPENAI_API_KEY"]
methods = ["POST"]
paths = ["/v1/chat/completions", "/v1/embeddings"]
```

```bash
docker run --rm -e SEEKRIT_TOKEN=skt_… \
  -v "$PWD/seekrit-proxy.toml:/seekrit-proxy.toml" \
  -p 8080:8080 seekritdev/proxy --listen 0.0.0.0:8080
```

Then point the workload's client at the proxy and give it a placeholder instead
of a key:

```ts
const openai = new OpenAI({
  baseURL: "http://127.0.0.1:8080/openai/v1",
  apiKey: "{{seekrit:OPENAI_API_KEY}}",
});
```

Any OpenAI-compatible endpoint works the same way — LiteLLM, OpenRouter, vLLM,
a self-hosted gateway. So does Anthropic, Stripe, or an internal service; only
the `upstream` and header change.

## The allowlist is the security boundary

Both modes are **default-deny**. A rule grants three things, and each one is
worth setting deliberately:

- `allow` — which secrets may be substituted toward this upstream. A key
  allowlisted for `api.openai.com` cannot leak toward `api.attacker.com`.
- `methods` — which HTTP methods. Omitted means any.
- `paths` — which paths, matched after the prefix is stripped (`*` within a
  segment, `**` across them). Omitted means any.

`methods` and `paths` are what turn the proxy from anti-theft into anti-misuse:
an agent with a legitimate Stripe key still cannot reach `DELETE /v1/customers`
if you did not grant it.

## Two modes

**Reverse proxy** (`[[route]]`, above) — the workload points its base URL at the
proxy. No certificates, works with any HTTP client, but every client needs its
base URL changed.

**Forward proxy** (`[forward]`) — the workload sets
`HTTPS_PROXY=http://127.0.0.1:8081` and trusts the proxy's CA. The proxy
intercepts TLS for ruled hosts, substitutes, and re-originates. Transparent to
code you cannot modify, which makes it the right mode for a sandbox running
arbitrary generated code. Hosts with no rule are tunnelled untouched by default;
set `unmatched_host_policy = "deny"` for a hard egress boundary.

## Sandboxes: never bake a key into the image

For E2B, Modal, Daytona, Vercel Sandbox, Cloudflare Sandbox/Containers, or a
plain container: **do not** resolve secrets and pass them as sandbox environment
variables. Run the proxy outside the sandbox, give the sandbox only the proxy
address and placeholders, and set `HTTPS_PROXY` plus the CA in its environment.
The sandbox can then use the credential and still holds nothing worth stealing —
and its egress is bounded by the same allowlist.

## Many agents, one proxy

When one proxy fronts several agents that should not have equal reach, the
trusted orchestrator mints a **session ticket** (`POST /session` on the
`[control]` listener, naming an agent identity and optionally a narrower secret
set) and hands the agent the opaque ticket, which it presents as
`x-seekrit-ticket`. Scopes can only narrow: the effective set is the ticket's
intersected with the published policy. The control token must be unreadable by
the agents, or any of them could mint a ticket for any identity.

## Policy from the dashboard, not the file

`[policy] source = "server"` takes the rules from agent access policy in the
seekrit dashboard, so adding an upstream is a UI change instead of a redeploy.
The bundle is signed in the publishing admin's browser and this proxy refuses
any bundle not signed by a thumbprint pinned in its own local file — seekrit can
withhold policy (the proxy fails closed) but cannot widen it. Keep the `signers`
list in the file, pin at least two admins, and commit it.

## When not to use this

- The process is trusted and you control it → inject env vars instead
  (**seekrit-secrets**). Simpler, no proxy to run.
- The credential is needed by a library that signs requests locally (AWS SigV4,
  some database drivers) → substitution happens on the wire, so a locally
  computed signature over a placeholder will not verify. Use a scoped token or a
  database lease instead.
- You want per-request model routing, cost tracking, or fallbacks → that is a
  model gateway's job. Put the gateway behind the proxy and keep its key in
  seekrit.

## Further reading

- [references/proxy-config.md](references/proxy-config.md) — annotated config
  for both modes, refresh intervals, and the fleet ceiling.
