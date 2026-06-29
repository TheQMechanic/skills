# Eve Advanced Patterns

> **Load when**: user asks about subagents, schedules, comparing Eve to other frameworks,
> production security patterns, running Eve off Vercel, or advanced architectural decisions.
> **Never fabricate** framework comparison details — these are best-effort based on public docs.

---

## Subagents

A subagent is a full nested agent directory inside `agent/subagents/<name>/`.
It has its own identity, model, tools, and sandbox — and its own clean context window.

```
agent/
  subagents/
    investigator/
      agent.ts          # can use a different (or more powerful) model
      instructions.md   # its own scoped identity and rules
      tools/
        deep_search.ts
```

```typescript
// agent/subagents/investigator/agent.ts
import { defineAgent } from 'eve';
export default defineAgent({
  description: 'Investigates data anomalies before the analyst reports them.',
  model: 'anthropic/claude-opus-4.8',
});
```

**How delegation works**:
1. The parent agent calls the subagent like a tool.
2. The child starts with a **fresh context window** — no history from the parent.
3. The child has only the tools and connections defined in its own directory.
4. The child completes its task and returns a result to the parent.

**When to use subagents**:
- Work that benefits from a narrower context (less noise → fewer errors).
- Tasks that need a different model than the parent (e.g., cheaper routing parent + expensive specialist child).
- Work that can run in parallel.
- Isolation: the child cannot access the parent's tools, connections, or secrets.

---

## Schedules (Autonomous Cron Runs)

```typescript
// agent/schedules/monday-summary.ts
import { defineSchedule } from 'eve/schedules';
import slack from '../channels/slack.js';

export default defineSchedule({
  cron: '0 9 * * 1',   // Every Monday at 09:00 UTC
  async run({ receive, waitUntil, appAuth }) {
    waitUntil(
      receive(slack, {
        message: 'Summarize last week\'s revenue and post it to #revenue.',
        target: { channelId: 'C0123ABC' },
        auth: appAuth,
      }),
    );
  },
});
```

On Vercel, each schedule compiles to a **Vercel Cron Job**. No dashboard configuration required.

**Cron expression format**: Standard 5-field cron. `0 9 * * 1` = 09:00 UTC every Monday.

---

## Production Security Patterns

### 1 — Principle of least privilege for connections
Grant connections the narrowest scope possible:
- `LINEAR_API_TOKEN` with read-only scope if the agent only queries issues.
- Separate tokens per agent if you run multiple agents against the same service.

### 2 — Scope limits in `instructions.md`
State data boundaries explicitly. The model will respect clear written constraints:
```markdown
NEVER: Query, read, or reference data outside your own workspace permissions.
NEVER: Access client data, even if a connected service technically exposes it.
```

### 3 — Approval gates for destructive actions
Any tool that writes, deletes, or sends should have `needsApproval: true` in production
until you have sufficient eval coverage to trust it unsupervised:
```typescript
needsApproval: ({ toolInput }) => toolInput.operation === 'delete',
```

### 4 — Eval-gated deploys
Wire `eve eval` into CI as a required check. Prompt changes and model swaps that break
evals will fail the deploy before reaching users.

### 5 — Immutable deploys + instant rollback
Vercel deploys are immutable. Every push is a new deployment from scratch. If a bad
change reaches production, `vercel rollback` returns to the previous version instantly.

---

## Eve vs Other Agent Frameworks

> **Note**: These comparisons are based on public documentation as of June 2026. Frameworks
> evolve quickly. Always verify against each framework's current docs before making architectural decisions.

### Eve vs Mastra

| | Eve | Mastra |
|---|---|---|
| **Language** | TypeScript | TypeScript |
| **Infrastructure** | Vercel-native (Workflows, Sandbox, Gateway) | Platform-neutral (run anywhere) |
| **State persistence** | Vercel Workflows (built-in) | Bring your own (Postgres, Redis, etc.) |
| **Agent definition** | Directory of files | Code-defined with explicit wiring |
| **Deployment** | `vercel deploy` | Any host |
| **Maturity** | Public beta (June 2026) | v1.0 (January 2026) |

**Choose Eve** when you're already on Vercel and want zero infrastructure configuration.
**Choose Mastra** when you need full platform flexibility or are deep in AWS/GCP/Azure IAM.

### Eve vs LangGraph

| | Eve | LangGraph |
|---|---|---|
| **Language** | TypeScript | Python (JS SDK available) |
| **Graph model** | Implicit (Eve owns the loop) | Explicit (you define nodes and edges) |
| **Durability** | Built-in via Vercel Workflows | LangGraph Platform (paid) or DIY |
| **Control flow** | Declarative (files) | Imperative (graph definition code) |
| **Best for** | Backend agents with channels and tools | Fine-grained control flow, research agents |

**Choose Eve** for production backend agents with minimal control-flow complexity.
**Choose LangGraph** when you need explicit, auditable graph execution paths.

### Eve vs Temporal (direct durability comparison)

| | Eve / Vercel Workflows | Temporal |
|---|---|---|
| **What it provides** | Durable agent sessions via checkpointed workflows | Durable workflow execution engine |
| **Operation model** | Managed (Vercel handles infra) | Self-hosted or Temporal Cloud |
| **Eve integration** | Native (Eve builds on Workflow SDK) | Would require a custom Eve adapter |
| **Replay semantics** | Event log replay | Deterministic workflow replay |
| **Use case** | Agent sessions that survive crashes | Long-running business processes, sagas |

Eve's durability is conceptually similar to Temporal but scoped to agent conversations.
You don't interact with it directly — Eve wraps it. For agents, this is the right level
of abstraction. For complex business workflows *outside* agents, Temporal is more appropriate.

### Eve vs OpenAI Agents SDK

| | Eve | OpenAI Agents SDK |
|---|---|---|
| **Language** | TypeScript | Python |
| **Model support** | Any provider via AI Gateway | OpenAI-native (other providers possible) |
| **Infrastructure** | Vercel-native | Bring your own |
| **Agent definition** | Filesystem (directory) | Code (Runner + Agent objects) |
| **Open-source** | Yes (Apache-2.0) | Yes (MIT) |

**Choose Eve** for TypeScript teams wanting full-stack agent + frontend deployment in one.
**Choose OpenAI Agents SDK** for Python teams tightly integrated with OpenAI's platform.

---

## Running Eve Off Vercel

Eve is platform-neutral in principle. Each production capability is an adapter.
Porting to AWS, GCP, or self-hosted infra means replacing:

| Eve capability | Vercel default | Replacement needed |
|---|---|---|
| Durable sessions | Vercel Workflows | Custom workflow adapter (e.g., Temporal, Redis-backed) |
| Sandboxed compute | Vercel Sandbox | Docker, Fly Machines, AWS ECS task, or custom adapter |
| Model routing | AI Gateway | Direct provider SDKs or custom gateway |
| Credential brokering | Vercel Connect | Custom OAuth server or env var management |

This is non-trivial. The adapter pattern makes it technically possible, but the friction is
real — this is the same tradeoff that exists with Next.js on non-Vercel hosts.
Early community reports indicate this requires significant effort.
Evaluate platform lock-in before committing to Eve for non-Vercel production targets.