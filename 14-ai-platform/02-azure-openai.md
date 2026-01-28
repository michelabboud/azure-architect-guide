# Azure OpenAI Service

## What is Azure OpenAI?

Azure OpenAI provides REST API access to OpenAI's powerful language models including GPT-4, GPT-4o, GPT-3.5-Turbo, DALL-E, and Whisper with the security and enterprise features of Azure.

### Key Differentiators from OpenAI API

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   AZURE OPENAI vs OPENAI API                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  OPENAI API                           AZURE OPENAI                          │
│  ──────────                           ────────────                          │
│                                                                              │
│  Data handling:                       Data handling:                        │
│  • May use data for                   • Your data is NOT used for           │
│    model improvement                    model training                      │
│  • Data stored in US                  • Data stays in your region           │
│                                       • Data processing agreements          │
│                                                                              │
│  Network:                             Network:                              │
│  • Public endpoint only               • Private endpoints                   │
│                                       • VNet integration                    │
│                                       • ExpressRoute support                │
│                                                                              │
│  Security:                            Security:                             │
│  • API key only                       • Azure AD authentication             │
│                                       • Managed identity                    │
│                                       • Customer-managed keys               │
│                                       • Azure Policy integration            │
│                                                                              │
│  Compliance:                          Compliance:                           │
│  • SOC 2                              • SOC 2, ISO 27001                   │
│  • Limited certifications             • HIPAA (with BAA)                   │
│                                       • FedRAMP                             │
│                                       • GDPR, regional compliance           │
│                                                                              │
│  Content Safety:                      Content Safety:                       │
│  • Basic moderation                   • Configurable content filters        │
│                                       • Custom blocklists                   │
│                                       • Jailbreak protection                │
│                                                                              │
│  SLA:                                 SLA:                                  │
│  • Best effort                        • 99.9% uptime SLA                   │
│                                       • Enterprise support                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Provisioning and Capacity

### Deployment Types

```
AZURE OPENAI DEPLOYMENT TYPES:
──────────────────────────────

1. STANDARD (Pay-per-token)
┌────────────────────────────────────────────────────────────────────────────┐
│ How it works:                                                               │
│ • Pay only for tokens consumed                                             │
│ • Shared capacity (rate limits apply)                                      │
│ • Token Per Minute (TPM) quota                                             │
│                                                                             │
│ Rate Limits (vary by region and model):                                    │
│ • GPT-4o: 30K-150K TPM typical                                             │
│ • GPT-4 Turbo: 10K-80K TPM typical                                         │
│ • GPT-3.5 Turbo: 120K-350K TPM typical                                     │
│                                                                             │
│ Best for: Variable workloads, development, testing                        │
└────────────────────────────────────────────────────────────────────────────┘

2. PROVISIONED THROUGHPUT (PTU)
┌────────────────────────────────────────────────────────────────────────────┐
│ How it works:                                                               │
│ • Reserved capacity (guaranteed throughput)                                │
│ • Pay hourly for Provisioned Throughput Units                             │
│ • Predictable performance, no throttling                                   │
│                                                                             │
│ PTU Capacity (approximate):                                                │
│ • 1 PTU ≈ 1-6 requests/second (model dependent)                           │
│ • GPT-4o: ~6 requests/sec per PTU                                         │
│ • GPT-4: ~1 request/sec per PTU                                           │
│                                                                             │
│ Pricing: ~$0.07/PTU-hour (varies by model)                                │
│                                                                             │
│ Best for: Production, high-volume, latency-sensitive                      │
└────────────────────────────────────────────────────────────────────────────┘

3. GLOBAL STANDARD
┌────────────────────────────────────────────────────────────────────────────┐
│ How it works:                                                               │
│ • Requests routed to region with available capacity                       │
│ • Higher availability, better rate limits                                  │
│ • Pay per token (like Standard)                                            │
│                                                                             │
│ Trade-off: Data may be processed in any region                            │
│                                                                             │
│ Best for: Non-sensitive data, maximum availability                        │
└────────────────────────────────────────────────────────────────────────────┘
```

### Quota Management

```bash
# View current quota usage
az cognitiveservices usage list \
  --location eastus \
  --output table

# Request quota increase
# Via Azure Portal: Azure OpenAI → Quotas → Request increase

# Check deployment capacity
az cognitiveservices account deployment list \
  --name my-openai \
  --resource-group myRG \
  --query "[].{Name:name, Model:properties.model.name, Capacity:sku.capacity}" \
  --output table
```

---

## API Patterns

### Chat Completion

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint="https://your-resource.openai.azure.com",
    api_key="your-api-key",
    api_version="2024-02-01"
)

# Basic chat completion
response = client.chat.completions.create(
    model="gpt-4o-deployment",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Explain cloud computing in simple terms."}
    ],
    temperature=0.7,
    max_tokens=500,
    top_p=0.95,
    frequency_penalty=0,
    presence_penalty=0
)

print(response.choices[0].message.content)
```

### Streaming

```python
# Streaming for real-time responses
response = client.chat.completions.create(
    model="gpt-4o-deployment",
    messages=[{"role": "user", "content": "Write a story about AI."}],
    stream=True
)

for chunk in response:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

### JSON Mode

```python
# Force JSON output
response = client.chat.completions.create(
    model="gpt-4o-deployment",
    messages=[
        {"role": "system", "content": "Extract information and return as JSON."},
        {"role": "user", "content": "John Smith, 35 years old, lives in Seattle, works as engineer."}
    ],
    response_format={"type": "json_object"}
)

import json
data = json.loads(response.choices[0].message.content)
# {"name": "John Smith", "age": 35, "city": "Seattle", "occupation": "engineer"}
```

