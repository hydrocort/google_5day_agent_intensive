# Day 4 Summary: Agent Observability & Evaluation

This document summarizes the key concepts taught in the Day 4 notebooks of the Kaggle 5-day AI Agents course.

---

## Day 4a: Agent Observability - Logs, Traces & Metrics

### Core Concept: Why Agent Observability Matters

**The Problem with AI Agents:**

Unlike traditional software that fails predictably, AI agents can fail mysteriously:

```
User: "Find quantum computing papers"
Agent: "I cannot help with that request."
You: 😭 WHY?? Is it the prompt? Missing tools? API error?
```

**The Solution: Agent Observability**

Provides complete visibility into your agent's decision-making process:
- See exactly what prompts are sent to the LLM
- Which tools are available
- How the model responds
- Where failures occur

```
DEBUG Log: LLM Request shows "Functions: []" (no tools!)
You: 🎯 Aha! Missing google_search tool - easy fix!
```

### Three Foundational Pillars of Observability

#### 1. **Logs**
- Record of a **single event**
- Tells you **what** happened at a specific moment
- Example: "Agent called google_search tool"

#### 2. **Traces**
- Connects logs into a **single story**
- Shows **why** a final result occurred
- Reveals the entire sequence of steps
- Example: User prompt → Agent thinking → Tool call → LLM response

#### 3. **Metrics**
- Summary numbers and statistics
- Tell you **how** well the agent is performing overall
- Examples: Average response time, success rate, error rate

### Debugging Pattern

**Core debugging workflow:**
```
Symptom → Logs → Root Cause → Fix
```

### Development Debugging: ADK Web UI

#### Using `adk web --log_level DEBUG`

**Key Features:**
- Shows **full LLM prompts** (complete request with system instructions, history, tools)
- Detailed API responses from services
- Internal state transitions and variable values
- Interactive debugging through web interface

**Log Levels:**
- `DEBUG`: Most detailed (development)
- `INFO`: General information
- `WARNING`: Warning messages
- `ERROR`: Error messages only

#### Events Tab - Viewing Traces

**In the ADK Web UI:**
1. **Events tab** shows chronological list of all agent actions
2. Click any event to expand details
3. **Trace button** shows timing information for each step
4. Inspect tool calls, function arguments, and responses
5. Examine LLM requests and responses

**Example Debugging Workflow:**
1. Observe symptom in agent response
2. Check Events tab for execution flow
3. Click specific spans to view details
4. Examine function calls and arguments
5. Identify root cause (e.g., wrong data type passed)
6. Fix and retest

---

## Production Observability: Plugins & Callbacks

### The Challenge

**Problem 1: Production Deployment**
```
You: "Let me open the ADK web UI to check why the agent failed"
DevOps: "This is a production server. No web UI access."
```

**Problem 2: Automated Systems**
```
Boss: "Which runs are slow? What's our success rate?"
You: "I'd have to manually check the web UI 1000 times..."
```

### Solution: Plugins and Callbacks

#### What is a Plugin?

A **Plugin** is a custom code module that runs automatically at various stages of your agent's lifecycle.

**Workflow:**
```
User message → Agent thinks → Calls tools → Returns response
    ↓              ↓              ↓              ↓
  Plugin       Plugin         Plugin         Plugin
  hooks        hooks          hooks          hooks
```

#### Callbacks: Building Blocks of Plugins

**Callbacks** are Python functions that run at specific points in an agent's lifecycle:

- **`before_agent_callback`**: Runs before an agent is invoked
- **`after_agent_callback`**: Runs after an agent completes
- **`before_tool_callback`**: Runs before a tool is called
- **`after_tool_callback`**: Runs after a tool executes
- **`before_model_callback`**: Runs before the LLM model is called
- **`after_model_callback`**: Runs after the LLM responds
- **`on_model_error_callback`**: Runs when a model error occurs

#### Example Plugin Structure

```python
from google.adk.plugins.base_plugin import BasePlugin

class CountInvocationPlugin(BasePlugin):
    """A custom plugin that counts agent and tool invocations."""
    
    def __init__(self):
        super().__init__(name="count_invocation")
        self.agent_count = 0
        self.tool_count = 0
    
    async def before_agent_callback(self, *, agent, callback_context):
        """Count agent runs."""
        self.agent_count += 1
        logging.info(f"[Plugin] Agent run count: {self.agent_count}")
    
    async def before_model_callback(self, *, callback_context, llm_request):
        """Count LLM requests."""
        self.llm_request_count += 1
        logging.info(f"[Plugin] LLM request count: {self.llm_request_count}")
```

**Key Insight:** Register a plugin **once** on your runner, and it automatically applies to **every agent, tool call, and LLM request** in your system.

---

### Built-in LoggingPlugin

