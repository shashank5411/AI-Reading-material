# Agentic Systems — Personal Reference
*10-minute refresher. Built from first principles, anchored to a fintech / data engineering context.*

---

## The One-Line Mental Model

> An agent is an LLM that can decide what actions to take, execute those actions, observe the results, and loop until the task is complete.

RAG gives an LLM knowledge. An agent gives it **agency** — the ability to decide what knowledge to fetch, in what order, and what to do when the first answer raises new questions.

---

## Why Agents Exist — The Problem RAG Alone Can't Solve

Single-shot RAG is a linear pipeline: retrieve → generate → done. It breaks the moment a task requires:

- Each step depending on the result of the previous step
- Unknown depth — you don't know how many hops the investigation needs
- Dynamic decisions — should I check pipeline logs or is the score pattern sufficient?
- Iteration — the first answer raises a new question

A human DQ investigator doesn't work linearly. They pull data, read it, form a hypothesis, test it, revise it, pull more data. That's an agent loop.

---

## The ReAct Loop — The Foundation of Every Agent

Every agent framework is built on one core pattern:

```
Thought:     What do I know? What do I need to find out?
Action:      Call a tool with specific inputs
Observation: Read the tool's output
Thought:     What does this tell me? What's still missing?
Action:      Call another tool
Observation: Read the output
             ... repeat until ...
Thought:     I have enough to answer.
Answer:      Final response
```

The LLM generates thoughts (reasoning), actions (tool calls), and reads observations (tool results). The loop continues until the LLM decides it's done.

**Cypher analogy for your world:** The alert fires → agent thinks → fetches score history → reads it → thinks → fetches pipeline logs → reads them → thinks → fetches owner → synthesizes. Each fetch is informed by what the previous fetch revealed. That's ReAct.

---

## The System Prompt — Where You Instill the Agent's Intelligence

The system prompt does three jobs simultaneously:

**1. Role and framing** — defines the lens through which the agent interprets everything.
```
You are a senior DQ engineer investigating anomalies in a 
fintech data pipeline. Never accept the first explanation — 
always check whether upstream data supports it.
```

**2. Methodology — your domain expertise made executable**
```
Investigate in this order:
1. Identify which sub-score drove the composite drop
2. Check if correlated dimensions exist
3. Form a hypothesis before checking pipeline logs
4. Verify hypothesis against pipeline logs
5. Confirm team ownership from registry — never infer from naming
6. State confidence level and what evidence would change it
```

**3. Output format — reduces variance in behavior**
```
Structure final answer as:
- Root cause (one sentence)
- Supporting evidence (bullet points with sources)
- Owning team and contact
- Confidence level (high/medium/low) and why
- Recommended action
```

**The fundamental principle:** The agent's domain intelligence is yours. The agent's reasoning capability is the LLM's. The system prompt is where they meet. A weak system prompt with a strong LLM produces inconsistent results. A strong system prompt with a capable LLM produces reliable production behavior.

---

## Agent Flavors

Every flavor exists because a specific problem made the simpler one insufficient.

### ReAct — the baseline
**Pattern:** Think → Act → Observe → repeat. No upfront plan.
**Good for:** Investigative tasks where the path depends on findings. Your DQ anomaly agent.
**Bad for:** Complex parallel workstreams — sequential by nature.

---

### Plan-and-Execute
**Pattern:** Phase 1: LLM generates a complete step-by-step plan. Phase 2: execute each step, revise plan if results change it.
**Good for:** Tasks where you want visibility before execution. Compliance, audit contexts, predictable workflows.
**Bad for:** Tasks where the plan genuinely can't be formed upfront — forcing a plan produces a brittle agent that can't adapt.

---

### Tool-use Agent
**Pattern:** Agent has a large diverse toolset. Tool selection itself is the core reasoning challenge.
**Good for:** Integration-heavy workflows touching multiple systems. Your MCP server setup.
**Bad for:** Simple tasks where a large toolset creates unnecessary decision overhead.
**Critical detail:** Tool descriptions in the system prompt directly determine selection quality. Vague descriptions → wrong tool selected → wrong results.

---

### Multi-agent (Orchestrator + Sub-agents)
**Pattern:** Orchestrator receives task, decomposes it, delegates to specialized sub-agents, synthesizes results.

```
Orchestrator: DQ anomaly agent
  └── Score Analyzer sub-agent     (PSI/Z-score expert, ReAct loop)
  └── Pipeline Investigator         (pipeline ops expert, Plan-and-Execute)
  └── Ownership Resolver            (deterministic lookup, single-shot)
  └── Incident History              (pattern search, hybrid RAG)
```

**Good for:** Complex tasks that decompose into independent sub-problems. Anything benefiting from specialization.
**Bad for:** Simple tasks — coordination overhead is real. Also: what happens when two sub-agents return contradictory findings? Orchestrator needs explicit conflict resolution logic.

