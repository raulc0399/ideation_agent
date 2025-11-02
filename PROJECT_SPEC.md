# Ideation Agent - Project Specification

**Created:** 2025-10-18
**Last Updated:** 2025-10-18

---

## Project Overview

An interactive CLI application for AI-powered brainstorming and research using multi-agent collaboration. The system helps users explore ideas, research topics, and refine solutions through autonomous agent collaboration with human-in-the-loop control.

---

## Core Use Cases

1. **Image Generation Prompts** - Generate and refine prompts for architecture visualization
   - Agents research best practices for prompt engineering
   - Iterative refinement of prompt variations
   - Quality scoring and selection

2. **Pipeline Design** - Design technical pipelines/workflows
   - Research existing solutions and patterns
   - Generate multiple architectural proposals
   - Compare and refine approaches

3. **Business Ideation** - Explore business ideas in specific domains
   - Market research and competitive analysis
   - Idea generation and validation
   - Feasibility assessment

4. **Generate new diffusion models architecture and training methods** - Research and propose novel architectures for diffusion models
   - Survey latest research papers
   - Brainstorm innovative architectural changes
   - Evaluate proposed designs

---

## Key Features

### 1. Multi-Agent System
- **Agent Types:**
  - **Planner** (1) - Generates execution plans for user approval
  - **Expert** (1) - Clarifies requirements and problem details (optional step)
  - **Researchers** (2-3) - Parallel research on different aspects (trends, materials, techniques)
  - **Brainstormers** (2-3) - Generate diverse solution variations (minimal, detailed, atmospheric)
  - **Evaluator** (1) - Scores all solutions for quality assessment
- **Parallel Execution** - Multiple researchers and brainstormers work simultaneously for diversity and speed

### 2. Workflow Control
- **Plan Approval** - Agent generates plan → user reviews → execution begins
- **Expert Clarification** - Optional phase when problem needs refinement
- **Iteration Control:**
  - User can choose to run agents for N iterations
  - User can run just 1 step at a time
  - User can skip their turn (hit Enter to pass)
  - Agents can run autonomously for specified iterations
- **Mode Switching:**
  - Brainstorm mode
  - Research mode
  - Combined mode (activated dynamically)

### 3. Quality-Based Termination
- Evaluator scores solutions continuously
- Auto-stop when quality threshold is reached
- Maximum iteration limit (e.g., 10 steps)
- Early termination if excellent solution found

### 4. Interactive Interface
- **Full process visibility** - Scrollable history showing complete conversation from start
- **Real-time agent streaming** - Watch multiple agents work in parallel
- **Stop anytime** - Interrupt agents at any moment to provide feedback
- Approve/reject generated plans
- Control agent execution (1 step vs N steps)
- Switch modes during execution
- Progress tracking with timestamps and completion status
- See individual agent specialties and contributions

### 5. Cost Tracking & Monitoring
- Track LLM API costs per message
- Display costs per agent and per LLM provider
- Real-time cost monitoring during session
- Session-level cost summaries
- Historical cost analysis
- Budget warnings and limits

---

## Technology Stack

### Core Framework
- **Agent Framework:** Microsoft Agent Framework (Python)
  - Graph-based workflow orchestration
  - Native human-in-the-loop support
  - Built-in checkpointing and state management
  - Multi-agent collaboration patterns

### CLI/UI Libraries
- **typer** - CLI framework for commands and arguments
- **rich** - Beautiful terminal output (tables, progress bars, syntax highlighting)
- **prompt_toolkit** - Interactive prompts and input validation

### LLM Integration
- **anthropic, openai, gemini** - for each agent, options to choose backend LLMs
- **pydantic** - Data validation and settings management

### Memory & Persistence
- **SQLite** - Session persistence and chat history
- **sqlalchemy** - Database ORM
- Future: Vector DB (Qdrant) for long-term semantic memory

### Optional/Future
- **qdrant-client** - Vector database for long-term memory
- **sentence-transformers** - Embeddings for semantic search
- **pyyaml** - Configuration file management

---

## Memory Architecture

### 1. Short-term Memory (Conversation History)
- **Implementation:** Custom SQLite-based ChatMessageStore
- **Purpose:** Persist conversation history across sessions
- **Storage:** Local SQLite database
- **Scope:** Current and past sessions

### 2. Long-term Memory (Future Enhancement)
- **Implementation:** Vector DB (Qdrant) with AIContextProvider
- **Purpose:** Semantic search of past insights and learnings
- **Storage:** Vector embeddings of key insights
- **Scope:** Cross-session knowledge base

### 3. Workflow State (Checkpointing)
- **Implementation:** Built-in CheckpointManager
- **Purpose:** Resume interrupted workflows
- **Storage:** Checkpoint files on disk
- **Scope:** Long-running brainstorming sessions

---

## Architecture

```
┌─────────────────────────────────────┐
│  CLI Layer (typer + rich)           │
│  - Commands, menus, display         │
│  - User input and approval prompts  │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  Orchestrator (Workflow Manager)    │
│  - Plan approval workflow           │
│  - Iteration management             │
│  - Mode switching                   │
│  - Quality threshold monitoring     │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  Agent Pool (MS Agent Framework)    │
│  ├── Planner Agent                  │
│  ├── Expert Agent (conditional)     │
│  ├── Researcher Agent               │
│  ├── Brainstormer Agent             │
│  └── Evaluator Agent                │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  Memory Layer                       │
│  ├── SQLite (chat history)          │
│  ├── Checkpoints (workflow state)   │
│  └── Vector DB (future)             │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  LLM Backend (Claude/GPT)           │
└─────────────────────────────────────┘
```

