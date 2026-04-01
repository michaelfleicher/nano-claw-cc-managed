# Phase 3: Calendar & Automation - Research

**Researched:** 2026-04-01
**Domain:** Google Calendar API integration via MCP server + NanoClaw task scheduler
**Confidence:** HIGH

## Summary

Phase 3 adds two capabilities to the NanoClaw WhatsApp assistant: (1) Google Calendar integration so the agent can check availability and create meetings, and (2) recurring task automation using NanoClaw's built-in scheduler. Both capabilities already have clear extension points in the NanoClaw architecture -- a new MCP server for Calendar, and the existing `schedule_task` MCP tool for automation.

The Google Calendar integration requires building a standalone MCP server (TypeScript, stdio transport) that wraps the `googleapis` library. This server runs inside the agent container alongside the existing `nanoclaw` MCP server. Authentication uses a Google Service Account with the consultant's calendar shared to the service account email -- this avoids domain-wide delegation (which requires Google Workspace admin) and eliminates token expiry issues entirely.

The recurring task automation (AUTO-01) requires no new code. NanoClaw already provides `schedule_task`, `list_tasks`, `pause_task`, `resume_task`, `cancel_task`, and `update_task` MCP tools. The agent can create cron-based, interval-based, or one-time tasks. The only work needed is prompt engineering to teach the agent how to use these tools effectively and testing to verify tasks fire and send WhatsApp messages on schedule.

**Primary recommendation:** Build a `gcal` MCP server following NanoClaw's existing MCP server pattern (McpServer + StdioServerTransport from `@modelcontextprotocol/sdk`), authenticate via Service Account + calendar sharing, and verify AUTO-01 works out of the box with NanoClaw's built-in scheduler.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| SCHED-01 | Assistant can check Google Calendar availability when a client asks about meeting times | Google Calendar FreeBusy API via `gcal` MCP server `check_availability` tool; returns busy/free time blocks for a date range |
| SCHED-02 | Assistant can create Google Calendar events with client name, time, topic, and meeting link | Google Calendar Events.insert API via `gcal` MCP server `create_event` tool; supports conferenceData for Google Meet link generation |
| SCHED-03 | Google Calendar integration uses Service Account auth (no token expiry issues) | Service Account with JSON key file + calendar sharing; no refresh token rotation needed; credentials mounted at `/workspace/secrets/` inside container |
| AUTO-01 | User can instruct the assistant to send recurring messages via NanoClaw's built-in task scheduler | Already built into NanoClaw: `schedule_task` MCP tool supports cron, interval, and one-time schedules; task scheduler polls every 60s and spawns containers to execute |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

- **Tech stack**: NanoClaw (TypeScript/Node.js) -- fork and customize, do not build from scratch
- **Calendar**: Google Calendar API via `googleapis` package (official client)
- **Auth**: Service Account auth for Google Calendar (locked decision from STATE.md)
- **Infrastructure**: Minimal AWS EC2 instance
- **AI model**: Claude via Anthropic API
- **Do not use**: Full `import { google } from 'googleapis'` (tree-shake with `google.calendar('v3')`)
- **Do not use**: Express/Fastify web server, ngrok/webhook tunnels
- **Anti-pattern**: Do not modify NanoClaw core files directly -- use extension points (MCP servers, CLAUDE.md, config overrides)
- **Anti-pattern**: Do not store Google credentials in CLAUDE.md files (mount as separate secrets volume)

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| googleapis | 171.4.0 | Google Calendar API client | Official Google API client for Node.js. Use `google.calendar('v3')` submodule specifically. |
| google-auth-library | 10.6.2 | Service Account authentication | Handles JWT-based service account auth, no browser flow needed. |
| @modelcontextprotocol/sdk | ^1.12.1 | MCP server framework | Same version NanoClaw's agent-runner uses. Provides McpServer + StdioServerTransport. |
| zod | ^4.0.0 | Tool schema validation | Same version NanoClaw's agent-runner uses for MCP tool schemas. |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| cron-parser | 5.5.0 | Parse cron expressions | Already in NanoClaw. Used by task scheduler for recurring tasks. No additional install needed. |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Service Account | OAuth2 refresh token | Only if consultant's Google account does not support calendar sharing with service accounts. OAuth2 requires periodic re-authorization and a browser-based consent flow. |
| Custom gcal MCP server | Agent using Bash + curl | Fragile, no type safety, credentials leak into agent context. MCP server keeps auth isolated. |
| googleapis SDK | Direct REST calls | No value. SDK handles pagination, retries, token refresh. Hand-rolling REST misses edge cases. |

