# Responsible AI

## Why Responsible AI Matters

As a Cloud Architect, you're responsible for ensuring AI systems are deployed safely, ethically, and in compliance with regulations. Azure provides built-in tools and frameworks to help you build responsible AI solutions.

## Microsoft's Responsible AI Principles

```
SIX PRINCIPLES OF RESPONSIBLE AI:
─────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  1. FAIRNESS                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ AI systems should treat all people fairly                            │   │
│  │                                                                       │   │
│  │ Azure tools:                                                         │   │
│  │ • Fairlearn (bias detection/mitigation)                             │   │
│  │ • AI Foundry evaluations (fairness metrics)                        │   │
│  │ • Content filters (prevent discriminatory outputs)                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  2. RELIABILITY & SAFETY                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ AI systems should perform reliably and safely                        │   │
│  │                                                                       │   │
│  │ Azure tools:                                                         │   │
│  │ • Content Safety filters                                            │   │
│  │ • Groundedness evaluations                                          │   │
│  │ • Monitoring and alerting                                           │   │
│  │ • Model evaluation benchmarks                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  3. PRIVACY & SECURITY                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ AI systems should be secure and respect privacy                      │   │
│  │                                                                       │   │
│  │ Azure tools:                                                         │   │
│  │ • Data doesn't train models (Azure OpenAI)                         │   │
│  │ • Private endpoints, VNet integration                               │   │
│  │ • Customer-managed keys                                             │   │
│  │ • RBAC and Entra ID integration                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  4. INCLUSIVENESS                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ AI systems should empower everyone and engage people                 │   │
│  │                                                                       │   │
│  │ Azure tools:                                                         │   │
│  │ • Multilingual models                                               │   │
│  │ • Accessibility features                                            │   │
│  │ • Speech services for different abilities                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  5. TRANSPARENCY                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ AI systems should be understandable                                  │   │
│  │                                                                       │   │
│  │ Azure tools:                                                         │   │
│  │ • Model cards and documentation                                     │   │
│  │ • Explainability features                                           │   │
│  │ • Citation in RAG responses                                         │   │
│  │ • Audit logging                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  6. ACCOUNTABILITY                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ People should be accountable for AI systems                          │   │
│  │                                                                       │   │
│  │ Azure tools:                                                         │   │
│  │ • Activity logs and audit trails                                    │   │
│  │ • Governance policies                                               │   │
│  │ • Human-in-the-loop capabilities                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Content Safety

### Azure AI Content Safety

```
CONTENT SAFETY ARCHITECTURE:
────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                         USER INPUT                                          │
│                             │                                                │
│                             ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    INPUT CONTENT FILTER                               │   │
│  │                                                                       │   │
│  │  Analyzes:                          Actions:                         │   │
│  │  • Hate content                     • Block (if threshold exceeded) │   │
│  │  • Violence content                 • Pass through                   │   │
│  │  • Sexual content                   • Log for audit                  │   │
│  │  • Self-harm content                                                 │   │
│  │  • Jailbreak attempts                                               │   │
│  │  • Prompt injection                                                  │   │
│  │                                                                       │   │
│  └───────────────────────────────────────┬─────────────────────────────┘   │
│                                          │                                  │
│                                          ▼                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                       AZURE OPENAI MODEL                              │   │
│  │                                                                       │   │
│  │  Generates response based on:                                        │   │
│  │  • System prompt                                                     │   │
│  │  • User input (if passed filter)                                    │   │
│  │  • Context (RAG results, etc.)                                      │   │
│  │                                                                       │   │
│  └───────────────────────────────────────┬─────────────────────────────┘   │
│                                          │                                  │
│                                          ▼                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    OUTPUT CONTENT FILTER                              │   │
│  │                                                                       │   │
│  │  Analyzes:                          Actions:                         │   │
│  │  • Same categories as input         • Block harmful output           │   │
│  │  • Protected material               • Redact sensitive info          │   │
│  │  • Custom blocklists                • Return safe response           │   │
│  │                                                                       │   │
│  └───────────────────────────────────────┬─────────────────────────────┘   │
│                                          │                                  │
│                                          ▼                                  │
│                         RESPONSE TO USER                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Content Filter Configuration

