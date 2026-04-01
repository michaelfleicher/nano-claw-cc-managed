# Nano-Claw WhatsApp Assistant

A personal AI assistant for an AI/Data/Software consultant, built on [NanoClaw](https://github.com/qwibitai/nanoclaw). Connected to the consultant's personal WhatsApp number, it replies to clients on his behalf when tagged, schedules meetings via Google Calendar, and handles repeated tasks.

## Features

- **Tag-to-reply** - Assistant responds when tagged in WhatsApp conversations
- **Knowledge-driven** - Loaded with a brief about services, pricing, and communication style
- **Calendar integration** - Checks availability and schedules meetings via Google Calendar
- **Voice matching** - Replies in the consultant's tone and style
- **Recurring tasks** - Handles follow-ups, reminders, and other repeated work

## Tech Stack

- **Framework**: [NanoClaw](https://github.com/qwibitai/nanoclaw) (TypeScript/Node.js, Claude Agent SDK)
- **AI Model**: Claude via Anthropic API
- **Infrastructure**: AWS EC2 (minimal instance, always-on)
- **Database**: SQLite (NanoClaw default)
- **WhatsApp**: Connected via NanoClaw's WhatsApp channel skill
- **Calendar**: Google Calendar API

## Architecture

```
WhatsApp (personal number)
    │
    ▼
NanoClaw (EC2 instance)
    ├── Claude Agent SDK (AI reasoning)
    ├── Knowledge Brief (services, pricing, style)
    ├── Google Calendar (scheduling)
    └── SQLite (conversation history)
```

## Getting Started

### Prerequisites

- AWS account with EC2 access
- Anthropic API key
- Google Calendar API credentials
- WhatsApp Business API access

### Setup

1. Fork and clone [NanoClaw](https://github.com/qwibitai/nanoclaw)
2. Configure environment variables (API keys, WhatsApp credentials)
3. Load the knowledge brief
4. Deploy to EC2
5. Connect WhatsApp channel via `/add-whatsapp`

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| NanoClaw as foundation | Small codebase (~3.9k lines), built for individual users, WhatsApp support via skills |
| EC2 over serverless | Long-running Node.js process with SQLite needs persistent compute |
| Tag-to-reply (not autopilot) | Keeps the consultant in control — assistant only acts when explicitly invoked |
| WhatsApp first | Primary client channel; email can be added later |

## Scope

### In Scope (v1)

- WhatsApp integration with personal phone number
- AI responses when tagged in conversations
- Google Calendar scheduling
- Knowledge brief for accurate client communication
- Recurring task handling

### Out of Scope (v1)

- Email integration
- Full autopilot (responding without being tagged)
- Mobile app or web dashboard
- Multi-user / team access

## License

Private repository.
