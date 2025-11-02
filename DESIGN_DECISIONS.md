# Design Decisions & Open Questions

**Created:** 2025-10-18
**Last Updated:** 2025-10-18

This document tracks design decisions that need to be made before implementation. Each question will be answered and documented here as we progress.

---

## Status Legend
- 🔴 **Open** - Not yet decided
- 🟡 **In Discussion** - Currently being explored
- 🟢 **Decided** - Decision made and documented

---

## 1. Agent Communication Pattern 🟢

**Question:** How do agents share context with each other?

**Sub-questions:**
- Does each agent see the full conversation history or just relevant parts?
- How does the Evaluator get access to what Brainstormer produced?
- Do agents pass messages directly or through a shared state?

**Options:**
- A) Full conversation history shared with all agents
- B) Filtered/relevant context per agent role
- C) Explicit message passing between agents
- D) Shared state object that agents read/write to

**Decision:** **Hybrid: Shared State Object (D) + Filtered Context (B)**

**Implementation:**
```python
# Structured state for data
class SessionState(BaseModel):
    user_request: str
    plan: Optional[Plan] = None
    research_findings: List[Finding] = []
    generated_prompts: List[Prompt] = []
    evaluations: List[Evaluation] = []
    iteration: int = 0

# Each agent gets filtered conversation history
planner_context = [user_message]
researcher_context = [user_message, planner_response]
brainstormer_context = [user_message, research_summary]  # NOT full research process
evaluator_context = [user_message, generated_prompts]    # NOT brainstorming process
```

**Rationale:**
- **Structured data in shared state** - Easy to checkpoint, resume, and access specific outputs
- **Filtered conversation history** - Reduces token costs and prevents irrelevant context from distracting agents
- **Prevents bias** - Evaluator doesn't see Brainstormer's reasoning process, ensures objective scoring
- **Type-safe** - Pydantic models provide validation and clear schema
- **Efficient** - Only pass relevant context to each agent's LLM calls
- **Flexible** - Can adjust filtering strategy per agent as needed
- **Framework-friendly** - Works well with MS Agent Framework's state management

---

## 2. Scoring System Details 🟢

**Question:** How should solutions be scored?

**Sub-questions:**
- What's the scale? (1-10, 0-100, letter grades?)
- What criteria does the Evaluator use?
- Single score or multiple dimensions (creativity, feasibility, completeness)?
- Is scoring deterministic or LLM-based?

**Options:**
- A) Simple 1-10 single score (LLM-based)
- B) Multi-dimensional scoring (creativity: X, feasibility: Y, clarity: Z)
- C) Hybrid: LLM generates score + human can override
- D) Comparative ranking instead of absolute scores

**Decision:** **Multi-dimensional Scoring (B) with Human Override (C)**

**Implementation:**
```python
class Evaluation(BaseModel):
    prompt_id: str
    quality: float        # 0-10: General goodness
    clarity: float        # 0-10: Easy to understand/use
    specificity: float    # 0-10: Detailed enough
    overall: float        # Average of dimensions
    reasoning: str        # Explanation for each score
    human_override: Optional[float] = None  # User can override overall score

    @property
    def final_score(self) -> float:
        return self.human_override if self.human_override else self.overall

# Usage:
# Autonomous mode: Uses overall score, terminates at >= 9.0
# Interactive mode: User can override any score
```

**Scoring Dimensions (MVP):**
- **Quality** - General excellence, fit for purpose
- **Clarity** - Easy to understand and actionable
- **Specificity** - Sufficient detail and precision
- **Overall** - Average of the three dimensions

**Human Override:**
- In **interactive mode**: User can override any evaluation after seeing it
- In **autonomous mode**: Override disabled, uses LLM scores
- Overrides are saved for future learning (Phase 4 enhancement)

**Rationale:**
- **Multi-dimensional feedback** - Agents know exactly what to improve
- **Transparency** - User understands WHY a score was given
- **Iterative improvement** - Brainstormer can target weak dimensions
- **Human-in-the-loop** - User maintains control when desired
- **Flexible** - Can add use-case specific dimensions later
- **Data-rich** - Better for cost/benefit analysis and analytics
- **Autonomous-friendly** - Works without human when needed

---

## 3. CLI Commands & Interface 🟢

**Question:** What commands and interface will users interact with?

**Sub-questions:**
- What top-level commands? (e.g., `start`, `resume`, `list`, `export`)
- How are agent thoughts displayed during execution?
- Real-time streaming output or batch display after each step?
- How to show multiple agents working in parallel?

**Options:**
- A) Simple commands: `ideation start`, `ideation resume <session-id>`
- B) Rich subcommands: `ideation session new`, `ideation session resume`
- C) Interactive mode: Launch into interactive shell
- D) Combination: Support both CLI commands and interactive mode

