# seekrit — agent plugin

Secrets for coding agents, packaged as an
[Agent Plugin](https://agent-plugins.org). One install gives your agent both
seekrit MCP servers **and** the instructions for using them without leaking a
value into a file, a command line, or its own transcript.

```bash
npx plugins add seekritdev/agent-plugin
```

The installer detects the agents you have and translates the plugin into each
one's native format — Claude Code, Codex, Cursor, GitHub Copilot, Kiro, VS Code,
ChatGPT, Gemini CLI, and others.

## What's inside

| Component | Purpose |
| --- | --- |
| **`seekrit-secrets`** skill | Store and use encrypted secrets: `run_command` / `seekrit run` instead of `.env`, the resource model, bootstrap from nothing, how to model a tenant |
| **`seekrit-agent-keys`** skill | Give untrusted code an API key it cannot read, via the egress proxy and `{{seekrit:NAME}}` placeholders behind a default-deny allowlist |
| **`seekrit-paperclip`** skill | Handling credentials from inside [Paperclip](https://docs.paperclip.ing): what must never go in an adapter env map or an issue comment, and why a seekrit secret cannot be a Paperclip `secret_ref` |
| **`seekrit`** MCP server | Local, stdio (`npx -y @seekrit/mcp`). Everything that touches a secret value — decryption happens here, next to your key |
| **`seekrit-cloud`** MCP server | Hosted, `mcp.seekrit.dev`. Metadata and management with no install; structurally cannot decrypt |

The two servers split along seekrit's encryption boundary: the hosted one never
holds a key, a data key, or a plaintext, and refuses service tokens outright.
One machine credential drives both.

## Paperclip

[Paperclip](https://docs.paperclip.ing) installs skills straight from this
repository, pinned to a commit — paste
`https://github.com/seekritdev/agent-plugin` into the source field on its Skills
page. Its agents get MCP servers from the adapter's runtime rather than from a
plugin manifest, so wire those with `seekrit paperclip init`, or install
`@seekrit/paperclip-plugin` to get the tools, the skills, and a secrets panel in
one step. The `seekrit-paperclip` skill covers the rest.

## No account yet?

Connecting needs none. Ask your agent to call `signup` on the hosted server — it
mints an organization and a machine credential in-band, no browser and no card —
then persist it with `seekrit login --client-id … --client-secret …`.

## Notes

- The stdio server is intentionally **unpinned** (`npx -y @seekrit/mcp`) so an
  install never goes stale against the published package.
- Requires Node 20+ for the local server. The hosted server needs nothing.
- Docs: <https://seekrit.dev/docs/guides/ai-agents>

## Contributing

This repository is a **read-only mirror** published from seekrit's monorepo, so
the instructions your agent follows are auditable. Every push overwrites `main`
— don't commit here. Issues and PRs are welcome; changes land in the monorepo.

MIT licensed.
