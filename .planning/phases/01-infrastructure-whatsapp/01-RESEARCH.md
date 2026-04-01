# Phase 1: Infrastructure & WhatsApp - Research

**Researched:** 2026-04-01
**Domain:** NanoClaw fork/deploy, EC2 infrastructure, WhatsApp channel integration via Baileys
**Confidence:** HIGH

## Summary

Phase 1 delivers the foundational infrastructure: fork NanoClaw, deploy it on EC2 with Docker and systemd-based process management, connect WhatsApp via the `/add-whatsapp` skill (which merges code from the `qwibitai/nanoclaw-whatsapp` remote), and verify per-group context isolation. NanoClaw's architecture is a single Node.js process that polls SQLite for messages, routes them to isolated Docker containers running the Claude Agent SDK, and returns responses through the originating channel. WhatsApp integration uses `@whiskeysockets/baileys` 6.x connecting as a linked device via the multi-device protocol -- no inbound ports needed.

The main risks are: (1) memory pressure on a t3.small (2 GB RAM) when running the host process plus Docker containers with Chromium, requiring MAX_CONCURRENT_CONTAINERS=1-2 and swap; (2) WhatsApp session stability since Baileys uses an unofficial protocol that can drop sessions requiring re-pairing; and (3) the container image is ~1 GB+ due to Chromium, which constrains the 20 GB EBS volume.

