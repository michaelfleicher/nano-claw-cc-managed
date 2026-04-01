# Phase 2: Knowledge & Communication - Research

**Researched:** 2026-04-01
**Domain:** Knowledge brief authoring, system prompt engineering, communication style matching, handoff behavior
**Confidence:** HIGH

## Summary

Phase 2 transforms the assistant from a generic chatbot into a consultant's digital representative. The work is primarily content and prompt engineering -- not code. NanoClaw already provides the infrastructure: the agent runner loads `groups/global/CLAUDE.md` as a read-only system prompt appendage for all non-main groups, and per-group `CLAUDE.md` files add client-specific context. The phase requires writing a structured knowledge brief (consultant identity, services, pricing, style guide, escalation rules), crafting handoff behavior via explicit prompt instructions, and validating the output against real conversation scenarios.

The key technical insight is that NanoClaw's agent runner appends the global `CLAUDE.md` content to the system prompt using the `claude_code` preset. This means the knowledge brief IS the system prompt customization -- there is no separate "personality configuration" step. Everything the assistant knows and how it behaves is defined by the content of these markdown files. The agent also has access to `send_message` MCP tool for immediate replies and can read conversation history from mounted workspace directories.

**Primary recommendation:** Structure the global CLAUDE.md as a multi-section knowledge brief with explicit sections for identity, services, pricing, communication style (with real message examples), topic boundaries (what to answer vs. hand off), and forbidden behaviors (never fabricate, never commit). Use XML tags within the CLAUDE.md to separate concerns clearly, and keep total brief under 2,000 tokens to control API costs.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| COMM-02 | Knowledge brief loaded describing consultant's services, pricing, expertise, and typical client interactions | Global CLAUDE.md file structure; NanoClaw auto-loads from groups/global/CLAUDE.md into agent system prompt; sections for services, pricing, expertise documented below |
| COMM-03 | Assistant replies match the consultant's voice, tone, and communication style | Few-shot examples in CLAUDE.md; Anthropic prompt engineering guidance on role definition and example-based steering; WhatsApp formatting rules |
| COMM-04 | Assistant gracefully hands off to the consultant when it encounters uncertain, sensitive, or out-of-scope topics | Explicit topic boundary definitions in CLAUDE.md; escalation phrase templates; send_message MCP tool for immediate notification |
| COMM-05 | Assistant never fabricates pricing, commitments, or technical claims not in the knowledge brief | Grounding instructions in system prompt; explicit "NEVER" rules with examples; topic boundary enforcement |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

- **Tech stack**: NanoClaw (TypeScript/Node.js) -- fork and customize, do not build from scratch
- **Infrastructure**: Minimal AWS EC2 instance
- **WhatsApp API**: Personal number via NanoClaw's WhatsApp channel skill
- **AI model**: Claude via Anthropic API (NanoClaw default)
- **Anti-pattern**: Do NOT modify NanoClaw core files (index.ts, container-runner.ts, etc.) -- use extension points (CLAUDE.md files, MCP servers, config overrides)
- **Testing**: vitest for unit/integration tests
- **Coding style**: Immutable patterns, small files, comprehensive error handling

## Standard Stack

### Core (No New Dependencies)

This phase requires NO new libraries or packages. All work is done through NanoClaw's existing extension points.

| Component | Location | Purpose | Why Standard |
|-----------|----------|---------|--------------|
| Global CLAUDE.md | `groups/global/CLAUDE.md` | Knowledge brief -- consultant identity, services, pricing, style, guardrails | NanoClaw agent runner auto-loads this into system prompt for all groups. This IS the customization mechanism. |
| Per-group CLAUDE.md | `groups/whatsapp_{client}/CLAUDE.md` | Client-specific context -- project name, preferences, history notes | NanoClaw mounts group folder read-write into containers. Agent discovers these automatically via `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` env var. |
| send_message MCP tool | Built into NanoClaw | Immediate message delivery during agent execution | Used for handoff notifications ("Let me check with [name]") before the agent finishes processing. |
| Sender allowlist | `~/.config/nanoclaw/sender-allowlist.json` | Restrict who can trigger the assistant | Prevents prompt injection from unknown numbers. JSON file, no code needed. |

### Supporting (Configuration Only)

| Config | Location | Purpose | When to Use |
|--------|----------|---------|-------------|
| ASSISTANT_NAME | `.env` | Trigger pattern (@Name) and self-reference | Set during Phase 1, referenced in knowledge brief |
| MAX_MESSAGES_PER_PROMPT | `.env` | Limits context window per interaction (default: 10) | Keep at 5-7 to control API costs while maintaining conversation context |
| Trigger pattern | NanoClaw config | Regex: `@${ASSISTANT_NAME}` (case-insensitive) | Default behavior, no change needed |

