# Pitfalls Research

**Domain:** WhatsApp AI Assistant for Consultants (NanoClaw on EC2)
**Researched:** 2026-04-01
**Confidence:** MEDIUM-HIGH (verified against NanoClaw source, WhatsApp library ecosystems, Claude API pricing docs)

## Critical Pitfalls

### Pitfall 1: WhatsApp Session Disconnection and QR Re-authentication

**What goes wrong:**
Unofficial WhatsApp Web protocol libraries (Baileys, whatsapp-web.js) maintain a WebSocket session that WhatsApp's servers can terminate at any time. When the session drops, the bot goes silent with no outbound error to clients -- messages simply stop being answered. Re-authentication requires scanning a QR code from a physical phone, which cannot be automated. On a headless EC2 instance, this means SSH-ing in, generating the QR, and scanning it manually.

**Why it happens:**
WhatsApp actively detects and terminates unofficial API connections. Session tokens expire. Server-side protocol changes break clients without warning. NanoClaw issue #1445 documents "silent message drops" on Linux. The WhatsApp Web protocol is reverse-engineered and undocumented -- any server-side change can break the connection.

**How to avoid:**
- Implement aggressive session persistence -- store auth credentials to disk (NanoClaw likely does this, but verify the Docker volume mapping preserves `/data` across container restarts)
- Build a health-check endpoint or cron job that verifies the WhatsApp connection is alive every 60 seconds
- Set up alerting (email, SMS, or a secondary WhatsApp number) when the connection drops
- Keep a documented runbook for re-authentication: SSH command, how to display QR in terminal, expected time to restore
- Consider running a lightweight web interface (like noVNC or a simple Express endpoint) that can display the QR code remotely

**Warning signs:**
- Messages sent to the bot receive no response
- NanoClaw logs show WebSocket close events or authentication errors
- Clients report the assistant "went quiet"

**Phase to address:**
Phase 1 (Infrastructure Setup) -- session persistence and health monitoring must be baked into the initial deployment. Alerting in Phase 2 alongside operational hardening.

---

### Pitfall 2: WhatsApp Account Ban or Restriction

**What goes wrong:**
WhatsApp restricts or bans phone numbers that use unofficial APIs. The consultant's personal WhatsApp number -- used for all client communication -- gets permanently banned. This is catastrophic: losing the primary business communication channel with all active clients.

**Why it happens:**
WhatsApp's Terms of Service explicitly prohibit unofficial API clients. Baileys maintainers warn against "bulk or automated messaging." whatsapp-web.js users report number restrictions within 24 hours of QR scanning (issue #3948, #3894). Aggressive message patterns, rapid responses, or unusual connection patterns trigger detection.

**How to avoid:**
- Use the tag-to-reply pattern (already planned) -- never send unsolicited messages
- Rate-limit outbound messages: add artificial delays (2-5 seconds) before responding to mimic human typing
- Never send bulk messages or identical messages to multiple contacts
- Avoid connecting/disconnecting frequently -- keep one persistent session
- Consider using a secondary WhatsApp number for initial testing and development, only connecting the real number after the system is stable
- Have a fallback plan: if the personal number gets banned, know WhatsApp's appeal process and have a temporary communication plan for clients

**Warning signs:**
- WhatsApp shows "This account is not allowed to use WhatsApp" or similar
- Temporary restrictions (24-48 hour send blocks) are a precursor to permanent bans
- Unusual captcha challenges or verification requests

**Phase to address:**
Phase 1 (Development/Testing) -- MUST use a test number, not the consultant's real number. Only connect the real number after the system is proven stable. Phase 2 should include rate limiting and human-like response delays.

---

### Pitfall 3: Claude API Cost Spiral from Unbounded Context

**What goes wrong:**
Each WhatsApp message triggers a Claude API call. NanoClaw sends up to MAX_MESSAGES_PER_PROMPT (default: 10) messages as context, plus the system prompt with knowledge brief, plus any tool definitions. With Claude Haiku 4.5 at $1/$5 per MTok (input/output), costs seem low -- but group chats with active conversations, long knowledge briefs, and tool-use loops can push a single interaction to 5-10K tokens. At 50 interactions/day, that is 250-500K tokens/day, or $7.50-$75/month on Haiku alone. Accidentally using Sonnet or Opus triples or quintuples costs.

