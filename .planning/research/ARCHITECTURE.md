# Architecture Patterns

**Domain:** NanoClaw-based WhatsApp AI assistant for consultants
**Researched:** 2026-04-01
**Confidence:** HIGH (based on direct source code analysis of NanoClaw upstream)

## System Overview

NanoClaw is a single Node.js process that orchestrates AI agent execution in isolated Docker containers. The architecture is already defined by the framework -- this document maps how the consultant's assistant fits into that architecture on EC2, identifies the components that need customization, and establishes the build order.

```
┌─────────────────────────────── EC2 (t3.micro / t3.small) ────────────────────────────────┐
│                                                                                           │
│  ┌─────────────────────────── NanoClaw Host Process (Node.js) ──────────────────────────┐ │
│  │                                                                                       │ │
│  │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐│ │
│  │  │  WhatsApp     │    │  Message      │    │  Task         │    │  IPC Watcher        ││ │
│  │  │  Channel      │    │  Loop         │    │  Scheduler    │    │  (file-based)       ││ │
│  │  │  (Baileys)    │    │  (2s poll)    │    │  (60s poll)   │    │  (1s poll)          ││ │
│  │  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘    └──────────────────────┘│ │
│  │         │                   │                   │                                     │ │
│  │         ▼                   ▼                   ▼                                     │ │
│  │  ┌──────────────────────────────────────────────────────────────────────────────────┐ │ │
│  │  │                          SQLite (messages.db)                                     │ │ │
│  │  │  Tables: messages, chats, registered_groups, sessions, scheduled_tasks,           │ │ │
│  │  │          task_run_logs, router_state                                               │ │ │
│  │  └──────────────────────────────────────────────────────────────────────────────────┘ │ │
│  │                                     │                                                 │ │
│  │                          ┌──────────┴──────────┐                                     │ │
│  │                          │    Group Queue       │                                     │ │
│  │                          │  (FIFO per group,    │                                     │ │
│  │                          │   global concurrency │                                     │ │
│  │                          │   limit: 5)          │                                     │ │
│  │                          └──────────┬──────────┘                                     │ │
│  └─────────────────────────────────────┼───────────────────────────────────────────────┘ │
│                                        │ spawns                                          │
│                                        ▼                                                 │
│  ┌──────────────────────── Docker Container (nanoclaw-agent) ──────────────────────────┐ │
│  │                                                                                      │ │
│  │  Agent Runner (Claude Agent SDK)                                                     │ │
│  │  ├── Claude Code CLI (claude-code npm package)                                      │ │
│  │  ├── MCP Server: nanoclaw (send_message, schedule_task, etc.)                       │ │
│  │  ├── Tools: Bash, Read, Write, Edit, WebSearch, WebFetch, agent-browser             │ │
│  │  └── Chromium (for web automation)                                                   │ │
│  │                                                                                      │ │
│  │  Volume Mounts:                                                                      │ │
│  │  ├── groups/{folder}/ --> /workspace/group  (read-write, agent working dir)          │ │
│  │  ├── groups/global/   --> /workspace/global (read-only, shared memory)               │ │
│  │  ├── data/sessions/{folder}/.claude/ --> /home/node/.claude/ (sessions)              │ │
│  │  └── data/ipc/{folder}/ --> /workspace/ipc  (IPC communication)                     │ │
│  │                                                                                      │ │
│  └──────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                           │
│  ┌─────────────────────────── External Services ──────────────────────────────────────┐  │
│  │  Anthropic API (Claude) ──── via OneCLI credential proxy or direct API key          │  │
│  │  Google Calendar API ─────── OAuth2, accessed by agent via tools inside container   │  │
│  │  WhatsApp (Baileys) ──────── WebSocket to WhatsApp servers, auth stored in store/   │  │
│  └────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                           │
└───────────────────────────────────────────────────────────────────────────────────────────┘
```

## Component Boundaries

### 1. Host Process Components (src/)

