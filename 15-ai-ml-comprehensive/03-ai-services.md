# Azure AI Services Comprehensive Guide

## Overview

Azure AI Services provides a comprehensive suite of AI capabilities that enable developers to build intelligent applications. This guide covers everything from Azure AI Foundry to individual Cognitive Services, with deployment patterns and best practices for enterprise integration.

## Azure AI Foundry

### What is Azure AI Foundry?

Azure AI Foundry (formerly Azure AI Studio) is Microsoft's unified platform for building, evaluating, and deploying generative AI applications. It brings together model catalog, prompt engineering, fine-tuning, and deployment management into a single experience.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AZURE AI FOUNDRY ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                           AI FOUNDRY HUB                             │   │
│   │   (Central governance, shared resources, connections)                │   │
│   │                                                                      │   │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │   │
│   │   │  Connections │  │   Compute    │  │   Security   │             │   │
│   │   │ • OpenAI     │  │ • Serverless │  │ • RBAC       │             │   │
│   │   │ • AI Search  │  │ • Dedicated  │  │ • Private    │             │   │
│   │   │ • Storage    │  │ • GPU/CPU    │  │   Endpoints  │             │   │
│   │   └──────────────┘  └──────────────┘  └──────────────┘             │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│           ┌────────────────────────┼────────────────────────┐               │
│           │                        │                        │               │
│           ▼                        ▼                        ▼               │
│   ┌───────────────┐        ┌───────────────┐        ┌───────────────┐      │
│   │   PROJECT A   │        │   PROJECT B   │        │   PROJECT C   │      │
│   │   (Chatbot)   │        │   (RAG App)   │        │   (Agents)    │      │
│   │               │        │               │        │               │      │
│   │ • Models      │        │ • Models      │        │ • Models      │      │
│   │ • Prompts     │        │ • Prompts     │        │ • Prompts     │      │
│   │ • Data        │        │ • Data        │        │ • Data        │      │
│   │ • Deployments │        │ • Deployments │        │ • Deployments │      │
│   └───────────────┘        └───────────────┘        └───────────────┘      │
│                                                                              │
│   KEY CAPABILITIES:                                                         │
│   ─────────────────                                                         │
│   • Model Catalog: GPT-4, Llama, Mistral, Phi, custom models               │
│   • Prompt Flow: Visual orchestration for AI workflows                      │
│   • Fine-tuning: Customize models with your data                           │
│   • Evaluations: Test quality, safety, and performance                     │
│   • Deployments: Managed endpoints with scaling                            │
│   • Tracing: Debug and monitor AI pipelines                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### AI Foundry Model Catalog

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          MODEL CATALOG OVERVIEW                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   OPENAI MODELS                     META LLAMA MODELS                       │
│   ─────────────────                 ─────────────────                       │
│   • GPT-4o (Latest flagship)        • Llama 3.1 405B (Largest)             │
│   • GPT-4o-mini (Cost efficient)    • Llama 3.1 70B (Balanced)             │
│   • GPT-4 Turbo (128K context)      • Llama 3.1 8B (Fast)                  │
│   • o1 (Reasoning model)            • Code Llama (Coding)                  │
│   • DALL-E 3 (Image generation)                                             │
│   • Whisper (Speech to text)        MISTRAL MODELS                         │
│   • text-embedding-3 (Embeddings)   ───────────────                        │
│                                     • Mistral Large (Flagship)              │
│   MICROSOFT MODELS                  • Mistral Small (Efficient)             │
│   ─────────────────                 • Mixtral 8x22B (MoE)                   │
│   • Phi-3-medium (14B params)                                               │
│   • Phi-3-small (7B params)         OTHER MODELS                            │
│   • Phi-3-mini (3.8B params)        ────────────                            │
│   • Florence (Vision)               • Cohere (Generation, Embeddings)      │
│   • Phi-3-vision (Multimodal)       • Jais (Arabic language)               │
│                                     • DBRX (Databricks)                     │
│                                                                              │
│   DEPLOYMENT OPTIONS:                                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ Serverless API        │ Pay-per-token, instant deployment           │   │
│   │ Managed Compute       │ Dedicated capacity, fine-tuned models       │   │
│   │ Self-hosted           │ Your infrastructure (AKS, VMs)              │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Deploying AI Foundry with Bicep

