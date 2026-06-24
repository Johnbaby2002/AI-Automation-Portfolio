# AI Automation Systems Portfolio

Designing and deploying AI-powered agents, workflow automation systems, and intelligent messaging solutions using OpenAI, n8n, Cloudflare, and modern APIs.

## What Is Included

This repository contains exported n8n workflow files, screenshots, and setup documentation for three independent automation systems. Each system is designed around a real operational workflow rather than a standalone chatbot demo.

```text
workflows/
  subscription-reply-and-labeling.json
  telegram-assistant.json
  telegram-search-subworkflow.json
  whatsapp-automation.json

docs/images/
  subscription-reply-and-labeling.png
  telegram-assistant.png
  telegram-search-subworkflow.png
  whatsapp-automation.png
```

| Automation System | Workflow file | Purpose |
| --- | --- | --- |
| Gmail Subscription Assistant | `workflows/subscription-reply-and-labeling.json` | Detects subscription-related emails, drafts replies, and labels matching Gmail messages. |
| Telegram Research Assistant | `workflows/telegram-assistant.json` and `workflows/telegram-search-subworkflow.json` | Replies to Telegram messages with AI and can call a separate research helper for web/search-style questions. |
| WhatsApp AI Assistant | `workflows/whatsapp-automation.json` | Responds to WhatsApp messages with AI, memory, tools, and optional Gmail sending. |

Credentials are intentionally not stored in this repository. After importing the workflows, reconnect the required credentials inside n8n.

## Gmail Subscription Assistant

![Gmail Subscription Assistant](docs/images/subscription-reply-and-labeling.png)

### Objective

Reduce repetitive inbox work by identifying subscription-related emails, preparing an AI-generated response, and applying a Gmail label for organization.

### What It Does

This workflow starts from a Gmail trigger. It fetches the email, normalizes fields, asks an AI Agent whether the message is subscription-related, branches on the result, generates a reply when needed, sends the Gmail reply, and applies a Gmail label.

### How I Built It

I built this as an event-driven n8n workflow where Gmail is the entry point. The trigger listens for incoming Gmail messages, then a Gmail node retrieves the full message content so the AI step has enough context to classify the email.

Before sending the message into the agent, I used an Edit Fields step to normalize the data that matters for classification. This keeps the prompt focused on the email content instead of passing a noisy raw Gmail payload.

The AI Agent is connected to an OpenAI Chat Model and a Structured Output Parser. The prompt asks the model to return a predictable JSON object with fields such as `isSubscription` and `reasoning`. That structured response is then used by an If node to decide whether the workflow should continue to reply and label the email or stop without action.

For the action path, the workflow uses an OpenAI model node to generate the reply text, a Gmail reply node to respond in the same thread, and a Gmail label node to mark the message. This demonstrates a full loop: event intake, AI classification, structured decision-making, content generation, and API-based action.

### Main Components

- Gmail Trigger for event-driven email processing
- Gmail node for retrieving the full email message
- Set/Edit Fields node for normalizing input data
- AI Agent for classification and reasoning
- OpenAI Chat Model for language understanding
- Structured Output Parser for reliable JSON output
- If node for branching based on AI classification
- Gmail reply and label actions

### Engineering Concepts Demonstrated

- Event-driven workflow automation
- Prompt engineering for classification
- Structured AI outputs
- Conditional routing based on model output
- Gmail API integration
- Human-readable reasoning for classification decisions
- Practical inbox automation

Main nodes:

- Gmail Trigger
- Get a message
- Edit Fields
- AI Agent
- OpenAI Chat Model
- Structured Output Parser
- If
- Message a model
- Reply to a message
- Add label to message

## Telegram Research Assistant

![Telegram Research Assistant](docs/images/telegram-assistant.png)

### Objective

Create a Telegram-based AI assistant that can answer user messages, preserve short-term context, and call a separate research workflow when external information is needed.

### What It Does

This workflow receives Telegram messages, passes the message text into an AI Agent, keeps short-term conversation memory, and replies back into the same Telegram chat. It can call the search helper workflow through a tool node named `search_agent`.