**Installation (inside container/agent-runner):**
```bash
npm install googleapis@171 google-auth-library@10
```

Note: `@modelcontextprotocol/sdk` and `zod` are already dependencies of the agent-runner.

## Architecture Patterns

### Recommended Project Structure
```
container/agent-runner/
  src/
    ipc-mcp-stdio.ts          # Existing NanoClaw MCP server
    gcal-mcp-server.ts         # NEW: Google Calendar MCP server (standalone)
    index.ts                   # Agent runner (add gcal to mcpServers config)
groups/global/
  CLAUDE.md                    # Add calendar usage instructions for agent
secrets/
  google-service-account.json  # Service account key (mounted into container)
```

### Pattern 1: Google Calendar as Container-Mounted MCP Server

**What:** A standalone TypeScript MCP server that wraps Google Calendar API, runs inside agent containers via stdio transport, registered alongside the existing nanoclaw MCP server.

**When to use:** This is the only pattern for Phase 3.

**How it connects:**
```typescript
// In container/agent-runner/src/index.ts, add to mcpServers in query() call:
mcpServers: {
  nanoclaw: { /* existing */ },
  gcal: {
    command: 'node',
    args: [gcalMcpServerPath],
    env: {
      GOOGLE_CREDENTIALS_PATH: '/workspace/secrets/google-service-account.json',
      GOOGLE_CALENDAR_ID: process.env.GOOGLE_CALENDAR_ID || 'primary',
      TZ: process.env.TZ,
    },
  },
}
```

The agent then calls tools like `mcp__gcal__check_availability` and `mcp__gcal__create_event` naturally.

### Pattern 2: Service Account Auth with Calendar Sharing

**What:** Instead of domain-wide delegation (requires Google Workspace admin), share the consultant's calendar with the service account's email address. The service account then has direct access to that specific calendar.

**Setup steps:**
1. Create a Google Cloud project
2. Enable Google Calendar API
3. Create a Service Account and download the JSON key
4. In Google Calendar settings, share the consultant's calendar with the service account email (e.g., `nanoclaw@project-id.iam.gserviceaccount.com`) with "Make changes to events" permission
5. Mount the JSON key file into the container at `/workspace/secrets/`

**Why this over domain-wide delegation:**
- Works with personal Google accounts (not just Workspace)
- No admin console access required
- Simpler setup, fewer moving parts
- Service account keys do not expire (unlike OAuth2 refresh tokens)

**Auth code pattern:**
```typescript
import { google } from 'googleapis';
import { GoogleAuth } from 'google-auth-library';

const auth = new GoogleAuth({
  keyFile: process.env.GOOGLE_CREDENTIALS_PATH,
  scopes: ['https://www.googleapis.com/auth/calendar'],
});

const calendar = google.calendar({ version: 'v3', auth });
```

### Pattern 3: MCP Server Tool Design (following NanoClaw's existing pattern)

**What:** Build the gcal MCP server using the exact same pattern as NanoClaw's `ipc-mcp-stdio.ts`.

**Server skeleton:**
```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { z } from 'zod';
import { google } from 'googleapis';
import { GoogleAuth } from 'google-auth-library';

const server = new McpServer({
  name: 'gcal',
  version: '1.0.0',
});

// Auth setup
const auth = new GoogleAuth({
  keyFile: process.env.GOOGLE_CREDENTIALS_PATH,
  scopes: ['https://www.googleapis.com/auth/calendar'],
});
const calendar = google.calendar({ version: 'v3', auth });
const calendarId = process.env.GOOGLE_CALENDAR_ID || 'primary';

// Tools registered here (see Code Examples section)

const transport = new StdioServerTransport();
await server.connect(transport);
```