ADK provides a built-in **LoggingPlugin** that automatically captures:
- 🚀 User messages and agent responses
- ⏱️ Timing data for performance analysis
- 🧠 LLM requests and responses for debugging
- 🔧 Tool calls and results
- ✅ Complete execution traces

#### Using LoggingPlugin

```python
from google.adk.runners import InMemoryRunner
from google.adk.plugins.logging_plugin import LoggingPlugin

runner = InMemoryRunner(
    agent=your_agent,
    plugins=[LoggingPlugin()],  # Handles standard observability
)

response = await runner.run_debug("Your question here")
```

---

### Decision Framework: When to Use Which Logging?

| Scenario | Logging Approach | Method |
|----------|-----------------|--------|
| **Development debugging** | ADK Web UI | `adk web --log_level DEBUG` |
| **Common production observability** | Built-in Plugin | `LoggingPlugin()` |
| **Custom requirements** | Custom Plugin | Build custom callbacks |

---

## Day 4b: Agent Evaluation

### Core Concept: What is Agent Evaluation?

The **systematic process** of testing and measuring how well an AI agent performs across different scenarios and quality dimensions.

### Why Agents Need Special Evaluation

**The Problem: Standard Testing ≠ Evaluation**

Traditional software testing doesn't work for agents because:
- **Non-deterministic**: Same input can produce different outputs
- **Unpredictable users**: Users give ambiguous, varied commands
- **Prompt sensitivity**: Small prompt changes cause dramatic behavior shifts
- **Complex trajectories**: Need to assess both the response AND the path taken

**Example Scenario:**
```
Week 1: 🚨 "Agent turned on fireplace when I asked for lights!"
Week 2: 🚨 "Agent won't respond to commands in guest room!"
Week 3: 🚨 "Agent gives rude responses when unavailable!"
```

### The Dual Nature of Observability & Evaluation

```
Observability (Reactive) + Agent Evaluation (Proactive)
      ↓                           ↓
  Debug failures            Prevent failures
  After problems            Before problems
  Understand issues         Catch degradation
```

---

## Interactive Evaluation with ADK Web UI

### Creating Test Cases in the UI

**Workflow:**
1. Have a conversation with your agent in ADK Web UI
2. Navigate to the **Eval** tab on the right-hand panel
3. Click **Create Evaluation set** and name it
4. Click **Add current session** to save the conversation
5. Give it a meaningful case name

**What Gets Saved:**
- User input
- Final response
- Tool usage (intermediate data)
- Complete conversation flow

### Running Evaluations

**Steps:**
1. In the Eval tab, check the test cases to run
2. Click **Run Evaluation** button
3. Configure evaluation metrics (or use defaults)
4. Click **Start**
5. View results in Evaluation History

### Understanding Evaluation Metrics

#### 1. **Response Match Score**
- Measures how similar the agent's actual response is to the expected response
- Uses text similarity algorithms
- **Scale:** 1.0 = perfect match, 0.0 = completely different
- **Tests:** Response quality and communication

#### 2. **Tool Trajectory Score**
- Measures whether the agent used correct tools with correct parameters
- Checks the sequence of tool calls against expected behavior
- **Scale:** 1.0 = perfect tool usage, 0.0 = wrong tools or parameters
- **Tests:** Decision-making and tool selection

### Analyzing Failures

**Example Analysis:**
```
Test Case: living_room_light_on
  ❌ response_match_score: 0.45/0.80 (threshold)
  ✅ tool_trajectory_avg_score: 1.0/1.0

Interpretation:
• TOOL USAGE: Perfect - correct tool with correct parameters
• RESPONSE QUALITY: Poor - response text too different
• ROOT CAUSE: Communication style, not functionality
• FIX: Update agent instructions for clearer language
```

---

## Systematic Evaluation: CLI & Automation

### The Four-Step Evaluation Process

```
1. Create Evaluation Configuration → Define metrics/thresholds
2. Create Test Cases → Sample test cases to compare against
3. Run Agent with Test Query → Execute agent
4. Compare Results → Automated comparison
```

### Step 1: Evaluation Configuration (`test_config.json`)

Defines pass/fail thresholds:

```json
{
  "criteria": {
    "tool_trajectory_avg_score": 1.0,   // Perfect tool usage required
    "response_match_score": 0.8         // 80% text similarity threshold
  }
}
```

### Step 2: Test Cases (`*.evalset.json`)

Contains multiple test cases (sessions):

```json
{
  "eval_set_id": "home_automation_integration_suite",
  "eval_cases": [
    {
      "eval_id": "living_room_light_on",
      "conversation": [
        {
          "user_content": {
            "parts": [{"text": "Please turn on the floor lamp"}]
          },
          "final_response": {
            "parts": [{"text": "Successfully set the lamp to on."}]
          },
          "intermediate_data": {
            "tool_uses": [
              {
                "name": "set_device_status",
                "args": {
                  "location": "living room",
                  "device_id": "floor lamp",
                  "status": "ON"
                }
              }
            ]
          }
        }
      ]
    }
  ]
}
```

