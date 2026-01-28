# AI Platform Quick Reference

## Azure OpenAI Models

### Model Comparison

```
AZURE OPENAI MODEL SELECTION:
─────────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│ Model          │ Best For                    │ Context  │ Cost (1K tokens)│
├────────────────┼─────────────────────────────┼──────────┼─────────────────┤
│ GPT-4o         │ Most capable, multimodal    │ 128K     │ $5/$15 (in/out) │
│ GPT-4o-mini    │ Fast, cost-effective        │ 128K     │ $0.15/$0.60     │
│ GPT-4 Turbo    │ Complex reasoning           │ 128K     │ $10/$30         │
│ GPT-4          │ High accuracy tasks         │ 8K/32K   │ $30/$60 (32K)   │
│ GPT-3.5 Turbo  │ Simple tasks, high volume   │ 16K      │ $0.50/$1.50     │
├────────────────┼─────────────────────────────┼──────────┼─────────────────┤
│ text-embedding │ Vector embeddings           │ 8K       │ $0.13           │
│ -3-large       │ (semantic search)           │          │                 │
│ text-embedding │ Smaller embeddings          │ 8K       │ $0.02           │
│ -3-small       │ (cost-effective)            │          │                 │
├────────────────┼─────────────────────────────┼──────────┼─────────────────┤
│ DALL-E 3       │ Image generation            │ N/A      │ $0.04-0.12/img  │
│ Whisper        │ Speech-to-text              │ N/A      │ $0.006/minute   │
└────────────────────────────────────────────────────────────────────────────┘

QUICK DECISION GUIDE:
─────────────────────

Need multimodal (images + text)?     → GPT-4o
Need highest accuracy?               → GPT-4 Turbo or GPT-4o
Need fast & cheap?                   → GPT-4o-mini or GPT-3.5 Turbo
Need embeddings for search?          → text-embedding-3-large/small
Need image generation?               → DALL-E 3
Need speech transcription?           → Whisper
```

### AWS Bedrock to Azure OpenAI Mapping

```
┌────────────────────────────────────────────────────────────────────────────┐
│ AWS Bedrock Model      │ Azure OpenAI Equivalent    │ Notes               │
├────────────────────────┼────────────────────────────┼─────────────────────┤
│ Claude 3 (Anthropic)   │ GPT-4o / GPT-4 Turbo      │ Similar capability  │
│ Claude Instant         │ GPT-4o-mini / GPT-3.5     │ Faster, cheaper     │
│ Titan Text             │ GPT-3.5 Turbo             │ General purpose     │
│ Titan Embeddings       │ text-embedding-3-*        │ Vector embeddings   │
│ Stable Diffusion       │ DALL-E 3                  │ Image generation    │
│ Llama 2/3              │ (in AI Foundry catalog)   │ Open source option  │
│ Mistral                │ (in AI Foundry catalog)   │ Open source option  │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Azure CLI Commands

### Azure OpenAI

```bash
# Create Azure OpenAI resource
az cognitiveservices account create \
  --name my-openai \
  --resource-group myRG \
  --kind OpenAI \
  --sku S0 \
  --location eastus \
  --custom-domain my-openai

# Deploy a model
az cognitiveservices account deployment create \
  --name my-openai \
  --resource-group myRG \
  --deployment-name gpt-4o-deployment \
  --model-name gpt-4o \
  --model-version "2024-05-13" \
  --model-format OpenAI \
  --sku-name Standard \
  --sku-capacity 10  # TPM in thousands (10 = 10K tokens/min)

# List deployments
az cognitiveservices account deployment list \
  --name my-openai \
  --resource-group myRG \
  --output table

# Get endpoint and key
az cognitiveservices account show \
  --name my-openai \
  --resource-group myRG \
  --query properties.endpoint -o tsv

az cognitiveservices account keys list \
  --name my-openai \
  --resource-group myRG \
  --query key1 -o tsv
```

### AI Foundry (via Azure CLI ML extension)

```bash
# Install ML extension
az extension add -n ml

# Create AI Foundry hub (workspace)
az ml workspace create \
  --name my-ai-hub \
  --resource-group myRG \
  --kind hub \
  --location eastus

# Create AI Foundry project
az ml workspace create \
  --name my-ai-project \
  --resource-group myRG \
  --kind project \
  --hub-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.MachineLearningServices/workspaces/my-ai-hub

# List connections (to Azure OpenAI, etc.)
az ml connection list \
  --workspace-name my-ai-hub \
  --resource-group myRG
```

---

## Code Examples

### Python SDK - Chat Completion

```python
from openai import AzureOpenAI

# Initialize client
client = AzureOpenAI(
    api_key="your-api-key",  # or use Azure AD
    api_version="2024-02-01",
    azure_endpoint="https://your-resource.openai.azure.com"
)

# Simple chat completion
response = client.chat.completions.create(
    model="gpt-4o-deployment",  # deployment name
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is Azure AI Foundry?"}
    ],
    temperature=0.7,
    max_tokens=500
)

print(response.choices[0].message.content)
```

### Python SDK - With Azure AD Authentication

```python
from openai import AzureOpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