**Key constraint:** Sub-agents cannot spawn further sub-agents. The hierarchy is deliberately flat: orchestrator → sub-agents only.

---

### Reflexion
**Pattern:** Agent runs → produces answer → reflection step evaluates it → agent runs again with feedback → repeat until quality threshold met.
**Good for:** Tasks with a clear evaluable quality criterion — code correctness, factual grounding, investigation completeness.
**Bad for:** Tasks with no clear evaluation criterion, early-stage systems.

---

### Memory-augmented
**Pattern:** Persists state across sessions. Three types: short-term (in-context conversation history), long-term episodic (past investigation summaries in vector store), long-term semantic (domain facts learned over time).
**Good for:** Recurring tasks where past patterns inform current ones. Your DQ agent learning "CID PSI spikes on last business day due to volume — not a real breach."
**Bad for:** One-shot tasks with no recurrence value.

---

## Sub-agents — The Architecture That Makes Orchestration Scale

Sub-agents solve two distinct problems:

**Specialization:** Each sub-agent has a focused system prompt, scoped tools, and the right RAG strategy for its specific task.

**Context isolation:** Each sub-agent runs in its own context window. It does its work — reads files, calls tools, reasons — and returns only its conclusion to the orchestrator. All the intermediate noise stays inside the sub-agent's window and never touches the orchestrator's context.

```
Without sub-agents:
Orchestrator context = system prompt + all tool outputs + 
                       all reasoning + all retrieved chunks
                     → context bloat → reasoning degrades

With sub-agents:
Orchestrator context = system prompt + sub-agent conclusions only
Sub-agent context    = its own isolated window (noise stays here)
                     → orchestrator stays clean → reasoning stays sharp
```

**Each sub-agent is fully independent:** its own flavor, its own RAG strategy, its own tools. You can swap the RAG type inside a sub-agent without touching the orchestrator. The layers are cleanly separated.

---

## RAG + Agents — They Go Hand in Hand

RAG and agents solve different problems but need each other:

**RAG without an agent:** single-shot lookup. Breaks on multi-step investigation.
**Agent without RAG:** reasoning engine with no external knowledge. Answers from stale training data.
**Together:** the agent decides what to retrieve, RAG provides the mechanism, the agent reasons over what came back, decides whether to retrieve again.

**RAG as a tool the agent calls:**
```
Agent tools:
  search_knowledge_base(query)      ← vector RAG
  query_score_history(hql)          ← structured RAG
  traverse_lineage_graph(entity)    ← GraphRAG
  search_incident_history(query)    ← hybrid RAG
  get_pipeline_run_logs(id)         ← direct tool call
```

Each sub-agent uses the RAG type that fits its specific task. The Ownership Resolver uses graph traversal. The Score Analyzer uses vector search over documentation. The Pipeline Investigator uses structured HQL. This is Modular RAG at its fullest expression.

---

## Implementation Frameworks

### The raw loop — understand this first
Every framework abstracts this pattern. Understanding it means understanding every failure mode:

```python
while True:
    response = llm.call(messages, tools)
    messages.append(response)
    
    if response.stop_reason == "end_turn":
        return response.text          # agent decided it's done
    
    elif response.stop_reason == "tool_use":
        for tool_call in response.tool_calls:
            result = execute_tool(tool_call.name, tool_call.inputs)
            messages.append(tool_result(tool_call.id, result))
        # loop continues — LLM reasons over tool results
```

Five things every agent framework solves: the loop, tool definitions, message history, memory, error handling.

### LangChain — rapid prototyping
Pre-built components for tool calling, memory, RAG pipelines. Large ecosystem of integrations. Good for getting something working fast.
**Honest criticism:** Over-abstracted. Debugging is painful when something goes wrong deep in the chain. Many teams prototype in LangChain, rewrite core logic in raw API for production.

### LangGraph — complex workflows
Agent as a state machine — nodes are steps, edges are transitions with conditions. Explicit state object carries everything the investigation has learned. Branching logic is Python code you write and test.

```python
# Your DQ investigation as a state machine
graph.add_node("analyze_scores", analyze_scores)
graph.add_node("check_pipeline_logs", check_pipeline_logs)
graph.add_node("identify_owner", identify_owner)
graph.add_conditional_edges("analyze_scores", should_check_logs)
```

**Good for:** Complex workflows with branching. Your DQ agent — "if hypothesis involves pipeline, check logs; otherwise go straight to ownership."

### AutoGen — multi-agent conversation
Agents as conversation participants. They message each other, debate, delegate. Good for research tasks where multiple perspectives improve quality.
**Honest tradeoff:** Expensive — multiple LLM calls per step. Coordination failures are a real failure mode.