---

## Workflow Example

### Scenario: "Generate architecture prompts for modern villa"

```
1. User Input
   └─> User: "Generate prompts for modern villa architecture"

2. Planning Phase
   └─> Planner Agent generates execution plan
   └─> Display plan to user
   └─> User approves/modifies plan

3. Expert Clarification (Optional)
   └─> Expert Agent: "What architectural style? Minimalist? Mediterranean?"
   └─> User provides clarification
   └─> Expert refines problem statement

4. Research & Brainstorming (Parallel)
   ├─> Researcher: Finds best practices for architecture prompts
   │   - Discovers importance of materials, lighting, composition
   │   - Finds example prompts from successful renders
   └─> Brainstormer: Generates initial prompt variations
       - Creates 5 different prompt approaches
       - Varies focus (materials vs. composition vs. atmosphere)

5. Evaluation Loop
   ├─> Evaluator scores each variation (1-10)
   ├─> If best score >= 9: Stop (success!)
   └─> If best score < 9 && iterations < 10:
       └─> Brainstormer refines top 2 variants
       └─> Repeat evaluation

6. User Control Points
   ├─> After each iteration: Show results
   ├─> User can: Continue, Skip turn, Adjust iterations, Stop
   └─> User can switch modes (more research vs. more brainstorming)

7. Results
   └─> Display top 3 prompts with scores
   └─> Save to session history
   └─> Store learnings in long-term memory (future)
```

---

## Technical Decisions

### Framework Choice: Microsoft Agent Framework
**Reasons:**
- ✅ Graph-based workflows for dynamic, conditional flows
- ✅ Built-in human-in-the-loop (plan approval, iteration control)
- ✅ Checkpointing for long-running sessions
- ✅ Native state management
- ✅ Active development and Microsoft backing

**Alternatives Considered:**
- Claude Agent SDK - Excellent for subagent patterns, but locked to Claude
- LangGraph - Very mature, but more boilerplate
- AutoGen - Now in maintenance mode
- CrewAI - Too rigid for our dynamic workflows
- Custom - Too much development overhead

### Memory Strategy: SQLite First
**Reasons:**
- ✅ Simple setup, no external dependencies
- ✅ Sufficient for MVP and medium-term usage
- ✅ Easy migration path to PostgreSQL if needed
- ✅ Built-in to Python, zero installation

**Future Enhancement:** Add Qdrant vector DB for semantic memory

### UI Choice: Polished Interactive CLI
**Not a Full TUI:**
- No need for full-screen widgets (like htop/vim)
- Interactive prompts + rich formatting is sufficient
- Keeps complexity low while providing great UX

### A2A Protocol: Not Needed
**Reason:** All agents live in single application process
- No external agent communication required
- Framework handles internal agent routing
- Can add later if we build agent marketplace/services

---

## Development Phases

### Phase 1: MVP (Core Functionality)
- [ ] Basic agent setup (Planner, Researcher, Brainstormer, Evaluator)
- [ ] Simple workflow orchestration
- [ ] In-memory chat history
- [ ] CLI with typer + rich
- [ ] Single use case: Image prompt generation

### Phase 2: Session Persistence
- [ ] SQLite chat history storage
- [ ] Checkpoint/resume functionality
- [ ] Multi-session support
- [ ] Enhanced CLI controls (iteration management)
- [ ] Basic cost tracking (per-session total)

### Phase 3: Full Feature Set
- [ ] Expert agent with conditional activation
- [ ] Quality-based auto-termination
- [ ] Mode switching (brainstorm/research/combined)
- [ ] All three use cases implemented
- [ ] Advanced progress visualization
- [ ] Detailed cost tracking (per-agent, per-message, per-LLM)
- [ ] Real-time cost display and budget warnings

### Phase 4: Long-term Learning
- [ ] Vector DB integration (Qdrant)
- [ ] Semantic memory retrieval
- [ ] Cross-session learning
- [ ] Quality improvements over time
- [ ] Cost analytics and historical trends

---

## Success Metrics

- User can complete a brainstorming session in < 30 minutes
- Agents produce quality score >= 8/10 within 5 iterations
- Session resumes work 100% reliably
- Users find results more creative than solo brainstorming
- Zero manual intervention needed after plan approval in autonomous mode
  - User approves plan once, agents run independently until completion
  - System auto-terminates when quality threshold met or max iterations reached
  - No additional user input required during execution
- Cost tracking provides accurate per-message, per-agent, and per-session reporting
- Users can set and monitor budget limits effectively

---

## References

- [Microsoft Agent Framework Docs](https://learn.microsoft.com/en-us/agent-framework/)
- [Agent Memory Guide](https://learn.microsoft.com/en-us/agent-framework/user-guide/agents/agent-memory)
- [Microsoft Agent Framework GitHub](https://github.com/microsoft/agent-framework)