### Pattern 4: Recurring Tasks via Built-in Scheduler (AUTO-01)

**What:** NanoClaw already has a complete task scheduling system. The agent calls `mcp__nanoclaw__schedule_task` with a cron expression, prompt, and target group. The host-side task scheduler polls every 60 seconds, spawns a container when a task is due, and the agent sends the message.

**No custom code needed.** The work for AUTO-01 is:
1. Verify the scheduler works end-to-end (create task, wait for it to fire, confirm WhatsApp message sent)
2. Add instructions to the global CLAUDE.md teaching the agent how/when to use scheduling tools
3. Test cron expression parsing for common patterns (daily, weekly, monthly)

**Existing MCP tools for tasks:**
- `schedule_task` -- create new (cron/interval/once)
- `list_tasks` -- view existing
- `pause_task` / `resume_task` -- toggle
- `cancel_task` -- delete
- `update_task` -- modify

### Anti-Patterns to Avoid

- **Storing credentials in CLAUDE.md:** Agent context files are visible in Claude's context window. Credentials could leak into responses. Mount secrets as a separate volume accessible only by the MCP server process.
- **Using OAuth2 user flow on headless EC2:** Requires browser redirect, breaks on headless. Use Service Account instead.
- **Modifying container-runner.ts core logic:** Add the gcal MCP server registration as a minimal, isolated change. Do not restructure the container runner.
- **Building a custom scheduler:** NanoClaw's task scheduler handles cron, interval, and one-time tasks with drift prevention. Do not rebuild this.
- **Importing full googleapis SDK:** `import { google } from 'googleapis'` pulls the entire SDK into memory. Use targeted imports.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Calendar availability checking | Custom REST calls to Google Calendar | `googleapis` `freebusy.query()` | Handles auth, pagination, error codes, timezone conversion |
| Event creation with Meet link | Manual HTTP POST with conferenceData | `googleapis` `events.insert()` with `conferenceDataVersion: 1` | Meet link creation is async; SDK handles pending/success state |
| Cron scheduling | Custom cron parser/executor | NanoClaw's built-in `schedule_task` + `cron-parser` | Already integrated with container spawning, IPC, and WhatsApp delivery |
| MCP server framework | Custom stdio JSON-RPC handler | `@modelcontextprotocol/sdk` McpServer + StdioServerTransport | Protocol compliance, tool discovery, schema validation |
| Token refresh for Service Account | Manual JWT signing and token management | `google-auth-library` GoogleAuth | Handles JWT creation, token caching, automatic refresh |
| Timezone handling for calendar | Manual UTC offset calculations | Pass `timeZone` parameter to all Calendar API calls; use `TZ` env var | Google Calendar API handles DST transitions correctly when timezone is specified |

## Common Pitfalls

### Pitfall 1: Service Account Cannot See Calendar Events
**What goes wrong:** Service account is created but consultant's calendar is not shared with it. All API calls return empty results or 404.
**Why it happens:** Service accounts are separate identities. Without explicit calendar sharing, they have no access to any user's calendar.
**How to avoid:** After creating the service account, go to Google Calendar > Settings > Share with specific people > Add the service account email with "Make changes to events" permission. Test with a simple `events.list()` call before building the full integration.
**Warning signs:** `freebusy.query()` returns empty busy arrays even when calendar has events; `events.list()` returns 0 items.

### Pitfall 2: Google Cloud Project in "Testing" Mode
**What goes wrong:** If using OAuth2 (fallback), refresh tokens expire after 7 days because the project is in "testing" mode on the OAuth consent screen.
**How to avoid:** This pitfall is avoided entirely by using Service Account auth (SCHED-03). If OAuth2 fallback is ever needed, set the Google Cloud project to "In production" (even for single-user internal use) to get non-expiring refresh tokens.

