# Day 1 Summary: Building AI Agents with Google ADK

This document summarizes the key concepts taught in the Day 1 notebooks of the Kaggle 5-day AI Agents course.

---

## Day 1a: From Prompt to Action - Your First AI Agent

### Core Concept: What is an AI Agent?

**Traditional LLM Interaction:**
```
Prompt → LLM → Text Response
```

**AI Agent Interaction:**
```
Prompt → Agent → Thought → Action → Observation → Final Answer
```

An AI agent goes beyond simple text responses—it can **think, take actions, and observe results** to provide better answers.

### Key Components

#### 1. Agent Development Kit (ADK)
- Open-source framework from Google for building and deploying AI agents
- Model-agnostic and deployment-agnostic
- Available in Python, Java, and Go
- Optimized for Gemini but works with other LLMs

#### 2. Agent Configuration
An `Agent` is configured with these key properties:

- **name**: Identifier for the agent
- **description**: Brief description of the agent's purpose
- **model**: The LLM powering the agent's reasoning (e.g., "gemini-2.5-flash-lite")
- **instruction**: The agent's guiding prompt that defines its goal and behavior
- **tools**: List of tools the agent can use to take actions

#### 3. Tools
Tools enable agents to interact with the external world:
- **google_search**: Allows agents to find up-to-date information online
- Tools are specified in the `tools` parameter when creating an agent

#### 4. Runner
The `InMemoryRunner` is the orchestrator that:
- Manages conversations
- Sends messages to agents
- Handles agent responses
- Method: `.run_debug()` for quick prototyping (abstracts session management)

### Example Agent Structure

```python
root_agent = Agent(
    name="helpful_assistant",
    model="gemini-2.5-flash-lite",
    description="A simple agent that can answer general questions.",
    instruction="You are a helpful assistant. Use Google Search for current info or if unsure.",
    tools=[google_search],
)

runner = InMemoryRunner(agent=root_agent)
response = await runner.run_debug("Your question here")
```

### Additional Features

- **ADK Web UI**: Built-in web interface for testing and debugging agents
- **Command-line tools**: `adk create`, `adk run`, `adk web`, `adk api_server`
- **Development workflow**: Create agent projects with `adk create` command

---

## Day 1b: Multi-Agent Systems & Workflow Patterns

### Core Concept: Why Multi-Agent Systems?

**The Problem with Monolithic Agents:**
- Trying to do everything (research, writing, editing, fact-checking) in one agent
- Long, confusing instruction prompts
- Hard to debug and maintain
- Unreliable results

**The Solution:**
Build a **team of specialized agents** where each agent has one clear job, making them:
- Easier to build and test
- More maintainable
- More powerful and reliable when working together

### Key Architecture Patterns

#### 1. LLM-Based Orchestration (Dynamic)

**When to Use:** When you need the LLM to make dynamic decisions about which agents to call and in what order.

**Structure:**
- Root agent acts as a "manager" or "coordinator"
- Sub-agents wrapped as tools using `AgentTool`
- LLM instructions guide the workflow

**Key Concepts:**
- `output_key`: Stores agent results in session state for other agents to use
- `{variable}` placeholders in instructions to inject state values
- `AgentTool`: Wraps sub-agents to make them callable tools

**Limitation:** Can be unpredictable—the LLM might skip steps or run them out of order.

**Example:**
```python
root_agent = Agent(
    name="ResearchCoordinator",
    model="gemini-2.5-flash-lite",
    instruction="Orchestrate research then summarization workflow",
    tools=[AgentTool(research_agent), AgentTool(summarizer_agent)],
)
```

---

#### 2. Sequential Agent (Assembly Line)

**When to Use:** When order matters and you need a guaranteed, step-by-step execution pipeline.

**Structure:**
- Fixed pipeline where agents run in the exact order listed
- Output of one agent automatically becomes input for the next
- Predictable and reliable workflow

**Use Cases:**
- Linear pipelines (Outline → Write → Edit)
- Tasks that build on previous steps
- When deterministic order is critical

**Example:**
```python
root_agent = SequentialAgent(
    name="BlogPipeline",
    sub_agents=[outline_agent, writer_agent, editor_agent],
)
```

**State Passing:**
- Each agent uses `output_key` to store its result
- Subsequent agents use `{output_key}` placeholders to access previous results

---

#### 3. Parallel Agent (Concurrent Execution)

**When to Use:** When you have independent tasks that can run simultaneously to speed up workflows.

**Structure:**
- Executes all sub-agents concurrently
- No dependencies between parallel tasks
- Results can be aggregated after completion

**Use Cases:**
- Multi-topic research
- Independent data collection
- When speed matters and tasks are truly independent

