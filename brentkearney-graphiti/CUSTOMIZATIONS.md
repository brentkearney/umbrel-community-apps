# Graphiti Umbrel App — Customizations

This document tracks modifications made to the upstream Graphiti Docker image
(`zepai/knowledge-graph-mcp:standalone`) for the Umbrel deployment.

## Patched Files

Both patches are applied via Docker volume mounts in `docker-compose.yml`,
overriding files inside the upstream container image.

### 1. `config/fastmcp-server.py`

**Mounted to:** `/app/mcp/.venv/lib/python3.11/site-packages/mcp/server/fastmcp/server.py`

**Based on:** FastMCP `server.py` from MCP SDK v1.26.0 (bundled in the upstream image)

**Changes:**

- **Bearer token authentication** — Added `BearerTokenMiddleware`, a minimal ASGI
  middleware that checks the `Authorization: Bearer <token>` header against the
  `MCP_API_KEY` environment variable. When `MCP_API_KEY` is set, all HTTP requests
  to the MCP endpoint require a valid Bearer token. Returns 401 Unauthorized if
  the token is missing or incorrect. This protects the MCP API from unauthorized
  access on the network. Activated in `streamable_http_app()` when the env var is
  present; no-op when unset.

### 2. `config/config.yaml`

**Mounted to:** `/app/mcp/config/config.yaml`

Graphiti MCP server configuration specifying HTTP transport, Neo4j backend
connection settings, and OpenAI-compatible LLM/embedding provider URLs.
Uses environment variable expansion for secrets (`NEO4J_PASSWORD`, etc.).

## Environment Variables

| Variable | Source | Purpose |
|----------|--------|---------|
| `NEO4J_AUTH` | `neo4j/${APP_PASSWORD}` | Neo4j built-in authentication |
| `NEO4J_URI` | `bolt://neo4j:7687` | MCP server → Neo4j connection |
| `NEO4J_USER` | `neo4j` | Neo4j username for MCP server |
| `NEO4J_PASSWORD` | `${APP_PASSWORD}` | Neo4j password for MCP server |
| `MCP_API_KEY` | `${APP_PASSWORD}` | Bearer token for MCP endpoint auth |
| `CONFIG_PATH` | `/app/mcp/config/config.yaml` | Graphiti config file path |

`APP_PASSWORD` is provided by Umbrel and is unique per app installation.

## Security Model

Two layers of authentication:

1. **Neo4j (port 7687)** — Built-in password auth via `NEO4J_AUTH`. Prevents
   direct database access from anything other than the MCP server.

2. **MCP Server (port 8000)** — Bearer token auth via `BearerTokenMiddleware`.
   Clients must send `Authorization: Bearer <MCP_API_KEY>` with every request.

### Connecting Claude Code

```bash
claude mcp add graphiti http://umbrel.local:8000/mcp \
  -t http -s user \
  -H "Authorization: Bearer <APP_PASSWORD>"
```

## Upstream Tracking

When upgrading the upstream Docker image (`zepai/knowledge-graph-mcp:standalone`),
the patched `fastmcp-server.py` must be rebased onto the new version of
`mcp/server/fastmcp/server.py` bundled in the image. Check for changes to
`streamable_http_app()` in particular.
