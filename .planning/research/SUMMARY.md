# Project Research Summary

**Project:** WhatsApp AI Assistant for Independent Consultants
**Domain:** AI-powered communication automation (NanoClaw-based, EC2-deployed)
**Researched:** 2026-04-01
**Confidence:** HIGH

## Executive Summary

This project involves deploying a WhatsApp AI assistant for an independent AI/Data/Software consultant using NanoClaw, an open-source AI agent framework built specifically for personal use with WhatsApp support. NanoClaw handles the complex parts (WhatsApp connectivity via Baileys, container-isolated agent execution, message routing, task scheduling) while the consultant customizes behavior through knowledge brief files and extends functionality via custom tools.

The recommended approach is to fork NanoClaw and deploy it natively (not containerized) on an EC2 t3.micro or t3.small instance running Amazon Linux 2023. The assistant will use a tag-to-reply pattern (never autonomous) to answer client questions, check Google Calendar availability, schedule meetings, and handle recurring tasks. This architecture supports 5-10 clients comfortably with a monthly cost of $10-35 (EC2 + Anthropic API). The key differentiator is Google Calendar integration, which requires building a custom MCP server that agents can call inside containers to check availability and create events.

The critical risks center on WhatsApp session stability (unofficial API can disconnect unpredictably, requiring manual QR re-authentication) and account bans (WhatsApp actively restricts unofficial clients). Prevention requires aggressive session monitoring, alerting, rate limiting, and using a test number during development. Secondary risks include API cost spirals from unbounded context, OOM kills on t3.micro if container concurrency isn't tuned, and Google OAuth token expiry in headless environments. The research provides clear mitigation strategies for each risk, with Phase 1 addressing infrastructure-level pitfalls before any feature work begins.

## Key Findings

### Recommended Stack

NanoClaw provides the foundation — a Node.js 22 process using TypeScript, better-sqlite3 for state, Docker for agent isolation, and Baileys 6.7.21 for WhatsApp connectivity. The project requires zero database servers (SQLite suffices), zero web frameworks (NanoClaw polls channels internally), and zero custom authentication (WhatsApp pairing, Claude API key, Google OAuth are all handled externally).

**Core technologies:**
- **NanoClaw (forked)**: AI agent orchestrator with WhatsApp support, message routing, container isolation, and task scheduling — chosen because it already solves 90% of the problem domain
- **Node.js 22 LTS + TypeScript 5.7**: Runtime environment — required by NanoClaw's .nvmrc specification
- **Docker 24+**: Container runtime for isolated agent execution — critical security boundary against prompt injection
- **SQLite via better-sqlite3**: Persistent state for messages, tasks, conversations — zero-config, single-file, sufficient for 5-10 clients
- **Baileys 6.7.21**: WhatsApp Web protocol client — reverse-engineered, unofficial but actively maintained, what NanoClaw uses
- **googleapis + google-auth-library**: Google Calendar API integration — official client library, mandatory for scheduling functionality
- **EC2 t3.micro/t3.small**: Infrastructure — persistent compute for long-running process (Lambda unsuitable), $3-8/month reserved pricing

**What NOT to use:**
- Baileys 7.x RC (stalled since Nov 2025, unstable)
- whatsapp-web.js (abandoned, unmaintained)
- AWS Lambda (ephemeral execution breaks persistent WhatsApp sessions)
- PostgreSQL/DynamoDB (over-engineered for single-user workload)
- Express/Fastify web server (creates attack surface with no benefit)

### Expected Features

The research identified clear boundaries between must-have table stakes (without which the assistant provides no value), differentiators (which reduce consultant workload meaningfully), and anti-features (deliberately avoided to prevent scope creep or risk).

**Must have (table stakes):**
- **Tag-to-reply in WhatsApp** — core value proposition, without this the assistant is useless
- **Knowledge brief ingestion** — consultant's services, pricing, and voice must shape all responses
- **Voice/style matching** — clients know this person; generic chatbot tone breaks trust
- **Google Calendar availability check** — "When are you free?" is the most common client question
- **Meeting scheduling** — checking availability is only half the value; must also create events
- **Multi-conversation context isolation** — never leak Client A's info into Client B's conversation
- **Graceful handoff/uncertainty handling** — say "Let me check with [name]" rather than hallucinate
- **Always-on availability** — clients message at any hour; EC2 with systemd ensures 24/7 operation