```bicep
// ai-foundry-infrastructure.bicep
@description('Location for all resources')
param location string = resourceGroup().location

@description('Environment name')
param environmentName string = 'prod'

@description('Enable public network access')
param publicNetworkAccess string = 'Disabled'

// AI Foundry Hub
resource aiHub 'Microsoft.MachineLearningServices/workspaces@2024-04-01' = {
  name: 'ai-hub-${environmentName}'
  location: location
  kind: 'Hub'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    friendlyName: 'AI Foundry Hub - ${environmentName}'
    description: 'Central hub for AI projects'
    publicNetworkAccess: publicNetworkAccess
    managedNetwork: {
      isolationMode: 'AllowInternetOutbound'
    }
  }
}

// AI Foundry Project
resource aiProject 'Microsoft.MachineLearningServices/workspaces@2024-04-01' = {
  name: 'ai-project-${environmentName}'
  location: location
  kind: 'Project'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    friendlyName: 'AI Project - ${environmentName}'
    hubResourceId: aiHub.id
    publicNetworkAccess: publicNetworkAccess
  }
}

// Azure OpenAI Connection
resource openAI 'Microsoft.CognitiveServices/accounts@2024-04-01-preview' = {
  name: 'aoai-${environmentName}-${uniqueString(resourceGroup().id)}'
  location: location
  kind: 'OpenAI'
  sku: {
    name: 'S0'
  }
  properties: {
    customSubDomainName: 'aoai-${environmentName}-${uniqueString(resourceGroup().id)}'
    publicNetworkAccess: publicNetworkAccess
    networkAcls: {
      defaultAction: 'Deny'
    }
  }
}

// GPT-4o Deployment
resource gpt4oDeployment 'Microsoft.CognitiveServices/accounts/deployments@2024-04-01-preview' = {
  parent: openAI
  name: 'gpt-4o'
  properties: {
    model: {
      format: 'OpenAI'
      name: 'gpt-4o'
      version: '2024-08-06'
    }
    raiPolicyName: 'Microsoft.DefaultV2'
  }
  sku: {
    name: 'GlobalStandard'
    capacity: 50
  }
}

// Embedding Model Deployment
resource embeddingDeployment 'Microsoft.CognitiveServices/accounts/deployments@2024-04-01-preview' = {
  parent: openAI
  name: 'text-embedding-3-large'
  properties: {
    model: {
      format: 'OpenAI'
      name: 'text-embedding-3-large'
      version: '1'
    }
  }
  sku: {
    name: 'Standard'
    capacity: 120
  }
  dependsOn: [gpt4oDeployment]
}

// AI Search for RAG
resource aiSearch 'Microsoft.Search/searchServices@2024-03-01-preview' = {
  name: 'search-${environmentName}-${uniqueString(resourceGroup().id)}'
  location: location
  sku: {
    name: 'standard'
  }
  properties: {
    hostingMode: 'default'
    publicNetworkAccess: publicNetworkAccess
    semanticSearch: 'standard'
  }
}

// Storage Account for data
resource storage 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'st${environmentName}${uniqueString(resourceGroup().id)}'
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    allowBlobPublicAccess: false
    minimumTlsVersion: 'TLS1_2'
    networkAcls: {
      defaultAction: 'Deny'
    }
  }
}

// Key Vault for secrets
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'kv-${environmentName}-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true
    publicNetworkAccess: publicNetworkAccess
  }
}

// Application Insights for monitoring
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: 'ai-insights-${environmentName}'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    publicNetworkAccessForIngestion: 'Enabled'
    publicNetworkAccessForQuery: 'Enabled'
  }
}

output hubId string = aiHub.id
output projectId string = aiProject.id
output openAIEndpoint string = openAI.properties.endpoint
output searchEndpoint string = 'https://${aiSearch.name}.search.windows.net'
```

---

## Azure OpenAI Service

### Model Comparison and Selection

