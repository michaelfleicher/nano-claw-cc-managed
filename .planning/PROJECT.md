# Nano-Claw WhatsApp Assistant

## What This Is

A personal AI assistant for an AI/Data/Software consultant, built on [NanoClaw](https://github.com/qwibitai/nanoclaw) and deployed on a minimal AWS EC2 instance. Connected to the consultant's personal WhatsApp number, it replies to clients on his behalf when tagged, schedules meetings via Google Calendar, and handles repeated tasks- freeing up time from day-to-day client communication.

## Core Value

When tagged in a WhatsApp conversation, the assistant replies to clients accurately and professionally on the consultant's behalf, using knowledge of his services and communication style.

## Requirements

### Validated

(None yet- ship to validate)

### Active

- [ ] NanoClaw forked, configured, and running on AWS EC2
- [ ] WhatsApp channel connected to personal phone number
- [ ] Assistant responds when tagged in WhatsApp conversations
- [ ] Knowledge brief loaded so assistant understands services, pricing, and typical client interactions
- [ ] Google Calendar integration- assistant checks availability and schedules meetings
- [ ] Assistant replies in the consultant's voice/style
- [ ] Repeated tasks- assistant can be told to do recurring work (follow-ups, reminders)
- [ ] Minimal EC2 instance (cost-efficient, always-on)

### Out of Scope

- Email integration- WhatsApp only for v1
- Full autopilot (answering without being tagged)- v1 requires explicit tagging
- Mobile app or web dashboard- interact via WhatsApp only
- Multi-user / team access- single consultant use only

## Context

- **Business:** AI, Data, and Software consulting. Helps organizations integrate AI into daily operations and develops custom solutions.
- **Client volume:** 5-10 active client conversations at any time, high-touch interactions.
- **Current workflow:** All client communication via WhatsApp and email. WhatsApp is the priority channel.
- **NanoClaw:** Open-source personal AI agent framework (TypeScript/Node.js, Claude Agent SDK, Docker containers, SQLite). Designed to be forked and customized. Connects to WhatsApp via skills (`/add-whatsapp`). ~3,900 lines, <10 dependencies.
- **Infrastructure:** AWS account already set up. Needs a minimal EC2 instance (t3.micro or similar).
- **Calendar:** Google Calendar for scheduling.
- **Knowledge source:** Consultant will provide a written brief about services, typical answers, and communication style.

## Constraints

- **Tech stack**: NanoClaw (TypeScript/Node.js)- fork and customize, don't build from scratch
- **Infrastructure**: Minimal AWS EC2 instance- keep costs low
- **WhatsApp API**: Personal number via NanoClaw's WhatsApp channel skill
- **AI model**: Claude via Anthropic API (NanoClaw default)
- **Calendar**: Google Calendar API for availability and scheduling

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Use NanoClaw as foundation | Small codebase (~3.9k lines), built for individual users, WhatsApp support via skills, container isolation for security |- Pending |
| Deploy on EC2 (not serverless) | NanoClaw is a long-running Node.js process with SQLite- needs persistent compute, not Lambda |- Pending |
| Tag-to-reply (not full autopilot) | v1 keeps the consultant in control- assistant only acts when explicitly invoked |- Pending |
| WhatsApp first, email later | WhatsApp is the primary client channel, email can be added in a future milestone |- Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check- still the right priority?
3. Audit Out of Scope- reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-01 after initialization*
