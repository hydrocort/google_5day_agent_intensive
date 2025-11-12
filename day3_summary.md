# Day 3 Summary: Context Engineering - Sessions & Memory

This document summarizes the key concepts taught in the Day 3 notebooks of the Kaggle 5-day AI Agents course.

---

## Day 3a: Session Management - Building Stateful Agents

### Core Concept: The Statefulness Problem

**The Problem:**
- Large Language Models are **inherently stateless** - they only know what you tell them in a single API call
- Without context management, agents can't remember previous conversations
- Imagine talking to someone who forgets everything after each sentence

**The Solution:**
ADK uses **Sessions** for short-term memory management and **Memory** for long-term memory.

---

## Section 1: Understanding Sessions

### What is a Session?

**📦 Session** = A container for a single, continuous conversation

A session encapsulates:
- Conversation history in chronological order
- All tool interactions and responses
- User-specific data (not shared with other users)
- Agent-specific context (not shared with other agents)

### Two Key Components

#### 1. **Session.Events** 📝

Events are the building blocks of a conversation.

**Types of Events:**
- **User Input**: Messages from the user (text, audio, image)
- **Agent Response**: The agent's reply to the user
- **Tool Call**: The agent's decision to use an external tool
- **Tool Output**: Data returned from tool calls

**Think of Events as:** Individual entries in a journal page

#### 2. **Session.State** {}

The agent's scratchpad for storing dynamic data during conversations.

**Characteristics:**
- Global `{key, value}` pair storage
- Available to all sub-agents and tools
- Persists throughout the session
- Can store user preferences, context variables, etc.

**Think of State as:** Sticky notes on your desk

---

## Section 2: Session Management Infrastructure

### Session Management Components

ADK provides two key components to manage sessions:

#### 1. **SessionService** 🗄️ (The Storage Layer)

Manages creation, storage, and retrieval of session data.

**Available Implementations:**

| Service | Persistence | Use Case |
|---------|-------------|----------|
| **InMemorySessionService** | ❌ Lost on restart | Development & prototyping |
| **DatabaseSessionService** | ✅ Survives restarts | Small to medium apps |
| **Agent Engine Sessions** | ✅ Fully managed | Production (GCP) |

#### 2. **Runner** 🤖 (The Orchestration Layer)

Manages the flow of information between user and agent.

**Responsibilities:**
- Maintains conversation history automatically
- Handles context engineering behind the scenes
- Coordinates between session and memory services

**Analogy:**
- **Session** = A notebook 📓
- **Events** = Individual entries in a page 📝
- **SessionService** = The filing cabinet 🗄️
- **Runner** = The assistant managing conversations 🤖

---

## Section 3: Implementing Stateful Agents

### Basic Setup

```python
# Step 1: Create the Agent
root_agent = Agent(
    model=Gemini(model="gemini-2.5-flash-lite"),
    name="text_chat_bot",
    description="A text chatbot",
)

# Step 2: Set up Session Management
session_service = InMemorySessionService()  # Temporary storage

# Step 3: Create the Runner
runner = Runner(
    agent=root_agent,
    app_name=APP_NAME,
    session_service=session_service
)
```

### The Problem with InMemorySessionService

**Limitation:** All conversation history is lost when the application stops.

**Why it matters:** In production, users expect to resume conversations after restarts.

---

## Section 4: Persistent Sessions

### Upgrading to DatabaseSessionService

For production applications, use persistent storage:

```python
# Step 1: Create the agent
chatbot_agent = LlmAgent(
    model=Gemini(model="gemini-2.5-flash-lite"),
    name="text_chat_bot",
    description="A text chatbot with persistent memory",
)

# Step 2: Switch to DatabaseSessionService
db_url = "sqlite:///my_agent_data.db"
session_service = DatabaseSessionService(db_url=db_url)

# Step 3: Create runner with persistent storage
runner = Runner(
    agent=chatbot_agent,
    app_name=APP_NAME,
    session_service=session_service
)
```

**Database Storage:**
- Sessions survive application restarts
- Events stored in SQL database
- Can query and analyze conversation history
- Isolated per session (private conversations)

