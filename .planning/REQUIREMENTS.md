# Requirements: Nano-Claw WhatsApp Assistant

**Defined:** 2026-04-01
**Core Value:** When tagged in a WhatsApp conversation, the assistant replies to clients accurately and professionally on the consultant's behalf.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Infrastructure

- [x] **INFRA-01**: NanoClaw forked, configured, and running on AWS EC2 t3.small with process management (systemd)
- [ ] **INFRA-02**: WhatsApp channel connected to consultant's personal phone number via NanoClaw skill
- [ ] **INFRA-03**: Per-group context isolation so client conversations never leak between each other
- [ ] **INFRA-04**: EC2 instance runs 24/7 with automatic crash recovery and log rotation
- [x] **INFRA-05**: Container concurrency tuned for t3.small (MAX_CONCURRENT_CONTAINERS=1-2, swap enabled)

### Communication

- [ ] **COMM-01**: User can tag the assistant in any WhatsApp conversation and receive a contextual reply
- [ ] **COMM-02**: Knowledge brief loaded describing consultant's services, pricing, expertise, and typical client interactions
- [ ] **COMM-03**: Assistant replies match the consultant's voice, tone, and communication style
- [ ] **COMM-04**: Assistant gracefully hands off to the consultant when it encounters uncertain, sensitive, or out-of-scope topics
- [ ] **COMM-05**: Assistant never fabricates pricing, commitments, or technical claims not in the knowledge brief

### Scheduling

- [ ] **SCHED-01**: Assistant can check Google Calendar availability when a client asks about meeting times
- [ ] **SCHED-02**: Assistant can create Google Calendar events with client name, time, topic, and meeting link
- [ ] **SCHED-03**: Google Calendar integration uses Service Account auth (no token expiry issues)

### Automation

- [ ] **AUTO-01**: User can instruct the assistant to send recurring messages (e.g., weekly reminders) via NanoClaw's built-in task scheduler

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Communication

- **COMM-06**: Multi-language support (Hebrew/English auto-detection)
- **COMM-07**: Proposal and quote drafting based on standard templates and pricing
- **COMM-08**: Web research to answer client questions beyond training data

### Scheduling

- **SCHED-04**: Smart availability suggestions based on calendar patterns
- **SCHED-05**: Meeting prep summaries sent before scheduled meetings

### Automation

- **AUTO-02**: Conditional follow-ups (send if client hasn't responded within N days)
- **AUTO-03**: Client-specific memory that builds up preferences over time
- **AUTO-04**: Weekly digest summarizing active clients, pending follow-ups, and upcoming meetings

## Out of Scope

| Feature | Reason |
|---------|--------|
| Full autopilot (replying without being tagged) | v1 requires explicit tagging- trust must be built gradually |
| Email integration | WhatsApp only for v1; email is a separate milestone |
| Web dashboard or admin panel | All interaction via WhatsApp; no frontend to maintain |
| CRM integration | 5-10 clients don't need Salesforce; per-group memory suffices |
| Payment processing / invoicing | Liability and compliance risk; consultant has existing workflows |
| Multi-user / team access | Single consultant use only |
| Custom mobile app | WhatsApp IS the interface |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| INFRA-01 | Phase 1 | Complete |
| INFRA-02 | Phase 1 | Pending |
| INFRA-03 | Phase 1 | Pending |
| INFRA-04 | Phase 1 | Pending |
| INFRA-05 | Phase 1 | Complete |
| COMM-01 | Phase 1 | Pending |
| COMM-02 | Phase 2 | Pending |
| COMM-03 | Phase 2 | Pending |
| COMM-04 | Phase 2 | Pending |
| COMM-05 | Phase 2 | Pending |
| SCHED-01 | Phase 3 | Pending |
| SCHED-02 | Phase 3 | Pending |
| SCHED-03 | Phase 3 | Pending |
| AUTO-01 | Phase 3 | Pending |

**Coverage:**
- v1 requirements: 14 total
- Mapped to phases: 14
- Unmapped: 0

---
*Requirements defined: 2026-04-01*
*Last updated: 2026-04-01 after roadmap creation*