| Component | File | Responsibility | Communicates With |
|-----------|------|---------------|-------------------|
| **Orchestrator** | `src/index.ts` | Main event loop, state management, startup/shutdown | All host components |
| **WhatsApp Channel** | `src/channels/whatsapp.ts` | Baileys WebSocket connection, send/receive messages, QR/pairing auth | Orchestrator (via callbacks) |
| **Channel Registry** | `src/channels/registry.ts` | Self-registration pattern for channels at startup | Orchestrator |
| **Message Loop** | `src/index.ts` (inline) | Polls SQLite every 2s for new messages in registered groups | SQLite, Group Queue |
| **Group Queue** | `src/group-queue.ts` | Per-group FIFO queue with global concurrency limit (default: 5 containers) | Container Runner |
| **Container Runner** | `src/container-runner.ts` | Spawns Docker containers, builds volume mounts, handles streaming output | Docker, Agent Runner |
| **IPC Watcher** | `src/ipc.ts` | Polls filesystem IPC directories for messages and task commands from containers | File system, SQLite |
| **Task Scheduler** | `src/task-scheduler.ts` | Polls SQLite every 60s for due tasks, enqueues them via Group Queue | SQLite, Group Queue |
| **Router** | `src/router.ts` | Finds owning channel for a JID, formats messages XML | Channels |
| **Database** | `src/db.ts` | SQLite schema, CRUD operations, migrations | better-sqlite3 |
| **Config** | `src/config.ts` | Environment variables, paths, trigger patterns | All components |

### 2. Container Components (container/)

| Component | File | Responsibility | Communicates With |
|-----------|------|---------------|-------------------|
| **Agent Runner** | `container/agent-runner/src/index.ts` | Receives prompt via stdin, runs Claude Agent SDK query loop, polls IPC for follow-up messages | Claude API, MCP Server |
| **NanoClaw MCP Server** | `container/agent-runner/src/ipc-mcp-stdio.ts` | Exposes tools (send_message, schedule_task, list_tasks, etc.) to the agent via MCP protocol, writes IPC files | Agent Runner, Host IPC Watcher |
| **Container Image** | `container/Dockerfile` | Node 22 + Chromium + claude-code CLI + agent-browser | Docker |

### 3. Data Layer

| Store | Location | Purpose | Access Pattern |
|-------|----------|---------|----------------|
| **SQLite** | `store/messages.db` | Messages, groups, sessions, tasks, router state | Host process only (single writer) |
| **WhatsApp Auth** | `store/auth/` | Baileys credentials (creds.json) | WhatsApp channel reads at connect |
| **Group Memory** | `groups/{folder}/CLAUDE.md` | Per-group knowledge, personality, preferences | Container reads/writes |
| **Global Memory** | `groups/global/CLAUDE.md` | Shared knowledge across all groups | Main writes, others read-only |
| **Session Transcripts** | `data/sessions/{folder}/.claude/` | Conversation history (JSONL) for session resume | Container reads/writes |
| **IPC Files** | `data/ipc/{folder}/` | Messages and task commands between container and host | Container writes, host reads |
| **Knowledge Brief** | `groups/{folder}/CLAUDE.md` | Consultant's services, pricing, communication style | Container reads |

### 4. External Service Integrations

| Service | Protocol | Auth Method | Where Used |
|---------|----------|-------------|------------|
| **Anthropic API** | HTTPS | API key or OAuth token (via OneCLI proxy or direct) | Inside container, by Claude Agent SDK |
| **WhatsApp** | WebSocket (Baileys) | QR code or pairing code, stored in store/auth/ | Host process, WhatsApp channel |
| **Google Calendar** | HTTPS (REST API) | OAuth2 (service account or user consent) | Inside container, agent uses it as a tool |

## Data Flow

### Inbound Message Flow (WhatsApp to Agent Response)

```
1. Client sends WhatsApp message
   │
   ▼
2. Baileys WebSocket receives message
   │
   ▼
3. WhatsApp Channel calls onMessage callback
   │ - Sender allowlist check (drop if denied)
   │ - storeMessage() writes to SQLite
   ▼
4. Message Loop (2s poll) detects new message
   │ - Is chat_jid registered? No --> ignore
   │ - Is trigger present? (e.g., "@Assistant") No --> store but don't process
   │ - Fetch all messages since last agent reply (catch-up context)
   ▼
5. Try to pipe to active container (queue.sendMessage via IPC file)
   │ - If active container exists --> write IPC file, done
   │ - If no active container --> enqueue in Group Queue
   ▼
6. Group Queue schedules container (respects concurrency limit)
   │
   ▼
7. Container Runner spawns Docker container
   │ - Builds volume mounts (group folder, sessions, IPC)
   │ - Passes formatted messages via stdin JSON
   │ - Registers container process with Group Queue
   ▼
8. Agent Runner inside container
   │ - Reads stdin JSON (prompt + session context)
   │ - Calls Claude Agent SDK query()
   │ - Agent reads CLAUDE.md files (knowledge brief)
   │ - Agent processes, uses tools as needed
   │ - Streams output via OUTPUT_MARKER protocol
   ▼
9. Container Runner receives streaming output
   │ - Strips <internal> tags
   │ - Calls channel.sendMessage() to reply
   │ - Tracks session ID for resume
   ▼
10. WhatsApp Channel sends reply via Baileys
    │
    ▼
11. Client receives response in WhatsApp
```