**Decision:** **Interactive Shell Mode (C) - No CLI commands needed for MVP**

**Agent Pool Architecture:**
- **1 Planner** - Creates execution plan
- **1 Expert** - Clarifies requirements (conditional)
- **Multiple Researchers** (2-3) - Parallel research on different aspects
- **Multiple Brainstormers** (2-3) - Generate diverse solutions
- **1 Evaluator** - Scores all outputs

**Interface Design:**
```
┌─────────────────────────────────────────────────────────────────────┐
│ Ideation Agent - Session abc123                   Step 3/10 [●] Live│
│ Mode: Autonomous | Cost: $0.23 | Time: 5m 12s                       │
└─────────────────────────────────────────────────────────────────────┘

━━━━━━━━━━━━━━━ PROCESS HISTORY (Scrollable) ━━━━━━━━━━━━━━━━━━━━━━━

[USER] 3:45 PM
Generate prompts for modern villa architecture

[Planner] 3:45 PM | Cost: $0.03
Created execution plan:
  1. Expert clarification (style, requirements)
  2. Parallel research (3 researchers: trends, materials, composition)
  3. Brainstorm prompts (3 variations: minimal, detailed, atmospheric)
  4. Evaluate and refine
✓ Plan approved by user

[Expert] 3:46 PM | Cost: $0.05
Clarifying requirements...
Q: What architectural style? Modern minimalist or contemporary?
Q: Any specific materials to emphasize?
User response: "Minimalist with lots of glass and concrete"
✓ Requirements clarified

[Researcher-1: Trends] 3:47 PM | Cost: $0.04
Researching modern villa trends...
Found 5 key trends:
  - Large glass facades for natural light
  - Open-plan living spaces
  - Integration with landscape
  - Sustainable materials
  - Minimalist clean lines
✓ Research complete

[Researcher-2: Materials] 3:47 PM | Cost: $0.04  [Parallel]
Researching glass and concrete in architecture...
Best practices:
  - Floor-to-ceiling glass for transparency
  - Exposed concrete for brutalist aesthetic
  - Balance warmth with cold materials
✓ Research complete

[Researcher-3: Composition] 3:48 PM | Cost: $0.05  [Parallel]
Researching architectural photography composition...
Key principles:
  - Golden hour lighting preferred
  - Eye-level or slightly elevated angles
  - Emphasis on geometry and lines
✓ Research complete

━━━━━━━━━━━━━ NOW WORKING (Real-time) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━

▶ [Brainstormer-1: Minimal] 3:49 PM | Cost: $0.06 (in progress)
Generating minimalist prompt variation...
"Modern minimalist villa, floor-to-ceiling glass facade, exposed
concrete walls, clean geometric lines, integrated with hillside
landscape, golden hour lighting, architectural photography..."

▶ [Brainstormer-2: Detailed] 3:49 PM | Cost: $0.07 (in progress) [Parallel]
Generating detailed prompt variation...
"Ultra-modern luxury villa, 12-meter glass panels, raw concrete
structure, cantilevered design, infinity pool..."

▶ [Brainstormer-3: Atmospheric] 3:49 PM | Cost: $0.06 (in progress) [Parallel]
Generating atmospheric prompt variation...
"Serene modern villa at dusk, warm interior lighting through floor..."

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Cost Breakdown:
  Planner: $0.03 | Expert: $0.05 | Researchers: $0.13 | Brainstormers: $0.19
  Total: $0.40 (Claude Sonnet 4.5)

Active Agents: Brainstormer-1, Brainstormer-2, Brainstormer-3
Waiting: Evaluator

┌─────────────────────────────────────────────────────────────────────┐
│ [Enter] Skip  [S] Stop & feedback  [P] Pause  [Q] Quit & save      │
└─────────────────────────────────────────────────────────────────────┘
> _
```

**Key Features:**
1. **Process History (Scrollable)** - Full conversation log showing everything that happened
   - Timestamps for each agent response
   - Cost per message
   - Completed work clearly marked with ✓
   - Shows parallel execution with [Parallel] tag
2. **Multiple Specialized Agents**
   - 2-3 Researchers with different focuses (trends, materials, composition)
   - 2-3 Brainstormers generating different variations (minimal, detailed, atmospheric)
   - Each agent labeled with their specialty
3. **Real-time "Now Working" Section** - Live streaming of active agents
   - Shows all parallel agents currently executing
   - In-progress output streams in real-time
4. **Comprehensive Cost Monitor** - Per-agent and total cost tracking
   - Individual agent costs visible in history
   - Aggregated cost breakdown by agent type
   - Running total with LLM provider
5. **Always Interruptible** - Press 'S' at any time to stop and give feedback
6. **Status Bar** - Quick glance: step, cost, time, active agents

