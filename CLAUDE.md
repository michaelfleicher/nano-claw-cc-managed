<!-- GSD:project-start source:PROJECT.md -->
## Project

**Nano-Claw WhatsApp Assistant**

A personal AI assistant for an AI/Data/Software consultant, built on [NanoClaw](https://github.com/qwibitai/nanoclaw) and deployed on a minimal AWS EC2 instance. Connected to the consultant's personal WhatsApp number, it replies to clients on his behalf when tagged, schedules meetings via Google Calendar, and handles repeated tasks — freeing up time from day-to-day client communication.

**Core Value:** When tagged in a WhatsApp conversation, the assistant replies to clients accurately and professionally on the consultant's behalf, using knowledge of his services and communication style.

### Constraints

- **Tech stack**: NanoClaw (TypeScript/Node.js) — fork and customize, don't build from scratch
- **Infrastructure**: Minimal AWS EC2 instance — keep costs low
- **WhatsApp API**: Personal number via NanoClaw's WhatsApp channel skill
- **AI model**: Claude via Anthropic API (NanoClaw default)
- **Calendar**: Google Calendar API for availability and scheduling
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## Recommended Stack
### Core Technologies
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| NanoClaw (fork) | latest main | AI agent framework — host orchestrator, message routing, container isolation, SQLite state | The entire project premise. Small codebase (~3.9k lines), built for individual users, WhatsApp support via skill, Claude Agent SDK integrated. Fork and customize, do not rebuild. |
| Node.js | 22 LTS | Runtime for NanoClaw host process | NanoClaw's `.nvmrc` specifies 22. LTS release with long-term support. |
| TypeScript | ^5.7 | Language | NanoClaw's native language (93.7% of codebase). Already configured with tsconfig. |
| Docker | 24+ | Container runtime for agent isolation | NanoClaw's security model runs every agent in an isolated Linux container. Docker is the default runtime on Linux/EC2. |
| SQLite via better-sqlite3 | 12.8.0 | Persistent storage — messages, tasks, groups, scheduling | NanoClaw's built-in database. Synchronous API, zero-config, single-file. No external database server needed on a t3.micro. |
| @whiskeysockets/baileys | 6.7.21 (stable) | WhatsApp Web multi-device protocol | NanoClaw's WhatsApp skill uses Baileys. It reverse-engineers the WhatsApp Web protocol, connecting as a linked device on your personal number. Use the 6.x stable line, not the 7.0.0-rc series. |
| @anthropic-ai/sdk | ^0.81.0 | Claude API client | NanoClaw uses this indirectly via the Claude Agent SDK inside containers. Credentials are injected via a credential proxy. |
| @onecli-sh/sdk | ^0.2.0 | OneCLI service integration | NanoClaw's core dependency for the agent execution layer. |
| googleapis | ^171.0.0 | Google Calendar API client | Official Google API client for Node.js. Actively maintained (daily releases). Use the `google.calendar('v3')` submodule specifically. |
| google-auth-library | ^10.6.0 | Google OAuth2 authentication | Required for Calendar API auth. Handles OAuth2 flow, token refresh, and service account credentials. |
### Supporting Libraries
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| cron-parser | 5.5.0 | Parse cron expressions for task scheduling | Already in NanoClaw. Used by the task scheduler for recurring tasks (follow-ups, reminders). |
| qrcode-terminal | 0.12.0 | Display QR code in terminal for WhatsApp auth | Required during WhatsApp pairing on headless EC2. Shows QR code via SSH. |
| qrcode | ^1.5.4 | Generate QR code images | Alternative WhatsApp auth method — generates image file if terminal QR is hard to scan over SSH. |
| dotenv | ^16.4.0 | Load .env files | Standard practice for managing secrets (Anthropic API key, Google credentials) on EC2. NanoClaw already reads from .env. |
| pm2 | ^5.4.0 | Process manager for Node.js | Run NanoClaw as a daemon on EC2. Auto-restart on crash, log rotation, startup script generation for systemd. See "Deployment" section below. |
### Infrastructure (EC2)
| Component | Specification | Purpose | Why |
|-----------|--------------|---------|-----|
| EC2 Instance | t3.micro (1 vCPU, 1 GB RAM) | Host NanoClaw process + Docker containers | Sufficient for single-user, 5-10 concurrent conversations. NanoClaw is a single Node.js process. Containers are short-lived (30 min timeout, max 5 concurrent). Cost: ~$8/month on-demand, ~$3/month reserved. |
| AMI | Amazon Linux 2023 | Operating system | Latest Amazon Linux with systemd, Docker support, and long-term security patches. Lighter than Ubuntu for a single-purpose server. |
| Storage | 20 GB gp3 EBS | OS + Docker images + SQLite database | NanoClaw agent container image + SQLite state. gp3 is cheaper than gp2 with better baseline IOPS. |
| Security Group | Inbound: SSH (22) from your IP only. No other inbound. | Network security | NanoClaw connects outbound to WhatsApp servers and Anthropic API. No inbound ports needed — it is not a web server. |
| Elastic IP | 1 static IP | Persistent address | Prevents IP change on instance stop/start, which would break SSH access. Free when attached to a running instance. |
### Development Tools
| Tool | Purpose | Notes |
|------|---------|-------|
| vitest | ^4.0.0 | Test runner | Already configured in NanoClaw. Use for unit and integration tests on custom skills. |
| tsx | ^4.19.0 | TypeScript execution | Already in NanoClaw. Run .ts files directly without compilation step during development. |
| eslint + prettier | ^9.35 / ^3.8 | Linting and formatting | Already configured in NanoClaw with custom rules. |
| gh CLI | latest | GitHub operations | Fork management, PR creation, upstream sync. |
## Installation
# Fork and clone NanoClaw
# Install NanoClaw dependencies (includes better-sqlite3, cron-parser, @onecli-sh/sdk)
# Add Google Calendar integration
# WhatsApp dependencies (installed by /add-whatsapp skill)
# @whiskeysockets/baileys, qrcode, qrcode-terminal
# These are added automatically when running the skill
# Process management for EC2
# Dev dependencies (already in NanoClaw)
# vitest, tsx, typescript, eslint, prettier — all included
## EC2 Deployment Stack
# On Amazon Linux 2023
# Install Node.js 22 via nvm
# Start Docker
# Clone fork, install deps
# Build agent container
# Configure environment
# Edit .env with: ANTHROPIC_API_KEY, ASSISTANT_NAME, TZ, etc.
# Start with pm2
## Google Calendar Integration Approach
## Alternatives Considered
| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Baileys 6.x (stable) | Baileys 7.0.0-rc.x | Only if 6.x has a blocking bug. The 7.x line is release candidate only (last RC: Nov 2025, then silence). Stable 6.x is safer for production. |
| Baileys (personal number) | WhatsApp Business API (official) | If you want official Meta support, higher message limits, or business verification. Requires a dedicated business number and Meta approval. Not suitable for personal number use. |
| t3.micro | t3.small (2 GB RAM) | If running more than 5 concurrent containers or adding memory-heavy tools. Monitor RAM usage — upgrade if OOM kills appear. |
| Amazon Linux 2023 | Ubuntu 24.04 LTS | If you prefer apt over dnf, or need Ubuntu-specific packages. Both work fine. AL2023 is slightly leaner. |
| pm2 | systemd unit file | If you prefer no extra dependency. pm2 is easier for Node.js (log rotation, restart, monitoring built in). |
| googleapis (full SDK) | google-calendar-api (REST calls) | Never. The googleapis package is the official client. Hand-rolling REST calls adds no value and misses auth token refresh. |
| SQLite (built-in) | PostgreSQL / DynamoDB | Never for v1. NanoClaw is architected around SQLite. Swapping databases means rewriting the data layer. SQLite handles this workload easily. |
| Service account (GCal auth) | OAuth2 refresh token | If the consultant's Google Workspace doesn't support service accounts, use OAuth2 with offline access and store the refresh token in .env. |
## What NOT to Use
| Avoid | Why | Use Instead |
|-------|-----|-------------|
| whatsapp-web.js | Abandoned/unmaintained since 2024. Was the main competitor to Baileys but development stalled. | @whiskeysockets/baileys — actively maintained, NanoClaw's chosen library |
| Baileys 7.0.0-rc.x | Release candidate since Sept 2025, no stable release. Last publish Nov 2025 — unclear if it will ship. | Baileys 6.7.21 (latest stable) |
| AWS Lambda | NanoClaw is a long-running process with SQLite, persistent WhatsApp sessions, and Docker containers. Lambda's ephemeral execution model breaks all of these. | EC2 (persistent compute) |
| RDS / DynamoDB | Over-engineered for single-user SQLite workload. Adds cost ($15+/month) and complexity for zero benefit. | better-sqlite3 (already integrated) |
| Puppeteer/Playwright for WhatsApp | Browser automation is fragile, resource-heavy, and breaks on WhatsApp Web updates. | Baileys (protocol-level, no browser needed) |
| Full googleapis SDK import | `import { google } from 'googleapis'` pulls the entire SDK. | `import { calendar_v3 } from 'googleapis'` or use `google.calendar('v3')` to tree-shake |
| Express/Fastify web server | NanoClaw is not a web app. It polls channels and routes messages internally. Adding a web server creates an attack surface with no benefit. | NanoClaw's built-in polling architecture |
| ngrok / webhook tunnels | Tempting for "receiving" WhatsApp messages, but Baileys maintains a persistent WebSocket — no inbound HTTP needed. | Baileys' built-in connection (outbound WebSocket) |
## Stack Patterns by Variant
- Upgrade to t3.small (2 GB RAM, ~$16/month on-demand)
- Or reduce MAX_CONCURRENT_CONTAINERS from 5 to 2 in .env
- Because Claude agent containers are the main RAM consumer
- Use the pairing code method (headless-friendly, no QR scan needed)
- NanoClaw's WhatsApp skill auto-detects headless environments
- Because pairing code is text-based, works over any terminal
- Fall back to OAuth2 with offline refresh token
- Run the OAuth consent flow once locally, copy the refresh token to EC2
- Because some Google Workspace admins restrict service account delegation
- Store auth state in NanoClaw's group folder (persisted across restarts)
- Baileys auto-reconnects on transient failures
- If persistent disconnects: WhatsApp may have flagged the linked device — re-pair
## Version Compatibility
| Package | Compatible With | Notes |
|---------|-----------------|-------|
| Node.js 22 | NanoClaw main | Required by .nvmrc. Do not use Node 20 (some NanoClaw deps may need 22 features). |
| better-sqlite3 12.x | Node.js 22 | Requires native compilation. Amazon Linux 2023 has gcc/make pre-installed. If build fails: `npm rebuild better-sqlite3`. |
| Baileys 6.7.x | Node.js 22 | Stable compatibility. |
| googleapis 171.x | Node.js 18+ | No compatibility issues with Node 22. |
| Docker 24+ | Amazon Linux 2023 | Available via `dnf install docker`. Ensure ec2-user is in the docker group. |
| TypeScript 5.7 | Node.js 22 | NanoClaw's tsconfig is pre-configured. |
## Cost Estimate (Monthly)
| Component | Cost |
|-----------|------|
| EC2 t3.micro (on-demand) | ~$8 |
| EC2 t3.micro (1yr reserved) | ~$3 |
| EBS 20 GB gp3 | ~$2 |
| Elastic IP | $0 (attached to running instance) |
| Anthropic API (Claude) | $5-30 (depends on usage, ~10 msgs/day) |
| **Total (reserved)** | **~$10-35/month** |
## Sources
- NanoClaw GitHub repository (qwibitai/nanoclaw) — repo structure, package.json, source code, SPEC.md, REQUIREMENTS.md [HIGH confidence]
- NanoClaw `/add-whatsapp` SKILL.md — WhatsApp integration details, Baileys usage, pairing flow [HIGH confidence]
- npm registry — version numbers and publish dates for all packages [HIGH confidence]
- AWS EC2 pricing page — instance costs [HIGH confidence, may vary by region]
- Google Calendar API documentation — googleapis usage pattern [HIGH confidence]
- Baileys npm publish history — 7.x RC stalled since Nov 2025 [HIGH confidence, verified via npm registry]
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd:quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd:debug` for investigation and bug fixing
- `/gsd:execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd:profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
