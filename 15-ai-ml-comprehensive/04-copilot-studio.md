# Microsoft Copilot Studio

## Overview

Microsoft Copilot Studio is the enterprise platform for building custom AI-powered copilots. It combines the power of generative AI with low-code development, enabling organizations to create sophisticated conversational AI solutions that integrate deeply with Microsoft 365 and business data.

## Understanding Copilot Studio

### The Copilot Ecosystem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MICROSOFT COPILOT ECOSYSTEM                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      MICROSOFT COPILOTS                              │   │
│   │                                                                      │   │
│   │   ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐   │   │
│   │   │ Microsoft  │  │   GitHub   │  │   Azure    │  │  Windows   │   │   │
│   │   │    365     │  │  Copilot   │  │  Copilot   │  │  Copilot   │   │   │
│   │   │  Copilot   │  │            │  │            │  │            │   │   │
│   │   │            │  │            │  │            │  │            │   │   │
│   │   │ Word,Excel │  │ Code gen,  │  │ Resource   │  │ OS tasks,  │   │   │
│   │   │ PPT,Teams  │  │ PR review  │  │ management │  │ Settings   │   │   │
│   │   └────────────┘  └────────────┘  └────────────┘  └────────────┘   │   │
│   │                                                                      │   │
│   │   Built by Microsoft    │    Enterprise features    │   AI-powered  │   │
│   └─────────────────────────┼───────────────────────────┼───────────────┘   │
│                             │                           │                    │
│                             ▼                           ▼                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                       COPILOT STUDIO                                 │   │
│   │                                                                      │   │
│   │   Build CUSTOM copilots for your organization                       │   │
│   │                                                                      │   │
│   │   ┌───────────────────────────────────────────────────────────────┐ │   │
│   │   │                                                               │ │   │
│   │   │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │ │   │
│   │   │   │   HR     │  │   IT     │  │  Sales   │  │ Customer │   │ │   │
│   │   │   │ Copilot  │  │ Copilot  │  │ Copilot  │  │ Service  │   │ │   │
│   │   │   │          │  │          │  │          │  │ Copilot  │   │ │   │
│   │   │   │ Benefits │  │ Helpdesk │  │ Pipeline │  │ Support  │   │ │   │
│   │   │   │ Policies │  │ Tickets  │  │ Quotes   │  │ FAQ      │   │ │   │
│   │   │   └──────────┘  └──────────┘  └──────────┘  └──────────┘   │ │   │
│   │   │                                                               │ │   │
│   │   │   Your data  │  Your processes  │  Your integrations         │ │   │
│   │   └───────────────────────────────────────────────────────────────┘ │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   EXTEND MICROSOFT 365 COPILOT:                                             │
│   ──────────────────────────────                                            │
│   • Copilot Studio copilots can be published AS plugins to M365 Copilot    │
│   • Bring custom data and actions to the M365 Copilot experience           │
│   • Declarative agents extend M365 Copilot capabilities                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Copilot Studio Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COPILOT STUDIO DEEP ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   USER INTERACTION                                                          │
│   ────────────────                                                          │
│   ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐       │
│   │ Teams  │ │  Web   │ │ Mobile │ │Facebook│ │ Custom │ │ M365   │       │
│   │  Chat  │ │ Widget │ │  App   │ │Messengr│ │  API   │ │Copilot │       │
│   └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘       │
│       │          │          │          │          │          │             │
│       └──────────┴──────────┴──────────┴──────────┴──────────┘             │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                       COPILOT RUNTIME ENGINE                         │   │
│   │                                                                      │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │                  ORCHESTRATION LAYER                         │   │   │
│   │   │                                                              │   │   │
│   │   │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │   │   │
│   │   │  │   Intent     │    │   Dialog     │    │   Response   │  │   │   │
│   │   │  │   Routing    │ -> │   Manager    │ -> │   Generator  │  │   │   │
│   │   │  │              │    │              │    │              │  │   │   │
│   │   │  │ • Topic      │    │ • Variables  │    │ • GPT-based  │  │   │   │
│   │   │  │   matching   │    │ • Conditions │    │ • Grounded   │  │   │   │
│   │   │  │ • Generative │    │ • Branching  │    │ • Citations  │  │   │   │
│   │   │  │   fallback   │    │ • Loops      │    │              │  │   │   │
│   │   │  └──────────────┘    └──────────────┘    └──────────────┘  │   │   │
│   │   │                                                              │   │   │
│   │   └─────────────────────────────────────────────────────────────┘   │   │
│   │                                                                      │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │                    AI CAPABILITIES                           │   │   │
│   │   │                                                              │   │   │
│   │   │   ┌───────────┐  ┌───────────┐  ┌───────────┐              │   │   │
│   │   │   │Generative │  │  Knowledge│  │  Content  │              │   │   │
│   │   │   │ Answers   │  │ Retrieval │  │  Safety   │              │   │   │
│   │   │   │           │  │           │  │           │              │   │   │
│   │   │   │Azure OpenAI│  │SharePoint│  │ Filters & │              │   │   │
│   │   │   │  GPT-4    │  │ AI Search │  │ Blocklist │              │   │   │
│   │   │   └───────────┘  └───────────┘  └───────────┘              │   │   │
│   │   │                                                              │   │   │
│   │   └─────────────────────────────────────────────────────────────┘   │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        INTEGRATION LAYER                             │   │
│   │                                                                      │   │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │   │
│   │   │Power Automate│  │  Dataverse   │  │   Custom     │             │   │
│   │   │   Flows      │  │   Tables     │  │  Connectors  │             │   │
│   │   │              │  │              │  │              │             │   │
│   │   │ 500+ actions │  │ Data storage │  │ Your APIs    │             │   │
│   │   │ HTTP calls   │  │ User context │  │ OAuth/API key│             │   │
│   │   │ Approvals    │  │ Session data │  │ OpenAPI spec │             │   │
│   │   └──────────────┘  └──────────────┘  └──────────────┘             │   │
│   │                                                                      │   │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │   │
│   │   │  SharePoint  │  │   Graph API  │  │   Copilot    │             │   │
│   │   │   Sites      │  │   Access     │  │   Plugins    │             │   │
│   │   │              │  │              │  │              │             │   │
│   │   │ Docs, lists  │  │ M365 data    │  │ Extensibility│             │   │
│   │   │ Knowledge    │  │ Calendar     │  │ Actions      │             │   │
│   │   └──────────────┘  └──────────────┘  └──────────────┘             │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Building Custom Copilots