### How I Built It

I built the Telegram assistant around a Telegram Trigger node that listens for new messages from a Telegram bot. The workflow extracts the incoming message text and passes it directly into an AI Agent as the main user prompt.

The AI Agent is connected to an OpenAI Chat Model for reasoning and response generation. I added Simple Memory so the assistant can keep short-term conversation context instead of treating every message as isolated.

The important design choice is the `search_agent` tool. Instead of placing all research tools inside the main Telegram workflow, I connected the agent to a separate n8n workflow tool. This keeps the Telegram workflow focused on chat intake and reply delivery, while the search workflow handles external lookup.

The final Telegram node sends the AI output back to the same chat ID from the original trigger. This makes the workflow a complete conversational loop: message received, context loaded, agent reasoning performed, optional tool workflow called, and response sent back through Telegram.

### Main Components

- Telegram Trigger for inbound messages
- AI Agent for reasoning and response generation
- OpenAI Chat Model for natural language processing
- Simple Memory for conversation context
- Workflow tool for calling the search helper
- Telegram send message node for replies

### Engineering Concepts Demonstrated

- Agentic workflow design
- Conversational memory
- Tool calling through a sub-workflow
- Telegram Bot API integration
- Multi-step reasoning across workflows
- Separation of message handling and research logic

Main nodes:

- Telegram Trigger
- AI Agent
- OpenAI Chat Model
- Simple Memory
- `search_agent`
- Send a text message

## Telegram Search Helper

![Telegram Search Helper](docs/images/telegram-search-subworkflow.png)

### Objective

Provide a reusable research sub-workflow that can be called by another workflow when a user request requires external lookup or broader context.

### What It Does

This helper workflow is called by the Telegram Research Assistant when a user asks for information that benefits from external lookup. It gives the AI Agent access to Wikipedia, Hacker News, and SerpApi-powered Google Search.

### How I Built It

I built this as a sub-workflow using the "When Executed by Another Workflow" trigger. That lets the main Telegram assistant call it like a tool and pass in a search query as structured input.

Inside the helper, an AI Agent receives the query and decides which research tool is most appropriate. It can use Wikipedia for general knowledge, Hacker News for technology-related information, and SerpApi for broader Google Search results.

This modular design keeps external research separate from message delivery. It also makes the research layer reusable: another workflow could call the same helper without duplicating search nodes or prompt logic.

### Main Components

- Execute Workflow Trigger for sub-workflow invocation
- AI Agent for deciding how to answer the research query
- OpenAI Chat Model for reasoning
- Wikipedia tool for encyclopedia-style lookup
- Hacker News tool for technology and startup information
- SerpApi tool for Google Search access

### Engineering Concepts Demonstrated

- Reusable workflow architecture
- Tool selection by an AI Agent
- External API orchestration
- Search-augmented responses
- Modular agent design
- Separation of concerns between interface workflow and research workflow

Main nodes:

- When Executed by Another Workflow
- AI Agent
- OpenAI Chat Model
- Wikipedia
- Get many items in Hacker News
- Google search in SerpApi

## WhatsApp AI Assistant

![WhatsApp AI Assistant](docs/images/whatsapp-automation.png)

### Objective

Build a WhatsApp-based AI assistant that can respond to messages, maintain context, use tools, and perform connected actions such as sending email.

### What It Does

This workflow turns WhatsApp into an AI assistant channel. It receives inbound WhatsApp messages, passes the message body to an AI Agent, remembers short-term context, and replies through WhatsApp. It also gives the agent tools for calculation, Wikipedia lookup, and Gmail sending.

### How I Built It

I built this workflow around a WhatsApp Trigger node connected to the WhatsApp Cloud API. Incoming WhatsApp message text is extracted from the webhook payload and passed into an AI Agent.

The agent uses an OpenAI Chat Model for reasoning and Simple Memory for short-term context. I added multiple tools to make the agent more useful than a basic responder: Calculator for arithmetic, Wikipedia for lookup, and a Gmail tool for composing and sending email when requested.

