# Chapter 05: AI Platform

## Overview

Azure's AI platform provides enterprise-grade tools for building, deploying, and managing AI solutions. This chapter focuses on Azure AI Foundry (formerly Azure AI Studio), Copilot Studio, and Azure OpenAI Service - the key components for "AI in the Enterprise" that modern Cloud Architects need to understand.

## AWS to Azure AI Mapping

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       AI SERVICES: AWS vs AZURE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS                                   AZURE                                │
│  ───                                   ─────                                │
│                                                                              │
│  Amazon Bedrock          ──────────▶   Azure OpenAI Service                 │
│  (Foundation models)                   (GPT-4, DALL-E, etc.)                │
│                                                                              │
│  Amazon SageMaker        ──────────▶   Azure Machine Learning               │
│  (ML platform)                         Azure AI Foundry                     │
│                                                                              │
│  AWS AI Services         ──────────▶   Azure AI Services                    │
│  (Rekognition, Textract)              (Vision, Speech, Language)            │
│                                                                              │
│  Amazon Q                ──────────▶   Microsoft Copilot / Copilot Studio   │
│  (AI assistant)                        (Custom copilots)                    │
│                                                                              │
│  Amazon Lex              ──────────▶   Copilot Studio / Bot Framework       │
│  (Conversational AI)                   (Low-code bots)                      │
│                                                                              │
│  KEY DIFFERENCE:                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  AWS: Multiple separate services for AI                              │   │
│  │       Bedrock for models, SageMaker for ML, Lex for bots            │   │
│  │                                                                       │   │
│  │  Azure: Unified platform approach                                    │   │
│  │         AI Foundry brings everything together                        │   │
│  │         Deep Microsoft 365 integration                               │   │
│  │         Copilot ecosystem across all products                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Azure AI Platform Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AZURE AI PLATFORM ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     AZURE AI FOUNDRY                                  │   │
│  │                  (Unified AI Development Platform)                    │   │
│  │                                                                       │   │
│  │  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐              │   │
│  │  │ Model Catalog │ │ Prompt Flow   │ │ Evaluations   │              │   │
│  │  │               │ │               │ │               │              │   │
│  │  │ GPT-4, Llama  │ │ Build AI apps │ │ Test quality  │              │   │
│  │  │ Mistral, etc. │ │ with RAG      │ │ & safety      │              │   │
│  │  └───────────────┘ └───────────────┘ └───────────────┘              │   │
│  │                                                                       │   │
│  │  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐              │   │
│  │  │ Fine-tuning   │ │ Deployments   │ │ Content       │              │   │
│  │  │               │ │               │ │ Safety        │              │   │
│  │  │ Custom models │ │ Real-time &   │ │               │              │   │
│  │  │               │ │ Batch         │ │ Filters &     │              │   │
│  │  └───────────────┘ └───────────────┘ │ guardrails    │              │   │
│  │                                       └───────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌───────────────────────────┐    ┌───────────────────────────────────┐   │
│  │      AZURE OPENAI          │    │      COPILOT STUDIO               │   │
│  │                            │    │                                   │   │
│  │  • GPT-4, GPT-4o, GPT-4   │    │  • Low-code/no-code copilots     │   │
│  │  • DALL-E 3               │    │  • Microsoft 365 integration     │   │
│  │  • Whisper                │    │  • Power Platform connectors     │   │
│  │  • Embeddings             │    │  • Custom plugins & actions      │   │
│  │                            │    │  • Teams, web, mobile           │   │
│  │  Enterprise features:      │    │                                   │   │
│  │  • Private endpoints      │    │  Built on:                        │   │
│  │  • Data stays in region   │    │  • Azure OpenAI                  │   │
│  │  • No training on data    │    │  • Dataverse                     │   │
│  │  • SOC 2, HIPAA, etc.     │    │  • Power Automate               │   │
│  └───────────────────────────┘    └───────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     AZURE AI SERVICES                                 │   │
│  │                  (Pre-built AI capabilities)                          │   │
│  │                                                                       │   │
│  │  Vision          Language        Speech          Decision            │   │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐         │   │
│  │  │ Image    │   │ Text     │   │ Speech   │   │ Content  │         │   │
│  │  │ analysis │   │ analytics│   │ to text  │   │ moderator│         │   │
│  │  │ OCR      │   │ QnA      │   │ Text to  │   │ Anomaly  │         │   │
│  │  │ Face     │   │ Language │   │ speech   │   │ detector │         │   │
│  │  │ Custom   │   │ Translate│   │ Speaker  │   │ Personal-│         │   │
│  │  │ Vision   │   │          │   │ recog    │   │ izer     │         │   │
│  │  └──────────┘   └──────────┘   └──────────┘   └──────────┘         │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Chapter Contents

### Quick Reference
- [Quick Reference](quick-reference.md) - Key concepts, models, and pricing

### Deep Dive Topics
1. [Azure AI Foundry](01-ai-foundry.md) - Unified AI development platform
2. [Azure OpenAI Service](02-azure-openai.md) - Enterprise GPT-4 and foundation models
3. [Copilot Studio](03-copilot-studio.md) - Building custom copilots
4. [Responsible AI](04-responsible-ai.md) - Governance, safety, and ethics
5. [Case Studies](case-studies.md) - Enterprise AI implementations

## Key Concepts for Cloud Architects

