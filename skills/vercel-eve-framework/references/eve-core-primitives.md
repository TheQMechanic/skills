# Eve Core Primitives

> **Load when**: user asks about `agent.ts`, `instructions.md`, model selection,
> `defineAgent` options, or "what are the required files".
> **Never fabricate** `defineAgent` fields not shown here — check `node_modules/eve/docs`.

---

## The Two Required Files

Every Eve agent needs exactly two files. Everything else is optional.

```
agent/
  agent.ts          ← model and runtime config
  instructions.md   ← always-on system prompt
```

---

## `agent/agent.ts` — Full Options

```typescript
import { defineAgent } from 'eve';

export default defineAgent({
  // Required: model string resolved via AI Gateway
  model: 'anthropic/claude-opus-4.8',

  // Optional: fallback models if primary is unavailable (AI Gateway handles routing)
  // modelFallbacks: ['anthropic/claude-sonnet-4.6', 'openai/gpt-5.4-mini'],

  // Optional: context window compaction strategy (beta — check bundled docs for current API)
  // compaction: 'auto',

  // Optional: extra instructions appended to instructions.md
  // system: 'Always respond in the user\'s language.',
});
```

### Model String Format

```
<provider>/<model-name>
```

Examples:
- `anthropic/claude-opus-4.8` — highest reasoning, best for judgment-heavy agents
- `anthropic/claude-sonnet-4.6` — balanced speed/quality
- `openai/gpt-5.4-mini` — fast and cheap, good for routing/dispatch agents
- `google/gemini-2.5-pro` — strong multimodal capabilities

**On Vercel**: model strings resolve via AI Gateway with OIDC — no per-provider API keys needed.

**Locally / off-Vercel**: set the provider env var:
```bash
ANTHROPIC_API_KEY=sk-ant-...    # for anthropic/
OPENAI_API_KEY=sk-...           # for openai/
GOOGLE_GENERATIVE_AI_API_KEY=...  # for google/
```

### Choosing a Model

| Agent type | Recommended tier |
|---|---|
| Data analysis, sales, content review | Opus-class — judgment errors are silent and hard to catch |
| Routing, dispatch, simple Q&A | Mini-class — fast, cheap, sufficient |
| Subagent (specialist, narrow task) | Can differ from parent — configure independently |

The model is swappable with one line change. No other file needs updating.

---

## `agent/instructions.md` — System Prompt

Plain Markdown. Eve prepends this verbatim to **every** model call.

```markdown
You are a senior data analyst. You answer questions about the team's data.

- Prefer exact numbers to hand-waving. If you can compute it, compute it.
- State the assumptions behind any number you report (date range, filters, grain).
- Use the tools available to you rather than guessing.
  If you cannot answer from the data, say so plainly.
```

### Writing Effective Instructions

**Do**:
- Name the agent's role in one sentence at the top.
- List hard limits explicitly ("Never query a client's workspace data.").
- Name which skills exist and when to invoke them ("Use the `revenue-definitions` skill
  before answering any revenue question.").
- Keep it under ~500 tokens — it's in context on every turn.

**Avoid**:
- Restating what the model already knows (general reasoning, common politeness).
- Vague scope ("be helpful with data"). Specific beats vague.
- Embedding domain knowledge here that should live in a `skills/` file.

### Scope + Hard Boundaries Example

```markdown
You are a financial assistant for Acme Corp.

SCOPE: Only answer questions about Acme's own financial data.
NEVER: Query or reference competitor data, client data, or data outside your permissions.
ALWAYS: Cite the SQL query you ran when giving a numeric answer.

Available skills: `revenue-definitions` (load before any ARR/MRR question).
```

---

## Project Layout Summary

```
agent/
  agent.ts             # required — model + runtime config
  instructions.md      # required — always-on system prompt

  tools/               # optional — one .ts file = one callable tool
  skills/              # optional — .md playbooks loaded on demand
  channels/            # optional — surfaces (HTTP always on, others opt-in)
  connections/         # optional — MCP or OpenAPI integrations
  schedules/           # optional — cron-triggered autonomous runs
  subagents/           # optional — nested child agents
  sandbox/             # optional — compute backend override

evals/                 # optional — scored test suites (outside agent/)
```

A file's location and name are its registration. Add the file; Eve handles the rest.