The response path uses the WhatsApp send message node to return the final agent output to the user. This workflow demonstrates a multi-tool messaging assistant where the AI agent can reason, choose a tool, perform an action, and reply through the original communication channel.

### Main Components

- WhatsApp Trigger for inbound messages
- AI Agent for response generation and tool use
- OpenAI Chat Model for reasoning
- Simple Memory for short-term context
- WhatsApp send message node for replies
- Calculator tool for arithmetic
- Wikipedia tool for lookup
- Gmail tool for sending email when requested

### Engineering Concepts Demonstrated

- Multi-channel AI assistant design
- Tool-augmented agent behavior
- WhatsApp Cloud API integration
- Conversational memory
- Cross-application automation
- AI-assisted communication workflows

Main nodes:

- WhatsApp Trigger
- AI Agent
- OpenAI Chat Model
- Simple Memory
- Send message
- Calculator
- Wikipedia
- Send a message in Gmail

## Executive Summary

This portfolio showcases practical AI-powered automation systems built with n8n, OpenAI models, external APIs, and self-hosted webhook infrastructure. The workflows demonstrate how AI agents can be connected to real communication channels, business tools, memory, structured outputs, and tool-calling workflows.

The focus is practical automation, not simple chatbot demos. Each system is designed around a real operational pattern: classifying and responding to email, answering messages through chat platforms, retrieving external information, and orchestrating actions across APIs.

The portfolio combines:

- AI agents for reasoning and response generation
- Tool calling for external actions and information retrieval
- Conversational memory for context-aware replies
- Structured outputs for reliable branching logic
- Event-driven workflow orchestration
- API integrations with Gmail, Telegram, WhatsApp, SerpApi, Wikipedia, and Hacker News
- Self-hosted n8n infrastructure exposed through Cloudflare Tunnel

## How The Systems Were Built

The systems were built in n8n using a visual workflow architecture. Each workflow is composed of trigger nodes, AI Agent nodes, OpenAI model nodes, tool nodes, memory nodes, and service-specific action nodes.

The build process followed a consistent pattern:

1. Define the event source, such as Gmail, Telegram, or WhatsApp.
2. Extract the message or email content from the incoming payload.
3. Normalize the payload so the AI agent receives only relevant context.
4. Connect the AI Agent to an OpenAI Chat Model.
5. Add tools such as Gmail, Wikipedia, SerpApi, Hacker News, Calculator, or a workflow tool.
6. Add memory where the channel benefits from conversation continuity.
7. Use structured outputs or conditional nodes where the workflow needs reliable branching.
8. Send the final output back through the original channel or perform the connected action.
9. Expose the local n8n instance through Cloudflare Tunnel so external webhook platforms can reach it over HTTPS.

This approach keeps the workflows inspectable. Each step in the system is visible as a node, which makes debugging easier than hiding the logic inside a single script.

## Why This Portfolio Exists

Repetitive administrative work is one of the strongest use cases for AI automation. Email handling, routine customer communication, information retrieval, internal workflow updates, and first-pass triage all consume time that can often be reduced with well-designed automation systems.

This portfolio explores how AI agents can support those workflows without relying on isolated chat interfaces. Instead of requiring a user to manually ask a chatbot for help, these systems are triggered by real events such as an incoming email, Telegram message, or WhatsApp message.

The work is also relevant to Health Informatics and Digital Health. Healthcare and health-adjacent organizations depend on communication, documentation, follow-up, search, routing, and administrative coordination. The same automation patterns shown here can support safer information access, reduce manual workload, and improve operational responsiveness when adapted responsibly to healthcare environments.

## System Architecture

```text
User channels
  WhatsApp
  Telegram
  Gmail
      |
      v
n8n workflows
      |
      v
OpenAI AI agents
      |
      v
Tools and APIs
  Gmail
  Wikipedia
  SerpApi
  Hacker News
  Calculator
      |
      v
Cloudflare Tunnel / self-hosted n8n infrastructure
```

At a high level, each system starts with an external event, routes that event into n8n, uses an AI Agent for reasoning, optionally calls tools or APIs, and sends a response or performs an action through the relevant channel.