**Why it happens:**
Developers test with small conversations and extrapolate linearly. But real WhatsApp groups have long message histories, and NanoClaw spawns containers that may make multiple Claude calls per interaction (tool use, retries). Without explicit cost controls, a busy week can blow through a monthly budget.

**How to avoid:**
- Use Claude Haiku 4.5 ($1/$5 per MTok) for all WhatsApp responses -- it is more than capable for scheduling and FAQ responses
- Set MAX_MESSAGES_PER_PROMPT to 5-7 (not 10) to limit context window per call
- Implement monthly/daily spending caps via Anthropic's API usage limits
- Monitor token usage per interaction -- log input_tokens and output_tokens from every API response
- Keep the knowledge brief concise (<2K tokens) -- distill, don't dump
- Use prompt caching for the system prompt and knowledge brief (0.1x input cost on cache hits)

**Warning signs:**
- Anthropic dashboard shows unexpected spending spikes
- Individual interactions consuming >5K output tokens
- Container logs showing multiple Claude calls per single WhatsApp message

**Phase to address:**
Phase 1 (Configuration) -- set the model to Haiku, configure MAX_MESSAGES_PER_PROMPT. Phase 2 -- add spending monitoring and alerts.

---

### Pitfall 4: t3.micro Running Out of Memory with Docker + Node.js + SQLite

**What goes wrong:**
A t3.micro has 1 GB RAM. NanoClaw runs Node.js (~100-200 MB baseline), spawns Docker containers for agent execution (each container is another Node.js process), and SQLite operations consume memory for large queries. With MAX_CONCURRENT_CONTAINERS=5 (default), peak memory easily exceeds 1 GB, triggering OOM kills. The process dies silently, WhatsApp disconnects, and clients get no response.

**Why it happens:**
NanoClaw was designed for desktop use (macOS with 8-16 GB RAM). The default MAX_CONCURRENT_CONTAINERS=5 assumes abundant resources. Docker itself has memory overhead. NanoClaw issue #1554 reports unbounded log file growth to "hundreds of megabytes," which compounds the problem.

**How to avoid:**
- Set MAX_CONCURRENT_CONTAINERS=1 or 2 -- a solo consultant with 5-10 clients will rarely need parallel processing
- Configure Docker memory limits per container (--memory=256m)
- Enable swap (1-2 GB swap file on EC2) as a safety net against OOM kills
- Set up log rotation immediately -- NanoClaw's unbounded log growth will eat disk and memory
- Monitor with CloudWatch free tier: track MemoryUtilization and trigger alerts at 80%
- Consider t3.small (2 GB) if budget allows -- the $4/month difference is cheap insurance

**Warning signs:**
- dmesg shows OOM killer messages
- Process exits with signal 9 (SIGKILL)
- System becomes unresponsive during peak usage
- Swap usage consistently above 50%

**Phase to address:**
Phase 1 (Infrastructure) -- configure container limits, swap, log rotation, and MAX_CONCURRENT_CONTAINERS before anything else. This is a deployment-blocking concern.

---

### Pitfall 5: Google Calendar OAuth Token Expiry in Headless Environment