### Pitfall 3: Timezone Mismatch in Scheduling
**What goes wrong:** Agent creates a meeting at "2pm" but uses UTC instead of the consultant's timezone. Client sees the meeting at the wrong time.
**Why it happens:** Google Calendar API defaults to UTC if no timezone is specified. The agent may not know the consultant's timezone.
**How to avoid:** Always pass `timeZone` parameter in Calendar API calls. Set `TZ` environment variable in the container. Include the consultant's timezone in the global CLAUDE.md knowledge brief. The cron-parser in NanoClaw's scheduler also uses the configured TZ.
**Warning signs:** Events appear at unexpected times; `next_run` values in scheduled_tasks table are offset by hours.

### Pitfall 4: Meeting Conflicts (Double-Booking)
**What goes wrong:** Agent creates a meeting without checking availability first, resulting in overlapping events.
**Why it happens:** Agent calls `create_event` directly without first calling `check_availability`.
**How to avoid:** Instruct the agent (via CLAUDE.md) to always check availability before creating events. The `check_availability` tool should be called first, and the agent should present available slots to the client before booking.

### Pitfall 5: Container Secrets Mount Not Configured
**What goes wrong:** The gcal MCP server starts but cannot find the credentials file. Calendar tools fail with auth errors.
**Why it happens:** The service account JSON key is on the host but not volume-mounted into the container.
**How to avoid:** Add a volume mount in container-runner.ts: `secrets/` directory on host mapped to `/workspace/secrets/` in container (read-only). Verify the mount exists before the MCP server attempts auth.

### Pitfall 6: Scheduled Tasks Fire But Agent Has No Context
**What goes wrong:** A recurring task fires (e.g., "weekly reminder") but the agent container has no knowledge brief or conversation history, so it sends a generic or incorrect message.
**Why it happens:** Tasks with `context_mode: 'isolated'` start fresh containers with no group history. Tasks with `context_mode: 'group'` include recent conversation history.
**How to avoid:** For recurring reminders, use `context_mode: 'group'` so the agent has the group's CLAUDE.md (knowledge brief). The task's prompt should be specific enough that the agent knows exactly what to send.

## Code Examples

### Check Calendar Availability (MCP Tool)
```typescript
// Source: Google Calendar API freebusy.query() docs
server.tool(
  'check_availability',
  'Check calendar availability for a date range. Returns busy time blocks.',
  {
    date_start: z.string().describe('Start date/time in ISO 8601 format, e.g., 2026-04-02T09:00:00'),
    date_end: z.string().describe('End date/time in ISO 8601 format, e.g., 2026-04-02T18:00:00'),
    timezone: z.string().optional().describe('IANA timezone, e.g., Asia/Jerusalem. Defaults to server TZ.'),
  },
  async ({ date_start, date_end, timezone }) => {
    const tz = timezone || process.env.TZ || 'UTC';
    const response = await calendar.freebusy.query({
      requestBody: {
        timeMin: date_start,
        timeMax: date_end,
        timeZone: tz,
        items: [{ id: calendarId }],
      },
    });
    const busy = response.data.calendars?.[calendarId]?.busy || [];
    return {
      content: [{
        type: 'text',
        text: busy.length === 0
          ? `Calendar is free from ${date_start} to ${date_end} (${tz})`
          : `Busy times:\n${busy.map(b => `- ${b.start} to ${b.end}`).join('\n')}`,
      }],
    };
  },
);
```

