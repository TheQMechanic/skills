# Eve Channel Adapters

> **Load when**: user asks about `agent/channels/`, Slack, Discord, Teams, Telegram,
> GitHub, Linear, web chat, custom channels, `defineChannel`, or multi-channel routing.
> **If uncertain**, check `node_modules/eve/docs`. Never fabricate channel config options
> not shown here.

---

## Channel Concepts

- **HTTP is always on** — no channel file required. Every Eve agent gets an HTTP endpoint.
- **Every other channel is one file** in `agent/channels/`.
- **Scaffold with**: `eve channels add <name>` (slack / discord / teams / telegram / github / linear)
- **One codebase, every surface** — the same agent, deployed once, answers on all channels.
- **Sessions cross channels**: an HTTP webhook can hand off to a Slack thread.

---

## Slack

```typescript
// agent/channels/slack.ts
import { defineSlackChannel } from 'eve/channels/slack';

export default defineSlackChannel({
  // Credentials via Vercel Connect — no SLACK_BOT_TOKEN in .env needed in production.
  // For local dev: set SLACK_BOT_TOKEN env var.
});
```

Scaffold: `eve channels add slack`

**Platform affordances (automatic)**:
- Approval gates render as interactive Slack buttons.
- Typing indicators while the agent is processing.
- Select menus for structured choices.
- Rich block-kit formatting.

---

## Discord

```typescript
// agent/channels/discord.ts
import { defineDiscordChannel } from 'eve/channels/discord';

export default defineDiscordChannel({
  // Local dev: set DISCORD_BOT_TOKEN env var
  // Production: route via Vercel Connect
});
```

---

## Microsoft Teams

```typescript
// agent/channels/teams.ts
import { defineTeamsChannel } from 'eve/channels/teams';

export default defineTeamsChannel({
  // Requires Azure Bot Service credentials
  // Set TEAMS_APP_ID and TEAMS_APP_PASSWORD (or route via Vercel Connect)
});
```

---

## Telegram

```typescript
// agent/channels/telegram.ts
import { defineTelegramChannel } from 'eve/channels/telegram';

export default defineTelegramChannel({
  // Set TELEGRAM_BOT_TOKEN env var (or route via Vercel Connect)
});
```

---

## GitHub

```typescript
// agent/channels/github.ts
import { defineGithubChannel } from 'eve/channels/github';

export default defineGithubChannel({
  // Responds to GitHub events: issues, PRs, comments, webhooks
  // Set GITHUB_APP_ID, GITHUB_PRIVATE_KEY, GITHUB_WEBHOOK_SECRET
  // (or route via Vercel Connect — recommended for production)
});
```

---

## Linear

```typescript
// agent/channels/linear.ts
import { defineLinearChannel } from 'eve/channels/linear';

export default defineLinearChannel({
  // Responds to Linear issue and comment events
  // Set LINEAR_WEBHOOK_SIGNING_SECRET (or route via Vercel Connect)
});
```

---

## Web Chat (Next.js)

```typescript
// agent/channels/web.ts
import { defineWebChannel } from 'eve/channels/web';
export default defineWebChannel({});
```

**Important**: Only scaffold web chat with `npx eve@latest init <name> --channel-web-nextjs`
at init time. This adds Next.js scaffolding. Do not add this flag unless the user explicitly
wants a browser-based chat UI — it is unnecessary for API, Slack, and Discord use cases.

---

## Custom Channels

```typescript
// agent/channels/my-system.ts
import { defineChannel } from 'eve/channels';

export default defineChannel({
  name: 'my-system',
  async receive(message, context) {
    // Parse incoming event from your system
    return { content: message.body.text, userId: message.body.userId };
  },
  async send(reply, context) {
    // Deliver the agent's reply back to your system
    await myApi.postMessage(context.userId, reply.content);
  },
});
```

---

## Cross-Channel Handoff

A session can start on one channel and continue on another.

```typescript
// agent/channels/incident-webhook.ts — arrives over HTTP, then opens a Slack thread
import { defineChannel } from 'eve/channels';
import slack from './slack.js';

export default defineChannel({
  name: 'incident-webhook',
  async receive(event) {
    return { content: `Incident: ${event.body.description}` };
  },
  async onComplete(result, context) {
    // Post findings to Slack after the agent finishes its investigation
    await context.send(slack, {
      content: result.content,
      target: { channelId: 'C0INCIDENTS' },
    });
  },
});
```

---

## Credential Routing Summary

| Channel | Local dev | Vercel production |
|---|---|---|
| Slack | `SLACK_BOT_TOKEN` env var | Vercel Connect (OAuth, auto-refresh) |
| Discord | `DISCORD_BOT_TOKEN` env var | Vercel Connect or env var |
| Teams | `TEAMS_APP_ID` + `TEAMS_APP_PASSWORD` | Vercel Connect or env var |
| Telegram | `TELEGRAM_BOT_TOKEN` env var | Vercel Connect or env var |
| GitHub | GitHub App credentials | Vercel Connect (recommended) |
| Linear | `LINEAR_WEBHOOK_SIGNING_SECRET` | Vercel Connect (recommended) |

Use **Vercel Connect** for production to avoid managing bot tokens in environment variables
and to get automatic token refresh without code changes.