### MCP — the connectivity layer (not an agent framework)
Standardized interface between agents and tools. Not the agent loop — the plumbing the agent calls into.

```
Agent (any framework)
        ↓
MCP protocol (standardized tool call format)
        ↓
Your tools (Hive, Oracle, vector store, graph DB)
```

**Why it matters for you:** Build your DQ tools as MCP servers once. Any MCP-compatible agent — Copilot today, Claude tomorrow — calls them without rewiring. Portability across LLM clients in your constrained environment.

**MCP transport (current state):**
- stdio — local, agent launches server as subprocess. Fully bidirectional. Use this first.
- Streamable HTTP — remote, single unified endpoint, bidirectional. Use when you need multi-client access.
- SSE — deprecated as of March 2025. Don't use for new builds.

**MCP is not dead:** Donated to Linux Foundation December 2025 with OpenAI, Google, Microsoft, AWS as co-founders. 97M monthly SDK downloads. Industry standard, not vendor lock-in.

---

## Context Engineering — The Critical Production Discipline

In a single-agent system, context management is simple. In orchestrated multi-agent systems it becomes a core engineering concern.

**Why it compounds:** Every agent window costs tokens. Every window has a quality ceiling — reasoning degrades as it fills. Orchestrator + 3 sub-agents = 4 simultaneous context windows, all burning tokens.

### The failure modes to watch for

**Context rot:** Orchestrator accumulates every sub-agent result and tool output across a long session. By investigation 5 it's carrying noise from investigations 1-4 and losing focus. Fix: sub-agents return summaries not raw outputs. Clear context between independent tasks.

**Prompt bloat:** Tool descriptions cost tokens before any reasoning begins. 20 MCP tools in one agent's context = 10,000-15,000 tokens of schema overhead per invocation. Fix: give each sub-agent only the tools it actually needs. Minimum viable toolset per agent.

**RAG chunk noise:** Retrieving 10 chunks when 3 would answer the question fills the context with tangential information. Fix: re-ranking. Lower top-k. Contextual chunking to improve precision.

**Unbounded history:** Long sessions accumulate stale conversation history resent on every turn. Fix: periodic summarization or context clearing between independent tasks.

### The levers you control

| Decision | Context impact |
|---|---|
| Sub-agent output format | Verbose vs structured summary — huge difference |
| Tool count per agent | Every tool description costs tokens upfront |
| RAG chunk count (top-k) | More chunks = richer context but more noise |
| Conversation history policy | Keep all vs summarize vs clear between tasks |
| System prompt length | Longer = more expensive on every single turn |
| Model choice per sub-agent | Match compute cost to task complexity |

### Model tiering — match cost to complexity

Not every sub-agent needs your most capable model:

```
Ownership Resolver → Haiku    (deterministic lookup, no reasoning needed)
Score Analyzer     → Sonnet   (pattern interpretation, moderate reasoning)
Root cause synthesis → Opus   (nuanced synthesis, highest stakes output)
```

Running everything on the most expensive model is the fastest way to make a multi-agent system economically unviable in production.

---

## Orchestration Design: Explicit vs Dynamic

**Explicit orchestration** — system prompt hardcodes when and what to delegate:
```
Always spawn these sub-agents in parallel:
1. Score Analyzer
2. Pipeline Investigator  
3. Ownership Resolver
```
Predictable, auditable, consistent. May spawn unnecessary sub-agents.

**Dynamic orchestration** — system prompt describes available sub-agents, LLM decides:
```
Use Score Analyzer when interpreting score trends.
Use Pipeline Investigator when you suspect a load issue.
Use Ownership Resolver once you've identified the pipeline.
```
Adaptive, efficient. Less predictable, harder to debug.

**Production pattern — combine both:**
```
ALWAYS (non-negotiable):
  Confirm team ownership before closing investigation
  State confidence level in final output

USE JUDGMENT (adaptive):
  Spawn Pipeline Investigator only if drop appears abrupt
  Spawn Incident History only if pattern looks recurring
```

Hardcode the non-negotiable quality standards. Leave room for judgment on the variable investigation paths.

---

## The Two Fundamental Ceilings of Agentic AI

Understanding where AI genuinely hits limits is senior engineering judgment.

### Ceiling 1: The reasoning ceiling

LLMs reason probabilistically through learned patterns. They degrade on:

**Non-linear reasoning** — where the correct next step isn't the most statistically likely one. Hard debugging scenarios, problems where the signal is the *absence* of something expected.

**Causal reasoning under uncertainty** — holding multiple competing hypotheses and systematically eliminating them. LLMs tend to commit to the most plausible hypothesis early and find confirming evidence rather than genuinely testing alternatives.

