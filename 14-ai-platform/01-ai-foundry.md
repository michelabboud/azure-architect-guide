# Azure AI Foundry

## What is Azure AI Foundry?

Azure AI Foundry (formerly Azure AI Studio) is Microsoft's unified platform for building generative AI applications. It brings together model access, development tools, evaluation capabilities, and deployment options in one place.

### AI Foundry vs AWS SageMaker

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   AI FOUNDRY vs AWS SAGEMAKER                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS SAGEMAKER                        AZURE AI FOUNDRY                      │
│  ─────────────                        ────────────────                      │
│                                                                              │
│  Focus: Traditional ML               Focus: Generative AI first            │
│         + Some GenAI (Bedrock)              + Traditional ML support       │
│                                                                              │
│  Components:                          Components:                           │
│  ┌─────────────────────┐             ┌─────────────────────────────────┐   │
│  │ SageMaker Studio    │             │ AI Foundry Portal               │   │
│  │ (Notebooks, IDE)    │             │ (Web-based, no local setup)     │   │
│  │                     │             │                                  │   │
│  │ JumpStart           │             │ Model Catalog                   │   │
│  │ (Model hub)         │             │ (GPT-4, Llama, Mistral, etc.)  │   │
│  │                     │             │                                  │   │
│  │ SageMaker Pipelines │             │ Prompt Flow                     │   │
│  │ (ML workflows)      │             │ (Visual AI app builder)        │   │
│  │                     │             │                                  │   │
│  │ Model Monitor       │             │ Evaluations                     │   │
│  │ (Inference monitor) │             │ (Quality, safety, groundedness)│   │
│  └─────────────────────┘             └─────────────────────────────────┘   │
│                                                                              │
│  Bedrock (separate):                 Integrated:                           │
│  • Foundation model                  • Azure OpenAI built-in              │
│    access via API                    • Third-party models in catalog      │
│  • Not in SageMaker                  • Fine-tuning in same portal        │
│    (separate console)                • Deployment from same place         │
│                                                                              │
│  Best for:                           Best for:                             │
│  • Custom ML training               • GenAI application development       │
│  • Complex MLOps                    • Rapid prototyping                   │
│  • Data science teams               • Enterprise AI solutions            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## AI Foundry Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AI FOUNDRY ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ORGANIZATIONAL STRUCTURE:                                                  │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         AI FOUNDRY HUB                                │   │
│  │                    (Shared resources & governance)                    │   │
│  │                                                                       │   │
│  │  Shared:                                                             │   │
│  │  • Azure OpenAI connection                                           │   │
│  │  • Storage account                                                   │   │
│  │  • Key Vault                                                         │   │
│  │  • Application Insights                                              │   │
│  │  • Container Registry                                                │   │
│  │                                                                       │   │
│  │  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐│   │
│  │  │   PROJECT 1       │  │   PROJECT 2       │  │   PROJECT 3       ││   │
│  │  │   (Team A)        │  │   (Team B)        │  │   (Experiment)    ││   │
│  │  │                   │  │                   │  │                   ││   │
│  │  │ • Prompt flows    │  │ • Prompt flows    │  │ • Prompt flows    ││   │
│  │  │ • Deployments     │  │ • Deployments     │  │ • Evaluations     ││   │
│  │  │ • Evaluations     │  │ • Fine-tuned      │  │ • Playground      ││   │
│  │  │ • Data assets     │  │   models          │  │                   ││   │
│  │  │                   │  │                   │  │                   ││   │
│  │  └───────────────────┘  └───────────────────┘  └───────────────────┘│   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  HUB vs PROJECT:                                                            │
│  • Hub = Organization-level (like AWS Organization)                        │
│  • Project = Team/application-level (like AWS Account)                     │
│  • Hub manages shared resources, Projects inherit them                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Model Catalog

### Available Models

