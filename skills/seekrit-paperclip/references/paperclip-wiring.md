# Wiring seekrit into Paperclip

Everything here is operator work, done once per company or once per agent. If you
are an agent reading this mid-run, the useful parts are the **adapter env** table
(what a correctly wired agent's env looks like, so you can tell whether yours is)
and **Why not a secret_ref** at the bottom.

## 1. The plugin

```bash
npx paperclipai plugin install @seekrit/paperclip-plugin
```

It contributes four agent tools (`seekrit:list_secrets`, `seekrit:run_command`,
`seekrit:store_secret`, `seekrit:write_proxy_config`), installs the seekrit
skills into the company's skills library, and adds a Secrets panel under Company
Settings that lists what each environment holds — names, versions, and when they
last changed, never values.

Its one piece of configuration is a **seekrit service token**, bound as a
Paperclip `secret_ref` in the plugin's settings form. That is the correct use of
Paperclip's secret store: the token is Paperclip's own credential, scoped to the
environments you want its agents to reach, and revocable without touching
anything else.

## 2. The skills

Paperclip installs skills from a GitHub URL and pins them to a commit. On the
Skills page, paste:

```
https://github.com/seekritdev/agent-plugin
```

That picks up `seekrit-secrets`, `seekrit-agent-keys`, and `seekrit-paperclip`.
Assign them per agent, or company-wide. The plugin installs the same three, so
do this only if you are not installing the plugin.

## 3. MCP servers, per adapter

Paperclip is the control plane; the **adapter's runtime** is what speaks MCP. So
MCP config is per-runtime and lives next to the agent's working directory, not in
Paperclip's database.

For the Claude Code adapter (and the Codex, Cursor, Gemini CLI, and OpenCode
adapters, which inherit their CLI's own MCP support the same way), the file is a
project-scoped `.mcp.json` in the agent's working directory:

```bash
seekrit paperclip init --dir /srv/paperclip/workspaces/storefront
```

That writes both seekrit servers into `.mcp.json`, merging with any servers
already there:

```json
{
  "mcpServers": {
    "seekrit": { "type": "stdio", "command": "npx", "args": ["-y", "@seekrit/mcp"] },
    "seekrit-cloud": { "type": "streamable-http", "url": "https://mcp.seekrit.dev/mcp" }
  }
}
```

Project scope is the right one here: an agent's working directory already pins
it, and two agents pointed at the same repo should see the same servers. For a
server one agent alone should see, give that agent its own working directory.

## 4. The proxy, for runs you do not trust

`seekrit paperclip init --proxy --preset anthropic --preset openai --preset gemini`
writes a `seekrit-proxy.toml` and prints the adapter env to paste. Run the proxy
beside Paperclip — same host, or a sidecar container — with a service token in
`SEEKRIT_TOKEN`.

Forward mode is usually right for Paperclip, because the runtimes are CLIs you
are not configuring base URLs for:

```toml
listen = "127.0.0.1:8080"

[forward]
listen = "127.0.0.1:8081"
ca_cert = "seekrit-proxy-ca.pem"

[[rule]]
host = "api.anthropic.com"
allow = ["ANTHROPIC_API_KEY"]
methods = ["GET", "POST"]
paths = ["/v1/**"]
```

### Adapter env

Paste this into the agent's **Environment variables** on its Configuration tab.
Every value is a plain string — a placeholder is not a secret, so none of these
needs a `secret_ref` and none trips strict mode:

| Variable | Value |
| --- | --- |
| `ANTHROPIC_API_KEY` | `{{seekrit:ANTHROPIC_API_KEY}}` |
| `OPENAI_API_KEY` | `{{seekrit:OPENAI_API_KEY}}` |
| `GEMINI_API_KEY` | `{{seekrit:GEMINI_API_KEY}}` |
| `HTTPS_PROXY` | `http://127.0.0.1:8081` |
| `NODE_EXTRA_CA_CERTS` | absolute path to `seekrit-proxy-ca.pem` |

Use `SSL_CERT_FILE` or `REQUESTS_CA_BUNDLE` instead of `NODE_EXTRA_CA_CERTS` for
a non-Node runtime; a Python adapter needs the latter.

A run wired this way holds no key. The proxy refuses any host not in the file,
and the file is not something the run can edit.

### Checking it worked

From the agent's working directory, ask the runtime to make one real call. A
`401` from the upstream means the placeholder went out unsubstituted — the
usual cause is a secret name in the env that is not in the rule's `allow` list.
A connection error to the upstream means `HTTPS_PROXY` is not reaching the
proxy. A TLS error means the CA env var is wrong for that runtime.

## 5. Injecting instead, for runs you do trust

Where the run is your own tooling, skip the proxy and inject:

```
seekrit:run_command { command: "pnpm", args: ["test"] }
```

The tool resolves the environment, injects every granted secret into the child
process only, and returns the exit code and captured output. It does not return
values, and there is no tool that does.

In a terminal on the same host, the same thing is `seekrit run -- pnpm test`.

## Why not a `secret_ref`

Paperclip resolves a `secret_ref` through its own provider system, and that
provider list is closed: `local_encrypted`, `aws_secrets_manager`,
`gcp_secret_manager`, `vault`. There is no seekrit provider to select, so there
is no honest way to express a seekrit secret as a Paperclip secret.

The dishonest way — copying a value out of seekrit and into Paperclip's store —
is worse than it looks. It creates a second plaintext copy under a different
lifecycle, so a rotation in seekrit silently leaves Paperclip serving the old
value, and a seekrit revocation does not revoke anything. Both paths above avoid
that by keeping resolution at the moment of use.

The one thing that *should* be a Paperclip `secret_ref` is the plugin's own
seekrit service token, in step 1.
