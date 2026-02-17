# Resource Allocation Agent

A Cloudflare Worker that manages shared company resources—equipment, software licenses, parking spots—so employees can request and return items via chat. The agent tracks who has what, maintains waitlists with automatic assignment when items are returned, sends return reminders, and exposes utilization metrics for the business.

**Live demo:** `https://resource-allocation-agent.ayekudemilade43.workers.dev`

---

## Highlights

- **Stateful agent** on [Cloudflare Durable Objects](https://developers.cloudflare.com/durable-objects/) using the [Agents SDK](https://developers.cloudflare.com/agents/) — request/return, waitlists, scheduled return reminders, and in-app notifications.
- **Chat interface** — employees use simple commands (`request P1`, `return P1`, `list resources`, `what do I have`, `utilization`); no AI/LLM required.
- **Utilization API** — `GET /api/utilization` returns JSON metrics (by resource, by date) for dashboards or reporting.
- **Optional MCP** — when `SLACK_MCP_URL` or `EMAIL_MCP_URL` are set, the agent notifies users via Slack or Email (waitlist position, resource available, return reminders).

Built with **TypeScript**, **Wrangler**, and **Cloudflare Workers**. No database beyond the agent’s built-in SQLite-backed state.

---

## Quick start

```bash
npm install
npm run dev
```

Optional: copy `.dev.vars.example` to `.dev.vars` and add any secrets (e.g. MCP server URLs). Deploy with:

```bash
npm run deploy
```

---

## Routes

| Route | Description |
|-------|-------------|
| `/` | Static landing page (from `public/`). |
| `/message` | Legacy demo response. |
| `GET /api/utilization` | Read-only utilization metrics. Query params: `resourceId`, `dateFrom`, `dateTo`. |
| `/agents/resource-allocation-agent/:name` | Agent WebSocket + HTTP (e.g. `default`, `office-1`). |

---

## Agent API (callable methods)

Connect to the agent via the [Agents SDK](https://developers.cloudflare.com/agents/) (e.g. `agent.stub.requestResource(...)`) or use `handleChat(userId, message)` for command-style interaction.

| Method | Description |
|--------|-------------|
| `requestResource(resourceId, userId, dueReturnAt?)` | Assign resource if available; otherwise add user to waitlist. |
| `returnResource(resourceId, userId)` | Mark returned; next on waitlist is auto-assigned. |
| `listMyAssignments(userId)` | Active assignments for the user. |
| `listResources(type?)` | All resources with current availability (`available` count). |
| `getUtilization(resourceId?, dateFrom?, dateTo?)` | Utilization metrics (by resource, date). |
| `handleChat(userId, message)` | Parse chat and run the right action (see commands below). |
| `clearNotifications(userId)` | Clear pending reminder messages for the user. |

### Chat commands (via `handleChat`)

- `request P1` / `request resource P1` — request resource by id  
- `return P1` — return resource  
- `list resources` / `resources` — list resources and availability  
- `what do I have` / `my assignments` — list your assignments  
- `utilization` — show utilization metrics  

---

## Return reminders

A daily cron job runs at 09:00 UTC, finds assignments with `dueReturnAt` in the next 24 hours, and:

- Pushes reminder messages into `state.notifications[userId]` (clients can show these and call `clearNotifications(userId)` when done).
- If MCP is configured, also sends reminders via Slack or Email.

---

## MCP (optional notifications)

With **Model Context Protocol** servers configured, the agent sends notifications for:

- **Waitlist** — when a user is added (“You’re #N for …”).
- **Auto-assign** — when a resource becomes available for the next person on the waitlist.
- **Reminders** — return-due reminders (in addition to in-app notifications).

**Setup:** Set `SLACK_MCP_URL` and/or `EMAIL_MCP_URL` (env vars or Wrangler secrets). The agent registers them in `onStart()`; failed connections are logged and do not affect core behavior.

Tool names used: **Slack** `send_message` (`channel`, `message`), **Email** `send_email` (`to`, `body`). Adjust [src/ResourceAllocationAgent.ts](src/ResourceAllocationAgent.ts) if your MCP server uses different tools. See [Cloudflare Agents MCP docs](https://developers.cloudflare.com/agents/api-reference/mcp-agent-api/).

---

## Project structure

```
├── src/
│   ├── index.ts                 # Worker entry: routes, /api/utilization, agent routing
│   └── ResourceAllocationAgent.ts # Agent: state, callables, MCP, reminders
├── public/
│   └── index.html               # Landing page
├── test/
│   └── index.spec.ts            # Vitest tests (message, random, utilization)
├── wrangler.jsonc               # Worker name, DO binding, migrations, assets
└── worker-configuration.d.ts     # Env types (ResourceAllocationAgent, ASSETS, MCP URLs)
```

---

## Notes

- **Local tests:** `npm test` runs Vitest with the Cloudflare Workers pool. On some Windows environments the Miniflare runtime may fail to start; deployment and production behavior are unchanged.
- **Secrets:** Use `.dev.vars` locally and `wrangler secret put` in production for `SLACK_MCP_URL` / `EMAIL_MCP_URL` if you use MCP.
