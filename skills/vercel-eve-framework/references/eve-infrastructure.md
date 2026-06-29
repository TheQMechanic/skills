# Eve Infrastructure

> **Load when**: user asks about Vercel Workflows, Vercel Sandbox, AI Gateway, Vercel Connect,
> OpenTelemetry, the Agent Runs dashboard, provider fallbacks, OIDC auth, or how Eve's
> production infrastructure works under the hood.
> **If uncertain**, check `node_modules/eve/docs`. Never fabricate infrastructure
> configuration options not shown here.

---

## The Four Infrastructure Layers

Eve bundles four Vercel platform services. Each is an adapter — the interface is stable
even if the implementation is swapped.

```
┌─────────────────────────────────────────────────┐
│  eve (the framework)                             │
├─────────────────────────────────────────────────┤
│  Vercel Workflows    — durable session state     │
│  Vercel Sandbox      — isolated code execution   │
│  AI Gateway          — model routing + fallback  │
│  Vercel Connect      — credential brokering      │
└─────────────────────────────────────────────────┘
```

All four are included when deploying to Vercel. Running off-Vercel requires replacing each
adapter — technically possible but non-trivial. See `eve-advanced.md` for platform tradeoffs.

---

## Vercel Workflows — Durable Execution

**What it does**: Every Eve session is a Vercel Workflow. State is persisted as an event log
and replayed deterministically to reconstruct context after any interruption.

**Durability guarantees**:
- Process crash mid-turn → resume from last checkpoint, re-run in-flight tool call.
- Cold start → restore session from event log, continue where it left off.
- New deployment → in-flight sessions finish on the version they started; new sessions use new code.
- Waiting for approval, next message, or async result → zero compute consumed while parked.

**The event log**:
Every tool call, model response, and skill load is written to the log before the next
step starts. "Replay" in the Agent Runs dashboard reconstructs this log visually.

**Built on**: The open-source [Workflow SDK](https://workflow-sdk.dev/). You can use it
directly for custom durable logic outside the Eve agent loop.

---

## Vercel Sandbox — Isolated Compute

**Security model**:
```
Agent Harness env vars ───┐
Agent Harness filesystem ──┼── NOT visible to sandbox
Agent Harness network ─────┘

Sandbox: its own filesystem, env, and network context
```

**How the sandbox adapter works**:

| Environment | Backend |
|---|---|
| `vercel deploy` | Vercel Sandbox (isolated VMs, managed by Vercel) |
| `eve dev` local | First available: Docker → microsandbox → just-bash |

Override the backend:
```typescript
// agent/sandbox/sandbox.ts
import { defineSandbox } from 'eve/sandbox';
export default defineSandbox({ backend: 'docker' });
```

**For BYOC (Bring Your Own Cloud)**: Eve can run on AWS via Vercel BYOC. Vercel becomes
the management layer; OIDC tokens are issued per-invocation but AWS IAM role assumption
is not supported. Custom sandbox adapters work with any OCI-compatible container runtime.

---

## AI Gateway — Model Routing

**What it does**: Resolves model strings like `anthropic/claude-opus-4.8` to the right
provider, streams responses, and handles provider-level failures with automatic fallback.

**On Vercel**: Authentication uses OIDC — no per-provider API keys needed in your codebase.

**Off Vercel**: Set the provider env var (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc.).

**Fallback behaviour**: If the primary model is unavailable, AI Gateway routes to the next
configured fallback without an error visible to the agent or user.

**Provider support**: Any model provider that AI SDK supports — OpenAI, Anthropic, Google,
Mistral, Cohere, and many more. The model string format is `<provider>/<model-id>`.

---

## Vercel Connect — Credential Brokering

**What it does**: Manages OAuth tokens and API keys for external services so your code
never stores or refreshes credentials.

**Token lifecycle**:
1. Initial OAuth consent → user-facing consent UI (Vercel-hosted)
2. Token storage → encrypted, never in your git repo or env vars
3. Token refresh → automatic, before expiry, no code changes required

**Supported at launch**: Slack, GitHub, Snowflake, Salesforce, Notion, Linear — plus anything
reachable over OAuth, API key, or MCP.

**Local dev fallback**: Use env vars locally (`SLACK_BOT_TOKEN`, `GITHUB_TOKEN`, etc.).
Switch to Connect before deploying to production.

---

## OpenTelemetry Observability

**Zero-config on Vercel**: Agent Runs tab appears automatically in your Vercel project's
Observability section. No instrumentation code required.

### Span tree (per turn)
```
ai.eve.turn                       ← one span per turn
├── ai.streamText                 ← the model call
│   └── ai.streamText.doStream
├── ai.toolCall                   ← one per tool invocation (includes inputs + outputs)
│   └── sandbox.bash              ← if the tool ran shell commands
└── ai.skillLoad                  ← if a skill was loaded
```

### Exporting to third-party backends

```typescript
// agent/instrumentation.ts
import { registerOTel } from '@vercel/otel';
export function register() {
  registerOTel({
    serviceName: 'my-agent',
    // Add exporters for Braintrust, Honeycomb, Datadog, Jaeger, etc.
  });
}
```

Or set env vars:
```bash
OTEL_EXPORTER_OTLP_ENDPOINT=https://api.honeycomb.io
OTEL_EXPORTER_OTLP_HEADERS="x-honeycomb-team=<your-key>"
```

### What's visible per session in the Agent Runs tab
- All turns in order, with timing and token counts
- Each tool call: inputs, outputs, duration
- Each skill load: which skill, which turn
- Replay: exact sequence of events for any past run
- Pending approvals: visible and actionable from the dashboard