**What goes wrong:**
Google OAuth2 refresh tokens can expire or be revoked (Google's policy: refresh tokens expire after 6 months if the app is in "testing" mode and not verified, or if the user revokes access). When the token expires on a headless EC2 instance, calendar operations silently fail. The assistant tells clients "Let me check your availability" and then either errors out or returns stale data.

**Why it happens:**
Google OAuth2 for unverified apps (which this will be -- no need for full verification for a single-user tool) has strict token lifetime limits. Developers test with fresh tokens and don't encounter expiry during development. The token expires weeks later in production with no automated recovery path.

**How to avoid:**
- Use a Google Service Account instead of OAuth2 user flow -- service accounts have long-lived credentials and don't require browser-based re-authentication
- If using OAuth2: store refresh tokens securely, implement automatic token refresh, and alert when refresh fails
- Keep the Google Cloud project in "production" mode (not "testing") even for internal use -- testing mode tokens expire in 7 days
- Implement a fallback response: "I'm unable to check the calendar right now, let me have [consultant name] get back to you"
- Test token refresh explicitly before deploying

**Warning signs:**
- Calendar queries return 401 Unauthorized
- Google API client logs show "Token has been expired or revoked"
- Clients report the assistant can't schedule meetings but can still chat

**Phase to address:**
Phase 3 (Calendar Integration) -- design the auth strategy before writing any calendar code. Service account is the recommended path.

---

### Pitfall 6: NanoClaw Container Crashes on Linux/EC2 (Known Issue)

**What goes wrong:**
NanoClaw has documented issues running inside containers on Linux. Issue #1487 reports "crashing consistently when run inside container" and issue #1445 documents "silent message drops, firewall blocking, .env corruption" on Linux. Since the project was primarily developed on macOS, Linux/Docker behavior differs.

**Why it happens:**
Container runtime differences between macOS (Docker Desktop / Apple Containers) and Linux (native Docker). Filesystem permissions, networking (iptables/nftables), and environment variable handling differ. NanoClaw's container-in-container pattern (host runs NanoClaw, NanoClaw spawns agent containers) is particularly fragile on Linux.

**How to avoid:**
- Run NanoClaw directly on the EC2 host (not inside a container) -- use Docker only for the agent containers NanoClaw spawns
- If containerizing NanoClaw itself: use Docker socket mounting (--volume /var/run/docker.sock) carefully, with proper permissions
- Test the exact EC2 AMI and Docker version before deploying -- don't assume macOS behavior transfers
- Pin Docker and Node.js versions in the deployment script
- Set up a staging EC2 instance for initial testing before going live

**Warning signs:**
- NanoClaw process exits immediately on startup
- Agent containers spawn but produce no output
- Messages received but never processed (silent drops)
- .env file contents corrupted or empty after container restart

**Phase to address:**
Phase 1 (Infrastructure) -- this is the first thing to validate. Spin up EC2, install NanoClaw natively (not containerized), verify it runs before any feature work.

---

### Pitfall 7: Knowledge Brief Drift and Stale Responses

**What goes wrong:**
The consultant loads a knowledge brief about services, pricing, and communication style. Over time, services change, pricing updates, and the brief becomes stale. The assistant confidently quotes outdated pricing or describes deprecated services to clients. Unlike a website that gets visibly updated, a system prompt is invisible -- there is no reminder to update it.

**Why it happens:**
The knowledge brief is a static text file loaded at startup. There is no version control, no review schedule, and no mechanism to flag when information might be outdated. The consultant is busy (that is the whole point of the assistant) and forgets to update it.

**How to avoid:**
- Version the knowledge brief in git with the deployment
- Add a "Last reviewed" date to the brief and a scheduled task that alerts the consultant monthly to review it
- Keep the brief structured (YAML or markdown sections) so updates are surgical, not full rewrites
- Include a "valid until" field for time-sensitive information (pricing, availability, promotions)
- Implement a WhatsApp command like "/update-brief" that lets the consultant update the brief without SSH

**Warning signs:**
- Clients report incorrect pricing or service descriptions
- Brief has not been updated in >30 days
- Consultant mentions new services in conversations but the assistant doesn't know about them

**Phase to address:**
Phase 2 (Knowledge Configuration) -- build the brief with maintenance in mind from day one. Phase 4 (Operations) -- add review reminders.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Hardcoding the knowledge brief in the system prompt | Fast to get running | Cannot update without redeploying; no audit trail | MVP only, replace with file-based loading in Phase 2 |
| No log rotation | One less thing to configure | Disk fills up, NanoClaw crashes (issue #1554) | Never -- configure from day one |
| Using consultant's real WhatsApp number for development | No need for second number | Risk of ban on primary business number | Never -- always use a test number for development |
| Skipping health monitoring | Faster initial deployment | Silent failures go undetected for hours/days | MVP only if deployment timeline is extremely tight |
| MAX_CONCURRENT_CONTAINERS at default (5) | Handles burst load | OOM kills on t3.micro | Never on t3.micro -- set to 1-2 immediately |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| WhatsApp (NanoClaw channel) | Assuming connection is persistent and reliable | Treat as unreliable: health checks, reconnection logic, alerting on disconnection |
| Claude API | Using Sonnet/Opus for simple FAQ responses | Default to Haiku 4.5 for all WhatsApp responses; only escalate model for complex reasoning if ever needed |
| Claude API | Sending full conversation history every call | Limit context with MAX_MESSAGES_PER_PROMPT; use prompt caching for system prompt |
| Google Calendar | Using OAuth2 user flow requiring browser re-auth | Use Service Account with domain-wide delegation or calendar sharing |
| Google Calendar | Not handling API rate limits (Calendar API: 500 requests/100 seconds) | Implement retry with exponential backoff; cache availability for short periods |
| Docker on EC2 | Running NanoClaw itself inside a container | Run NanoClaw natively; only use Docker for agent containers it spawns |
| SQLite | Not persisting the database file across deployments | Mount SQLite DB on a persistent path; back up regularly |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Unbounded log growth | Disk full, process crash | Configure logrotate immediately; NanoClaw issue #1554 confirms this | Days to weeks depending on traffic |
| Large system prompts + message history | Slow responses, high API costs | Cap system prompt at 2K tokens, MAX_MESSAGES_PER_PROMPT at 5-7 | Noticeable when knowledge brief exceeds 3K tokens |
| Container accumulation | Memory exhaustion, OOM kills | Set MAX_CONCURRENT_CONTAINERS=1-2; verify containers are cleaned up after use | With >3 concurrent conversations on t3.micro |
| SQLite WAL file growth | Disk space, slow writes | Run periodic PRAGMA wal_checkpoint; NanoClaw does not configure WAL mode explicitly | After weeks of continuous operation |
| Docker image cache on small disk | Root partition full, Docker daemon stops | Prune unused images weekly; use a larger EBS root volume (16-20 GB) | When agent image is rebuilt or updated multiple times |

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Storing Anthropic API key in .env without file permissions | Anyone with EC2 access reads the key and incurs charges | chmod 600 .env; use AWS Secrets Manager or Parameter Store for production |
| Exposing Docker socket to agent containers | Container escape: agent code can control the host | NanoClaw uses container isolation by design; verify agent containers do NOT get the Docker socket |
| Google Calendar credentials in the repo | Credential leak via git history | .gitignore all credential files; store in ~/.config/ or AWS Secrets Manager |
| No sender allowlist on WhatsApp | Anyone who messages the number gets AI responses | Configure NanoClaw's sender-allowlist.ts to restrict which numbers/groups trigger the assistant |
| Running NanoClaw as root | Container escape has full system access | Create a dedicated nanoclaw user; run with minimal privileges |

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Assistant responds to every message in a group | Noise; group members get annoyed by bot chatter | Tag-to-reply only (already planned); make sure NanoClaw's trigger pattern is strict |
| No "I don't know" fallback | Assistant halluccinates answers about services it doesn't understand | Explicit instruction in system prompt: "If unsure, say you'll have [name] follow up personally" |
| Instant responses (0-1 second) | Feels robotic; clients realize it's a bot immediately | Add 2-5 second delay before responding to simulate reading/typing |
| No timezone handling | Meeting scheduling suggests times in UTC or wrong timezone | Configure TZ in NanoClaw config; confirm timezone in knowledge brief |
| Assistant schedules over existing commitments | Double-booked calendar | Always check calendar availability before proposing times; handle "busy" blocks |

## "Looks Done But Isn't" Checklist

- [ ] **WhatsApp connection:** Bot connected and responding -- but no health monitoring, so you won't know when it disconnects
- [ ] **Calendar integration:** Can create events -- but refresh token will expire in 7 days if Google project is in "testing" mode
- [ ] **Knowledge brief loaded:** Assistant answers correctly today -- but no process to update when services/pricing change
- [ ] **Deployed on EC2:** Process running -- but no systemd service or process manager, so it won't restart after a reboot or crash
- [ ] **Docker configured:** Containers spawning -- but no memory limits set, so OOM kills will happen under load
- [ ] **Log files created:** Logging works -- but no rotation configured, so disk will fill up (NanoClaw issue #1554)
- [ ] **Sender allowlist:** Configured for known contacts -- but new client numbers won't get responses unless allowlist is updated
- [ ] **Backup:** SQLite DB exists -- but no automated backup, so a disk failure loses all conversation history and scheduled tasks

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| WhatsApp session drop | LOW | SSH into EC2, restart NanoClaw, re-scan QR code (5-10 minutes if prepared) |
| WhatsApp account ban | HIGH | Appeal to WhatsApp (uncertain timeline); get a new number; notify all clients of number change |
| API cost overrun | LOW | Set spending limit in Anthropic dashboard; reduce MAX_MESSAGES_PER_PROMPT; switch to Haiku |
| EC2 OOM crash | LOW | Restart process; reduce MAX_CONCURRENT_CONTAINERS; add swap; consider t3.small upgrade |
| Google token expiry | MEDIUM | Re-authenticate via browser (requires port forwarding or web UI); switch to service account to prevent recurrence |
| NanoClaw Linux crash | MEDIUM | Check issues #1487, #1445 for fixes; run natively instead of containerized; test on fresh EC2 |
| Stale knowledge brief | LOW | Update the brief file; restart NanoClaw; takes effect immediately |
| SQLite corruption | HIGH | Restore from backup (if one exists); rebuild from message history; prevent by proper shutdown handling |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| WhatsApp session disconnection | Phase 1 (Infrastructure) | Health check endpoint returns "connected"; alerting fires on disconnect |
| WhatsApp account ban | Phase 1 (Development) | All development and testing uses a separate test number |
| Claude API cost spiral | Phase 1 (Configuration) | Model set to Haiku; MAX_MESSAGES_PER_PROMPT <= 7; spending cap configured |
| t3.micro OOM | Phase 1 (Infrastructure) | MAX_CONCURRENT_CONTAINERS=1-2; swap enabled; CloudWatch memory alert set |
| Google OAuth token expiry | Phase 3 (Calendar) | Service account used; or OAuth refresh tested and monitored |
| NanoClaw Linux crashes | Phase 1 (Infrastructure) | NanoClaw runs natively on EC2 (not containerized); verified stable for 24h before proceeding |
| Knowledge brief drift | Phase 2 (Knowledge) | Brief has "last reviewed" date; monthly review reminder scheduled |
| Log file growth | Phase 1 (Infrastructure) | logrotate configured; disk usage monitored |
| No process manager | Phase 1 (Infrastructure) | systemd unit file created; process auto-restarts on crash and reboot |
| No backup strategy | Phase 2 (Operations) | Daily SQLite backup to S3; tested restore procedure |

## Sources

- NanoClaw GitHub repository: https://github.com/qwibitai/nanoclaw
- NanoClaw issues: #1445 (Linux setup issues), #1487 (container crashes), #1522 (media messages), #1554 (unbounded logs), #1485 (Docker support)
- NanoClaw source: config.ts (MAX_CONCURRENT_CONTAINERS=5, CONTAINER_TIMEOUT defaults), db.ts (SQLite schema, no WAL mode), index.ts (shutdown handling), container-runner.ts (no memory limits)
- Baileys GitHub: https://github.com/WhiskeySockets/Baileys (ToS warning, ban risk disclaimer)
- whatsapp-web.js issues: #3948, #3894 (account restrictions after QR scanning)
- Anthropic API pricing: https://platform.claude.com/docs/en/docs/about-claude/pricing (Haiku 4.5: $1/$5 MTok; Sonnet: $3/$15 MTok)
- AWS EC2 t3.micro specs: 1 GB RAM, 2 vCPUs (burstable), ~$8.50/month

---
*Pitfalls research for: WhatsApp AI Assistant (NanoClaw on EC2)*
*Researched: 2026-04-01*
