# CLI equivalents

Install: `npm install -g @seekrit/cli` (or `npx @seekrit/cli`). For containers
and CI there is `seekrit-run`, a static single binary with the same injection
behaviour and no Node dependency.

## Authenticate

```bash
seekrit login --client-id <id> --client-secret <secret>   # machine credential
seekrit login --token skt_…                               # existing service token
seekrit login                                             # browser approval, for a human
seekrit whoami
```

Credentials land in `~/.config/seekrit/config.json`. `SEEKRIT_TOKEN`,
`SEEKRIT_CLIENT_ID` / `SEEKRIT_CLIENT_SECRET`, and `SEEKRIT_API_URL` override it.

## Run something with secrets injected

```bash
seekrit run -- pnpm dev
seekrit run --env staging -- ./migrate
seekrit run --branch pr-1234 -- pnpm test
seekrit run --with platform=sandbox -- ./server   # override one group's slice
seekrit run --explain -- true                     # where each variable came from
```

`--explain` prints resolution provenance to stderr and never prints values. It
is the right way to debug "why is this variable wrong".

## Secrets

```bash
seekrit secrets list                    # names and metadata only
seekrit secrets set STRIPE_KEY          # reads the value from stdin
seekrit secrets set TLS_KEY --file key.pem   # from a file, no shell quoting
seekrit secrets get NAME                # prints the value — think first
seekrit secrets import .env             # then delete the file
seekrit secrets history NAME
seekrit secrets restore NAME 3
seekrit secrets rm NAME
```

Omit the value argument so it arrives on stdin, or use `--file` for a PEM or
JSON credential. Passing the value inline puts it in shell history.

## Structure

```bash
seekrit init                              # writes seekrit.json for this project
seekrit app create / list / show
seekrit env create / list / groups
seekrit group create / env create         # shared secret bags and their slices
seekrit branch create <slug> --from production --ttl 7d
seekrit token create / list / revoke
seekrit grant --token skt_… --app storefront --env production
seekrit audit
```

## In CI and containers

Give the job a token scoped to one environment, then wrap the command:

```bash
seekrit-run -- ./deploy.sh
```

Nothing is written to disk, and the child process is the only thing that ever
holds the values. For GitHub Actions there is a dedicated action; for Kubernetes
use the seekrit External Secrets Operator chart rather than copying values into
a `Secret` by hand.