| Model | Best For | Context Window | Cost (per 1M tokens) |
|-------|----------|----------------|---------------------|
| **GPT-4o** | Complex reasoning, multimodal | 128K | $2.50 input / $10 output |
| **GPT-4o-mini** | General tasks, cost efficiency | 128K | $0.15 input / $0.60 output |
| **o1** | Advanced reasoning, math, code | 128K | $15 input / $60 output |
| **o1-mini** | Fast reasoning tasks | 128K | $3 input / $12 output |
| **GPT-4 Turbo** | Long context, JSON mode | 128K | $10 input / $30 output |
| **DALL-E 3** | Image generation | N/A | $0.04 - $0.12 per image |
| **text-embedding-3-large** | Semantic search, RAG | 8K | $0.13 per 1M tokens |

### Advanced API Patterns

```python
from openai import AzureOpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
import json

# Initialize with managed identity (production recommended)
credential = DefaultAzureCredential()
token_provider = get_bearer_token_provider(
    credential, "https://cognitiveservices.azure.com/.default"
)

client = AzureOpenAI(
    azure_ad_token_provider=token_provider,
    api_version="2024-08-01-preview",
    azure_endpoint="https://your-resource.openai.azure.com"
)

# Structured Output with JSON Schema
def extract_structured_data(text: str) -> dict:
    """Extract structured data using JSON schema enforcement."""
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {
                "role": "system",
                "content": "Extract customer information from the text."
            },
            {"role": "user", "content": text}
        ],
        response_format={
            "type": "json_schema",
            "json_schema": {
                "name": "customer_info",
                "strict": True,
                "schema": {
                    "type": "object",
                    "properties": {
                        "name": {"type": "string"},
                        "email": {"type": "string"},
                        "company": {"type": "string"},
                        "sentiment": {
                            "type": "string",
                            "enum": ["positive", "neutral", "negative"]
                        },
                        "topics": {
                            "type": "array",
                            "items": {"type": "string"}
                        }
                    },
                    "required": ["name", "sentiment"],
                    "additionalProperties": False
                }
            }
        }
    )
    return json.loads(response.choices[0].message.content)


# Function Calling for Tool Use
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_customer_orders",
            "description": "Retrieve recent orders for a customer",
            "parameters": {
                "type": "object",
                "properties": {
                    "customer_id": {
                        "type": "string",
                        "description": "The customer's unique identifier"
                    },
                    "limit": {
                        "type": "integer",
                        "description": "Maximum number of orders to return",
                        "default": 10
                    }
                },
                "required": ["customer_id"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "create_support_ticket",
            "description": "Create a new support ticket",
            "parameters": {
                "type": "object",
                "properties": {
                    "title": {"type": "string"},
                    "description": {"type": "string"},
                    "priority": {
                        "type": "string",
                        "enum": ["low", "medium", "high", "critical"]
                    },
                    "category": {"type": "string"}
                },
                "required": ["title", "description", "priority"]
            }
        }
    }
]

def process_with_tools(user_message: str):
    """Process user request with tool calling."""
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {
                "role": "system",
                "content": "You are a helpful customer service agent."
            },
            {"role": "user", "content": user_message}
        ],
        tools=tools,
        tool_choice="auto"
    )

    message = response.choices[0].message

    if message.tool_calls:
        for tool_call in message.tool_calls:
            function_name = tool_call.function.name
            arguments = json.loads(tool_call.function.arguments)

            # Execute the function (implement your logic)
            result = execute_function(function_name, arguments)

            # Return result to model for final response
            # ... continue conversation with tool results

    return message.content


# Assistants API with Code Interpreter
def create_data_analyst_assistant():
    """Create an assistant that can analyze data."""
    assistant = client.beta.assistants.create(
        name="Data Analyst",
        instructions="""You are a data analyst. When given data:
        1. Analyze patterns and trends
        2. Create visualizations using Python
        3. Provide actionable insights
        Always explain your findings clearly.""",
        model="gpt-4o",
        tools=[{"type": "code_interpreter"}]
    )
    return assistant
```

### DALL-E 3 Image Generation

