# AI Automation Assistant

AI Automation Assistant is a portfolio of three independent n8n workflow packages for AI-powered messaging, research, and inbox automation. Each workflow can be imported and used on its own, with credentials configured inside n8n.

The workflows use n8n, OpenAI chat models, Gmail, Telegram, WhatsApp, search tools, and Cloudflare Tunnel hosting for public webhook access.

## What Is Included

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

The project contains three automation packages:

| Package | Workflow file | Purpose |
| --- | --- | --- |
| Gmail Subscription Assistant | `workflows/subscription-reply-and-labeling.json` | Detects subscription-related emails, drafts replies, and labels matching Gmail messages. |
| Telegram Research Assistant | `workflows/telegram-assistant.json` and `workflows/telegram-search-subworkflow.json` | Replies to Telegram messages with AI and can call a separate research helper for web/search-style questions. |
| WhatsApp AI Assistant | `workflows/whatsapp-automation.json` | Responds to WhatsApp messages with AI, memory, tools, and optional Gmail sending. |

Credentials are intentionally not stored in this repository. After importing the workflows, reconnect the required credentials inside n8n.

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

## Workflow Screenshots

### Gmail Subscription Assistant

![Gmail Subscription Assistant](docs/images/subscription-reply-and-labeling.png)

This workflow starts from a Gmail trigger. It fetches the email, normalizes fields, asks an AI Agent whether the message is subscription-related, branches on the result, generates a reply when needed, sends the Gmail reply, and applies a Gmail label.

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

### Telegram Research Assistant

![Telegram Research Assistant](docs/images/telegram-assistant.png)

This workflow receives Telegram messages, passes the message text into an AI Agent, keeps short-term conversation memory, and replies back into the same Telegram chat. It can call the search helper workflow through a tool node named `search_agent`.

Main nodes:

- Telegram Trigger
- AI Agent
- OpenAI Chat Model
- Simple Memory
- `search_agent`
- Send a text message

### Telegram Search Helper

![Telegram Search Helper](docs/images/telegram-search-subworkflow.png)

This helper workflow is called by the Telegram Research Assistant when a user asks for information that benefits from external lookup. It gives the AI Agent access to Wikipedia, Hacker News, and SerpApi-powered Google Search.

Main nodes:

- When Executed by Another Workflow
- AI Agent
- OpenAI Chat Model
- Wikipedia
- Get many items in Hacker News
- Google search in SerpApi

### WhatsApp AI Assistant

![WhatsApp AI Assistant](docs/images/whatsapp-automation.png)

This workflow turns WhatsApp into an AI assistant channel. It receives inbound WhatsApp messages, passes the message body to an AI Agent, remembers short-term context, and replies through WhatsApp. It also gives the agent tools for calculation, Wikipedia lookup, and Gmail sending.

Main nodes:

- WhatsApp Trigger
- AI Agent
- OpenAI Chat Model
- Simple Memory
- Send message
- Calculator
- Wikipedia
- Send a message in Gmail

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

The three automation packages are exported, documented, and ready to import into n8n. The workflows demonstrate practical AI automation across Gmail, Telegram, WhatsApp, web search, memory, and tool use.

Next improvements:

- Add demo videos for each package.
- Add example test messages for Telegram and WhatsApp.
- Add credential setup screenshots.
- Add a VPS deployment guide for production hosting.