**Sources for Evalsets:**
- Created synthetically
- Exported from ADK Web UI conversations
- Generated from real user interactions

### Step 3: Running CLI Evaluation

```bash
adk eval agent_directory \
         evalset_file.evalset.json \
         --config_file_path=test_config.json \
         --print_detailed_results
```

**What Happens:**
1. CLI loads agent and configuration
2. Runs each test case in the evalset
3. Compares actual vs expected results
4. Calculates scores for each metric
5. Prints summary and detailed results

### Step 4: Analyzing Results

**Detailed output shows:**
- Per-test-case scores
- Pass/Fail status based on thresholds
- Side-by-side comparison of actual vs expected
- Turn-by-turn breakdown with diffs
- Root cause identification

---

## Advanced: User Simulation

### The Limitation of Static Tests

**Problem:** Fixed test cases don't capture:
- Dynamic conversation flow
- Unexpected user turns
- Context maintenance across turns
- Natural conversation unpredictability

### Solution: User Simulation

**How It Works:**
1. Define a `ConversationScenario` with user's goals
2. Create a `conversation_plan` to guide dialogue
3. LLM acts as simulated user
4. Generates realistic, varied prompts dynamically
5. Tests agent's adaptability and robustness

**Benefits:**
- Uncovers edge cases
- Tests context maintenance
- Validates complex multi-turn scenarios
- More realistic than static tests
- Improves agent robustness

---

## Key File Types in Evaluation

### 1. `test_config.json` (Optional)
- **Purpose:** Define evaluation criteria and thresholds
- **Location:** Agent root directory
- **Contains:** Metric names and pass/fail thresholds

### 2. `*.evalset.json` (Required)
- **Purpose:** Test cases with expected behavior
- **Location:** Agent directory
- **Contains:** Conversations with user input, expected responses, and tool usage

### 3. `*.test.json` (Pytest format)
- **Purpose:** Python-based test definitions
- **Location:** Test directory
- **Contains:** Pytest test functions

---

## Evaluation Best Practices

### Creating Effective Test Cases

1. **Cover Edge Cases:** Not just happy paths
   - Ambiguous commands
   - Invalid inputs
   - Complex multi-step requests
   - Error scenarios

2. **Test Different Dimensions:**
   - **Functionality:** Does it work?
   - **Quality:** Is the response good?
   - **Safety:** Does it avoid harmful actions?
   - **Efficiency:** Does it choose optimal tools?

3. **Real-World Scenarios:**
   - Use actual user conversations
   - Include common failure patterns
   - Test boundary conditions

### Setting Thresholds

**Response Match Score:**
- `1.0`: Exact match required (too strict for most cases)
- `0.8-0.9`: High similarity (recommended for most agents)
- `0.6-0.7`: Moderate similarity (more lenient)

**Tool Trajectory Score:**
- `1.0`: Perfect tool usage (recommended)
- `0.8-0.9`: Allows minor variations
- Lower values: Too permissive for critical applications

### Regression Testing Workflow

```
1. Make change to agent (prompt, tools, model)
   ↓
2. Run evaluation suite: `adk eval`
   ↓
3. Review results and scores
   ↓
4. If failures: Analyze diffs, fix issues
   ↓
5. Re-run evaluation
   ↓
6. If passes: Deploy with confidence
```

---

## Comparison: Observability vs Evaluation

| Aspect | Observability | Evaluation |
|--------|---------------|------------|
| **Timing** | Reactive (after failure) | Proactive (before deployment) |
| **Purpose** | Debug and understand | Test and validate |
| **Method** | Logs, traces, metrics | Test cases, scoring |
| **When** | Runtime, production | Development, pre-deployment |
| **Output** | Diagnostic data | Pass/Fail scores |
| **Tools** | LoggingPlugin, DEBUG logs | `adk eval`, test suites |

**Both Are Essential:**
- **Evaluation** catches issues before deployment
- **Observability** diagnoses issues in production
- Together they provide complete quality assurance

---

## Key Takeaways

### From Day 4a (Observability):
1. **Three Pillars:** Logs (what), Traces (why), Metrics (how well)
2. **Debugging Pattern:** Symptom → Logs → Root Cause → Fix
3. **Development:** Use `adk web --log_level DEBUG`
4. **Production:** Use `LoggingPlugin()` or custom plugins
5. **Plugins:** Callbacks that hook into agent lifecycle
6. **Register Once:** Plugin applies to all agents automatically