### Creating a Copilot Step-by-Step

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COPILOT CREATION WORKFLOW                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   STEP 1: CREATE COPILOT                                                    │
│   ──────────────────────                                                    │
│   • Go to copilot.microsoft.com                                            │
│   • Select "Create" → "New copilot"                                        │
│   • Choose template or start blank                                          │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ Basic Configuration:                                                 │   │
│   │                                                                      │   │
│   │ Name: IT Helpdesk Assistant                                         │   │
│   │ Description: Helps employees with IT support requests               │   │
│   │ Language: English                                                    │   │
│   │ Icon: [Custom upload or select]                                     │   │
│   │                                                                      │   │
│   │ Solution: IT Support Solution                                       │   │
│   │ Environment: Production                                             │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   STEP 2: CONFIGURE AI SETTINGS                                             │
│   ─────────────────────────────                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ Generative AI:                                                       │   │
│   │ [x] Enable generative answers                                       │   │
│   │ [x] Allow copilot to use its own knowledge                         │   │
│   │ [x] Show citations in answers                                       │   │
│   │                                                                      │   │
│   │ Instructions (System Prompt):                                       │   │
│   │ ┌───────────────────────────────────────────────────────────────┐   │   │
│   │ │ You are an IT helpdesk assistant for Contoso Corporation.     │   │   │
│   │ │ Your role is to help employees resolve technical issues.      │   │   │
│   │ │                                                                │   │   │
│   │ │ Guidelines:                                                    │   │   │
│   │ │ - Be professional and helpful                                  │   │   │
│   │ │ - Ask clarifying questions when needed                         │   │   │
│   │ │ - Escalate to human agents for hardware issues                │   │   │
│   │ │ - Never share confidential system information                  │   │   │
│   │ │ - Always verify user identity for account changes             │   │   │
│   │ └───────────────────────────────────────────────────────────────┘   │   │
│   │                                                                      │   │
│   │ Content moderation: Medium                                          │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   STEP 3: ADD KNOWLEDGE SOURCES                                             │
│   ─────────────────────────────                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ Knowledge Sources:                                                   │   │
│   │                                                                      │   │
│   │ [+] SharePoint Site                                                 │   │
│   │     URL: https://contoso.sharepoint.com/sites/IT-KnowledgeBase     │   │
│   │     Status: Connected (1,247 documents indexed)                     │   │
│   │                                                                      │   │
│   │ [+] Public Website                                                  │   │
│   │     URL: https://support.contoso.com                               │   │
│   │     Crawl depth: 3 levels                                          │   │
│   │     Status: Synced (Updated daily)                                 │   │
│   │                                                                      │   │
│   │ [+] Uploaded Files                                                  │   │
│   │     • IT_Policies_2024.pdf                                         │   │
│   │     • Software_Installation_Guide.docx                             │   │
│   │     • VPN_Setup_Instructions.pdf                                   │   │
│   │                                                                      │   │
│   │ [+] Dataverse Tables                                                │   │
│   │     • IT_Equipment (inventory lookup)                              │   │
│   │     • Known_Issues (common problems)                               │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   STEP 4: CREATE TOPICS                                                     │
│   ─────────────────────                                                     │
│   Topics define structured conversation flows                               │
│   (See detailed topic design below)                                        │
│                                                                              │
│   STEP 5: ADD ACTIONS                                                       │
│   ───────────────────                                                       │
│   Connect to external systems via Power Automate or plugins                │
│   (See actions section below)                                              │
│                                                                              │
│   STEP 6: TEST AND PUBLISH                                                  │
│   ────────────────────────                                                  │
│   • Use built-in test chat                                                 │
│   • Review conversation analytics                                          │
│   • Publish to channels (Teams, Web, etc.)                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Advanced Topic Design

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TOPIC: PASSWORD RESET REQUEST                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   TRIGGER PHRASES:                                                          │
│   • "I forgot my password"                                                  │
│   • "Reset my password"                                                     │
│   • "Can't log in to my account"                                           │
│   • "Password expired"                                                      │
│   • "Locked out of my account"                                             │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                     CONVERSATION FLOW                                │   │
│   │                                                                      │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │  [Trigger: Password reset intent detected]                   │   │   │
│   │   └─────────────────────────┬───────────────────────────────────┘   │   │
│   │                             │                                        │   │
│   │                             ▼                                        │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │  [Message]                                                   │   │   │
│   │   │  "I can help you reset your password. First, let me verify  │   │   │
│   │   │   your identity for security purposes."                      │   │   │
│   │   └─────────────────────────┬───────────────────────────────────┘   │   │
│   │                             │                                        │   │
│   │                             ▼                                        │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │  [Condition: Is user authenticated?]                         │   │   │
│   │   │                                                              │   │   │
│   │   │  System.User.IsAuthenticated == true                        │   │   │
│   │   └──────────────────┬─────────────────┬────────────────────────┘   │   │
│   │                      │ YES             │ NO                          │   │
│   │                      ▼                 ▼                             │   │
│   │   ┌──────────────────────┐  ┌──────────────────────────────────┐   │   │
│   │   │  [Get user from AAD] │  │  [Message]                        │   │   │
│   │   │                      │  │  "Please sign in first to         │   │   │
│   │   │  Use Graph API to    │  │   verify your identity."          │   │   │
│   │   │  get user details    │  │                                   │   │   │
│   │   └──────────┬───────────┘  │  [Authentication Node]            │   │   │
│   │              │              │  Redirect to Entra ID login       │   │   │
│   │              │              └──────────────────────────────────┘   │   │
│   │              │                                                      │   │
│   │              ▼                                                      │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │  [Question]                                                  │   │   │
│   │   │  "Which account needs a password reset?"                    │   │   │
│   │   │                                                              │   │   │
│   │   │  Options (Multiple choice):                                 │   │   │
│   │   │  • My work email ({System.User.Email})                     │   │   │
│   │   │  • VPN account                                              │   │   │
│   │   │  • Application-specific (SAP, Salesforce, etc.)            │   │   │
│   │   │                                                              │   │   │
│   │   │  Save to: Topic.AccountType                                 │   │   │
│   │   └─────────────────────────┬───────────────────────────────────┘   │   │
│   │                             │                                        │   │
│   │                             ▼                                        │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │  [Condition: Account type routing]                           │   │   │
│   │   │                                                              │   │   │
│   │   │  Topic.AccountType ==                                       │   │   │
│   │   └───────┬──────────────┬────────────────┬─────────────────────┘   │   │
│   │           │              │                │                          │   │
│   │     Work Email      VPN Account     Application                     │   │
│   │           │              │                │                          │   │
│   │           ▼              ▼                ▼                          │   │
│   │   ┌───────────────┐ ┌───────────────┐ ┌───────────────────────────┐ │   │
│   │   │ [Action]      │ │ [Action]      │ │ [Message]                 │ │   │
│   │   │ Power Automate│ │ Power Automate│ │ "For application-specific │ │   │
│   │   │ Flow:         │ │ Flow:         │ │  passwords, please visit  │ │   │
│   │   │ "AAD Password │ │ "VPN Reset"   │ │  the application portal   │ │   │
│   │   │  Reset"       │ │               │ │  or contact the app owner"│ │   │
│   │   │               │ │ Input: User   │ │                           │ │   │
│   │   │ Input: UPN    │ │ Output: Link  │ │ [Adaptive Card: App links]│ │   │
│   │   │ Output: Email │ │               │ │                           │ │   │
│   │   └───────┬───────┘ └───────┬───────┘ └───────────────────────────┘ │   │
│   │           │                 │                                        │   │
│   │           ▼                 ▼                                        │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │  [Message with Adaptive Card]                                │   │   │
│   │   │                                                              │   │   │
│   │   │  "I've initiated the password reset. You should receive     │   │   │
│   │   │   an email at {System.User.Email} within 5 minutes with     │   │   │
│   │   │   instructions to create a new password.                    │   │   │
│   │   │                                                              │   │   │
│   │   │   Reference: {Topic.TicketID}                               │   │   │
│   │   │                                                              │   │   │
│   │   │   [Button: Check Email]  [Button: Contact IT]"              │   │   │
│   │   └─────────────────────────┬───────────────────────────────────┘   │   │
│   │                             │                                        │   │
│   │                             ▼                                        │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │  [Question]                                                  │   │   │
│   │   │  "Is there anything else I can help you with?"              │   │   │
│   │   │                                                              │   │   │
│   │   │  If YES → Return to greeting                                │   │   │
│   │   │  If NO → End conversation                                   │   │   │
│   │   └─────────────────────────────────────────────────────────────┘   │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Power Automate Actions

