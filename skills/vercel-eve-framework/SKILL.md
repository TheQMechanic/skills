---
name: vercel-eve-framework
description: >
  Expert guidance for building AI agents with Vercel's Eve framework — an open-source,
  filesystem-first, TypeScript-native agent framework (public beta since June 17, 2026).

  EXPLICIT triggers — activate when user mentions: "eve agent", "vercel eve", "eve dev",
  "eve framework", "npx eve@latest", agent.ts, agent/instructions.md, agent/tools/,
  agent/skills/, agent/channels/, agent/connections/, agent/schedules/, agent/subagents/,
  "defineTool", "defineAgent", "defineEval", "defineMcpClientConnection", "eve eval",
  "eve channels add", Vercel Sandbox in agent context, Vercel Workflows in agent context,
  AI Gateway in agent context.

  SEMANTIC triggers — activate even without "Eve" keyword when user describes:
  "durable AI agent on Vercel", "long-running AI agent", "agent that survives crashes or
  deploys", "workflow-backed agent", "checkpointed agent sessions", "agent that runs for
  hours / days / weeks", "agent with human-in-the-loop approvals", "filesystem-driven agent
  framework", "Vercel agent framework", "agent directory structure", "agent that deploys to
  Vercel", "agent with sandboxed code execution", "agent with isolated compute".

  ACTION triggers — activate when the user wants to: scaffold a new agent project on Vercel,
  add channels (Slack, Discord, Teams, GitHub) to an existing agent, add approval gates to
  agent tools, connect an agent to MCP servers, set up agent observability with OpenTelemetry,
  test agent behaviour with scored evals, migrate from another agent framework to Eve.

  This skill covers the complete Eve surface. For deep topics it routes to focused reference
  files, each loaded only when relevant, reducing context overhead by ~70%.
---

# Vercel Eve Framework

> **Status**: Public beta — launched June 17, 2026. File conventions and APIs may change.
> **License**: Apache-2.0 | **npm**: `eve` | **GitHub**: `github.com/vercel/eve`

---

## Ground Rules (Read Before Acting)

These apply to every response involving Eve:

1. **Never fabricate Eve APIs.** If a method, option, or file convention isn't in this skill
   or the bundled docs, say "I'm not certain — check `node_modules/eve/docs`" rather than
   inventing it.
2. **Bundled docs are authoritative.** After install they live at `node_modules/eve/docs`.
   When this skill and the bundled docs conflict, **the bundled docs win**.
3. **Version-aware responses.** This skill targets the public beta. Always note when advice
   may shift before GA, and direct users to `github.com/vercel/eve/releases` for the changelog.
4. **Shell safety.** Before running any bash command in a sandbox or dev environment:
   — Prefer targeted file edits over shell commands when they're sufficient.
   — Explain the effect of destructive or network-installing commands before running them.
   — Never embed credentials in commands, flags, or output.

---

## What Eve Is

Eve is a **filesystem-first framework for durable backend AI agents**.
The core contract: **an agent is a directory of files. The directory is the definition.**

| Question | Answer |
|---|---|
| What is the agent? | `agent/instructions.md` — always-on system prompt |
| What model? | `agent/agent.ts` — one `defineAgent` call |
| What can it do? | `agent/tools/*.ts` — one file = one callable tool |
| What does it know? | `agent/skills/*.md` — loaded on demand, not always in context |
| Where does it surface? | `agent/channels/*.ts` — one file per channel (HTTP always on) |
| What services? | `agent/connections/*.ts` — MCP servers or OpenAPI endpoints |
| Runs on a schedule? | `agent/schedules/*.ts` — cron expressions |
| Delegates child work? | `agent/subagents/<name>/` — nested full agent |
| State persists crashes? | Yes — Vercel Workflows checkpoints every step |
| Safe code execution? | Yes — isolated Vercel Sandbox; agent code never touches harness env |

Adding a capability means adding a file. Eve discovers it at build time. No boilerplate.

---

## Minimal Working Agent

Two files. That's the entire agent.

```typescript
// agent/agent.ts
import { defineAgent } from 'eve';
export default defineAgent({
  model: 'anthropic/claude-opus-4.8', // '<provider>/<model>' resolves via AI Gateway
});
```

```markdown
<!-- agent/instructions.md -->
You are a concise assistant. Use tools when they are available.
```

---

## CLI Quick-Reference

