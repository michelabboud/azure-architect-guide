# AI Agents

## Understanding AI Agents

AI agents are autonomous or semi-autonomous systems that can perceive their environment, make decisions, and take actions to achieve specific goals. They represent the evolution from simple chatbots to intelligent systems that can handle complex tasks.

## Agent Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AI AGENT ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         AGENT CORE                                   │   │
│   │                                                                      │   │
│   │   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐        │   │
│   │   │   Reasoning   │   │    Memory     │   │   Planning    │        │   │
│   │   │    Engine     │   │    Store      │   │    Module     │        │   │
│   │   │               │   │               │   │               │        │   │
│   │   │  • LLM-based  │   │  • Short-term │   │  • Goal       │        │   │
│   │   │  • Chain-of-  │   │  • Long-term  │   │    decompos.  │        │   │
│   │   │    thought    │   │  • Semantic   │   │  • Task       │        │   │
│   │   │  • ReAct      │   │    retrieval  │   │    scheduling │        │   │
│   │   └───────────────┘   └───────────────┘   └───────────────┘        │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│                           ┌────────┴────────┐                               │
│                           │                 │                               │
│   ┌───────────────────────┴─────┐  ┌───────┴───────────────────────────┐   │
│   │           TOOLS             │  │          KNOWLEDGE                │   │
│   │                             │  │                                   │   │
│   │  ┌─────────────┐            │  │  ┌─────────────┐                 │   │
│   │  │ API Calls   │            │  │  │ Vector DB   │                 │   │
│   │  │ • REST APIs │            │  │  │ • Embeddings│                 │   │
│   │  │ • GraphQL   │            │  │  │ • RAG       │                 │   │
│   │  └─────────────┘            │  │  └─────────────┘                 │   │
│   │                             │  │                                   │   │
│   │  ┌─────────────┐            │  │  ┌─────────────┐                 │   │
│   │  │ Code Exec   │            │  │  │ Documents   │                 │   │
│   │  │ • Python    │            │  │  │ • PDFs      │                 │   │
│   │  │ • SQL       │            │  │  │ • Web pages │                 │   │
│   │  └─────────────┘            │  │  └─────────────┘                 │   │
│   │                             │  │                                   │   │
│   │  ┌─────────────┐            │  │  ┌─────────────┐                 │   │
│   │  │ External    │            │  │  │ Structured  │                 │   │
│   │  │ • Search    │            │  │  │ • Databases │                 │   │
│   │  │ • Email     │            │  │  │ • APIs      │                 │   │
│   │  └─────────────┘            │  │  └─────────────┘                 │   │
│   │                             │  │                                   │   │
│   └─────────────────────────────┘  └───────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Agent Frameworks in Azure

### Semantic Kernel

Microsoft's open-source SDK for building AI agents.

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Agents;
using Microsoft.SemanticKernel.ChatCompletion;

// Create the kernel
var builder = Kernel.CreateBuilder();
builder.AddAzureOpenAIChatCompletion(
    deploymentName: "gpt-4",
    endpoint: Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")!,
    apiKey: Environment.GetEnvironmentVariable("AZURE_OPENAI_KEY")!
);

var kernel = builder.Build();

// Define a plugin (tool)
public class WeatherPlugin
{
    [KernelFunction, Description("Gets the weather for a location")]
    public string GetWeather(
        [Description("The city name")] string city)
    {
        // In production, call actual weather API
        return $"The weather in {city} is sunny, 72°F";
    }
}

// Add plugin to kernel
kernel.ImportPluginFromType<WeatherPlugin>();

// Create an agent
var agent = new ChatCompletionAgent
{
    Name = "WeatherAssistant",
    Instructions = """
        You are a helpful weather assistant.
        Use the weather tool to get current conditions.
        Always provide temperatures in both Fahrenheit and Celsius.
        """,
    Kernel = kernel
};

// Use the agent
var chat = new AgentGroupChat(agent);
chat.AddChatMessage(new ChatMessageContent(AuthorRole.User, "What's the weather in Seattle?"));

await foreach (var message in chat.InvokeAsync())
{
    Console.WriteLine($"{message.AuthorName}: {message.Content}");
}
```

### Multi-Agent Orchestration

```csharp
// Define multiple specialized agents
var researchAgent = new ChatCompletionAgent
{
    Name = "Researcher",
    Instructions = "You research topics and gather information. Summarize findings clearly.",
    Kernel = kernel
};

