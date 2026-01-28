# Quick Reference: AI & Machine Learning

## Azure OpenAI CLI Commands

```bash
# Create Azure OpenAI resource
az cognitiveservices account create \
    --name "aoai-prod" \
    --resource-group "rg-ai" \
    --kind "OpenAI" \
    --sku "S0" \
    --location "eastus"

# Deploy GPT-4 model
az cognitiveservices account deployment create \
    --name "aoai-prod" \
    --resource-group "rg-ai" \
    --deployment-name "gpt-4" \
    --model-name "gpt-4" \
    --model-version "0613" \
    --model-format "OpenAI" \
    --sku-capacity 10 \
    --sku-name "Standard"

# Deploy embedding model
az cognitiveservices account deployment create \
    --name "aoai-prod" \
    --resource-group "rg-ai" \
    --deployment-name "text-embedding-ada-002" \
    --model-name "text-embedding-ada-002" \
    --model-version "2" \
    --model-format "OpenAI" \
    --sku-capacity 120 \
    --sku-name "Standard"

# List deployments
az cognitiveservices account deployment list \
    --name "aoai-prod" \
    --resource-group "rg-ai" \
    --output table
```

## Azure AI Search Commands

```bash
# Create search service
az search service create \
    --name "srch-prod" \
    --resource-group "rg-ai" \
    --sku "standard" \
    --location "eastus"

# Create index (using REST)
curl -X PUT "https://srch-prod.search.windows.net/indexes/documents?api-version=2023-11-01" \
    -H "Content-Type: application/json" \
    -H "api-key: ${SEARCH_ADMIN_KEY}" \
    -d @index-schema.json
```

## Python SDK Examples

### Azure OpenAI Chat Completion

```python
from openai import AzureOpenAI
import os

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_key=os.environ["AZURE_OPENAI_KEY"],
    api_version="2024-02-15-preview"
)

# Simple chat completion
response = client.chat.completions.create(
    model="gpt-4",  # deployment name
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is Azure?"}
    ],
    temperature=0.7,
    max_tokens=500
)

print(response.choices[0].message.content)
```

### Embeddings for RAG

```python
from openai import AzureOpenAI
import numpy as np

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_key=os.environ["AZURE_OPENAI_KEY"],
    api_version="2024-02-15-preview"
)

def get_embedding(text: str) -> list[float]:
    """Get embedding vector for text."""
    response = client.embeddings.create(
        model="text-embedding-ada-002",
        input=text
    )
    return response.data[0].embedding

def cosine_similarity(a: list[float], b: list[float]) -> float:
    """Calculate cosine similarity between two vectors."""
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# Example usage
doc_embedding = get_embedding("Azure is Microsoft's cloud platform")
query_embedding = get_embedding("What is Microsoft's cloud service?")
similarity = cosine_similarity(doc_embedding, query_embedding)
print(f"Similarity: {similarity:.4f}")  # ~0.93
```

### RAG with Azure AI Search

```python
from azure.search.documents import SearchClient
from azure.core.credentials import AzureKeyCredential
from openai import AzureOpenAI

# Initialize clients
search_client = SearchClient(
    endpoint=os.environ["AZURE_SEARCH_ENDPOINT"],
    index_name="documents",
    credential=AzureKeyCredential(os.environ["AZURE_SEARCH_KEY"])
)

openai_client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_key=os.environ["AZURE_OPENAI_KEY"],
    api_version="2024-02-15-preview"
)

def rag_query(user_question: str) -> str:
    """Perform RAG query."""

    # 1. Get embedding for the question
    embedding_response = openai_client.embeddings.create(
        model="text-embedding-ada-002",
        input=user_question
    )
    query_vector = embedding_response.data[0].embedding

    # 2. Search for relevant documents
    search_results = search_client.search(
        search_text=user_question,
        vector_queries=[{
            "vector": query_vector,
            "k_nearest_neighbors": 3,
            "fields": "content_vector"
        }],
        select=["title", "content"],
        top=5
    )

    # 3. Build context from search results
    context = "\n\n".join([
        f"Document: {doc['title']}\n{doc['content']}"
        for doc in search_results
    ])

    # 4. Generate answer with context
    response = openai_client.chat.completions.create(
        model="gpt-4",
        messages=[
            {
                "role": "system",
                "content": f"""Answer questions based on the provided context.
                If the answer is not in the context, say so.

                Context:
                {context}"""
            },
            {"role": "user", "content": user_question}
        ],
        temperature=0.3
    )

    return response.choices[0].message.content
```

## C# SDK Examples

### Azure OpenAI with Semantic Kernel

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using Microsoft.SemanticKernel.Connectors.OpenAI;

// Create kernel
var builder = Kernel.CreateBuilder();
builder.AddAzureOpenAIChatCompletion(
    deploymentName: "gpt-4",
    endpoint: Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")!,
    apiKey: Environment.GetEnvironmentVariable("AZURE_OPENAI_KEY")!
);

var kernel = builder.Build();

// Simple chat
var chatService = kernel.GetRequiredService<IChatCompletionService>();
var history = new ChatHistory();
history.AddSystemMessage("You are a helpful assistant.");
history.AddUserMessage("What is Azure?");