**User Interactions:**
- **Launch:** Simply run `ideation` → drops into interactive session
- **Start Session:** Prompted for topic, mode (interactive/autonomous), max iterations
- **During Execution:**
  - Press `S` to stop agents and give feedback
  - Press `Enter` to skip your turn (pass to agents)
  - Press `P` to pause (agents stop at current step)
  - Press `C` to continue if paused
  - Press `Q` to quit and save checkpoint
- **After Agent Response:**
  - View results in real-time
  - Override scores if in interactive mode
  - Provide feedback/direction

**Feedback Mechanism:**
```
[You pressed 'S' - Agents paused]

What would you like to do?
> 1. Give feedback/direction
> 2. Override last evaluation
> 3. Change mode (autonomous ↔ interactive)
> 4. Adjust max iterations
> 5. Continue

Your choice: 1

Enter feedback: "Focus more on sustainable materials and passive cooling"

[Feedback saved - Resuming with updated context]
```

**No CLI Commands for MVP:**
- Launch directly into interactive mode
- Session management (list/resume/export) can be added later
- Focus on the brainstorming experience first

**Rationale:**
- **User wanted interactive** - No need for CLI command complexity
- **Full process visibility** - Scrollable history shows entire journey from start
- **Multiple agents for diversity** - 2-3 researchers/brainstormers generate richer, more varied output
- **Parallel execution visible** - User sees multiple agents working simultaneously
- **Stop anytime** - 'S' key immediately pauses agents for feedback
- **Rich information display** - All relevant data visible: costs per message, timestamps, status, full conversation
- **Natural workflow** - Feels like watching a team collaborate, not commands
- **MVP-focused** - Simpler to implement than command infrastructure
- **Engaging** - Real-time updates keep user connected to agent work
- **Always in control** - Can interrupt, pause, or redirect at any moment
- **Transparency** - See what each specialized agent contributed

---

## 4. Configuration & Settings 🔴

**Question:** How should configuration be managed?

**Sub-questions:**
- Where are API keys stored? (.env file? config.yaml?)
- How to configure quality thresholds, max iterations, default modes?
- Where are agent personalities/system prompts defined?
- User-level vs. project-level configuration?

**Options:**
- A) `.env` for secrets, `config.yaml` for settings
- B) All in `.env` file
- C) Interactive first-time setup wizard
- D) Code-based config with optional file overrides

**Decision:** [To be decided]

**Rationale:** [To be filled]

---

## 5. Error Handling & Recovery 🔴

**Question:** How to handle errors and failures?

**Sub-questions:**
- What happens if LLM API fails mid-session?
- What if an agent gets stuck in a loop?
- How to detect and recover from infinite loops?
- Retry strategies for API failures?

**Options:**
- A) Fail fast: Stop session, save checkpoint, let user resume
- B) Auto-retry with exponential backoff
- C) Fallback to simpler agent behavior
- D) User prompt: "API failed, retry or abort?"

**Decision:** [To be decided]

**Rationale:** [To be filled]

---

## 6. Output Formats & Export 🔴

**Question:** How are results presented and exported?

**Sub-questions:**
- Final results format in CLI (table, list, panels?)
- Export formats (JSON, Markdown, PDF, HTML?)
- Include agent reasoning/thoughts in export?
- Session history format and storage?

**Options:**
- A) Rich CLI display + JSON export
- B) Rich CLI display + Markdown export
- C) Multiple export formats (JSON, MD, HTML)
- D) Customizable templates for export

**Decision:** [To be decided]

**Rationale:** [To be filled]

---

## 7. Parallel Agent Execution 🔴

**Question:** How do multiple agents run concurrently?

**Sub-questions:**
- True parallel execution (async) or sequential with batching?
- How does synchronization work (wait for all, first-to-finish, streaming)?
- Does user see both agents working simultaneously?
- How to handle dependencies between parallel agents?

**Options:**
- A) True async parallel execution with Python asyncio
- B) Sequential execution but present as parallel in UI
- C) Parallel where possible, sequential where needed
- D) User chooses mode (fast parallel vs. observable sequential)

**Decision:** [To be decided]

**Rationale:** [To be filled]

---

## 8. Plan Modification by User 🔴

**Question:** How can users modify the generated plan?

**Sub-questions:**
- Edit plan as text? Choose from alternative plans? Step-by-step approval?
- Can user add/remove steps from the plan?
- Can user reorder steps?
- Freeform text editing or structured modifications?

**Options:**
- A) Approve/Reject only (no modifications)
- B) Interactive step-by-step approval (approve/skip/modify each step)
- C) Text editor opens for freeform plan editing
- D) Structured prompts to add/remove/modify specific steps

**Decision:** [To be decided]

**Rationale:** [To be filled]