**Primary recommendation:** Fork NanoClaw, merge the WhatsApp skill from `nanoclaw-whatsapp` remote, deploy on EC2 t3.small with systemd (NanoClaw's built-in setup), configure MAX_CONCURRENT_CONTAINERS=2, and enable 1 GB swap. Use pairing code auth for headless WhatsApp connection.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| INFRA-01 | NanoClaw forked, configured, and running on AWS EC2 t3.small with process management (systemd) | NanoClaw has built-in `setup/service.ts` that generates systemd user units with lingering enabled, handles daemon-reload/enable/start. Fork via `gh repo fork`, deploy with `npm run build`, run setup step. |
| INFRA-02 | WhatsApp channel connected to consultant's personal phone number via NanoClaw skill | The `/add-whatsapp` skill merges code from `qwibitai/nanoclaw-whatsapp` remote. Adds `src/channels/whatsapp.ts` (WhatsAppChannel class), auth scripts, and Baileys dependency. Use pairing code method for headless EC2. |
| INFRA-03 | Per-group context isolation so client conversations never leak between each other | Built into NanoClaw core: each registered group gets its own folder (`groups/<folder>/`), own CLAUDE.md, own container mount, and own SQLite records. Containers only mount their group's folder. Cross-group access is impossible at OS level. |
| INFRA-04 | EC2 instance runs 24/7 with automatic crash recovery and log rotation | systemd unit with `Restart=on-failure` handles crash recovery. `loginctl enable-linger` keeps service alive after SSH logout. Log rotation via systemd journal or logrotate for `logs/` directory. |
| INFRA-05 | Container concurrency tuned for t3.small (MAX_CONCURRENT_CONTAINERS=1-2, swap enabled) | `config.ts` reads `MAX_CONCURRENT_CONTAINERS` from env (default 5). Set to 2 in `.env`. Container image includes Chromium (~500 MB). Each running container uses ~200-400 MB RAM. With 2 GB total, 2 concurrent is the safe max with swap. |
| COMM-01 | User can tag the assistant in any WhatsApp conversation and receive a contextual reply | NanoClaw trigger system: `DEFAULT_TRIGGER = @{ASSISTANT_NAME}`. Non-main groups require trigger by default (`requires_trigger=1` in `registered_groups` table). WhatsApp channel parses incoming messages, matches trigger pattern, enqueues for container processing. |
</phase_requirements>

## Standard Stack

### Core (Already in NanoClaw)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NanoClaw (fork) | latest main | AI agent framework | Project premise. ~3.9k lines, WhatsApp skill available, Claude Agent SDK integrated. |
| Node.js | 22 LTS | Runtime | NanoClaw `.nvmrc` specifies 22. Required by some dependencies. |
| TypeScript | ^5.7 | Language | NanoClaw native (93.7% of codebase). tsconfig pre-configured. |
| Docker | 24+ | Container runtime | NanoClaw security model -- every agent runs in isolated Linux container. |
| better-sqlite3 | 12.8.0 | Database | NanoClaw built-in. Messages, groups, sessions, tasks all in SQLite. |
| @whiskeysockets/baileys | 6.7.21 | WhatsApp protocol | Merged via nanoclaw-whatsapp remote. Stable 6.x line. |
| @onecli-sh/sdk | ^0.2.0 | Agent execution layer | NanoClaw core dependency for credential proxy and container orchestration. |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| qrcode-terminal | 0.12.0 | QR display in terminal | WhatsApp auth fallback if pairing code fails. |
| qrcode | ^1.5.4 | QR image generation | Alternative auth method. |
| pino | (bundled) | Logging | NanoClaw uses pino internally. Baileys logger set to silent level. |

### Infrastructure

| Component | Specification | Purpose |
|-----------|--------------|---------|
| EC2 Instance | t3.small (2 GB RAM) | Host NanoClaw + Docker containers. Requirements specify t3.small, not t3.micro. |
| AMI | Amazon Linux 2023 | OS with systemd, Docker support, gcc/make for native modules. |
| Storage | 20 GB gp3 EBS | OS + Docker images + SQLite. |
| Security Group | SSH (22) from admin IP only | No inbound needed -- Baileys uses outbound WebSocket. |
| Elastic IP | 1 static IP | Stable SSH access across stop/start. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| systemd (user unit) | pm2 | pm2 adds a dependency; NanoClaw's setup already generates systemd units with linger. Use systemd. |
| Pairing code auth | QR code in terminal | QR requires camera scan over SSH which is awkward. Pairing code is text-based, recommended for headless. |
| t3.small | t3.micro (1 GB) | Requirements specify t3.small. 1 GB is too tight for Docker + Chromium containers. |

## Architecture Patterns

### NanoClaw Message Flow

```
WhatsApp (Baileys WebSocket)
    |
    v
WhatsAppChannel.onMessage()
    |
    v
SQLite: messages table (insert)
    |
    v
Polling loop (2s interval, POLL_INTERVAL)
    |
    v
GroupQueue.enqueueMessageCheck()
    |
    v
Container spawned (Docker)
  - Mounts: group folder (rw), project root (ro for main only)
  - Stdin: JSON with prompt, sessionId, groupFolder, chatJid
  - Claude Agent SDK runs inside container
  - Credentials via OneCLI credential proxy (never in container)
    |
    v
IPC output (filesystem-based)
    |
    v
Router.ts -> WhatsAppChannel.send()
    |
    v
Baileys sendMessage() -> WhatsApp
```

### Group Isolation Model

```
groups/
  whatsapp_main/       # Admin/self-chat (is_main=1, no trigger required)
    CLAUDE.md           # Per-group agent memory
    logs/
  whatsapp_client_a/   # Client A group (requires_trigger=1)
    CLAUDE.md
    logs/
  whatsapp_client_b/   # Client B group (requires_trigger=1)
    CLAUDE.md
    logs/
```

Each group:
- Has its own `registered_groups` row in SQLite (JID, folder, trigger_pattern)
- Gets a dedicated filesystem folder mounted into its container
- Has independent session tracking (`sessions` table: group_folder -> session_id)
- Non-main groups only see their own folder (container mount restriction)
- Main group gets read-only project root mount (with .env shadowed by /dev/null)

### EC2 Deployment Structure

```
/home/ec2-user/
  nanoclaw/              # Cloned fork
    .env                 # ANTHROPIC_API_KEY, ASSISTANT_NAME, TZ, MAX_CONCURRENT_CONTAINERS
    data/env/env         # Copy of .env for container access (synced manually)
    store/
      auth/              # Baileys auth state (creds.json + signal keys)
      messages.db        # SQLite database
    groups/              # Per-group folders
    logs/                # Application logs
    dist/                # Built TypeScript output
    container/           # Dockerfile for agent containers
  .config/nanoclaw/
    mount-allowlist.json # Security: allowed additional mounts
    sender-allowlist.json # Security: allowed senders
```

### systemd Service Pattern

NanoClaw's `setup/service.ts` generates a user-level systemd unit:
- `Restart=on-failure` for crash recovery
- `loginctl enable-linger` so service survives SSH logout
- Environment variables loaded from the unit file
- Logs to `logs/nanoclaw.log` and systemd journal

For Amazon Linux 2023 (non-root ec2-user):
- Unit placed at `~/.config/systemd/user/nanoclaw.service`
- Managed via `systemctl --user {start|stop|restart|status} nanoclaw`
- Docker group membership required: `sudo usermod -aG docker ec2-user`

### Anti-Patterns to Avoid

- **Running as root on EC2:** NanoClaw setup detects root vs user and generates appropriate systemd units. Run as ec2-user, not root.
- **Using pm2 AND systemd:** Pick one. NanoClaw's setup generates systemd units. Adding pm2 creates conflicting process managers.
- **Mounting .env into containers:** NanoClaw explicitly shadows .env with /dev/null in container mounts. Credentials go through OneCLI proxy.
- **Opening inbound ports:** Baileys uses outbound WebSocket. No Express, no ngrok, no webhooks.
- **Setting MAX_CONCURRENT_CONTAINERS=5 on t3.small:** Each container has Chromium. 5 containers would consume all RAM instantly.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| WhatsApp protocol | Custom WebSocket client | Baileys via nanoclaw-whatsapp merge | Protocol complexity is enormous (encryption, multi-device sync, retry logic) |
| Process management | Custom daemon/watchdog | systemd via NanoClaw's setup/service.ts | Handles restart, linger, logging, boot start |
| Container orchestration | Custom Docker management | NanoClaw's container-runner.ts + group-queue.ts | Handles concurrency limits, retries, cleanup, mount security |
| WhatsApp auth | Manual credential management | NanoClaw's setup/whatsapp-auth.ts | Handles QR, pairing code, multi-file auth state, credential persistence |
| Group isolation | Custom per-group logic | NanoClaw's registered_groups + group-folder.ts | Already enforced at DB, filesystem, and container mount level |
| Channel routing | Custom message routing | NanoClaw's channel registry pattern | Self-registration, factory pattern, null-when-unconfigured |

**Key insight:** NanoClaw has already solved all the hard infrastructure problems. Phase 1 is about deploying and configuring, not building from scratch. The `/add-whatsapp` skill provides a complete merge-based installation path.

## Common Pitfalls

### Pitfall 1: WhatsApp Session Drops on EC2
**What goes wrong:** Baileys sessions can disconnect due to WhatsApp server-side revocation, IP changes, or linked device limits (max 4 linked devices per account).
**Why it happens:** Baileys uses an unofficial protocol. WhatsApp can revoke linked device sessions at any time.
**How to avoid:** Store auth state in `store/auth/` (NanoClaw default), ensure Elastic IP is attached (no IP changes on restart), monitor logs for `DisconnectReason` events. The WhatsApp channel auto-reconnects on transient failures.
**Warning signs:** `connection: 'close'` events in logs, `DisconnectReason.loggedOut` means re-pairing required.

### Pitfall 2: OOM Kills on t3.small
**What goes wrong:** Docker containers with Chromium consume 200-400 MB each. With the host process (~100 MB), Docker daemon (~100 MB), and OS overhead, 2 GB fills fast.
**Why it happens:** Default MAX_CONCURRENT_CONTAINERS=5 is designed for larger machines. Agent container includes full Chromium for browser automation.
**How to avoid:** Set `MAX_CONCURRENT_CONTAINERS=2` in .env. Enable 1 GB swap file. Monitor with `free -h` and `dmesg | grep -i oom`.
**Warning signs:** Process killed without error, systemd showing `SIGKILL` exit status, `dmesg` showing oom-killer invocations.

### Pitfall 3: Docker Group Not Active After Adding User
**What goes wrong:** After `sudo usermod -aG docker ec2-user`, Docker commands still fail with permission denied.
**Why it happens:** Group membership changes require a new login session. The current SSH session still has the old groups.
**How to avoid:** Log out and back in after adding to docker group. NanoClaw's setup detects this (`checkDockerGroupStale()`).
**Warning signs:** `docker info` fails with permission error despite user being in docker group.

### Pitfall 4: Pairing Code Expiry During Auth
**What goes wrong:** The WhatsApp pairing code expires in ~60 seconds. If the user doesn't enter it fast enough, auth fails.
**Why it happens:** WhatsApp's security policy on pairing codes.
**How to avoid:** Have the user ready at Settings > Linked Devices > Link a Device BEFORE generating the code. The skill writes the code to `store/pairing-code.txt` which can be polled.
**Warning signs:** Auth log shows `AUTH_STATUS: failed` with timeout.

### Pitfall 5: Forgetting to Sync .env to data/env/env
**What goes wrong:** Containers don't see updated environment variables.
**Why it happens:** NanoClaw reads .env for the host process, but containers get env from `data/env/env`. These must be manually synced.
**How to avoid:** After any .env change, run `mkdir -p data/env && cp .env data/env/env`.
**Warning signs:** Container logs show missing API keys or wrong config values.

### Pitfall 6: Container Image Not Built
**What goes wrong:** NanoClaw starts but agents fail with "image not found."
**Why it happens:** The Docker agent container image (`nanoclaw-agent:latest`) must be built from `container/Dockerfile` before first run.
**How to avoid:** Run `container/build.sh` or `docker build -t nanoclaw-agent:latest container/` during setup.
**Warning signs:** Logs show Docker pull errors or "no such image" for nanoclaw-agent.

### Pitfall 7: better-sqlite3 Native Build Failure
**What goes wrong:** `npm install` fails when compiling better-sqlite3 native module.
**Why it happens:** Missing build tools (gcc, make, python3) on the EC2 instance.
**How to avoid:** Amazon Linux 2023 has gcc/make pre-installed. If it fails: `sudo dnf install gcc-c++ make python3` then `npm rebuild better-sqlite3`.
**Warning signs:** npm install errors mentioning `node-gyp` or `better-sqlite3`.

## Code Examples

### Environment Configuration (.env)

```bash
# Required
ANTHROPIC_API_KEY=sk-ant-...       # Or use ANTHROPIC_AUTH_TOKEN
ASSISTANT_NAME=Andy                 # Trigger will be @Andy
TZ=Asia/Jerusalem                   # Consultant's timezone

# Performance tuning for t3.small
MAX_CONCURRENT_CONTAINERS=2         # Default is 5, too many for 2 GB RAM
CONTAINER_TIMEOUT=1800000           # 30 min container timeout (default)
IDLE_TIMEOUT=1800000                # 30 min idle timeout (default)

# Optional
ASSISTANT_HAS_OWN_NUMBER=false      # true if using a dedicated phone/SIM
```

### Swap File Setup (EC2)

```bash
# Create 1 GB swap file
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Persist across reboots
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
```

### WhatsApp Skill Merge Flow

```bash
# Add the WhatsApp remote
git remote add whatsapp https://github.com/qwibitai/nanoclaw-whatsapp.git

# Fetch and merge
git fetch whatsapp main
git merge whatsapp/main
# Resolve package-lock.json conflicts if any:
# git checkout --theirs package-lock.json && git add package-lock.json && git merge --continue

# Install new dependencies (baileys, qrcode, qrcode-terminal)
npm install

# Build and test
npm run build
npx vitest run src/channels/whatsapp.test.ts
```

### WhatsApp Pairing Code Auth (Headless EC2)

```bash
# Clean any previous auth state
rm -rf store/auth/

# Start auth in background with pairing code method
rm -f store/pairing-code.txt
npx tsx setup/index.ts --step whatsapp-auth -- --method pairing-code --phone 972XXXXXXXXX > /tmp/wa-auth.log 2>&1 &

# Poll for the code (appears within ~10 seconds)
for i in $(seq 1 20); do [ -f store/pairing-code.txt ] && cat store/pairing-code.txt && break; sleep 1; done

# User enters code on phone: Settings > Linked Devices > Link with phone number
# Poll for completion
for i in $(seq 1 60); do
  grep -q 'AUTH_STATUS: authenticated' /tmp/wa-auth.log 2>/dev/null && echo "Success" && break
  grep -q 'AUTH_STATUS: failed' /tmp/wa-auth.log 2>/dev/null && echo "Failed" && break
  sleep 2
done

# Verify
test -f store/auth/creds.json && echo "Auth OK" || echo "Auth FAILED"
```

### Group Registration

```bash
# Register main/self-chat (no trigger required)
npx tsx setup/index.ts --step register \
  --jid "972XXXXXXXXX@s.whatsapp.net" \
  --name "Main" \
  --trigger "@Andy" \
  --folder "whatsapp_main" \
  --channel whatsapp \
  --assistant-name "Andy" \
  --is-main \
  --no-trigger-required

# Register a client group (trigger required)
npx tsx setup/index.ts --step register \
  --jid "120363XXXXX@g.us" \
  --name "Client A" \
  --trigger "@Andy" \
  --folder "whatsapp_client_a" \
  --channel whatsapp
```

### Build Agent Container

```bash
cd container
./build.sh
# Or manually:
docker build -t nanoclaw-agent:latest .
```

### systemd Service Verification

```bash
# Check service status
systemctl --user status nanoclaw

# View recent logs
journalctl --user -u nanoclaw --since "10 min ago"

# Restart after config change
systemctl --user restart nanoclaw

# Verify linger is enabled (survives SSH logout)
loginctl show-user ec2-user | grep Linger
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| whatsapp-web.js | @whiskeysockets/baileys | 2024 | whatsapp-web.js abandoned. Baileys is the only maintained option. |
| Baileys 7.0.0-rc | Baileys 6.7.21 stable | 2025 | 7.x RC stalled since Nov 2025. Use 6.x stable line. |
| QR code auth | Pairing code auth | 2024 | Text-based auth works better on headless/SSH environments. |
| pm2 for Node.js | systemd user units | NanoClaw default | NanoClaw generates systemd units with linger. No extra dependency. |

**Deprecated/outdated:**
- whatsapp-web.js: Abandoned since 2024. Do not use.
- Baileys 7.0.0-rc.x: RC since Sept 2025, last publish Nov 2025, unclear if it will ship. Use 6.7.21.

## Open Questions

1. **OneCLI Service Availability**
   - What we know: NanoClaw uses `@onecli-sh/sdk` for the credential proxy. The host connects to `ONECLI_URL` (default `http://localhost:10254`).
   - What's unclear: How OneCLI is installed and started on EC2. NanoClaw's setup has an `init-onecli` skill.
   - Recommendation: Run the `/setup` skill which includes OneCLI initialization. This is a prerequisite for container-based agent execution.

2. **Agent Container Image Size**
   - What we know: The Dockerfile installs Chromium + fonts + agent-browser + claude-code globally. This will produce a large image (1-2 GB).
   - What's unclear: Exact size on Amazon Linux 2023, and whether 20 GB EBS is sufficient for OS + image + SQLite.
   - Recommendation: Build the image early and check `docker images` size. If tight, consider increasing EBS to 30 GB (adds ~$1/month).

3. **WhatsApp Account Ban Risk**
   - What we know: Baileys uses an unofficial protocol. Meta could theoretically flag or ban accounts using it.
   - What's unclear: Actual enforcement rates for personal-use linked devices with moderate message volumes.
   - Recommendation: Use a test number during development (as decided in STATE.md). Low message volume (<50/day) and natural conversation patterns reduce risk. Have the consultant's real number as a separate phase after validation.

## Environment Availability

> This phase deploys to EC2 (remote), not the local development machine. Environment checks below are for the local machine used to manage the project. EC2 provisioning is a task in the plan.

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| git | Fork management | Verify on EC2 | -- | Pre-installed on Amazon Linux 2023 |
| gh CLI | Fork creation | Verify locally | -- | Manual GitHub fork via web UI |
| ssh | EC2 access | Standard on macOS/Linux | -- | -- |
| AWS CLI or Console | EC2 provisioning | Manual via console | -- | Console is fine for single instance |
| Node.js 22 | NanoClaw runtime | Install via nvm on EC2 | -- | nvm handles installation |
| Docker 24+ | Container runtime | `sudo dnf install docker` on EC2 | -- | Required, no fallback |
| gcc/make | better-sqlite3 build | Pre-installed on AL2023 | -- | `sudo dnf install gcc-c++ make` |

**Missing dependencies with no fallback:**
- Docker on EC2: Must be installed. NanoClaw cannot run agents without containers.
- Node.js 22 on EC2: Must be installed via nvm.

**Missing dependencies with fallback:**
- gh CLI: Can fork via GitHub web UI instead.

## Project Constraints (from CLAUDE.md)

- **Tech stack:** NanoClaw (TypeScript/Node.js) -- fork and customize, don't build from scratch
- **Infrastructure:** Minimal AWS EC2 instance -- keep costs low
- **WhatsApp API:** Personal number via NanoClaw's WhatsApp channel skill
- **AI model:** Claude via Anthropic API (NanoClaw default)
- **Process management:** Requirements say "systemd" (INFRA-01), not pm2
- **Instance size:** Requirements specify t3.small (INFRA-05), not t3.micro
- **Test number:** Use test WhatsApp number during development (STATE.md decision)
- **Native EC2:** NanoClaw runs natively on EC2 host, not containerized (STATE.md decision)
- **GSD Workflow:** All changes through GSD commands
- **Immutability:** Create new objects, never mutate existing ones
- **File size:** 200-400 lines typical, 800 max
- **Error handling:** Handle errors explicitly at every level
- **Testing:** vitest already configured in NanoClaw. 80% coverage target.
- **Security:** No hardcoded secrets. Use .env for API keys. Validate inputs.

## Sources

### Primary (HIGH confidence)
- NanoClaw GitHub repo (qwibitai/nanoclaw) -- source code analysis of index.ts, config.ts, env.ts, db.ts, container-runner.ts, container-runtime.ts, group-queue.ts, setup/service.ts
- NanoClaw WhatsApp repo (qwibitai/nanoclaw-whatsapp) -- WhatsApp channel implementation (src/channels/whatsapp.ts)
- NanoClaw `/add-whatsapp` SKILL.md -- complete skill installation flow, auth methods, registration, troubleshooting
- NanoClaw container/Dockerfile -- agent container image specification

### Secondary (MEDIUM confidence)
- NanoClaw README (GitHub) -- architecture overview, philosophy, deployment requirements
- Project CLAUDE.md stack research -- version numbers, alternatives analysis (verified against npm registry at research time)

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - Directly read NanoClaw source code and package.json
- Architecture: HIGH - Analyzed all core source files (index.ts, config.ts, db.ts, container-runner.ts, channels/whatsapp.ts)
- Pitfalls: HIGH - Identified from source code analysis (session handling, memory constraints, Docker group stale detection) and skill troubleshooting docs

**Research date:** 2026-04-01
**Valid until:** 2026-05-01 (NanoClaw is actively developed; check for breaking changes)