var response = await chatService.GetChatMessageContentAsync(history);
Console.WriteLine(response.Content);
```

### Function Calling

```csharp
using Microsoft.SemanticKernel;

// Define a plugin
public class DatabasePlugin
{
    [KernelFunction, Description("Query the database for sales data")]
    public async Task<string> QuerySalesAsync(
        [Description("The date range start")] string startDate,
        [Description("The date range end")] string endDate,
        [Description("Product category")] string? category = null)
    {
        // Implement actual database query
        return $"Sales from {startDate} to {endDate}: $1,234,567";
    }
}

// Use the plugin
kernel.ImportPluginFromType<DatabasePlugin>();

var settings = new OpenAIPromptExecutionSettings
{
    ToolCallBehavior = ToolCallBehavior.AutoInvokeKernelFunctions
};

var result = await kernel.InvokePromptAsync(
    "What were our sales last month?",
    new KernelArguments(settings)
);
```

## Bicep Templates

### Azure OpenAI with Private Endpoint

```bicep
param location string = resourceGroup().location
param openAIName string = 'aoai-${uniqueString(resourceGroup().id)}'

// Azure OpenAI
resource openAI 'Microsoft.CognitiveServices/accounts@2023-10-01-preview' = {
  name: openAIName
  location: location
  kind: 'OpenAI'
  sku: {
    name: 'S0'
  }
  properties: {
    customSubDomainName: openAIName
    publicNetworkAccess: 'Disabled'
    networkAcls: {
      defaultAction: 'Deny'
    }
  }
}

// Private Endpoint
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-05-01' = {
  name: 'pe-${openAIName}'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: 'openai-connection'
        properties: {
          privateLinkServiceId: openAI.id
          groupIds: ['account']
        }
      }
    ]
  }
}

// GPT-4 Deployment
resource gpt4 'Microsoft.CognitiveServices/accounts/deployments@2023-10-01-preview' = {
  parent: openAI
  name: 'gpt-4'
  properties: {
    model: {
      format: 'OpenAI'
      name: 'gpt-4'
      version: '0613'
    }
  }
  sku: {
    name: 'Standard'
    capacity: 20
  }
}

// Embedding Model
resource embedding 'Microsoft.CognitiveServices/accounts/deployments@2023-10-01-preview' = {
  parent: openAI
  name: 'text-embedding-ada-002'
  dependsOn: [gpt4]
  properties: {
    model: {
      format: 'OpenAI'
      name: 'text-embedding-ada-002'
      version: '2'
    }
  }
  sku: {
    name: 'Standard'
    capacity: 120
  }
}
```

### AI Search Index Schema

```json
{
  "name": "documents",
  "fields": [
    {"name": "id", "type": "Edm.String", "key": true, "filterable": true},
    {"name": "title", "type": "Edm.String", "searchable": true},
    {"name": "content", "type": "Edm.String", "searchable": true},
    {"name": "content_vector", "type": "Collection(Edm.Single)",
     "dimensions": 1536, "vectorSearchProfile": "vector-profile"},
    {"name": "category", "type": "Edm.String", "filterable": true, "facetable": true},
    {"name": "created_date", "type": "Edm.DateTimeOffset", "filterable": true, "sortable": true}
  ],
  "vectorSearch": {
    "algorithms": [
      {"name": "hnsw-algorithm", "kind": "hnsw",
       "hnswParameters": {"m": 4, "efConstruction": 400, "efSearch": 500, "metric": "cosine"}}
    ],
    "profiles": [
      {"name": "vector-profile", "algorithm": "hnsw-algorithm"}
    ]
  },
  "semantic": {
    "configurations": [
      {
        "name": "semantic-config",
        "prioritizedFields": {
          "titleField": {"fieldName": "title"},
          "contentFields": [{"fieldName": "content"}]
        }
      }
    ]
  }
}
```

## Prompt Templates

### System Prompt for RAG

```text
You are an AI assistant for [Company Name]. Your role is to help users find information from our knowledge base.

## Instructions
1. Answer questions based ONLY on the provided context
2. If the context doesn't contain the answer, say "I don't have that information in my knowledge base"
3. Always cite the source document when providing information
4. Be concise but thorough
5. If asked to do something outside your scope, politely redirect

## Context
{context}

## Response Format
- Start with a direct answer
- Provide supporting details
- End with source citation: [Source: document_name]
```

### Chain of Thought Prompt

```text
You are a problem-solving assistant. Work through problems step by step.

User Question: {question}

Think through this step by step:

1. First, identify what information is needed
2. Then, break down the problem into smaller parts
3. Work through each part logically
4. Finally, synthesize the final answer

Show your reasoning at each step before providing the final answer.
```

## Cost Optimization Tips

| Strategy | Implementation | Savings |
|----------|---------------|---------|
| Use GPT-3.5 for simple tasks | Route based on complexity | 10-20x cheaper |
| Implement caching | Redis for common queries | 50-70% reduction |
| Optimize prompts | Reduce token count | 20-40% reduction |
| Use PTU for predictable loads | Provisioned throughput | 30-50% vs pay-as-you-go |
| Batch embeddings | Process in batches | Reduced overhead |

---

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
