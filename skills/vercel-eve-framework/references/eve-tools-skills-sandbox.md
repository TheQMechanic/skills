# Eve Tools, Skills & Sandbox

> **Load when**: user asks about `agent/tools/`, `agent/skills/`, `agent/sandbox/`,
> `defineTool`, Zod schemas, `needsApproval`, skill `.md` files, or sandbox backends.
> **If uncertain**, check `node_modules/eve/docs`. Never fabricate `defineTool` or
> `defineSandbox` options not shown here.

---

## Tools — Typed Callable Actions

Each file in `agent/tools/` is one tool. The **filename becomes the tool name** the model sees.

```typescript
// agent/tools/run_sql.ts  →  tool name: `run_sql`
import { defineTool } from 'eve/tools';
import { z } from 'zod';

export default defineTool({
  // What the model reads to decide when to call this tool. Be precise.
  description: 'Run a read-only SQL query against the orders and customers tables.',

  // Zod schema — type-checked and validated at runtime
  inputSchema: z.object({
    sql: z.string().describe('A single read-only SELECT statement.'),
  }),

  async execute({ sql }) {
    const { columns, rows } = await runReadOnlySql(sql);
    return { columns, rows: rows.slice(0, 500), truncated: rows.length > 500 };
  },
});
```

### Tool Naming Rules

- Filename = tool name. `get_weather.ts` → `get_weather`. Use `snake_case`.
- `runSql.ts` → tool name `runSql` (camelCase), **not** `run_sql`. Be intentional.
- The model's decision to use a tool depends entirely on `description`. Make it specific.

### Common Mistakes That Break Tools

| Mistake | Effect | Fix |
|---|---|---|
| `runSql.ts` (camelCase filename) | Tool name is `runSql`, not `run_sql`. Surprises everyone. | Rename to `run_sql.ts` |
| Vague `description: "Do SQL stuff"` | Model never calls the tool; can't distinguish it from others. | Be specific: what tables, what operations, what constraints |
| `inputSchema: z.object({ sql: z.string() })` with no `.describe()` | Model makes up SQL syntax; doesn't know the constraints. | `z.string().describe('A single read-only SELECT statement only.')` |
| Returning a raw array of 10,000 rows | Fills the context window; subsequent turns degrade. | Truncate: `rows.slice(0, 500)` and return a `truncated` flag |
| `async execute(input)` throws an unhandled error | Session fails; no useful error message reaches the model. | Wrap in try/catch and return `{ error: err.message }` |

### Tool Return Values

- Return any JSON-serialisable value.
- Keep responses under ~2,000 tokens when possible — the full return value goes into context.
- For large results (DB rows, file contents), truncate and signal truncation in the return object.

---

## Human-in-the-Loop Approvals

Add `needsApproval` to any tool to gate it behind a human decision.

```typescript
export default defineTool({
  description: 'Run a SQL query against the data warehouse.',
  inputSchema: z.object({ sql: z.string() }),

  // Function form: conditional approval based on input
  needsApproval: ({ toolInput }) => estimateScanGb(toolInput.sql) > 50,

  // Boolean form: always require approval
  // needsApproval: true,

  async execute({ sql }) { /* ... */ },
});
```

**Approval behaviour**:
- The agent **parks** at the gate and consumes **zero compute** while waiting.
- The session remains resumable indefinitely — no timeout.
- On Slack, approval requests render as interactive buttons automatically.
- Once approved, execution continues from exactly the paused point.

---

## Skills — On-Demand Playbooks

Skills are Markdown files in `agent/skills/` loaded **only when relevant** — not always in context.
Use them for domain knowledge, multi-step procedures, or rules that shouldn't bloat every prompt.

```markdown
<!-- agent/skills/revenue-definitions.md -->
---
description: How this team defines revenue. Load before answering any revenue question.
---

Revenue is recognized net of refunds, over the subscription term.
Weeks are Monday-anchored, in UTC.
Exclude trial and internal accounts from every number.
```

### The `description` Frontmatter is Required

Eve uses `description` to decide when to load the skill. Without it, or with a vague value,
the skill may never be activated.

- ✅ `"How this team defines revenue. Load before answering any revenue question."`
- ❌ `"Revenue info"` — too vague to trigger reliably

### Skills vs Instructions

| Put it in `instructions.md` | Put it in a `skills/` file |
|---|---|
| Always-on rules the agent must never forget | Domain knowledge needed only sometimes |
| Hard scope limits | Step-by-step procedures for one category |
| Under ~500 tokens | Can be hundreds of lines |
| Identity and standing rules | Reference material: glossaries, policies, playbooks |

---

## Sandbox — Isolated Compute

Every Eve agent gets a real shell. **All agent-generated code runs in an isolated sandbox**,
separated from the harness environment and its credentials.

```
┌──────────────────────────────────┐
│  Agent Harness (your tools, Eve) │  ← sees env vars, manages the session
└─────────────┬────────────────────┘
              │ mediated by Eve
┌─────────────▼────────────────────┐
│  Sandbox (isolated VM / container)│  ← bash, python, file I/O; cannot see harness env
└──────────────────────────────────┘
```

What the agent can do in the sandbox:
- Run bash commands, shell scripts
- Execute Python, Node, or any installed runtime
- Read and write files in the sandbox filesystem
- Install packages (within the sandbox, not the harness)

### Sandbox Adapter Pattern

The backend is pluggable. Override it in `agent/sandbox/sandbox.ts`:

```typescript
import { defineSandbox } from 'eve/sandbox';

export default defineSandbox({
  backend: 'docker',  // 'microsandbox' | 'just-bash' | custom adapter
});
```

| Environment | Default backend |
|---|---|
| `vercel deploy` | Vercel Sandbox (isolated VMs) |
| `eve dev` locally | Tries Docker → microsandbox → just-bash (first available) |

### Shell Safety Checklist (for coding agents generating sandbox commands)

Before writing or executing a bash command in the sandbox:
1. **Can a file edit achieve this?** If yes, do that instead.
2. **Does this install packages?** Confirm the user wants the install; name what's being installed.
3. **Is this destructive?** (`rm`, `truncate`, database writes) — explain before running.
4. **Does the output contain credentials?** Strip or redact before logging.
5. **Is the path relative or absolute?** Absolute paths that reach outside the workspace can
   be unexpected — verify intent.