**Installation:** None. This phase is content authoring and configuration.

## Architecture Patterns

### Recommended File Structure

```
groups/
  global/
    CLAUDE.md              # Global knowledge brief (all groups read this)
  whatsapp_main/
    CLAUDE.md              # Main chat rules (consultant's own chat, no trigger needed)
  whatsapp_{client-name}/
    CLAUDE.md              # Client-specific context template
```

### Pattern 1: Structured Knowledge Brief (Global CLAUDE.md)

**What:** A multi-section markdown file that defines the assistant's identity, knowledge, communication style, and behavioral boundaries. Uses XML tags to separate concerns for Claude's parsing.

**When to use:** This is the ONLY pattern for Phase 2. All knowledge and behavior customization goes here.

**Structure:**

```markdown
# [Assistant Name] - [Consultant Name]'s Assistant

<identity>
## Identity

You are [Assistant Name], the AI assistant for [Consultant Name], an independent
AI/Data/Software consultant. You respond to clients on [his/her] behalf when tagged
in WhatsApp conversations.

You ARE [Consultant Name] for the purposes of these conversations. Clients should
feel they are talking to the consultant, not to an AI.
</identity>

<services>
## Services & Pricing

### Consulting Services
- [Service 1]: [Description]. Rate: [price]
- [Service 2]: [Description]. Rate: [price]
- [Service 3]: [Description]. Rate: [price]

### Engagement Types
- [Type 1]: [Description, typical duration, price range]
- [Type 2]: [Description, typical duration, price range]

### What We Do NOT Offer
- [Explicitly list services NOT provided]
</services>

<communication_style>
## Communication Style

### Tone
- [Professional but approachable / Casual but competent / etc.]
- [Specific vocabulary preferences]
- [Formality level]

### Examples of How [Consultant Name] Writes

<example>
Client: "Hi, we're interested in AI consulting for our team"
[Consultant Name]: "[Actual real response example from consultant]"
</example>

<example>
Client: "What would a 2-day workshop cost?"
[Consultant Name]: "[Actual real response example from consultant]"
</example>

<example>
Client: "Can you help with [specific topic]?"
[Consultant Name]: "[Actual real response example from consultant]"
</example>

### Formatting Rules (WhatsApp)
- Use *bold* for emphasis (single asterisks)
- Use _italic_ for titles or soft emphasis (underscores)
- NO double asterisks (**bold**) -- WhatsApp does not render them
- NO markdown links [text](url) -- WhatsApp does not render them
- Keep messages concise (2-4 short paragraphs max)
- Use line breaks between paragraphs for readability
</communication_style>

<boundaries>
## Topic Boundaries

### Topics You CAN Handle (respond directly)
- General questions about services and expertise
- Pricing inquiries (using exact prices from this brief)
- Scheduling requests (Phase 3 -- for now, say "Let me check my calendar and get back to you")
- Follow-up on existing conversations
- General AI/Data/Software questions within expertise

### Topics You MUST Hand Off (do NOT answer)
- Custom pricing negotiations or discounts
- Scope changes to active projects
- Legal, contractual, or NDA-related questions
- Complaints or disputes
- Anything involving money, invoices, or payments
- Technical commitments not listed in this brief
- Questions you are uncertain about

### Handoff Phrase
When handing off, use this pattern:
"Let me check with [Consultant Name] on that and get back to you."
or
"That's something [Consultant Name] will want to handle personally. I'll flag it for [him/her]."

NEVER say "I'm an AI" or "I don't have that information." Always frame it as
"[Consultant Name] will handle this personally."
</boundaries>

<rules>
## Absolute Rules (NEVER violate)

1. NEVER fabricate pricing. Only quote prices explicitly listed in this brief.
2. NEVER make technical commitments or delivery promises not in this brief.
3. NEVER claim availability or schedule without checking (Phase 3 adds calendar).
4. NEVER discuss other clients' projects or information.
5. NEVER reveal you are an AI unless directly and specifically asked.
6. NEVER use standard markdown formatting -- use WhatsApp formatting only.
7. If uncertain about ANY factual claim, hand off to the consultant.
</rules>
```

**Confidence:** HIGH -- This structure follows Anthropic's documented best practices for system prompts (clear role definition, few-shot examples, explicit boundaries, XML tags for structured sections). NanoClaw's agent runner auto-loads this content.

### Pattern 2: Client-Specific Context (Per-Group CLAUDE.md)