```yaml
# Example Power Automate Flow for IT Ticketing

trigger:
  type: PowerVirtualAgents
  inputs:
    - name: subject
      type: string
    - name: description
      type: string
    - name: priority
      type: string
    - name: userEmail
      type: string

actions:
  - name: CreateServiceNowTicket
    type: ServiceNow.CreateRecord
    inputs:
      table: incident
      fields:
        short_description: "@{triggerBody()['subject']}"
        description: "@{triggerBody()['description']}"
        priority: "@{triggerBody()['priority']}"
        caller_email: "@{triggerBody()['userEmail']}"
        category: IT Support
        assignment_group: IT Helpdesk

  - name: GetTicketNumber
    type: Compose
    inputs: "@{body('CreateServiceNowTicket')?['number']}"

  - name: SendConfirmationEmail
    type: Office365.SendEmail
    inputs:
      to: "@{triggerBody()['userEmail']}"
      subject: "IT Support Ticket Created - @{outputs('GetTicketNumber')}"
      body: |
        Your IT support request has been submitted.

        Ticket Number: @{outputs('GetTicketNumber')}
        Subject: @{triggerBody()['subject']}
        Priority: @{triggerBody()['priority']}

        We will respond within our SLA timeframe.

outputs:
  - name: ticketNumber
    value: "@{outputs('GetTicketNumber')}"
  - name: status
    value: "Created"
```