# Use managed identity or Azure CLI credential
credential = DefaultAzureCredential()
token_provider = get_bearer_token_provider(
    credential, "https://cognitiveservices.azure.com/.default"
)

client = AzureOpenAI(
    azure_ad_token_provider=token_provider,
    api_version="2024-02-01",
    azure_endpoint="https://your-resource.openai.azure.com"
)

# Same API calls work with AD auth
response = client.chat.completions.create(
    model="gpt-4o-deployment",
    messages=[{"role": "user", "content": "Hello!"}]
)
```

### Python - RAG with AI Search

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    api_key="your-api-key",
    api_version="2024-02-01",
    azure_endpoint="https://your-resource.openai.azure.com"
)

# Chat completion with Azure AI Search as data source
response = client.chat.completions.create(
    model="gpt-4o-deployment",
    messages=[
        {"role": "user", "content": "What are our company's vacation policies?"}
    ],
    extra_body={
        "data_sources": [
            {
                "type": "azure_search",
                "parameters": {
                    "endpoint": "https://your-search.search.windows.net",
                    "index_name": "company-policies",
                    "authentication": {
                        "type": "api_key",
                        "key": "your-search-key"
                    }
                }
            }
        ]
    }
)

# Response includes citations
print(response.choices[0].message.content)
print(response.choices[0].message.context)  # Source citations
```

### C# SDK - Chat Completion

```csharp
using Azure;
using Azure.AI.OpenAI;

// Initialize client
var client = new OpenAIClient(
    new Uri("https://your-resource.openai.azure.com"),
    new AzureKeyCredential("your-api-key")
);

// Chat completion
var chatOptions = new ChatCompletionsOptions()
{
    DeploymentName = "gpt-4o-deployment",
    Messages = {
        new ChatRequestSystemMessage("You are a helpful assistant."),
        new ChatRequestUserMessage("What is Azure AI Foundry?")
    },
    Temperature = 0.7f,
    MaxTokens = 500
};

Response<ChatCompletions> response = await client.GetChatCompletionsAsync(chatOptions);
Console.WriteLine(response.Value.Choices[0].Message.Content);
```

### JavaScript/TypeScript SDK

```typescript
import { OpenAI } from "openai";

const client = new OpenAI({
  apiKey: process.env.AZURE_OPENAI_API_KEY,
  baseURL: `${process.env.AZURE_OPENAI_ENDPOINT}/openai/deployments/${deploymentName}`,
  defaultQuery: { "api-version": "2024-02-01" },
  defaultHeaders: { "api-key": process.env.AZURE_OPENAI_API_KEY },
});

const response = await client.chat.completions.create({
  model: "gpt-4o-deployment",
  messages: [
    { role: "system", content: "You are a helpful assistant." },
    { role: "user", content: "What is Azure AI Foundry?" }
  ],
});

console.log(response.choices[0].message.content);
```

---

## Copilot Studio Quick Start

### Creating a Copilot

```
COPILOT STUDIO WORKFLOW:
────────────────────────

1. CREATE COPILOT
   ┌─────────────────────────────────────────────────────────────────────┐
   │ copilot.microsoft.com → Create → Name & describe your copilot      │
   │                                                                       │
   │ Options:                                                             │
   │ • Template: Start from template (HR, IT, Customer Service)          │
   │ • Blank: Start from scratch                                         │
   │ • Import: Import existing Power Virtual Agents bot                  │
   └─────────────────────────────────────────────────────────────────────┘

2. ADD KNOWLEDGE SOURCES
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Knowledge → Add knowledge                                           │
   │                                                                       │
   │ Sources:                                                             │
   │ • SharePoint sites (auto-indexes documents)                         │
   │ • Public websites (crawls and indexes)                              │
   │ • Dataverse tables                                                  │
   │ • Files (upload directly)                                           │
   │ • Custom connectors (APIs)                                          │
   └─────────────────────────────────────────────────────────────────────┘

3. CREATE TOPICS (Conversations)
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Topics → New topic → From blank                                     │
   │                                                                       │
   │ Components:                                                          │
   │ • Trigger phrases: "How do I reset my password?"                    │
   │ • Message nodes: Bot responses                                      │
   │ • Question nodes: Collect user input                                │
   │ • Condition nodes: Branch logic                                     │
   │ • Action nodes: Call APIs, Power Automate flows                    │
   └─────────────────────────────────────────────────────────────────────┘

4. ADD ACTIONS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Actions → Add an action                                             │
   │                                                                       │
   │ Types:                                                               │
   │ • Power Automate flows (300+ connectors)                           │
   │ • Custom connectors (your APIs)                                     │
   │ • Prebuilt actions (send email, create ticket)                     │
   │ • Plugin actions (Copilot plugins)                                  │
   └─────────────────────────────────────────────────────────────────────┘

5. PUBLISH & DEPLOY
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Publish → Publish                                                   │
   │                                                                       │
   │ Channels:                                                            │
   │ • Microsoft Teams (most common enterprise)                          │
   │ • Website (embed widget)                                            │
   │ • Mobile app                                                        │
   │ • Facebook, etc.                                                    │
   └─────────────────────────────────────────────────────────────────────┘
```