### Scheduled Task Flow

```
1. User says "@Assistant remind me every Monday at 9am to follow up with clients"
   │
   ▼
2. Agent calls mcp__nanoclaw__schedule_task via MCP
   │ - Writes JSON file to /workspace/ipc/tasks/
   ▼
3. IPC Watcher reads task file, creates row in scheduled_tasks table
   │
   ▼
4. Task Scheduler (60s poll) finds due tasks
   │ - Enqueues via Group Queue
   ▼
5. Container Runner spawns container with task prompt
   │ - Agent runs, sends result via send_message MCP tool
   │ - IPC Watcher delivers message to WhatsApp channel
   ▼
6. Task Scheduler computes next_run, updates SQLite
```

### Google Calendar Integration Flow

```
1. User: "@Assistant schedule a meeting with John next Tuesday at 2pm"
   │
   ▼
2. Agent inside container needs Calendar access
   │ - Option A: Agent uses Bash + curl with OAuth token (mounted or env var)
   │ - Option B: Custom MCP server for Google Calendar (preferred)
   │ - Option C: Agent uses WebFetch with pre-authenticated token
   ▼
3. Agent checks calendar availability via Google Calendar API
   │
   ▼
4. Agent creates event, confirms to user via send_message
```

## Recommended Architecture for This Project

### What NanoClaw Provides Out of the Box

NanoClaw handles the hard parts already. Do NOT rewrite:

- WhatsApp connectivity (Baileys-based channel, QR/pairing auth)
- Message routing, queuing, and trigger detection
- Container isolation for agent execution
- Session persistence and conversation resume
- Scheduled task infrastructure (cron, interval, once)
- File-based IPC between containers and host
- Per-group memory hierarchy (CLAUDE.md files)

### What Needs to Be Built/Customized

1. **Knowledge Brief** (groups/global/CLAUDE.md and group-specific CLAUDE.md files)
   - Consultant's services, pricing, typical answers
   - Communication style guide
   - Client interaction patterns
   - What to say and what NOT to say

2. **Google Calendar MCP Server** (new component)
   - Custom MCP server that agents can use inside containers
   - Handles OAuth2 authentication to Google Calendar API
   - Tools: check_availability, create_event, list_events, update_event, cancel_event
   - Mounted into container, registered as additional MCP server

3. **EC2 Deployment Configuration** (new)
   - systemd service file (replaces macOS launchd)
   - Docker installation and image build
   - .env configuration (Anthropic API key, Google OAuth credentials)
   - Security groups, EBS volume for persistence

4. **Sender Allowlist** (~/.config/nanoclaw/sender-allowlist.json)
   - Restrict which WhatsApp contacts can trigger the assistant
   - Prevents prompt injection from unknown numbers

5. **Container Concurrency Tuning**
   - t3.micro has 1 vCPU, 1GB RAM -- MAX_CONCURRENT_CONTAINERS should be 1-2
   - IDLE_TIMEOUT should be reduced from 30min to 5-10min to free memory

### Component Interaction Map (Customized for This Project)

```
                 WhatsApp (Baileys)
                       │
                       ▼
              ┌─────────────────┐
              │  NanoClaw Host   │
              │  (index.ts)      │
              │                  │
              │  Message Loop    │◄── polls SQLite every 2s
              │  Task Scheduler  │◄── polls SQLite every 60s
              │  IPC Watcher     │◄── polls filesystem every 1s
              │  Group Queue     │◄── concurrency: 1-2 (t3.micro)
              └────────┬────────┘
                       │ spawns
                       ▼
              ┌─────────────────┐
              │  Docker          │
              │  Container       │
              │                  │
              │  Agent Runner    │── Claude Agent SDK ── Anthropic API
              │  MCP: nanoclaw   │── send_message, schedule_task
              │  MCP: gcal       │── Google Calendar API (NEW)
              │                  │
              │  CLAUDE.md       │── Knowledge brief (consultant identity)
              └─────────────────┘
```