var writerAgent = new ChatCompletionAgent
{
    Name = "Writer",
    Instructions = "You write clear, engaging content based on research provided.",
    Kernel = kernel
};

var editorAgent = new ChatCompletionAgent
{
    Name = "Editor",
    Instructions = "You review and improve written content for clarity and accuracy.",
    Kernel = kernel
};

// Define selection strategy
var selectionStrategy = new SequentialSelectionStrategy();

// Define termination strategy
var terminationStrategy = new MaximumIterationsTerminationStrategy(maxIterations: 10);

// Create agent group chat
var groupChat = new AgentGroupChat(researchAgent, writerAgent, editorAgent)
{
    ExecutionSettings = new AgentGroupChatSettings
    {
        SelectionStrategy = selectionStrategy,
        TerminationStrategy = terminationStrategy
    }
};

// Run the multi-agent workflow
groupChat.AddChatMessage(new ChatMessageContent(
    AuthorRole.User,
    "Write a blog post about Azure AI services"
));

await foreach (var message in groupChat.InvokeAsync())
{
    Console.WriteLine($"[{message.AuthorName}]: {message.Content}\n");
}
```

### AutoGen Integration

```python
# Python example using AutoGen with Azure OpenAI
import autogen
from azure.identity import DefaultAzureCredential

config_list = [
    {
        "model": "gpt-4",
        "api_type": "azure",
        "api_base": os.environ["AZURE_OPENAI_ENDPOINT"],
        "api_key": os.environ["AZURE_OPENAI_KEY"],
        "api_version": "2024-02-15-preview"
    }
]

# Create assistant agent
assistant = autogen.AssistantAgent(
    name="assistant",
    llm_config={"config_list": config_list},
    system_message="""You are a helpful AI assistant that can:
    1. Answer questions about Azure services
    2. Help write code
    3. Explain technical concepts
    """
)

# Create user proxy agent
user_proxy = autogen.UserProxyAgent(
    name="user",
    human_input_mode="NEVER",
    max_consecutive_auto_reply=5,
    code_execution_config={
        "work_dir": "coding",
        "use_docker": False
    }
)

# Start conversation
user_proxy.initiate_chat(
    assistant,
    message="Create a Python function to list all Azure resource groups"
)
```

## Agent Patterns

### ReAct Pattern (Reasoning + Acting)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ReAct PATTERN                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   User Query: "What's the total revenue for Q4 2023?"                       │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ LOOP until answer found or max iterations                            │   │
│   │                                                                      │   │
│   │   ┌───────────────────────────────────────────────────────────────┐ │   │
│   │   │ THOUGHT: I need to query the sales database for Q4 2023      │ │   │
│   │   │          revenue figures.                                     │ │   │
│   │   └───────────────────────────────────────────────────────────────┘ │   │
│   │                              ↓                                       │   │
│   │   ┌───────────────────────────────────────────────────────────────┐ │   │
│   │   │ ACTION: query_database                                        │ │   │
│   │   │ INPUT: SELECT SUM(revenue) FROM sales                        │ │   │
│   │   │        WHERE quarter = 'Q4' AND year = 2023                  │ │   │
│   │   └───────────────────────────────────────────────────────────────┘ │   │
│   │                              ↓                                       │   │
│   │   ┌───────────────────────────────────────────────────────────────┐ │   │
│   │   │ OBSERVATION: Query returned $4,250,000                        │ │   │
│   │   └───────────────────────────────────────────────────────────────┘ │   │
│   │                              ↓                                       │   │
│   │   ┌───────────────────────────────────────────────────────────────┐ │   │
│   │   │ THOUGHT: I have the total revenue. I should format this      │ │   │
│   │   │          nicely for the user.                                 │ │   │
│   │   └───────────────────────────────────────────────────────────────┘ │   │
│   │                              ↓                                       │   │
│   │   ┌───────────────────────────────────────────────────────────────┐ │   │
│   │   │ ANSWER: The total revenue for Q4 2023 was $4,250,000.        │ │   │
│   │   └───────────────────────────────────────────────────────────────┘ │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Tool Use Pattern

```csharp
// Define tools with function calling
var tools = new List<ChatCompletionsFunctionToolDefinition>
{
    new ChatCompletionsFunctionToolDefinition
    {
        Name = "search_knowledge_base",
        Description = "Search the company knowledge base for relevant information",
        Parameters = BinaryData.FromObjectAsJson(new
        {
            type = "object",
            properties = new
            {
                query = new { type = "string", description = "The search query" },
                max_results = new { type = "integer", description = "Maximum results to return" }
            },
            required = new[] { "query" }
        })
    },
    new ChatCompletionsFunctionToolDefinition
    {
        Name = "create_ticket",
        Description = "Create a support ticket",
        Parameters = BinaryData.FromObjectAsJson(new
        {
            type = "object",
            properties = new
            {
                title = new { type = "string" },
                description = new { type = "string" },
                priority = new { type = "string", @enum = new[] { "low", "medium", "high" } }
            },
            required = new[] { "title", "description" }
        })
    }
};