## Importing The Workflows

Import any package you want to use:

```text
n8n UI -> Workflows -> Import from File
```

For the Telegram package, import both files:

```text
workflows/telegram-search-subworkflow.json
workflows/telegram-assistant.json
```

Then open `telegram-assistant.json` in n8n and confirm that the `search_agent` tool points to the imported search sub-workflow.

The Gmail and WhatsApp workflows are independent and can be imported separately.

## Demonstrated Competencies

- AI Agents
- OpenAI APIs
- Workflow Automation
- Agentic Systems
- Prompt Engineering
- Tool Calling
- Structured Outputs
- Conversational Memory
- API Integration
- Event-Driven Architecture
- Cloudflare Tunnel
- Self-Hosted Services
- Webhook Infrastructure
- Digital Health Automation

## Engineering Challenges

### Context Management

The messaging workflows need to preserve enough context for useful replies without allowing conversations to become noisy or inconsistent. Simple Memory is used to keep short-term context while keeping workflow behavior understandable.

### Tool Selection

The AI Agent must decide when to answer directly and when to call tools such as Wikipedia, SerpApi, Hacker News, Calculator, Gmail, or a sub-workflow. This requires clear tool descriptions and prompt design.

### Workflow Reliability

The systems use explicit workflow nodes, structured outputs, and conditional branching to make AI behavior easier to inspect and debug. This is especially important when workflows perform actions such as replying to email or sending messages.

### Webhook Exposure

Messaging platforms require public HTTPS webhook endpoints. Cloudflare Tunnel is used as a deployment setup for exposing self-hosted n8n without placing credentials or local services directly in a public repository.

### Credential Security

Credentials are managed inside n8n and are not committed to Git. The repository stores workflow structure and documentation only, not secrets, API keys, tokens, phone numbers, or private tunnel IDs.

### Structured AI Outputs

The Gmail workflow demonstrates structured model output so the workflow can reliably branch on `isSubscription` rather than depending on free-form text.

## Relevance to Digital Health

As a Health Informatics student, I am interested in how automation can support healthcare and digital health operations without replacing professional judgment. Many healthcare workflows involve repeated administrative tasks: routing messages, summarizing information, preparing responses, retrieving policy or knowledge-base content, and coordinating follow-up.

The patterns in this portfolio can support digital health environments by:

- Reducing repetitive administrative workload
- Improving access to relevant information
- Supporting communication workflows across channels
- Helping staff triage routine requests
- Connecting messaging systems with internal tools
- Demonstrating how AI can assist operational workflows while keeping human oversight in the loop

These examples are not clinical decision systems. They are automation patterns that can be adapted responsibly for health operations, patient communication support, internal knowledge retrieval, and administrative coordination.

## Required Accounts And Credentials

Create these credentials in n8n after importing the workflows:

| Credential | Used by | Notes |
| --- | --- | --- |
| OpenAI API | All AI Agent / OpenAI model nodes | Required for AI responses and tool reasoning. |
| Gmail OAuth2 | Gmail workflow and WhatsApp Gmail tool | Required for reading, replying, labeling, and sending email. |
| Telegram Bot API | Telegram workflow | Create a Telegram bot with BotFather and connect the bot token in n8n. |
| WhatsApp Cloud API | WhatsApp workflow | Requires Meta WhatsApp Cloud API setup and webhook configuration. |
| SerpApi | Telegram search helper | Required for Google Search through SerpApi. |

Do not commit credential files, API keys, database files, Cloudflare tunnel credential files, or n8n runtime data.

## Hosting Overview

These workflows can run in any n8n instance. The setup used for this project is self-hosted n8n behind Cloudflare Tunnel:

1. n8n runs on port `5678`.
2. Cloudflare Tunnel exposes the n8n instance over HTTPS.
3. A Cloudflare DNS route points a public hostname to the tunnel.
4. n8n uses the public HTTPS hostname for webhook URLs.