```python
# Image Generation with DALL-E 3
def generate_image(prompt: str, quality: str = "standard") -> str:
    """Generate an image using DALL-E 3."""
    response = client.images.generate(
        model="dall-e-3",
        prompt=prompt,
        size="1024x1024",  # Options: 1024x1024, 1792x1024, 1024x1792
        quality=quality,   # Options: standard, hd
        style="vivid",     # Options: vivid, natural
        n=1
    )
    return response.data[0].url

# Image Editing (with DALL-E 2)
def edit_image(image_path: str, mask_path: str, prompt: str) -> str:
    """Edit an image using DALL-E 2."""
    with open(image_path, "rb") as image, open(mask_path, "rb") as mask:
        response = client.images.edit(
            model="dall-e-2",
            image=image,
            mask=mask,
            prompt=prompt,
            size="1024x1024",
            n=1
        )
    return response.data[0].url
```

---

## Azure Cognitive Services

### Service Categories

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AZURE AI SERVICES BREAKDOWN                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         AZURE AI VISION                              │   │
│   │                                                                      │   │
│   │   Image Analysis 4.0          Custom Vision                         │   │
│   │   ─────────────────           ─────────────                         │   │
│   │   • Object detection          • Train custom classifiers            │   │
│   │   • Image tagging             • Object detection models             │   │
│   │   • Smart cropping            • Edge deployment                     │   │
│   │   • Background removal        • AutoML training                     │   │
│   │   • Dense captions                                                  │   │
│   │                                                                      │   │
│   │   Face API                    OCR / Document Intelligence           │   │
│   │   ────────                    ──────────────────────────            │   │
│   │   • Face detection            • Read API (printed/handwritten)      │   │
│   │   • Verification              • Layout analysis                     │   │
│   │   • Identification            • Pre-built models (invoice, receipt) │   │
│   │   • Liveness detection        • Custom document models              │   │
│   │   (Requires approval)                                               │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        AZURE AI LANGUAGE                             │   │
│   │                                                                      │   │
│   │   Text Analytics              Custom Language                       │   │
│   │   ──────────────              ───────────────                       │   │
│   │   • Sentiment analysis        • Custom NER                          │   │
│   │   • Key phrase extraction     • Custom classification               │   │
│   │   • Named entity recognition  • Conversational language             │   │
│   │   • Language detection        • Orchestration workflows             │   │
│   │   • PII detection                                                   │   │
│   │                                                                      │   │
│   │   Translator                  Question Answering                    │   │
│   │   ──────────                  ────────────────────                  │   │
│   │   • 100+ languages            • Knowledge base Q&A                  │   │
│   │   • Document translation      • Chit-chat personality               │   │
│   │   • Custom translator         • Active learning                     │   │
│   │   • Real-time translation     • Multi-turn conversations            │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         AZURE AI SPEECH                              │   │
│   │                                                                      │   │
│   │   Speech to Text              Text to Speech                        │   │
│   │   ──────────────              ──────────────                        │   │
│   │   • Real-time transcription   • 400+ neural voices                  │   │
│   │   • Batch transcription       • Custom neural voice                 │   │
│   │   • Custom speech models      • SSML support                        │   │
│   │   • Pronunciation assessment  • Viseme (lip sync)                   │   │
│   │   • Speaker diarization       • Audio content creation              │   │
│   │                                                                      │   │
│   │   Speech Translation          Speaker Recognition                   │   │
│   │   ──────────────────          ───────────────────                   │   │
│   │   • Real-time translation     • Speaker verification                │   │
│   │   • 30+ languages             • Speaker identification              │   │
│   │   • Custom vocabulary         • Text-independent                    │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        AZURE AI DECISION                             │   │
│   │                                                                      │   │
│   │   Content Safety              Personalizer                          │   │
│   │   ──────────────              ────────────                          │   │
│   │   • Text moderation           • Personalized recommendations        │   │
│   │   • Image moderation          • Reinforcement learning              │   │
│   │   • Custom blocklists         • A/B testing built-in                │   │
│   │   • Jailbreak detection       • Multi-slot personalization          │   │
│   │   • Prompt shields                                                  │   │
│   │                                                                      │   │
│   │   Anomaly Detector            Immersive Reader                      │   │
│   │   ────────────────            ────────────────                      │   │
│   │   • Time series anomalies     • Reading assistance                  │   │
│   │   • Batch & streaming         • Translation                         │   │
│   │   • Root cause analysis       • Text to speech                      │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Multi-Service Deployment with CLI

