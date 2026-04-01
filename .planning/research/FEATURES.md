# Feature Landscape

**Domain:** WhatsApp AI assistant for an independent AI/Data/Software consultant
**Researched:** 2026-04-01
**Confidence:** MEDIUM-HIGH (based on NanoClaw docs, WhatsApp business AI ecosystem research, and consultant workflow analysis)

## Table Stakes

Features the consultant will expect to work from day one. Missing any of these and the assistant provides no value over just typing the replies himself.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **Tag-to-reply in WhatsApp** | Core value proposition -- assistant does nothing without this. Consultant tags the bot in a group or DM, it replies on his behalf. | Low | NanoClaw handles this natively via WhatsApp channel skill. Just needs configuration. |
| **Knowledge brief ingestion** | Without knowing services, pricing, and typical answers, replies will be generic and useless. Consultant must be able to load a written brief that shapes all responses. | Low | NanoClaw supports per-group `CLAUDE.md` files for persistent context. A global knowledge brief can be placed in the container filesystem. |
| **Consultant voice/style matching** | Clients know this person. If the assistant sounds like a generic chatbot, trust erodes immediately. Must match tone, formality level, vocabulary. | Medium | Requires careful prompt engineering in the system prompt. The knowledge brief should include style examples (real message samples). Not a code problem, a content problem. |
| **Google Calendar availability check** | "When are you free this week?" is the most common client question. If the assistant cannot check real availability, the consultant still has to intervene for every scheduling request. | Medium | Requires Google Calendar API integration as a tool the agent can call. NanoClaw supports custom tools via container filesystem. OAuth2 setup is the hard part. |
| **Meeting scheduling** | Checking availability is only half the value. The assistant must also create calendar events with the client's details (time, topic, link). | Medium | Extension of calendar availability check. Needs write access to Google Calendar API. Must handle timezone correctly. |
| **Multi-conversation context isolation** | Consultant has 5-10 active clients. The assistant must never leak Client A's information into Client B's conversation. | Low | NanoClaw's architecture already provides this -- per-group containers with isolated filesystems. Verify this works correctly; do not build custom isolation. |
| **Graceful handoff / uncertainty handling** | When the assistant does not know something or the question is sensitive (pricing negotiation, project scope change), it must say "Let me check with [consultant name] and get back to you" rather than hallucinate or commit to something wrong. | Medium | Prompt engineering problem. The system prompt must define clear boundaries: what topics the assistant can handle autonomously vs. what requires human escalation. |
| **Always-on availability** | Clients message at any hour. The assistant must be running 24/7, not just during work hours. | Low | EC2 deployment with process management (systemd or PM2). NanoClaw is designed as a long-running process. |

## Differentiators

Features that elevate this from "auto-responder" to "genuinely useful AI assistant." Not expected on day one, but each one meaningfully reduces consultant workload.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Scheduled follow-ups** | Consultant tells the assistant "Follow up with Client X on Friday if they haven't responded about the proposal." Assistant sends the follow-up automatically. Prevents dropped balls. | Medium | NanoClaw has built-in task scheduling ("send X every weekday at 9am"). Needs adaptation for conditional follow-ups (check if client responded before sending). |
| **Meeting prep summaries** | Before a scheduled meeting, the assistant sends the consultant a summary of recent conversation with that client -- open questions, last discussed topics, action items. Saves 10-15 min of scrolling per meeting. | Medium | Requires combining calendar data (upcoming meetings) with conversation history. NanoClaw's per-group memory makes history accessible. Needs a scheduled task that triggers before meetings. |
| **Client-specific memory / preferences** | Assistant remembers that Client A prefers Tuesday meetings, Client B's project is called "Atlas", Client C always asks about AI training data. Builds up over time, makes responses increasingly personalized. | Low-Medium | NanoClaw's per-group `CLAUDE.md` files already support this. The assistant can be instructed to update these files with key facts after conversations. |
| **Proposal / quote drafting** | Consultant tags the assistant and says "Draft a quick quote for 3 days of AI strategy consulting." Assistant produces a formatted proposal based on standard pricing and templates. | Medium | Requires a proposal template in the knowledge brief plus pricing rules. Output needs to be good enough to send with minor edits, not just a starting point. |
| **Smart availability suggestions** | Instead of just listing free slots, the assistant suggests optimal meeting times based on patterns: "You usually prefer mornings for client calls. How about Tuesday at 10am or Wednesday at 9am?" | Medium | Requires reading calendar patterns, not just free/busy status. Can be built incrementally -- start with simple free slot listing, add intelligence later. |
| **Weekly digest / activity summary** | Every Sunday evening or Monday morning, the assistant sends a summary: active clients, pending follow-ups, meetings this week, unanswered messages. Replaces the consultant's manual review. | Medium | Scheduled task combining calendar, conversation history, and pending items. High value for a consultant managing multiple clients. |
| **Web research on behalf of client** | Client asks "What's the latest on EU AI Act compliance for our sector?" The assistant can search the web and provide a researched answer, not just training data. | Low | NanoClaw already has web access (search + fetch). Just needs to be enabled and the assistant prompted to use it for research questions. |
| **Recurring task automation** | "Every Monday, remind Client X about the weekly standup link" or "Every month, send Client Y an invoice reminder." Consultant sets it once, assistant handles it forever. | Low | NanoClaw's scheduler handles this natively. Mainly a matter of giving the assistant clear instructions on what to send and when. |
| **Multi-language support** | Some clients may communicate in Hebrew, others in English. The assistant should detect and respond in the appropriate language. | Low | Claude handles multilingual conversations natively. The knowledge brief should include examples in both languages. No custom code needed, just prompt instruction. |