**Example:**
```python
parallel_research_team = ParallelAgent(
    name="ParallelResearchTeam",
    sub_agents=[tech_researcher, health_researcher, finance_researcher],
)

# Often combined with Sequential for aggregation
root_agent = SequentialAgent(
    name="ResearchSystem",
    sub_agents=[parallel_research_team, aggregator_agent],
)
```

---

#### 4. Loop Agent (Iterative Refinement)

**When to Use:** When a task needs to be improved through cycles of feedback and revision.

**Structure:**
- Runs sub-agents repeatedly until a condition is met or max iterations reached
- Creates refinement cycles for self-improvement
- Requires explicit exit mechanism

**Use Cases:**
- Iterative improvement workflows
- Quality refinement (Writer + Critic)
- Tasks requiring multiple revision cycles

**Key Components:**
- **Exit function**: Python function that signals loop termination
- **FunctionTool**: Wraps the exit function so an agent can call it
- **max_iterations**: Prevents infinite loops

**Example:**
```python
def exit_loop():
    """Signal to exit the refinement loop"""
    return {"status": "approved", "message": "Exiting loop"}

refiner_agent = Agent(
    name="RefinerAgent",
    instruction="If critique is 'APPROVED', call exit_loop, else refine story",
    tools=[FunctionTool(exit_loop)],
    output_key="current_story",
)

story_refinement_loop = LoopAgent(
    name="StoryRefinementLoop",
    sub_agents=[critic_agent, refiner_agent],
    max_iterations=2,
)
```

---

### Decision Framework: Choosing the Right Pattern

| Pattern | When to Use | Key Feature | Example Use Case |
|---------|-------------|-------------|------------------|
| **LLM-based Orchestration** | Dynamic decisions needed | LLM decides what to call | Research + Summarization |
| **Sequential** | Order matters, linear pipeline | Deterministic order | Outline → Write → Edit |
| **Parallel** | Independent tasks, speed matters | Concurrent execution | Multi-topic research |
| **Loop** | Iterative improvement needed | Repeated cycles | Writer + Critic refinement |

### Decision Tree

1. **Need a fixed pipeline?** → Use `SequentialAgent`
2. **Need concurrent execution?** → Use `ParallelAgent`
3. **Need iterative refinement?** → Use `LoopAgent`
4. **Need dynamic orchestration?** → Use LLM-based coordination with `AgentTool`

---

## Key Takeaways

### From Day 1a:
1. **Agents vs LLMs**: Agents can take actions, not just respond with text
2. **Tools**: Enable agents to interact with external systems (like Google Search)
3. **ADK Framework**: Provides structure for building agent-based applications
4. **Runner**: Orchestrates agent execution and manages conversation flow

### From Day 1b:
1. **Specialization**: Break complex tasks into specialized agents rather than one monolithic agent
2. **State Management**: Use `output_key` and `{placeholders}` to pass data between agents
3. **Workflow Patterns**: Choose the right pattern based on task dependencies and requirements
4. **Composition**: Patterns can be nested (e.g., Parallel inside Sequential)
5. **Tools for Agents**: Agents can be tools for other agents using `AgentTool`

---

## Common Patterns Summary

### State Sharing Between Agents
```python
agent1 = Agent(
    name="Agent1",
    instruction="Do task 1",
    output_key="result1",  # Stores result in session state
)

agent2 = Agent(
    name="Agent2",
    instruction="Use this data: {result1}",  # Accesses previous result
    output_key="result2",
)
```

### Nested Workflows
```python
# Parallel execution followed by aggregation
parallel_step = ParallelAgent(
    sub_agents=[agent1, agent2, agent3]
)

root = SequentialAgent(
    sub_agents=[parallel_step, aggregator_agent]
)
```

### Loop with Exit Condition
```python
def exit_loop():
    return {"status": "done"}

decision_agent = Agent(
    instruction="If condition met, call exit_loop",
    tools=[FunctionTool(exit_loop)],
)

loop = LoopAgent(
    sub_agents=[worker_agent, decision_agent],
    max_iterations=5,
)
```

---

## Development Best Practices

1. **Start Simple**: Begin with single agents, then compose into multi-agent systems
2. **Clear Instructions**: Give each agent specific, focused instructions
3. **Use output_key**: Always specify output_key for state management
4. **Test Incrementally**: Test each agent individually before combining
5. **Choose the Right Pattern**: Match the workflow pattern to your task requirements
6. **Set Max Iterations**: Always set max_iterations on LoopAgent to prevent infinite loops
7. **Use Web UI**: Leverage ADK web interface for debugging and visualization

---

## Next Steps

Day 2 will cover:
- Custom functions
- MCP (Model Context Protocol) tools
- Managing long-running operations

---

*Created from notebooks by Kristopher Overholt for the Kaggle 5-day AI Agents course*

