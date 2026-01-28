# Copilot Studio

## What is Copilot Studio?

Copilot Studio (formerly Power Virtual Agents) is Microsoft's low-code/no-code platform for building custom AI-powered copilots. It enables rapid development of conversational AI solutions with deep Microsoft 365 integration.

### Copilot Studio vs AWS Lex

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   COPILOT STUDIO vs AWS LEX                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS LEX                              COPILOT STUDIO                        │
│  ───────                              ──────────────                        │
│                                                                              │
│  Architecture:                        Architecture:                         │
│  • Intent-based (traditional NLU)     • Generative AI first                │
│  • Define intents, slots, utterances  • Topics + Generative answers        │
│  • Lambda for fulfillment             • Power Automate for actions         │
│                                                                              │
│  AI Model:                            AI Model:                             │
│  • Custom NLU per bot                 • Azure OpenAI (GPT-4)               │
│  • Bedrock integration (separate)     • Built-in generative AI             │
│                                                                              │
│  Knowledge:                           Knowledge:                            │
│  • Amazon Kendra integration          • SharePoint sites                   │
│  • Custom data sources (complex)      • Public websites                    │
│                                       • Dataverse                          │
│                                       • Files (upload directly)            │
│                                                                              │
│  Actions:                             Actions:                              │
│  • Lambda functions                   • Power Automate (500+ connectors)  │
│  • Step Functions                     • Custom connectors                  │
│  • Custom code                        • Copilot plugins                    │
│                                                                              │
│  Channels:                            Channels:                             │
│  • Web, Facebook, Slack               • Teams, web, mobile                 │
│  • Connect, SMS                       • Deep M365 integration              │
│  • Amazon Alexa                       • Microsoft 365 Copilot              │
│                                                                              │
│  Development:                         Development:                         │
│  • Console + SDK                      • Low-code web designer             │
│  • Infrastructure to manage           • No infrastructure                  │
│                                       • Citizen developer friendly         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Copilot Studio Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COPILOT STUDIO ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  USER CHANNELS:                                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│  │  Teams   │ │   Web    │ │  Mobile  │ │ Facebook │ │  Custom  │         │
│  │  App     │ │  Widget  │ │   App    │ │Messenger │ │    API   │         │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘         │
│       │            │            │            │            │                │
│       └────────────┴────────────┴────────────┴────────────┘                │
│                                 │                                          │
│                                 ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                        COPILOT RUNTIME                                │  │
│  │                                                                       │  │
│  │  ┌───────────────────────────────────────────────────────────────┐  │  │
│  │  │                    CONVERSATION ORCHESTRATION                   │  │  │
│  │  │                                                                 │  │  │
│  │  │  1. Topic Triggering                                           │  │  │
│  │  │     └── Match user input to topics or generative answers       │  │  │
│  │  │                                                                 │  │  │
│  │  │  2. Dialog Management                                          │  │  │
│  │  │     └── Execute topic flows, collect info, branch logic       │  │  │
│  │  │                                                                 │  │  │
│  │  │  3. Generative AI                                              │  │  │
│  │  │     └── Azure OpenAI for natural responses                    │  │  │
│  │  │                                                                 │  │  │
│  │  │  4. Knowledge Retrieval                                        │  │  │
│  │  │     └── Search SharePoint, websites, Dataverse                │  │  │
│  │  │                                                                 │  │  │
│  │  └───────────────────────────────────────────────────────────────┘  │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                 │                                          │
│                                 ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                        INTEGRATIONS                                   │  │
│  │                                                                       │  │
│  │  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐              │  │
│  │  │Power Automate │ │   Dataverse   │ │ Azure OpenAI  │              │  │
│  │  │ (Actions)     │ │ (Data store)  │ │ (AI models)   │              │  │
│  │  └───────────────┘ └───────────────┘ └───────────────┘              │  │
│  │                                                                       │  │
│  │  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐              │  │
│  │  │  SharePoint   │ │  Connectors   │ │Custom Plugins │              │  │
│  │  │ (Knowledge)   │ │  (500+ apps)  │ │   (APIs)      │              │  │
│  │  └───────────────┘ └───────────────┘ └───────────────┘              │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Building a Copilot

### Step 1: Create and Configure