## Anti-Features

Features to deliberately NOT build. Each temptation comes with a reason to resist.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| **Full autopilot (replying without being tagged)** | The consultant explicitly scoped this out. Unsolicited AI replies in client conversations would be jarring, potentially embarrassing, and could commit to things the consultant hasn't approved. Trust must be built gradually. | Keep tag-to-reply as the only trigger. Revisit autopilot only after months of validated tag-based usage. |
| **Web dashboard or admin panel** | Adds a whole frontend layer with auth, hosting, and maintenance. The consultant manages everything via WhatsApp -- that is the entire point. A dashboard splits attention. | All configuration via WhatsApp commands or config files on EC2. NanoClaw is designed for this. |
| **CRM integration** | For 5-10 clients, a CRM is overhead. The per-group memory in NanoClaw already acts as a lightweight CRM. Integrating Salesforce/HubSpot adds complexity with no proportional value at this scale. | Use per-group CLAUDE.md files as client context. If the consultant outgrows this, revisit. |
| **Email channel** | Explicitly out of scope for v1. Adding email means a second channel to monitor, different formatting rules, and email-specific quirks (threading, signatures, attachments). | WhatsApp only. Email can be a future milestone after WhatsApp is solid. |
| **Payment processing / invoicing** | Mixing financial transactions into a WhatsApp bot is a liability and compliance headache. The consultant has existing invoicing workflows. | The assistant can remind about invoicing or draft invoice text, but never handle actual money. |
| **Team / multi-user access** | Scoped to a single consultant. Multi-user adds auth, permissions, shared state, conflict resolution -- all for zero current value. | Single-user only. If the consultant hires, this becomes a new project, not a feature. |
| **Custom mobile app** | WhatsApp IS the interface. Building a separate app duplicates functionality and splits the user experience. | Everything happens in WhatsApp. The assistant is a participant in conversations, not a separate tool. |
| **Complex workflow builder / no-code automation** | The consultant is technical. A visual workflow builder adds UI complexity for something that can be done with natural language instructions to the assistant. | Use natural language: "Every Friday, follow up with clients who haven't responded this week." NanoClaw's scheduling + Claude's language understanding handles this. |

## Feature Dependencies

```
Knowledge Brief Ingestion
  |
  +---> Tag-to-Reply (needs knowledge to reply well)
  |       |
  |       +---> Voice/Style Matching (refines how replies sound)
  |       |
  |       +---> Graceful Handoff (needs to know what it CAN'T answer)
  |
  +---> Proposal Drafting (needs pricing + service knowledge)

Google Calendar API (OAuth2 setup)
  |
  +---> Availability Check
  |       |
  |       +---> Meeting Scheduling (write access extends read access)
  |       |       |
  |       |       +---> Meeting Prep Summaries (needs calendar + conversation data)
  |       |
  |       +---> Smart Availability Suggestions (extends basic availability)

Per-Group Context Isolation (NanoClaw built-in)
  |
  +---> Client-Specific Memory (extends isolation with persistent facts)
  |
  +---> Conversation History Access
          |
          +---> Meeting Prep Summaries
          |
          +---> Weekly Digest

NanoClaw Task Scheduler (built-in)
  |
  +---> Recurring Task Automation
  |
  +---> Scheduled Follow-ups (adds conditional logic)
  |
  +---> Weekly Digest (scheduled summary generation)
```

## MVP Recommendation

Prioritize (Phase 1 -- get the core loop working):
1. **NanoClaw deployment on EC2** -- nothing works without the foundation running
2. **WhatsApp channel connection** -- connect to the consultant's personal number
3. **Knowledge brief ingestion** -- load services, pricing, style examples
4. **Tag-to-reply with voice matching** -- the core value proposition
5. **Graceful handoff for unknown topics** -- prevent embarrassing hallucinations

Prioritize (Phase 2 -- scheduling, the second biggest time-saver):
6. **Google Calendar availability check** -- answer "when are you free?"
7. **Meeting scheduling** -- create events, not just list slots

Prioritize (Phase 3 -- automation and intelligence):
8. **Recurring task automation** -- leverage NanoClaw's built-in scheduler
9. **Scheduled follow-ups** -- prevent dropped client conversations
10. **Client-specific memory** -- make responses improve over time

Defer:
- **Meeting prep summaries**: High value but requires solid calendar + memory foundation first
- **Weekly digest**: Nice to have, build after follow-ups work
- **Proposal drafting**: Valuable but not urgent; the consultant can draft proposals himself initially
- **Smart availability suggestions**: Refinement on top of basic scheduling; premature optimization early on

## Sources

- NanoClaw GitHub repository (https://github.com/qwibitai/nanoclaw) -- architecture, built-in capabilities, per-group isolation, scheduling [HIGH confidence]
- PROJECT.md -- consultant requirements, constraints, out-of-scope items [HIGH confidence]
- WATI WhatsApp AI chatbot guide (wati.io) -- business chatbot feature landscape [MEDIUM confidence]
- Trengo WhatsApp AI chatbot guide (trengo.com) -- knowledge base, escalation, CRM integration patterns [MEDIUM confidence]
- Domain knowledge of consultant workflows and WhatsApp business communication patterns [MEDIUM confidence]