**What:** A lightweight markdown file in each client's group folder that adds context about that specific client relationship.

**When to use:** When registering a new client group.

**Structure:**

```markdown
# Client: [Client Name]

## Project
- Project name: [name]
- Started: [date]
- Current phase: [description]

## Preferences
- Preferred meeting day: [day]
- Communication language: [English/Hebrew/etc.]
- Key contacts: [names and roles]

## Notes
- [Any relevant context about this client]

## History
- [Key milestones or decisions]
```

**How it works:** NanoClaw mounts the group folder at `/workspace/group` (read-write) and sets `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1`. The Claude Agent SDK automatically discovers and loads CLAUDE.md files from these directories. The agent sees both the global brief AND the client-specific brief.

### Pattern 3: Handoff Behavior via Prompt Boundaries

**What:** Define clear topic boundaries in the knowledge brief so the agent knows when to answer vs. when to escalate.

**When to use:** Every interaction. The boundaries are always active.

**How it works in NanoClaw:**
1. The agent receives the message (formatted as XML: `<message sender="ClientName" time="...">text</message>`)
2. Agent reads CLAUDE.md files (global + group-specific)
3. Agent decides based on boundary rules: respond directly or hand off
4. For handoff: agent uses `send_message` MCP tool to immediately reply with the handoff phrase
5. Agent can also use `<internal>` tags for reasoning that gets logged but not sent to the client

**Key insight:** There is no separate "handoff mechanism" to build. The handoff IS the response. The agent simply replies with the handoff phrase instead of attempting to answer. The prompt boundaries in CLAUDE.md are what make this work -- Claude is excellent at following explicit behavioral rules when they are clearly stated.

### Anti-Patterns to Avoid

- **Overly long knowledge brief:** Keep under 2,000 tokens. Every token in the system prompt is sent with EVERY interaction, multiplying API costs. Be concise. Use sections the agent can reference, not walls of text.
- **Vague style instructions:** "Be professional" tells Claude nothing. Provide 3-5 actual message examples from the consultant. Few-shot examples are the most reliable way to match tone (per Anthropic docs).
- **Implicit boundaries:** "Use good judgment about what to answer" will lead to hallucination. Explicitly list topics the agent CAN handle and topics it MUST hand off. Binary rules beat judgment calls.
- **Forgetting WhatsApp formatting:** Standard markdown (double asterisks, links) does not render in WhatsApp. The brief MUST specify WhatsApp-specific formatting rules or the agent will produce unreadable messages.
- **Testing with ideal inputs only:** Real clients send typos, partial messages, voice-to-text garbage, and multi-message streams. Test with messy, realistic inputs.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Knowledge ingestion | Custom file parser, vector DB, RAG pipeline | `groups/global/CLAUDE.md` plain text | NanoClaw auto-loads CLAUDE.md into system prompt. Claude's context window handles 2K tokens of knowledge trivially. No retrieval needed for a single consultant's service catalog. |
| Style matching | Fine-tuned model, style transfer system | Few-shot examples in CLAUDE.md | Anthropic docs confirm 3-5 examples reliably steer tone. Fine-tuning is overkill and unavailable for Claude. |
| Topic routing | Classification model, intent detection, rules engine | Explicit boundary lists in CLAUDE.md | Claude is excellent at following explicit rules. A bulleted list of "handle" vs. "hand off" topics is more maintainable than code. |
| Handoff mechanism | Webhook, notification system, separate alert channel | Agent reply with handoff phrase | The handoff IS the WhatsApp reply. No infrastructure needed -- the agent simply responds differently. |
| Client context | CRM, database schema, entity extraction | Per-group CLAUDE.md files | NanoClaw's per-group isolation already provides client separation. A markdown file per client is the right abstraction at 5-10 clients. |

**Key insight:** Phase 2 is a content authoring phase, not a coding phase. The temptation is to build systems -- resist it. NanoClaw already has the infrastructure. Write the knowledge brief, write client templates, configure the allowlist, and test.

## Common Pitfalls

### Pitfall 1: Knowledge Brief Too Long (Token Cost Spiral)

**What goes wrong:** Consultant dumps everything they know into CLAUDE.md -- 5,000+ tokens of services, case studies, blog posts, detailed pricing tables. Every single WhatsApp interaction now costs 5-10x more in API tokens because the entire brief is in the system prompt.
**Why it happens:** Natural instinct is "more knowledge = better answers."
**How to avoid:** Hard limit of 2,000 tokens for global CLAUDE.md. Structure as reference sections, not narrative. Use the `MAX_MESSAGES_PER_PROMPT=5-7` setting to keep total prompt size manageable. Monitor token usage per interaction after deployment.
**Warning signs:** API costs exceeding $0.50/day for 10-20 interactions. Each interaction using 5K+ input tokens.