---

## Section 5: Context Compaction

### The Problem: Growing Context Windows

As conversations grow, event lists become very large, leading to:
- Slower performance
- Higher costs (more tokens to process)
- Context window limits

### The Solution: EventsCompactionConfig

**Context Compaction** automatically summarizes the past to reduce stored context.

```python
from google.adk.apps.app import App, EventsCompactionConfig

# Create App with compaction enabled
research_app_compacting = App(
    name="research_app",
    root_agent=chatbot_agent,
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=3,  # Trigger compaction every 3 turns
        overlap_size=1,         # Keep 1 previous turn for context
    ),
)

# Create runner with the App
runner = Runner(
    app=research_app_compacting,
    session_service=session_service
)
```

**How it Works:**

```
Before Compaction:
Turn 1: User asks about AI
Turn 2: Agent responds (1000 tokens)
Turn 3: User asks follow-up
Turn 4: Agent responds (1000 tokens)
Turn 5: User asks another question
Total: 5000 tokens in context

After Compaction (triggered at turn 3):
Summary: "User asked about AI in healthcare, discussed drug discovery"
Turn 3: User asks follow-up
Turn 4: Agent responds
Turn 5: User asks another question
Total: 2000 tokens in context ✅
```

**Benefits:**
- Reduces token usage and costs
- Maintains relevant context
- Improves performance
- Prevents context window overflow

### Advanced Context Engineering

**Custom Compaction:**
- Define custom `SlidingWindowCompactor`
- Control summarization prompt
- Use specialized LLM for summarization

