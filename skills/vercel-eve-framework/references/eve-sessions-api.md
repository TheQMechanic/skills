# Eve Sessions & HTTP API

> **Load when**: user asks about the HTTP API, sessions, streaming, continuation tokens,
> NDJSON events, multi-turn conversations, or session durability behaviour.
> **If uncertain**, check `node_modules/eve/docs`. Never fabricate API routes or
> response shapes not shown here.

---

## Session Concepts

Every Eve conversation is a **durable session** — a Vercel Workflow under the hood.

| Property | Value |
|---|---|
| State persistence | Checkpointed at every step. Survives crashes, cold starts, new deploys. |
| Compute while waiting | Zero. Session parks between messages at no cost. |
| Max pause duration | None. A session waiting for human approval can wait days or weeks. |
| Deploy interruption | None. An in-flight session finishes on the version it started on. |
| Resumability | Any session can be reattached to via its stream endpoint. |

---

## HTTP API Endpoints

Base URL: `http://127.0.0.1:3000` locally; your Vercel deployment URL in production.

---

### POST `/eve/v1/session` — Create a session

```bash
curl -X POST http://127.0.0.1:3000/eve/v1/session \
  -H 'content-type: application/json' \
  -d '{"message": "What was revenue last week?"}'
```

**Response headers**:
- `x-eve-session-id` — the session ID (use to reattach to the stream)

**Response body** (JSON):
```json
{
  "continuationToken": "<token>",
  "sessionId": "<id>"
}
```

---

### GET `/eve/v1/session/:id/stream` — Attach to the live stream

```bash
curl http://127.0.0.1:3000/eve/v1/session/<sessionId>/stream
```

Returns **NDJSON** (newline-delimited JSON). Each line is one lifecycle event.

#### Common event types

```jsonc
// Model producing output
{ "type": "text", "content": "Revenue for the week of June 1 was $4.2M..." }

// Tool being called
{ "type": "tool_call", "tool": "run_sql", "input": { "sql": "SELECT ..." } }

// Tool result returned
{ "type": "tool_result", "tool": "run_sql", "output": { "rows": [...] } }

// Skill loaded for this turn
{ "type": "skill_load", "skill": "revenue-definitions" }

// Session paused, waiting for approval
{ "type": "approval_required", "tool": "run_sql", "input": { "sql": "..." } }

// Session complete
{ "type": "done" }
```

---

### POST `/eve/v1/session/:id/message` — Send a follow-up

```bash
curl -X POST http://127.0.0.1:3000/eve/v1/session/<sessionId>/message \
  -H 'content-type: application/json' \
  -H 'x-eve-continuation-token: <continuationToken>' \
  -d '{"message": "Break that down by region."}'
```

The response returns a new `continuationToken` for the next follow-up.

---

### Approval workflow via HTTP

When a tool with `needsApproval` fires, the stream emits `approval_required`.
Resume the session by posting an approval decision:

```bash
# Approve
curl -X POST http://127.0.0.1:3000/eve/v1/session/<sessionId>/approve \
  -H 'x-eve-continuation-token: <token>' \
  -d '{"approved": true}'

# Deny
curl -X POST http://127.0.0.1:3000/eve/v1/session/<sessionId>/approve \
  -H 'x-eve-continuation-token: <token>' \
  -d '{"approved": false, "reason": "Query too expensive"}'
```

---

## Minimal Node.js Client

```typescript
// Start a session and stream events
const res = await fetch('http://127.0.0.1:3000/eve/v1/session', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ message: 'What is the weather in Brooklyn?' }),
});

const sessionId = res.headers.get('x-eve-session-id');
const { continuationToken } = await res.json();

// Attach to stream
const stream = await fetch(`http://127.0.0.1:3000/eve/v1/session/${sessionId}/stream`);
const reader = stream.body!.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  for (const line of decoder.decode(value).split('\n').filter(Boolean)) {
    const event = JSON.parse(line);
    console.log(event.type, event.content ?? event.tool ?? '');
  }
}
```

---

## Session Durability in Practice

```
Agent starts turn 1
  ↓ checkpoint
Agent calls run_sql
  ↓ checkpoint
Server crashes mid-tool-call
  ↓ restart
Workflow replays from last checkpoint
  ↓ tool result re-fetched
Agent completes turn 1
  ↓ awaiting next message (zero compute consumed while parked)
User sends turn 2
  ↓ resume
...
```

A `vercel deploy` during an active session does not interrupt it. The running session
finishes on the code version it started on; the new version serves subsequent sessions.