### Pitfall 2: Generic Style Instructions

**What goes wrong:** Brief says "Be professional and friendly." Agent produces generic corporate chatbot replies. Clients notice the difference from the consultant's actual communication style.
**Why it happens:** Describing communication style in the abstract is hard. Most people cannot articulate their own style rules.
**How to avoid:** Collect 10-15 real WhatsApp messages from the consultant (sanitize client names). Include 3-5 as few-shot examples in the `<communication_style>` section. Anthropic's prompt engineering docs confirm that examples are "one of the most reliable ways to steer Claude's output format, tone, and structure."
**Warning signs:** Consultant reads test responses and says "I would never say it that way."

### Pitfall 3: Hallucinated Pricing and Commitments

**What goes wrong:** Client asks about a service not in the brief. Agent improvises a plausible-sounding price or timeline. Consultant is now committed to something they did not offer.
**Why it happens:** Without explicit "NEVER fabricate" rules, Claude tries to be helpful and makes reasonable guesses.
**How to avoid:** The `<rules>` section must explicitly state: "NEVER fabricate pricing. Only quote prices explicitly listed in this brief." The `<boundaries>` section must list topics that require handoff. Double-defense: explicit rules AND explicit topic boundaries.
**Warning signs:** Agent responds to pricing questions with hedging language ("typically around...", "usually...") instead of quoting exact figures or handing off.

### Pitfall 4: WhatsApp Formatting Breaks

**What goes wrong:** Agent uses standard markdown (`**bold**`, `[links](url)`, `## headers`) in WhatsApp messages. Client sees raw markup characters instead of formatted text.
**Why it happens:** Claude defaults to standard markdown. NanoClaw's default global CLAUDE.md includes WhatsApp formatting rules, but if overwritten or ignored, formatting breaks.
**How to avoid:** Include WhatsApp-specific formatting rules in the knowledge brief. The NanoClaw default CLAUDE.md already documents this: single asterisks for bold, underscores for italic, no markdown links. Preserve these rules when writing the custom brief.
**Warning signs:** Test messages contain `**bold**` or `[text](url)` patterns.

### Pitfall 5: Boundary Ambiguity Leading to Inconsistent Handoffs

**What goes wrong:** Some topics are not clearly in "handle" or "hand off" lists. Agent sometimes answers, sometimes hands off the same type of question. Client experience is inconsistent.
**Why it happens:** Topic boundaries are written too broadly ("sensitive topics") instead of specifically ("pricing negotiations, contract changes, complaints").
**How to avoid:** Write specific, concrete topic lists. Test each boundary with 2-3 example questions. If a topic is ambiguous, default to handoff -- it is always safer to escalate than to fabricate.
**Warning signs:** Same type of question gets different handling in different test runs.

### Pitfall 6: Knowledge Brief Drift

**What goes wrong:** Services change, pricing updates, brief stays the same. Agent confidently quotes outdated pricing.
**Why it happens:** No review process. Brief is written once and forgotten.
**How to avoid:** Add a "last reviewed" date and "valid until" field at the top of the brief. Version the brief in git. Set a monthly review reminder (Phase 3 can automate this via scheduled task). Structure pricing in a single, easy-to-update section.
**Warning signs:** Consultant changes their rates but forgets to update the brief.

## Code Examples

### Example 1: NanoClaw Message Flow (How the Agent Sees Messages)

The agent receives messages formatted as XML by NanoClaw's router:

```xml
<context timezone="Asia/Jerusalem" />
<messages>
<message sender="John Smith" time="2026-04-01 14:30">Hi, I'm interested in your AI consulting services. What kind of workshops do you offer?</message>
<message sender="John Smith" time="2026-04-01 14:31">And what would the pricing look like for a 2-day engagement?</message>
</messages>
```

The agent reads this, consults the CLAUDE.md knowledge brief, and responds. If pricing is in the brief, it quotes it directly. If not, it hands off.

### Example 2: send_message MCP Tool Usage (Agent Handoff)

When the agent encounters a handoff topic, it uses the send_message MCP tool:

```
Agent reasoning (in <internal> tags, not sent to client):
"The client is asking about a custom discount. This falls under 'pricing negotiations'
in my handoff list. I should not attempt to negotiate -- hand off to the consultant."

Agent action:
mcp__nanoclaw__send_message(text="That's a great question about custom pricing.
Let me check with [Consultant Name] on that and get back to you shortly.")
```

### Example 3: Sender Allowlist Configuration