## Patterns to Follow

### Pattern 1: Knowledge Brief as CLAUDE.md

**What:** Store the consultant's identity, services, communication style, and guardrails as CLAUDE.md files in the group folder hierarchy.

**When:** From day one -- this is how the agent gets its personality and knowledge.

**Why:** NanoClaw's agent runner already loads CLAUDE.md files automatically via the Claude Agent SDK. Global CLAUDE.md is read by all groups, group-specific CLAUDE.md adds per-client context.

**Structure:**
```
groups/
├── global/
│   └── CLAUDE.md          # Consultant identity, services, pricing, global rules
├── whatsapp_main/
│   └── CLAUDE.md          # Main control chat rules (no trigger required)
└── whatsapp_{client}/
    └── CLAUDE.md          # Client-specific context (their project, preferences)
```

### Pattern 2: Google Calendar as Container-Mounted MCP Server

**What:** Build a standalone MCP server (TypeScript, stdio transport) that wraps the Google Calendar API. Register it alongside the existing nanoclaw MCP server in the agent runner.

**When:** Phase 2, after basic WhatsApp connectivity works.

**Why:** This follows NanoClaw's existing pattern -- the nanoclaw MCP server already runs inside containers via stdio. Adding a second MCP server is a supported configuration (agent runner's `mcpServers` config accepts multiple servers). The agent can then naturally call `mcp__gcal__check_availability` or `mcp__gcal__create_event`.

**Implementation approach:**
```typescript
// In container-runner.ts, add to mcpServers config:
mcpServers: {
  nanoclaw: { /* existing */ },
  gcal: {
    command: 'node',
    args: ['/app/gcal-mcp.js'],
    env: {
      GOOGLE_CREDENTIALS_PATH: '/workspace/secrets/google-credentials.json',
    },
  },
}
```

### Pattern 3: Systemd Service on EC2

**What:** Use systemd user service instead of macOS launchd for process management on EC2.

**When:** Phase 1, deployment.

**Why:** NanoClaw already documents the Linux systemd approach. EC2 Amazon Linux 2023 has systemd. The service should auto-restart on failure and start on boot.

```ini
[Unit]
Description=NanoClaw WhatsApp Assistant
After=network.target docker.service

[Service]
Type=simple
WorkingDirectory=/home/ec2-user/nanoclaw
ExecStart=/usr/bin/node dist/index.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production
Environment=ASSISTANT_NAME=<name>

[Install]
WantedBy=default.target
```

### Pattern 4: Reduced Resource Configuration for t3.micro

**What:** Tune NanoClaw's defaults for a constrained 1 vCPU / 1 GB RAM environment.

**When:** Phase 1, configuration.

**Why:** Default settings (5 concurrent containers, 30-min idle timeout) are designed for desktop machines. EC2 t3.micro will OOM with those defaults.

**Key overrides:**
```bash
MAX_CONCURRENT_CONTAINERS=1       # Only 1 container at a time
IDLE_TIMEOUT=300000                # 5 minutes, not 30
CONTAINER_TIMEOUT=600000           # 10 min hard limit
```