**Should have (competitive):**
- **Scheduled follow-ups** — "Follow up with Client X on Friday if they haven't responded" prevents dropped balls
- **Meeting prep summaries** — before scheduled meetings, send consultant a summary of recent conversation
- **Client-specific memory** — remember Client A prefers Tuesday meetings, Client B's project name, etc.
- **Recurring task automation** — "Every Monday, remind Client X about weekly standup link"
- **Web research on behalf of client** — answer "What's the latest on EU AI Act?" with fresh data, not just training data
- **Weekly digest** — every Monday morning, summary of active clients, pending follow-ups, meetings this week

**Defer (v2+):**
- Proposal/quote drafting (valuable but not urgent; consultant can draft initially)
- Smart availability suggestions (refinement on basic scheduling; premature early on)
- Multi-language support (consultant's clients primarily English/Hebrew; Claude handles this natively anyway)

**Anti-features (deliberately avoided):**
- Full autopilot replying without tags (trust must be built gradually; unsolicited AI replies could embarrass)
- Web dashboard or admin panel (everything managed via WhatsApp or config files)
- CRM integration (5-10 clients don't need Salesforce; per-group CLAUDE.md acts as lightweight CRM)
- Email channel (out of scope for v1; different formatting rules and threading complexity)
- Payment processing/invoicing (mixing financial transactions into WhatsApp is liability)

### Architecture Approach

NanoClaw is a single Node.js process that orchestrates AI agent execution in isolated Docker containers. The host process manages WhatsApp connectivity (Baileys WebSocket), polls SQLite every 2 seconds for new messages in registered groups, enqueues work in per-group FIFO queues with global concurrency limits, spawns Docker containers that run the Claude Agent SDK, and watches filesystem IPC directories for messages containers want to send back. Agents run in containers with mounted group folders (read-write workspace), global folder (read-only shared knowledge), and session directories (conversation history for resumption). This architecture is already defined by the framework — customization happens via CLAUDE.md files (knowledge brief), custom MCP servers (Google Calendar), and container tuning (memory limits, concurrency).

**Major components:**
1. **NanoClaw Host Process** — orchestrates everything: WhatsApp channel (Baileys), message loop (2s poll), task scheduler (60s poll), IPC watcher (1s poll), group queue (FIFO with concurrency limit), container runner (Docker spawn/cleanup)
2. **Agent Containers** — isolated Docker environments running Claude Agent SDK with tools (Bash, Read, Write, WebSearch, etc.) and MCP servers (nanoclaw for send_message/schedule_task, custom gcal for calendar operations)
3. **Data Layer** — SQLite for transactional state (messages, tasks, sessions), filesystem for group memory (CLAUDE.md), WhatsApp auth (Baileys creds), and IPC (container-to-host communication)
4. **Google Calendar MCP Server (NEW)** — custom TypeScript MCP server using stdio transport, wraps googleapis client, exposes tools (check_availability, create_event, list_events), mounted into containers as additional MCP server alongside nanoclaw

**Key architecture decisions:**
- Run NanoClaw natively on EC2 host (not containerized) — avoids known Linux container-in-container issues (#1487)
- Use t3.small over t3.micro if budget allows — extra 1GB RAM ($4/month) prevents OOM kills under load
- Set MAX_CONCURRENT_CONTAINERS=1-2 — consultant has 5-10 clients, parallel processing rarely needed
- Store knowledge brief in groups/global/CLAUDE.md — agent runner loads this automatically, no custom code
- Mount Google credentials separately from CLAUDE.md — prevents tokens from leaking into agent context

### Critical Pitfalls

Research identified seven critical pitfalls, each with Phase 1-3 prevention strategies. The most dangerous risks involve WhatsApp stability and account bans, which are existential threats to a consultant using their personal business number.

1. **WhatsApp session disconnection** — Baileys WebSocket can drop at any time due to WhatsApp server-side detection, protocol changes, or token expiry. Reconnection requires manual QR scanning over SSH. **Prevention:** aggressive session persistence (verify Docker volumes preserve auth state), health-check cron (verify connection every 60s), alerting (email/SMS on disconnect), documented runbook for re-auth, consider lightweight web interface to display QR remotely.

2. **WhatsApp account ban** — Using unofficial APIs violates WhatsApp ToS. Consultant's personal number gets permanently banned, losing primary business communication channel with all clients. **Prevention:** never use real number for development/testing (use separate test number), implement rate limiting (2-5 second artificial delays before responses), never send bulk or identical messages, avoid connect/disconnect cycling, have fallback plan (WhatsApp appeal process, temporary client communication strategy).

3. **Claude API cost spiral** — Each WhatsApp message triggers Claude API call. With 10 messages of context, long knowledge briefs, and tool-use loops, a single interaction can hit 5-10K tokens. At 50 interactions/day, that's $7.50-$75/month on Haiku alone. **Prevention:** use Haiku 4.5 exclusively ($1/$5 per MTok), set MAX_MESSAGES_PER_PROMPT=5-7, implement spending caps via Anthropic dashboard, monitor token usage per interaction, keep knowledge brief under 2K tokens, enable prompt caching for system prompt.

4. **t3.micro OOM kills** — 1GB RAM with Node.js (~100-200MB), Docker containers (each another Node.js process), and SQLite queries easily exceeds capacity with MAX_CONCURRENT_CONTAINERS=5 default. **Prevention:** set MAX_CONCURRENT_CONTAINERS=1-2, configure Docker memory limits (--memory=256m), enable 1-2GB swap file, immediate log rotation (NanoClaw issue #1554 documents unbounded log growth), monitor with CloudWatch at 80% threshold, consider t3.small upgrade.

5. **Google OAuth token expiry** — Refresh tokens expire after 6 months for apps in "testing" mode, or if revoked by user. When expired on headless EC2, calendar operations silently fail. **Prevention:** use Service Account instead of OAuth2 user flow (long-lived credentials, no browser re-auth), or if OAuth: store refresh tokens securely, implement automatic refresh, keep project in "production" mode, implement fallback response ("unable to check calendar, will follow up").

6. **NanoClaw Linux container crashes** — Documented issues (#1445, #1487) with running NanoClaw inside containers on Linux due to filesystem permissions, networking (iptables/nftables), and environment variable handling differences between macOS and Linux. **Prevention:** run NanoClaw directly on EC2 host (not containerized), use Docker only for agent containers NanoClaw spawns, test exact EC2 AMI and Docker version before deploying, pin versions in deployment script, staging instance for initial testing.

7. **Knowledge brief drift** — Static text file loaded at startup has no reminder to update. Over time, services change, pricing updates, brief becomes stale. Assistant confidently quotes outdated pricing. **Prevention:** version brief in git with deployment, add "last reviewed" date + monthly review reminder, structure as markdown sections for surgical updates, include "valid until" field for time-sensitive info, implement WhatsApp command "/update-brief" for updates without SSH.

## Implications for Roadmap

Based on combined research, the project should be structured into 5 phases with clear dependencies. The ordering is driven by three factors: NanoClaw's architecture (must establish foundation before customization), feature dependencies (calendar requires OAuth before scheduling), and pitfall mitigation (address infrastructure risks before feature development).

### Phase 1: Infrastructure & Foundation
**Rationale:** Everything depends on a stable, properly-tuned NanoClaw deployment on EC2. All infrastructure-level pitfalls must be addressed before feature work begins, or they will block development later.

**Delivers:**
- EC2 instance (t3.small recommended) with Amazon Linux 2023, Docker, Node.js 22
- NanoClaw forked and deployed natively on host (not containerized)
- WhatsApp channel connected using **test number** (not real consultant number)
- systemd service for auto-restart and boot startup
- Container tuning (MAX_CONCURRENT_CONTAINERS=1-2, memory limits, reduced timeouts)
- Log rotation configured (prevents disk-full crashes from issue #1554)
- 1-2GB swap file (safety net against OOM kills)
- CloudWatch memory monitoring with 80% alert threshold
- Basic health-check endpoint or cron to verify WhatsApp connection alive

**Addresses features:**
- Tag-to-reply in WhatsApp (table stakes)
- Always-on availability (table stakes)
- Multi-conversation context isolation (NanoClaw built-in, verify it works)

**Avoids pitfalls:**
- #4: t3.micro OOM kills — addressed via container tuning, swap, monitoring
- #6: NanoClaw Linux crashes — addressed via native host deployment, testing on target AMI
- #1: WhatsApp session disconnection — addressed via health monitoring foundation (alerting added Phase 2)

**Research flag:** Standard deployment pattern; NanoClaw docs + EC2 setup are well-documented. No additional research needed.

---

### Phase 2: Knowledge & Identity
**Rationale:** With infrastructure stable, the next bottleneck is giving the agent identity and domain knowledge. Without a knowledge brief, responses will be generic and useless. This phase also hardens operations (alerting, backups) now that the foundation is proven.

**Delivers:**
- global CLAUDE.md knowledge brief (consultant identity, services, pricing, communication style, escalation rules)
- Per-group CLAUDE.md templates for client-specific context
- Sender allowlist configured (restrict which contacts can trigger assistant)
- Registers first client group (trigger-based, not autopilot)
- WhatsApp reconnection alerting (email/SMS when session drops)
- Documented runbook for QR re-authentication
- Daily SQLite backup to S3 or EBS snapshot
- Version knowledge brief in git with "last reviewed" date
- Test messaging with real scenarios, verify voice/style matching

**Addresses features:**
- Knowledge brief ingestion (table stakes)
- Voice/style matching (table stakes)
- Graceful handoff/uncertainty handling (table stakes)
- Client-specific memory foundation (differentiator)

**Avoids pitfalls:**
- #7: Knowledge brief drift — addressed via versioning, review date, git tracking
- #2: WhatsApp account ban — still using test number; real number connected only after style/behavior validated
- #1: WhatsApp session disconnection — alerting now in place for fast recovery

**Research flag:** Prompt engineering and style guidelines. May benefit from `/gsd:research-phase` on "consultant communication style best practices" if consultant has no existing style guide.

---

### Phase 3: Calendar Integration
**Rationale:** Google Calendar is the key differentiator that makes the assistant genuinely useful (answers "when are you free?"). This is the first custom code beyond configuration. Calendar must come before recurring tasks because scheduled follow-ups may reference calendar events.

**Delivers:**
- Google Cloud project setup (service account or OAuth2, production mode)
- Custom Google Calendar MCP server (TypeScript, stdio transport)
- Tools: check_availability, create_event, list_events, update_event, cancel_event
- OAuth2 authentication (service account preferred to avoid token expiry)
- MCP server mounted into agent containers, registered alongside nanoclaw MCP
- Timezone handling in knowledge brief and API calls
- Fallback response when calendar unavailable ("Let me check and get back to you")
- Test: "Schedule meeting with John next Tuesday at 2pm" creates event

**Addresses features:**
- Google Calendar availability check (table stakes)
- Meeting scheduling (table stakes)

**Avoids pitfalls:**
- #5: Google OAuth token expiry — addressed via service account or production-mode OAuth with refresh monitoring

**Research flag:** Moderate complexity. Google Calendar API is well-documented, but MCP server creation and container integration may need `/gsd:research-phase` on "building custom MCP servers for NanoClaw" if patterns aren't clear from existing nanoclaw MCP code.

---

### Phase 4: Task Automation & Intelligence
**Rationale:** With calendar working, leverage NanoClaw's built-in task scheduler for recurring tasks and follow-ups. These features compound value over time (weekly digests, automatic reminders) without adding system complexity.

**Delivers:**
- Recurring task automation ("Every Monday at 9am, send standup link to Client X")
- Scheduled follow-ups with conditional logic ("Follow up Friday if no response")
- Meeting prep summaries (scheduled task before meetings, combines calendar + conversation history)
- Weekly digest (scheduled Sunday evening: active clients, pending follow-ups, week's meetings)
- Task monitoring dashboard (or WhatsApp command to list active tasks)
- Client-specific memory updates (agent instructed to update CLAUDE.md with key facts after conversations)

**Addresses features:**
- Scheduled follow-ups (differentiator)
- Meeting prep summaries (differentiator)
- Client-specific memory (differentiator)
- Recurring task automation (differentiator)
- Weekly digest (differentiator)

**Avoids pitfalls:**
- No new pitfalls introduced; builds on proven scheduler infrastructure

**Research flag:** Standard NanoClaw scheduling patterns. May need brief research on "conditional task scheduling patterns" for follow-ups that check message history before sending.

---

### Phase 5: Production Transition & Refinement
**Rationale:** After validation with test number, transition to consultant's real WhatsApp number. Add polish (response delays, web research, refinements) and production hardening (cost monitoring, crash recovery testing).

**Delivers:**
- Transition WhatsApp connection from test number to consultant's real number (careful cutover)
- Response delay (2-5 seconds) to simulate human typing
- Web research enabled (NanoClaw's search + fetch tools) for client questions
- API cost monitoring dashboard (Anthropic token usage alerts)
- Spending caps enforced ($50/month limit or similar)
- Crash recovery testing (process restart, container cleanup verification)
- EBS snapshots for disaster recovery
- Knowledge brief review reminder (monthly scheduled task)
- Documentation: operations runbook, troubleshooting guide, client onboarding process

**Addresses features:**
- Web research on behalf of client (differentiator)

**Avoids pitfalls:**
- #2: WhatsApp account ban — real number connected only after extensive validation, with rate limiting in place
- #3: Claude API cost spiral — monitoring and caps prevent overruns

**Research flag:** Operational best practices. Standard patterns; no additional research needed.

---

### Phase Ordering Rationale

- **Phase 1 before all others:** NanoClaw must run stably on EC2 before any feature work. Addressing OOM, log rotation, and Linux container issues first prevents rework later.
- **Phase 2 before 3:** The agent needs identity (knowledge brief) before it can intelligently use tools like calendar. Without domain knowledge, calendar integration would just be generic API calls.
- **Phase 3 before 4:** Scheduled tasks (follow-ups, meeting prep) reference calendar data. Calendar must exist first.
- **Phase 4 before 5:** Recurring tasks and intelligence features need validation with test number before risking real number.
- **Phase 5 last:** Production cutover only after all features proven stable. Polish and hardening after core functionality works.

This ordering also aligns with pitfall prevention: infrastructure pitfalls (#4, #6, #1 partially) addressed in Phase 1; knowledge pitfall (#7) in Phase 2; calendar pitfall (#5) in Phase 3; account ban risk (#2) deferred to Phase 5 after extensive testing.

### Research Flags

**Phases needing deeper research during planning:**
- **Phase 2 (Knowledge & Identity):** If consultant has no existing communication style guide, may benefit from research on "AI assistant personality design for professional services" to structure the knowledge brief effectively.
- **Phase 3 (Calendar Integration):** Building custom MCP servers is not extensively documented in NanoClaw's repo. May need `/gsd:research-phase` on "MCP server protocol implementation patterns" or "extending NanoClaw with custom tools via MCP."

**Phases with standard patterns (skip research-phase):**
- **Phase 1 (Infrastructure):** EC2 setup, NanoClaw deployment, systemd service configuration are all well-documented. STACK.md provides complete setup instructions.
- **Phase 4 (Task Automation):** NanoClaw's task scheduler is built-in with documented API (schedule_task MCP tool). Implementation is straightforward.
- **Phase 5 (Production Transition):** Operational best practices (monitoring, backups, cost controls) are standard DevOps patterns with abundant documentation.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | NanoClaw source code directly analyzed; all dependencies verified against npm registry; EC2 specifications from AWS docs; no uncertainty about technology choices |
| Features | MEDIUM-HIGH | Feature landscape based on NanoClaw capabilities (HIGH confidence) and consultant workflow analysis (MEDIUM confidence from domain knowledge and business chatbot research, not direct consultant interviews) |
| Architecture | HIGH | Architecture defined by NanoClaw's existing design (source code analysis); component boundaries and data flow verified directly from codebase; custom components (Calendar MCP) follow established patterns |
| Pitfalls | MEDIUM-HIGH | Critical pitfalls verified from NanoClaw GitHub issues (#1445, #1487, #1522, #1554), WhatsApp library ecosystems (Baileys, whatsapp-web.js), and Claude API pricing docs; recovery strategies based on documented incidents; resource constraints verified against AWS t3.micro specs |

**Overall confidence:** HIGH

The technical stack and architecture have very high confidence because NanoClaw is open-source with analyzable code, and all dependencies are established libraries with clear documentation. Feature prioritization has slightly lower confidence because it's based on inferred consultant workflows rather than direct requirements gathering, but the table stakes vs. differentiators distinction is well-supported by WhatsApp business automation research. Pitfall identification has high confidence for infrastructure risks (verified in actual GitHub issues) and medium-high for operational risks (based on community experience and API pricing docs).

### Gaps to Address

**Calendar MCP server implementation patterns:**
- Research identified *what* to build (custom MCP server using stdio transport, wrapping googleapis) but not detailed *how*. The nanoclaw MCP server source (container/agent-runner/src/ipc-mcp-stdio.ts) provides a reference implementation, but adapter patterns for external APIs may need clarification during Phase 3 planning.
- **How to handle:** Inspect nanoclaw MCP server code in detail during Phase 3 kickoff; if unclear, run `/gsd:research-phase calendar-mcp-implementation` to find MCP protocol examples and API wrapper patterns.

**Conditional task scheduling (follow-ups that check for replies):**
- NanoClaw's scheduler supports cron, interval, and one-time tasks, but conditional logic ("follow up only if no response received") is not documented. The scheduled_tasks table schema doesn't show fields for conditional checks.
- **How to handle:** During Phase 4 planning, analyze whether this requires custom MCP tool logic (task checks message history before deciding whether to send) or an extension to the scheduler. If unclear, may need brief research on "event-driven task patterns in NanoClaw."

**WhatsApp re-authentication UX on headless EC2:**
- Research documents that QR re-authentication is required when sessions drop, but the exact UX for displaying QR codes over SSH (qrcode-terminal vs. qrcode image file vs. web interface) needs validation.
- **How to handle:** Test both qrcode-terminal and qrcode image generation during Phase 1. Validate which works better over SSH (terminal may render poorly depending on SSH client). Consider simple Express endpoint just for QR display if terminal doesn't work.

**Consultant communication style guidelines:**
- Voice/style matching is a table stakes feature, but research didn't include actual consultant message samples. The knowledge brief structure will need consultant input.
- **How to handle:** During Phase 2 planning, request 10-15 real message samples from consultant (sanitized client names). Use these to write style guide section of CLAUDE.md. If consultant has no existing documentation, may benefit from research on "professional communication style frameworks" to structure the guidelines.

## Sources

### Primary (HIGH confidence)
- **NanoClaw GitHub repository** (qwibitai/nanoclaw, HEAD @ 2026-04-01) — complete source code analysis: architecture, component boundaries, data flow, default configurations, known issues
- **NanoClaw SPEC.md** — upstream architecture documentation with diagrams and deployment instructions
- **NanoClaw REQUIREMENTS.md** — design principles and component specifications
- **NanoClaw issues** — #1445 (Linux setup), #1487 (container crashes), #1522 (media messages), #1554 (unbounded logs), #1485 (Docker support)
- **npm registry** — version numbers, publish dates, and deprecation status for all dependencies (baileys, better-sqlite3, googleapis, etc.)
- **AWS EC2 pricing and specifications** — t3.micro/small specs, regional pricing
- **Anthropic API pricing documentation** — Claude Haiku/Sonnet/Opus per-token costs
- **Google Calendar API documentation** — googleapis usage patterns, OAuth2 flow, service account setup

### Secondary (MEDIUM confidence)
- **Baileys GitHub repository** (WhiskeySockets/Baileys) — ToS warnings, ban risk disclaimers, community reports
- **whatsapp-web.js issues** — #3948, #3894 documenting account restrictions after QR scanning (illustrates ban risk)
- **WATI WhatsApp AI chatbot guide** (wati.io) — business chatbot feature landscape
- **Trengo WhatsApp AI chatbot guide** (trengo.com) — knowledge base, escalation, CRM integration patterns
- **PROJECT.md** — consultant requirements, constraints, out-of-scope items

### Tertiary (LOW confidence, needs validation)
- **Domain knowledge of consultant workflows** — inferred from business communication patterns, not validated with actual consultant interviews
- **WhatsApp Business API ecosystem** — used for competitive feature research, but consultant is not using official Business API

---

*Research completed: 2026-04-01*
*Ready for roadmap: yes*
