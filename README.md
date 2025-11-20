# Google 5-Day Agent Intensive

A comprehensive learning repository for building production-ready AI agents using Google's Agent Development Kit (ADK). This project contains materials, notebooks, and resources from the Kaggle 5-Day AI Agents course.

## 📚 About This Course

This intensive 5-day course teaches you how to build, test, and deploy intelligent AI agents that go beyond simple LLM interactions. Learn to create agents that can think, take actions, observe results, and work together in sophisticated multi-agent systems.

### What You'll Learn

- **Day 1**: Agent fundamentals, multi-agent architectures, and workflow patterns
- **Day 2**: Custom tools, function integration, and Model Context Protocol (MCP)
- **Day 3**: Session management, memory systems, and state sharing
- **Day 4**: Observability, debugging, evaluation, and quality assurance
- **Day 5**: Agent-to-Agent (A2A) communication and production deployment

## 🚀 Quick Start

### Prerequisites

- Python 3.11
- Conda or Miniconda installed
- Google AI Studio API Key ([Get one here](https://aistudio.google.com/app/api-keys))

### Installation

1. **Clone this repository** (if you haven't already)
   ```bash
   cd google_5day_agent_intensive
   ```

2. **Create and activate the conda environment**
   ```bash
   conda env create -f environment.yml
   conda activate google-agents
   ```

3. **Configure your API key**
   ```bash
   # Copy the template
   cp env.template .env
   
   # Edit .env and replace with your actual API key
   # GOOGLE_API_KEY=your_actual_api_key_here
   ```

4. **Launch Jupyter Notebook**
   ```bash
   jupyter notebook
   ```

5. **Start with Day 1a**
   - Navigate to `notebooks/day-1a-from-prompt-to-action/`
   - Open the notebook and begin learning!

## 📖 Course Structure

### Day 1: Building Your First AI Agent
- **Day 1a**: From Prompt to Action
  - Introduction to AI agents vs traditional LLMs
  - Agent Development Kit (ADK) fundamentals
  - Creating your first agent with tools
  - Using Google Search integration
  
- **Day 1b**: Multi-Agent Architectures
  - LLM-based orchestration
  - Sequential agents (assembly line pattern)
  - Parallel agents (concurrent execution)
  - Loop agents (iterative refinement)

### Day 2: Advanced Tools & Interoperability
- **Day 2a**: Agent Tools
  - Custom function tools
  - Built-in tools (Google Search, Code Execution)
  - Tool design best practices
  
- **Day 2b**: MCP Integration
  - Model Context Protocol
  - Connecting external data sources
  - Tool interoperability

### Day 3: Context Engineering
- **Day 3a**: Agent Sessions
  - Session management
  - Conversation state handling
  - Multi-turn interactions
  
- **Day 3b**: Agent Memory
  - Short-term memory (session-based)
  - Long-term memory (persistent)
  - State sharing between agents

### Day 4: Agent Quality
- **Day 4a**: Agent Observability
  - Logging, tracing, and metrics
  - Debugging with ADK Web UI
  - Plugins and callbacks
  
- **Day 4b**: Agent Evaluation
  - Testing strategies
  - Evaluation frameworks
  - Quality metrics
  - Regression prevention

### Day 5: Prototype to Production
- **Day 5a**: Agent2Agent (A2A) Communication
  - A2A protocol fundamentals
  - Exposing agents with `to_a2a()`
  - Consuming remote agents with `RemoteA2aAgent`
  - Cross-organization, cross-language integration
  
- **Day 5b**: Agent Deployment
  - Deploying to Vertex AI Agent Engine
  - Production best practices
  - Memory Bank for long-term persistence
  - Cost management and monitoring

## 📁 Project Structure

```
google_5day_agent_intensive/
├── README.md                                          # This file
├── SETUP.md                                          # Detailed setup instructions
├── environment.yml                                   # Conda environment specification
├── env.template                                      # Template for API keys
├── .env                                              # Your API keys (create this, not in git)
│
├── notebooks/                                        # Interactive course notebooks
│   ├── day-1a-from-prompt-to-action/
│   ├── day-1b-agent-architectures/
│   ├── day-2a-agent-tools/
│   ├── day-2b-agent-tools-best-practices/
│   ├── day-3a-agent-sessions/
│   ├── day-3b-agent-memory/
│   ├── day-4a-agent-observability/
│   ├── day-4b-agent-evaluation/
│   ├── day-5a-A2A-communications/
│   └── day-5b-agent-deployment/
│
├── day1_summary.md                                   # Day 1 concepts summary
├── day2_summary.md                                   # Day 2 concepts summary
├── day3_summary.md                                   # Day 3 concepts summary
├── day4_summary.md                                   # Day 4 concepts summary
├── day5_summary.md                                   # Day 5 concepts summary
│
├── 1_introduction_to_agents_whitepaper.pdf          # Course whitepapers
├── 2_agent_tools_interoperability_with_mcp.pdf
├── 3_Context Engineering_ Sessions & Memory.pdf
├── 4_Agent Quality.pdf
├── 5_Prototype to Production.pdf
│
└── daily_podcast.txt                                 # Additional course notes
```

## 🛠️ Core Technologies

- **Google ADK (Agent Development Kit)**: Open-source framework for building and deploying AI agents
- **Gemini Models**: Google's state-of-the-art language models (gemini-2.5-flash-lite)
- **Vertex AI**: Production deployment platform for AI agents
- **A2A Protocol**: Standard for agent-to-agent communication
- **Python 3.11**: Programming language
- **Jupyter Notebooks**: Interactive development environment

## 🔑 Key Concepts

### What is an AI Agent?

**Traditional LLM:**
```
Prompt → LLM → Text Response
```

**AI Agent:**
```
Prompt → Agent → Thought → Action → Observation → Final Answer
```

Agents can:
- ✅ Take actions using tools
- ✅ Observe and learn from results
- ✅ Make decisions dynamically
- ✅ Work in teams with other agents
- ✅ Remember context across sessions

### Multi-Agent Patterns

| Pattern | Use Case | Key Feature |
|---------|----------|-------------|
| **Sequential** | Linear pipelines | Guaranteed order execution |
| **Parallel** | Independent tasks | Concurrent execution |
| **Loop** | Iterative refinement | Repeated improvement cycles |
| **LLM-based** | Dynamic workflows | LLM decides execution order |

### A2A Communication

Agent-to-Agent protocol enables:
- 🌐 Network-based agent communication
- 🔗 Cross-framework integration (ADK, LangChain, etc.)
- 🌍 Cross-language support (Python, Java, Go)
- 🏢 Cross-organization collaboration

## 📊 Environment Dependencies

Key packages installed via conda:

```yaml
- python=3.11
- jupyter>=1.0.0
- ipykernel>=6.0.0
- notebook>=7.0.0
- google-adk[eval,a2a]>=0.1.0
- python-dotenv>=1.0.0
```

## 🔧 Troubleshooting

### "GOOGLE_API_KEY not found" Error
- Ensure you created the `.env` file in the project root
- Check that your API key is correctly formatted (no quotes, no spaces)
- Verify the `.env` file location: `/home/chrisfkh/google_5day_agent_intensive/.env`

### Import Errors
```bash
# Ensure environment is activated
conda activate google-agents

# If issues persist, recreate environment
conda env remove -n google-agents
conda env create -f environment.yml
```

### API Key Issues
- Verify your API key at [Google AI Studio](https://aistudio.google.com/app/api-keys)
- Check for usage limits or billing issues
- Ensure `GOOGLE_GENAI_USE_VERTEXAI=FALSE` in `.env` for AI Studio API

## 📚 Additional Resources

### Official Documentation
- [Google ADK Documentation](https://google.github.io/adk-docs/)
- [A2A Protocol Specification](https://a2a-protocol.org/)
- [Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview)
- [Gemini API Documentation](https://ai.google.dev/docs)

### Course Materials
- [Kaggle 5-Day AI Agents Course](https://www.kaggle.com/learn-guide/5-day-genai)
- [Kaggle Discord Community](https://discord.com/invite/kaggle)

### GitHub Resources
- [Google ADK GitHub](https://github.com/google/adk)
- [Generative AI Examples](https://github.com/GoogleCloudPlatform/generative-ai)

## 🎯 Learning Path

Follow this recommended sequence:

1. **Week 1**: Complete Days 1-2
   - Master agent fundamentals
   - Build your first multi-agent system
   - Create custom tools

2. **Week 2**: Complete Days 3-4
   - Implement session management
   - Add memory systems
   - Set up observability
   - Create evaluation tests

3. **Week 3**: Complete Day 5
   - Integrate A2A communication
   - Deploy to production
   - Implement monitoring

4. **Week 4**: Build Your Own Project
   - Apply learned concepts
   - Create a custom agent system
   - Share with the community

## 🤝 Contributing

This is a personal learning repository, but feel free to:
- Report issues with setup instructions
- Suggest improvements
- Share your own learning notes
- Connect on learning progress

## 📝 License

This repository contains educational materials from the Kaggle 5-Day AI Agents course. Please respect the original course creators' intellectual property.

## 🙏 Acknowledgments

- **Kaggle** for hosting the 5-Day AI Agents course
- **Google ADK Team** for creating excellent developer tools
- **Course Instructors**: Kristopher Overholt, Lavi Nigam, and the Google team
- **Community Contributors** who share knowledge and support learners

## 🚦 Next Steps

Ready to begin? Start here:

```bash
# Activate environment
conda activate google-agents

# Launch Jupyter
jupyter notebook

# Open: notebooks/day-1a-from-prompt-to-action/day-1a-from-prompt-to-action.ipynb
```

Happy learning! 🎉

---

**Note**: Remember to never commit your `.env` file with real API keys to version control. The `.gitignore` file is configured to protect your credentials.