---

## Microsoft 365 Integration Scenarios

### Extending M365 Copilot with Custom Copilots

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                M365 COPILOT EXTENSION SCENARIOS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   SCENARIO 1: DECLARATIVE AGENT (Copilot Plugin)                            │
│   ─────────────────────────────────────────────────                         │
│                                                                              │
│   User in M365 Copilot: "@HR Assistant what's the vacation policy?"        │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ M365 Copilot → Routes to HR Copilot → Queries knowledge base       │   │
│   │            → Returns answer with citations to M365 Copilot         │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   Configuration (declarative-agent.json):                                   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ {                                                                    │   │
│   │   "$schema": "...",                                                 │   │
│   │   "name": "HR Assistant",                                           │   │
│   │   "description": "Answers HR policy questions",                     │   │
│   │   "instructions": "You help employees understand HR policies...",   │   │
│   │   "conversation_starters": [                                        │   │
│   │     "What's the vacation policy?",                                  │   │
│   │     "How do I submit expenses?"                                     │   │
│   │   ],                                                                │   │
│   │   "capabilities": {                                                 │   │
│   │     "knowledge_sources": [                                          │   │
│   │       { "type": "SharePoint", "url": "...HR-Policies" }            │   │
│   │     ],                                                              │   │
│   │     "actions": ["SubmitTimeOff", "CheckBalance"]                   │   │
│   │   }                                                                 │   │
│   │ }                                                                    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   SCENARIO 2: TEAMS INTEGRATION                                             │
│   ─────────────────────────────                                             │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │   [Teams App]                                                       │   │
│   │   ┌───────────────────────────────────────────────────────────┐    │   │
│   │   │  IT Helpdesk                              [Pin to sidebar] │    │   │
│   │   │  ───────────────────────────────────────────────────────── │    │   │
│   │   │                                                            │    │   │
│   │   │  [Bot] Hi! I'm your IT assistant. How can I help?        │    │   │
│   │   │                                                            │    │   │
│   │   │  [User] My laptop is running slow                         │    │   │
│   │   │                                                            │    │   │
│   │   │  [Bot] I can help troubleshoot that. Let me gather        │    │   │
│   │   │        some information:                                   │    │   │
│   │   │                                                            │    │   │
│   │   │        ┌────────────────────────────────────────────┐     │    │   │
│   │   │        │ Laptop Model: [Detected: Dell XPS 15]     │     │    │   │
│   │   │        │ Last Restart: 14 days ago                  │     │    │   │
│   │   │        │ Disk Space: 12% free                       │     │    │   │
│   │   │        │                                             │     │    │   │
│   │   │        │ [Run Diagnostics]  [Create Ticket]         │     │    │   │
│   │   │        └────────────────────────────────────────────┘     │    │   │
│   │   │                                                            │    │   │
│   │   │  Type a message...                              [Send]    │    │   │
│   │   └───────────────────────────────────────────────────────────┘    │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   SCENARIO 3: PROACTIVE MESSAGING                                           │
│   ───────────────────────────────                                           │
│                                                                              │
│   Copilot initiates conversations based on events:                         │
│   • Password expiring in 3 days → Send reminder                            │
│   • New policy published → Notify affected users                           │
│   • System maintenance scheduled → Alert users                             │
│                                                                              │
│   Power Automate Trigger:                                                  │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ Trigger: When password expires in 3 days (Scheduled + AAD query)   │   │
│   │    ↓                                                                │   │
│   │ Action: Get user Teams ID                                          │   │
│   │    ↓                                                                │   │
│   │ Action: Send proactive message via Copilot                         │   │
│   │    "Hi {User}, your password expires on {Date}. Would you like     │   │
│   │     me to help you reset it now? [Yes] [Remind me tomorrow]"       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Adaptive Cards for Rich Interactions

