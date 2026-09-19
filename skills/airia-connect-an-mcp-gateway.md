---
name: airia-connect-an-mcp-gateway
description: >-
  Connect an MCP client (Claude Code, Claude Desktop, Cursor, or any Streamable-HTTP client) to an
  Airia MCP Gateway, and manage gateways and deployments over the REST API. Use when wiring an agent
  to Airia's governed tool layer.
api: Airia MCP Gateway
endpoint: https://mcp-gateway.airia.ai/gateway/{gateway-id}/mcp
auth: OAuth 2.1 (RFC 9728 protected resource; scopes mcp.read, mcp.write)
operations:
  - McpDeployments_GetDeployments
  - McpDeployments_CreateDeployment
  - McpDeployments_EnsureDeployment
  - McpDeployments_GetCachedTools
  - McpServers_GetMcpSererInfo2
  - SkillsRepositories_GetRepositories2
  - SkillsRepositories_CreateRepository2
  - Tools_GetToolDefinitions2
generated: '2026-09-19'
method: generated
source: https://airia.ai/docs/mcp-servers/end-user-usage/how-to-set-up-a-gateway + openapi/airia-openapi.yml
---

# Connect to an Airia MCP Gateway

An Airia Gateway is **not** a fixed server with a fixed tool list. An administrator composes it from
approved MCP servers, SpecLink-converted OpenAPI specs, Agent Skills repositories and Airia-deployed
agents; the result is one endpoint per gateway.

## 1. The endpoint

```
https://mcp-gateway.airia.ai/gateway/{your-gateway-id}/mcp
https://mcp-gateway.airia.ai/gateway/{your-gateway-id}/radar   # Radar tool-search mode
```

Regional hosts exist — `prodaus.mcp-gateway.airia.ai` serves the Australian environment. The
`{gateway-id}` is issued when the gateway is created; there is no tenant-agnostic URL, and an
anonymous POST to `https://mcp-gateway.airia.ai/mcp` returns **401 `WWW-Authenticate: Bearer`**.

## 2. Connect a client

```bash
# Claude Code
claude mcp add --scope user --transport http my-gateway \
  "https://mcp-gateway.airia.ai/gateway/{your-gateway-id}/mcp"
```

```json
// Claude Desktop, for clients without native remote transport
{ "mcpServers": { "my-gateway": {
    "command": "npx",
    "args": ["-y", "mcp-remote", "https://mcp-gateway.airia.ai/gateway/{your-gateway-id}/mcp"] } } }
```

Cursor installs it in one click. Sign-in is browser-based OAuth on first use.

## 3. The auth, if you are writing the client

The gateway publishes both discovery documents anonymously:

- `GET /.well-known/oauth-protected-resource` — RFC 9728: `resource`, `authorization_servers`,
  `bearer_methods_supported: ["header"]`, scopes including `mcp.read` and `mcp.write`.
- `GET /.well-known/oauth-authorization-server` — RFC 8414, proxying the Keycloak realm at
  `https://identity.airia.ai/auth/realms/airia`, with **dynamic client registration** at
  `/.well-known/oauth-authorization-server/v1/register` and S256 PKCE.

So a client can register itself and complete the flow with no human provisioning a client id. Copies
are saved in `well-known/`.

## 4. Radar, for large gateways

Radar turns tool discovery into a search instead of loading every definition up front. Use the
`/radar` path when a gateway exposes enough tools to crowd the context window.

## 5. What can be behind a gateway

- **Catalogue servers** — Jira, Confluence, Box, Snowflake, Microsoft Graph, Airtable, Brave Search
  and others an admin has approved.
- **Airia Deployed Agents** — any agent with the Tool & MCP interface enabled becomes a callable
  tool; calls always run the latest published version. Passthrough auth runs the call as the caller,
  so two people on one gateway may legitimately see different tools.
- **SpecLink** — a hosted OpenAPI turned into tools without writing a server.
- **Skills over MCP** — `SKILL.md` folders served from a GitHub repo or a remote skills server, with
  the same prompt-injection scanning Airia applies to tool definitions.
- **Airia Datasource MCP Server** — deployed automatically when a data source is attached to an AI
  Model step; exposes semantic/keyword search, multi-store search, filename search, file content
  retrieval, Cypher graph query and multi-store SQL.

## 6. Managing it from the REST API

```
GET  /v1/McpDeployments                       # McpDeployments_GetDeployments
POST /v1/McpDeployments                       # McpDeployments_CreateDeployment
PUT  /v1/McpDeployments/ensure                # McpDeployments_EnsureDeployment  (upsert)
GET  /v1/McpDeployments/{deploymentId}/tools  # McpDeployments_GetCachedTools
GET  /v1/SkillsRepositories                   # SkillsRepositories_GetRepositories2
POST /v1/SkillsRepositories                   # SkillsRepositories_CreateRepository2
GET  /v1/Tools                                # Tools_GetToolDefinitions2
```

`McpDeployments_EnsureDeployment` is an upsert and is the closest thing in this API to a
replay-safe write — there is no `Idempotency-Key` header anywhere in the contract, so treat every
other POST as one that will fire twice if you retry it.

Rate limits apply: `POST /v1/McpDeployments`, `PUT /v1/McpDeployments/{id}`, the `ensure` upsert and
system provisioning all declare **429**.