```bash
# Create a multi-service Cognitive Services resource
az cognitiveservices account create \
    --name "ai-services-prod" \
    --resource-group "rg-ai-platform" \
    --kind "CognitiveServices" \
    --sku "S0" \
    --location "eastus" \
    --yes

# Create specialized services for better control
# Document Intelligence
az cognitiveservices account create \
    --name "doc-intelligence-prod" \
    --resource-group "rg-ai-platform" \
    --kind "FormRecognizer" \
    --sku "S0" \
    --location "eastus"

# Speech Service
az cognitiveservices account create \
    --name "speech-prod" \
    --resource-group "rg-ai-platform" \
    --kind "SpeechServices" \
    --sku "S0" \
    --location "eastus"

# Content Safety
az cognitiveservices account create \
    --name "content-safety-prod" \
    --resource-group "rg-ai-platform" \
    --kind "ContentSafety" \
    --sku "S0" \
    --location "eastus"

# Configure private endpoints
az network private-endpoint create \
    --name "ai-services-pe" \
    --resource-group "rg-ai-platform" \
    --vnet-name "vnet-ai-platform" \
    --subnet "snet-private-endpoints" \
    --private-connection-resource-id $(az cognitiveservices account show \
        --name "ai-services-prod" \
        --resource-group "rg-ai-platform" \
        --query id -o tsv) \
    --group-id "account" \
    --connection-name "ai-services-connection"
```

### Document Intelligence Example

```python
from azure.ai.formrecognizer import DocumentAnalysisClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
client = DocumentAnalysisClient(
    endpoint="https://your-doc-intelligence.cognitiveservices.azure.com/",
    credential=credential
)

def analyze_invoice(document_url: str) -> dict:
    """Extract structured data from an invoice."""
    poller = client.begin_analyze_document_from_url(
        "prebuilt-invoice",
        document_url
    )
    result = poller.result()

    invoice_data = {}
    for document in result.documents:
        for name, field in document.fields.items():
            if field.value:
                invoice_data[name] = {
                    "value": field.value,
                    "confidence": field.confidence
                }

    return invoice_data

# Custom Document Model Training
def train_custom_model(training_data_url: str, model_id: str):
    """Train a custom document extraction model."""
    from azure.ai.formrecognizer import DocumentModelAdministrationClient

    admin_client = DocumentModelAdministrationClient(
        endpoint="https://your-doc-intelligence.cognitiveservices.azure.com/",
        credential=credential
    )

    poller = admin_client.begin_build_document_model(
        build_mode="template",
        blob_container_url=training_data_url,
        model_id=model_id,
        description="Custom invoice model for ACME Corp"
    )

    model = poller.result()
    return model
```

---