```json
{
  "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
  "type": "AdaptiveCard",
  "version": "1.5",
  "body": [
    {
      "type": "TextBlock",
      "text": "IT Support Ticket",
      "weight": "Bolder",
      "size": "Large"
    },
    {
      "type": "FactSet",
      "facts": [
        { "title": "Ticket #", "value": "${ticketNumber}" },
        { "title": "Status", "value": "${status}" },
        { "title": "Priority", "value": "${priority}" },
        { "title": "Created", "value": "${createdDate}" }
      ]
    },
    {
      "type": "TextBlock",
      "text": "Description",
      "weight": "Bolder",
      "spacing": "Medium"
    },
    {
      "type": "TextBlock",
      "text": "${description}",
      "wrap": true
    },
    {
      "type": "ActionSet",
      "actions": [
        {
          "type": "Action.OpenUrl",
          "title": "View in Portal",
          "url": "${portalUrl}"
        },
        {
          "type": "Action.Submit",
          "title": "Escalate",
          "data": {
            "action": "escalate",
            "ticketId": "${ticketNumber}"
          }
        },
        {
          "type": "Action.Submit",
          "title": "Close Ticket",
          "data": {
            "action": "close",
            "ticketId": "${ticketNumber}"
          }
        }
      ]
    }
  ]
}
```

---

## Enterprise Deployment Patterns

### Multi-Environment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ENTERPRISE DEPLOYMENT ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   DEVELOPMENT ENVIRONMENT                                                   │
│   ─────────────────────────                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ Power Platform Environment: contoso-copilot-dev                     │   │
│   │                                                                      │   │
│   │ ┌───────────────┐  ┌───────────────┐  ┌───────────────┐            │   │
│   │ │  Copilot      │  │  Dataverse    │  │  Power        │            │   │
│   │ │  (Unmanaged)  │  │  (Dev data)   │  │  Automate     │            │   │
│   │ │               │  │               │  │  (Dev flows)  │            │   │
│   │ └───────────────┘  └───────────────┘  └───────────────┘            │   │
│   │                                                                      │   │
│   │ Users: Makers, Developers                                           │   │
│   │ Data: Sample/synthetic data only                                    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│                    Export Solution │ (Managed)                               │
│                                    ▼                                         │
│   TEST ENVIRONMENT                                                          │
│   ────────────────                                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ Power Platform Environment: contoso-copilot-test                    │   │
│   │                                                                      │   │
│   │ ┌───────────────┐  ┌───────────────┐  ┌───────────────┐            │   │
│   │ │  Copilot      │  │  Dataverse    │  │  Power        │            │   │
│   │ │  (Managed)    │  │  (Test data)  │  │  Automate     │            │   │
│   │ │               │  │               │  │  (Managed)    │            │   │
│   │ └───────────────┘  └───────────────┘  └───────────────┘            │   │
│   │                                                                      │   │
│   │ Testing:                                                            │   │
│   │ • Functional testing (conversation flows)                           │   │
│   │ • Integration testing (connectors, actions)                        │   │
│   │ • Performance testing (concurrent users)                           │   │
│   │ • Security testing (authentication, authorization)                 │   │
│   │                                                                      │   │
│   │ Users: QA team, selected pilot users                               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│                    Import Solution │ (Approved)                              │
│                                    ▼                                         │
│   PRODUCTION ENVIRONMENT                                                    │
│   ──────────────────────                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ Power Platform Environment: contoso-copilot-prod                    │   │
│   │                                                                      │   │
│   │ ┌───────────────┐  ┌───────────────┐  ┌───────────────┐            │   │
│   │ │  Copilot      │  │  Dataverse    │  │  Power        │            │   │
│   │ │  (Managed)    │  │  (Prod data)  │  │  Automate     │            │   │
│   │ │               │  │               │  │  (Managed)    │            │   │
│   │ └───────────────┘  └───────────────┘  └───────────────┘            │   │
│   │                                                                      │   │
│   │ Channels:                                                           │   │
│   │ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐               │   │
│   │ │  Teams   │ │   Web    │ │   M365   │ │ Mobile   │               │   │
│   │ │  App     │ │  Widget  │ │ Copilot  │ │   App    │               │   │
│   │ └──────────┘ └──────────┘ └──────────┘ └──────────┘               │   │
│   │                                                                      │   │
│   │ Users: All employees                                                │   │
│   │ Governance: Change management, approval workflows                   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### CI/CD with Azure DevOps