### Vision (GPT-4o)

```python
# Image analysis with GPT-4o
response = client.chat.completions.create(
    model="gpt-4o-deployment",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "What's in this image?"},
                {
                    "type": "image_url",
                    "image_url": {
                        "url": "https://example.com/image.png",
                        # Or base64: "data:image/png;base64,..."
                    }
                }
            ]
        }
    ],
    max_tokens=300
)
```

### Embeddings

```python
# Generate embeddings for semantic search
response = client.embeddings.create(
    model="text-embedding-3-large-deployment",
    input=["The quick brown fox", "A fast auburn canine"]
)

# Access embeddings
embedding1 = response.data[0].embedding  # 3072 dimensions
embedding2 = response.data[1].embedding

# Calculate similarity
import numpy as np
similarity = np.dot(embedding1, embedding2) / (
    np.linalg.norm(embedding1) * np.linalg.norm(embedding2)
)
print(f"Similarity: {similarity}")  # ~0.85 (semantically similar)
```

---

## Enterprise Configuration

### Private Endpoint Setup

```bash
# Create private endpoint for Azure OpenAI
az network private-endpoint create \
  --name openai-pe \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet private-endpoints \
  --private-connection-resource-id "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.CognitiveServices/accounts/my-openai" \
  --group-id account \
  --connection-name openai-connection

# Create private DNS zone
az network private-dns zone create \
  --resource-group myRG \
  --name "privatelink.openai.azure.com"

# Link to VNet
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name "privatelink.openai.azure.com" \
  --name openai-dns-link \
  --virtual-network myVNet \
  --registration-enabled false

# Add DNS record
az network private-dns record-set a create \
  --resource-group myRG \
  --zone-name "privatelink.openai.azure.com" \
  --name my-openai

az network private-dns record-set a add-record \
  --resource-group myRG \
  --zone-name "privatelink.openai.azure.com" \
  --record-set-name my-openai \
  --ipv4-address <private-endpoint-ip>

# Disable public access
az cognitiveservices account update \
  --name my-openai \
  --resource-group myRG \
  --public-network-access Disabled
```

### Managed Identity Authentication

```python
from openai import AzureOpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

# Use managed identity (no API keys!)
credential = DefaultAzureCredential()
token_provider = get_bearer_token_provider(
    credential, "https://cognitiveservices.azure.com/.default"
)

client = AzureOpenAI(
    azure_ad_token_provider=token_provider,
    api_version="2024-02-01",
    azure_endpoint="https://your-resource.openai.azure.com"
)

# All API calls work the same way
response = client.chat.completions.create(
    model="gpt-4o-deployment",
    messages=[{"role": "user", "content": "Hello!"}]
)
```

### Customer-Managed Keys

```bash
# Enable CMK for Azure OpenAI
az cognitiveservices account update \
  --name my-openai \
  --resource-group myRG \
  --encryption-key-source Microsoft.KeyVault \
  --encryption-key-name myKey \
  --encryption-key-vault https://mykeyvault.vault.azure.net \
  --encryption-key-version <key-version>
```

---

## Monitoring and Cost Management

### Monitoring Metrics

```kusto
// Azure Monitor metrics for Azure OpenAI

// Request count by deployment
AzureMetrics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where MetricName == "ProcessedPromptTokens" or MetricName == "GeneratedCompletionTokens"
| summarize TotalTokens = sum(Total) by Resource, MetricName, bin(TimeGenerated, 1h)

// Latency percentiles
AzureDiagnostics
| where ResourceType == "ACCOUNTS"
| where Category == "RequestResponse"
| summarize
    P50 = percentile(DurationMs, 50),
    P95 = percentile(DurationMs, 95),
    P99 = percentile(DurationMs, 99)
by bin(TimeGenerated, 5m)

// Error rates
AzureDiagnostics
| where ResourceType == "ACCOUNTS"
| where Category == "RequestResponse"
| summarize
    Total = count(),
    Errors = countif(ResultSignature != "200")
by bin(TimeGenerated, 5m)
| extend ErrorRate = round(100.0 * Errors / Total, 2)
```

### Cost Optimization

```
COST OPTIMIZATION STRATEGIES:
─────────────────────────────

1. CHOOSE THE RIGHT MODEL
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • GPT-4o-mini is 100x cheaper than GPT-4 for simple tasks          │
   │ • Use GPT-3.5 Turbo for high-volume, simple use cases              │
   │ • Reserve GPT-4 for complex reasoning                               │
   └─────────────────────────────────────────────────────────────────────┘

2. OPTIMIZE PROMPTS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Shorter prompts = fewer input tokens                              │
   │ • Set max_tokens to limit output                                    │
   │ • Use system prompts efficiently (cached in some cases)            │
   └─────────────────────────────────────────────────────────────────────┘

3. CACHE RESPONSES
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Cache common queries (Redis, Cosmos DB)                          │
   │ • Hash prompt + parameters for cache key                           │
   │ • Set appropriate TTL based on content freshness needs             │
   └─────────────────────────────────────────────────────────────────────┘

4. USE BATCH FOR NON-REAL-TIME
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Batch API for bulk processing (coming soon)                      │
   │ • 50% cost reduction for non-interactive workloads                 │
   └─────────────────────────────────────────────────────────────────────┘

5. MONITOR AND ALERT
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Set budget alerts                                                 │
   │ • Track tokens per user/feature                                    │
   │ • Identify unexpected usage patterns                               │
   └─────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Copilot Studio](03-copilot-studio.md)* | *Back to [AI Foundry](01-ai-foundry.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