**Context Caching:**
- Cache static instructions
- Reduce token size of repeated prompts
- [Documentation](https://google.github.io/adk-docs/context/caching/)

---

## Section 6: Working with Session State

### Manual State Management

You can create custom tools to manage session state using `ToolContext`.

#### Example: Storing User Information

```python
def save_userinfo(
    tool_context: ToolContext,
    user_name: str,
    country: str
) -> Dict[str, Any]:
    """Tool to record user name and country in session state."""
    # Write to session state using 'user:' prefix
    tool_context.state["user:name"] = user_name
    tool_context.state["user:country"] = country
    return {"status": "success"}

def retrieve_userinfo(tool_context: ToolContext) -> Dict[str, Any]:
    """Tool to retrieve user info from session state."""
    user_name = tool_context.state.get("user:name", "Not found")
    country = tool_context.state.get("user:country", "Not found")
    return {"status": "success", "user_name": user_name, "country": country}
```

#### Using State Management Tools

```python
# Create agent with state tools
root_agent = LlmAgent(
    model=Gemini(model="gemini-2.5-flash-lite"),
    name="text_chat_bot",
    description="""A text chatbot with state management.
    * Use `save_userinfo` to store username and country
    * Use `retrieve_userinfo` to fetch stored information
    """,
    tools=[save_userinfo, retrieve_userinfo],
)
```

### State Scope Best Practices

Use descriptive key prefixes for organization:

| Prefix | Use Case | Example |
|--------|----------|---------|
| `user:` | User-specific data | `user:name`, `user:preferences` |
| `app:` | Application-level data | `app:config`, `app:version` |
| `temp:` | Temporary data | `temp:calculation`, `temp:draft` |

### State Isolation

**Important:** Session state is isolated per session by default.
- State from "session-01" won't automatically appear in "session-02"
- Each conversation maintains its own scratchpad
- Enables private, user-specific experiences

---

## Day 3b: Memory Management - Long-Term Knowledge Storage

### Core Concept: Sessions vs Memory

**The Distinction:**

| Aspect | Session | Memory |
|--------|---------|--------|
| **Duration** | Short-term (single conversation) | Long-term (across conversations) |
| **Scope** | Current conversation thread | All past conversations |
| **Analogy** | Application state (RAM) | Database storage |
| **Use Case** | "What did I just say?" | "What's my favorite color from last week?" |

**Example:**
- 🗣️ **Session**: Remembers what you said 10 minutes ago in THIS conversation
- 🧠 **Memory**: Remembers your preferences from conversations LAST WEEK

---

## Section 1: Why Memory?

### Capabilities Memory Provides

| Capability | What It Means | Example |
|------------|---------------|---------|
| **Cross-Conversation Recall** | Access info from any past conversation | "What preferences has this user mentioned?" |
| **Intelligent Extraction** | LLM consolidates key facts | Stores "allergic to peanuts" not 50 messages |
| **Semantic Search** | Meaning-based retrieval | "preferred hue" matches "favorite color is blue" |
| **Persistent Storage** | Survives application restarts | Knowledge grows over time |

---

## Section 2: Memory Workflow

### Three-Step Integration Process

**1. Initialize** → Create `MemoryService` and provide to Runner
**2. Ingest** → Transfer session data to memory using `add_session_to_memory()`
**3. Retrieve** → Search stored memories using `search_memory()`

```
┌─────────────┐
│  User Query │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  Session Events │  ← Short-term (current conversation)
└──────┬──────────┘
       │ add_session_to_memory()
       ▼
┌──────────────┐
│    Memory    │  ← Long-term (all conversations)
└──────┬───────┘
       │ search_memory()
       ▼
┌──────────────┐
│ Agent Recall │
└──────────────┘
```

---

## Section 3: Initialize MemoryService

### Available Memory Services

ADK provides multiple implementations:

| Service | Search Method | Persistence | Use Case |
|---------|--------------|-------------|----------|
| **InMemoryMemoryService** | Keyword matching | ❌ In-memory only | Learning & testing |
| **VertexAiMemoryBankService** | Semantic search | ✅ Cloud storage | Production |

### Basic Setup

```python
from google.adk.memory import InMemoryMemoryService

# Step 1: Initialize Memory Service
memory_service = InMemoryMemoryService()

# Step 2: Create agent
user_agent = LlmAgent(
    model=Gemini(model="gemini-2.5-flash-lite"),
    name="MemoryDemoAgent",
    instruction="Answer user questions in simple words.",
)

# Step 3: Create runner with BOTH services
runner = Runner(
    agent=user_agent,
    app_name="MemoryDemoApp",
    session_service=session_service,  # Short-term memory
    memory_service=memory_service,    # Long-term memory
)
```

**💡 Important:** Adding `memory_service` to Runner makes memory *available* but doesn't automatically use it. You must explicitly ingest data and enable retrieval.

---

## Section 4: Ingest Data into Memory

### Why Transfer Sessions to Memory?

**The Flow:**
1. Conversations start in Sessions (raw events)
2. You explicitly transfer to Memory (processed knowledge)
3. Memory becomes searchable across all sessions

### Manual Ingestion

```python
# Step 1: Have a conversation (stored in session)
await run_session(
    runner,
    "My favorite color is blue-green.",
    "conversation-01"
)

# Step 2: Get the session
session = await session_service.get_session(
    app_name=APP_NAME,
    user_id=USER_ID,
    session_id="conversation-01"
)

# Step 3: Transfer to memory
await memory_service.add_session_to_memory(session)
```

**What happens during transfer:**

**InMemoryMemoryService:**
- Stores raw conversation events
- No transformation

**VertexAiMemoryBankService (Production):**
- Intelligently consolidates conversations
- Extracts key facts
- Discards conversational noise
- Creates searchable knowledge

---

## Section 5: Enable Memory Retrieval

### Memory Retrieval Tools

Agents need tools to search memory. ADK provides two built-in options:

#### 1. **load_memory** (Reactive)

```python
from google.adk.tools import load_memory

user_agent = LlmAgent(
    model=Gemini(model="gemini-2.5-flash-lite"),
    name="MemoryAgent",
    instruction="Use load_memory tool when you need past context.",
    tools=[load_memory],  # Agent decides when to search
)
```

**Characteristics:**
- Agent decides when to search memory
- More efficient (saves tokens)
- Only retrieves when needed
- Risk: Agent might forget to search

**Use Case:** General conversations where memory isn't always needed

---

#### 2. **preload_memory** (Proactive)

```python
from google.adk.tools import preload_memory

user_agent = LlmAgent(
    model=Gemini(model="gemini-2.5-flash-lite"),
    name="MemoryAgent",
    instruction="Answer questions using available memory.",
    tools=[preload_memory],  # Automatically loads memory
)
```

**Characteristics:**
- Automatically searches before every turn
- Memory always available to agent
- Guaranteed context
- Less efficient (searches even when not needed)

**Use Case:** Personalized assistants where context is critical

---

### Comparison

**Think of it like studying for an exam:**

| Pattern | Analogy | Efficiency | Reliability |
|---------|---------|------------|-------------|
| **load_memory** | Looking up info only when needed | ✅ High | ⚠️ Depends on agent |
| **preload_memory** | Reading all notes before each question | ⚠️ Lower | ✅ Guaranteed |

---

## Section 6: Manual Memory Search

### Programmatic Memory Access

Beyond agent tools, search memories directly in your code:

```python
# Search for specific information
search_response = await memory_service.search_memory(
    app_name=APP_NAME,
    user_id=USER_ID,
    query="What is the user's favorite color?"
)

# Process results
for memory in search_response.memories:
    if memory.content and memory.content.parts:
        print(f"[{memory.author}]: {memory.content.parts[0].text}")
```

**Use Cases:**
- Debugging memory contents
- Building analytics dashboards
- Creating custom memory management UIs
- Exporting user data

### How Search Works

**InMemoryMemoryService:**
- **Method:** Keyword matching
- **Example:** "favorite color" matches because those words exist
- **Limitation:** "preferred hue" won't match

**VertexAiMemoryBankService:**
- **Method:** Semantic search via embeddings
- **Example:** "preferred hue" WILL match "favorite color"
- **Advantage:** Understands meaning, not just keywords

---

## Section 7: Automating Memory Storage

### The Problem with Manual Ingestion

Manual calls to `add_session_to_memory()` don't scale:
- Easy to forget
- Requires explicit code after each conversation
- Not suitable for production

**Solution:** Use callbacks for automation

---

### Callbacks in ADK

**What are Callbacks?**

Callbacks are Python functions that ADK automatically invokes at specific stages of agent execution.

**Think of callbacks as:** Event listeners in the agent's lifecycle

### Available Callback Types

```python
┌───────────────────────────────┐
│   before_agent_callback       │  ← Before agent processes request
└───────────────────────────────┘
               │
               ▼
┌───────────────────────────────┐
│   before_model_callback       │  ← Before LLM call
└───────────────────────────────┘
               │
               ▼
┌───────────────────────────────┐
│   before_tool_callback        │  ← Before tool execution
└───────────────────────────────┘
               │
               ▼
┌───────────────────────────────┐
│   after_tool_callback         │  ← After tool execution
└───────────────────────────────┘
               │
               ▼
┌───────────────────────────────┐
│   after_model_callback        │  ← After LLM responds
└───────────────────────────────┘
               │
               ▼
┌───────────────────────────────┐
│   after_agent_callback        │  ← After agent completes turn
└───────────────────────────────┘
```

### Common Use Cases

| Callback Type | Use Case |
|--------------|----------|
| `before_agent_callback` | Logging, authentication checks |
| `after_agent_callback` | **Memory storage**, analytics |
| `before_tool_callback` | Tool access validation |
| `after_tool_callback` | Tool usage logging |
| `on_model_error_callback` | Error handling, retry logic |

---

### Automatic Memory Storage Implementation

#### Step 1: Define the Callback

```python
async def auto_save_to_memory(callback_context):
    """Automatically save session to memory after each agent turn."""
    await callback_context._invocation_context.memory_service.add_session_to_memory(
        callback_context._invocation_context.session
    )
```

**Key Concept:** `callback_context`
- ADK automatically passes this parameter
- Provides access to memory service and current session
- You don't create it; ADK provides it

#### Step 2: Attach Callback to Agent

```python
auto_memory_agent = LlmAgent(
    model=Gemini(model="gemini-2.5-flash-lite"),
    name="AutoMemoryAgent",
    instruction="Answer user questions.",
    tools=[preload_memory],              # Automatic retrieval
    after_agent_callback=auto_save_to_memory,  # Automatic storage
)
```

#### Step 3: Test Fully Automated System

```python
# Create runner
auto_runner = Runner(
    agent=auto_memory_agent,
    app_name=APP_NAME,
    session_service=session_service,
    memory_service=memory_service,
)

# Test 1: Tell agent something (automatically saved)
await run_session(
    auto_runner,
    "I gifted a toy to my nephew on his 1st birthday!",
    "auto-save-test"
)

# Test 2: Ask about it in NEW session (automatically retrieved)
await run_session(
    auto_runner,
    "What did I gift my nephew?",
    "auto-save-test-2"  # Different session - proves memory works!
)
```

**What happens automatically:**
1. First conversation → Callback saves to memory ✅
2. Second conversation (new session) → `preload_memory` retrieves ✅
3. Agent answers correctly ✅
4. **Zero manual memory calls!** ✅

---

### When to Save Sessions to Memory

| Timing | Implementation | Best For |
|--------|----------------|----------|
| **After every turn** | `after_agent_callback` | Real-time memory updates |
| **End of conversation** | Manual call when session ends | Batch processing, reduce API calls |
| **Periodic intervals** | Timer-based background job | Long-running conversations |

---

## Section 8: Memory Consolidation

### The Problem with Raw Storage

**What we've stored so far:**
- Every user message
- Every agent response
- Every tool call

**The scaling problem:**

```
Session: 50 messages = 10,000 tokens
Memory:  All 50 messages stored
Search:  Returns all 50 messages
Agent:   Must process 10,000 tokens
```

**This doesn't scale.** We need consolidation.

---

### What is Memory Consolidation?

**Memory Consolidation** = Extracting **only important facts** while discarding conversational noise.

#### Before Consolidation (Raw Storage)

```
User: "My favorite color is BlueGreen. I also like purple.
       Actually, I prefer BlueGreen most of the time."
Agent: "Great! I'll remember that."
User: "Thanks!"
Agent: "You're welcome!"

→ Stores ALL 4 messages (redundant, verbose)
→ Total: ~100 tokens
```

#### After Consolidation

```
Extracted Memory: "User's favorite color: BlueGreen"

→ Stores 1 concise fact
→ Total: ~10 tokens
```

**Benefits:**
- 90% reduction in storage
- Faster retrieval
- More accurate answers
- Better scalability

---

### How Consolidation Works

**The Pipeline:**

```
1. Raw Session Events
   ↓
2. LLM analyzes conversation
   ↓
3. Extracts key facts
   ↓
4. Removes redundancy
   ↓
5. Stores concise memories
   ↓
6. Merges with existing memories (deduplication)
```

**Example Transformation:**

```
Input:
User: "I'm allergic to peanuts."
User: "I can't eat anything with nuts."
User: "No tree nuts either."

Output:
Memory {
  allergy: "peanuts, tree nuts"
  severity: "must avoid"
  category: "food"
}
```

**Result:** Natural language → Structured, actionable data

---

### Consolidation in Practice

**InMemoryMemoryService (This notebook):**
- ❌ No consolidation
- Stores raw events as-is
- Good for learning

**VertexAiMemoryBankService (Production):**
- ✅ Automatic consolidation
- LLM extracts key facts
- Merges duplicate information
- You'll explore this in Day 5!

**💡 Important:** The API remains the same:
- `add_session_to_memory()` ← Same method
- `search_memory()` ← Same method

Only the **behind-the-scenes processing** changes.

---

## Summary: Key Patterns and Best Practices

### Sessions Best Practices

**1. Choose the Right SessionService:**

| Need | Use | Persistence |
|------|-----|-------------|
| Prototyping | `InMemorySessionService` | ❌ Temporary |
| Production (self-hosted) | `DatabaseSessionService` | ✅ Database |
| Production (GCP) | Agent Engine Sessions | ✅ Managed |

**2. Enable Context Compaction for Long Conversations:**
```python
events_compaction_config=EventsCompactionConfig(
    compaction_interval=3,  # Summarize every 3 turns
    overlap_size=1          # Keep 1 turn for context
)
```

**3. Use State Prefixes for Organization:**
- `user:name` → User-specific data
- `app:config` → Application settings
- `temp:draft` → Temporary calculations

**4. Isolate Sessions for Privacy:**
- Each user gets their own sessions
- Sessions don't share state by default
- Enables personalized experiences

---

### Memory Best Practices

**1. Choose the Right MemoryService:**

| Need | Use | Search Method |
|------|-----|--------------|
| Learning | `InMemoryMemoryService` | Keyword matching |
| Production | `VertexAiMemoryBankService` | Semantic search |

**2. Choose the Right Retrieval Pattern:**

| Pattern | When to Use |
|---------|-------------|
| `load_memory` | General conversations, efficiency matters |
| `preload_memory` | Personalization critical, always need context |

**3. Automate Memory Storage:**
```python
after_agent_callback=auto_save_to_memory  # Automatic after each turn
```

**4. Leverage Consolidation in Production:**
- Use managed services (Vertex AI Memory Bank)
- Automatic fact extraction
- Reduced storage and retrieval costs
- Better accuracy

**5. Test Memory Isolation:**
- Memories are user-specific
- Don't leak across users
- Test privacy boundaries

---

## Decision Framework

### When to Use What?

**Use Session State when:**
- Need temporary data for current conversation
- Tracking workflow state
- Storing intermediate calculations
- Single-session context sufficient

**Use Memory when:**
- Need long-term knowledge across conversations
- User preferences and characteristics
- Historical interactions matter
- Building personalized experiences

**Use Both when:**
- Building production assistants
- Session tracks current conversation flow
- Memory provides long-term context
- Example: Customer support bot

---

## Key Takeaways

### From Day 3a (Sessions):

1. **Sessions provide short-term memory** for single conversations
2. **Events capture everything** that happens (messages, tool calls, responses)
3. **State provides a scratchpad** for storing dynamic data
4. **DatabaseSessionService enables persistence** across restarts
5. **Context Compaction reduces costs** by summarizing history
6. **Manual state management** requires custom tools and discipline

### From Day 3b (Memory):

1. **Memory provides long-term storage** across multiple conversations
2. **Three-step workflow**: Initialize, Ingest, Retrieve
3. **Two retrieval patterns**: `load_memory` (reactive) vs `preload_memory` (proactive)
4. **Callbacks enable automation** of memory storage
5. **Consolidation extracts facts** from raw conversations
6. **Managed services provide semantic search** and automatic consolidation

---

## Common Patterns Summary

### Complete Stateful Agent Setup

```python
# 1. Initialize Services
session_service = DatabaseSessionService(db_url="sqlite:///agent_data.db")
memory_service = InMemoryMemoryService()

# 2. Define Callback for Auto-Save
async def auto_save_to_memory(callback_context):
    await callback_context._invocation_context.memory_service.add_session_to_memory(
        callback_context._invocation_context.session
    )

# 3. Create Agent with Memory Tools
agent = LlmAgent(
    model=Gemini(model="gemini-2.5-flash-lite"),
    name="StatefulAgent",
    instruction="Answer questions using past context.",
    tools=[preload_memory],  # or load_memory
    after_agent_callback=auto_save_to_memory,
)

# 4. Create App with Compaction
app = App(
    name="MyApp",
    root_agent=agent,
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=3,
        overlap_size=1,
    ),
)

# 5. Create Runner with All Services
runner = Runner(
    app=app,
    session_service=session_service,
    memory_service=memory_service,
)
```

### Manual Memory Management

```python
# Have conversation
await run_session(runner, "My birthday is March 15", "session-01")

# Get session
session = await session_service.get_session(
    app_name=APP_NAME,
    user_id=USER_ID,
    session_id="session-01"
)

# Save to memory
await memory_service.add_session_to_memory(session)

# Search memory
results = await memory_service.search_memory(
    app_name=APP_NAME,
    user_id=USER_ID,
    query="When is my birthday?"
)
```

### Session State Management

```python
def save_preference(tool_context: ToolContext, key: str, value: str):
    """Save user preference to session state."""
    tool_context.state[f"user:{key}"] = value
    return {"status": "success"}

def get_preference(tool_context: ToolContext, key: str):
    """Retrieve user preference from session state."""
    value = tool_context.state.get(f"user:{key}", "Not found")
    return {"status": "success", "value": value}

# Add tools to agent
agent = LlmAgent(
    tools=[save_preference, get_preference],
    # ... other config
)
```

---

## Architecture Diagram

```
┌────────────────────────────────────────────────────────┐
│                   Agentic Application                  │
└────────────────────────────────────────────────────────┘
                         │
        ┌────────────────┴────────────────┐
        │                                 │
        ▼                                 ▼
┌──────────────┐                 ┌──────────────┐
│   Sessions   │                 │    Memory    │
│ (Short-term) │                 │ (Long-term)  │
└──────────────┘                 └──────────────┘
        │                                 │
        ├─────────────┐                   │
        ▼             ▼                   ▼
┌────────────┐  ┌────────────┐   ┌────────────┐
│   Events   │  │   State    │   │Consolidated│
│            │  │            │   │   Facts    │
│ • User msg │  │ • user:name│   │            │
│ • Agent    │  │ • app:cfg  │   │ • Prefs    │
│ • Tool use │  │ • temp:val │   │ • History  │
└────────────┘  └────────────┘   │ • Context  │
                                 └────────────┘
        │                                 │
        └─────────────┬───────────────────┘
                      │
                      ▼
              ┌──────────────┐
              │    Runner    │
              └──────────────┘
                      │
                      ▼
              ┌──────────────┐
              │    Agent     │
              └──────────────┘
```

---

## Production Considerations

### Context Management Strategy

**Short Conversations (<10 turns):**
- Sessions sufficient
- No compaction needed
- Minimal memory storage

**Medium Conversations (10-50 turns):**
- Enable context compaction
- Periodic memory saves
- Use `load_memory` pattern

**Long Conversations (>50 turns):**
- Aggressive compaction (interval=3)
- Automatic memory saves after each turn
- Use `preload_memory` pattern
- Consider session rotation

### Performance Optimization

**Token Efficiency:**
- Enable context compaction early
- Use memory consolidation (Vertex AI)
- Prefer `load_memory` over `preload_memory` when possible

**Storage Optimization:**
- Regular memory consolidation
- Prune old sessions
- Archive inactive conversations

**Search Performance:**
- Semantic search (Vertex AI) scales better
- Keyword search (InMemory) limited to small datasets
- Index frequently accessed patterns

### Privacy & Compliance

**Data Isolation:**
- Sessions are user-specific by default
- Memories are user-scoped
- Test cross-user data leakage

**Data Retention:**
- Implement session expiration
- Memory cleanup policies
- User data deletion workflows

**Audit Trail:**
- Events provide complete conversation logs
- Database sessions enable compliance reporting
- Callback logging for monitoring

---

## Next Steps

Day 4 will cover:
- **Observability**: Understanding what your agents are doing
- **Evaluation**: Measuring agent performance
- **Production monitoring**: Ensuring reliability at scale

---

## Learn More

**Official Documentation:**
- [ADK Sessions](https://google.github.io/adk-docs/sessions/)
- [ADK Memory](https://google.github.io/adk-docs/sessions/memory/)
- [Context Compaction](https://google.github.io/adk-docs/context/compaction/)
- [Context Caching](https://google.github.io/adk-docs/context/caching/)
- [Callbacks](https://google.github.io/adk-docs/agents/callbacks/)

**Production Services:**
- [Vertex AI Memory Bank](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/memory-bank/overview)
- [Memory Consolidation Guide](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/memory-bank/generate-memories)

**Advanced Topics:**
- [Session State Management](https://medium.com/google-cloud/2-minute-adk-manage-context-efficiently-with-artifacts-6fcc6683d274)
- [Custom Compaction Strategies](https://google.github.io/adk-docs/context/compaction/#define-compactor)

---

*Created from notebooks by Sampath M for the Kaggle 5-day AI Agents course*