This pattern is useful because Telegram, WhatsApp, Gmail, and other webhook-based integrations need a stable HTTPS endpoint, while n8n can still run outside a traditional cloud server.

## Cloudflare Tunnel Configuration

The tunnel needs an ingress rule that maps the public hostname to the n8n service:

```yaml
tunnel: <tunnel-id-or-name>
credentials-file: <path-to-cloudflared-credentials-json>

ingress:
  - hostname: n8n.example.com
    service: http://localhost:5678
  - service: http_status:404
```

The key part is the `ingress` section. Without it, cloudflared can connect to Cloudflare but will return `503` because it does not know where to send incoming HTTP traffic.

Create the DNS route with:

```powershell
cloudflared tunnel route dns <tunnel-name> n8n.example.com
```

Run the tunnel with:

```powershell
cloudflared tunnel --config "<path-to-config.yml>" run <tunnel-name>
```

## n8n Public URL Environment Variables

Set these variables so n8n generates correct public webhook URLs:

```powershell
[Environment]::SetEnvironmentVariable('WEBHOOK_URL','https://n8n.example.com','User')
[Environment]::SetEnvironmentVariable('N8N_HOST','n8n.example.com','User')
[Environment]::SetEnvironmentVariable('N8N_PROTOCOL','https','User')
[Environment]::SetEnvironmentVariable('N8N_PROXY_HOPS','1','User')
```

Replace `n8n.example.com` with the real public hostname.

Restart n8n after setting environment variables.

## Start And Verify n8n

Start n8n:

```powershell
n8n start
```

Verify the local service:

```powershell
Invoke-WebRequest -UseBasicParsing http://localhost:5678
```

Verify the public service:

```powershell
Invoke-WebRequest -UseBasicParsing https://n8n.example.com
```

Both checks should return `200 OK`.

Validate Cloudflare Tunnel routing:

```powershell
cloudflared tunnel ingress validate
cloudflared tunnel ingress rule https://n8n.example.com
```

Expected result:

```text
Matched rule #0
service: http://localhost:5678
```

## Common Issues

### cloudflared returns 503

Cause: the tunnel is running without ingress rules.

Fix: add the `ingress` section to the tunnel config and restart cloudflared with that config.

### Cloudflare returns 502

Cause: Cloudflare reached the tunnel, but n8n was not running on port `5678`.

Fix:

```powershell
n8n start
Invoke-WebRequest -UseBasicParsing http://localhost:5678
```

### n8n says port 5678 is already in use

Cause: another n8n process is already running.

Fix:

```powershell
$portOwner = Get-NetTCPConnection -LocalPort 5678 -State Listen |
  Select-Object -First 1 -ExpandProperty OwningProcess
Stop-Process -Id $portOwner -Force
n8n start
```

### Telegram workflow cannot use search

Cause: the search helper workflow was not imported, or the tool node points to the wrong workflow.

Fix: import `telegram-search-subworkflow.json`, then open `telegram-assistant.json` and reconnect the `search_agent` tool to the imported search helper.

### Gmail nodes fail

Cause: Gmail OAuth2 credentials are missing or expired.

Fix: reconnect Gmail OAuth2 in n8n credentials, then reopen the Gmail workflow and select the credential on all Gmail nodes.

## Security Checklist

- Keep n8n credentials in n8n, not in Git.
- Do not commit `.n8n` runtime data.
- Do not commit Cloudflare tunnel credential files.
- Use HTTPS public webhooks only.
- Keep `N8N_PROXY_HOPS=1` when running behind Cloudflare Tunnel.
- Review AI-generated email replies before fully automating production inboxes.
- Use test Telegram and WhatsApp bots before connecting real customer channels.

## Project Status

The three automation systems are exported, documented, and ready to import into n8n. The workflows demonstrate practical AI automation across Gmail, Telegram, WhatsApp, web search, memory, tool use, structured outputs, and self-hosted webhook infrastructure.

Next improvements:

- Add demo videos for each automation system.
- Add example test messages for Telegram and WhatsApp.
- Add credential setup screenshots.
- Add a VPS deployment guide for production hosting.