```json
{
  "allowedSenders": [
    "972501234567@s.whatsapp.net",
    "972509876543@s.whatsapp.net"
  ]
}
```

Place at `~/.config/nanoclaw/sender-allowlist.json`. Only messages from these numbers trigger the assistant. Others are stored but ignored.

### Example 4: Per-Group CLAUDE.md Template

```markdown
# Client: Acme Corp

## Project
- Project name: AI Strategy Workshop
- Started: 2026-03-15
- Current phase: Discovery
- Key deliverable: AI readiness assessment report

## Preferences
- Primary contact: Sarah (VP Engineering)
- Preferred meeting times: Mornings (9-12)
- Communication style: Direct and technical

## Important Context
- Budget approved for Q2 only
- Previous vendor failed to deliver -- they are cautious about promises
- Sarah reports to CTO David who has final approval
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| RAG with vector DB for knowledge | Direct system prompt injection | Always (for small knowledge bases) | For a single consultant's service catalog (~2K tokens), RAG adds complexity with zero benefit. System prompt injection is simpler and more reliable. |
| Separate intent classification for routing | Explicit boundary rules in system prompt | Claude 3+ (2024) | Claude follows explicit rule lists reliably. No need for a separate classification step. |
| Fine-tuning for style matching | Few-shot examples in prompt | Always (fine-tuning unavailable for Claude) | 3-5 examples in the system prompt reliably steer tone without any model training. |
| Prefilled assistant responses for format control | Explicit formatting instructions | Claude 4.6 (2026) | Prefills deprecated in Claude 4.6. Use explicit instructions instead. |

## Open Questions

1. **Consultant message samples not yet available**
   - What we know: The knowledge brief needs 10-15 real WhatsApp messages from the consultant to extract communication style and create few-shot examples.
   - What's unclear: The consultant has not provided these samples yet. The style guide section cannot be finalized without them.
   - Recommendation: Phase 2 planning should include a task to collect and sanitize message samples from the consultant. This is a blocking dependency for COMM-03. The knowledge brief structure can be written with placeholder examples, then updated when real samples arrive.

2. **Optimal knowledge brief length vs. API cost tradeoff**
   - What we know: More context = better answers but higher cost. Anthropic charges per input token. System prompt is sent with every interaction.
   - What's unclear: The exact sweet spot for this consultant's use case. Too short and answers are vague; too long and costs spike.
   - Recommendation: Start with 1,500-2,000 tokens. Monitor API costs for the first week. Trim if costs exceed $0.50/day at 10-20 interactions/day. This is an empirical tuning problem, not a design decision.

3. **Hebrew/English language handling**
   - What we know: The consultant may have clients who communicate in Hebrew. Claude handles multilingual conversations natively.
   - What's unclear: Whether the knowledge brief should include Hebrew examples, or if Claude can translate the style naturally.
   - Recommendation: For v1, write the knowledge brief in English only. Add a single instruction: "Match the language of the client's message. If the client writes in Hebrew, respond in Hebrew." Claude handles this without additional prompting. Defer Hebrew-specific examples to v2 (COMM-06).

## Sources

### Primary (HIGH confidence)
- **NanoClaw source code** (qwibitai/nanoclaw) -- agent-runner/src/index.ts (system prompt construction), ipc-mcp-stdio.ts (MCP tools), container-runner.ts (volume mounts), config.ts (env vars), router.ts (message formatting)
- **NanoClaw groups/global/CLAUDE.md** -- default knowledge brief structure with WhatsApp formatting rules, send_message usage, scheduling patterns
- **Anthropic prompt engineering docs** (platform.claude.com) -- role definition, few-shot examples, XML tag structuring, explicit instructions, output formatting guidance

### Secondary (MEDIUM confidence)
- **Project research** (.planning/research/) -- architecture patterns, feature landscape, pitfalls, stack recommendations
- **CLAUDE.md project instructions** -- stack constraints, anti-patterns, deployment approach

### Tertiary (LOW confidence)
- **Knowledge brief token cost estimates** -- based on Anthropic pricing docs and estimated interaction volume; actual costs depend on real usage patterns

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- NanoClaw's extension points (CLAUDE.md, MCP tools, sender allowlist) are verified from source code
- Architecture: HIGH -- Knowledge brief pattern is NanoClaw's documented customization approach; prompt engineering patterns from Anthropic official docs
- Pitfalls: HIGH -- Token cost, formatting, hallucination risks are well-documented in AI assistant deployment literature and Anthropic docs

**Research date:** 2026-04-01
**Valid until:** 2026-05-01 (stable -- content authoring patterns do not change rapidly)
