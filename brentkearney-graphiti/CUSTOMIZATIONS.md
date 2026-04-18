# Graphiti Umbrel App — Customizations

This Umbrel app runs a custom-built Graphiti MCP server image:

**Image:** `ghcr.io/brentkearney/graphiti-mcp:feat-bearer-auth`

Built from the `feat/mcp-bearer-auth` branch of
[brentkearney/graphiti](https://github.com/brentkearney/graphiti), which is a
fork of [getzep/graphiti](https://github.com/getzep/graphiti) with the
following additions on top of upstream `main`:

1. **Bearer token authentication** (`BearerTokenMiddleware` in
   `graphiti_mcp_server.py`) — optional, opt-in via `MCP_API_KEY`.
2. **DNS rebinding fix** — FastMCP's default rebinding protection rejects
   non-localhost `Host` headers; `MCP_HOSTNAME` re-enables protection while
   permitting the Umbrel hostname.
3. **README docs** for the two env vars above.

The image is built from the local graphiti-core sources (not PyPI) so it
ships in-tree graphiti_core improvements that have not yet been released to
PyPI.

## Config files (mounted by docker-compose.yml)

### `config/config.yaml`
Mounted to `/app/mcp/config/config.yaml`. Graphiti MCP server configuration:
HTTP transport, Neo4j backend connection, OpenAI-compatible LLM/embedding
provider URLs. Uses env var expansion for secrets.

### `config/graphiti.env`
Mounted to `/app/mcp/config/graphiti.env` and loaded via `env_file`.
User-editable file for LLM provider settings and optional MCP auth. The
settings web UI (planned) will read and update it at runtime.

## Environment Variables

| Variable | Source | Purpose |
|----------|--------|---------|
| `NEO4J_AUTH` | `neo4j/${APP_PASSWORD}` | Neo4j built-in authentication |
| `NEO4J_URI` | `bolt://neo4j:7687` | MCP server → Neo4j connection |
| `NEO4J_USER` | `neo4j` | Neo4j username for MCP server |
| `NEO4J_PASSWORD` | `${APP_PASSWORD}` | Neo4j password for MCP server |
| `MCP_API_KEY` | `graphiti.env` (optional) | Bearer token for MCP endpoint auth |
| `MCP_HOSTNAME` | `graphiti.env` (optional) | Hostname for DNS rebinding allowlist |
| `OPENAI_API_KEY` | `graphiti.env` | LLM provider API key |
| `OPENAI_API_URL` | `graphiti.env` | LLM provider base URL |
| `OPENAI_BASE_URL` | `graphiti.env` | OpenAI SDK base URL |
| `CONFIG_PATH` | `/app/mcp/config/config.yaml` | Graphiti config file path |

`APP_PASSWORD` is provided by Umbrel and is unique per app installation.

## Security Model

Two layers of authentication, both opt-in by default:

1. **Neo4j (port 7687)** — built-in password auth via `NEO4J_AUTH`. Prevents
   direct database access from anything other than the MCP server.
2. **MCP Server (port 8000)** — optional Bearer token auth via
   `BearerTokenMiddleware`. When `MCP_API_KEY` is set in `graphiti.env`,
   clients must send `Authorization: Bearer <token>` with every request.
   When unset, the MCP endpoint is open (no auth required).

The MCP API key is intended to be configured through the app's settings web
UI (planned), which reads and writes `graphiti.env`.

### Connecting Claude Code

Without MCP auth (default):
```bash
claude mcp add graphiti http://umbrel.local:8000/mcp \
  -t http -s user
```

With MCP auth enabled:
```bash
claude mcp add graphiti http://umbrel.local:8000/mcp \
  -t http -s user \
  -H "Authorization: Bearer <MCP_API_KEY>"
```

## Upgrading the image

1. In the graphiti fork: `git rebase origin/main` the `feat/mcp-bearer-auth` branch.
2. Run the bearer-auth tests: `uv run pytest tests/test_bearer_auth.py` (from `mcp_server/`).
3. Build + push the multi-arch image: `./mcp_server/docker/build-local.sh`.
4. Capture the new manifest digest: `docker buildx imagetools inspect ghcr.io/brentkearney/graphiti-mcp:feat-bearer-auth`.
5. Update the `@sha256:...` digest in `docker-compose.yml` in this directory.
6. Commit here, then redeploy on Umbrel (UI update, or `docker compose pull && docker compose up -d`).