### From Day 4b (Evaluation):
1. **Proactive Approach:** Catch issues before production
2. **Dual Assessment:** Evaluate both response AND trajectory
3. **Two Metrics:** Response match (quality) + Tool trajectory (decisions)
4. **Interactive Creation:** Use ADK Web UI for test cases
5. **Automated Testing:** Use `adk eval` CLI for regression testing
6. **User Simulation:** Dynamic testing for realistic scenarios

---

## Decision Framework: Observability & Evaluation

### When to Use What?

**Development & Debugging:**
```
Problem: Agent not working as expected
Solution: adk web --log_level DEBUG
Result: See exact prompts, tool calls, responses
```

**Production Monitoring:**
```
Problem: Need visibility into production agent
Solution: LoggingPlugin() + custom plugins
Result: Automated logging without manual intervention
```

**Pre-Deployment Testing:**
```
Problem: Want to ensure agent quality before release
Solution: adk eval with test suite
Result: Automated pass/fail validation
```

**Regression Prevention:**
```
Problem: Changes might break existing functionality
Solution: Run evaluation suite after each change
Result: Catch regressions before deployment
```

**Edge Case Discovery:**
```
Problem: Need to test unpredictable scenarios
Solution: User Simulation
Result: Dynamic, realistic testing
```

---

## Common Evaluation Patterns

### Pattern 1: Basic Regression Suite

```python
# test_config.json
{
  "criteria": {
    "tool_trajectory_avg_score": 1.0,
    "response_match_score": 0.8
  }
}

# Run after every change
!adk eval my_agent my_agent/regression.evalset.json \
    --config_file_path=my_agent/test_config.json
```

### Pattern 2: Multi-Dimensional Testing

**Create separate evalsets for different dimensions:**
- `functionality.evalset.json` - Core features work
- `safety.evalset.json` - No harmful actions
- `edge_cases.evalset.json` - Unusual inputs
- `performance.evalset.json` - Speed and efficiency

### Pattern 3: Progressive Evaluation

```
1. Quick smoke tests (5 cases) - fast feedback
   ↓
2. Core functionality (20 cases) - main features
   ↓
3. Comprehensive suite (100+ cases) - full coverage
   ↓
4. User simulation - dynamic scenarios
```

---

## Debugging Workflow with Observability & Evaluation

### Comprehensive Quality Assurance Workflow

```
Phase 1: Development
├─ Build agent
├─ Test with adk web --log_level DEBUG
├─ Debug issues using Events tab
└─ Fix problems iteratively

Phase 2: Testing
├─ Create evaluation test cases
├─ Set appropriate thresholds
├─ Run: adk eval agent/ tests.evalset.json
└─ Analyze failures and iterate

Phase 3: Pre-Deployment
├─ Run full evaluation suite
├─ Check all metrics pass thresholds
├─ Review edge cases
└─ Get approval to deploy

Phase 4: Production
├─ Deploy with LoggingPlugin enabled
├─ Monitor logs and metrics
├─ Detect anomalies
└─ Debug with observability data

Phase 5: Continuous Improvement
├─ Collect real user interactions
├─ Add to evaluation suite
├─ Re-run evaluations regularly
└─ Catch regressions early
```

---

## Tools & Commands Reference

### ADK CLI Commands

```bash
# Start web UI with debug logging
adk web --log_level DEBUG

# Run evaluation suite
adk eval AGENT_DIR EVALSET_FILE \
    --config_file_path CONFIG_FILE \
    --print_detailed_results

# Create new agent
adk create agent_name --model MODEL_NAME
```

### Python API

```python
# Add logging plugin
from google.adk.plugins.logging_plugin import LoggingPlugin

runner = InMemoryRunner(
    agent=my_agent,
    plugins=[LoggingPlugin()]
)

# Custom plugin
from google.adk.plugins.base_plugin import BasePlugin

class MyPlugin(BasePlugin):
    async def before_agent_callback(self, *, agent, callback_context):
        # Custom logging logic
        pass
```

---

## Resources

### Observability
- [ADK Observability Documentation](https://google.github.io/adk-docs/observability/logging/)
- [Custom Plugins](https://google.github.io/adk-docs/plugins/)
- [External Integrations](https://google.github.io/adk-docs/observability/cloud-trace/)

### Evaluation
- [ADK Evaluation Overview](https://google.github.io/adk-docs/evaluate/)
- [Evaluation Criteria](https://google.github.io/adk-docs/evaluate/criteria/)
- [Pytest-based Evaluation](https://google.github.io/adk-docs/evaluate/#2-pytest-run-tests-programmatically)
- [User Simulation](https://google.github.io/adk-docs/evaluate/user-sim/)

---

## Next Steps

Day 5 will cover:
- Deploying agents to production
- Agent2Agent Protocol
- Scaling agent systems

---

*Created from notebooks by Sita Lakshmi Sangameswaran and Ivan Nardini for the Kaggle 5-day AI Agents course*

