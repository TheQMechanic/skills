# Eve Connections

> **Load when**: user asks about `agent/connections/`, MCP servers, OpenAPI integrations,
> Vercel Connect, OAuth, credential handling, or connecting to external services
> (GitHub, Slack, Snowflake, Salesforce, Notion, Linear, Stripe, etc.).
> **If uncertain**, check `node_modules/eve/docs`. Never fabricate connection config
> options not shown here.

---

## What Connections Do

A connection is a file that points at an MCP server or OpenAPI-compatible API. Eve:
1. Discovers the remote tools automatically.
2. Hands their descriptions to the model.
3. Brokers authentication between the model and the service.

**The model never sees the connection's URL or credentials.**
This is the security contract. Credentials live in env vars or Vercel Connect; the model
only sees tool descriptions and return values.

---

## MCP Server Connection

```typescript
// agent/connections/linear.ts
import { defineMcpClientConnection } from 'eve/connections';

export default defineMcpClientConnection({
  url: 'https://mcp.linear.app/sse',
  description: 'Linear workspace: issues, projects, cycles, and comments.',
  auth: {
    getToken: async () => ({ token: process.env.LINEAR_API_TOKEN! }),
  },
});
```

```typescript
// agent/connections/github.ts
import { defineMcpClientConnection } from 'eve/connections';

export default defineMcpClientConnection({
  url: 'https://api.githubcopilot.com/mcp/',
  description: 'GitHub: repos, issues, PRs, code search, commits.',
  auth: {
    getToken: async () => ({ token: process.env.GITHUB_TOKEN! }),
  },
});
```

---

## OpenAPI / REST Connection

```typescript
// agent/connections/orders-api.ts
import { defineApiConnection } from 'eve/connections';

export default defineApiConnection({
  openApiUrl: 'https://api.example.com/openapi.json',
  description: 'Internal orders API: create, read, and update orders.',
  auth: {
    type: 'bearer',
    getToken: async () => process.env.ORDERS_API_KEY!,
  },
});
```

The OpenAPI document is fetched once and used to derive the model's available tools.
Only endpoints listed in the document are exposed. The model cannot reach unlisted routes.

---

## Vercel Connect (Managed OAuth)

For production agents on Vercel, use **Vercel Connect** instead of raw env vars:

```typescript
// agent/connections/slack.ts — using Vercel Connect
import { defineMcpClientConnection } from 'eve/connections';
import { getVercelConnectToken } from 'eve/connect';

export default defineMcpClientConnection({
  url: 'https://mcp.slack.com/sse',
  description: 'Slack workspace: messages, channels, users, reactions.',
  auth: {
    getToken: async (context) => ({
      token: await getVercelConnectToken(context, 'slack'),
    }),
  },
});
```

**Vercel Connect handles**:
- Initial OAuth consent flow (user-facing UI)
- Encrypted token storage (not in your codebase)
- Automatic token refresh before expiry

**Supported OAuth services at launch**: Slack, GitHub, Snowflake, Salesforce, Notion, Linear —
plus anything reachable over OAuth, an API key, or any MCP server.

---

## Connection File Naming

The filename is how you and the model identify the service. Use the service name:
`slack.ts`, `github.ts`, `snowflake.ts`, `orders-api.ts`. No registration needed.

---

## Supported Connection Sources

| Type | Use when |
|---|---|
| MCP server (`defineMcpClientConnection`) | Service publishes an MCP endpoint (Linear, GitHub Copilot, etc.) |
| OpenAPI (`defineApiConnection`) | Service has an OpenAPI 3.x spec; want auto-tool-discovery from endpoints |
| Raw tool in `tools/` | Service has no MCP/OpenAPI; writing a custom TypeScript wrapper is simpler |

---

## Security Checklist for Connections

- **Never hardcode tokens** in connection files. Use env vars or Vercel Connect.
- **Use read-only scopes** where possible (`LINEAR_API_TOKEN` with read scope only, etc.).
- **Use `description` carefully** — it tells the model what the service exposes.
  An overly broad description may cause the model to attempt actions you didn't intend.
- **Audit MCP server trust** before adding. A malicious MCP server can send tool descriptions
  that attempt prompt injection. Only add MCP servers from trusted sources.
- **Local dev vs production** — always use real credentials via env vars locally, and switch
  to Vercel Connect before deploying.