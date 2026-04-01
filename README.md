# Nano-Claw WhatsApp Assistant

A personal AI assistant for an independent AI/Data/Software consultant, built on [NanoClaw](https://github.com/qwibitai/nanoclaw) and deployed on AWS EC2. When tagged in a WhatsApp conversation, it replies to clients on the consultant's behalf — answering questions, scheduling meetings via Google Calendar, and handling recurring tasks.

## How It Works

```
Client sends WhatsApp message → tags @Assistant
    │
    ▼
NanoClaw (EC2) picks up the message
    │
    ▼
Spawns isolated Docker container with Claude Agent SDK
    ├── Reads knowledge brief (services, pricing, style)
    ├── Checks Google Calendar (if scheduling)
    └── Composes reply in consultant's voice
    │
    ▼
Reply sent back to WhatsApp conversation
```

The assistant only responds when explicitly tagged — no autopilot. This keeps the consultant in control while offloading routine client communication.

## Features

### v1 (Current Scope)

- **Tag-to-reply** — Responds when tagged in any WhatsApp group or DM
- **Knowledge-driven responses** — Loaded with a brief about services, pricing, expertise, and communication style
- **Voice matching** — Replies in the consultant's tone, not generic chatbot speak
- **Graceful handoff** — Says "let me check with [name]" when uncertain, never fabricates commitments
- **Google Calendar integration** — Checks real availability, schedules meetings with client details
- **Recurring tasks** — Weekly reminders, follow-ups, scheduled messages via NanoClaw's built-in scheduler
- **Context isolation** — Client A's conversation never leaks into Client B's context
- **Always-on** — Runs 24/7 on EC2 with automatic crash recovery

### v2 (Planned)

- Multi-language support (Hebrew/English auto-detection)
- Proposal and quote drafting from templates
- Conditional follow-ups ("remind if no response in 3 days")
- Meeting prep summaries before scheduled calls
- Weekly digest of active clients and pending items

## Architecture

```
┌────────────────────── EC2 (t3.small) ──────────────────────┐
│                                                             │
│  NanoClaw Host (Node.js 22, native — not containerized)    │
│  ├── WhatsApp Channel (Baileys 6.7.x WebSocket)           │
│  ├── Message Loop (polls SQLite every 2s)                  │
│  ├── Task Scheduler (polls SQLite every 60s)               │
│  ├── Group Queue (FIFO, max 1-2 concurrent containers)     │
│  └── IPC Watcher (filesystem-based, 1s poll)               │
│                                                             │
│  Docker Containers (agent execution)                        │
│  ├── Claude Agent SDK (Haiku 4.5 for cost efficiency)      │
│  ├── MCP: nanoclaw (send_message, schedule_task)           │
│  ├── MCP: gcal (check_availability, create_event) [NEW]    │
│  ├── Knowledge Brief (groups/global/CLAUDE.md)             │
│  └── Per-client context (groups/{client}/CLAUDE.md)        │
│                                                             │
│  Data                                                       │
│  ├── SQLite (messages, tasks, sessions, groups)            │
│  ├── WhatsApp auth (Baileys creds in store/auth/)          │
│  └── Google credentials (mounted separately from agent)     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Key design decisions:**
- NanoClaw runs natively on EC2 host (not containerized) to avoid known Linux container-in-container issues
- Docker is used only for agent containers that NanoClaw spawns — security boundary against prompt injection
- Container concurrency tuned for t3.small (MAX_CONCURRENT_CONTAINERS=1-2, memory limits, reduced timeouts)
- Google Calendar via custom MCP server (not hardcoded API calls) — follows NanoClaw's extension patterns
- Service Account auth for Google Calendar (no token expiry issues on headless EC2)

## Tech Stack

| Component | Technology | Why |
|-----------|-----------|-----|
| Framework | [NanoClaw](https://github.com/qwibitai/nanoclaw) (fork) | ~3.9k lines, built for individual users, WhatsApp + scheduling built-in |
| Runtime | Node.js 22 LTS, TypeScript 5.7 | Required by NanoClaw |
| AI Model | Claude Haiku 4.5 via Anthropic API | Cost-efficient ($1/$5 per MTok), sufficient for FAQ + scheduling |
| WhatsApp | Baileys 6.7.21 (stable) | Protocol-level, no browser needed, NanoClaw's chosen library |
| Database | SQLite via better-sqlite3 | Zero-config, single-file, sufficient for 5-10 clients |
| Calendar | googleapis + google-auth-library | Official Google client, Service Account auth |
| Containers | Docker 24+ | Agent isolation — each request runs in its own container |
| Infrastructure | AWS EC2 t3.small, Amazon Linux 2023 | Persistent compute for long-running process, ~$10-35/month total |

## Roadmap

The project is structured in 3 phases, each delivering a complete capability:

| Phase | Goal | Status |
|-------|------|--------|
| **1. Infrastructure & WhatsApp** | NanoClaw deployed on EC2, WhatsApp connected, stable 24/7 operation | Not started |
| **2. Knowledge & Communication** | Knowledge brief loaded, assistant replies in consultant's voice with safe handoff | Not started |
| **3. Calendar & Automation** | Google Calendar scheduling + recurring task automation | Not started |

See [.planning/ROADMAP.md](.planning/ROADMAP.md) for detailed phase breakdown and success criteria.

## Getting Started

### Prerequisites

- AWS account with EC2 access
- Anthropic API key
- Google Cloud project with Calendar API enabled
- A test WhatsApp number (never develop with the real business number)

### Deployment

```bash
# On EC2 (Amazon Linux 2023)
sudo dnf update -y && sudo dnf install -y docker git
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh | bash
nvm install 22

# Clone fork, install deps
git clone <fork-url> ~/nanoclaw && cd ~/nanoclaw
npm ci && npm install googleapis google-auth-library

# Build agent container
cd container && bash build.sh && cd ..

# Configure
cp .env.example .env  # Set ANTHROPIC_API_KEY, ASSISTANT_NAME, etc.

# Start with systemd (or pm2)
pm2 start npm --name nanoclaw -- start
pm2 save && pm2 startup
```

### Configuration

```bash
# Key .env overrides for EC2
MAX_CONCURRENT_CONTAINERS=1    # Tuned for t3.small
IDLE_TIMEOUT=300000             # 5 min (not default 30)
CONTAINER_TIMEOUT=600000        # 10 min hard limit
```

## Known Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| WhatsApp session drops (unofficial API) | High | Health monitoring, alerting, documented QR re-auth runbook |
| WhatsApp account ban | Critical | Test number for development, rate limiting, no bulk messages |
| API cost spiral | Medium | Haiku model, context limits, Anthropic spending caps |
| EC2 memory (OOM) | Medium | Container limits, swap file, CloudWatch alerts |
| Google token expiry | Medium | Service Account auth (long-lived credentials) |

## Cost Estimate

| Component | Monthly Cost |
|-----------|-------------|
| EC2 t3.small (reserved) | ~$6 |
| EBS 20 GB gp3 | ~$2 |
| Anthropic API (Haiku, ~10 msgs/day) | $5-30 |
| **Total** | **~$13-38** |

## Out of Scope (v1)

- Full autopilot (responding without being tagged)
- Email integration
- Web dashboard or admin panel
- CRM integration
- Payment processing / invoicing
- Multi-user / team access

## Project Structure

```
.planning/              # Project planning and research
├── PROJECT.md          # Project definition and requirements
├── ROADMAP.md          # Phase breakdown and success criteria
├── REQUIREMENTS.md     # Detailed requirement traceability
├── STATE.md            # Current execution state
└── research/           # Domain research (stack, architecture, pitfalls, features)
```