**Novel problems** — if the problem has no close analogues in training data, the model pattern-matches to the nearest known thing, which may be subtly wrong in ways that compound across agent loops.

**Compensation:** Your system prompt methodology — "form a hypothesis, then actively try to refute it" — is you compensating for the LLM's natural confirmation bias.

### Ceiling 2: The context ceiling

The problem space for many real tasks is simply larger than any context window, and the token cost of exploring it is prohibitive.

**Combinatorial explosion** — complex system investigations may require considering interactions between dozens of components. The number of possible explanations grows exponentially. You can't fit all the relevant context.

**Long-horizon tasks** — tasks requiring coherent state across hundreds of steps. Context fills, history gets truncated, agent loses the thread. Current agents are genuinely brittle beyond a certain task horizon.

**Global pattern synthesis** — the insight lives in the pattern *across* 10,000 documents, not in any individual one. Chunking + retrieval loses the global view.

### The practical map

```
                    Reasoning complexity
                Low                  High
            ┌──────────────────┬──────────────────┐
        Low │ Solved           │ Hard but          │
Context     │ (RAG, simple Q&A)│ tractable         │
space       │                  │ (your DQ agent)   │
            ├──────────────────┼──────────────────┤
       High │ Expensive but    │ Currently         │
            │ manageable       │ very hard         │
            │ (sub-agents help)│ (open research)   │
            └──────────────────┴──────────────────┘
```

Your DQ anomaly agent is tractable: reasoning is non-trivial but structured enough to encode as methodology, context space is large but bounded by your domain.

---

## Your Full DQ Agent Architecture

```
Alert fires
      ↓
LangGraph state machine     ← manages investigation flow and branching
      ↓
Orchestrator (ReAct loop)   ← LLM reasoning with tool use
      ↓
Spawns sub-agents based on system prompt policy
      ↓
┌─────────────────────────────────────────────────────┐
│ Score Analyzer         → vector RAG on DQ runbook   │
│ (ReAct, Sonnet)          + structured RAG on QVDs   │
│                                                     │
│ Pipeline Investigator  → structured RAG on run logs │
│ (Plan-and-Execute,       HQL against Oracle         │
│  Sonnet)                                            │
│                                                     │
│ Ownership Resolver     → graph traversal            │
│ (single-shot, Haiku)     pipeline → team → contact  │
│                                                     │
│ Incident History       → hybrid search              │
│ (single-shot, Haiku)     over past incidents        │
└─────────────────────────────────────────────────────┘
      ↓
Sub-agents return clean summaries (not raw outputs)
      ↓
Orchestrator synthesizes
      ↓
Reflexion check          ← are numbers grounded? owner confirmed?
      ↓
Stakeholder output       ← root cause + evidence + owner + confidence
```

Each layer is independently buildable and testable:
- Phase 1: Build and test each retrieval source in isolation
- Phase 2: Build each sub-agent against working retrieval
- Phase 3: Build orchestrator system prompt against working sub-agents

---

## The Plumbing vs Brain Separation

**The system prompt is the brain** — defines role, methodology, sub-agent delegation policy, output format, quality standards.

**The plumbing is the body** — vector stores, graph DB, SQL connections, MCP servers, chunking pipelines, re-rankers. The agent assumes it exists. If the plumbing is broken, the best system prompt returns garbage.

Build the plumbing first. Test it independently. Wrap it in agents second. The most common failure mode in agentic systems is building the agent before the tools it calls actually work reliably.

---

## One-Line Summary of Every Concept

| Concept | One line |
|---|---|
| Agent | LLM that decides actions, executes them, loops until done |
| ReAct | Think, act, observe, repeat — the foundation of every agent |
| System prompt | Your domain expertise + methodology made executable |
| Tool-use agent | Large toolset, tool selection is the core skill |
| Plan-and-execute | Plan upfront, execute step by step, revise if needed |
| Multi-agent | Orchestrator delegates to specialized sub-agents |
| Reflexion | Runs, self-evaluates, revises until quality threshold met |
| Memory-augmented | Learns from past sessions, gets better over time |
| Sub-agent | Specialized isolated agent — own context, own RAG, own tools |
| Context isolation | Sub-agent noise stays in sub-agent window, not orchestrator |
| Model tiering | Match compute cost to task complexity per sub-agent |
| Context engineering | Keeping every window clean, minimal, purposeful |
| MCP | Standardized connectivity between agents and tools |
| Explicit orchestration | System prompt hardcodes delegation — predictable |
| Dynamic orchestration | LLM decides what to delegate — adaptive |
| Reasoning ceiling | LLMs degrade on non-linear, causal, novel problems |
| Context ceiling | Some problems are simply too large for any window |

---

*Built from a learning conversation — DQ scoring engine and fintech data engineering context. Companion documents: RAG Architecture Reference, Graph DB Reference.*