// Execute tool when called
async Task<string> ExecuteTool(string toolName, JsonDocument arguments)
{
    return toolName switch
    {
        "search_knowledge_base" => await SearchKnowledgeBase(
            arguments.RootElement.GetProperty("query").GetString()!,
            arguments.RootElement.TryGetProperty("max_results", out var mr) ? mr.GetInt32() : 5
        ),
        "create_ticket" => await CreateTicket(
            arguments.RootElement.GetProperty("title").GetString()!,
            arguments.RootElement.GetProperty("description").GetString()!,
            arguments.RootElement.TryGetProperty("priority", out var p) ? p.GetString()! : "medium"
        ),
        _ => throw new ArgumentException($"Unknown tool: {toolName}")
    };
}
```

## Enterprise Agent Deployment

### Security Considerations

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AGENT SECURITY ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        SECURITY LAYERS                               │   │
│   │                                                                      │   │
│   │   1. INPUT VALIDATION                                               │   │
│   │      • Prompt injection detection                                   │   │
│   │      • Content safety filters                                       │   │
│   │      • Schema validation                                            │   │
│   │                                                                      │   │
│   │   2. AUTHENTICATION & AUTHORIZATION                                 │   │
│   │      • User identity (Entra ID)                                     │   │
│   │      • Agent identity (Managed Identity)                            │   │
│   │      • Tool permissions (RBAC)                                      │   │
│   │                                                                      │   │
│   │   3. TOOL RESTRICTIONS                                              │   │
│   │      • Allowlist of permitted tools                                 │   │
│   │      • Parameter validation                                         │   │
│   │      • Rate limiting                                                │   │
│   │                                                                      │   │
│   │   4. DATA ACCESS CONTROLS                                           │   │
│   │      • Row-level security                                           │   │
│   │      • Data classification                                          │   │
│   │      • PII masking                                                  │   │
│   │                                                                      │   │
│   │   5. OUTPUT FILTERING                                               │   │
│   │      • Content moderation                                           │   │
│   │      • Sensitive data detection                                     │   │
│   │      • Response validation                                          │   │
│   │                                                                      │   │
│   │   6. AUDIT & MONITORING                                             │   │
│   │      • All interactions logged                                      │   │
│   │      • Tool invocations tracked                                     │   │
│   │      • Anomaly detection                                            │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Monitoring Agents

```csharp
// Agent telemetry with Application Insights
public class MonitoredAgent
{
    private readonly TelemetryClient _telemetry;
    private readonly ChatCompletionAgent _agent;

    public async Task<string> ProcessRequestAsync(string userMessage)
    {
        using var operation = _telemetry.StartOperation<RequestTelemetry>("AgentRequest");

        var stopwatch = Stopwatch.StartNew();
        var toolCalls = 0;

        try
        {
            var response = await _agent.InvokeAsync(userMessage);

            // Track metrics
            _telemetry.TrackMetric("AgentLatencyMs", stopwatch.ElapsedMilliseconds);
            _telemetry.TrackMetric("ToolCallsPerRequest", toolCalls);
            _telemetry.TrackEvent("AgentSuccess", new Dictionary<string, string>
            {
                ["AgentName"] = _agent.Name,
                ["RequestLength"] = userMessage.Length.ToString(),
                ["ResponseLength"] = response.Length.ToString()
            });

            return response;
        }
        catch (Exception ex)
        {
            _telemetry.TrackException(ex);
            operation.Telemetry.Success = false;
            throw;
        }
    }
}
```

---

*Continue to [Azure AI Services](03-ai-services.md)*

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