Note: t3.micro may be too small if containers include Chromium (which NanoClaw's Dockerfile installs). Consider t3.small (2 GB RAM) or building a slimmer container image without Chromium if browser automation is not needed.

## Anti-Patterns to Avoid

### Anti-Pattern 1: Modifying NanoClaw Core Files Directly

**What:** Editing index.ts, container-runner.ts, group-queue.ts, or other core files to add custom logic.

**Why bad:** Makes it impossible to pull upstream NanoClaw updates. The project is actively maintained. You will diverge permanently and miss bug fixes and new features.

**Instead:** Use NanoClaw's extension points:
- CLAUDE.md files for agent behavior
- MCP servers for new tools (like Google Calendar)
- Container skills for agent capabilities
- .env and config overrides for tuning
- Sender allowlist for access control

### Anti-Pattern 2: Running Without Container Isolation

**What:** Running agents directly on the host (no Docker) to save resources.

**Why bad:** WhatsApp messages from clients can contain prompt injection attacks. Without container isolation, a manipulated agent could access the entire EC2 instance -- SSH keys, .env files, other services.

**Instead:** Keep containers. If resources are tight, use a slimmer Docker image (remove Chromium) and t3.small instead of t3.micro.

### Anti-Pattern 3: Storing Google OAuth Tokens in CLAUDE.md

**What:** Putting API credentials in knowledge brief files that agents can read.

**Why bad:** CLAUDE.md is part of the agent's context window -- tokens could leak into Claude's responses to clients.

**Instead:** Mount credentials as a separate file into the container (/workspace/secrets/), accessible only by the MCP server process, not by the agent's read/write tools.

### Anti-Pattern 4: Full Autopilot Without Trigger

**What:** Having the assistant respond to every message without being tagged.

**Why bad:** The assistant will respond to client messages the consultant didn't intend, potentially embarrassing or wrong. For a consultant using a personal WhatsApp number, this is dangerous.

**Instead:** Stick with tag-to-reply (requiresTrigger: true). The consultant tags when they want the assistant to jump in. This is already the NanoClaw default for non-main groups.

## Scalability Considerations

| Concern | At 5 clients (current) | At 20 clients | At 100+ clients |
|---------|------------------------|---------------|-----------------|
| **Compute** | t3.micro/small, 1 container | t3.medium, 2 containers | Not this architecture -- need multi-instance or serverless |
| **Storage** | 1 GB EBS, SQLite handles fine | 5 GB EBS, SQLite still fine | PostgreSQL needed |
| **WhatsApp** | Single linked device | Single linked device (Baileys limit) | WhatsApp Business API required |
| **API costs** | ~$5-15/month Anthropic | ~$30-60/month | Need cost controls per client |
| **Memory** | SQLite + CLAUDE.md per group | Same, monitor DB size | Need archiving strategy |

For a 5-10 client consultant, t3.small with 1-2 concurrent containers is the right size. Do not over-engineer.

## Build Order (Dependencies)

The components have clear dependencies that dictate build order:

```
Phase 1: Foundation
├── Fork NanoClaw repo
├── EC2 instance setup (Docker, Node.js, systemd)
├── Build container image
├── WhatsApp channel authentication (QR/pairing code)
├── Register main chat (self-chat or solo group)
├── Configure .env (Anthropic API key, assistant name)
├── Tune for EC2 (MAX_CONCURRENT_CONTAINERS=1, reduced timeouts)
└── Verify: send message, get response

Phase 2: Knowledge & Identity
├── Write global CLAUDE.md (consultant identity, services, style)
├── Write group-specific CLAUDE.md templates
├── Configure sender allowlist
├── Register first client group (trigger-based)
└── Verify: tag in client group, get on-brand response

Phase 3: Google Calendar Integration
├── Google Cloud project + OAuth2 setup
├── Build Google Calendar MCP server (TypeScript, stdio)
├── Mount into container configuration
├── Register as additional MCP server in container-runner
└── Verify: "schedule a meeting" triggers calendar creation

Phase 4: Recurring Tasks
├── Test cron-based reminders ("follow up every Monday")
├── Test one-time tasks ("remind me at 5pm")
├── Configure task monitoring
└── Verify: scheduled tasks fire and send WhatsApp messages

Phase 5: Production Hardening
├── EBS snapshots for backup
├── CloudWatch monitoring / log rotation
├── WhatsApp reconnection stability testing
├── Crash recovery testing (process restart, container cleanup)
└── Cost monitoring (Anthropic API usage)
```

**Phase ordering rationale:**
- Phase 1 must be first: everything else depends on a working NanoClaw instance
- Phase 2 before 3: the agent needs identity before learning to use calendar
- Phase 3 before 4: calendar is the key differentiator, tasks are bonus
- Phase 4 before 5: recurring tasks need to work before hardening
- Phase 5 last: polish after core functionality is proven

## Sources

- NanoClaw source code: https://github.com/qwibitai/nanoclaw (direct analysis of HEAD, all key files read)
- NanoClaw SPEC.md: upstream documentation with architecture diagrams and deployment instructions
- NanoClaw add-whatsapp skill: WhatsApp channel setup and authentication flow
- NanoClaw container/Dockerfile: container image specification
- NanoClaw container/agent-runner/src/index.ts: agent execution protocol
- NanoClaw container/agent-runner/src/ipc-mcp-stdio.ts: MCP server tools available to agents