### AI Project Lifecycle

```
AI PROJECT LIFECYCLE IN AZURE:
──────────────────────────────

Phase 1: DISCOVER & EVALUATE
┌────────────────────────────────────────────────────────────────────────────┐
│ Azure AI Foundry Model Catalog                                              │
│                                                                             │
│ • Browse foundation models (GPT-4, Llama, Mistral, etc.)                  │
│ • Compare capabilities and costs                                           │
│ • Test in playground                                                       │
│ • Evaluate with your data                                                  │
└────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
Phase 2: BUILD & TEST
┌────────────────────────────────────────────────────────────────────────────┐
│ Azure AI Foundry + Prompt Flow                                              │
│                                                                             │
│ • Create AI application flows                                              │
│ • Add RAG (Retrieval Augmented Generation)                                │
│ • Connect to your data sources                                             │
│ • Iterate on prompts                                                       │
│ • Evaluate quality and safety                                              │
└────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
Phase 3: DEPLOY & SCALE
┌────────────────────────────────────────────────────────────────────────────┐
│ Azure AI Foundry Deployments                                                │
│                                                                             │
│ • Deploy to real-time endpoints                                            │
│ • Configure autoscaling                                                    │
│ • Set up content filtering                                                 │
│ • Enable monitoring                                                        │
└────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
Phase 4: MONITOR & GOVERN
┌────────────────────────────────────────────────────────────────────────────┐
│ Azure Monitor + AI Foundry + Responsible AI                                │
│                                                                             │
│ • Track usage and costs                                                    │
│ • Monitor for quality degradation                                          │
│ • Audit AI decisions                                                       │
│ • Ensure content safety                                                    │
│ • Iterate and improve                                                      │
└────────────────────────────────────────────────────────────────────────────┘
```

### Enterprise AI Architecture Patterns

```
PATTERN 1: SIMPLE Q&A / CHATBOT
───────────────────────────────
User → Copilot Studio → Azure OpenAI
                      → Dataverse (knowledge base)

Best for: Customer service, IT helpdesk, HR FAQ
Complexity: Low
Time to deploy: Days


PATTERN 2: RAG (RETRIEVAL AUGMENTED GENERATION)
────────────────────────────────────────────────
User → App → Azure AI Search → Azure OpenAI
                    ↑
            Your documents (Blob, SharePoint, etc.)

Best for: Searching internal documents, personalized answers
Complexity: Medium
Time to deploy: Weeks


PATTERN 3: AGENTIC AI / COPILOT
───────────────────────────────
User → Custom Copilot → Azure OpenAI (orchestration)
                      → Function calling → Business APIs
                      → Azure AI Search → Knowledge base
                      → Power Automate → Actions

Best for: Complex multi-step tasks, process automation
Complexity: High
Time to deploy: Months


PATTERN 4: FINE-TUNED / CUSTOM MODEL
────────────────────────────────────
Your data → Azure AI Foundry → Fine-tune GPT-4
                             → Deploy custom model
                             → Application

Best for: Domain-specific language, brand voice, specialized tasks
Complexity: Very High
Time to deploy: Months
```

## Learning Path

```
RECOMMENDED STUDY ORDER:
────────────────────────

Week 1: Azure OpenAI Fundamentals
├── Read: Quick Reference (models, pricing)
├── Do: Create Azure OpenAI resource
├── Do: Test in Azure AI Foundry playground
└── Practice: Build simple chat completion

Week 2: Building with AI Foundry
├── Read: AI Foundry deep dive
├── Do: Create AI Foundry project
├── Do: Build Prompt Flow with RAG
└── Practice: Connect to your documents

Week 3: Copilot Studio
├── Read: Copilot Studio deep dive
├── Do: Create a custom copilot
├── Do: Integrate with Microsoft 365
└── Practice: Add custom actions

Week 4: Responsible AI & Production
├── Read: Responsible AI deep dive
├── Do: Implement content filtering
├── Do: Set up monitoring
├── Review: Case studies
└── Practice: Deploy to production
```

## Key Services Summary

| Service | Purpose | Use When | Cost Model |
|---------|---------|----------|------------|
| Azure OpenAI | Access GPT-4, DALL-E | Need foundation models | Per 1K tokens |
| AI Foundry | Build AI apps | Full development lifecycle | Free (pay for compute) |
| Copilot Studio | Low-code copilots | M365 integration, rapid deploy | Per message |
| AI Services | Pre-built APIs | Specific tasks (OCR, speech) | Per transaction |
| AI Search | Vector search | RAG, document search | Per tier + storage |

## Security & Compliance

```
ENTERPRISE AI SECURITY:
───────────────────────

DATA PROTECTION:
• Your data stays in your tenant
• Azure OpenAI doesn't train on customer data
• Private endpoints supported
• Customer-managed keys available

COMPLIANCE:
• SOC 2 Type 2
• ISO 27001
• HIPAA (with BAA)
• GDPR compliant
• FedRAMP High

ACCESS CONTROL:
• Azure RBAC integration
• Managed identities supported
• Entra ID authentication
• Resource-level permissions

CONTENT SAFETY:
• Built-in content filtering
• Customizable filters
• Jailbreak protection
• Harmful content blocking
```

---

*Next: [Quick Reference](quick-reference.md)* | *Back to [Chapter 04: Observability](../04-observability/README.md)*