## AWS Bedrock / SageMaker Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              AZURE AI SERVICES vs AWS AI SERVICES COMPARISON                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  CATEGORY          AZURE                        AWS                         │
│  ────────          ─────                        ───                         │
│                                                                              │
│  UNIFIED AI        Azure AI Foundry             Amazon Bedrock              │
│  PLATFORM          • Model catalog              • Foundation models         │
│                    • Prompt Flow                • Agents, Knowledge bases   │
│                    • Fine-tuning                • Model evaluation          │
│                    • Responsible AI             • Guardrails                │
│                    • Projects & Hubs            • No hub concept            │
│                                                                              │
│  GENERATIVE AI     Azure OpenAI Service         Amazon Bedrock              │
│                    • GPT-4, GPT-4o              • Claude (Anthropic)        │
│                    • DALL-E, Whisper            • Llama (Meta)              │
│                    • Exclusive OpenAI models    • Titan (Amazon)            │
│                    • PTU capacity               • Provisioned throughput    │
│                                                                              │
│  ML PLATFORM       Azure Machine Learning       Amazon SageMaker            │
│                    • Designer (low-code)        • Studio (notebooks)        │
│                    • AutoML                     • Autopilot                 │
│                    • MLOps pipelines            • ML pipelines              │
│                    • Managed endpoints          • Inference endpoints       │
│                    • Responsible AI dashboard   • Model cards               │
│                                                                              │
│  COMPUTER          Azure AI Vision              Amazon Rekognition          │
│  VISION            • Image Analysis 4.0        • Image/video analysis      │
│                    • Custom Vision              • Custom labels             │
│                    • Face API                   • Face detection            │
│                    • Document Intelligence      • Textract                  │
│                                                                              │
│  LANGUAGE          Azure AI Language            Amazon Comprehend           │
│                    • Text Analytics             • NLP analysis              │
│                    • Custom NER                 • Custom classification     │
│                    • Translator                 • Amazon Translate          │
│                    • Question Answering         • Amazon Kendra             │
│                                                                              │
│  SPEECH            Azure AI Speech              Amazon Transcribe/Polly     │
│                    • Speech-to-Text             • Transcribe                │
│                    • Text-to-Speech             • Polly                     │
│                    • Custom neural voice        • Brand Voice               │
│                    • Speech Translation         • Transcribe streaming      │
│                                                                              │
│  CONVERSATIONAL    Copilot Studio               Amazon Lex                  │
│  AI                • Low-code bots              • Intent-based bots         │
│                    • M365 integration           • Lambda integration        │
│                    • Generative AI first        • Bedrock integration       │
│                    • Power Platform             • Connect integration       │
│                                                                              │
│  CONTENT SAFETY    Azure AI Content Safety      Amazon Bedrock Guardrails   │
│                    • Text/image moderation      • Content filters           │
│                    • Prompt shields             • Topic blocking            │
│                    • Custom blocklists          • PII redaction             │
│                    • Groundedness detection     • Word filters              │
│                                                                              │
│  PRICING MODEL     • Pay-per-token (standard)   • Pay-per-token             │
│                    • PTU (provisioned)          • Provisioned throughput    │
│                    • Commitment discounts       • Savings plans             │
│                                                                              │
│  KEY ADVANTAGES:                                                            │
│  ───────────────                                                            │
│  Azure:                                                                     │
│  • Exclusive access to OpenAI models (GPT-4, DALL-E)                       │
│  • Deep M365 and Power Platform integration                                 │
│  • Comprehensive responsible AI tooling                                     │
│  • Enterprise compliance certifications                                     │
│                                                                              │
│  AWS:                                                                       │
│  • Multi-model flexibility (Claude, Llama, Titan)                          │
│  • Mature ML infrastructure (SageMaker)                                     │
│  • Broader region availability                                              │
│  • AWS service ecosystem integration                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Best Practices for AI Integration

### Architecture Patterns

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                ENTERPRISE AI INTEGRATION ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        API MANAGEMENT LAYER                          │   │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │   │
│   │   │Rate Limiting│  │  Caching    │  │  Routing    │                │   │
│   │   │ per tenant  │  │  responses  │  │  to models  │                │   │
│   │   └─────────────┘  └─────────────┘  └─────────────┘                │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│                                    ▼                                         │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                       AI GATEWAY SERVICE                             │   │
│   │                                                                      │   │
│   │   Responsibilities:                                                  │   │
│   │   • Load balancing across AI endpoints                               │   │
│   │   • Fallback between models/regions                                  │   │
│   │   • Token counting and cost allocation                               │   │
│   │   • Prompt injection detection                                       │   │
│   │   • PII detection and masking                                        │   │
│   │   • Response caching                                                 │   │
│   │   • Audit logging                                                    │   │
│   │                                                                      │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │                   CIRCUIT BREAKER                            │   │   │
│   │   │   Primary: Azure OpenAI East US                             │   │   │
│   │   │   Fallback 1: Azure OpenAI West US                          │   │   │
│   │   │   Fallback 2: Azure OpenAI Europe                           │   │   │
│   │   └─────────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│              ┌─────────────────────┼─────────────────────┐                  │
│              │                     │                     │                  │
│              ▼                     ▼                     ▼                  │
│   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐         │
│   │  Azure OpenAI   │   │  Azure OpenAI   │   │  Cognitive      │         │
│   │   (GPT-4o)      │   │  (Embeddings)   │   │   Services      │         │
│   └─────────────────┘   └─────────────────┘   └─────────────────┘         │
│                                                                              │
│   SUPPORTING SERVICES:                                                      │
│   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐         │
│   │  Redis Cache    │   │  Cosmos DB      │   │  Event Hub      │         │
│   │  (Response      │   │  (Conversation  │   │  (Telemetry     │         │
│   │   cache)        │   │   history)      │   │   streaming)    │         │
│   └─────────────────┘   └─────────────────┘   └─────────────────┘         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Security Best Practices