```yaml
# azure-pipelines.yml for Copilot Studio deployment

trigger:
  branches:
    include:
      - main
  paths:
    include:
      - solutions/copilot-solution/**

variables:
  - group: PowerPlatform-Variables
  - name: solutionName
    value: 'ContosoITCopilot'

stages:
  - stage: Build
    displayName: 'Export and Build Solution'
    jobs:
      - job: ExportSolution
        pool:
          vmImage: 'windows-latest'
        steps:
          - task: PowerPlatformToolInstaller@2
            inputs:
              DefaultVersion: true

          - task: PowerPlatformExportSolution@2
            inputs:
              authenticationType: 'PowerPlatformSPN'
              PowerPlatformSPN: 'PowerPlatform-Dev-Connection'
              Environment: '$(DevEnvironmentUrl)'
              SolutionName: '$(solutionName)'
              SolutionOutputFile: '$(Build.ArtifactStagingDirectory)/$(solutionName).zip'
              Managed: true

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)'
              ArtifactName: 'solution'

  - stage: DeployTest
    displayName: 'Deploy to Test'
    dependsOn: Build
    condition: succeeded()
    jobs:
      - deployment: DeployToTest
        environment: 'copilot-test'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: PowerPlatformToolInstaller@2
                  inputs:
                    DefaultVersion: true

                - task: PowerPlatformImportSolution@2
                  inputs:
                    authenticationType: 'PowerPlatformSPN'
                    PowerPlatformSPN: 'PowerPlatform-Test-Connection'
                    Environment: '$(TestEnvironmentUrl)'
                    SolutionInputFile: '$(Pipeline.Workspace)/solution/$(solutionName).zip'
                    OverwriteUnmanagedCustomizations: true

  - stage: DeployProd
    displayName: 'Deploy to Production'
    dependsOn: DeployTest
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployToProd
        environment: 'copilot-prod'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: PowerPlatformToolInstaller@2
                  inputs:
                    DefaultVersion: true

                - task: PowerPlatformImportSolution@2
                  inputs:
                    authenticationType: 'PowerPlatformSPN'
                    PowerPlatformSPN: 'PowerPlatform-Prod-Connection'
                    Environment: '$(ProdEnvironmentUrl)'
                    SolutionInputFile: '$(Pipeline.Workspace)/solution/$(solutionName).zip'
                    OverwriteUnmanagedCustomizations: true
```

---

## Authentication and Security