```
CREATING A COPILOT:
───────────────────

1. Go to copilot.microsoft.com (or via Power Platform admin center)

2. Create new copilot:
   • Name: "HR Assistant"
   • Description: "Helps employees with HR questions"
   • Language: English
   • Icon: Custom or template

3. Configure settings:
   ┌────────────────────────────────────────────────────────────────────────┐
   │ Generative AI:                                                         │
   │ • ☑ Allow generative answers (recommended)                            │
   │ • ☑ Boost conversational capabilities                                 │
   │                                                                        │
   │ Content moderation:                                                    │
   │ • ☑ Enable content safety                                             │
   │ • Sensitivity: Medium                                                  │
   │                                                                        │
   │ Authentication:                                                        │
   │ • ☑ Require sign-in with Microsoft Entra ID                          │
   │ • Scope: Organization only                                            │
   └────────────────────────────────────────────────────────────────────────┘
```

### Step 2: Add Knowledge Sources

```
KNOWLEDGE CONFIGURATION:
────────────────────────

SharePoint Sites:
┌────────────────────────────────────────────────────────────────────────────┐
│ Site: https://contoso.sharepoint.com/sites/HR-Policies                     │
│                                                                             │
│ Indexed content:                                                           │
│ • All documents in document libraries                                      │
│ • Wiki pages                                                               │
│ • List items                                                               │
│                                                                             │
│ Refresh: Automatic (near real-time)                                       │
└────────────────────────────────────────────────────────────────────────────┘

Public Websites:
┌────────────────────────────────────────────────────────────────────────────┐
│ URL: https://www.contoso.com/benefits                                      │
│                                                                             │
│ Options:                                                                   │
│ • Crawl depth: 2 levels                                                   │
│ • Include subdomains: No                                                  │
│ • Exclude patterns: /login/*, /admin/*                                   │
│                                                                             │
│ Refresh: Daily                                                            │
└────────────────────────────────────────────────────────────────────────────┘

Uploaded Files:
┌────────────────────────────────────────────────────────────────────────────┐
│ • Employee_Handbook_2024.pdf                                              │
│ • Benefits_Summary.docx                                                   │
│ • FAQ_Document.xlsx                                                       │
│                                                                             │
│ Supported formats: PDF, Word, Excel, PowerPoint, HTML, Text              │
└────────────────────────────────────────────────────────────────────────────┘
```

### Step 3: Create Topics

```
TOPIC STRUCTURE:
────────────────

Topic: "Request Time Off"
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│ TRIGGER PHRASES:                                                           │
│ • "I want to request vacation"                                            │
│ • "How do I take time off"                                                │
│ • "Submit PTO request"                                                     │
│ • "Book holiday"                                                           │
│                                                                             │
│ CONVERSATION FLOW:                                                         │
│                                                                             │
│ ┌─────────────────────────────────────────────────────────────────────┐   │
│ │ [Message]                                                            │   │
│ │ "I'd be happy to help you request time off! Let me gather           │   │
│ │  some information."                                                  │   │
│ └─────────────────────────────────────────────────────────────────────┘   │
│                          │                                                 │
│                          ▼                                                 │
│ ┌─────────────────────────────────────────────────────────────────────┐   │
│ │ [Question]                                                           │   │
│ │ "What type of time off are you requesting?"                         │   │
│ │                                                                      │   │
│ │ Options (Multiple choice):                                          │   │
│ │ • Vacation                                                          │   │
│ │ • Sick leave                                                        │   │
│ │ • Personal day                                                      │   │
│ │                                                                      │   │
│ │ Save to: Topic.TimeOffType                                          │   │
│ └─────────────────────────────────────────────────────────────────────┘   │
│                          │                                                 │
│                          ▼                                                 │
│ ┌─────────────────────────────────────────────────────────────────────┐   │
│ │ [Question]                                                           │   │
│ │ "What is the start date?"                                           │   │
│ │                                                                      │   │
│ │ Entity: Date                                                        │   │
│ │ Save to: Topic.StartDate                                            │   │
│ └─────────────────────────────────────────────────────────────────────┘   │
│                          │                                                 │
│                          ▼                                                 │
│ ┌─────────────────────────────────────────────────────────────────────┐   │
│ │ [Question]                                                           │   │
│ │ "How many days?"                                                    │   │
│ │                                                                      │   │
│ │ Entity: Number                                                      │   │
│ │ Save to: Topic.Days                                                 │   │
│ └─────────────────────────────────────────────────────────────────────┘   │
│                          │                                                 │
│                          ▼                                                 │
│ ┌─────────────────────────────────────────────────────────────────────┐   │
│ │ [Action: Power Automate]                                            │   │
│ │ Flow: "Create Time Off Request"                                     │   │
│ │ Inputs:                                                             │   │
│ │   - Employee: System.User.Email                                     │   │
│ │   - Type: Topic.TimeOffType                                         │   │
│ │   - Start: Topic.StartDate                                          │   │
│ │   - Days: Topic.Days                                                │   │
│ │ Output: Topic.RequestID                                             │   │
│ └─────────────────────────────────────────────────────────────────────┘   │
│                          │                                                 │
│                          ▼                                                 │
│ ┌─────────────────────────────────────────────────────────────────────┐   │
│ │ [Message]                                                            │   │
│ │ "Your time off request has been submitted!                          │   │
│ │  Request ID: {Topic.RequestID}                                      │   │
│ │  Your manager will be notified for approval."                       │   │
│ └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Step 4: Add Actions

```
ACTION TYPES:
─────────────