```yaml
# AI Security Checklist

Authentication:
  - Use managed identities (never API keys in code)
  - Implement token-based auth for client apps
  - Apply least-privilege RBAC roles
  - Rotate any API keys regularly

Network Security:
  - Deploy with private endpoints
  - Use Azure Firewall for egress control
  - Implement Web Application Firewall
  - Enable DDoS protection

Data Protection:
  - Enable customer-managed keys (CMK)
  - Implement data classification
  - Apply PII detection before AI processing
  - Store sensitive data in separate systems

Content Safety:
  - Enable content filters (Violence, Sexual, Self-harm, Hate)
  - Implement custom blocklists
  - Add prompt shields for jailbreak detection
  - Monitor for harmful content in responses

Monitoring:
  - Log all AI API calls
  - Track token usage per user/tenant
  - Set up anomaly detection alerts
  - Implement real-time abuse detection
```

### Cost Optimization Strategies

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AI COST OPTIMIZATION STRATEGIES                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. MODEL SELECTION                                                        │
│   ──────────────────                                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ Use Case               Recommended Model        Cost Savings        │   │
│   │ ────────               ─────────────────        ────────────        │   │
│   │ Simple Q&A             GPT-4o-mini              90% vs GPT-4o      │   │
│   │ Complex reasoning      GPT-4o                   Baseline           │   │
│   │ Code generation        GPT-4o-mini              90% vs GPT-4o      │   │
│   │ Embeddings             text-embedding-3-small   62% vs large       │   │
│   │ Classification         Fine-tuned mini          80% vs base        │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   2. PROMPT OPTIMIZATION                                                    │
│   ──────────────────────                                                    │
│   • Shorter system prompts (cached after first call)                        │
│   • Remove unnecessary context                                              │
│   • Use structured outputs to reduce retries                                │
│   • Set appropriate max_tokens limits                                       │
│   • Batch similar requests when possible                                    │
│                                                                              │
│   3. CACHING STRATEGIES                                                     │
│   ────────────────────                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ Cache Type          Implementation              TTL                │   │
│   │ ──────────          ──────────────              ───                │   │
│   │ Exact match         Redis hash(prompt+params)   24 hours           │   │
│   │ Semantic cache      Vector similarity search    1 hour             │   │
│   │ Embedding cache     Redis/Cosmos DB             7 days             │   │
│   │ RAG results         Azure Cache for Redis       15 minutes         │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   4. PROVISIONED THROUGHPUT (PTU)                                           │
│   ───────────────────────────────                                           │
│   Consider PTU when:                                                        │
│   • Predictable, high-volume workloads                                     │
│   • Latency-sensitive applications                                         │
│   • Estimated monthly spend > $10K                                         │
│   • Need guaranteed capacity                                                │
│                                                                              │
│   PTU Sizing Example:                                                       │
│   • 100 PTU GPT-4o ≈ 600 requests/minute                                   │
│   • Cost: ~$7,000/month (vs $15K+ on standard)                             │
│   • Break-even: ~$0.10 per 1K tokens or more                               │
│                                                                              │
│   5. MONITORING AND ALERTING                                                │
│   ──────────────────────────                                                │
│   • Set budget alerts at 50%, 75%, 90%                                     │
│   • Track cost per feature/tenant                                          │
│   • Identify and optimize high-cost queries                                │
│   • Review weekly usage reports                                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### AI Gateway Implementation

