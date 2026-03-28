# Graphiti Umbrel App — Customizations

This document tracks modifications made to the upstream Graphiti Docker image
(`zepai/knowledge-graph-mcp:standalone`) for the Umbrel deployment.

## Patched Files

Patches are applied via Docker volume mounts in `docker-compose.yml`,
overriding files inside the upstream container image.

### 1. `config/fastmcp-server.py`

**Mounted to:** `/app/mcp/.venv/lib/python3.11/site-packages/mcp/server/fastmcp/server.py`

**Based on:** FastMCP `server.py` from MCP SDK v1.26.0 (bundled in the upstream image)

**Changes:**

- **Bearer token authentication** — Added `BearerTokenMiddleware`, a minimal ASGI
  middleware that checks the `Authorization: Bearer <token>` header against the
  `MCP_API_KEY` environment variable. Activated in `streamable_http_app()` when
  `MCP_API_KEY` is set; no-op when unset (open access, the default). Returns 401
  Unauthorized if the token is missing or incorrect.

### 2. `config/config.yaml`

**Mounted to:** `/app/mcp/config/config.yaml`

Graphiti MCP server configuration specifying HTTP transport, Neo4j backend
connection settings, and OpenAI-compatible LLM/embedding provider URLs.
Uses environment variable expansion for secrets (`NEO4J_PASSWORD`, etc.).

### 3. `config/graphiti.env`

**Mounted to:** `/app/mcp/config/graphiti.env`

User-editable environment file for LLM provider settings and optional MCP
authentication. Loaded as `env_file` by Docker Compose and also mounted as a
volume so the settings web UI can read and update it at runtime.

## Environment Variables

| Variable | Source | Purpose |
|----------|--------|---------|
| `NEO4J_AUTH` | `neo4j/${APP_PASSWORD}` | Neo4j built-in authentication |
| `NEO4J_URI` | `bolt://neo4j:7687` | MCP server → Neo4j connection |
| `NEO4J_USER` | `neo4j` | Neo4j username for MCP server |
| `NEO4J_PASSWORD` | `${APP_PASSWORD}` | Neo4j password for MCP server |
| `MCP_API_KEY` | `graphiti.env` (optional) | Bearer token for MCP endpoint auth |
| `OPENAI_API_KEY` | `graphiti.env` | LLM provider API key |
| `OPENAI_API_URL` | `graphiti.env` | LLM provider base URL |
| `OPENAI_BASE_URL` | `graphiti.env` | OpenAI SDK base URL |
| `CONFIG_PATH` | `/app/mcp/config/config.yaml` | Graphiti config file path |

`APP_PASSWORD` is provided by Umbrel and is unique per app installation.

## Security Model

Two layers of authentication, both opt-in by default:

1. **Neo4j (port 7687)** — Built-in password auth via `NEO4J_AUTH`. Prevents
   direct database access from anything other than the MCP server.

2. **MCP Server (port 8000)** — Optional Bearer token auth via
   `BearerTokenMiddleware`. When `MCP_API_KEY` is set in `graphiti.env`, clients
   must send `Authorization: Bearer <token>` with every request. When unset, the
   MCP endpoint is open (no auth required).

The MCP API key is intended to be configured through the app's settings web UI,
which reads and writes `graphiti.env`.

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

## Settings Web UI (Planned)

The app's landing page will provide a settings interface for:
- MCP API key (generate, display, regenerate)
- Neo4j credentials (username, password)
- LLM provider (API key, base URL, model)
- Embedding provider (API key, base URL, model)
- Neo4j Browser link
- MCP connection URL (with auth header example)

All settings are backed by `graphiti.env`.

## Upstream Tracking

When upgrading the upstream Docker image (`zepai/knowledge-graph-mcp:standalone`),
the patched `fastmcp-server.py` must be rebased onto the new version of
`mcp/server/fastmcp/server.py` bundled in the image. Check for changes to
`streamable_http_app()` in particular.