### Create Calendar Event (MCP Tool)
```typescript
// Source: Google Calendar API events.insert() docs
server.tool(
  'create_event',
  'Create a Google Calendar event with optional Google Meet link.',
  {
    summary: z.string().describe('Event title, e.g., "Meeting with John - AI Strategy"'),
    start_time: z.string().describe('Start time in ISO 8601, e.g., 2026-04-03T14:00:00'),
    end_time: z.string().describe('End time in ISO 8601, e.g., 2026-04-03T15:00:00'),
    timezone: z.string().optional().describe('IANA timezone, e.g., Asia/Jerusalem'),
    description: z.string().optional().describe('Event description or meeting notes'),
    location: z.string().optional().describe('Meeting location (physical address or URL)'),
    add_meet_link: z.boolean().optional().describe('Whether to add a Google Meet link. Default: true'),
    attendees: z.array(z.string()).optional().describe('Email addresses of attendees'),
  },
  async ({ summary, start_time, end_time, timezone, description, location, add_meet_link, attendees }) => {
    const tz = timezone || process.env.TZ || 'UTC';
    const eventBody: any = {
      summary,
      start: { dateTime: start_time, timeZone: tz },
      end: { dateTime: end_time, timeZone: tz },
    };
    if (description) eventBody.description = description;
    if (location) eventBody.location = location;
    if (attendees) eventBody.attendees = attendees.map(email => ({ email }));
    if (add_meet_link !== false) {
      eventBody.conferenceData = {
        createRequest: {
          requestId: `nanoclaw-${Date.now()}`,
          conferenceSolutionKey: { type: 'hangoutsMeet' },
        },
      };
    }

    const response = await calendar.events.insert({
      calendarId,
      requestBody: eventBody,
      conferenceDataVersion: add_meet_link !== false ? 1 : 0,
      sendUpdates: attendees ? 'all' : 'none',
    });

    const event = response.data;
    const meetLink = event.conferenceData?.entryPoints?.find(
      (e: any) => e.entryPointType === 'video'
    )?.uri;

    return {
      content: [{
        type: 'text',
        text: [
          `Event created: ${event.summary}`,
          `Time: ${event.start?.dateTime} - ${event.end?.dateTime}`,
          meetLink ? `Meet link: ${meetLink}` : '',
          `Calendar link: ${event.htmlLink}`,
        ].filter(Boolean).join('\n'),
      }],
    };
  },
);
```

### List Upcoming Events (MCP Tool)
```typescript
server.tool(
  'list_events',
  'List upcoming calendar events for a date range.',
  {
    date_start: z.string().optional().describe('Start date in ISO 8601. Defaults to now.'),
    date_end: z.string().optional().describe('End date in ISO 8601. Defaults to 7 days from now.'),
    max_results: z.number().optional().describe('Maximum events to return. Default: 10'),
  },
  async ({ date_start, date_end, max_results }) => {
    const now = new Date();
    const response = await calendar.events.list({
      calendarId,
      timeMin: date_start || now.toISOString(),
      timeMax: date_end || new Date(now.getTime() + 7 * 24 * 60 * 60 * 1000).toISOString(),
      maxResults: max_results || 10,
      singleEvents: true,
      orderBy: 'startTime',
    });
    const events = response.data.items || [];
    if (events.length === 0) {
      return { content: [{ type: 'text', text: 'No upcoming events found.' }] };
    }
    const formatted = events.map(e =>
      `- ${e.start?.dateTime || e.start?.date}: ${e.summary}${e.location ? ` (${e.location})` : ''}`
    ).join('\n');
    return { content: [{ type: 'text', text: `Upcoming events:\n${formatted}` }] };
  },
);
```