```csharp
// AI Gateway with load balancing and fallback
public class AzureOpenAIGateway
{
    private readonly List<OpenAIEndpoint> _endpoints;
    private readonly IDistributedCache _cache;
    private readonly ITelemetryClient _telemetry;

    public async Task<ChatCompletion> GetCompletionAsync(
        ChatRequest request,
        CancellationToken cancellationToken = default)
    {
        // Check cache first
        var cacheKey = ComputeCacheKey(request);
        var cached = await _cache.GetAsync<ChatCompletion>(cacheKey);
        if (cached != null)
        {
            _telemetry.TrackMetric("CacheHit", 1);
            return cached;
        }

        // Try endpoints with circuit breaker
        foreach (var endpoint in _endpoints.Where(e => e.IsHealthy))
        {
            try
            {
                var response = await endpoint.Client.GetChatCompletionsAsync(
                    request.DeploymentName,
                    request.ToChatCompletionsOptions(),
                    cancellationToken
                );

                // Track usage
                _telemetry.TrackMetric("TokensUsed",
                    response.Value.Usage.TotalTokens);
                _telemetry.TrackMetric("LatencyMs",
                    response.GetRawResponse().Headers.GetValues("x-ms-response-time")
                        .FirstOrDefault());

                // Cache response
                var completion = MapToCompletion(response.Value);
                await _cache.SetAsync(cacheKey, completion,
                    TimeSpan.FromMinutes(15));

                return completion;
            }
            catch (RequestFailedException ex) when (ex.Status == 429)
            {
                // Rate limited, try next endpoint
                endpoint.MarkUnhealthy(TimeSpan.FromMinutes(1));
                _telemetry.TrackEvent("RateLimited",
                    new Dictionary<string, string>
                    {
                        ["Endpoint"] = endpoint.Name
                    });
            }
        }

        throw new AllEndpointsExhaustedException(
            "All Azure OpenAI endpoints are unavailable");
    }

    private string ComputeCacheKey(ChatRequest request)
    {
        var content = JsonSerializer.Serialize(new
        {
            request.Messages,
            request.Temperature,
            request.MaxTokens,
            request.DeploymentName
        });

        using var sha256 = SHA256.Create();
        var hash = sha256.ComputeHash(Encoding.UTF8.GetBytes(content));
        return $"chat:{Convert.ToBase64String(hash)}";
    }
}
```

---

## Monitoring and Observability

### KQL Queries for AI Monitoring

```kusto
// Token usage by deployment
AzureMetrics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where MetricName in ("ProcessedPromptTokens", "GeneratedCompletionTokens")
| summarize
    TotalTokens = sum(Total),
    AvgTokensPerCall = avg(Total)
    by Resource, MetricName, bin(TimeGenerated, 1h)
| render timechart

// Latency percentiles
AzureDiagnostics
| where ResourceType == "ACCOUNTS"
| where Category == "RequestResponse"
| summarize
    P50 = percentile(DurationMs, 50),
    P95 = percentile(DurationMs, 95),
    P99 = percentile(DurationMs, 99),
    MaxLatency = max(DurationMs)
    by bin(TimeGenerated, 5m), OperationName
| render timechart

// Error analysis
AzureDiagnostics
| where ResourceType == "ACCOUNTS"
| where Category == "RequestResponse"
| where ResultSignature != "200"
| summarize
    ErrorCount = count(),
    UniqueErrors = dcount(ResultSignature)
    by ResultSignature, OperationName, bin(TimeGenerated, 15m)
| order by ErrorCount desc

// Content filter triggers
AzureDiagnostics
| where ResourceType == "ACCOUNTS"
| where Category == "RequestResponse"
| where properties_s contains "content_filter"
| extend FilterResult = parse_json(properties_s).content_filter_result
| summarize
    FilteredRequests = count()
    by tostring(FilterResult.hate),
       tostring(FilterResult.violence),
       bin(TimeGenerated, 1h)

// Cost estimation
AzureMetrics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where MetricName == "ProcessedPromptTokens" or MetricName == "GeneratedCompletionTokens"
| summarize TotalTokens = sum(Total) by MetricName, bin(TimeGenerated, 1d)
| extend CostEstimate = case(
    MetricName == "ProcessedPromptTokens", TotalTokens * 0.0000025,  // GPT-4o input
    MetricName == "GeneratedCompletionTokens", TotalTokens * 0.00001, // GPT-4o output
    0.0
)
| summarize DailyCost = sum(CostEstimate) by bin(TimeGenerated, 1d)
```

---

*Continue to [Copilot Studio](04-copilot-studio.md)*

*Back to [AI Agents](02-ai-agents.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