1. POWER AUTOMATE FLOWS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • 500+ pre-built connectors                                         │
   │ • Create records in business systems                               │
   │ • Send emails, Teams messages                                      │
   │ • Update SharePoint, Dataverse                                     │
   │ • Call REST APIs                                                   │
   └─────────────────────────────────────────────────────────────────────┘

2. CUSTOM CONNECTORS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Connect to your own APIs                                         │
   │ • OpenAPI/Swagger definition                                       │
   │ • Authentication: API key, OAuth, Basic                           │
   │                                                                     │
   │ Example:                                                           │
   │ Your HR System API → Custom connector → Copilot action            │
   └─────────────────────────────────────────────────────────────────────┘

3. PREBUILT ACTIONS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Send email (Office 365)                                          │
   │ • Create calendar event                                            │
   │ • Post to Teams channel                                           │
   │ • Create task in Planner                                          │
   │ • Search SharePoint                                               │
   └─────────────────────────────────────────────────────────────────────┘

4. COPILOT PLUGINS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Use plugins from Microsoft 365 Copilot ecosystem                │
   │ • Extend with custom plugins                                       │
   │ • Share across copilots                                           │
   └─────────────────────────────────────────────────────────────────────┘
```

---

## Deployment and Channels

### Publishing to Teams

```
TEAMS DEPLOYMENT:
─────────────────

1. Publish copilot:
   Copilot Studio → Publish → Publish latest content

2. Make available in Teams:
   Channels → Microsoft Teams → Turn on

3. Availability options:
   ┌────────────────────────────────────────────────────────────────────────┐
   │ • Show in Teams app store (org-wide)                                  │
   │ • Share link (specific users)                                         │
   │ • Embed in Teams app (custom tab)                                     │
   │ • Add to team/channel                                                 │
   └────────────────────────────────────────────────────────────────────────┘

4. Admin approval (if required):
   Teams Admin Center → Teams apps → Manage apps → Approve
```

### Web Widget

```html
<!-- Embed copilot in your website -->
<iframe
  src="https://web.powerva.microsoft.com/environments/{env-id}/bots/{bot-id}/webchat"
  style="width: 100%; height: 600px; border: none;"
></iframe>

<!-- Or use the customizable web chat -->
<script src="https://cdn.botframework.com/botframework-webchat/latest/webchat.js"></script>
<script>
  window.WebChat.renderWebChat({
    directLine: window.WebChat.createDirectLine({
      token: 'YOUR_TOKEN'
    }),
    styleOptions: {
      accent: '#0078d4',
      botAvatarImage: 'https://your-domain.com/bot-avatar.png',
      userAvatarImage: 'https://your-domain.com/user-avatar.png'
    }
  }, document.getElementById('webchat'));
</script>
```

---

## Security and Governance

```
COPILOT SECURITY:
─────────────────

AUTHENTICATION:
┌────────────────────────────────────────────────────────────────────────────┐
│ • Microsoft Entra ID (recommended for enterprise)                         │
│ • No authentication (public-facing)                                       │
│ • Custom authentication (API-based)                                       │
│                                                                             │
│ User context available:                                                    │
│ • System.User.Email                                                       │
│ • System.User.DisplayName                                                 │
│ • System.User.Id                                                          │
│ • Custom claims from Entra ID                                             │
└────────────────────────────────────────────────────────────────────────────┘

DATA LOSS PREVENTION:
┌────────────────────────────────────────────────────────────────────────────┐
│ • DLP policies apply to copilot actions                                   │
│ • Block sensitive connectors                                              │
│ • Audit all actions                                                       │
│                                                                             │
│ Power Platform Admin Center → Data policies                               │
└────────────────────────────────────────────────────────────────────────────┘

ENVIRONMENT ISOLATION:
┌────────────────────────────────────────────────────────────────────────────┐
│ • Separate environments for dev/test/prod                                 │
│ • Solution-based ALM                                                      │
│ • Export/import for deployment                                            │
│                                                                             │
│ Dev → Test → Production (via managed solutions)                          │
└────────────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Responsible AI](04-responsible-ai.md)* | *Back to [Azure OpenAI](02-azure-openai.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