### Authentication Patterns

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COPILOT AUTHENTICATION PATTERNS                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   PATTERN 1: NO AUTHENTICATION (Public)                                     │
│   ─────────────────────────────────────                                     │
│   • Anyone can access the copilot                                          │
│   • Suitable for public FAQs, general information                          │
│   • No personalization, no secure actions                                  │
│                                                                              │
│   Use case: Public website chatbot, product information                    │
│                                                                              │
│   PATTERN 2: ENTRA ID AUTHENTICATION (Recommended)                         │
│   ────────────────────────────────────────────────                         │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │   User                Copilot              Entra ID                 │   │
│   │     │                    │                    │                     │   │
│   │     │  1. Start chat     │                    │                     │   │
│   │     │ ──────────────────>│                    │                     │   │
│   │     │                    │                    │                     │   │
│   │     │  2. Redirect to login                  │                     │   │
│   │     │ <─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│                     │   │
│   │     │                    │                    │                     │   │
│   │     │  3. Authenticate   │                    │                     │   │
│   │     │ ──────────────────────────────────────>│                     │   │
│   │     │                    │                    │                     │   │
│   │     │  4. ID Token + Access Token            │                     │   │
│   │     │ <──────────────────────────────────────│                     │   │
│   │     │                    │                    │                     │   │
│   │     │  5. Resume chat with token             │                     │   │
│   │     │ ──────────────────>│                    │                     │   │
│   │     │                    │                    │                     │   │
│   │     │  6. Validated user context available   │                     │   │
│   │     │    • System.User.Email                 │                     │   │
│   │     │    • System.User.DisplayName           │                     │   │
│   │     │    • System.User.Id (AAD Object ID)    │                     │   │
│   │     │                    │                    │                     │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   Configuration:                                                            │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ Settings > Security > Authentication                                │   │
│   │                                                                      │   │
│   │ Authentication: ● Authenticate with Microsoft                       │   │
│   │                 ○ No authentication                                 │   │
│   │                 ○ Authenticate manually                             │   │
│   │                                                                      │   │
│   │ Require users to sign in: [x] Yes                                  │   │
│   │                                                                      │   │
│   │ Scope: ● Organization (your tenant only)                           │   │
│   │        ○ Any Azure AD tenant                                       │   │
│   │                                                                      │   │
│   │ Service Provider: Microsoft Entra ID                               │   │
│   │ Client ID: [Auto-generated]                                        │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   PATTERN 3: MANUAL/CUSTOM AUTHENTICATION                                   │
│   ───────────────────────────────────────                                   │
│   • Use custom OAuth providers                                             │
│   • Integrate with existing identity systems                               │
│   • Requires custom authentication topic                                   │
│                                                                              │
│   Use case: B2C scenarios, non-Microsoft identity providers               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Security Best Practices

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COPILOT SECURITY CHECKLIST                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   AUTHENTICATION & AUTHORIZATION                                            │
│   ──────────────────────────────                                            │
│   [x] Enable Entra ID authentication for internal copilots                 │
│   [x] Configure appropriate tenant restrictions                            │
│   [x] Use Conditional Access policies for sensitive copilots               │
│   [x] Implement MFA for high-privilege actions                             │
│   [x] Define security groups for copilot access                            │
│                                                                              │
│   DATA PROTECTION                                                           │
│   ───────────────                                                           │
│   [x] Review knowledge sources for sensitive content                       │
│   [x] Implement DLP policies in Power Platform                             │
│   [x] Enable sensitivity labels on source documents                        │
│   [x] Audit data access through copilot interactions                       │
│   [x] Configure conversation transcript retention                          │
│                                                                              │
│   CONTENT SAFETY                                                            │
│   ──────────────                                                            │
│   [x] Enable content moderation (appropriate level)                        │
│   [x] Configure custom blocklists for sensitive terms                      │
│   [x] Set up alerts for content safety violations                          │
│   [x] Review and tune moderation settings regularly                        │
│                                                                              │
│   ACTION SECURITY                                                           │
│   ───────────────                                                           │
│   [x] Review all Power Automate flow permissions                           │
│   [x] Use least-privilege service accounts for integrations                │
│   [x] Implement approval workflows for sensitive actions                   │
│   [x] Log all actions for audit purposes                                   │
│   [x] Test error handling (don't leak sensitive info)                      │
│                                                                              │
│   GOVERNANCE                                                                │
│   ──────────                                                                │
│   [x] Document copilot purpose and data handling                           │
│   [x] Establish change management process                                  │
│   [x] Define ownership and support responsibilities                        │
│   [x] Create incident response procedures                                  │
│   [x] Schedule regular security reviews                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Use Cases and Examples

### HR Self-Service Copilot

| Feature | Implementation |
|---------|---------------|
| **Benefits Q&A** | Knowledge source: SharePoint HR policies site |
| **Time-off requests** | Power Automate → Workday/SAP SuccessFactors API |
| **Payroll questions** | Generative answers from payroll documentation |
| **Onboarding help** | Guided topic with checklist and resources |
| **Policy lookup** | AI Search across HR document library |
| **Manager escalation** | Handoff to live HR representative in Teams |

### IT Helpdesk Copilot

| Feature | Implementation |
|---------|---------------|
| **Password reset** | Power Automate → Microsoft Graph API → Entra ID |
| **Device issues** | Diagnostic flow with Adaptive Cards |
| **Software requests** | ServiceNow/Jira integration via connector |
| **Network problems** | Guided troubleshooting topic |
| **Equipment orders** | Approval workflow with manager notification |
| **Ticket status** | Real-time lookup from ITSM system |

### Customer Service Copilot

| Feature | Implementation |
|---------|---------------|
| **Product FAQ** | Knowledge from website and product docs |
| **Order status** | API call to order management system |
| **Returns/refunds** | Guided topic with RMA creation |
| **Shipping updates** | Integration with shipping carrier APIs |
| **Account management** | Authenticated self-service actions |
| **Human handoff** | Transfer to Dynamics 365 Contact Center |

### Sample Implementation: Order Status Lookup

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ORDER STATUS TOPIC IMPLEMENTATION                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Topic: Check Order Status                                                 │
│                                                                              │
│   Trigger Phrases:                                                          │
│   • "Where is my order"                                                    │
│   • "Track my shipment"                                                    │
│   • "Order status for [order number]"                                      │
│   • "When will my package arrive"                                          │
│                                                                              │
│   Flow:                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │   [Trigger] → [Check if order number provided]                      │   │
│   │                         │                                            │   │
│   │              ┌──────────┴──────────┐                                │   │
│   │              │                     │                                │   │
│   │          YES (entity              NO                                │   │
│   │          extracted)                │                                │   │
│   │              │                     ▼                                │   │
│   │              │            [Question: Ask for order number]         │   │
│   │              │            "What is your order number? It           │   │
│   │              │             starts with ORD-"                       │   │
│   │              │                     │                                │   │
│   │              └──────────┬──────────┘                                │   │
│   │                         │                                            │   │
│   │                         ▼                                            │   │
│   │   [Power Automate: Get Order Details]                               │   │
│   │   Input: Topic.OrderNumber                                          │   │
│   │   API: GET /api/orders/{orderId}                                    │   │
│   │   Output: orderData (JSON)                                          │   │
│   │                         │                                            │   │
│   │              ┌──────────┴──────────┐                                │   │
│   │              │                     │                                │   │
│   │         Found                 Not Found                            │   │
│   │              │                     │                                │   │
│   │              ▼                     ▼                                │   │
│   │   [Adaptive Card:            [Message: Order not found]            │   │
│   │    Order Details]            "I couldn't find that order.          │   │
│   │                               Please check the number and          │   │
│   │   ┌────────────────────┐     try again, or contact support."      │   │
│   │   │ Order: ORD-12345   │                                           │   │
│   │   │ Status: In Transit │                                           │   │
│   │   │ Est. Delivery: 1/30│                                           │   │
│   │   │                    │                                           │   │
│   │   │ Items:             │                                           │   │
│   │   │ • Widget Pro (x2)  │                                           │   │
│   │   │ • Gadget Plus (x1) │                                           │   │
│   │   │                    │                                           │   │
│   │   │ [Track Package]    │                                           │   │
│   │   │ [Contact Support]  │                                           │   │
│   │   └────────────────────┘                                           │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Analytics and Monitoring

### Key Metrics to Track

| Metric | Description | Target |
|--------|-------------|--------|
| **Resolution rate** | % of conversations resolved without escalation | > 70% |
| **Customer satisfaction** | Post-conversation survey score | > 4.0/5.0 |
| **Engagement rate** | % of sessions with 2+ user messages | > 60% |
| **Escalation rate** | % handed off to human agents | < 20% |
| **Abandonment rate** | % of incomplete conversations | < 15% |
| **Response accuracy** | % of correct/helpful responses | > 90% |
| **Avg. resolution time** | Time from start to resolution | < 3 minutes |

### Analytics Dashboard Queries

```kusto
// Copilot Analytics - Power BI Dataflows or Azure Data Explorer

// Conversation outcomes by topic
ConversationTranscripts
| where Timestamp > ago(30d)
| summarize
    TotalSessions = dcount(ConversationId),
    Resolved = countif(Outcome == "Resolved"),
    Escalated = countif(Outcome == "Escalated"),
    Abandoned = countif(Outcome == "Abandoned")
    by TopicName
| extend ResolutionRate = round(100.0 * Resolved / TotalSessions, 1)
| order by TotalSessions desc

// Hourly usage patterns
ConversationTranscripts
| where Timestamp > ago(7d)
| extend Hour = datetime_part("hour", Timestamp)
| summarize Sessions = dcount(ConversationId) by Hour
| order by Hour asc
| render columnchart

// User satisfaction trends
SurveyResponses
| where Timestamp > ago(90d)
| summarize
    AvgSatisfaction = avg(Rating),
    Responses = count()
    by bin(Timestamp, 1d)
| render timechart

// Most common unresolved intents
ConversationTranscripts
| where Timestamp > ago(30d)
| where Outcome == "Unresolved" or Outcome == "NoMatch"
| summarize Count = count() by UserMessage
| top 20 by Count desc
```

---

*Continue to [Responsible AI](05-responsible-ai.md)*

*Back to [Azure AI Services](03-ai-services.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