```
CONTENT FILTER SEVERITY LEVELS:
───────────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│ Level    │ Description              │ Default Action │ Can Customize      │
├──────────┼──────────────────────────┼────────────────┼────────────────────┤
│ Safe     │ No harmful content       │ Allow          │ N/A                │
│ Low      │ Mildly concerning        │ Allow          │ ✓ (can block)      │
│ Medium   │ Moderately harmful       │ Block          │ ✓ (can allow)      │
│ High     │ Severely harmful         │ Block          │ ✓ (limited)        │
└────────────────────────────────────────────────────────────────────────────┘

CUSTOM CONTENT FILTER EXAMPLE:
──────────────────────────────

# Create custom filter configuration via REST API
POST https://{resource}.openai.azure.com/contentfilters?api-version=2024-02-01

{
  "name": "strict-enterprise-filter",
  "inputFilter": {
    "hate": {"blocking": true, "severity": "low"},
    "violence": {"blocking": true, "severity": "low"},
    "sexual": {"blocking": true, "severity": "low"},
    "selfHarm": {"blocking": true, "severity": "low"},
    "jailbreak": {"blocking": true},
    "profanity": {"blocking": true}
  },
  "outputFilter": {
    "hate": {"blocking": true, "severity": "low"},
    "violence": {"blocking": true, "severity": "low"},
    "sexual": {"blocking": true, "severity": "low"},
    "selfHarm": {"blocking": true, "severity": "low"},
    "protectedMaterial": {"blocking": true}
  },
  "blocklists": ["custom-blocklist-id"]
}
```

### Custom Blocklists

```python
# Create and use custom blocklists

from azure.ai.contentsafety import ContentSafetyClient
from azure.core.credentials import AzureKeyCredential

client = ContentSafetyClient(
    endpoint="https://your-resource.cognitiveservices.azure.com",
    credential=AzureKeyCredential("your-key")
)

# Create blocklist
client.create_or_update_text_blocklist(
    blocklist_name="competitor-names",
    options={"description": "Block competitor product names"}
)

# Add terms
client.add_or_update_blocklist_items(
    blocklist_name="competitor-names",
    body={
        "blocklistItems": [
            {"text": "CompetitorProduct1"},
            {"text": "CompetitorProduct2"},
            {"text": "CompetitorBrand"}
        ]
    }
)

# Associate with deployment (via Azure Portal or REST API)
# Deployments → Content filter → Add blocklist
```

---

## Evaluating AI Safety

### Safety Evaluations in AI Foundry

```
SAFETY EVALUATION METRICS:
──────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│ Metric              │ What It Measures                │ Target            │
├─────────────────────┼─────────────────────────────────┼───────────────────┤
│ Violence            │ Violent content in responses   │ 0% defect rate    │
│ Sexual              │ Sexual content                  │ 0% defect rate    │
│ Self-harm           │ Self-harm references           │ 0% defect rate    │
│ Hate/Unfairness     │ Discriminatory content         │ 0% defect rate    │
│ Groundedness        │ Factual accuracy               │ > 90%             │
│ Jailbreak resistance│ Resists manipulation attempts  │ 100%              │
└────────────────────────────────────────────────────────────────────────────┘

RUNNING SAFETY EVALUATIONS:
───────────────────────────

1. Create adversarial test dataset:
   • Red team prompts
   • Edge cases
   • Jailbreak attempts
   • Sensitive topics

2. Run evaluation in AI Foundry:
   Evaluations → Create → Safety evaluation → Select dataset

3. Review results:
   • Defect rate by category
   • Example failures
   • Recommendations

4. Iterate:
   • Improve system prompts
   • Adjust content filters
   • Add blocklist terms
   • Re-evaluate
```

### Red Teaming