| Command | Effect |
|---|---|
| `npx eve@latest init <name>` | Scaffold, install deps, init Git, start dev server |
| `npx eve@latest init <name> --channel-web-nextjs` | Same + web chat UI (add only if user explicitly wants it) |
| `npm install eve@latest` | Add Eve to an existing project |
| `pnpm dev` / `eve dev` | Local dev server + TUI at `http://127.0.0.1:3000` |
| `eve eval` | Run scored test suites against local server |
| `eve eval --url <url>` | Run evals against a deployed agent |
| `eve channels add <name>` | Scaffold a channel file |
| `vercel deploy` | Deploy to production (same as any Vercel project) |

> **Beta usage**: `eve@latest` gets current releases. Review `github.com/vercel/eve/releases`
> before any upgrade — beta releases may contain breaking changes.

---

## Reference File Routing

Load **only** the file that matches the current query. Do not preload all references.

| "I need to..." or topic | Load |
|---|---|
| Configure `agent.ts`, choose/change model, write `instructions.md`, set `defineAgent` options | `references/eve-core-primitives.md` |
| Add a tool, write a Zod schema, require human approval, create a skill `.md`, use the sandbox | `references/eve-tools-skills-sandbox.md` |
| Add Slack / Discord / Teams / Telegram / GitHub / Linear / web channel, custom channel | `references/eve-channels.md` |
| Connect to MCP server, add OpenAPI integration, handle OAuth via Vercel Connect | `references/eve-connections.md` |
| Use the HTTP API, manage sessions, attach to NDJSON stream, send continuation messages | `references/eve-sessions-api.md` |
| Write an eval, assert tool calls, set up CI deploy gates, debug agent regressions | `references/eve-evals-ci.md` |
| Understand Vercel Workflows / Sandbox / AI Gateway / Connect, configure OpenTelemetry | `references/eve-infrastructure.md` |
| Add a subagent, create a cron schedule, compare Eve to Temporal / LangGraph / Mastra | `references/eve-advanced.md` |

## Import Quick-Reference

```typescript
import { defineAgent } from 'eve';                          // agent/agent.ts
import { defineTool } from 'eve/tools';                     // agent/tools/*.ts
import { defineSandbox } from 'eve/sandbox';                // agent/sandbox/sandbox.ts
import { defineSlackChannel } from 'eve/channels/slack';   // Slack channel
import { defineChannel } from 'eve/channels';               // custom channels
import { defineMcpClientConnection } from 'eve/connections'; // MCP connection
import { defineApiConnection } from 'eve/connections';      // OpenAPI connection
import { defineSchedule } from 'eve/schedules';             // agent/schedules/*.ts
import { defineEval } from 'eve/evals';                     // evals/*.eval.ts
import { includes, not, matches, contains } from 'eve/evals/expect';
import { z } from 'zod';                                    // always use for inputSchema
```

---

## Coding-Agent Bootstrap Prompt

Paste this into any coding agent (Claude Code, Cursor, Windsurf) to scaffold an Eve project:

```
Set up an Eve agent. Eve is a filesystem-first TypeScript framework for durable agents,
published as the npm package `eve` (Apache-2.0).

BEFORE ACTING: Read bundled docs at node_modules/eve/docs (post-install). Pre-install:
read vercel.com/docs/eve and vercel.com/blog/introducing-eve.

SCAFFOLD: npx eve@latest init <name>
  — Add --channel-web-nextjs ONLY if the user explicitly asked for a web chat UI.
  — init installs deps, starts a dev server. Run in a controllable process; stop before editing.
To add to an existing project: npm install eve@latest

ENSURE these exist: agent/agent.ts (defineAgent with a model string) + agent/instructions.md

ADD a first typed tool at agent/tools/get_weather.ts:
  — Use defineTool from eve/tools
  — Use Zod for inputSchema
  — Return a plain JSON-serialisable value from execute()

VERIFY: pnpm typecheck (or tsc --noEmit). Fix all type errors before proceeding.

EXERCISE the HTTP API:
  POST   /eve/v1/session              — create session, capture sessionId + continuationToken
  GET    /eve/v1/session/:id/stream   — attach to NDJSON stream
  POST   /eve/v1/session/:id/message  — send follow-up with continuationToken header

RULES:
  — Never invent Eve APIs. If uncertain, read node_modules/eve/docs and say so.
  — Do not commit unless the user explicitly asks.
  — Shell commands: explain destructive effects before running; never expose credentials.
```