---

## Content Safety Configuration

```
CONTENT FILTERING LEVELS:
─────────────────────────

Azure OpenAI includes built-in content filters for all deployments.

DEFAULT FILTERS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Category        │ Default Action │ Severity Levels Blocked               │
├─────────────────┼────────────────┼───────────────────────────────────────┤
│ Hate            │ Block          │ Medium, High                          │
│ Violence        │ Block          │ Medium, High                          │
│ Sexual          │ Block          │ Medium, High                          │
│ Self-harm       │ Block          │ Medium, High                          │
│ Jailbreak       │ Block          │ All attempts                          │
│ Protected IP    │ Block          │ All (code, lyrics, etc.)             │
└────────────────────────────────────────────────────────────────────────────┘

CUSTOM CONTENT FILTER (Portal):
1. Azure AI Foundry → Project → Content filters
2. Create filter configuration
3. Adjust severity thresholds per category
4. Add blocklists (specific words/phrases)
5. Apply to deployment

# CLI: Associate filter with deployment
az cognitiveservices account deployment update \
  --name my-openai \
  --resource-group myRG \
  --deployment-name gpt-4o-deployment \
  --content-filter-policy-name my-custom-filter
```

---

## Pricing Quick Reference

```
AZURE OPENAI PRICING (East US, approximate):
────────────────────────────────────────────

PER 1,000 TOKENS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Model                │ Input           │ Output                           │
├──────────────────────┼─────────────────┼──────────────────────────────────┤
│ GPT-4o               │ $0.005          │ $0.015                           │
│ GPT-4o-mini          │ $0.00015        │ $0.0006                          │
│ GPT-4 Turbo          │ $0.01           │ $0.03                            │
│ GPT-4 (8K)           │ $0.03           │ $0.06                            │
│ GPT-4 (32K)          │ $0.06           │ $0.12                            │
│ GPT-3.5 Turbo        │ $0.0005         │ $0.0015                          │
│ text-embedding-3-large│ $0.00013       │ N/A                              │
│ text-embedding-3-small│ $0.00002       │ N/A                              │
└────────────────────────────────────────────────────────────────────────────┘

PROVISIONED THROUGHPUT (PTU):
• For predictable, high-volume workloads
• Reserved capacity, hourly rate
• ~$0.07/PTU-hour (varies by model)
• 1 PTU ≈ 1-6 requests/second (model dependent)

IMAGE GENERATION (DALL-E 3):
• Standard: $0.04/image (1024x1024)
• HD: $0.08/image (1024x1024)
• HD: $0.12/image (1792x1024 or 1024x1792)

COPILOT STUDIO:
• Included with Microsoft 365 Copilot license
• Or standalone: ~$200/tenant/month + $0.01/message
```

---

## Common Patterns

### RAG Architecture

```
RAG (RETRIEVAL AUGMENTED GENERATION):
─────────────────────────────────────

User Question → Embedding → Vector Search → Context → LLM → Answer

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  INGESTION PIPELINE:                                                        │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────────────┐  │
│  │ Documents│───▶│ Chunking │───▶│ Embedding│───▶│ Azure AI Search      │  │
│  │ (Blob)   │    │ (split)  │    │ (vectors)│    │ (vector index)       │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────────────────┘  │
│                                                                              │
│  QUERY PIPELINE:                                                            │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────────────┐  │
│  │ User     │───▶│ Embedding│───▶│ Vector   │───▶│ Top K results        │  │
│  │ Question │    │          │    │ Search   │    │ (context)            │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┬───────────┘  │
│                                                              │              │
│                                                              ▼              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ PROMPT:                                                               │   │
│  │ System: Answer based on the following context. If not in context,    │   │
│  │         say "I don't have that information."                         │   │
│  │                                                                       │   │
│  │ Context: [Retrieved document chunks]                                 │   │
│  │                                                                       │   │
│  │ User: [Original question]                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                              │              │
│                                                              ▼              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ Azure OpenAI → Grounded Answer (with citations)                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Function Calling

```python
# Function calling allows the model to call your functions

import json
from openai import AzureOpenAI

client = AzureOpenAI(...)

# Define available functions
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get the current weather in a location",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "The city name"
                    }
                },
                "required": ["location"]
            }
        }
    }
]

# Call with tools
response = client.chat.completions.create(
    model="gpt-4o-deployment",
    messages=[{"role": "user", "content": "What's the weather in Seattle?"}],
    tools=tools,
    tool_choice="auto"
)

# Check if model wants to call a function
if response.choices[0].message.tool_calls:
    tool_call = response.choices[0].message.tool_calls[0]
    function_name = tool_call.function.name
    arguments = json.loads(tool_call.function.arguments)

    # Execute your function
    result = get_weather(arguments["location"])

    # Send result back to model
    messages.append(response.choices[0].message)
    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": json.dumps(result)
    })

    final_response = client.chat.completions.create(
        model="gpt-4o-deployment",
        messages=messages
    )
```

---

*Next: [Azure AI Foundry](01-ai-foundry.md)* | *Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
