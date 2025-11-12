# Day 2 Summary: Agent Tools and Best Practices

This document summarizes the key concepts taught in the Day 2 notebooks of the Kaggle 5-day AI Agents course.

---

## Day 2a: Agent Tools - Building Custom Functions and Multi-Tool Agents

### Core Concept: Why Agents Need Tools

**The Problem:**
- Without tools, an agent's knowledge is frozen in time
- Cannot access current information (today's news, live data)
- No connection to the outside world
- Cannot take actions beyond text responses

**The Solution:**
Tools transform an isolated LLM into a capable agent that can actually get things done.

---

## Section 1: Custom Function Tools

### What Are Custom Tools?

**Custom Tools** = Tools you build yourself using your own code and business logic.

**When to Use:**
- Every business has unique requirements that generic tools can't handle
- Implement specific business logic
- Connect to your systems
- Solve domain-specific problems

### How to Create a Function Tool

**Any Python function can become an agent tool** by following these guidelines:

#### Best Practices for Tool Functions

1. **Dictionary Returns** - Return structured responses with status
2. **Clear Docstrings** - LLMs use docstrings to understand when/how to use tools
3. **Type Hints** - Enable ADK to generate proper schemas
4. **Error Handling** - Structured error responses help LLMs handle failures gracefully

#### Example Pattern

```python
def get_exchange_rate(base_currency: str, target_currency: str) -> dict:
    """Looks up and returns the exchange rate between two currencies.
    
    Args:
        base_currency: The ISO 4217 currency code of the currency you
                       are converting from (e.g., "USD").
        target_currency: The ISO 4217 currency code of the currency you
                         are converting to (e.g., "EUR").
    
    Returns:
        Dictionary with status and rate information.
        Success: {"status": "success", "rate": 0.93}
        Error: {"status": "error", "error_message": "Unsupported currency pair"}
    """
    # Implementation here
    rate = get_rate_from_database(base, target)
    
    if rate is not None:
        return {"status": "success", "rate": rate}
    else:
        return {
            "status": "error", 
            "error_message": f"Unsupported currency pair: {base_currency}/{target_currency}"
        }
```

#### Key Points:

- **Type hints** enable proper schema generation: `str`, `dict`, etc.
- **Docstrings** help LLM understand tool purpose and usage
- **Structured returns** with `status` field for consistent error handling
- **Validation** and error handling built in

### Adding Tools to Agents

```python
currency_agent = LlmAgent(
    name="currency_agent",
    model=Gemini(model="gemini-2.5-flash-lite"),
    instruction="""You are a currency conversion assistant.
    
    For currency conversion requests:
    1. Use get_fee_for_payment_method() to find transaction fees
    2. Use get_exchange_rate() to get conversion rates
    3. Check the "status" field in each tool's response
    4. Calculate the final amount and provide a clear breakdown
    """,
    tools=[get_fee_for_payment_method, get_exchange_rate],  # Just add functions to tools list!
)
```

**How it works:**
- Add Python functions to the `tools=[]` list
- ADK automatically converts them to callable tools
- Agent references tools by exact function names in its reasoning
- Agent can call multiple tools as needed

---

## Section 2: Code Execution for Reliable Calculations

### The Problem: LLMs Aren't Good at Math

- LLMs can make calculation errors
- May use inconsistent formulas
- Unreliable for precise arithmetic

### The Solution: Built-in Code Executor

**Strategy:** Have the agent generate Python code and execute it in a sandbox.

#### BuiltInCodeExecutor

ADK provides a built-in code executor that runs code safely:

```python
calculation_agent = LlmAgent(
    name="CalculationAgent",
    model=Gemini(model="gemini-2.5-flash-lite"),
    instruction="""You are a specialized calculator that ONLY responds with Python code.
    
    RULES:
    1. Your output MUST be ONLY a Python code block
    2. Do NOT write any text before or after the code
    3. The Python code MUST calculate the result
    4. The Python code MUST print the final result to stdout
    5. You are PROHIBITED from performing calculation yourself
    """,
    code_executor=BuiltInCodeExecutor(),  # Gives agent code execution capabilities
)
```

**Benefits:**
- More reliable than LLM arithmetic
- Generates verifiable, inspectable code
- Can handle complex calculations accurately

---

## Section 3: Agent Tools - Using Agents as Tools

### Concept: Agents Can Be Tools for Other Agents

Use `AgentTool` to wrap a specialist agent and make it callable by another agent.

```python
enhanced_currency_agent = LlmAgent(
    name="enhanced_currency_agent",
    model=Gemini(model="gemini-2.5-flash-lite"),
    instruction="""You are a currency conversion assistant.
    
    For calculations:
    - You must use the calculation_agent tool to generate Python code
    - You are strictly prohibited from performing arithmetic yourself
    """,
    tools=[
        get_fee_for_payment_method,      # Function Tool
        get_exchange_rate,                # Function Tool
        AgentTool(agent=calculation_agent),  # Agent Tool!
    ],
)
```

### Agent Tools vs Sub-Agents: What's the Difference?

| Aspect | Agent Tools | Sub-Agents |
|--------|-------------|------------|
| **Control Flow** | Agent B's response goes back to Agent A | Agent B takes over completely |
| **Who's in charge** | Agent A stays in control | Agent B handles all future input |
| **Agent A's role** | Continues conversation with results | Out of the loop |
| **Use Case** | Delegation for specific tasks (calculations, lookups) | Handoff to specialists (customer support tiers) |

**Example:**
- **Agent Tool**: Currency agent calls calculation agent, gets result, continues working
- **Sub-Agent**: Support agent transfers customer to billing specialist who takes over

---

## Section 4: Complete ADK Tool Types

ADK provides two categories of tools:

### 1. Custom Tools
**What:** Tools you build yourself for specific needs  
**Advantage:** Complete control over functionality

#### Function Tools ✅
- **What:** Python functions converted to agent tools
- **When:** Need custom business logic
- **Examples:** `get_fee_for_payment_method`, `get_exchange_rate`

#### Long Running Function Tools ✅
- **What:** Functions for operations that take significant time
- **When:** Need human-in-the-loop approvals, long background tasks
- **Examples:** Approval workflows, file processing

#### Agent Tools ✅
- **What:** Other agents used as tools
- **When:** Need specialist capabilities (calculation, analysis)
- **Examples:** `AgentTool(agent=calculation_agent)`

#### MCP Tools ✅
- **What:** Tools from Model Context Protocol servers
- **When:** Need standardized external service connections
- **Examples:** Filesystem access, Google Maps, databases

#### OpenAPI Tools
- **What:** Tools automatically generated from API specifications
- **When:** Need to interact with REST APIs
- **Examples:** Any REST API endpoint becomes a callable tool

### 2. Built-in Tools
**What:** Pre-built tools provided by ADK  
**Advantage:** No development time - use immediately

#### Gemini Tools ✅
- **What:** Tools leveraging Gemini's capabilities
- **Examples:** `google_search`, `BuiltInCodeExecutor`

#### Google Cloud Tools
- **What:** Tools for Google Cloud services (requires GCP access)
- **Examples:** `BigQueryToolset`, `SpannerToolset`, `APIHubToolset`

#### Third-party Tools
- **What:** Wrappers for existing tool ecosystems
- **Examples:** Hugging Face, Firecrawl, GitHub Tools

---

## Day 2b: Tool Patterns and Best Practices

### Core Concepts: MCP and Long-Running Operations

This notebook covers two production-ready patterns:
1. **MCP Integration** - Connect to external standardized services
2. **Long-Running Operations** - Handle workflows that need to pause for external input

---

## Section 1: Model Context Protocol (MCP)

### The Problem

Connecting to external systems (GitHub, databases, Slack) requires writing and maintaining API clients - lots of custom integration code.

### The Solution: MCP

**Model Context Protocol (MCP)** is an open standard that lets agents use community-built integrations.

**Benefits:**
- ✅ Access live, external data without custom integration code
- ✅ Leverage community-built tools with standardized interfaces
- ✅ Scale capabilities by connecting to multiple specialized servers

### How MCP Works

```
┌──────────────────┐
│   Your Agent     │
│   (MCP Client)   │
└────────┬─────────┘
         │
         │ Standard MCP Protocol
         │
    ┌────┴────┬────────┬────────┐
    │         │        │        │
    ▼         ▼        ▼        ▼
┌────────┐ ┌─────┐ ┌──────┐ ┌─────┐
│ GitHub │ │Slack│ │ Maps │ │ ... │
│ Server │ │ MCP │ │ MCP  │ │     │
└────────┘ └─────┘ └──────┘ └─────┘
```

**Architecture:**
- **MCP Server**: Provides specific tools (image generation, database access)
- **MCP Client**: Your agent that uses those tools
- **Standard Interface**: All servers work the same way

### Using MCP in 4 Steps

#### Step 1: Choose an MCP Server

Example: Everything MCP Server (for testing)
- Package: `@modelcontextprotocol/server-everything`
- Provides: `getTinyImage` tool

Find more servers at: [modelcontextprotocol.io/examples](https://modelcontextprotocol.io/examples)

#### Step 2: Create the MCP Toolset

```python
mcp_image_server = McpToolset(
    connection_params=StdioConnectionParams(
        server_params=StdioServerParameters(
            command="npx",  # Run MCP server via npx
            args=[
                "-y",  # Auto-confirm install
                "@modelcontextprotocol/server-everything",
            ],
            tool_filter=["getTinyImage"],  # Only use this tool
        ),
        timeout=30,
    )
)
```

#### Step 3: Add MCP Tool to Agent

```python
image_agent = LlmAgent(
    model=Gemini(model="gemini-2.5-flash-lite"),
    name="image_agent",
    instruction="Use the MCP Tool to generate images for user queries",
    tools=[mcp_image_server],  # Add MCP toolset
)
```

#### Step 4: Test the Agent

```python
runner = InMemoryRunner(agent=image_agent)
response = await runner.run_debug("Provide a sample tiny image")
```

### Behind the Scenes

1. **Server Launch**: ADK runs `npx -y @modelcontextprotocol/server-everything`
2. **Handshake**: Establishes stdio communication channel
3. **Tool Discovery**: Server tells ADK: "I provide getTinyImage"
4. **Integration**: Tools appear in agent's tool list automatically
5. **Execution**: When agent calls tool, ADK forwards to MCP server
6. **Response**: Server result returned to agent seamlessly

### Other MCP Servers

#### Kaggle MCP Server

```python
McpToolset(
    connection_params=StdioConnectionParams(
        server_params=StdioServerParameters(
            command='npx',
            args=['-y', 'mcp-remote', 'https://www.kaggle.com/mcp'],
        ),
        timeout=30,
    )
)
```

**Provides:**
- 📊 Search and download Kaggle datasets
- 📓 Access notebook metadata
- 🏆 Query competition information

#### GitHub MCP Server

```python
McpToolset(
    connection_params=StreamableHTTPServerParams(
        url="https://api.githubcopilot.com/mcp/",
        headers={
            "Authorization": f"Bearer {GITHUB_TOKEN}",
            "X-MCP-Toolsets": "all",
            "X-MCP-Readonly": "true"
        },
    ),
)
```

---

## Section 2: Long-Running Operations (Human-in-the-Loop)

### The Problem: Immediate Execution Isn't Always Desired

Current flow:
```
User asks → Agent calls tool → Tool returns result → Agent responds
```

**But what if:**
- You need human approval before completing an action?
- The operation takes a long time to complete?
- Compliance requires checkpoints?

### The Solution: Long-Running Operations (LRO)

```
User asks → Agent calls tool → Tool PAUSES → Human approves → Tool completes → Agent responds
```

### When to Use LROs

- 💰 **Financial transactions** requiring approval (transfers, purchases)
- 🗑️ **Bulk operations** (delete 1000 records - confirm first!)
- 📋 **Compliance checkpoints** (regulatory approval needed)
- 💸 **High-cost actions** (spin up 50 servers - are you sure?)
- ⚠️ **Irreversible operations** (permanently delete account)

### Key Components

#### 1. ToolContext

Every long-running tool receives a `ToolContext` parameter (ADK provides this automatically).

**Capabilities:**
- `tool_context.request_confirmation()` - Request approval
- `tool_context.tool_confirmation` - Check approval status

#### 2. Three-Scenario Tool Pattern

```python
def place_shipping_order(
    num_containers: int, 
    destination: str, 
    tool_context: ToolContext
) -> dict:
    """Places a shipping order. Requires approval if ordering more than 5 containers."""
    
    # SCENARIO 1: Small orders - auto-approve
    if num_containers <= 5:
        return {
            "status": "approved",
            "order_id": f"ORD-{num_containers}-AUTO",
            "message": f"Order auto-approved"
        }
    
    # SCENARIO 2: Large order - FIRST CALL - PAUSE
    if not tool_context.tool_confirmation:
        tool_context.request_confirmation(
            hint=f"⚠️ Large order: {num_containers} containers to {destination}. Approve?",
            payload={"num_containers": num_containers, "destination": destination}
        )
        return {
            "status": "pending",
            "message": "Order requires approval"
        }
    
    # SCENARIO 3: Large order - RESUMED CALL - Check approval
    if tool_context.tool_confirmation.confirmed:
        return {
            "status": "approved",
            "order_id": f"ORD-{num_containers}-HUMAN",
            "message": "Order approved"
        }
    else:
        return {
            "status": "rejected",
            "message": "Order rejected"
        }
```

### How the Scenarios Work

**Scenario 1: Small order (≤5 containers)**
- Returns immediately with auto-approved status
- Never checks `tool_context.tool_confirmation`

**Scenario 2: Large order - FIRST CALL**
- Tool detects first call: `if not tool_context.tool_confirmation:`
- Calls `request_confirmation()` to pause
- Returns `{'status': 'pending', ...}`
- ADK creates `adk_request_confirmation` event
- Agent execution pauses

**Scenario 3: Large order - RESUMED CALL**
- Tool detects resumption: `tool_context.tool_confirmation` is now present
- Checks decision: `tool_context.tool_confirmation.confirmed`
- If True → Returns approved status
- If False → Returns rejected status

### Creating a Resumable Agent System

#### Step 1: Create the Agent

```python
shipping_agent = LlmAgent(
    name="shipping_agent",
    model=Gemini(model="gemini-2.5-flash-lite"),
    instruction="""You are a shipping coordinator assistant.
    
    When users request to ship containers:
    1. Use place_shipping_order tool
    2. If status is 'pending', inform user approval is required
    3. After final result, provide clear summary
    """,
    tools=[FunctionTool(func=place_shipping_order)],
)
```

#### Step 2: Wrap in Resumable App

**The Problem:** A regular `LlmAgent` is stateless - no memory between calls.

**The Solution:** Wrap agent in an `App` with resumability enabled.

```python
shipping_app = App(
    name="shipping_coordinator",
    root_agent=shipping_agent,
    resumability_config=ResumabilityConfig(is_resumable=True),
)
```

**What gets saved when tool pauses:**
- All conversation messages
- Which tool was called
- Tool parameters
- Where exactly it paused

When resumed, App loads saved state so agent continues exactly where it left off.

#### Step 3: Create Runner with App

```python
session_service = InMemorySessionService()

shipping_runner = Runner(
    app=shipping_app,  # Pass app instead of agent
    session_service=session_service,
)
```

### Building the Workflow

Your workflow code must:
1. **Detect the pause** - Check for `adk_request_confirmation` event
2. **Get human decision** - In production, show UI; in demo, simulate
3. **Resume the agent** - Send decision back with saved `invocation_id`

#### Key Technical Concepts

**Events:**
- ADK creates events as agent executes
- Tool calls, responses, function results all become events

**`adk_request_confirmation` event:**
- Special event signaling "pause here!"
- Automatically created when tool calls `request_confirmation()`
- Contains the `invocation_id`

**`invocation_id`:**
- Every call to `run_async()` gets unique ID
- When tool pauses, save this ID
- When resuming, pass same ID so ADK continues (not restarts)

### Workflow Implementation

```python
async def run_shipping_workflow(query: str, auto_approve: bool = True):
    """Runs shipping workflow with approval handling."""
    
    # Generate unique session
    session_id = f"order_{uuid.uuid4().hex[:8]}"
    await session_service.create_session(
        app_name="shipping_coordinator",
        user_id="test_user", 
        session_id=session_id
    )
    
    query_content = types.Content(role="user", parts=[types.Part(text=query)])
    events = []
    
    # STEP 1: Send initial request
    async for event in shipping_runner.run_async(
        user_id="test_user", 
        session_id=session_id, 
        new_message=query_content
    ):
        events.append(event)
    
    # STEP 2: Check for approval request
    approval_info = check_for_approval(events)
    
    # STEP 3: Handle approval workflow
    if approval_info:
        print("⏸️  Pausing for approval...")
        
        # Resume with approval decision
        async for event in shipping_runner.run_async(
            user_id="test_user",
            session_id=session_id,
            new_message=create_approval_response(approval_info, auto_approve),
            invocation_id=approval_info["invocation_id"],  # Critical: same ID = RESUME
        ):
            # Display agent response
            print_agent_response(event)
    else:
        # No approval needed - completed immediately
        print_agent_response(events)
```

### Execution Flow Timeline

```
TIME 1: User sends "Ship 10 containers"
TIME 2: Workflow calls run_async(), gets invocation_id = "abc123"
TIME 3: Agent decides to use place_shipping_order tool
TIME 4: ADK calls place_shipping_order(10, "Rotterdam", tool_context)
TIME 5: Tool checks: 10 > 5, calls request_confirmation()
TIME 6: Tool returns {'status': 'pending', ...}
TIME 7: ADK creates adk_request_confirmation event with invocation_id="abc123"
TIME 8: Workflow detects event, saves approval_id and invocation_id="abc123"
TIME 9: Workflow gets human decision → True (approve)
TIME 10: Workflow calls run_async(..., invocation_id="abc123")
TIME 11: ADK sees "abc123" - knows to RESUME (not start new)
TIME 12: ADK calls place_shipping_order again, now with approval
TIME 13: Tool returns {'status': 'approved', ...}
TIME 14: Agent responds to user
```

**Key Point:** The `invocation_id` tells ADK to resume the paused execution instead of starting new.

---

## Summary: Key Patterns for Advanced Tools

| Pattern | When to Use | Key ADK Components |
|---------|-------------|-------------------|
| **Function Tools** | Custom business logic | Function with type hints, docstring |
| **Code Execution** | Reliable calculations | `BuiltInCodeExecutor` |
| **Agent Tools** | Delegation to specialists | `AgentTool` |
| **MCP Integration** | External standardized services without custom code | `McpToolset` |
| **Long-Running Operations** | Pause workflow for external events (human approval) | `ToolContext`, `request_confirmation`, `App`, `ResumabilityConfig` |

---

## Production-Ready Concepts

You now understand how to build agents that:
- 🌐 **Scale**: Leverage community tools instead of building everything
- ⏳ **Handle Time**: Manage operations spanning minutes, hours, or days
- 🔒 **Ensure Compliance**: Add human oversight to critical operations
- 🔄 **Maintain State**: Resume conversations exactly where they paused
- 🛠️ **Delegate**: Use specialist agents for specific tasks
- 🧮 **Calculate Reliably**: Execute code for precise arithmetic

---

## Best Practices Summary

### Function Tool Design
1. Always use type hints for parameters and return values
2. Write clear, detailed docstrings (LLM reads these!)
3. Return structured dictionaries with `status` field
4. Include error handling with descriptive error messages
5. Keep functions focused on one task

### Agent Tool Usage
- Use for delegation (calculation, specialized analysis)
- Agent A stays in control, uses Agent B's results
- Don't confuse with sub-agents (which take over control)

### MCP Integration
- Start with community servers - don't rebuild what exists
- Use `tool_filter` to limit exposed tools
- Same pattern works for any MCP server
- Only connection parameters change

### Long-Running Operations
1. Use `ToolContext` parameter in function signature
2. Implement three scenarios: auto-approve, pause, resume
3. Always wrap agent in resumable `App`
4. Save and reuse `invocation_id` for resumption
5. Detect `adk_request_confirmation` event in workflow

### Development Strategy
**Start Simple → Add Complexity as Needed:**
1. Begin with custom function tools
2. Add MCP services for external integrations
3. Add code execution for calculations
4. Add agent tools for specialization
5. Add long-running operations for approvals

---

## Key Takeaways

### From Day 2a:
1. **Any Python function** can become an agent tool with proper structure
2. **Code execution** is more reliable than LLM math
3. **Agent Tools** enable delegation while maintaining control
4. **Multiple tool types** available for different needs
5. **Built-in tools** provide instant capabilities without coding

### From Day 2b:
1. **MCP** provides standardized way to connect external services
2. **Long-Running Operations** enable human-in-the-loop workflows
3. **Resumability** requires wrapping agents in `App`
4. **Tool state** managed through `ToolContext`
5. **invocation_id** is critical for resuming paused workflows

---

## Next Steps

Day 3 will cover:
- Session management in detail
- State and memory management
- Building stateful, context-aware agents

---

*Created from notebooks by Laxmi Harikumar for the Kaggle 5-day AI Agents course*