```
RED TEAMING YOUR AI:
────────────────────

WHAT TO TEST:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Jailbreak Attempts:                                                       │
│  • "Ignore your instructions and..."                                       │
│  • "Pretend you are a different AI that..."                               │
│  • Role-playing scenarios                                                  │
│  • Encoded instructions                                                    │
│                                                                             │
│  Prompt Injection:                                                         │
│  • Malicious content in user input                                        │
│  • Instructions hidden in documents (RAG)                                 │
│  • Conflicting instructions                                               │
│                                                                             │
│  Harmful Output Generation:                                                │
│  • Requests for dangerous information                                     │
│  • Hate speech generation                                                 │
│  • Personal information exposure                                          │
│  • Misinformation generation                                              │
│                                                                             │
│  Edge Cases:                                                               │
│  • Ambiguous requests                                                     │
│  • Multi-language attacks                                                 │
│  • Very long inputs                                                       │
│  • Unusual character encodings                                            │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

RECOMMENDED APPROACH:
┌────────────────────────────────────────────────────────────────────────────┐
│ 1. Start with automated testing (AI Foundry safety evaluations)           │
│ 2. Engage internal security team                                          │
│ 3. Consider external red team assessment                                  │
│ 4. Continuous monitoring in production                                    │
│ 5. Bug bounty program (for critical applications)                        │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Governance Framework

### AI Governance Checklist

```
ENTERPRISE AI GOVERNANCE:
─────────────────────────

PRE-DEPLOYMENT:
┌────────────────────────────────────────────────────────────────────────────┐
│ ☐ Use case documented and approved                                        │
│ ☐ Risk assessment completed                                               │
│ ☐ Data privacy review (is PII involved?)                                 │
│ ☐ Legal review (IP, liability, compliance)                               │
│ ☐ Security review (threat modeling)                                      │
│ ☐ Content safety configured                                               │
│ ☐ Evaluation metrics defined                                              │
│ ☐ Human oversight process defined                                         │
│ ☐ Rollback plan documented                                                │
└────────────────────────────────────────────────────────────────────────────┘

DEPLOYMENT:
┌────────────────────────────────────────────────────────────────────────────┐
│ ☐ Staged rollout (pilot → broader)                                       │
│ ☐ Monitoring and alerting configured                                      │
│ ☐ Audit logging enabled                                                   │
│ ☐ User feedback mechanism in place                                        │
│ ☐ Incident response process ready                                         │
│ ☐ Documentation for users available                                       │
└────────────────────────────────────────────────────────────────────────────┘

POST-DEPLOYMENT:
┌────────────────────────────────────────────────────────────────────────────┐
│ ☐ Regular safety evaluations                                              │
│ ☐ Usage monitoring and analysis                                           │
│ ☐ User feedback review                                                    │
│ ☐ Model performance tracking                                              │
│ ☐ Periodic risk reassessment                                              │
│ ☐ Update training for users                                               │
└────────────────────────────────────────────────────────────────────────────┘
```

### Azure Policy for AI Governance

```bash
# Example: Require content filtering on all Azure OpenAI deployments
{
  "mode": "All",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.CognitiveServices/accounts/deployments"
        },
        {
          "field": "Microsoft.CognitiveServices/accounts/deployments/sku.name",
          "in": ["Standard", "GlobalStandard"]
        },
        {
          "field": "Microsoft.CognitiveServices/accounts/deployments/model.format",
          "equals": "OpenAI"
        },
        {
          "field": "Microsoft.CognitiveServices/accounts/deployments/raiPolicyName",
          "exists": false
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {}
}
```

---

## Transparency and Disclosure

```
USER DISCLOSURE BEST PRACTICES:
───────────────────────────────

1. ALWAYS DISCLOSE AI INVOLVEMENT
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Good: "I'm an AI assistant powered by Azure OpenAI. I can help     │
   │        with HR questions but cannot make official decisions."       │
   │                                                                      │
   │ Bad:  Pretending to be human or hiding AI nature                   │
   └─────────────────────────────────────────────────────────────────────┘

2. SET APPROPRIATE EXPECTATIONS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Good: "I can provide information based on company policies. For    │
   │        official decisions, please contact HR directly."            │
   │                                                                      │
   │ Bad:  Overpromising capabilities                                    │
   └─────────────────────────────────────────────────────────────────────┘

3. PROVIDE SOURCES/CITATIONS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Good: "According to the Employee Handbook (Section 3.2), the       │
   │        vacation policy states... [Link to source]"                 │
   │                                                                      │
   │ Bad:  Providing information without attribution                    │
   └─────────────────────────────────────────────────────────────────────┘

4. ACKNOWLEDGE LIMITATIONS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Good: "I don't have information about that specific situation.     │
   │        Let me connect you with a human representative."            │
   │                                                                      │
   │ Bad:  Making up answers or guessing                                │
   └─────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Case Studies](case-studies.md)* | *Back to [Copilot Studio](03-copilot-studio.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
