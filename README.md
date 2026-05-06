# Live Chat Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source live chat platform with real-time messaging, intelligent routing, and CRM integration.

Live Chat Platform is a real-time customer conversation system for support, sales, and customer success teams. It combines WebSocket-based messaging with AI assist, intent-aware routing, and CRM synchronisation, targeting teams that find incumbents like Intercom too expensive and tools like Chatwoot too thin on AI capability.

---

## Why Live Chat Platform?

- Mid-market and enterprise live chat is dominated by Intercom at $74+/seat/month and custom-priced enterprise tiers, putting modern AI-native chat out of reach for many teams.
- Open-source options (Chatwoot) offer self-hosting and freedom from vendor lock-in but lag noticeably on AI capabilities such as autonomous resolution and intent scoring.
- Tools like LiveChat, Tidio, and HubSpot Live Chat are affordable but offer limited routing sophistication and no autonomous tier-1 handling.
- Several high-value capabilities remain underserved across the field: pre-chat visitor intent prediction, sentiment-driven escalation, cross-session context, and dynamic agent skill matching.
- Routing intelligence (by intent, account tier, and CRM data) is becoming the key competitive battleground, but most platforms still rely on static rules.

---

## Key Features

### Real-Time Messaging Core

- WebSocket-based delivery with sub-second latency
- Multi-channel support across web, email, and SMS
- Persistent chat history and searchable transcripts
- Pre-chat forms for visitor qualification
- Canned messages and quick replies

### Routing and Workflow

- Skill-based and team-based routing rules
- SLA management and escalation
- Proactive chat triggers based on visitor behaviour
- Auto-assignment based on conversation criteria
- Custom, brandable chat widget

### AI Assist

- Suggested replies surfaced as the agent types
- AI-drafted responses based on knowledge base content
- Visitor tracking and identification
- Foundations for autonomous tier-1 resolution

### CRM and Integrations

- Native CRM integration (HubSpot, Salesforce)
- REST API for conversations and customer data
- Webhooks for real-time events
- Agent mobile app for on-the-go response

### Analytics

- Response time and resolution rate metrics
- Engagement and conversation analytics
- Reporting on agent performance and SLA adherence

---

## AI-Native Advantage

The platform is designed around AI capabilities that incumbents either charge premium prices for or do not yet offer at all: visitor intent scoring that proactively opens conversations with high-value accounts, autonomous tier-1 resolution with confidence-based handoff to humans, CRM-aware routing that reads deal stage and customer health, real-time agent assist that surfaces relevant articles and past conversation summaries, and sentiment plus churn-risk detection during live chats that can trigger escalation or retention offers.

---

## Tech Stack & Deployment

The system is built on the WebSocket Protocol (RFC 6455) for real-time delivery. It is intended to be deployable both as a self-hosted service (for data residency and full control) and as a managed cloud offering. Integration is exposed through a REST API plus webhooks, with native connectors to Salesforce and HubSpot. Compliance is anchored in GDPR/CCPA (consent, retention, right-to-erasure), ISO/IEC 27001 expectations for enterprise buyers, and WCAG 2.1 accessibility for the chat widget.

---

## Market Context

The live chat software market exceeds $1.5 billion and is growing at roughly 8–10% CAGR, with Gartner projecting that conversational AI will reduce global contact-centre agent labour costs by $80 billion by 2026. Pricing today ranges from free tiers (HubSpot, Crisp, Tidio) and SMB plans at $19–$29/agent/month up to mid-market tiers at $74–$200+/seat/month, with enterprise AI-agent deployments custom-priced by conversation volume. Primary buyers are customer success managers at PLG SaaS companies, inbound sales teams, support managers focused on ticket deflection, and e-commerce operators needing 24/7 coverage.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