---

## 9. LLM Provider 🔴

**Question:** Which LLM provider to use?

**Sub-questions:**
- Claude (Anthropic) vs GPT (OpenAI)?
- Support multiple providers from day 1?
- Model selection (GPT-4, Claude Sonnet 4.5, etc.)?
- Cost vs. quality tradeoffs?

**Options:**
- A) Claude only (Sonnet 4.5 for best reasoning)
- B) GPT only (GPT-4 for broad compatibility)
- C) Support both, user chooses in config
- D) Different models for different agents (cheap for simple, expensive for complex)

**Recommendation from previous discussion:** Start with Claude for best reasoning

**Decision:** [To be decided]

**Rationale:** [To be filled]

---

## 10. Deployment Model 🔴

**Question:** How will the application be deployed?

**Sub-questions:**
- Local CLI only or client-server architecture?
- Single-user or multi-user?
- Where is state stored (local files, database, cloud)?

**Options:**
- A) Local CLI application, all state local (MVP approach)
- B) Local CLI + optional cloud sync
- C) Client-server architecture (CLI client, API server)
- D) Web app + API (future evolution)

**Recommendation from previous discussion:** MVP = Local CLI application

**Decision:** [To be decided]

**Rationale:** [To be filled]

---

## 11. Configuration Management 🔴

**Question:** YAML files vs. Interactive setup vs. Code-based?

**Sub-questions:**
- How do users customize agent personas?
- Are prompt templates in code or config files?
- Version control for configurations?

**Options:**
- A) Code-based config (simple, version controlled)
- B) YAML files (flexible, non-technical users can edit)
- C) Interactive first-time setup wizard → generates config
- D) Hybrid: Defaults in code, overrides in YAML

**Recommendation from previous discussion:** Start with code-based config, add YAML when needed

**Decision:** [To be decided]

**Rationale:** [To be filled]

---

## 12. Prompt Templates 🔴

**Question:** Should prompt templates be hardcoded or user-customizable?

**Sub-questions:**
- How are agent system prompts defined?
- Can users customize agent personalities?
- Versioning of prompts?

**Options:**
- A) Hardcoded in code (simple, optimized)
- B) External files (YAML/JSON) (flexible)
- C) Database stored (dynamic updates)
- D) Hybrid: Defaults in code, user overrides in files

**Recommendation from previous discussion:** MVP = Hardcoded optimized prompts, Future = Allow customization

**Decision:** [To be decided]

**Rationale:** [To be filled]

---

## 13. Cost Tracking & Monitoring 🔴

**Question:** How should LLM API costs be tracked and displayed?

**Sub-questions:**
- Track costs per message, per agent, per session, or all of the above?
- Real-time cost display or summary at the end?
- How to handle different LLM providers with different pricing?
- Store cost history for analysis?
- Set budget limits or warnings?

**Options:**
- A) Simple: Total session cost displayed at the end
- B) Detailed: Per-agent cost breakdown shown in real-time
- C) Comprehensive: Per-message tracking with running total + budget warnings
- D) Analytics: Full cost history stored in DB with reports and trends

**Display Options:**
- Show running cost in CLI footer/status bar
- Cost summary after each agent response
- Detailed cost report at session end
- Separate `ideation costs` command to view history

**Tracking Granularity:**
```
Session Level:
  └─ "Total cost: $0.45"

Agent Level:
  ├─ Planner: $0.05
  ├─ Researcher: $0.15
  ├─ Brainstormer: $0.20
  └─ Evaluator: $0.05

Message Level:
  ├─ Message 1 (Planner): $0.02 (input: 500 tokens, output: 300 tokens)
  ├─ Message 2 (Researcher): $0.08 (input: 1200 tokens, output: 800 tokens)
  └─ ...

LLM Provider Breakdown:
  ├─ Claude (Sonnet 4.5): $0.30
  └─ GPT-4: $0.15
```

**Implementation Considerations:**
- Need pricing tables for each provider/model
- Track input tokens vs output tokens (different pricing)
- Update pricing when providers change rates
- Exchange rate handling for non-USD pricing?

**Decision:** [To be decided]

**Rationale:** [To be filled]

---

## Decision Process

When answering each question, we'll follow this process:

1. **Discuss options** - Explore pros/cons of each approach
2. **Consider MVP vs. Future** - What's needed now vs. later?
3. **Make decision** - Choose an option
4. **Document rationale** - Why this choice?
5. **Update status** - 🔴 → 🟡 → 🟢
6. **Update PROJECT_SPEC.md** - Add concrete details to main spec

---

## Notes

- Decisions should balance MVP speed with future flexibility
- Prefer simple solutions that can evolve over complex premature optimization
- Document tradeoffs and alternatives considered
- Revisit decisions as we learn more during implementation
