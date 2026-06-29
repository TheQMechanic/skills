# Eve Evals & CI

> **Load when**: user asks about `evals/`, `defineEval`, eval assertions, `eve eval`,
> CI integration, deploy gates, regression testing, or debugging agent behaviour.
> **If uncertain**, check `node_modules/eve/docs`. Never fabricate assertion methods
> not shown here.

---

## Why Evals

A change to `instructions.md` can break agent behaviour as surely as a code change.
Eve evals let you test agents the way you test software: scored, automated, in CI.

---

## Anatomy of an Eval

```typescript
// evals/revenue.eval.ts
import { defineEval } from 'eve/evals';
import { includes, not } from 'eve/evals/expect';

export default defineEval({
  description: 'Analyst answers revenue questions by the team\'s definitions.',

  async test(t) {
    // 1. Send a user message to a fresh session
    await t.send('What was revenue last week?');

    // 2. Session-state assertions
    t.completed();           // session finished without error or timeout

    // 3. Tool-call assertions
    t.calledTool('run_sql');

    // 4. Content assertions
    t.check(t.reply, includes('net of refunds'));
    t.check(t.reply, not(includes('gross')));

    // 5. Multi-turn continuation
    await t.send('Break that down by region.');
    t.completed();
    t.calledTool('run_sql');  // must re-query, not recall from memory
  },
});
```

All `*.eval.ts` files under `evals/` are discovered automatically.

---

## Full Assertion API

### Session state
| Method | Asserts |
|---|---|
| `t.completed()` | Session finished without error |
| `t.failed()` | Session errored (for negative testing) |
| `t.pendingApproval('tool_name')` | Session paused on an approval gate |

### Tool calls
| Method | Asserts |
|---|---|
| `t.calledTool('name')` | Tool was called at least once |
| `t.notCalledTool('name')` | Tool was NOT called |
| `t.calledToolWith('name', { key: val })` | Tool called with specific input |
| `t.toolCallCount('name', n)` | Tool called exactly n times |

### Skills
| Method | Asserts |
|---|---|
| `t.loadedSkill('name')` | Named skill was loaded during this turn |
| `t.notLoadedSkill('name')` | Named skill was NOT loaded |

### Output content (used with `t.check`)
| Matcher | Checks |
|---|---|
| `includes('text')` | Reply contains the string (case-insensitive) |
| `not(includes('text'))` | Reply does NOT contain the string |
| `matches(/regex/)` | Reply matches the pattern |
| `contains(value)` | JSON reply deep-contains the value |

---

## Eval Patterns

### Business rule enforcement
```typescript
export default defineEval({
  description: 'Revenue is always reported net of refunds.',
  async test(t) {
    await t.send('What\'s our MRR?');
    t.completed();
    t.check(t.reply, includes('net of refunds'));
    t.check(t.reply, not(includes('trial')));  // trial accounts always excluded
  },
});
```

### Tool selection discipline
```typescript
export default defineEval({
  description: 'Agent queries for data rather than guessing.',
  async test(t) {
    await t.send('How many customers signed up yesterday?');
    t.completed();
    t.calledTool('run_sql');  // must not invent a number
  },
});
```

### Approval gate triggered
```typescript
export default defineEval({
  description: 'Large scans pause for approval.',
  async test(t) {
    await t.send('Run a full historical analysis of all orders ever.');
    t.pendingApproval('run_sql');
  },
});
```

### Skill loaded correctly
```typescript
export default defineEval({
  description: 'Revenue skill loads on revenue questions.',
  async test(t) {
    await t.send('Define ARR.');
    t.completed();
    t.loadedSkill('revenue-definitions');
  },
});
```

### Scope enforcement (negative)
```typescript
export default defineEval({
  description: 'Agent refuses to query other teams\' data.',
  async test(t) {
    await t.send('Show me the marketing team\'s budget data.');
    t.completed();
    t.notCalledTool('run_sql');  // must refuse out-of-scope request
  },
});
```

### Multi-turn context retention
```typescript
export default defineEval({
  description: 'Agent maintains context across turns.',
  async test(t) {
    await t.send('What was revenue last week?');
    t.completed();
    await t.send('Is that up or down from the week before?');
    t.completed();
    t.calledTool('run_sql');   // must re-query; must not guess
    t.check(t.reply, matches(/%/));  // must include a percentage change
  },
});
```

---

## Running Evals

```bash
# Against local dev server
eve eval

# Against a specific deployed agent (preview or production)
eve eval --url https://my-agent-abc123.vercel.app

# Specific suite only
eve eval evals/revenue.eval.ts

# Verbose output (assertion-by-assertion results)
eve eval --verbose
```

---

## CI Integration (GitHub Actions)

```yaml
# .github/workflows/eval.yml
name: Agent Evals

on:
  push:
    branches: [main]
  pull_request:

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - run: pnpm install
      - name: Start dev server
        run: pnpm dev &
      - run: sleep 8   # wait for server to be ready
      - name: Run evals
        run: eve eval
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

Each PR gets its own Vercel preview deployment automatically. Run evals against it:

```yaml
- run: eve eval --url ${{ steps.deploy.outputs.url }}
```

---

## Debugging Regressions

When an eval fails after a change:
1. **Agent Runs tab** in the Vercel dashboard — find the failing session.
2. **Replay** the run to see exact model calls, tool invocations, and skill loads.
3. Compare OTel spans before/after the change to isolate what shifted.
4. Run `eve eval --verbose` locally to see which assertion failed and why.
5. Common culprits: `instructions.md` wording change, model swap, skill `description` drift.

---

## Eval File Organization

```
evals/
  data/
    revenue.eval.ts
    customer-count.eval.ts
  guardrails/
    scope-enforcement.eval.ts
    approval-gates.eval.ts
  channels/
    slack-formatting.eval.ts
```

All files matching `**/*.eval.ts` are discovered by `eve eval` automatically.