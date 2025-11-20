# Day 5 Summary: Agent2Agent Communication & Production Deployment

This document summarizes the key concepts taught in the Day 5 notebooks of the Kaggle 5-day AI Agents course.

---

## Day 5a: Agent2Agent (A2A) Communication

### Core Concept: Multi-Agent System Integration

**The Problem:**

As you build more complex AI systems:
- **A single agent can't do everything** - Specialized agents for different domains work better
- **You need agents to collaborate** - Customer support needs product data, order systems need inventory info
- **Different teams build different agents** - You want to integrate agents from external vendors
- **Agents may use different languages/frameworks** - You need a standard communication protocol

**The Solution: A2A Protocol**

The [Agent2Agent (A2A) Protocol](https://a2a-protocol.org/) is a **standard** that allows agents to:
- ✨ **Communicate over networks** - Agents can be on different machines
- ✨ **Use each other's capabilities** - One agent can call another agent like a tool
- ✨ **Work across frameworks** - Language/framework agnostic
- ✨ **Maintain formal contracts** - Agent cards describe capabilities

### Three Common A2A Architecture Patterns

The A2A protocol is particularly useful in three scenarios:

#### 1. **Cross-Framework Integration**
- ADK agent communicating with agents built in other frameworks
- Example: ADK agent calling LangChain agent

#### 2. **Cross-Language Communication**
- Python agent calling Java or Node.js agent
- Example: Python support agent using Java inventory service

#### 3. **Cross-Organization Boundaries**
- Your internal agent integrating with external vendor services
- Example: Your company's support agent using vendor's product catalog

### Architecture: E-Commerce Integration Example

```text
┌──────────────────────┐           ┌──────────────────────┐
│ Customer Support     │  ─A2A──▶  │ Product Catalog      │
│ Agent (Consumer)     │           │ Agent (Vendor)       │
│ Your Company         │           │ External Service     │
│ (localhost:8000)     │           │ (localhost:8001)     │
└──────────────────────┘           └──────────────────────┘
```

**Why this justifies A2A:**
- Product Catalog is maintained by an external vendor (you can't modify their code)
- Different organizations with separate systems
- Formal contract needed between services
- Product Catalog could be in a different language/framework

---

## A2A vs Local Sub-Agents: Decision Table

| Factor | Use A2A | Use Local Sub-Agents |
|--------|---------|---------------------|
| **Agent Location** | External service, different codebase | Same codebase, internal |
| **Ownership** | Different team/organization | Your team |
| **Network** | Agents on different machines | Same process/machine |
| **Performance** | Network latency acceptable | Need low latency |
| **Language/Framework** | Cross-language/framework needed | Same language |
| **Contract** | Formal API contract required | Internal interface |
| **Example** | External vendor product catalog | Internal order processing steps |

**Key Decision Criteria:**

**Choose A2A when:**
- Working with external vendors or third-party services
- Agents are maintained by different teams/organizations
- Need cross-language or cross-framework communication
- Formal API contracts are required
- Agents are deployed on different infrastructure

**Choose Local Sub-Agents when:**
- All agents are in your codebase
- Same team maintains everything
- Low latency is critical
- No need for network communication
- Internal workflow orchestration

---

## Key A2A Components

### 1. Agent Cards

An **agent card** is a JSON document that serves as a "business card" for your agent.

**What an Agent Card Contains:**
- **Name**: Agent identifier
- **Description**: What the agent does
- **Version**: Agent version number
- **URL**: Where the agent is hosted
- **Skills**: Capabilities the agent provides (tools/functions)
- **Protocol Version**: A2A protocol version
- **Input/Output Modes**: Supported data formats (e.g., `text/plain`)

**Standard Location:**
```
/.well-known/agent-card.json
```

**Example Agent Card Structure:**
```json
{
  "name": "product_catalog_agent",
  "description": "External vendor's product catalog agent",
  "url": "http://localhost:8001",
  "version": "0.0.1",
  "protocolVersion": "0.3.0",
  "defaultInputModes": ["text/plain"],
  "defaultOutputModes": ["text/plain"],
  "skills": [
    {
      "id": "product_catalog_agent",
      "name": "model",
      "description": "Product catalog specialist",
      "tags": ["llm"]
    },
    {
      "id": "get_product_info",
      "name": "get_product_info",
      "description": "Get product information",
      "tags": ["llm", "tools"]
    }
  ]
}
```

**Think of it as:** The "contract" that tells other agents how to work with your agent.

### 2. `to_a2a()` - Exposing Agents

**Purpose:** Make your ADK agent accessible via the A2A protocol.

**What `to_a2a()` does:**
- 🔧 Wraps your agent in an A2A-compatible server (FastAPI/Starlette)
- 📋 Auto-generates an **agent card** 
- 🌐 Serves the agent card at `/.well-known/agent-card.json`
- ✨ Handles all A2A protocol details (request/response formatting, task endpoints)

**Example:**
```python
from google.adk.a2a.utils.agent_to_a2a import to_a2a

# Convert agent to A2A-compatible application
product_catalog_a2a_app = to_a2a(
    product_catalog_agent, 
    port=8001
)

# Agent card will be available at:
# http://localhost:8001/.well-known/agent-card.json
```

**Running the A2A Server:**
```python
# Save agent code to file for uvicorn
with open("/tmp/product_catalog_server.py", "w") as f:
    f.write(agent_code)

# Start uvicorn server in background
server_process = subprocess.Popen([
    "uvicorn",
    "product_catalog_server:app",
    "--host", "localhost",
    "--port", "8001"
], cwd="/tmp")
```

### 3. `RemoteA2aAgent` - Consuming Remote Agents

**Purpose:** Connect to and use remote agents as if they were local sub-agents.

**How it works:**
- Creates a **client-side proxy** for the remote agent
- Reads the remote agent's card to understand capabilities
- Translates sub-agent calls into A2A protocol requests (HTTP POST to `/tasks`)
- Handles all protocol details transparently

**Example:**
```python
from google.adk.agents.remote_a2a_agent import RemoteA2aAgent, AGENT_CARD_WELL_KNOWN_PATH

# Create a proxy to the remote agent
remote_product_catalog_agent = RemoteA2aAgent(
    name="product_catalog_agent",
    description="Remote product catalog agent from external vendor",
    agent_card=f"http://localhost:8001{AGENT_CARD_WELL_KNOWN_PATH}"
)

# Use it like a local sub-agent!
customer_support_agent = LlmAgent(
    name="customer_support_agent",
    instruction="Use product_catalog_agent to look up product information",
    sub_agents=[remote_product_catalog_agent]  # Just add to sub_agents!
)
```

**Key Insight:** The Customer Support Agent doesn't need to know it's talking to a remote agent - ADK handles all the network communication!

---

## A2A Communication Flow

### Complete Request-Response Cycle

```
1. Customer asks Support Agent: "What's the price of iPhone 15 Pro?"
   ↓
2. Support Agent (LlmAgent) receives question
   ↓
3. Support Agent decides it needs product information
   ↓
4. Support Agent calls product_catalog_agent sub-agent
   ↓
5. RemoteA2aAgent translates call to A2A protocol request
   ↓
6. HTTP POST to http://localhost:8001/tasks
   ↓
7. Product Catalog Agent (server) receives A2A request
   ↓
8. Product Catalog Agent calls get_product_info("iPhone 15 Pro")
   ↓
9. Product Catalog Agent returns data via A2A response
   ↓
10. RemoteA2aAgent receives response
    ↓
11. Support Agent formulates final answer
    ↓
12. Customer receives: "iPhone 15 Pro costs $999 and is in stock"
```

### Behind the Scenes: A2A Protocol

**Request Format (HTTP POST to `/tasks`):**
```json
{
  "task": {
    "input": "Find price for iPhone 15 Pro",
    "parameters": {}
  }
}
```

**Response Format:**
```json
{
  "status": "success",
  "result": {
    "output": "iPhone 15 Pro, $999, Low Stock (8 units)"
  }
}
```

---

## Complete A2A Tutorial: Building a Product Catalog Integration

### Tutorial Overview

**Goal:** Build a system where:
1. **Product Catalog Agent** (vendor's service) - Provides product information
2. **Customer Support Agent** (your service) - Helps customers using product data

**6 Key Steps:**
1. Create the Product Catalog Agent
2. Expose via A2A using `to_a2a()`
3. Start the server
4. Create the Customer Support Agent with `RemoteA2aAgent`
5. Test A2A communication
6. Understand the flow

### Step 1: Create the Product Catalog Agent

```python
from google.adk.agents import LlmAgent
from google.adk.models.google_llm import Gemini

def get_product_info(product_name: str) -> str:
    """Get product information from catalog."""
    product_catalog = {
        "iphone 15 pro": "iPhone 15 Pro, $999, Low Stock (8 units)",
        "samsung galaxy s24": "Samsung Galaxy S24, $799, In Stock (31 units)",
        "macbook pro 14": "MacBook Pro 14\", $1,999, In Stock (22 units)"
    }
    
    product_lower = product_name.lower().strip()
    if product_lower in product_catalog:
        return f"Product: {product_catalog[product_lower]}"
    else:
        return f"Sorry, product not found"

product_catalog_agent = LlmAgent(
    model=Gemini(model="gemini-2.5-flash-lite"),
    name="product_catalog_agent",
    description="External vendor's product catalog agent",
    instruction="""
    You are a product catalog specialist from an external vendor.
    Use the get_product_info tool to fetch product data.
    Provide clear, accurate information about price, availability, and specs.
    """,
    tools=[get_product_info]
)
```

### Step 2: Expose via A2A

```python
from google.adk.a2a.utils.agent_to_a2a import to_a2a

# Convert to A2A-compatible application
product_catalog_a2a_app = to_a2a(
    product_catalog_agent, 
    port=8001
)

# This creates:
# - FastAPI/Starlette server
# - Agent card at /.well-known/agent-card.json
# - A2A protocol endpoints
```

### Step 3: Start the Server

```python
import subprocess

# Start uvicorn in background
server_process = subprocess.Popen([
    "uvicorn",
    "product_catalog_server:app",
    "--host", "localhost",
    "--port", "8001"
])

# Verify server is running
response = requests.get("http://localhost:8001/.well-known/agent-card.json")
print(f"Server ready! Agent card: {response.json()}")
```

### Step 4: Create Customer Support Agent (Consumer)

```python
from google.adk.agents.remote_a2a_agent import RemoteA2aAgent

# Create proxy to remote agent
remote_product_catalog_agent = RemoteA2aAgent(
    name="product_catalog_agent",
    description="Remote product catalog from vendor",
    agent_card="http://localhost:8001/.well-known/agent-card.json"
)

# Create consumer agent that uses the remote agent
customer_support_agent = LlmAgent(
    model=Gemini(model="gemini-2.5-flash-lite"),
    name="customer_support_agent",
    description="Customer support assistant",
    instruction="""
    You are a customer support agent.
    When customers ask about products, use product_catalog_agent to look up information.
    Always get product info from the sub-agent before answering.
    """,
    sub_agents=[remote_product_catalog_agent]  # Use remote agent!
)
```

### Step 5: Test A2A Communication

```python
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types

async def test_a2a():
    session_service = InMemorySessionService()
    session = await session_service.create_session(
        app_name="support_app",
        user_id="demo_user",
        session_id="demo_session"
    )
    
    runner = Runner(
        agent=customer_support_agent,
        app_name="support_app",
        session_service=session_service
    )
    
    message = types.Content(
        parts=[types.Part(text="Tell me about the iPhone 15 Pro")]
    )
    
    async for event in runner.run_async(
        user_id="demo_user",
        session_id="demo_session",
        new_message=message
    ):
        if event.is_final_response():
            print(event.content.parts[0].text)

# Run the test
await test_a2a()
```

**Expected Output:**
```
The iPhone 15 Pro costs $999 and is currently in low stock with 8 units available.
```

### Step 6: Understanding the Flow

**What happened behind the scenes:**
1. User sent message to Customer Support Agent
2. Support Agent determined it needs product data
3. Support Agent called `product_catalog_agent` sub-agent
4. **RemoteA2aAgent** sent HTTP POST to `http://localhost:8001/tasks`
5. Product Catalog Agent received A2A request
6. Product Catalog Agent called `get_product_info("iPhone 15 Pro")`
7. Result returned via A2A protocol
8. Support Agent received product data
9. Support Agent formulated natural language response
10. User received final answer

---

## Key Benefits of A2A

### 1. **Transparency**
- Consumer agent doesn't "know" the provider agent is remote
- Works just like a local sub-agent
- No special handling needed

### 2. **Standard Protocol**
- Uses industry-standard A2A protocol
- Any A2A-compatible agent can communicate
- Framework and language agnostic

### 3. **Easy Integration**
- Single line: `sub_agents=[remote_agent]`
- No complex networking code
- ADK handles all protocol details

### 4. **Separation of Concerns**
- Vendor maintains their agent (product catalog)
- Your team maintains your agent (support)
- Clean boundaries and responsibilities

### 5. **Microservices Architecture**
- Each agent is an independent service
- Can be deployed separately
- Scales independently

---

## Production A2A Patterns

### Pattern 1: Cross-Organization Integration

**Scenario:** Your company integrates with vendor services

```python
# Vendor hosts product catalog at their domain
remote_vendor_agent = RemoteA2aAgent(
    name="vendor_product_catalog",
    agent_card="https://vendor.com/.well-known/agent-card.json"
)

# Your internal agent uses it
internal_support_agent = LlmAgent(
    name="support_agent",
    sub_agents=[remote_vendor_agent]
)
```

### Pattern 2: Multi-Agent Collaboration

**Scenario:** Support agent coordinates multiple services

```python
# Connect to multiple remote agents
remote_product_catalog = RemoteA2aAgent(
    agent_card="https://vendor1.com/.well-known/agent-card.json"
)

remote_inventory = RemoteA2aAgent(
    agent_card="https://vendor2.com/.well-known/agent-card.json"
)

remote_shipping = RemoteA2aAgent(
    agent_card="https://vendor3.com/.well-known/agent-card.json"
)

# Orchestrate all three
orchestrator_agent = LlmAgent(
    name="order_orchestrator",
    sub_agents=[
        remote_product_catalog,
        remote_inventory,
        remote_shipping
    ]
)
```

### Pattern 3: Cross-Language Services

**Scenario:** Python agent calling Java service

```python
# Java agent exposed via A2A (vendor maintains)
# Running on https://java-service.company.com

# Python agent consumes it (your team maintains)
remote_java_agent = RemoteA2aAgent(
    name="java_inventory_service",
    agent_card="https://java-service.company.com/.well-known/agent-card.json"
)

python_consumer = LlmAgent(
    name="python_app",
    sub_agents=[remote_java_agent]
)
```

---

## Day 5b: Agent Deployment to Production

### Core Concept: From Development to Production

**The Problem:**

You've built an amazing AI agent:
- ✅ Works perfectly on your machine
- ✅ Responds intelligently
- ✅ Everything seems ready

**But there's a critical issue:**

> **Your agent is not publicly available!**

- Only lives in your notebook/development environment
- When you stop the notebook, it stops working
- Teammates can't access it
- Users can't interact with it

**The Solution: Production Deployment**

Deploy your agent to a production-ready hosting platform where it can:
- Run 24/7 without your laptop
- Handle multiple concurrent users
- Scale automatically based on demand
- Be accessed by anyone with proper permissions

---

## Deployment Platforms for ADK Agents

### 1. **Vertex AI Agent Engine** (Focus of Day 5b)

**Best For:** Fully managed, serverless agent hosting

**Key Features:**
- ✨ **Fully managed** service specifically designed for AI agents
- ✨ **Auto-scaling** with built-in session management
- ✨ **Easy deployment** using ADK CLI (`adk deploy agent_engine`)
- ✨ **Monthly free tier** available

**Pricing:**
- Free tier: Deploy up to 10 agents
- Pay-as-you-go after free tier
- [Pricing details](https://docs.cloud.google.com/agent-builder/agent-engine/overview#pricing)

### 2. **Cloud Run** (Alternative)

**Best For:** Serverless deployment for small-to-medium workloads

**Key Features:**
- Easiest serverless option
- Perfect for demos and prototypes
- Pay per request (no traffic = no cost)

### 3. **Google Kubernetes Engine (GKE)** (Alternative)

**Best For:** Complex, multi-agent systems with full control

**Key Features:**
- Full control over containerized deployments
- Best for enterprise-scale systems
- Requires Kubernetes expertise

---

## Deploying to Vertex AI Agent Engine

### Prerequisites

**Before deployment, you need:**

1. **Google Cloud Platform (GCP) account**
   - Sign up: https://cloud.google.com/free
   - New users get **$300 in free credits** (90 days)

2. **Billing enabled** (even for free tier)
   - Credit card required for verification
   - Won't be charged unless you upgrade

3. **APIs enabled:**
   - Vertex AI API
   - Cloud Storage API
   - Cloud Logging API
   - Cloud Monitoring API
   - Cloud Trace API
   - Telemetry API

### Deployment Architecture

**Directory Structure:**
```
sample_agent/
├── agent.py                      # Agent code (logic)
├── requirements.txt              # Python dependencies
├── .env                          # Environment configuration
└── .agent_engine_config.json    # Deployment settings
```

### File 1: `agent.py` - The Agent Code

**Purpose:** Defines your agent's behavior, tools, and instructions

```python
from google.adk.agents import Agent
import vertexai
import os

# Initialize Vertex AI
vertexai.init(
    project=os.environ["GOOGLE_CLOUD_PROJECT"],
    location=os.environ["GOOGLE_CLOUD_LOCATION"]
)

def get_weather(city: str) -> dict:
    """
    Returns weather information for a given city.
    
    Args:
        city: Name of the city (e.g., "Tokyo", "New York")
    
    Returns:
        dict: Weather report or error message
    """
    weather_data = {
        "san francisco": {
            "status": "success", 
            "report": "Sunny, 72°F (22°C)"
        },
        "tokyo": {
            "status": "success", 
            "report": "Clear, 70°F (21°C)"
        }
    }
    
    city_lower = city.lower()
    if city_lower in weather_data:
        return weather_data[city_lower]
    else:
        return {
            "status": "error",
            "error_message": f"Weather data not available for {city}"
        }

# Define the agent
root_agent = Agent(
    name="weather_assistant",
    model="gemini-2.5-flash-lite",
    description="A helpful weather assistant",
    instruction="""
    You are a friendly weather assistant.
    When users ask about weather:
    1. Identify the city name
    2. Use get_weather tool to fetch information
    3. Respond in a friendly, conversational tone
    """,
    tools=[get_weather]
)
```

### File 2: `requirements.txt` - Python Dependencies

**Purpose:** Specifies Python packages needed by your agent

```txt
google-adk
opentelemetry-instrumentation-google-genai
```

**What these packages do:**
- `google-adk` - Core ADK framework
- `opentelemetry-instrumentation-google-genai` - Telemetry for observability

### File 3: `.env` - Environment Configuration

**Purpose:** Sets environment variables for cloud configuration

```bash
# Use global endpoint for Gemini API
GOOGLE_CLOUD_LOCATION="global"

# Use Vertex AI backend (not Google AI Studio)
GOOGLE_GENAI_USE_VERTEXAI=1
```

**Configuration explained:**
- `GOOGLE_CLOUD_LOCATION="global"` - Uses global endpoint for Gemini
- `GOOGLE_GENAI_USE_VERTEXAI=1` - Configures ADK to use Vertex AI

### File 4: `.agent_engine_config.json` - Deployment Settings

**Purpose:** Controls infrastructure and scaling settings

```json
{
    "min_instances": 0,
    "max_instances": 1,
    "resource_limits": {
        "cpu": "1",
        "memory": "1Gi"
    }
}
```

**Configuration explained:**
- `"min_instances": 0` - Scales to zero when not in use (saves costs)
- `"max_instances": 1` - Maximum of 1 instance (sufficient for demo)
- `"cpu": "1"` - 1 CPU core per instance
- `"memory": "1Gi"` - 1 GB RAM per instance

**Cost Optimization:**
- `min_instances: 0` means zero cost when idle
- Only pay when agent is actively processing requests
- Perfect for development and testing

---

## Deployment Commands & Workflow

### Step 1: Authenticate with Google Cloud

```bash
# Local development - authenticate using CLI
gcloud auth application-default login
```

### Step 2: Set Project ID

```python
import os

PROJECT_ID = "your-project-id"  # Replace with your actual project ID
os.environ["GOOGLE_CLOUD_PROJECT"] = PROJECT_ID
```

### Step 3: Deploy the Agent

```bash
# Deploy to Agent Engine
adk deploy agent_engine \
    --project=$PROJECT_ID \
    --region=us-west1 \
    sample_agent \
    --agent_engine_config_file=sample_agent/.agent_engine_config.json
```

**What happens during deployment:**
1. ✅ ADK packages your agent code
2. ✅ Uploads to Google Cloud
3. ✅ Creates containerized deployment
4. ✅ Returns resource name: `projects/PROJECT_NUMBER/locations/REGION/reasoningEngines/ID`

**Deployment time:** Typically 2-5 minutes

**Output example:**
```
✅ Created agent engine: projects/549186278804/locations/europe-west1/reasoningEngines/9135499067562917888
```

### Step 4: Retrieve the Deployed Agent

```python
import vertexai
from vertexai import agent_engines

# Initialize Vertex AI
vertexai.init(project=PROJECT_ID, location="us-west1")

# Get the most recently deployed agent
agents_list = list(agent_engines.list())
remote_agent = agents_list[0]

print(f"✅ Connected to: {remote_agent.resource_name}")
```

### Step 5: Test the Deployed Agent

```python
# Query the deployed agent
async for item in remote_agent.async_stream_query(
    message="What is the weather in Tokyo?",
    user_id="user_42"
):
    print(item)
```

**Expected response flow:**
1. **Function call** - Agent decides to call `get_weather` tool
2. **Function response** - Result from the tool
3. **Final response** - Natural language answer

**Example output:**
```
The weather in Tokyo is clear with a temperature of 70°F (21°C).
```

---

## Agent Engine Regions

### Available Regions

Agent Engine is available in specific regions:

```python
regions_list = [
    "europe-west1",    # Belgium
    "europe-west4",    # Netherlands
    "us-east4",        # Virginia
    "us-west1"         # Oregon
]
```

### Choosing a Region

**Production considerations:**
- ✅ Choose a region **close to your users** (lower latency)
- ✅ Consider **data residency requirements** (regulations)
- ✅ Check [Agent Engine locations](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview#locations)

**For this demo:**
- Region selection is random (for learning purposes)
- In production, use strategic region placement

---

## Long-Term Memory with Vertex AI Memory Bank

### The Memory Problem

**Session Memory (Built-in):**
- ✅ Remembers conversation **within** a session
- ❌ Forgets when session ends
- ❌ Each new conversation starts from scratch

**Example Problem:**
```
Session 1:
User: "I prefer Celsius"
Agent: "Got it!"

Session 2 (next day):
User: "What's the weather in Tokyo?"
Agent: "70°F (21°C)"  ❌ Forgot user prefers Celsius
```

### Solution: Vertex AI Memory Bank

**Long-term memory across all sessions:**

| Session Memory | Memory Bank |
|----------------|-------------|
| Single conversation | All conversations |
| Forgets when session ends | Remembers permanently |
| "What did I just say?" | "What's my favorite city?" |
| Built-in | Requires configuration |

### How Memory Bank Works

**During conversations:**
1. Agent uses memory tools to search past facts
2. Retrieves relevant information from previous sessions

**After conversations:**
3. Agent Engine extracts key information
4. Stores facts like "User prefers Celsius"

**Next session:**
5. Agent automatically recalls stored facts
6. Provides personalized responses

### Enabling Memory Bank

**Memory Bank requires:**
1. Add memory tools to your agent (`PreloadMemoryTool`, `LoadMemoryTool`)
2. Add a callback to save conversations
3. Redeploy your agent

**Example with Memory Bank:**
```python
from google.adk.tools.memory_tools import PreloadMemoryTool

# Add memory tools to your agent
weather_agent_with_memory = Agent(
    name="weather_assistant",
    model="gemini-2.5-flash-lite",
    instruction="""
    You are a weather assistant with memory.
    Use PreloadMemoryTool to recall user preferences.
    Remember: temperature units, favorite cities, etc.
    """,
    tools=[get_weather, PreloadMemoryTool()]
)
```

### Memory Bank Resources

- [ADK Memory Guide](https://google.github.io/adk-docs/sessions/memory/)
- [Memory Tools Documentation](https://google.github.io/adk-docs/tools/built-in-tools/)
- [Get Started with Memory Bank](https://github.com/GoogleCloudPlatform/generative-ai/blob/main/agents/agent_engine/memory_bank/get_started_with_memory_bank_on_adk.ipynb)

---

## Cost Management & Cleanup

### Understanding Costs

**Agent Engine Pricing:**
- ✅ **Free tier:** Deploy up to 10 agents
- ✅ **Pay-as-you-go** after free tier
- ⚠️ Leaving agents running can incur costs

**Cost factors:**
- Running time (hours deployed)
- Number of requests
- Resource usage (CPU, memory)
- Model API calls (Gemini usage)

### Best Practices for Cost Management

**1. Use `min_instances: 0` for development:**
```json
{
    "min_instances": 0,  // Scales to zero = no cost when idle
    "max_instances": 1
}
```

**2. Delete test deployments when finished:**
```python
# Always clean up after testing!
agent_engines.delete(
    resource_name=remote_agent.resource_name,
    force=True
)
```

**3. Monitor usage in Google Cloud Console:**
- Visit [Agent Engine Console](https://console.cloud.google.com/vertex-ai/agents/agent-engines)
- Check active deployments
- Review resource usage

**4. Set budget alerts:**
- Configure budget alerts in Google Cloud
- Get notified before spending limits
- Prevent unexpected charges

### Cleanup Commands

**Delete a deployed agent:**
```python
from vertexai import agent_engines

# Delete the agent
agent_engines.delete(
    resource_name=remote_agent.resource_name,
    force=True  # Force deletion even if running
)

print("✅ Agent successfully deleted")
```

**Verify deletion:**
- Check [Agent Engine Console](https://console.cloud.google.com/vertex-ai/agents/agent-engines)
- Deletion takes 1-2 minutes
- Confirm agent no longer appears in list

---

## Production Best Practices

### 1. **Enable Observability**

**Add tracing and logging:**
```python
from google.adk.plugins.logging_plugin import LoggingPlugin

runner = Runner(
    agent=weather_agent,
    plugins=[LoggingPlugin()]
)
```

**Monitor in Google Cloud:**
- Cloud Logging for logs
- Cloud Trace for performance
- Cloud Monitoring for metrics

### 2. **Error Handling**

**Add retry configuration:**
```python
from google.genai import types

retry_config = types.HttpRetryOptions(
    attempts=5,
    exp_base=7,
    initial_delay=1,
    http_status_codes=[429, 500, 503, 504]
)

agent = Agent(
    model=Gemini(
        model="gemini-2.5-flash-lite",
        retry_options=retry_config
    )
)
```

### 3. **Security**

**Follow security best practices:**
- ✅ Use environment variables for secrets
- ✅ Never commit credentials to git
- ✅ Use Cloud Secret Manager for sensitive data
- ✅ Implement proper authentication
- ✅ Follow [ADK security guidelines](https://google.github.io/adk-docs/safety/)

### 4. **Resource Limits**

**Set appropriate limits:**
```json
{
    "min_instances": 0,
    "max_instances": 10,  // Allow scaling for production
    "resource_limits": {
        "cpu": "2",       // More CPU for higher throughput
        "memory": "4Gi"   // More memory for complex agents
    }
}
```

### 5. **Testing Strategy**

**Test before deploying to production:**
```
1. Local testing (development)
   ↓
2. Deploy to dev environment
   ↓
3. Run evaluation suite
   ↓
4. Deploy to production
   ↓
5. Monitor and observe
```

### 6. **Version Control**

**Tag deployments:**
- Use git tags for versions
- Document changes in CHANGELOG
- Track which code is in production
- Easy rollback if issues occur

---

## Comparison: Development vs Production

| Aspect | Development | Production |
|--------|-------------|-----------|
| **Running** | Local notebook/script | Cloud platform (Agent Engine) |
| **Availability** | Only when you run it | 24/7 uptime |
| **Scaling** | Single instance | Auto-scaling |
| **Access** | Only you | Multiple users |
| **Monitoring** | Print statements | Cloud Logging/Monitoring |
| **Cost** | Free | Pay for usage |
| **Session Management** | InMemorySessionService | Built-in persistent sessions |
| **Debugging** | `adk web --log_level DEBUG` | Cloud Trace + Logging |

---

## Key Takeaways

### From Day 5a (A2A Communication):
1. **A2A Protocol:** Standard for agent-to-agent communication across networks
2. **Use Cases:** Cross-organization, cross-language, cross-framework integration
3. **Agent Cards:** JSON documents that describe agent capabilities
4. **Exposing:** Use `to_a2a()` to make agents accessible
5. **Consuming:** Use `RemoteA2aAgent` to connect to remote agents
6. **Transparency:** Remote agents work like local sub-agents
7. **Decision Framework:** Use A2A for external/distributed agents, local sub-agents for internal workflows

### From Day 5b (Production Deployment):
1. **Deployment Need:** Agents must be deployed to be publicly accessible
2. **Vertex AI Agent Engine:** Fully managed platform for agent hosting
3. **File Requirements:** agent.py, requirements.txt, .env, .agent_engine_config.json
4. **Deployment Command:** `adk deploy agent_engine`
5. **Memory Bank:** Long-term memory across sessions (optional)
6. **Cost Management:** Use `min_instances: 0` and delete when done
7. **Production Best Practices:** Observability, error handling, security, monitoring

---

## Complete Deployment Workflow

### End-to-End Production Deployment

```
Phase 1: Development
├─ Build agent locally
├─ Test with adk web
├─ Debug with observability
└─ Create evaluation suite

Phase 2: Pre-Deployment
├─ Create deployment files:
│  ├─ agent.py
│  ├─ requirements.txt
│  ├─ .env
│  └─ .agent_engine_config.json
├─ Run evaluation tests
└─ Review configurations

Phase 3: Deployment
├─ Authenticate with GCP
├─ Set PROJECT_ID
├─ Run: adk deploy agent_engine
└─ Wait 2-5 minutes

Phase 4: Testing
├─ Retrieve deployed agent
├─ Test with async_stream_query
├─ Verify responses
└─ Check observability

Phase 5: Production
├─ Monitor Cloud Console
├─ Review logs and traces
├─ Handle issues
└─ Scale as needed

Phase 6: Cleanup
├─ Delete test deployments
├─ Verify in console
└─ Check billing
```

---

## Advanced Topics

### 1. Multi-Agent Deployment with A2A

**Scenario:** Deploy multiple agents that communicate via A2A

```python
# Deploy Agent 1: Product Catalog
!adk deploy agent_engine \
    --project=$PROJECT_ID \
    --region=us-west1 \
    product_catalog_agent

# Deploy Agent 2: Customer Support (consumes Agent 1 via A2A)
!adk deploy agent_engine \
    --project=$PROJECT_ID \
    --region=us-west1 \
    customer_support_agent
```

**Benefits:**
- Each agent scales independently
- Clear service boundaries
- Easier to maintain and update
- Different teams can own different agents

### 2. Hybrid Deployment

**Scenario:** Mix local and remote agents

```python
# Remote agent (deployed to Agent Engine)
remote_inventory = RemoteA2aAgent(
    agent_card="https://production.company.com/.well-known/agent-card.json"
)

# Local agent (runs in your infrastructure)
local_processor = LlmAgent(
    name="order_processor",
    sub_agents=[remote_inventory]  # Uses remote agent
)
```

### 3. Multi-Region Deployment

**Scenario:** Deploy to multiple regions for global coverage

```python
# Deploy to US
!adk deploy agent_engine --region=us-west1 my_agent

# Deploy to Europe
!adk deploy agent_engine --region=europe-west1 my_agent

# Deploy to Asia
!adk deploy agent_engine --region=asia-northeast1 my_agent
```

**Benefits:**
- Lower latency for global users
- Better availability (redundancy)
- Compliance with regional regulations

---

## Decision Framework: A2A + Deployment

### When to Use A2A in Production

**Use A2A when:**
- ✅ Integrating with external vendors
- ✅ Different teams own different agents
- ✅ Agents deployed on different infrastructure
- ✅ Need cross-language/framework communication
- ✅ Formal API contracts required

**Don't use A2A when:**
- ❌ All agents in same codebase
- ❌ Low latency is critical (milliseconds)
- ❌ No need for network communication
- ❌ Simple internal workflows

### Deployment Platform Choice

| Platform | Best For | Complexity | Cost |
|----------|----------|------------|------|
| **Agent Engine** | Fully managed, agent-specific | Low | Free tier + usage |
| **Cloud Run** | Serverless, simple apps | Low | Pay per request |
| **GKE** | Enterprise, full control | High | Pay for cluster |

**Choose Agent Engine when:**
- Want fully managed solution
- Need built-in session management
- Want easy deployment with `adk deploy`
- Starting with production deployment

**Choose Cloud Run when:**
- Want simplest serverless option
- Have existing Cloud Run experience
- Need fine-grained container control

**Choose GKE when:**
- Need complex multi-agent orchestration
- Require custom networking
- Have Kubernetes expertise
- Enterprise-scale requirements

---

## Resources & Documentation

### A2A Communication
- [A2A Protocol Official Website](https://a2a-protocol.org/)
- [A2A Protocol Specification](https://a2a-protocol.org/latest/spec/)
- [ADK A2A Introduction](https://google.github.io/adk-docs/a2a/intro/)
- [Exposing Agents Quickstart](https://google.github.io/adk-docs/a2a/quickstart-exposing/)
- [Consuming Agents Quickstart](https://google.github.io/adk-docs/a2a/quickstart-consuming/)

### Agent Deployment
- [ADK Deployment Overview](https://google.github.io/adk-docs/deploy/)
- [Deploy to Agent Engine](https://google.github.io/adk-docs/deploy/agent-engine/)
- [Deploy to Cloud Run](https://google.github.io/adk-docs/deploy/cloud-run/)
- [Deploy to GKE](https://google.github.io/adk-docs/deploy/gke/)
- [Agent Engine Documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview)

### Memory & Advanced Features
- [ADK Memory Guide](https://google.github.io/adk-docs/sessions/memory/)
- [Memory Bank with ADK](https://github.com/GoogleCloudPlatform/generative-ai/blob/main/agents/agent_engine/memory_bank/get_started_with_memory_bank_on_adk.ipynb)
- [Built-in Tools](https://google.github.io/adk-docs/tools/built-in-tools/)

### Security & Best Practices
- [ADK Security Guidelines](https://google.github.io/adk-docs/safety/)
- [Google Cloud Security Best Practices](https://cloud.google.com/security/best-practices)

---

## 5-Day Course Recap

Congratulations on completing the 5-day AI Agents course! Here's what you've learned:

### **Day 1:** Agent Fundamentals & Multi-Agent Workflows
- Building your first AI agent
- Tools and instructions
- Multi-agent patterns (Sequential, Parallel, Loop, LLM-based)

### **Day 2:** Advanced Tools & Custom Functions
- Custom function tools
- Built-in tools (Google Search, Code Execution)
- MCP (Model Context Protocol) integration

### **Day 3:** Sessions & Memory Management
- Session management
- Short-term and long-term memory
- State sharing between agents
- Persistent conversation history

### **Day 4:** Observability & Evaluation
- Logs, traces, and metrics
- Debugging with ADK Web UI
- Plugins and callbacks
- Agent evaluation and testing
- Regression prevention

### **Day 5:** Production Deployment & Integration
- A2A protocol for agent communication
- Exposing and consuming remote agents
- Deploying to Vertex AI Agent Engine
- Production best practices
- Cost management and cleanup

---

## What's Next?

**You now have the complete toolkit to build, test, and deploy production-ready AI agents!**

### Next Steps:

1. **Build Your Own Agent**
   - Apply what you've learned
   - Start with a simple use case
   - Iterate and improve

2. **Experiment with A2A**
   - Create multi-agent systems
   - Integrate external services
   - Build microservices architecture

3. **Deploy to Production**
   - Start with Agent Engine free tier
   - Test with real users
   - Monitor and optimize

4. **Join the Community**
   - Share your projects on [Kaggle Discord](https://discord.com/invite/kaggle)
   - Contribute to discussions
   - Learn from others

5. **Explore Advanced Patterns**
   - Custom plugins
   - Advanced memory patterns
   - Multi-region deployment
   - Complex orchestration

**Happy building! 🚀**

---

*Created from notebooks by Lavi Nigam for the Kaggle 5-day AI Agents course*

