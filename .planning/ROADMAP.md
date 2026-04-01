# Roadmap: Nano-Claw WhatsApp Assistant

## Overview

This roadmap delivers a personal WhatsApp AI assistant for an independent consultant, built on NanoClaw and deployed to AWS EC2. The journey moves from stable infrastructure (fork, deploy, connect WhatsApp) through consultant identity and communication intelligence (knowledge brief, style matching, handoff behavior) to calendar integration and task automation (Google Calendar MCP, recurring tasks). Each phase delivers a complete, verifiable capability on top of the previous one.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Infrastructure & WhatsApp** - NanoClaw forked, deployed on EC2, WhatsApp connected and stable
- [ ] **Phase 2: Knowledge & Communication** - Knowledge brief loaded, assistant replies in consultant's voice with safe handoff behavior
- [ ] **Phase 3: Calendar & Automation** - Google Calendar integration for scheduling, plus recurring task automation

## Phase Details

### Phase 1: Infrastructure & WhatsApp
**Goal**: NanoClaw runs reliably on EC2 with WhatsApp connected, so the assistant can receive and respond to tagged messages in client conversations
**Depends on**: Nothing (first phase)
**Requirements**: INFRA-01, INFRA-02, INFRA-03, INFRA-04, INFRA-05, COMM-01
**Success Criteria** (what must be TRUE):
  1. User can tag the assistant in a WhatsApp group and receive a reply (using test number)
  2. The EC2 instance stays running 24/7 and automatically recovers from crashes without manual intervention
  3. Messages in one client group never appear in another client group's context
  4. The NanoClaw process runs within the memory constraints of a t3.small instance without OOM kills
**Plans**: 3 plans

Plans:
- [ ] 01-01-PLAN.md -- Fork NanoClaw, merge WhatsApp skill, prepare configuration and EC2 scripts
- [ ] 01-02-PLAN.md -- Provision EC2 and deploy NanoClaw with systemd, Docker, and swap
- [ ] 01-03-PLAN.md -- Connect WhatsApp, register groups, verify end-to-end message flow and isolation

### Phase 2: Knowledge & Communication
**Goal**: The assistant understands the consultant's services, pricing, and communication style, and replies to clients as if it were the consultant
**Depends on**: Phase 1
**Requirements**: COMM-02, COMM-03, COMM-04, COMM-05
**Success Criteria** (what must be TRUE):
  1. When asked about the consultant's services or pricing, the assistant provides accurate answers from the knowledge brief
  2. Replies match the consultant's tone and communication style (validated by consultant reviewing sample conversations)
  3. When the assistant encounters a question outside its knowledge or a sensitive topic, it explicitly hands off to the consultant rather than guessing
  4. The assistant never fabricates pricing, availability claims, or technical commitments not present in the knowledge brief
**Plans**: 2 plans

Plans:
- [ ] 02-01-PLAN.md -- Write global knowledge brief and client context template
- [ ] 02-02-PLAN.md -- Validation tests and consultant review checkpoint

### Phase 3: Calendar & Automation
**Goal**: The assistant can check availability, schedule meetings on Google Calendar, and execute recurring tasks on behalf of the consultant
**Depends on**: Phase 2
**Requirements**: SCHED-01, SCHED-02, SCHED-03, AUTO-01
**Success Criteria** (what must be TRUE):
  1. When a client asks "when are you free?", the assistant checks Google Calendar and responds with actual available time slots
  2. The assistant can create a Google Calendar event with client name, time, topic, and meeting link from a WhatsApp conversation
  3. Google Calendar authentication uses a service account that does not require periodic manual re-authorization
  4. User can instruct the assistant to send a recurring message (e.g., weekly reminder) and it executes on schedule
**Plans**: TBD

Plans:
- [ ] 03-01: TBD
- [ ] 03-02: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Infrastructure & WhatsApp | 0/3 | Planned | - |
| 2. Knowledge & Communication | 0/2 | Planned | - |
| 3. Calendar & Automation | 0/0 | Not started | - |
