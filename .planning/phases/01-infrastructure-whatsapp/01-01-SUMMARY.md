---
phase: 01-infrastructure-whatsapp
plan: 01
subsystem: infra
tags: [nanoclaw, fork, whatsapp, baileys, ec2, docker, swap]

# Dependency graph
requires: []
provides:
  - NanoClaw fork with WhatsApp skill merged and building
  - .env.example with t3.small tuning
  - EC2 bootstrap scripts (Docker, Node.js 22, swap)
affects: [01-infrastructure-whatsapp]

# Tech tracking
tech-stack:
  added: ["@whiskeysockets/baileys 6.x", "qrcode-terminal", "qrcode"]
  patterns: ["NanoClaw skill merge via git remote"]

key-files:
  created:
    - nanoclaw/setup/ec2-init.sh
    - nanoclaw/setup/swap.sh
  modified:
    - nanoclaw/.env.example
    - nanoclaw/package.json
    - nanoclaw/src/channels/whatsapp.ts

key-decisions:
  - "Used gh repo fork for fork creation with --clone and --remote flags"
  - "Resolved package-lock.json conflict with --theirs strategy during WhatsApp merge"
  - "Used alternate npm cache dir to work around root-owned cache permission issue"

patterns-established:
  - "WhatsApp skill merge: git remote add whatsapp + fetch + merge from nanoclaw-whatsapp repo"
  - "EC2 bootstrap: separate init and swap scripts for modularity"

requirements-completed: [INFRA-01, INFRA-05]

# Metrics
duration: 2min
completed: 2026-04-01
---

# Phase 01 Plan 01: Fork NanoClaw and Prepare Deployment Summary

**NanoClaw forked with WhatsApp skill merged (280 tests pass), .env.example tuned for t3.small, EC2 bootstrap scripts ready**

## Performance

- **Duration:** 2 min
- **Started:** 2026-04-01T14:01:05Z
- **Completed:** 2026-04-01T14:03:39Z
- **Tasks:** 2
- **Files modified:** 5

## Accomplishments
- Forked qwibitai/nanoclaw to michaelfleicher/nanoclaw with WhatsApp skill merged from nanoclaw-whatsapp remote
- TypeScript builds cleanly, all 280 tests pass across 19 test files
- .env.example configured for t3.small deployment (MAX_CONCURRENT_CONTAINERS=2, 30min timeouts, Asia/Jerusalem timezone)
- EC2 bootstrap script installs Docker, Node.js 22 via nvm, verifies build tools, and sets up swap
- Swap script creates and persists a 1 GB swapfile via fallocate and fstab

## Task Commits

Each task was committed atomically:

1. **Task 1: Fork NanoClaw and merge WhatsApp skill** - `252d1bb` (feat)
2. **Task 2: Prepare environment configuration and EC2 bootstrap scripts** - `bf9a096` (feat)

## Files Created/Modified
- `nanoclaw/.env.example` - Environment template with t3.small tuning (ANTHROPIC_API_KEY, ASSISTANT_NAME, TZ, MAX_CONCURRENT_CONTAINERS=2)
- `nanoclaw/setup/ec2-init.sh` - EC2 bootstrap: Docker, Node.js 22, build tools, swap
- `nanoclaw/setup/swap.sh` - 1 GB swapfile creation with fstab persistence
- `nanoclaw/src/channels/whatsapp.ts` - WhatsApp channel class (from merge)
- `nanoclaw/package.json` - Updated with @whiskeysockets/baileys dependency (from merge)

## Decisions Made
- Used `gh repo fork --clone=true --remote=true` for clean fork setup with upstream tracking
- Resolved package-lock.json merge conflict using `--theirs` strategy (WhatsApp remote's lockfile is more complete)
- Used alternate npm cache (`/tmp/npm-cache-nanoclaw`) to work around root-owned `.npm` cache permission issue in local environment

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] npm cache permission error**
- **Found during:** Task 1 (npm install)
- **Issue:** `/Users/michaelfleicher/.npm/_cacache` contained root-owned files, causing EACCES error
- **Fix:** Used `--cache /tmp/npm-cache-nanoclaw` flag to bypass the broken cache
- **Files modified:** None (runtime workaround)
- **Verification:** npm install completed successfully with 336 packages
- **Committed in:** 252d1bb (part of Task 1)

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** Minor workaround for local environment issue. No scope creep.

## Issues Encountered
- npm cache had root-owned files from a previous sudo npm operation. Resolved by using an alternate cache directory.

## User Setup Required
None - no external service configuration required for this plan.

## Next Phase Readiness
- NanoClaw fork is ready with WhatsApp skill merged and building
- EC2 bootstrap scripts are ready for deployment (Plan 02)
- .env.example provides the template for EC2 configuration
- Next step: EC2 instance provisioning and deployment (Plan 02)

## Self-Check: PASSED

- nanoclaw/.env.example: FOUND
- nanoclaw/setup/ec2-init.sh: FOUND
- nanoclaw/setup/swap.sh: FOUND
- nanoclaw/src/channels/whatsapp.ts: FOUND
- Commit 252d1bb: FOUND
- Commit bf9a096: FOUND

---
*Phase: 01-infrastructure-whatsapp*
*Completed: 2026-04-01*
