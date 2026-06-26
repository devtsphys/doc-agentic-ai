# Agentic AI — Comprehensive Reference Card

> **Scope:** Design, architectures, workflows, MCP, skills, tools, multi-agent patterns, prompt engineering, orchestration, evaluation, and production best practices.
> **Audience:** ML/AI engineers building production agentic systems.

---

## Table of Contents

1. [Core Concepts & Terminology](#1-core-concepts--terminology)
2. [Agent Anatomy](#2-agent-anatomy)
3. [Architecture Patterns](#3-architecture-patterns)
4. [Reasoning & Planning Strategies](#4-reasoning--planning-strategies)
5. [Tool Use & Function Calling](#5-tool-use--function-calling)
6. [Model Context Protocol (MCP)](#6-model-context-protocol-mcp)
7. [Skills & Skill Design](#7-skills--skill-design)
8. [Multi-Agent Systems](#8-multi-agent-systems)
9. [Memory Systems](#9-memory-systems)
10. [Context Window Management](#10-context-window-management)
11. [Prompt Engineering for Agents](#11-prompt-engineering-for-agents)
12. [Agentic Workflows In Depth](#12-agentic-workflows-in-depth)
13. [Evaluation & Observability](#13-evaluation--observability)
14. [Safety & Guardrails](#14-safety--guardrails)
15. [Sample Project Structures](#15-sample-project-structures)
16. [Framework Comparison](#16-framework-comparison)
17. [Methods & API Reference Table](#17-methods--api-reference-table)
18. [Design Patterns Catalogue](#18-design-patterns-catalogue)
19. [Anti-Patterns](#19-anti-patterns)
20. [Best Practices](#20-best-practices)

---

## 1. Core Concepts & Terminology

| Term | Definition |
|---|---|
| **Agent** | An LLM-powered system that autonomously perceives, reasons, plans, and acts through tools or sub-agents to accomplish goals. |
| **Agentic AI** | AI operating in loops with real-world side-effects, extended context, and deferred human feedback. |
| **Autonomy spectrum** | Continuum from fully human-in-the-loop (HITL) → supervised automation → fully autonomous. |
| **Tool / Function** | A callable unit (code, API, search) the agent can invoke. Structured input/output contract. |
| **Skill** | A reusable, scoped capability package (prompt template + tools + config + docs). |
| **Orchestrator** | Component that decides which agent or tool to invoke next and assembles the result. |
| **Executor / Subagent** | Downstream agent that implements a specific task on behalf of the orchestrator. |
| **Perception** | How an agent ingests environment state: text, files, images, structured data, tool results. |
| **Action space** | The full set of tools/actions available to an agent in a given context. |
| **Context window** | The finite token buffer the LLM sees per inference call. |
| **Scratchpad / CoT** | Internal reasoning trace produced before final output (Chain-of-Thought). |
| **Plan** | An ordered sequence of steps or subgoals the agent intends to execute. |
| **Reflection** | Agent critiques its own outputs to improve them without external feedback. |
| **Grounding** | Anchoring agent decisions in retrieved, verified, or tool-confirmed facts. |
| **Hallucination** | LLM generating plausible but false information — mitigated by grounding and verification. |
| **RAG** | Retrieval-Augmented Generation — inject retrieved documents into context at inference time. |
| **Memory** | Information persisted beyond a single context window (working/episodic/semantic/procedural). |
| **MCP** | Model Context Protocol — open standard for agent-to-tool and agent-to-agent communication. |
| **Human-in-the-loop** | Points where human review, confirmation, or correction is injected into an agentic flow. |
| **Interrupt / Breakpoint** | Programmatic pause in a workflow for HITL intervention. |
| **Sycophancy** | Agent conforming to user preference rather than ground truth — a key safety risk. |
| **Reward hacking** | Agent optimising a proxy metric in unintended ways. |

---

## 2. Agent Anatomy

```
┌─────────────────────────────────────────────────────────────────────┐
│                            AGENT LOOP                               │
│                                                                     │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐              │
│  │ Perceive │───▶│  Reason/Plan │───▶│  Select Tool │              │
│  └──────────┘    └──────────────┘    └──────┬───────┘              │
│       ▲                                     │                       │
│       │           ┌──────────────┐          │                       │
│       └───────────│  Observe     │◀──────── │ Execute Tool          │
│                   └──────────────┘          │                       │
│                                       ┌─────▼──────┐               │
│                                       │ Tool Layer │               │
│                                       │  (MCP/API) │               │
│                                       └────────────┘               │
└─────────────────────────────────────────────────────────────────────┘

Key components:
  • System Prompt     — persona, constraints, instructions, tool descriptions
  • Working Memory    — current conversation + scratchpad (in-context)
  • Long-term Memory  — vector DB, KV store, relational DB (external)
  • Tool Registry     — structured catalogue of callable functions
  • Planner           — decomposition logic (CoT, ReAct, LATS, etc.)
  • Executor          — calls external tools/APIs
  • Observer          — parses tool responses, updates state
  • Termination check — goal achieved? error? max steps exceeded?
```

### Core Agent State Machine

```
IDLE ──[task received]──▶ PLANNING ──[plan ready]──▶ EXECUTING
  ▲                           │                          │
  │                     [replan trigger]           [step done]
  │                           │                          │
  └──[goal met / error]── EVALUATING ◀─────────────────┘
```

---

## 3. Architecture Patterns

### 3.1 Single-Agent (ReAct)

```
System Prompt
  └── LLM
        ├── Thought → Action → Observation (loop)
        └── Final Answer
```

**Use when:** Single-domain tasks, moderate complexity, latency matters.

### 3.2 Orchestrator–Executor

```
User ──▶ Orchestrator LLM
              ├──▶ Executor Agent A (search)
              ├──▶ Executor Agent B (code)
              └──▶ Executor Agent C (write)
         Orchestrator assembles final output
```

**Use when:** Parallel subtasks, heterogeneous tools, need centralized control.

### 3.3 Hierarchical Multi-Agent

```
Top-level Planner
  ├── Domain Orchestrator A
  │     ├── Specialist A1
  │     └── Specialist A2
  └── Domain Orchestrator B
        ├── Specialist B1
        └── Specialist B2
```

**Use when:** Complex multi-domain tasks (research + coding + writing), large teams of agents.

### 3.4 Pipeline / DAG

```
Step 1 (extract) ──▶ Step 2 (transform) ──▶ Step 3 (generate) ──▶ Step 4 (validate)
```

**Use when:** Deterministic multi-step processes (ETL, document processing, CI/CD).

### 3.5 Supervisor–Worker

```
Supervisor (routes, monitors, aggregates)
  ├── Worker Pool [W1, W2, W3, ...]
  └── Human-in-the-Loop checkpoint
```

**Use when:** Parallel work with quality control, dynamic scaling.

### 3.6 Debate / Critic Pattern

```
Generator Agent ──▶ Critic Agent ──▶ Refined Output
                       │
                  [Repeat N rounds]
```

**Use when:** High-stakes outputs, factuality requirements, code review.

### 3.7 Map–Reduce Agent

```
Input (large doc / dataset)
  │
  ├──[Map] Agent chunk 1 ──▶ result 1
  ├──[Map] Agent chunk 2 ──▶ result 2
  └──[Map] Agent chunk N ──▶ result N
                │
          [Reduce] Aggregator Agent ──▶ Final Output
```

**Use when:** Large document analysis, parallelizable summarization.

---

## 4. Reasoning & Planning Strategies

| Strategy | Acronym | Mechanism | Best For |
|---|---|---|---|
| Chain-of-Thought | CoT | Sequential reasoning steps in text | General reasoning |
| Tree of Thoughts | ToT | Branching + BFS/DFS search over thoughts | Complex planning |
| ReAct | ReAct | Interleave Thought / Action / Observation | Tool-using agents |
| Plan-and-Execute | P&E | Separate planning from execution phases | Long tasks |
| Reflexion | — | Self-critique + memory → retry | Error correction |
| Language Agent Tree Search | LATS | Monte Carlo Tree Search over LLM actions | Exploration tasks |
| Least-to-Most Prompting | LtM | Decompose → solve sub-problems sequentially | Complex multi-step |
| Self-Consistency | SC | Sample N paths → majority vote | Factual accuracy |
| Skeleton-of-Thought | SoT | Outline → parallel fill-in → merge | Long-form content |

### ReAct Loop (Pseudo-code)

```python
def react_loop(task: str, tools: dict, max_steps: int = 20) -> str:
    history = []
    for step in range(max_steps):
        thought, action, action_input = llm_reason(task, history, tools)
        if action == "finish":
            return action_input  # final answer

        observation = tools[action](action_input)
        history.append({
            "thought": thought,
            "action": action,
            "action_input": action_input,
            "observation": observation
        })
    return "MAX_STEPS_EXCEEDED"
```

### Plan-and-Execute (Pseudo-code)

```python
def plan_and_execute(task: str, tools: dict) -> str:
    # Phase 1: Planning (one LLM call)
    plan: list[str] = planner_llm(task)

    # Phase 2: Execution (one call per step, may replan)
    results = []
    for step in plan:
        result = executor_llm(step, tools, context=results)
        results.append(result)
        if needs_replan(result):
            plan = replanner_llm(task, plan, results)

    return synthesizer_llm(task, results)
```

---

## 5. Tool Use & Function Calling

### Tool Definition Schema (OpenAI / Anthropic style)

```python
tool_schema = {
    "name": "search_web",
    "description": "Search the web for current information. Use when you need facts beyond your training data.",
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "The search query string. Be specific and concise."
            },
            "num_results": {
                "type": "integer",
                "description": "Number of results to return (1–10). Default: 5.",
                "default": 5
            }
        },
        "required": ["query"]
    }
}
```

### Tool Result Injection

```python
# Anthropic messages format
messages = [
    {"role": "user", "content": "What is today's weather in Berlin?"},
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "tu_01", "name": "get_weather",
         "input": {"city": "Berlin"}}
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "tu_01",
         "content": '{"temp": 18, "condition": "partly cloudy"}'}
    ]}
]
```

### Tool Categories

| Category | Examples | Notes |
|---|---|---|
| **Search / Retrieval** | web_search, vector_search, sql_query | Ground agents in facts |
| **Code Execution** | python_repl, bash, js_sandbox | Sandboxed; resource limits |
| **File Operations** | read_file, write_file, list_dir | Path validation essential |
| **External APIs** | http_request, rest_call | Rate limits, auth injection |
| **Database** | db_query, db_write | RBAC, read-only by default |
| **Communication** | send_email, post_slack | Confirm before send |
| **Memory** | memory_store, memory_retrieve | Long-term state |
| **Sub-agent** | spawn_agent, call_agent | Multi-agent composition |
| **Computer use** | screenshot, click, type | Stateful session management |

### Parallel Tool Calling

```python
# Anthropic Claude supports parallel tool calls in one turn
response = client.messages.create(
    model="claude-sonnet-4-6",
    tools=[search_tool, calculator_tool, calendar_tool],
    messages=[{"role": "user", "content": "Research AI trends and compute market CAGR"}]
)
# Response may contain multiple tool_use blocks simultaneously
tool_calls = [b for b in response.content if b.type == "tool_use"]
results = await asyncio.gather(*[execute_tool(tc) for tc in tool_calls])
```

---

## 6. Model Context Protocol (MCP)

### What is MCP?

MCP (Model Context Protocol) is an open standard (Anthropic, 2024) that defines how AI agents communicate with external tools, data sources, and other agents. It decouples the **host** (the LLM application) from **servers** (tools/resources) via a typed, discoverable protocol.

```
┌─────────────┐     MCP Protocol     ┌──────────────────┐
│  MCP Host   │◀────────────────────▶│   MCP Server     │
│ (Agent/App) │   JSON-RPC 2.0       │ (Tool provider)  │
└─────────────┘                      └──────────────────┘
      │                                      │
      │                              ┌───────┴────────┐
      │                              │  Resources     │
      │                              │  Tools         │
      │                              │  Prompts       │
      │                              └────────────────┘
      │
 Multiple MCP Servers (stdio / SSE / HTTP)
```

### MCP Primitives

| Primitive | Description | Example |
|---|---|---|
| **Tool** | Callable function with typed input/output | `search_documents(query: str)` |
| **Resource** | Readable data source (static or dynamic) | `file://project/README.md`, `db://customers` |
| **Prompt** | Reusable prompt template the server exposes | `summarize_template`, `review_pr` |
| **Sampling** | Server requests the host LLM to generate | Used in agentic server flows |

### MCP Transport Layers

| Transport | Use Case | Format |
|---|---|---|
| **stdio** | Local processes, CLI tools | JSON-RPC over stdin/stdout |
| **HTTP + SSE** | Remote servers, cloud services | Server-Sent Events for streaming |
| **WebSocket** | Bidirectional real-time | Full-duplex JSON-RPC |

### MCP Server Implementation (Python — FastMCP)

```python
# pip install mcp fastmcp
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-data-server")

@mcp.tool()
def search_products(query: str, max_results: int = 10) -> list[dict]:
    """Search the product catalog by keyword.

    Args:
        query: Search terms to match against product names and descriptions.
        max_results: Maximum number of products to return (1–50).
    Returns:
        List of product dicts with keys: id, name, price, category.
    """
    return db.search(query, limit=max_results)

@mcp.resource("catalog://categories")
def get_categories() -> str:
    """List all product categories."""
    return json.dumps(db.get_all_categories())

@mcp.prompt()
def product_review_prompt(product_id: str) -> str:
    """Template for generating product reviews."""
    product = db.get_product(product_id)
    return f"Write a balanced review for: {product['name']}\nSpecs: {product['specs']}"

if __name__ == "__main__":
    mcp.run()  # stdio transport by default
```

### MCP Client (Python)

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

server_params = StdioServerParameters(
    command="python",
    args=["my_mcp_server.py"]
)

async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()

        # Discover available tools
        tools = await session.list_tools()

        # Call a tool
        result = await session.call_tool(
            "search_products",
            arguments={"query": "wireless headphones", "max_results": 5}
        )

        # Read a resource
        content, mime = await session.read_resource("catalog://categories")
```

### MCP Server Config (claude_desktop_config.json)

```json
{
  "mcpServers": {
    "my-database": {
      "command": "python",
      "args": ["servers/db_server.py"],
      "env": { "DB_URL": "postgresql://localhost/mydb" }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    },
    "remote-api": {
      "url": "https://api.example.com/mcp/sse",
      "headers": { "Authorization": "Bearer ${API_KEY}" }
    }
  }
}
```

### MCP Security Checklist

- [ ] Authenticate every server connection (API key, OAuth, mTLS)
- [ ] Scope tool permissions (read-only vs. write)
- [ ] Validate and sanitize all tool inputs server-side
- [ ] Rate-limit tool invocations per session
- [ ] Log all tool calls with full arguments for audit
- [ ] Sandbox code execution tools (Docker, gVisor)
- [ ] Never expose secrets through tool outputs

---

## 7. Skills & Skill Design

A **Skill** is a self-contained, reusable capability unit composed of:
- A SKILL.md instruction file (read by the agent at runtime)
- Tool definitions specific to the skill
- Example inputs/outputs (few-shot)
- Validation / eval cases

### SKILL.md Template

```markdown
---
name: web-researcher
description: >
  Searches the web for current information and synthesizes findings
  into structured reports. Use when the task requires up-to-date
  data, fact-checking, or competitive intelligence.
version: 1.2.0
author: platform-team
tools: [web_search, web_fetch, summarize]
---

# Web Researcher Skill

## Overview
Retrieve and synthesize information from multiple web sources into
a factual, cited report.

## Instructions
1. Decompose the research question into 3–5 targeted search queries.
2. Execute searches in parallel where possible.
3. Fetch the top 2 sources per query for full content.
4. Cross-reference facts across sources; flag conflicts.
5. Cite every factual claim with [source: URL].
6. Never hallucinate citations — only cite pages you fetched.

## Output Format
```json
{
  "summary": "...",
  "key_findings": ["...", "..."],
  "sources": [{"url": "...", "title": "...", "relevance": "..."}],
  "confidence": "high|medium|low",
  "gaps": ["unanswered questions"]
}
```

## Examples
### Input
"What are the current LLM context window limits for frontier models?"

### Output
{"summary": "As of 2025, frontier models offer...", ...}

## Evaluation Criteria
- All cited URLs must return HTTP 200
- Minimum 3 distinct sources
- No contradictory facts without explicit flagging
- JSON schema validates
```

### Skill Registry Pattern

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class Skill:
    name: str
    description: str
    tools: list[str]
    instruction_path: str
    examples: list[dict]
    validator: Callable | None = None

class SkillRegistry:
    def __init__(self):
        self._registry: dict[str, Skill] = {}

    def register(self, skill: Skill):
        self._registry[skill.name] = skill

    def get(self, name: str) -> Skill:
        return self._registry[name]

    def match(self, task_description: str) -> list[Skill]:
        """Return skills relevant to the task (semantic match)."""
        return semantic_search(task_description, list(self._registry.values()))

    def load_instructions(self, name: str) -> str:
        skill = self.get(name)
        return Path(skill.instruction_path).read_text()
```

---

## 8. Multi-Agent Systems

### Communication Patterns

| Pattern | Description | When to Use |
|---|---|---|
| **Direct call** | Agent A calls Agent B and awaits response | Simple delegation |
| **Message queue** | Agents communicate via async queue | Decoupled, high-throughput |
| **Shared state** | Agents read/write common state store | Collaborative editing |
| **Broadcast** | One agent publishes; many subscribe | Event-driven pipelines |
| **Negotiation** | Agents bid/propose until consensus | Resource allocation |

### Agent Roles

| Role | Responsibility |
|---|---|
| **Planner** | Decomposes high-level goal into subtasks |
| **Router** | Selects the appropriate agent/skill for each task |
| **Executor** | Carries out a specific subtask with tools |
| **Critic/Reviewer** | Evaluates outputs; requests revisions |
| **Aggregator** | Merges multiple agent outputs into one coherent result |
| **Monitor** | Watches for errors, timeouts, policy violations |
| **Memory Manager** | Curates and retrieves relevant memories |

### Multi-Agent Orchestration (Python)

```python
from anthropic import Anthropic
import asyncio

client = Anthropic()

async def run_agent(role: str, system: str, task: str, tools: list) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=4096,
        system=system,
        tools=tools,
        messages=[{"role": "user", "content": task}]
    )
    return extract_text(response)

async def orchestrate(user_goal: str):
    # Step 1: Plan
    plan = await run_agent(
        role="planner",
        system=PLANNER_SYSTEM,
        task=user_goal,
        tools=[]
    )
    subtasks = parse_plan(plan)

    # Step 2: Execute in parallel where possible
    results = await asyncio.gather(*[
        run_agent("executor", EXECUTOR_SYSTEM, task, EXECUTOR_TOOLS)
        for task in subtasks
    ])

    # Step 3: Critic review
    critique = await run_agent(
        role="critic",
        system=CRITIC_SYSTEM,
        task=f"Goal: {user_goal}\nResults: {results}",
        tools=[]
    )

    # Step 4: Aggregate
    if needs_revision(critique):
        return await orchestrate_revision(user_goal, results, critique)

    return await run_agent("aggregator", AGGREGATOR_SYSTEM,
                           f"Combine: {results}", tools=[])
```

### LangGraph Multi-Agent (State Machine)

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]
    plan: list[str]
    results: list[str]
    current_step: int
    critique: str
    final_output: str

def planner_node(state: AgentState) -> AgentState:
    plan = planner_llm(state["messages"][-1]["content"])
    return {"plan": plan, "current_step": 0}

def executor_node(state: AgentState) -> AgentState:
    step = state["plan"][state["current_step"]]
    result = executor_llm(step)
    return {
        "results": state["results"] + [result],
        "current_step": state["current_step"] + 1
    }

def critic_node(state: AgentState) -> AgentState:
    critique = critic_llm(state["results"])
    return {"critique": critique}

def route_after_critic(state: AgentState) -> str:
    if "APPROVED" in state["critique"]:
        return "aggregate"
    return "executor"  # revise

builder = StateGraph(AgentState)
builder.add_node("planner", planner_node)
builder.add_node("executor", executor_node)
builder.add_node("critic", critic_node)
builder.add_node("aggregate", aggregate_node)

builder.set_entry_point("planner")
builder.add_edge("planner", "executor")
builder.add_conditional_edges("executor",
    lambda s: "critic" if s["current_step"] >= len(s["plan"]) else "executor",
    {"critic": "critic", "executor": "executor"})
builder.add_conditional_edges("critic", route_after_critic,
    {"aggregate": "aggregate", "executor": "executor"})
builder.add_edge("aggregate", END)

graph = builder.compile()
```

---

## 9. Memory Systems

```
┌────────────────────────────────────────────────────────────┐
│                    AGENT MEMORY TAXONOMY                   │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │  In-Context  │  │  External    │  │  Parametric  │    │
│  │  (Working)   │  │  (Episodic / │  │  (Fine-tuned │    │
│  │              │  │   Semantic)  │  │   weights)   │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
│  • Conversation    • Vector DB        • LoRA / QLoRA      │
│  • Scratchpad      • KV store         • Distillation      │
│  • Tool results    • SQL / Graph DB   • Continual         │
│  • Retrieved docs  • File system        learning          │
└────────────────────────────────────────────────────────────┘
```

### Memory Types

| Type | Storage | Retrieval | TTL | Use Case |
|---|---|---|---|---|
| **Working** | Context window | Implicit | Session | Ongoing task state |
| **Episodic** | Vector DB | Semantic similarity | Long-term | Past conversations, events |
| **Semantic** | Vector DB / KG | Embedding search | Permanent | Facts, domain knowledge |
| **Procedural** | SKILL.md / code | Lookup | Permanent | How to do things |
| **Cache** | Redis / KV | Exact key | TTL | Repeated queries, tool results |

### Memory Operations

```python
class AgentMemory:
    def __init__(self, vector_store, kv_store):
        self.vector = vector_store   # e.g. ChromaDB, Pinecone, pgvector
        self.kv = kv_store           # e.g. Redis

    def store(self, content: str, metadata: dict, namespace: str = "default"):
        embedding = embed(content)
        self.vector.upsert(
            id=generate_id(),
            embedding=embedding,
            metadata={"content": content, "namespace": namespace, **metadata}
        )

    def retrieve(self, query: str, namespace: str = "default",
                 top_k: int = 5, min_score: float = 0.7) -> list[str]:
        results = self.vector.query(
            embedding=embed(query),
            filter={"namespace": namespace},
            top_k=top_k
        )
        return [r.metadata["content"] for r in results if r.score >= min_score]

    def forget(self, memory_id: str):
        self.vector.delete(memory_id)

    def summarize_and_compress(self, memories: list[str]) -> str:
        """Compress many memories into a summary to save context space."""
        return summarizer_llm("\n".join(memories))
```

### Memory Injection Pattern

```python
def build_system_prompt(user_id: str, task: str) -> str:
    memories = memory.retrieve(task, namespace=user_id, top_k=3)
    memory_block = "\n".join(f"- {m}" for m in memories) if memories else "None"

    return f"""You are a helpful assistant with memory of past interactions.

## Relevant Memory
{memory_block}

## Instructions
Use your memory to personalize responses. Update memory after each interaction.
"""
```

---

## 10. Context Window Management

### Token Budget Strategy

```
┌──────────────────────────────────────────────────────────┐
│                  CONTEXT WINDOW BUDGET                   │
│                                                          │
│  System Prompt       ████░░░░░░░░░░░░  ~10–20%          │
│  Retrieved Memory    ██░░░░░░░░░░░░░░  ~5–10%           │
│  Tool Schemas        ███░░░░░░░░░░░░░  ~5–15%           │
│  Conversation Hist   ████████░░░░░░░░  ~20–30%          │
│  Current Task/Docs   ████████░░░░░░░░  ~20–30%          │
│  Scratchpad Reserve  ████░░░░░░░░░░░░  ~10–20%          │
│  Output Buffer       ██░░░░░░░░░░░░░░  ~10%             │
└──────────────────────────────────────────────────────────┘
```

### Techniques

| Technique | Description |
|---|---|
| **Sliding window** | Keep only last N turns; compress older history |
| **Hierarchical summarization** | Summarize old conversation chunks; retain summaries |
| **Selective retrieval** | Only inject memories/docs relevant to current query |
| **Chunking** | Split large documents into overlapping chunks |
| **Packing** | Combine short documents into one context request |
| **Context caching** | Cache static system prompt + docs (Anthropic Prompt Caching) |
| **Document reranking** | Score retrieved chunks by relevance; keep top-K only |

### Prompt Caching (Anthropic)

```python
# Cache static content at the "breakpoint"
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": LONG_STATIC_SYSTEM_PROMPT,
            "cache_control": {"type": "ephemeral"}  # ← Cache this block
        }
    ],
    messages=[{"role": "user", "content": user_query}]
)
# Subsequent calls with same cached content: ~90% token cost reduction
```

---

## 11. Prompt Engineering for Agents

### System Prompt Structure

```
[IDENTITY]          Who the agent is, its role and persona.
[CONTEXT]           What it knows: domain, tools, environment.
[INSTRUCTIONS]      Step-by-step task guidance, reasoning style.
[TOOLS]             Available tools and when to use each.
[OUTPUT FORMAT]     Expected response schema / format.
[CONSTRAINTS]       What the agent must never do.
[EXAMPLES]          1–3 few-shot demonstrations.
```

### Sample Agent System Prompt

```
You are ARIA, an autonomous research intelligence agent.

## Identity
You are a meticulous, evidence-based researcher. You never speculate
without flagging uncertainty. You prefer cited facts over opinion.

## Instructions
1. Read the task carefully before acting.
2. Decompose complex tasks into sub-questions.
3. Search before answering factual questions.
4. Cross-validate findings with at least 2 sources.
5. Reflect on your answer before finalizing it.
6. If stuck for more than 3 tool calls, ask for clarification.

## Tool Usage
- web_search: for current events, statistics, any fact post-2023.
- vector_search: for internal knowledge base documents.
- python_repl: for calculations, data processing, chart generation.
- NEVER call a tool with fabricated arguments.

## Output Format
Always respond in valid JSON:
{"answer": "...", "confidence": "high|medium|low",
 "sources": ["url1", "url2"], "caveats": ["..."]}

## Constraints
- Never claim certainty when sources conflict.
- Never include PII in outputs or logs.
- Never execute shell commands outside the sandbox.
- Stop and report if a tool returns an error 3 times in a row.
```

### Prompt Patterns for Agents

```markdown
## Decomposition Prompt
"Break this task into clear, ordered subtasks. Number them.
Each subtask must be completable with one tool call."

## Verification Prompt
"Before providing your final answer, verify each key claim by
re-reading the tool outputs. Flag any inconsistencies."

## Self-Critique Prompt
"Review your proposed action. Would a cautious expert endorse it?
If not, what is the safer alternative?"

## Uncertainty Prompt
"Rate your confidence (1–10) and list the top 3 things that
could make this answer wrong."

## Tool Selection Prompt
"Given the available tools, which is the most efficient path?
Prefer fewer tool calls when possible."
```

---

## 12. Agentic Workflows In Depth

### 12.1 Human-in-the-Loop Checkpoints

```python
from enum import Enum

class InterruptType(Enum):
    APPROVAL = "approval"          # Must get yes/no before proceeding
    INFORMATION = "information"    # Human provides missing data
    CORRECTION = "correction"      # Human fixes agent error
    REVIEW = "review"              # Human reviews output quality

@dataclass
class Interrupt:
    type: InterruptType
    message: str
    context: dict
    timeout_seconds: int = 300
    default_on_timeout: str = "abort"  # abort | proceed | retry

class HumanInTheLoop:
    async def checkpoint(self, interrupt: Interrupt) -> str:
        """Pause workflow and await human response."""
        await self.notify_human(interrupt)
        response = await asyncio.wait_for(
            self.await_human_response(),
            timeout=interrupt.timeout_seconds
        )
        return response or interrupt.default_on_timeout
```

### 12.2 Error Handling & Retry Logic

```python
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

class AgentExecutor:
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=4, max=60)
    )
    async def call_tool(self, tool_name: str, args: dict) -> dict:
        try:
            result = await self.tool_registry[tool_name](**args)
            self.validate_result(result)
            return result
        except ToolAuthError:
            raise  # Don't retry auth errors
        except ToolRateLimitError as e:
            await asyncio.sleep(e.retry_after)
            raise
        except ToolTimeoutError:
            # Log and raise for retry
            self.logger.warning(f"Tool {tool_name} timed out")
            raise

    def handle_max_retries(self, tool_name: str, error: Exception):
        """Called when all retries exhausted."""
        if self.has_fallback(tool_name):
            return self.call_fallback(tool_name)
        return self.request_human_intervention(tool_name, error)
```

### 12.3 Streaming Agent Responses

```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    system=AGENT_SYSTEM,
    tools=TOOLS,
    messages=messages
) as stream:
    for event in stream:
        if event.type == "content_block_delta":
            if hasattr(event.delta, "text"):
                yield ("text", event.delta.text)
            elif hasattr(event.delta, "partial_json"):
                yield ("tool_partial", event.delta.partial_json)

        elif event.type == "content_block_stop":
            if current_block.type == "tool_use":
                result = await execute_tool(current_block)
                yield ("tool_result", result)
```

### 12.4 Long-Running Task Management

```python
import uuid
from datetime import datetime

class TaskManager:
    def __init__(self, db, agent):
        self.db = db
        self.agent = agent

    async def submit_task(self, user_id: str, task: str) -> str:
        task_id = str(uuid.uuid4())
        await self.db.create_task({
            "id": task_id,
            "user_id": user_id,
            "task": task,
            "status": "queued",
            "created_at": datetime.utcnow().isoformat()
        })
        asyncio.create_task(self._run_task(task_id, task))
        return task_id

    async def _run_task(self, task_id: str, task: str):
        try:
            await self.db.update_task(task_id, {"status": "running"})
            result = await self.agent.run(task, task_id=task_id)
            await self.db.update_task(task_id, {
                "status": "completed",
                "result": result,
                "completed_at": datetime.utcnow().isoformat()
            })
        except Exception as e:
            await self.db.update_task(task_id, {
                "status": "failed",
                "error": str(e)
            })

    async def get_status(self, task_id: str) -> dict:
        return await self.db.get_task(task_id)

    async def cancel_task(self, task_id: str):
        await self.db.update_task(task_id, {"status": "cancelled"})
        # Signal running coroutine via cancellation token
```

---

## 13. Evaluation & Observability

### Evaluation Dimensions

| Dimension | Metric | How to Measure |
|---|---|---|
| **Task success** | Pass@k, exact match, F1 | Automated test suite |
| **Tool use correctness** | Tool selection accuracy | Ground-truth tool traces |
| **Efficiency** | Steps to success, token count | Trajectory logging |
| **Safety** | Refusal rate on unsafe inputs | Red-team eval set |
| **Hallucination** | Factuality score | GPT-4 judge or human eval |
| **Latency** | P50/P95 wall-clock time | Tracing (OpenTelemetry) |
| **Cost** | $ per successful task | Token billing + tool costs |
| **Consistency** | Variance across N runs | Run same task 5–10 times |

### Observability Stack

```
Agent ──▶ OpenTelemetry SDK ──▶ Collector ──▶ ┌─ Jaeger (traces)
                                               ├─ Prometheus (metrics)
                                               └─ Loki (logs)

Key spans to instrument:
  • LLM inference (model, tokens, latency)
  • Tool invocation (name, args, result, latency)
  • Memory read/write (namespace, query, results)
  • Agent loop iteration (step #, action taken)
  • Human interrupt (type, wait time, response)
```

### LangSmith / LangFuse Tracing Pattern

```python
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context

langfuse = Langfuse()

@observe(name="agent-run")
async def run_agent(task: str) -> str:
    langfuse_context.update_current_trace(
        name="Research Agent",
        user_id=user_id,
        metadata={"task_type": classify_task(task)}
    )

    @observe(name="llm-call")
    def call_llm(messages):
        response = client.messages.create(...)
        langfuse_context.update_current_observation(
            model="claude-sonnet-4-6",
            usage={"input": response.usage.input_tokens,
                   "output": response.usage.output_tokens}
        )
        return response

    @observe(name="tool-call")
    def call_tool(name, args):
        return tool_registry[name](**args)

    return await agent_loop(task, call_llm, call_tool)
```

### Automated Evaluation (LLM-as-Judge)

```python
JUDGE_PROMPT = """Evaluate the agent's response to the task.

Task: {task}
Agent Response: {response}
Ground Truth: {ground_truth}

Score each dimension 1–5:
- Correctness: Does it answer the task accurately?
- Completeness: Are all parts of the task addressed?
- Conciseness: Is it free of unnecessary content?
- Tool Use: Were tools used appropriately and efficiently?

Respond ONLY with JSON:
{{"correctness": N, "completeness": N, "conciseness": N,
  "tool_use": N, "reasoning": "..."}}"""

def judge_response(task, response, ground_truth) -> dict:
    result = judge_llm(JUDGE_PROMPT.format(
        task=task, response=response, ground_truth=ground_truth
    ))
    return json.loads(result)
```

---

## 14. Safety & Guardrails

### Input Guardrails

```python
class InputGuardrail:
    def check(self, user_input: str) -> GuardrailResult:
        checks = [
            self.check_prompt_injection(user_input),
            self.check_pii(user_input),
            self.check_policy_violations(user_input),
            self.check_jailbreak_patterns(user_input),
        ]
        violations = [c for c in checks if not c.passed]
        return GuardrailResult(
            passed=len(violations) == 0,
            violations=violations,
            sanitized_input=self.sanitize(user_input)
        )
```

### Output Guardrails

```python
class OutputGuardrail:
    def check(self, agent_output: str, tool_calls: list) -> GuardrailResult:
        return GuardrailResult(
            passed=all([
                not self.contains_pii(agent_output),
                not self.contains_harmful_content(agent_output),
                self.validate_tool_calls(tool_calls),
                not self.exceeds_scope(agent_output),
            ])
        )
```

### Tool Permission Levels

```python
TOOL_PERMISSIONS = {
    # Read-only operations — always allowed
    "web_search":      PermissionLevel.AUTO,
    "vector_search":   PermissionLevel.AUTO,
    "read_file":       PermissionLevel.AUTO,

    # Write operations — require confirmation
    "write_file":      PermissionLevel.CONFIRM,
    "send_email":      PermissionLevel.CONFIRM,
    "post_to_slack":   PermissionLevel.CONFIRM,
    "db_write":        PermissionLevel.CONFIRM,

    # Destructive operations — require explicit approval
    "delete_file":     PermissionLevel.APPROVE,
    "drop_table":      PermissionLevel.APPROVE,
    "deploy":          PermissionLevel.APPROVE,

    # Blocked in production
    "shell_exec":      PermissionLevel.BLOCKED,
}
```

### Minimal Footprint Principle

```
✅ DO:
  - Request only permissions needed for the current task
  - Prefer reversible actions (write draft → review → send)
  - Store only data required for task completion
  - Confirm before irreversible actions

❌ DON'T:
  - Cache credentials beyond the session
  - Request broad scopes "just in case"
  - Execute code without sandbox containment
  - Take side-effecting actions in parallel without ordering guarantees
```

---

## 15. Sample Project Structures

### 15.1 Single-Agent Application

```
my-agent/
├── agent/
│   ├── __init__.py
│   ├── agent.py           # Main agent class with loop
│   ├── prompts.py         # System prompts, templates
│   └── tools/
│       ├── __init__.py
│       ├── search.py      # web_search, vector_search
│       ├── code.py        # python_repl, bash
│       └── files.py       # read_file, write_file
├── memory/
│   ├── vector_store.py    # ChromaDB / pgvector wrapper
│   └── kv_store.py        # Redis wrapper
├── skills/
│   ├── researcher/
│   │   ├── SKILL.md
│   │   └── examples.json
│   └── coder/
│       ├── SKILL.md
│       └── examples.json
├── guardrails/
│   ├── input_guard.py
│   └── output_guard.py
├── api/
│   └── main.py            # FastAPI endpoint
├── eval/
│   ├── test_cases.json
│   └── run_eval.py
├── config/
│   └── settings.py        # Model, tool, budget config
├── pyproject.toml
└── README.md
```

### 15.2 Multi-Agent System

```
multi-agent-system/
├── agents/
│   ├── planner/
│   │   ├── agent.py
│   │   └── prompts.py
│   ├── researcher/
│   │   ├── agent.py
│   │   └── prompts.py
│   ├── coder/
│   │   ├── agent.py
│   │   └── prompts.py
│   └── critic/
│       ├── agent.py
│       └── prompts.py
├── orchestrator/
│   ├── graph.py           # LangGraph state machine
│   ├── router.py          # Task → agent assignment
│   └── state.py           # Shared state schema
├── mcp_servers/
│   ├── database/
│   │   ├── server.py      # FastMCP database server
│   │   └── schema.sql
│   ├── filesystem/
│   │   └── server.py
│   └── external_api/
│       └── server.py
├── shared/
│   ├── memory.py          # Shared memory store
│   ├── tools.py           # Common tool registry
│   └── schemas.py         # Pydantic models
├── monitoring/
│   ├── tracer.py          # OpenTelemetry setup
│   └── dashboard.py       # Metrics dashboard
├── hitl/
│   ├── checkpoint.py      # Interrupt points
│   └── ui/                # Human review interface
├── eval/
│   ├── benchmarks/
│   └── run_eval.py
├── docker-compose.yml     # Postgres, Redis, Chroma
├── mcp_config.json        # MCP server manifest
└── pyproject.toml
```

### 15.3 MCP Server Project

```
my-mcp-server/
├── src/
│   ├── server.py          # FastMCP main server
│   ├── tools/
│   │   ├── query.py       # Tool implementations
│   │   ├── write.py
│   │   └── admin.py
│   ├── resources/
│   │   └── catalog.py     # Resource providers
│   ├── prompts/
│   │   └── templates.py   # Prompt templates
│   └── auth/
│       ├── middleware.py   # Auth validation
│       └── scopes.py      # Permission definitions
├── tests/
│   ├── test_tools.py
│   └── test_resources.py
├── deployment/
│   ├── Dockerfile
│   └── helm/              # K8s Helm chart
├── docs/
│   └── TOOLS.md           # Tool documentation for agents
├── pyproject.toml
└── mcp_manifest.json      # Discoverable server metadata
```

---

## 16. Framework Comparison

| Framework | Language | Style | Strengths | Weaknesses |
|---|---|---|---|---|
| **LangChain** | Python/JS | Modular chains | Huge ecosystem, RAG support | Abstraction complexity |
| **LangGraph** | Python | State machine (DAG) | Robust multi-agent, HITL, streaming | Steeper learning curve |
| **AutoGen** | Python | Conversational agents | Multi-agent chat, code exec | Less structured control |
| **CrewAI** | Python | Role-based crew | Easy role definition, good DX | Limited customization |
| **Swarm (OpenAI)** | Python | Lightweight handoff | Simple, educational | Not production-ready |
| **Semantic Kernel** | C#/Python | Plugin-based | Enterprise (.NET), Azure native | Python port lags |
| **DSPy** | Python | Programmatic prompting | Prompt optimization, modular | Less agentic focus |
| **Haystack** | Python | Pipeline-first | RAG & search, production-ready | Agent capabilities growing |
| **Claude Code** | TypeScript | Agentic coding | Native tool use, computer use | Claude-specific |

---

## 17. Methods & API Reference Table

### Anthropic Messages API — Agent-Relevant Parameters

| Parameter | Type | Description |
|---|---|---|
| `model` | `str` | Model ID (e.g., `claude-sonnet-4-6`) |
| `max_tokens` | `int` | Maximum output tokens (required) |
| `system` | `str \| list` | System prompt or blocks with cache_control |
| `messages` | `list[dict]` | Conversation history (role/content pairs) |
| `tools` | `list[dict]` | Tool schemas available to the model |
| `tool_choice` | `dict` | `auto` / `any` / `tool` (force specific tool) |
| `temperature` | `float` | 0.0–1.0; lower = more deterministic |
| `top_p` | `float` | Nucleus sampling; alternative to temperature |
| `stop_sequences` | `list[str]` | Sequences that halt generation |
| `stream` | `bool` | Enable streaming response |
| `metadata` | `dict` | `user_id`, custom fields for logging |
| `betas` | `list[str]` | Enable beta features (e.g., `computer-use-2024-10-22`) |

### Tool Definition Fields

| Field | Required | Description |
|---|---|---|
| `name` | ✅ | Snake_case unique identifier |
| `description` | ✅ | When and how to use this tool (agent reads this) |
| `input_schema` | ✅ | JSON Schema of parameters |
| `input_schema.type` | ✅ | Always `"object"` |
| `input_schema.properties` | ✅ | Parameter definitions |
| `input_schema.required` | ⭕ | List of required parameter names |

### Response Content Block Types

| Type | When | Key Fields |
|---|---|---|
| `text` | Normal response | `.text` |
| `tool_use` | Agent calls a tool | `.id`, `.name`, `.input` |
| `tool_result` | User injects tool response | `.tool_use_id`, `.content`, `.is_error` |
| `thinking` | Extended thinking enabled | `.thinking` (visible CoT) |

### LangChain Core Methods

| Method | Class | Description |
|---|---|---|
| `invoke(input)` | Any Runnable | Synchronous single call |
| `ainvoke(input)` | Any Runnable | Async single call |
| `stream(input)` | Any Runnable | Streaming generator |
| `batch(inputs)` | Any Runnable | Parallel batch processing |
| `bind_tools(tools)` | ChatModel | Attach tool schemas to model |
| `with_structured_output(schema)` | ChatModel | Force JSON schema output |
| `from_messages(template)` | ChatPromptTemplate | Build prompt from messages |
| `add_messages(messages)` | StateGraph | LangGraph state updater |
| `add_node(name, fn)` | StateGraph | Register agent node |
| `add_edge(from, to)` | StateGraph | Fixed transition |
| `add_conditional_edges(from, fn, map)` | StateGraph | Conditional routing |
| `compile()` | StateGraph | Build executable graph |
| `astream_events(input)` | Compiled Graph | Stream all graph events |

### MCP SDK Methods

| Method | Description |
|---|---|
| `session.initialize()` | Negotiate capabilities with MCP server |
| `session.list_tools()` | Discover available tools |
| `session.call_tool(name, arguments)` | Invoke a tool |
| `session.list_resources()` | Discover available resources |
| `session.read_resource(uri)` | Fetch resource content |
| `session.list_prompts()` | Discover prompt templates |
| `session.get_prompt(name, args)` | Retrieve a prompt template |
| `mcp.tool()` | Decorator: register a tool (server-side) |
| `mcp.resource(uri_template)` | Decorator: register a resource (server-side) |
| `mcp.prompt()` | Decorator: register a prompt (server-side) |
| `mcp.run(transport)` | Start the MCP server |

---

## 18. Design Patterns Catalogue

### Pattern 1: Router

```
Input ──▶ Router LLM ──▶ [classify intent]
               ├── "code"    ──▶ Code Agent
               ├── "search"  ──▶ Research Agent
               ├── "data"    ──▶ SQL Agent
               └── "general" ──▶ General Agent

Router prompt: "Classify the user's intent into one of:
[code, search, data, general]. Respond with ONLY the class name."
```

### Pattern 2: Fallback Chain

```
Primary LLM call ──[failure/low confidence]──▶ Fallback LLM ──▶ Human
                                                    │
                                              [still fails]
                                                    │
                                             Return error + context
```

### Pattern 3: Validation Loop

```
Generator ──▶ Validator ──▶ [pass] ──▶ Output
     ▲               │
     │          [fail + reason]
     └───────────────┘
     (max N iterations)
```

### Pattern 4: Speculative Execution

```
Multiple agents run same task with different approaches in parallel
          │           │           │
      Approach A  Approach B  Approach C
          └───────────┴───────────┘
                      │
               [Select best result]
                      │
                 Final Output

Best for: tasks where quality > latency, exploration needed
```

### Pattern 5: Checkpoint-Resume

```
Task ──▶ Step 1 ──[checkpoint]──▶ Step 2 ──[checkpoint]──▶ Step 3
              │ (persist state)          │ (persist state)
              │                          │
         [crash/interrupt]          [crash/interrupt]
              │                          │
         Resume from                Resume from
         Step 1 state               Step 2 state
```

### Pattern 6: Tool Result Caching

```python
@lru_cache(maxsize=256)
def cached_web_search(query: str, date: str) -> str:
    """Cache search results; include date to invalidate daily."""
    return real_web_search(query)

# Or use Redis with TTL for distributed agents:
def cached_tool_call(tool_name: str, args_hash: str, ttl: int = 3600):
    cached = redis.get(f"tool:{tool_name}:{args_hash}")
    if cached:
        return json.loads(cached)
    result = call_tool(tool_name, ...)
    redis.setex(f"tool:{tool_name}:{args_hash}", ttl, json.dumps(result))
    return result
```

### Pattern 7: Constitutional AI (Output Filtering)

```
Raw Agent Output
      │
      ▼
Critique Agent ──▶ "Does this violate: safety? accuracy? scope?"
      │
      ▼ (if violation)
Revision Agent ──▶ Fixed Output
      │
      ▼
Final Output (guaranteed policy-compliant)
```

---

## 19. Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| **Infinite loop** | Agent calls tool → result triggers same tool | Max step limit + loop detection |
| **Tool spam** | Agent calls irrelevant tools "just in case" | Strict tool selection prompts |
| **Context stuffing** | Injecting entire database into context | Semantic retrieval; top-K only |
| **Trusting tool output** | Agent echoes tool results without validation | Validate + cross-reference |
| **Sycophantic planner** | Planner agrees with every user suggestion | Critic agent or adversarial prompt |
| **God agent** | One agent does everything | Split by domain; use multi-agent |
| **No error budget** | Agent retries indefinitely | Max retries + exponential backoff |
| **Silent failure** | Tool fails; agent invents result | Check tool success; halt on error |
| **Prompt injection** | User input overrides system prompt | Input sanitization + validation |
| **Overly broad permissions** | Agent has delete/deploy access always | Least-privilege; scoped tokens |
| **No HITL for irreversible** | Agent auto-deploys or auto-deletes | Mandatory approval for destructive ops |
| **Memory without expiry** | Stale memories mislead agent | TTL on episodic memory |
| **Hardcoded routing** | Router cannot handle new tasks | Dynamic classification or fallback |

---

## 20. Best Practices

### Architecture

- Start with the simplest architecture that solves the problem; add complexity only when needed.
- Define clear interfaces between agents (Pydantic schemas); don't pass raw strings.
- Design for idempotency: every step should be safe to repeat after a crash.
- Keep agents stateless where possible; externalize all state.

### Prompting

- Write system prompts as instruction manuals, not vague role descriptions.
- Include 1–3 concrete examples in system prompts (few-shot beats instructions alone).
- Specify exact output schemas; validate with Pydantic or JSON Schema before processing.
- Use explicit "stop" conditions in your prompts to prevent runaway loops.

### Tool Design

- Every tool must have a precise, honest description — the agent's tool selection is only as good as the descriptions.
- Make tools idempotent and atomic where possible.
- Return structured errors, not raw exceptions — the agent must be able to reason about failures.
- Log every tool call with arguments, latency, and result for debugging.

### Safety

- Apply minimal footprint principle: prefer read-only, reversible, scoped actions.
- Mandate HITL checkpoints before any destructive, public, or financial action.
- Implement both input AND output guardrails; don't rely on model refusals alone.
- Red-team your agent with adversarial inputs before production deployment.

### Evaluation

- Build your eval suite before building the agent (test-driven agent development).
- Use diverse test sets: happy path + edge cases + adversarial + regression.
- Track efficiency metrics (steps, tokens, cost) alongside quality metrics.
- Run periodic re-evals after model updates or prompt changes.

### Production Operations

- Instrument every agent loop iteration with distributed tracing (OpenTelemetry).
- Set token and cost budgets per task; fail fast if exceeded.
- Implement circuit breakers for tool dependencies (external APIs).
- Version your prompts and skills; never edit in place without a rollback path.
- Monitor for prompt drift — model behavior changes between updates even with the same prompt.

---

*Last updated: June 2026 — covers Anthropic Messages API, MCP 1.x, LangGraph 0.2+, LangChain 0.3+*
