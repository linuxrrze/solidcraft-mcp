# solidcraft-mcp

Docker image serving the [SolidCraft](https://github.com/tangency-labs/solidcraft)
MCP server over HTTP, built via Gitea Actions -- see
[`.gitea/workflows/solidcraft-mcp-rebuild.yml`](../.gitea/workflows/solidcraft-mcp-rebuild.yml).
Part of the Craft MCP stack whose hub is [`storytold-mcp/`](../storytold-mcp/):
the compose sample, the runtime `.env` knobs and the codingenv client setup
are documented there.

Based on `ubuntu:26.04`, with `solidcraft` installed from the `solidcraft`
apt repo of [linuxrrze/deb-repos](https://github.com/linuxrrze/deb-repos).
`solidcraft-cli mcp --headless` is a stdio-only MCP server; `mcp-gateway`
serves it on port 8080, one `solidcraft-cli` process per MCP session:

| Endpoint | |
| --- | --- |
| `http://solidcraft-mcp:8080/servers/solidcraft/mcp` | streamable HTTP |
| `http://solidcraft-mcp:8080/status` | running sessions (JSON) |
| `http://solidcraft-mcp:8080/healthz` | liveness (healthcheck) |

The image works exactly like storytold-mcp (same `mcp-gateway`,
`mcp-servers` helper and entrypoint, identical copies; session lifetime and
limits are described there) with one served app. Several sessions run side
by side without conflicts; they share SolidCraft's recovery folder in
`$HOME/.local/share/solidcraft`. `--allow-path $WORK_DIR`
(in `app-args.conf`) lets SolidCraft read/write the shared work dir
`/home/sandbox/work` next to its default design/scratch folders.
Screenshots render on the CPU (Mesa lavapipe).

## Files

| File | Used by CI? | Purpose |
| --- | --- | --- |
| `docker/Dockerfile` | yes | Adds the solidcraft repo and installs `solidcraft` (`ARG SOLIDCRAFT_VERSION` pins it; empty/unset installs the repo's newest) plus the Mesa libs, and probes `solidcraft-cli mcp` at build time. |
| `docker/files/mcp-gateway` | yes | HTTP gateway -- identical copy of `storytold-mcp/docker/files/mcp-gateway`. |
| `docker/files/mcp-servers` | yes | Detect/render/client-config helper -- identical copy of `storytold-mcp/docker/files/mcp-servers`. |
| `docker/files/app-args.conf` | yes | `solidcraft --headless --allow-path $WORK_DIR`. |
| `docker/files/entrypoint.sh` | yes | Identical copy of storytold-mcp's: checks the work dir, renders the config, execs mcp-gateway. |
| `.solidcraft-mcp-version` | yes | `solidcraft=<version>` CI last built. Updated automatically by CI. |
| `.solidcraft-mcp-base-digest` | yes | Registry digest of the `ubuntu:26.04` base image the last build used. Updated automatically by CI. |
| `.env.sample` | yes (build) | `SOLIDCRAFT_VERSION` (kept in sync by CI) and `TZ`. Runtime settings are in the hub's `.env.sample`. No secrets. |

## Build & version tracking

Same as storytold-mcp (every 3 hours, manual dispatch, pushes touching
`solidcraft-mcp/docker/**` or the workflow), tracking the newest
`solidcraft` in the repo's `binary-amd64/Packages` (via `sort -V`) and
the base image digest. Tags: the `solidcraft` version (epoch stripped,
`+` -> `-`), `<version>-<YYYYMMDD>` (rollback) and `latest`.
