# Agentic AI — Comprehensive Reference Card

> Version 1.0 · June 2026 · Covers design, components, architectures, memory, tooling, workflows, multi-agent patterns, MCP, evaluation, and safety

---

## Table of Contents

1. [Core Concepts & Definitions](#1-core-concepts--definitions)
2. [Agent Anatomy — Components](#2-agent-anatomy--components)
3. [Memory Systems](#3-memory-systems)
4. [Tool & Action Design](#4-tool--action-design)
5. [Planning Strategies](#5-planning-strategies)
6. [Agent Architectures](#6-agent-architectures)
7. [Multi-Agent Patterns](#7-multi-agent-patterns)
8. [Workflow Structures](#8-workflow-structures)
9. [Context & State Management](#9-context--state-management)
10. [Model Context Protocol (MCP)](#10-model-context-protocol-mcp)
11. [Prompting Patterns for Agents](#11-prompting-patterns-for-agents)
12. [Evaluation & Observability](#12-evaluation--observability)
13. [Safety, Security & Human Oversight](#13-safety-security--human-oversight)
14. [Framework Ecosystem](#14-framework-ecosystem)
15. [Design Principles & Anti-Patterns](#15-design-principles--anti-patterns)
16. [Quick Decision Matrix](#16-quick-decision-matrix)

---

## 1. Core Concepts & Definitions

### What is an Agent?

An **AI agent** is an LLM-powered system that perceives an environment, reasons about it, decides on actions, executes those actions via tools, observes the results, and iterates — all toward accomplishing a goal, with minimal or no human intervention per step.

```
Perceive → Reason → Plan → Act → Observe → [repeat until done]
```

### Agent vs. Standard LLM Call

| Dimension          | Standard LLM Call          | Agentic System                          |
|--------------------|----------------------------|-----------------------------------------|
| Control flow       | Single request → response  | Dynamic, multi-step loops               |
| State              | Stateless                  | Maintains state across turns            |
| Tool use           | None                       | Reads/writes external systems           |
| Planning           | Implicit in prompt         | Explicit, decomposed sub-tasks          |
| Autonomy           | Zero                       | Partial to full                         |
| Latency            | Seconds                    | Seconds to minutes                      |
| Error recovery     | None                       | Self-corrects via feedback loops        |
| Scope of action    | Text output only           | Real-world side effects possible        |

### Fundamental Properties of Agents

| Property         | Description                                                                 |
|------------------|-----------------------------------------------------------------------------|
| **Autonomy**     | Acts without step-by-step human direction                                   |
| **Reactivity**   | Responds to changes in its environment or new information                   |
| **Proactivity**  | Takes initiative to pursue goals, not just reacts                           |
| **Social ability** | Coordinates with other agents or humans                                   |
| **Goal-directedness** | Has explicit objectives it works toward                               |
| **Persistence**  | Continues across sessions via memory                                        |

### Levels of Agency (Spectrum)

```
Level 0  ──  Single-shot prompting
Level 1  ──  Chain of Thought (internal reasoning, no external actions)
Level 2  ──  Tool-augmented LLM (one call, multiple tools)
Level 3  ──  ReAct loop (iterative reason-act-observe)
Level 4  ──  Autonomous agent with memory and planning
Level 5  ──  Multi-agent system with coordination protocols
Level 6  ──  Self-improving / meta-learning agent systems
```

---

## 2. Agent Anatomy — Components

### The Five Core Components

```
┌──────────────────────────────────────────────────────────┐
│                        AGENT CORE                        │
│                                                          │
│  ┌───────────┐   ┌───────────┐   ┌───────────────────┐  │
│  │  BRAIN    │   │  MEMORY   │   │      TOOLS        │  │
│  │  (LLM)   │◄──┤  (State)  │   │  (Action Space)   │  │
│  │           │   │           │   │                   │  │
│  └─────┬─────┘   └───────────┘   └─────────┬─────────┘  │
│        │                                   │             │
│  ┌─────▼─────────────────────────────────▼─────────┐    │
│  │                  ORCHESTRATOR                     │    │
│  │      (Planning · Decision-making · Routing)       │    │
│  └───────────────────────┬───────────────────────────┘    │
│                          │                               │
│  ┌───────────────────────▼───────────────────────────┐   │
│  │               PERCEPTION / INPUT                   │   │
│  │   (Text, Images, Documents, Tool results, APIs)    │   │
│  └───────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

### Component Breakdown

#### 2.1 Brain (LLM)

The cognitive core — reasons, plans, generates text, decides tool calls.

| Aspect               | Detail                                                    |
|----------------------|-----------------------------------------------------------|
| Role                 | Central reasoning engine                                  |
| Input                | Prompt + context (memory + tool results + history)        |
| Output               | Text, structured JSON, tool call instructions             |
| Temperature          | Lower (0.0–0.3) for planning/tool use; higher for creativity |
| Context window       | Determines how much history/memory fits in one call       |
| Model selection      | Frontier (GPT-4o, Claude 3.5) for complex reasoning; smaller for sub-tasks |

#### 2.2 Memory

See Section 3 for full detail.

#### 2.3 Tools

See Section 4 for full detail.

#### 2.4 Orchestrator / Controller

Coordinates the agent loop:
- Determines when to invoke tools vs. return final answer
- Manages sub-task decomposition
- Routes between specialized agents in multi-agent systems
- Handles retries, errors, and loop termination

#### 2.5 Perception / Input Processing

| Input Type           | Handling                                              |
|----------------------|-------------------------------------------------------|
| Raw text             | Direct injection into context                         |
| Structured data      | JSON/CSV → summarize or embed                         |
| Images               | Vision-capable LLM or image description tool          |
| Documents (PDF)      | Chunking + retrieval (RAG)                            |
| Audio                | Transcription → text                                  |
| Tool results         | Serialized and appended to context as observations    |
| Environment state    | Serialized world state (for simulation agents)        |

---

## 3. Memory Systems

### Memory Taxonomy

```
Memory
├── In-Context (Working Memory)
│   ├── Conversation history
│   ├── System prompt
│   ├── Current task state
│   └── Immediate tool results
│
├── External Memory
│   ├── Semantic (Vector Store)        ← long-term knowledge retrieval
│   ├── Episodic (Event Log/DB)        ← past experiences, session history
│   ├── Procedural (Tool/Skill Store)  ← how-to knowledge, saved plans
│   └── Declarative (KV / Graph DB)    ← structured facts, entities
│
└── Parametric Memory
    └── Weights of the LLM itself      ← implicit knowledge from training
```

### Memory Types in Detail

| Type           | Storage              | Retrieval           | Use Case                             |
|----------------|----------------------|---------------------|--------------------------------------|
| **Working**    | Context window       | Always present      | Current task, recent messages        |
| **Semantic**   | Vector DB (Chroma, Pinecone, Qdrant) | Similarity search | Domain knowledge, RAG         |
| **Episodic**   | Relational / NoSQL DB | Lookup by session/time | Past interactions, user history  |
| **Procedural** | File / DB            | Fuzzy search / keyword | Saved workflows, successful plans |
| **Declarative** | KV store / Graph DB | Exact key / graph traversal | Entities, facts, relationships |
| **Parametric** | Model weights        | Implicit in inference | Broad world knowledge             |

### Memory Operations

```python
# Canonical memory interface
class AgentMemory:
    def store(self, key: str, value: Any, memory_type: str) -> None: ...
    def retrieve(self, query: str, k: int = 5) -> list[MemoryItem]: ...
    def forget(self, key: str) -> None: ...
    def summarize(self, window: list[Message]) -> str: ...
    def compress(self) -> None:  # Evict/summarize old entries
```

### Context Window Management Strategies

| Strategy             | Description                                               | When to Use              |
|----------------------|-----------------------------------------------------------|--------------------------|
| **Sliding window**   | Keep last N messages, drop oldest                         | Simple chatbots           |
| **Summarization**    | LLM summarizes old turns into compressed memory           | Long-running agents       |
| **RAG injection**    | Retrieve only relevant memories by similarity             | Large knowledge bases     |
| **Hierarchical**     | Recent: full detail · Old: summary · Very old: vector     | Production agents         |
| **Token budget**     | Reserve X tokens for each component (system/history/tools/output) | Any agent         |

### Token Budget Template

```
Total context: 128k tokens
├── System prompt:          ~2k
├── Long-term memory (RAG): ~8k
├── Tool definitions:       ~4k
├── Conversation history:   ~16k
├── Current task + state:   ~4k
└── Reserved for output:    ~4k
────────────────────────────────
Actual usage headroom:      ~90k
```

---

## 4. Tool & Action Design

### Tool Taxonomy

```
Tools
├── Read / Retrieval
│   ├── Web search
│   ├── Document retrieval (RAG)
│   ├── Database query
│   ├── API GET
│   └── File read
│
├── Write / Execute
│   ├── Code execution (Python, bash)
│   ├── API POST/PUT/DELETE
│   ├── File write / create
│   ├── Database write
│   └── Email / calendar / messaging
│
├── Process / Transform
│   ├── Data transformation
│   ├── Image processing
│   ├── Audio transcription
│   └── Document parsing
│
└── Meta / Agent
    ├── Spawn sub-agent
    ├── Delegate to specialist
    ├── Human escalation
    └── Wait / schedule
```

### Tool Definition Schema (OpenAI/Anthropic style)

```json
{
  "name": "search_knowledge_base",
  "description": "Search internal documents for relevant information. Use when the user asks about company policies, product specs, or historical data. Do NOT use for real-time information.",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Natural language search query. Be specific."
      },
      "top_k": {
        "type": "integer",
        "description": "Number of results to return (default: 5, max: 20)",
        "default": 5
      },
      "filter_date_after": {
        "type": "string",
        "format": "date",
        "description": "Only return documents created after this date (ISO 8601)"
      }
    },
    "required": ["query"]
  }
}
```

### Tool Design Principles

| Principle              | Description                                               |
|------------------------|-----------------------------------------------------------|
| **Single responsibility** | Each tool does one thing well                         |
| **Idempotency**        | Read tools must be safe to call multiple times            |
| **Explicit side effects** | Name write tools clearly (`create_`, `delete_`, `send_`) |
| **Rich descriptions**  | LLM chooses tools based on description — make it precise  |
| **Error returns**      | Return structured errors, not exceptions                  |
| **Schema strictness**  | Use enums, formats, and constraints to reduce hallucinations |
| **Timeouts**           | Always set — agents can loop on hanging tools             |
| **Logging**            | Log every tool call + result for observability            |

### Tool Call Lifecycle

```
LLM generates tool call
        │
        ▼
Parse & validate call parameters
        │
  ┌─────┴──────────┐
  │ Valid?         │
  │ Yes            │ No → Return error message to LLM
  ▼                │
Execute tool       │
  │                │
  ▼                │
Check result       │
  │                │
  ▼                │
Serialize result   │
  │                │
  ▼                │
Inject into context as observation
        │
        ▼
LLM reasons on observation → next action
```

### Parallel Tool Execution

```python
# Execute independent tools concurrently
import asyncio

async def run_tools_parallel(tool_calls: list[ToolCall]) -> list[ToolResult]:
    tasks = [execute_tool(tc) for tc in tool_calls]
    return await asyncio.gather(*tasks, return_exceptions=True)

# Pattern: LLM requests multiple tools in one response
# Agent runner detects non-dependent calls and parallelizes
```

---

## 5. Planning Strategies

### Planning Taxonomy

```
Planning Approaches
├── Reactive (no explicit plan)
│   └── ReAct: reason → act → observe each step
│
├── Plan-then-Execute
│   ├── Linear plan decomposition
│   └── Directed Acyclic Graph (DAG) of sub-tasks
│
├── Hierarchical
│   ├── Strategic level (what to do)
│   ├── Tactical level (how to do it)
│   └── Operational level (execute steps)
│
├── Search-based
│   ├── Tree of Thoughts (ToT)
│   └── LATS (Language Agent Tree Search)
│
└── Reflective / Iterative
    ├── Reflexion (learn from past failures)
    └── Self-RAG (retrieval-guided planning)
```

### Chain of Thought (CoT)

Internal reasoning before answering. No external actions.

```
Prompt: "Think step by step before answering."

Model output:
  <thinking>
    Step 1: Identify the goal — ...
    Step 2: What information do I need? — ...
    Step 3: What approach should I use? — ...
    Conclusion: ...
  </thinking>
  Final answer: ...
```

### ReAct (Reasoning + Acting)

Interleaves thought, action, and observation in a loop.

```
Thought: I need to find the current price of AAPL.
Action: search_web(query="AAPL stock price today")
Observation: Apple Inc. (AAPL) is trading at $193.42 as of market close.

Thought: Now I have the price. I need to compare it to last week.
Action: search_web(query="AAPL stock price last week close")
Observation: AAPL closed at $187.10 last Friday.

Thought: I have both data points. I can compute the change.
Action: calculate(expr="(193.42 - 187.10) / 187.10 * 100")
Observation: 3.38

Thought: I now have everything needed to answer.
Final Answer: AAPL rose approximately 3.38% over the past week.
```

### Plan-and-Execute

Separate planning phase from execution phase.

```
Phase 1 — PLANNER LLM
Input: Goal
Output: Ordered list of sub-tasks

  Plan:
    1. Search for recent earnings reports for AAPL
    2. Extract revenue, EPS, and guidance
    3. Compare to analyst expectations
    4. Summarize key takeaways

Phase 2 — EXECUTOR (sub-agents or tools)
  Execute each sub-task in sequence or parallel.
  Pass results to next step.

Phase 3 — SYNTHESIZER LLM
  Combine all results into final response.
```

### Tree of Thoughts (ToT)

Explore multiple reasoning paths simultaneously, select best.

```
                 [Problem]
                /    |    \
           [Path A] [Path B] [Path C]
           /   \      |      /   \
        [A1]  [A2]  [B1]  [C1] [C2]
              ✓               ✓
              └───────────────┘
                 Select best leaf
```

| Phase     | Action                                      |
|-----------|---------------------------------------------|
| Generate  | Produce N candidate reasoning steps         |
| Evaluate  | Score each step (by LLM or heuristic)       |
| Prune     | Remove low-scoring branches                 |
| Expand    | Continue from best branches                 |
| Terminate | Return answer from best complete path       |

### Reflexion

Agent learns from failures via self-critique and memory.

```
Attempt N
    │
    ▼
Evaluate result (success / failure)
    │
    ├── Success → Return result
    │
    └── Failure → Generate verbal reflection
                  "I failed because I didn't check X before Y"
                        │
                        ▼
                 Store reflection in episodic memory
                        │
                        ▼
                 Attempt N+1 with reflection in context
```

---

## 6. Agent Architectures

### 6.1 Single Agent (Standard Loop)

```
                ┌─────────────────┐
User/System ──► │   AGENT LOOP    │ ──► Output
                │                 │
                │  LLM + Tools    │
                │  + Memory       │
                └─────────────────┘
```

**Best for:** Focused single-domain tasks. Code generation, research, document analysis.

---

### 6.2 ReAct Agent

```
System Prompt
     │
     ▼
┌──────────────┐     Tool Call      ┌──────────────┐
│  LLM Reasoner│ ──────────────── ► │  Tool Runner │
│              │ ◄──────────────── │  (Sandbox)   │
│  [Thought]   │   Tool Result      └──────────────┘
│  [Action]    │
│  [Obs.]      │ ─── Final Answer ──► User
└──────────────┘
    ▲      │
    └──────┘ (loop)
```

**Implementation:**
```python
def react_loop(goal: str, tools: list[Tool], max_steps: int = 10) -> str:
    messages = [{"role": "system", "content": REACT_SYSTEM_PROMPT}]
    messages.append({"role": "user", "content": goal})

    for step in range(max_steps):
        response = llm.generate(messages, tools=tools)

        if response.finish_reason == "stop":
            return response.content  # Final answer

        if response.finish_reason == "tool_calls":
            for tool_call in response.tool_calls:
                result = execute_tool(tool_call)
                messages.append(tool_result_message(tool_call.id, result))

    return "Max steps reached without final answer."
```

---

### 6.3 Orchestrator-Worker Architecture

```
                    ┌─────────────────┐
User Task ────────► │   ORCHESTRATOR  │
                    │   (Planner LLM) │
                    └────┬────┬───┬───┘
                         │    │   │
            ┌────────────┘    │   └──────────────┐
            ▼                 ▼                  ▼
     ┌──────────┐      ┌──────────┐      ┌──────────┐
     │ Worker A │      │ Worker B │      │ Worker C │
     │ (Search) │      │ (Code)   │      │ (Write)  │
     └────┬─────┘      └────┬─────┘      └────┬─────┘
          │                 │                  │
          └─────────────────┴──────────────────┘
                            │
                            ▼
                    ┌─────────────────┐
                    │   SYNTHESIZER   │ ──► Final Output
                    │   (LLM)         │
                    └─────────────────┘
```

**Key Design Decisions:**
- Orchestrator prompt must define worker capabilities precisely
- Workers can be: tool-using agents, simple LLM calls, or even rule-based functions
- Synthesizer can be the same LLM as Orchestrator
- Add a critic/reviewer step before final output for quality gating

---

### 6.4 Blackboard Architecture

Inspired by AI planning systems. Agents share a common "blackboard" (shared state store).

```
                    ┌───────────────────────────┐
                    │        BLACKBOARD          │
                    │  (Shared State/Workspace)  │
                    │                            │
                    │  task: {...}               │
                    │  partial_results: [...]    │
                    │  agent_status: {...}       │
                    │  decisions: [...]          │
                    └────────┬──────────────────┘
                             │ (read/write)
               ┌─────────────┼──────────────────┐
               ▼             ▼                  ▼
        ┌──────────┐  ┌──────────┐      ┌──────────┐
        │ Agent A  │  │ Agent B  │      │ Control  │
        │ (domain) │  │ (domain) │      │ (coord.) │
        └──────────┘  └──────────┘      └──────────┘
```

**Blackboard Entry Schema:**
```json
{
  "id": "task-001",
  "type": "research_subtask",
  "status": "pending | in_progress | done | failed",
  "owner": "agent_b",
  "input": "Find Q3 2025 revenue for AAPL",
  "output": null,
  "created_at": "2025-06-01T10:00:00Z",
  "updated_at": "2025-06-01T10:01:00Z"
}
```

---

### 6.5 Reflection / Critic-Revision Architecture

```
┌──────────────┐     Draft      ┌──────────────┐
│   GENERATOR  │ ─────────────► │    CRITIC    │
│   (LLM)      │                │    (LLM)     │
└──────────────┘                └──────┬───────┘
        ▲                              │
        │       Critique/Score         │
        └──────────────────────────────┘
                    │
             (If score < threshold)
                    │
                    ▼ (revise with feedback)
            [Repeat until pass or max_iter]
                    │
                    ▼
             Final Output
```

**Critic Prompt Template:**
```
You are a critic reviewing the following output for [task].
Evaluate on these dimensions (score 1–5 each):
  - Accuracy: Does it correctly address the task?
  - Completeness: Are all aspects covered?
  - Format: Is the output well-structured?
  - Tone: Is the language appropriate?

Output JSON: {"scores": {...}, "total": N, "feedback": "...", "pass": true/false}
```

---

### 6.6 Plan-and-Execute Architecture

```
                ┌─────────────────────────────┐
User Goal ────► │         PLANNER             │
                │  Decomposes into sub-tasks  │
                │  Returns: DAG or list       │
                └──────────────┬──────────────┘
                               │
                 ┌─────────────▼──────────────┐
                 │       TASK QUEUE           │
                 │  [task1, task2, task3, ...] │
                 └─────────────┬──────────────┘
                               │
                 ┌─────────────▼──────────────┐
                 │        EXECUTOR            │
                 │  Runs tasks (sequential or │
                 │  parallel based on DAG)    │
                 └─────────────┬──────────────┘
                               │
                 ┌─────────────▼──────────────┐
                 │       RE-PLANNER           │
                 │  Adjusts plan if a task    │
                 │  fails or reveals new info │
                 └─────────────┬──────────────┘
                               │
                 ┌─────────────▼──────────────┐
                 │       SYNTHESIZER          │
                 │  Merges results → answer   │
                 └────────────────────────────┘
```

---

### 6.7 LLM Compiler

Maximizes parallelism by compiling a task into a DAG before execution.

```
Goal
 │
 ▼
[Planner] ──► Dependency Graph (JSON):
              {
                "tasks": [
                  {"id": 1, "tool": "search", "depends_on": []},
                  {"id": 2, "tool": "search", "depends_on": []},
                  {"id": 3, "tool": "calculate", "depends_on": [1,2]},
                  {"id": 4, "tool": "write", "depends_on": [3]}
                ]
              }
 │
 ▼
[Scheduler] ──► Execute tasks with no pending deps in parallel
                → Tasks 1 & 2 run concurrently
                → Task 3 starts when both complete
                → Task 4 starts when Task 3 completes
 │
 ▼
[Joiner LLM] ──► Determine if task is done or re-plan
```

---

## 7. Multi-Agent Patterns

### Communication Topologies

#### 7.1 Sequential (Pipeline)

```
Agent A ──► Agent B ──► Agent C ──► Output
```
Each agent's output becomes the next agent's input.
Use when tasks have strict dependency order.

#### 7.2 Hierarchical (Supervisor)

```
         ┌────────────────┐
         │   SUPERVISOR   │
         └───┬────┬───┬───┘
             │    │   │
         ┌───▼┐ ┌─▼──┐ ┌▼───┐
         │ A1 │ │ A2 │ │ A3 │
         └────┘ └────┘ └────┘
```
Supervisor delegates, collects, and decides. Workers don't communicate directly.

#### 7.3 Peer-to-Peer (Collaborative)

```
Agent A ◄──► Agent B
    ▲              ▲
    │              │
Agent D ◄──► Agent C
```
Agents can communicate directly. Requires explicit routing/addressing protocol.

#### 7.4 Broadcast (Swarm)

```
         COORDINATOR
         broadcasts task
        /     |     \     \
      A1     A2     A3    A4
      all observe shared state
      best result wins
```
Useful for parallelism + diversity (multiple solutions, pick best).

### Agent Role Definitions

| Role             | Responsibility                                          | Typical Prompt Focus                     |
|------------------|---------------------------------------------------------|------------------------------------------|
| **Planner**      | Decompose goals into sub-tasks                          | Structured output (JSON/YAML plan)       |
| **Researcher**   | Gather information from tools/web                       | Search, read, summarize                  |
| **Coder**        | Write and execute code                                  | Correctness, edge cases, tests           |
| **Writer**       | Generate text, reports, docs                            | Tone, structure, completeness            |
| **Critic**       | Review and score outputs                                | Scoring rubric, concise feedback         |
| **Executor**     | Run actions, API calls, mutations                       | Safety checks, idempotency               |
| **Coordinator**  | Route tasks, manage dependencies                        | Status tracking, failure handling        |
| **Memory Manager** | Maintain and retrieve context                         | Retrieval quality, compression           |

### Inter-Agent Communication Protocols

| Protocol         | Description                                            | Implementation                        |
|------------------|--------------------------------------------------------|---------------------------------------|
| **Message passing** | Agents send structured messages via queue          | Redis Streams, Kafka, in-memory queue |
| **Shared state** | Blackboard/shared DB all agents read/write            | Redis, Postgres, SQLite               |
| **Tool invocation** | Agent A calls Agent B as a tool                   | Function calling schema               |
| **Event-driven** | Agents subscribe to event channels                    | pub/sub (Redis, RabbitMQ)             |
| **Handoff**      | One agent transfers full context to another           | Direct context serialization          |

### Handoff Schema

```json
{
  "from_agent": "researcher",
  "to_agent": "writer",
  "task_id": "task-42",
  "context": {
    "original_goal": "Write a market analysis for EV sector",
    "completed_steps": ["search_market_data", "extract_financials"],
    "findings": {
      "market_size_2024": "$500B",
      "cagr": "22%",
      "key_players": ["Tesla", "BYD", "Rivian"]
    },
    "pending_steps": ["synthesize_report", "add_charts"]
  },
  "instructions": "Write a 3-page market analysis using the findings above. Use professional tone.",
  "timestamp": "2025-06-01T12:34:56Z"
}
```

---

## 8. Workflow Structures

### 8.1 Sequential Workflow

```
[Start] ──► [Step 1] ──► [Step 2] ──► [Step 3] ──► [End]
```
Each step waits for the previous. Simple, predictable, slower.

---

### 8.2 Parallel Workflow

```
                ┌──► [Step A] ──┐
[Start] ──► [Fork]──► [Step B] ──►[Join] ──► [End]
                └──► [Step C] ──┘
```
Independent steps run concurrently. Requires synchronization at join.

---

### 8.3 Conditional / Branching Workflow

```
[Start] ──► [Classify]
                │
         ┌──────┴──────┐
         ▼             ▼
      "simple"       "complex"
         │             │
     [Quick Answer] [Research Loop]
         │             │
         └──────┬───── ┘
                ▼
             [Format Output]
```

---

### 8.4 Loop / Retry Workflow

```
[Start]
   │
   ▼
[Attempt]
   │
   ▼
[Evaluate] ─── Pass ──► [End]
   │
  Fail
   │
   ▼
[Reflect / Adjust] ── max_iter reached ──► [Escalate]
   │
   └──► [Attempt] (loop)
```

---

### 8.5 Map-Reduce Workflow

```
[Input List: item1, item2, ... itemN]
           │
           ▼
        [MAP phase]
  item1 ──► [Agent/Tool] ──► result1
  item2 ──► [Agent/Tool] ──► result2
  ...
  itemN ──► [Agent/Tool] ──► resultN
           │
           ▼
      [REDUCE phase]
  [Synthesizer LLM]
  result1 + result2 + ... + resultN ──► Final Summary
```

Best for: Parallel document analysis, batch classification, bulk data enrichment.

---

### 8.6 Human-in-the-Loop (HITL) Workflow

```
[Agent Step]
     │
     ▼
[Checkpoint: confidence < threshold OR action is irreversible]
     │
     ▼
[Request Human Approval]
     │
  ┌──┴───┐
  │       │
Approve  Reject / Edit
  │       │
  ▼       ▼
[Continue] [Revise with feedback]
```

**HITL Trigger Conditions:**
- Tool would delete/modify production data
- Confidence score below threshold
- Cost/risk above policy limit
- Novel situation not covered by training
- Regulatory/compliance checkpoint required

---

### 8.7 Event-Driven (Reactive) Workflow

```
External Events ──► Event Bus ──► [Router]
                                      │
                        ┌─────────────┼──────────────┐
                        ▼             ▼              ▼
                  [Email Agent] [Calendar Agent] [Alert Agent]
                        │             │              │
                        └─────────────┴──────────────┘
                                      │
                              [Aggregator / Reporter]
```

---

## 9. Context & State Management

### State Architecture

```
Agent State
├── Ephemeral (in-memory, this run)
│   ├── messages[]          ← full conversation history
│   ├── current_task        ← what we're doing now
│   ├── tool_results[]      ← recent observations
│   └── scratch_pad         ← working notes
│
├── Session (persisted, this user session)
│   ├── session_id
│   ├── user_profile
│   ├── completed_tasks[]
│   └── user_preferences
│
└── Long-term (persisted, cross-session)
    ├── user_history
    ├── learned_facts
    ├── cached_results
    └── model_adapters (LoRA, fine-tune)
```

### Serializable State Schema

```json
{
  "agent_id": "research-agent-01",
  "session_id": "sess-2025-06-01-abc123",
  "status": "running | paused | completed | failed",
  "goal": "Analyze EV market trends for 2025",
  "plan": ["search", "extract", "synthesize"],
  "completed_steps": ["search"],
  "current_step": "extract",
  "messages": [...],
  "tool_results": [...],
  "memory_refs": ["mem-001", "mem-042"],
  "created_at": "2025-06-01T10:00:00Z",
  "updated_at": "2025-06-01T10:05:30Z",
  "metadata": {"model": "claude-3-5-sonnet", "tokens_used": 12400}
}
```

### Checkpointing Pattern

```python
class AgentCheckpointer:
    """Save/restore agent state for fault tolerance and pause/resume."""

    def save(self, state: AgentState, checkpoint_id: str) -> None:
        self.store.set(f"checkpoint:{checkpoint_id}", state.to_json())

    def restore(self, checkpoint_id: str) -> AgentState:
        raw = self.store.get(f"checkpoint:{checkpoint_id}")
        return AgentState.from_json(raw)

    def list_checkpoints(self, agent_id: str) -> list[str]:
        return self.store.list(prefix=f"checkpoint:{agent_id}:")
```

### Context Compression Strategies

| Strategy              | How It Works                                               | Token Savings |
|-----------------------|------------------------------------------------------------|---------------|
| Summarization         | LLM rewrites old messages into summary                     | 60–80%        |
| Selective pruning     | Remove low-relevance messages (by LLM scoring)             | 30–60%        |
| Tool result trimming  | Keep only key fields from tool outputs                     | 40–70%        |
| Chunked retrieval     | Never load full docs; only retrieve relevant chunks        | 70–90%        |
| Progressive compression | Oldest messages most compressed; recent messages full   | 50–70%        |

---

## 10. Model Context Protocol (MCP)

### What is MCP?

MCP (Model Context Protocol by Anthropic) is an **open standard** for connecting AI models to external tools, data sources, and capabilities via a unified client-server protocol.

```
                ┌─────────────────────────┐
                │       AI Model           │
                │    (Claude, GPT, etc.)   │
                └────────────┬────────────┘
                             │ MCP Protocol
                ┌────────────▼────────────┐
                │       MCP CLIENT        │
                │  (embedded in host app) │
                └───┬──────┬──────┬───────┘
                    │      │      │
            ┌───────▼─┐ ┌──▼────┐ ┌▼────────┐
            │MCP      │ │ MCP   │ │ MCP     │
            │Server A │ │Server │ │ Server  │
            │(Files)  │ │(DB)   │ │ (APIs)  │
            └─────────┘ └───────┘ └─────────┘
```

### MCP Architecture

| Component         | Role                                                          |
|-------------------|---------------------------------------------------------------|
| **MCP Host**      | Application that embeds the AI (Claude Desktop, IDE plugin)  |
| **MCP Client**    | Protocol layer inside the host; manages server connections   |
| **MCP Server**    | Exposes tools/resources/prompts to clients via MCP protocol  |
| **Transport**     | How client and server communicate (Stdio, HTTP/SSE)          |

### Transport Types

| Transport        | How it Works                         | Use Case                          |
|------------------|--------------------------------------|-----------------------------------|
| **Stdio**        | stdin/stdout JSON-RPC                | Local servers, CLI tools          |
| **HTTP + SSE**   | HTTP requests + Server-Sent Events   | Remote servers, cloud APIs        |
| **WebSocket**    | Full-duplex connection                | Real-time, low-latency            |

### MCP Primitives

| Primitive     | Description                                              | Example                                 |
|---------------|----------------------------------------------------------|-----------------------------------------|
| **Tools**     | Actions the model can invoke                             | `create_file`, `query_db`, `send_email` |
| **Resources** | Data the model can read (context injection)              | `file://report.pdf`, `db://customers`   |
| **Prompts**   | Predefined prompt templates with parameters              | `summarize_document(url, length)`       |
| **Sampling**  | Server requests model generation (inverse flow)          | Server asks model to generate text      |

### MCP Server Structure (Python SDK)

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

app = Server("my-data-server")

@app.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="query_database",
            description="Execute a read-only SQL query against the analytics database.",
            inputSchema={
                "type": "object",
                "properties": {
                    "sql": {"type": "string", "description": "SQL SELECT statement"}
                },
                "required": ["sql"]
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "query_database":
        result = db.execute(arguments["sql"])
        return [TextContent(type="text", text=str(result))]

async def main():
    async with stdio_server() as streams:
        await app.run(streams[0], streams[1], app.create_initialization_options())
```

### MCP Configuration (claude_desktop_config.json)

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/tobias/projects"],
      "env": {}
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {"DATABASE_URL": "postgresql://localhost:5432/mydb"}
    },
    "custom-server": {
      "command": "python",
      "args": ["/path/to/my_server.py"],
      "env": {"API_KEY": "${MY_API_KEY}"}
    }
  }
}
```

---

## 11. Prompting Patterns for Agents

### System Prompt Structure for Agents

```
# Role and Identity
You are [AGENT_NAME], a [ROLE]. Your primary goal is [GOAL].

# Capabilities
You have access to the following tools:
- [TOOL_1]: [description]
- [TOOL_2]: [description]

# Workflow
Follow this approach for every task:
1. [Step 1]
2. [Step 2]
3. [Step 3]

# Constraints
- NEVER [prohibited action]
- ALWAYS [required behavior]
- If uncertain, [fallback behavior]

# Output Format
[Specify exact format: JSON schema, markdown template, etc.]

# Examples
[Include 1–2 few-shot examples demonstrating correct behavior]
```

### Prompting Patterns

| Pattern                     | Structure                                                  | Use Case                              |
|-----------------------------|------------------------------------------------------------|---------------------------------------|
| **Zero-shot**               | Task description only                                      | Simple, well-defined tasks            |
| **Few-shot**                | Task + N examples                                         | Tasks requiring specific format       |
| **Chain-of-Thought**        | "Think step by step before answering"                     | Multi-step reasoning                  |
| **Self-consistency**        | Generate N responses, vote on best                         | High-stakes reasoning                 |
| **Role prompting**          | "You are an expert in X with 20 years experience"         | Domain-specific quality boost         |
| **Skeleton-of-thought**     | Generate outline first, then fill in sections             | Long-form generation                  |
| **Meta-prompting**          | Ask model to generate the best prompt for a task          | Prompt optimization                   |
| **Program of Thought**      | Model writes code, executes it for final answer           | Math, data analysis                   |

### Tool-Use Prompting Tips

```
✓ Be explicit: "Use the search tool to find this, then use the calculator"
✓ Specify format: "Return results as JSON with keys: title, url, summary"
✓ Set stopping condition: "Stop when you have found 3 valid sources"
✓ Define tool priority: "Prefer internal KB search over web search"
✗ Don't: "Figure out a way to find the information" (too vague)
✗ Don't: Omit what to do if a tool fails
```

### Structured Output Prompting

```
Respond ONLY with valid JSON matching this schema. No markdown, no prose.

Schema:
{
  "action": "tool_call | final_answer",
  "tool_name": "string | null",
  "tool_args": "object | null",
  "reasoning": "string",
  "final_answer": "string | null"
}
```

---

## 12. Evaluation & Observability

### Evaluation Dimensions

| Dimension             | Metric / Method                                            |
|-----------------------|------------------------------------------------------------|
| **Task completion**   | % tasks completed correctly end-to-end                    |
| **Tool call accuracy**| % tool calls with correct arguments                       |
| **Hallucination rate**| % responses containing factual errors                     |
| **Efficiency**        | Average steps / tokens per task                           |
| **Latency**           | Wall-clock time from task start to completion             |
| **Cost**              | Token cost per task                                        |
| **Recovery rate**     | % of failed steps that are successfully self-corrected    |
| **Safety**            | % of completions that violate safety policies             |

### Tracing Schema (OpenTelemetry-compatible)

```json
{
  "trace_id": "4bf92f3577b34da6",
  "spans": [
    {
      "span_id": "00f067aa0ba902b7",
      "name": "agent.step",
      "start_time": "2025-06-01T10:00:00.000Z",
      "end_time": "2025-06-01T10:00:01.234Z",
      "attributes": {
        "agent.id": "research-agent",
        "step.number": 1,
        "llm.model": "claude-3-5-sonnet",
        "llm.input_tokens": 1240,
        "llm.output_tokens": 320,
        "tool.called": "search_web",
        "tool.args": {"query": "AAPL Q3 earnings 2025"},
        "tool.result_length": 1850,
        "tool.latency_ms": 423
      }
    }
  ]
}
```

### Observability Stack

```
Agent ──► Instrumentation
              │
    ┌─────────▼─────────┐
    │  Trace Collector   │ ← LangSmith, Langfuse, Helicone, Arize
    │  (OpenTelemetry)   │
    └─────────┬──────────┘
              │
    ┌─────────▼──────────┐
    │   Storage          │ ← ClickHouse, Postgres, S3
    └─────────┬──────────┘
              │
    ┌─────────▼──────────┐
    │   Dashboard        │ ← Grafana, Langfuse UI, custom
    └────────────────────┘
```

### Key Observability Signals

| Signal                | What to Capture                                          |
|-----------------------|----------------------------------------------------------|
| **LLM call**          | Model, input/output tokens, latency, prompt, completion  |
| **Tool call**         | Tool name, args, result, latency, error if any           |
| **Agent step**        | Step number, reasoning, decision, outcome                |
| **Memory access**     | Query, retrieved docs, relevance scores                  |
| **Error**             | Type, message, stack, step context, recovery action      |
| **Cost**              | Tokens × price per token per LLM call                   |

### Evaluation Harnesses

| Framework      | Strengths                                                 |
|----------------|-----------------------------------------------------------|
| **LangSmith**  | Native LangChain integration, dataset management          |
| **Langfuse**   | Open-source, self-hostable, multi-framework               |
| **Ragas**      | RAG-specific metrics (faithfulness, context precision)    |
| **DeepEval**   | Unit-test style evals, CI integration                     |
| **PromptFoo**  | YAML-based, fast local eval runs                          |
| **HELM**       | Stanford benchmark suite for holistic evaluation          |

---

## 13. Safety, Security & Human Oversight

### Threat Model for Agentic Systems

| Threat                    | Description                                                  | Mitigation                                    |
|---------------------------|--------------------------------------------------------------|-----------------------------------------------|
| **Prompt injection**      | Malicious instructions embedded in tool results or data      | Input sanitization, output validation         |
| **Tool misuse**           | Agent calls destructive tools inadvertently                  | Allowlist, confirmation for write ops         |
| **Scope creep**           | Agent takes unauthorized actions beyond goal                 | Capability scoping, action logs               |
| **Data exfiltration**     | Agent sends sensitive data to external services              | Network egress controls, DLP                  |
| **Jailbreak via tool**    | Attacker tricks agent via web content or API response        | Result sandboxing, trust tiers                |
| **Hallucinated tool args**| Wrong args cause data corruption or system errors            | Schema validation, dry-run mode               |
| **Infinite loops**        | Agent loops without progress                                 | Step limits, loop detection                   |
| **Cascading failures**    | One agent's error propagates across the system               | Circuit breakers, isolation boundaries        |

### Safety Design Principles

```
1. MINIMAL FOOTPRINT
   → Only request permissions actually needed
   → Prefer read-only over read-write
   → Scope API keys to minimum required resources

2. REVERSIBILITY PREFERENCE
   → Prefer reversible actions (soft delete > hard delete)
   → Archive instead of destroy
   → Dry-run mode before live execution

3. EXPLICIT > IMPLICIT
   → Require confirmation for irreversible high-impact actions
   → Surface assumptions to the user before acting
   → Log every action with full context

4. FAIL SAFE
   → On error, default to safe state (stop, not continue)
   → Never silently swallow errors
   → Escalate to human on uncertainty

5. LEAST PRIVILEGE
   → Tools have only the access they need
   → Agents have only the tools they need
   → Sessions expire; tokens rotate
```

### Input Sanitization Pattern

```python
def sanitize_tool_result(result: str, max_length: int = 8000) -> str:
    """
    Prevent prompt injection from tool results.
    """
    # Truncate oversized results
    result = result[:max_length]

    # Flag suspicious patterns (instruction injection)
    injection_patterns = [
        r"ignore previous instructions",
        r"you are now",
        r"system prompt",
        r"<\|im_start\|>",
        r"\[INST\]",
    ]
    for pattern in injection_patterns:
        if re.search(pattern, result, re.IGNORECASE):
            return f"[SUSPICIOUS CONTENT DETECTED — RESULT WITHHELD]: {pattern}"

    return result
```

### Human Oversight Checklist

```
Before deployment:
  □ Define explicit stopping conditions for all agent loops
  □ Set max_steps and max_cost limits
  □ Identify all irreversible actions and add HITL gates
  □ Implement full action logging with replay capability
  □ Define escalation paths (to human or fallback system)
  □ Test failure modes: what happens if tool X fails?

At runtime:
  □ Monitor cost per task against budget
  □ Alert on unusual tool call patterns
  □ Surface agent uncertainty scores
  □ Enable pause/resume for long-running tasks

Post-run review:
  □ Audit all write/delete operations
  □ Review error recovery chains
  □ Collect human feedback on outputs
  □ Identify hallucinations or tool misuse
```

---

## 14. Framework Ecosystem

### Framework Comparison

| Framework      | Paradigm              | Best For                                  | Language    |
|----------------|-----------------------|-------------------------------------------|-------------|
| **LangGraph**  | Graph-based stateful  | Complex flows, stateful multi-agent       | Python      |
| **LangChain**  | Chain composition     | Rapid prototyping, broad ecosystem        | Python / JS |
| **LlamaIndex** | RAG-first             | Document/knowledge agents                 | Python      |
| **AutoGen**    | Conversational multi-agent | Debate, coding, research agents      | Python      |
| **CrewAI**     | Role-based crews      | Human-team analogies, quick setup         | Python      |
| **Semantic Kernel** | Plugin-based     | Enterprise, .NET / Python integration     | .NET / Python |
| **Haystack**   | Pipeline-based        | NLP pipelines, production RAG             | Python      |
| **Ollama**     | Local model serving   | Privacy-first local agents                | REST API    |

### LangGraph Core Concepts

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict

class AgentState(TypedDict):
    messages: list
    tool_results: list
    step: int
    finished: bool

# Define nodes (each is a function or agent)
def research_node(state: AgentState) -> AgentState:
    ...

def write_node(state: AgentState) -> AgentState:
    ...

def should_continue(state: AgentState) -> str:
    return END if state["finished"] else "research"

# Build graph
graph = StateGraph(AgentState)
graph.add_node("research", research_node)
graph.add_node("write", write_node)
graph.add_edge("research", "write")
graph.add_conditional_edges("write", should_continue, {"research": "research", END: END})
graph.set_entry_point("research")

agent = graph.compile()
```

### Local Agent Stack (Ollama + LangChain)

```python
from langchain_ollama import ChatOllama
from langchain.agents import AgentExecutor, create_react_agent
from langchain_core.tools import tool

# 1. Local LLM
llm = ChatOllama(model="llama3.2", temperature=0)

# 2. Define tools
@tool
def search_local_kb(query: str) -> str:
    """Search internal knowledge base. Input: natural language query."""
    return vector_store.similarity_search(query, k=5)

# 3. Create ReAct agent
agent = create_react_agent(llm, [search_local_kb], REACT_PROMPT)
executor = AgentExecutor(agent=agent, tools=[search_local_kb], verbose=True)

# 4. Run
result = executor.invoke({"input": "What are our Q3 targets?"})
```

---

## 15. Design Principles & Anti-Patterns

### Core Design Principles

| Principle               | Guideline                                                      |
|-------------------------|----------------------------------------------------------------|
| **Start simple**        | Single agent with 3–5 tools beats over-engineered multi-agent |
| **Fail loudly**         | Surface errors immediately; don't mask failures               |
| **Instrument everything** | Trace every LLM call, every tool call from day one          |
| **Constrain the output** | Structured output (JSON) > free text for reliability        |
| **Design for latency**  | Parallelize; cache tool results; stream responses             |
| **Separate concerns**   | Planning ≠ execution ≠ memory ≠ tool runner                  |
| **Make goals explicit** | Ambiguous goals lead to hallucinated plans                    |
| **Right-size the model**| Use small/fast models for routing; large for reasoning        |
| **Test failure paths**  | What happens when the search tool returns nothing?            |
| **Bound everything**    | Token budgets, step limits, cost caps, timeouts               |

### Anti-Patterns

| Anti-Pattern                  | Problem                                             | Fix                                         |
|-------------------------------|-----------------------------------------------------|---------------------------------------------|
| **God agent**                 | One agent with 30+ tools; can't decide what to use  | Break into specialized sub-agents           |
| **Unbounded loops**           | Agent loops infinitely without termination          | Enforce max_steps; detect repeated actions  |
| **Implicit state**            | State passed as raw text in messages                | Use explicit typed state objects            |
| **Tool result injection**     | Raw tool output pasted into prompt without sanitizing | Sanitize and truncate all external input  |
| **Premature multi-agent**     | Multi-agent for simple tasks; adds latency + cost   | Use single agent first; scale out later     |
| **No memory eviction**        | Context window fills; quality degrades silently     | Implement compression and eviction strategy |
| **Vague tool descriptions**   | LLM picks wrong tool; results in errors             | Write precise, discriminating descriptions  |
| **Synchronous everything**    | All tools run serially; massive latency             | Identify and parallelize independent tools  |
| **No checkpointing**          | Long task fails at step 19/20; restart from scratch | Checkpoint after every successful step      |
| **Blind trust in LLM plans**  | Agent acts on hallucinated sub-task list            | Validate plans before execution             |
| **No HITL gate**              | Agent deletes production data autonomously          | Add confirmation for irreversible actions   |
| **Token waste**               | Full documents in every prompt                      | Chunk and retrieve only what's needed       |

---

## 16. Quick Decision Matrix

### Which Architecture Should I Use?

```
Q1: How many distinct domains/skills are needed?
    └── 1–2 → Single Agent with tools (Section 6.1/6.2)
    └── 3+  → Go to Q2

Q2: Do the sub-tasks have strict sequential dependencies?
    └── Yes → Plan-and-Execute (Section 6.6)
    └── No  → Go to Q3

Q3: Can sub-tasks run in parallel?
    └── Yes → LLM Compiler / Orchestrator-Worker (Section 6.3/6.7)
    └── No  → Orchestrator-Worker sequential (Section 6.3)

Q4: Do you need quality gates / revision?
    └── Yes → Critic-Revision loop (Section 6.5)

Q5: Do agents need to communicate peer-to-peer?
    └── Yes → Blackboard or Peer-to-peer (Section 6.4/7.1)
```

### Which Memory Strategy?

```
Knowledge base < 10k tokens?     → Stuff into system prompt
Knowledge base 10k–100k tokens?  → RAG with vector store
Knowledge base > 100k tokens?    → RAG + hierarchical indexing
Cross-session memory needed?      → Episodic + declarative store
User-specific preferences?        → Profile KV store
Complex relationships?            → Graph DB (Neo4j, NetworkX)
```

### Which Workflow Pattern?

| Need                                      | Pattern                        |
|-------------------------------------------|--------------------------------|
| Simple linear task                        | Sequential (8.1)               |
| Batch processing N items                  | Map-Reduce (8.5)               |
| Independent sub-tasks                     | Parallel (8.2)                 |
| Different handling per input type         | Conditional (8.3)              |
| Quality improvement needed                | Loop/Retry with Critic (8.4)   |
| Irreversible actions                      | HITL checkpoint (8.6)          |
| Trigger on external events                | Event-driven (8.7)             |

---

## Appendix: Key Terminology

| Term                  | Definition                                                                     |
|-----------------------|--------------------------------------------------------------------------------|
| **Agentic loop**      | The perceive→reason→act→observe cycle an agent repeats                        |
| **Tool call**         | LLM-generated request to execute an external function                          |
| **Grounding**         | Connecting LLM outputs to real data/facts via retrieval or tools               |
| **Hallucination**     | LLM generating plausible but factually incorrect content                       |
| **RAG**               | Retrieval-Augmented Generation: inject retrieved docs into context             |
| **Context window**    | Maximum token length the LLM can process in one call                          |
| **Orchestrator**      | Component that coordinates multiple agents or steps                            |
| **Handoff**           | Passing task context from one agent to another                                 |
| **HITL**              | Human-in-the-Loop: requiring human approval at key decision points            |
| **MCP**               | Model Context Protocol: open standard for LLM ↔ tool communication           |
| **ReAct**             | Reasoning + Acting: alternating thought and action in agent loops              |
| **ToT**               | Tree of Thoughts: explore multiple reasoning paths simultaneously              |
| **LATS**              | Language Agent Tree Search: MCTS-inspired search over reasoning paths         |
| **Reflexion**         | Agent self-critique and learning from past failures                            |
| **DAG**               | Directed Acyclic Graph: task dependency structure for parallel execution       |
| **Episodic memory**   | Memory of specific past events and interactions                                |
| **Semantic memory**   | General knowledge stored in vector embeddings                                  |
| **Parametric memory** | Knowledge encoded in the LLM's weights during training                        |
| **Prompt injection**  | Malicious instructions hidden in external data to hijack agent behavior       |
| **Swarm intelligence**| Emergent behavior from multiple simple agents with local rules                |
| **Checkpointing**     | Persisting agent state so tasks can be resumed after failure                  |

---

*Reference card · Last updated: June 2026*
