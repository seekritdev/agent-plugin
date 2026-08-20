# Proxy configuration reference

The token comes from the environment (`SEEKRIT_TOKEN`) — **never** from this
file. The container reads its config from `/seekrit-proxy.toml`; override with
`--config`. Bind to loopback unless the proxy is reached across a network
namespace, in which case `--listen 0.0.0.0:8080` and let the network boundary do
the work.

## Reverse proxy — one route per upstream

```toml
listen = "127.0.0.1:8080"

# Request bodies are buffered to substitute placeholders, so they are capped.
# max_request_body_bytes = 2097152   # 2 MiB default

[[route]]
prefix = "/openai"
upstream = "https://api.openai.com"
allow = ["OPENAI_API_KEY"]
methods = ["POST"]
paths = ["/v1/chat/completions", "/v1/embeddings"]
label = "chat + embeddings"          # shown in refusal logs

[[route]]
prefix = "/anthropic"
upstream = "https://api.anthropic.com"
allow = ["ANTHROPIC_API_KEY"]
methods = ["POST"]
paths = ["/v1/messages"]

[[route]]
prefix = "/stripe"
upstream = "https://api.stripe.com"
allow = ["STRIPE_SECRET_KEY"]
methods = ["GET", "POST"]
paths = ["/v1/customers", "/v1/customers/*", "/v1/charges"]
```

The workload calls `http://127.0.0.1:8080/openai/v1/chat/completions` with
`Authorization: Bearer {{seekrit:OPENAI_API_KEY}}`. Paths are matched *after*
the prefix is stripped.

## Forward proxy — for code you cannot modify

```toml
[forward]
listen = "127.0.0.1:8081"

# Hosts with no rule:
#   "tunnel" (default) — passed through untouched, no interception
#   "deny"             — refused, so only ruled hosts are reachable at all
unmatched_host_policy = "deny"

# Generated and persisted on first run if absent.
ca_cert = "seekrit-proxy-ca.pem"
ca_key  = "seekrit-proxy-ca-key.pem"

[[forward.host]]
match   = "api.openai.com"           # bare hostname: no scheme, port, or path
allow   = ["OPENAI_API_KEY"]
methods = ["POST"]
paths   = ["/v1/**"]

# Several rules may name one host; first match wins, so narrow goes above broad.
[[forward.host]]
match   = "api.openai.com"
methods = ["GET"]                    # reads allowed, but carry no credential
```

The workload needs `HTTPS_PROXY=http://127.0.0.1:8081` and the CA in its trust
store — `NODE_EXTRA_CA_CERTS`, `SSL_CERT_FILE`, `REQUESTS_CA_BUNDLE`, or the
system store, depending on the runtime.

## Rules from the dashboard

```toml
[policy]
source = "server"
agent = "nova"                       # this deployment's agent identity
refresh_interval = "10s"

# The trust anchor — the one thing that must not come from the server. Copy it
# from the dashboard's trust-anchor panel and commit it. Pin a second admin too,
# or one lost passphrase means nobody can publish.
signers = ["kNc8…thumbprint"]

# Optional fleet ceiling: server policy may only narrow this. Inappropriate for
# interactive development, where the local agent can edit local files anyway.
[[policy.ceiling]]
host  = "api.openai.com"
allow = ["OPENAI_API_KEY"]
```

In server mode, `allow` / `methods` / `paths` in the file are **rejected**
rather than ignored, and secrets are re-resolved on the same interval so a new
rule and the credential it names arrive together.

## Session tickets (one proxy, several agents)

```toml
[control]
listen  = "127.0.0.1:9090"
ttl     = "1h"
max_ttl = "12h"
```

Requires `SEEKRIT_PROXY_CONTROL_TOKEN` in the environment, and that token must
not be readable by the agents. The orchestrator `POST`s `/session` to mint a
ticket; the agent presents it as `x-seekrit-ticket`.

## Picking up newly added secrets

```toml
[secrets]
refresh_interval = "30s"
```

Without this (in file-policy mode) the proxy resolves once at startup, so a
secret added later never reaches a healthy running proxy. Server-policy mode
turns refresh on implicitly. No new grant is involved — a new secret in an
already-granted environment decrypts with the key the proxy already holds.

## Telemetry

Traces, metrics, and logs go to **your** OTLP collector, configured with the
standard `OTEL_*` variables, and are inert unless
`OTEL_EXPORTER_OTLP_ENDPOINT` is set. Secret names, hosts, durations, status
codes, and refusal reasons are recorded; values, bodies, and `Authorization`
headers never are. `propagate_trace_upstream = true` additionally sends W3C
trace headers to the upstream — off by default, since most upstreams are third
parties.