### Container Volume Mount for Secrets
```typescript
// In container-runner.ts buildVolumeMounts(), add:
// Host path: <project-root>/secrets/google-service-account.json
// Container path: /workspace/secrets/google-service-account.json
// Mode: read-only (ro)
args.push('-v', `${secretsPath}:/workspace/secrets:ro`);
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| OAuth2 user flow with refresh tokens | Service Account + calendar sharing | Always available, but recommended for server-to-server | No token expiry, no browser needed, simpler setup |
| `@google-cloud/local-auth` for desktop OAuth | `google-auth-library` GoogleAuth with keyFile | N/A (different auth flows) | Service accounts use GoogleAuth directly, no local-auth needed |
| MCP SDK 0.x | MCP SDK ^1.12.1 | NanoClaw uses 1.12.1+ | Zod 4 support, improved tool registration API |
| Manual cron implementation | NanoClaw built-in `schedule_task` | Built into NanoClaw | Drift prevention, SQLite persistence, container-based execution |

## Open Questions

1. **Google Account Type**
   - What we know: Service Account auth is the locked decision (STATE.md)
   - What's unclear: Whether the consultant uses a personal Google account or Google Workspace. Personal accounts can share calendars with service accounts, but some features (domain-wide delegation) require Workspace.
   - Recommendation: Use calendar sharing (works with both account types). Only consider domain-wide delegation if calendar sharing fails for some reason.

2. **Calendar ID**
   - What we know: The consultant's primary calendar is the target
   - What's unclear: Whether the service account should use 'primary' (which refers to the service account's own calendar) or the consultant's email as calendar ID
   - Recommendation: Use the consultant's email address as the calendar ID (e.g., `consultant@gmail.com`). The 'primary' alias refers to the authenticated user's calendar, which for a service account would be the service account's own (empty) calendar.

3. **Google Meet Link Generation**
   - What we know: Events.insert supports `conferenceData.createRequest` for Meet links
   - What's unclear: Whether service accounts can create Meet links (some sources indicate this requires a Google Workspace license on the calendar owner)
   - Recommendation: Implement with Meet link support, but make it optional (`add_meet_link` parameter). If Meet creation fails for the account type, degrade gracefully to event-only creation.

## Environment Availability

> This section covers dependencies needed for the gcal MCP server development.

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All | Check on EC2 at deploy time | 22 LTS required | -- |
| googleapis npm package | Calendar API calls | Install in agent-runner | 171.4.0 (latest) | -- |
| google-auth-library npm package | Service Account auth | Install in agent-runner | 10.6.2 (latest) | -- |
| @modelcontextprotocol/sdk | MCP server framework | Already in agent-runner | ^1.12.1 | -- |
| Google Cloud Console access | Service Account creation | Requires consultant to set up | -- | Consultant creates SA and shares key file |
| Docker | Container runtime | Required by Phase 1 | 24+ | -- |

**Missing dependencies with no fallback:**
- Google Cloud project and Service Account must be created manually (one-time setup, cannot be automated)

**Missing dependencies with fallback:**
- None

## Sources

### Primary (HIGH confidence)
- NanoClaw source code: `container/agent-runner/src/ipc-mcp-stdio.ts` -- MCP server pattern, tool registration, Zod schemas
- NanoClaw source code: `container/agent-runner/src/index.ts` -- mcpServers configuration, how additional MCP servers are registered
- NanoClaw source code: `src/task-scheduler.ts` -- task execution flow, cron parsing, drift prevention
- NanoClaw source code: `src/db.ts` -- scheduled_tasks table schema
- NanoClaw source code: `container/agent-runner/package.json` -- dependency versions (@modelcontextprotocol/sdk ^1.12.1, zod ^4.0.0)
- Google Calendar API official docs: events.insert(), freebusy.query(), OAuth scopes
- Google Service Account docs: JWT auth, calendar sharing vs domain-wide delegation
- npm registry: googleapis 171.4.0, google-auth-library 10.6.2, @modelcontextprotocol/sdk 1.29.0, cron-parser 5.5.0

### Secondary (MEDIUM confidence)
- NanoClaw architecture research (`.planning/research/ARCHITECTURE.md`) -- pattern 2 documents the gcal MCP server approach
- NanoClaw pitfalls research (`.planning/research/PITFALLS.md`) -- pitfall 5 covers OAuth token expiry, pitfall 3 covers timezone issues

### Tertiary (LOW confidence)
- Google Meet link creation with service accounts -- needs validation that service accounts can create conferenceData without Workspace license

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all packages verified against npm registry, versions confirmed
- Architecture: HIGH -- based on direct NanoClaw source code analysis, follows existing MCP server pattern exactly
- Pitfalls: HIGH -- verified against prior pitfalls research and Google Calendar API documentation
- AUTO-01 (task scheduler): HIGH -- NanoClaw's built-in scheduler is well-documented in source and already has MCP tools

**Research date:** 2026-04-01
**Valid until:** 2026-05-01 (stable domain, no fast-moving dependencies)