```
AI FOUNDRY MODEL CATALOG:
─────────────────────────

MICROSOFT / AZURE OPENAI:
┌────────────────────────────────────────────────────────────────────────────┐
│ Model          │ Type              │ Deployment       │ Fine-tune         │
├────────────────┼───────────────────┼──────────────────┼───────────────────┤
│ GPT-4o         │ Chat, Multimodal  │ ✓ Real-time     │ Coming soon       │
│ GPT-4o-mini    │ Chat              │ ✓ Real-time     │ Coming soon       │
│ GPT-4 Turbo    │ Chat              │ ✓ Real-time     │ ✓ Yes             │
│ GPT-4          │ Chat              │ ✓ Real-time     │ ✓ Yes             │
│ GPT-3.5 Turbo  │ Chat              │ ✓ Real-time     │ ✓ Yes             │
│ text-embedding │ Embedding         │ ✓ Real-time     │ ✗ No              │
│ DALL-E 3       │ Image generation  │ ✓ Real-time     │ ✗ No              │
│ Whisper        │ Speech-to-text    │ ✓ Real-time     │ ✗ No              │
└────────────────────────────────────────────────────────────────────────────┘

OPEN SOURCE (Meta, Mistral, etc.):
┌────────────────────────────────────────────────────────────────────────────┐
│ Model          │ Type              │ Deployment       │ Fine-tune         │
├────────────────┼───────────────────┼──────────────────┼───────────────────┤
│ Llama 3.1      │ Chat              │ ✓ Serverless    │ ✓ Yes             │
│ Llama 3        │ Chat              │ ✓ Serverless    │ ✓ Yes             │
│ Mistral Large  │ Chat              │ ✓ Serverless    │ ✓ Yes             │
│ Mistral Small  │ Chat              │ ✓ Serverless    │ ✓ Yes             │
│ Phi-3          │ Chat (small)      │ ✓ Serverless    │ ✓ Yes             │
│ Cohere Command │ Chat              │ ✓ Serverless    │ ✓ Yes             │
│ JAIS           │ Chat (Arabic)     │ ✓ Serverless    │ ✗ No              │
└────────────────────────────────────────────────────────────────────────────┘

DEPLOYMENT OPTIONS:
• Azure OpenAI: Managed, pay-per-token
• Serverless API: Pay-per-token for open models
• Managed Compute: Deploy on your VMs (control, but manage infra)
```

### Model Benchmarks

```
MODEL SELECTION BY USE CASE:
────────────────────────────

ACCURACY/QUALITY (Higher is better):
┌────────────────────────────────────────────────────────────────────────────┐
│ Use Case                    │ Best Model               │ Alternatives     │
├─────────────────────────────┼──────────────────────────┼──────────────────┤
│ Complex reasoning           │ GPT-4o / GPT-4 Turbo    │ Claude 3 Opus    │
│ Code generation            │ GPT-4o                   │ Llama 3.1 405B   │
│ Creative writing           │ GPT-4o                   │ Claude 3         │
│ Data extraction            │ GPT-4o-mini             │ GPT-3.5 Turbo    │
│ Simple Q&A                 │ GPT-4o-mini             │ Llama 3.1 8B     │
│ High-volume, simple        │ GPT-3.5 Turbo           │ Phi-3            │
│ Multilingual               │ GPT-4o                   │ Llama 3.1        │
│ Domain-specific            │ Fine-tuned model         │ RAG with base    │
└────────────────────────────────────────────────────────────────────────────┘

LATENCY CONSIDERATIONS:
• Smaller models = faster response
• GPT-4o-mini: ~500ms first token
• GPT-4o: ~800ms first token
• Streaming reduces perceived latency
```

---

## Prompt Flow

### Building AI Applications

```
PROMPT FLOW:
────────────

Visual tool for building AI application workflows (like AWS Step Functions for AI).

FLOW TYPES:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  STANDARD FLOW:                                                            │
│  • General-purpose orchestration                                           │
│  • Connect multiple LLM calls                                             │
│  • Add tools and functions                                                │
│                                                                             │
│  CHAT FLOW:                                                                │
│  • Optimized for conversational AI                                        │
│  • Built-in chat history handling                                         │
│  • Memory management                                                       │
│                                                                             │
│  EVALUATION FLOW:                                                          │
│  • Test and score other flows                                             │
│  • Quality metrics                                                        │
│  • A/B comparison                                                         │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

FLOW NODES:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  LLM Node:                     Tool Node:                                  │
│  ┌─────────────────────┐      ┌─────────────────────┐                     │
│  │ • Azure OpenAI      │      │ • Python code       │                     │
│  │ • Custom model      │      │ • API calls         │                     │
│  │ • Prompt template   │      │ • Data processing   │                     │
│  └─────────────────────┘      └─────────────────────┘                     │
│                                                                             │
│  Prompt Node:                  Embedding Node:                             │
│  ┌─────────────────────┐      ┌─────────────────────┐                     │
│  │ • Jinja2 templates  │      │ • text-embedding-3  │                     │
│  │ • Variable injection│      │ • Custom embeddings │                     │
│  │ • Few-shot examples │      │ • Vector generation │                     │
│  └─────────────────────┘      └─────────────────────┘                     │
│                                                                             │
│  Index Lookup:                 Content Safety:                             │
│  ┌─────────────────────┐      ┌─────────────────────┐                     │
│  │ • Azure AI Search   │      │ • Content filter    │                     │
│  │ • Vector search     │      │ • Blocklist check   │                     │
│  │ • Hybrid search     │      │ • PII detection     │                     │
│  └─────────────────────┘      └─────────────────────┘                     │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Example RAG Flow

```yaml
# flow.dag.yaml - RAG Flow Definition

inputs:
  question:
    type: string
    default: "What are the company vacation policies?"

outputs:
  answer:
    type: string
    reference: ${generate_answer.output}

nodes:
  - name: embed_question
    type: embedding
    source:
      type: code
      path: embed.py
    inputs:
      input: ${inputs.question}
    connection: my-openai-connection
    model: text-embedding-3-large

  - name: search_documents
    type: python
    source:
      type: code
      path: search.py
    inputs:
      query_vector: ${embed_question.output}
      top_k: 5
    connection: my-search-connection

  - name: generate_answer
    type: llm
    source:
      type: code
      path: generate.jinja2
    inputs:
      context: ${search_documents.output}
      question: ${inputs.question}
    connection: my-openai-connection
    model: gpt-4o
```

```jinja2
{# generate.jinja2 #}
system:
You are a helpful HR assistant. Answer questions based only on the provided context.
If the answer is not in the context, say "I don't have that information."

user:
Context:
{{context}}

Question: {{question}}

Please provide a helpful answer based on the context above.
```

---

## Evaluations

### Built-in Evaluation Metrics

```
AI FOUNDRY EVALUATION METRICS:
──────────────────────────────

QUALITY METRICS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Metric          │ What It Measures                │ Range    │ Target     │
├─────────────────┼─────────────────────────────────┼──────────┼────────────┤
│ Coherence       │ Logical flow of response        │ 1-5      │ > 4        │
│ Fluency         │ Grammatical quality             │ 1-5      │ > 4        │
│ Groundedness    │ Based on provided context       │ 1-5      │ > 4        │
│ Relevance       │ Answers the actual question     │ 1-5      │ > 4        │
│ Similarity      │ Matches reference answer        │ 0-1      │ > 0.8      │
│ F1 Score        │ Token overlap with reference    │ 0-1      │ Varies     │
└────────────────────────────────────────────────────────────────────────────┘

SAFETY METRICS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Metric          │ What It Measures                │ Range    │ Target     │
├─────────────────┼─────────────────────────────────┼──────────┼────────────┤
│ Violence        │ Violent content in response     │ Defect   │ 0          │
│ Sexual          │ Sexual content in response      │ Defect   │ 0          │
│ Self-harm       │ Self-harm content               │ Defect   │ 0          │
│ Hate/Unfairness │ Discriminatory content          │ Defect   │ 0          │
│ Jailbreak       │ Attempts to bypass safety       │ Defect   │ 0          │
└────────────────────────────────────────────────────────────────────────────┘

RAG-SPECIFIC METRICS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Metric              │ What It Measures                │ Target           │
├─────────────────────┼─────────────────────────────────┼──────────────────┤
│ Retrieval Score     │ Did we find relevant docs?     │ > 0.8            │
│ Citation Accuracy   │ Are citations correct?         │ 100%             │
│ Answer Faithfulness │ Is answer based on retrieved?  │ > 0.9            │
│ Context Relevance   │ Is retrieved context useful?   │ > 0.8            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Running Evaluations

```python
# Run evaluation on a prompt flow

from azure.ai.evaluation import evaluate
from azure.ai.evaluation.metrics import (
    CoherenceMetric,
    GroundednessMetric,
    RelevanceMetric,
)

# Define evaluation data
eval_data = [
    {
        "question": "What is the vacation policy?",
        "context": "Employees get 20 days PTO per year.",
        "answer": "You receive 20 days of paid time off annually.",
        "ground_truth": "Employees receive 20 days PTO."
    },
    # ... more test cases
]

# Run evaluation
results = evaluate(
    data=eval_data,
    metrics=[
        CoherenceMetric(),
        GroundednessMetric(),
        RelevanceMetric(),
    ],
    azure_ai_project={
        "subscription_id": "...",
        "resource_group_name": "...",
        "project_name": "my-ai-project"
    }
)

# View results
print(results.metrics_summary)
# {'coherence': 4.2, 'groundedness': 4.5, 'relevance': 4.1}
```

---

## Fine-Tuning

### When to Fine-Tune

```
FINE-TUNING DECISION:
─────────────────────

DO Fine-tune when:
┌────────────────────────────────────────────────────────────────────────────┐
│ ✓ Need consistent output format/style                                       │
│ ✓ Have domain-specific terminology                                         │
│ ✓ Want to reduce prompt size (bake instructions into model)               │
│ ✓ Need higher quality for specific task                                   │
│ ✓ Have 50+ high-quality examples                                          │
│ ✓ RAG not giving good enough results                                       │
└────────────────────────────────────────────────────────────────────────────┘

DON'T Fine-tune when:
┌────────────────────────────────────────────────────────────────────────────┐
│ ✗ Just need factual knowledge (use RAG instead)                           │
│ ✗ Have fewer than 50 examples                                              │
│ ✗ Task can be solved with good prompting                                  │
│ ✗ Need model to stay current (fine-tuned = frozen knowledge)             │
│ ✗ Frequent updates to knowledge needed                                     │
└────────────────────────────────────────────────────────────────────────────┘

COMPARISON:
┌─────────────────────────────────────────────────────────────────────────────┐
│                   │ Prompting     │ RAG           │ Fine-tuning            │
├───────────────────┼───────────────┼───────────────┼────────────────────────┤
│ Setup time        │ Minutes       │ Days          │ Weeks                  │
│ Cost              │ Lowest        │ Medium        │ Highest                │
│ Flexibility       │ Highest       │ High          │ Low (retrain needed)  │
│ Knowledge update  │ Easy          │ Easy          │ Hard (retrain)        │
│ Output control    │ Medium        │ Medium        │ Highest                │
│ Token efficiency  │ Low           │ Low           │ High                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Fine-Tuning Process

```
FINE-TUNING WORKFLOW:
─────────────────────

1. PREPARE DATA
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Format: JSONL with messages                                          │
   │                                                                       │
   │ {"messages": [                                                       │
   │   {"role": "system", "content": "You are a legal assistant..."},    │
   │   {"role": "user", "content": "Summarize this contract clause..."},│
   │   {"role": "assistant", "content": "This clause states..."}        │
   │ ]}                                                                   │
   │                                                                       │
   │ Requirements:                                                        │
   │ • Minimum 50 examples (recommend 500+)                              │
   │ • High-quality, representative data                                 │
   │ • Balanced across use cases                                         │
   │ • 80/20 train/validation split                                      │
   └─────────────────────────────────────────────────────────────────────┘

2. UPLOAD & CONFIGURE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ AI Foundry → Fine-tuning → Create                                   │
   │                                                                       │
   │ Settings:                                                            │
   │ • Base model: GPT-3.5-turbo or GPT-4                                │
   │ • Training file: your_training.jsonl                                │
   │ • Validation file: your_validation.jsonl                            │
   │ • Epochs: 1-10 (start with 3)                                       │
   │ • Learning rate multiplier: 0.1-2.0 (start with 1.0)               │
   │ • Batch size: Auto or specify                                       │
   └─────────────────────────────────────────────────────────────────────┘

3. TRAIN & EVALUATE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Training takes hours to days depending on:                          │
   │ • Data size                                                          │
   │ • Number of epochs                                                   │
   │ • Base model                                                         │
   │                                                                       │
   │ Monitor:                                                             │
   │ • Training loss (should decrease)                                   │
   │ • Validation loss (should decrease, then plateau)                  │
   │ • Token accuracy                                                    │
   └─────────────────────────────────────────────────────────────────────┘

4. DEPLOY & USE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Deploy fine-tuned model like any other model                        │
   │                                                                       │
   │ Pricing:                                                             │
   │ • Training: ~$0.008/1K tokens                                       │
   │ • Hosting: ~$1.70/hour while deployed                              │
   │ • Inference: Same as base model                                     │
   └─────────────────────────────────────────────────────────────────────┘
```

---

## Deployment Options

```
AI FOUNDRY DEPLOYMENT OPTIONS:
──────────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  REAL-TIME ENDPOINTS (Azure OpenAI):                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Synchronous API calls                                              │   │
│  │ • Pay per token                                                      │   │
│  │ • Auto-scaling included                                              │   │
│  │ • Content filtering included                                         │   │
│  │ • Best for: Interactive applications                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SERVERLESS API (Models as a Service):                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • For open source models (Llama, Mistral)                           │   │
│  │ • Pay per token                                                      │   │
│  │ • No infrastructure to manage                                        │   │
│  │ • Quick to deploy                                                    │   │
│  │ • Best for: Trying different models, variable workloads             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MANAGED COMPUTE (Self-hosted):                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Deploy on Azure VMs                                                │   │
│  │ • Pay for compute (hourly)                                          │   │
│  │ • Full control over infrastructure                                  │   │
│  │ • Custom scaling rules                                              │   │
│  │ • Best for: High-volume, predictable workloads                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  BATCH ENDPOINTS:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • For large-scale, non-interactive processing                       │   │
│  │ • Process files asynchronously                                      │   │
│  │ • Cost-effective for bulk operations                                │   │
│  │ • Best for: Document processing, bulk analysis                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Azure OpenAI Service](02-azure-openai.md)* | *Back to [Quick Reference](quick-reference.md)*
