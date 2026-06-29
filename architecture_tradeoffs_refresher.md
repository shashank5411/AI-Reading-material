# Architecture & Tradeoffs — Personal Reference / Interview Prep
*Maps the real trade-platform system onto standard agentic-engineering vocabulary. Companion to the eval and guardrails reference docs.*

This doc is for answering "describe your architecture" and "why did you
choose X over Y" questions with precision — using your actual system's
real numbers, not generic description. Every claim below is grounded in
your actual config values and documented decisions, not idealized.

---

## 1. The one distinction worth nailing cold: ReAct vs. planner-executor

**Your system is BOTH, layered — not a hedge, a deliberate composition.**

| Layer | Pattern | Where |
|---|---|---|
| Orchestration (which agents, what order) | **Planner-executor** | `planner.py` produces the full DAG upfront, before any agent runs |
| Within each sub-agent (which tool, what next) | **ReAct** | `sub_agents.py`'s shared `_run_agent()` loop |

**ReAct** (Reasoning and Acting): one continuous loop where the model
interleaves a "Thought" with an "Action" (tool call), observes the
result, and decides the *next* step incrementally — the plan is not
fixed in advance. Your sub-agents do this literally: FilingsAgent
calling `get_prose` → observing a short stub → reasoning "this section
came back empty, try semantic search instead" → calling
`semantic_search`. The decision at step 2 depends on what step 1
actually returned.

**Planner-executor**: a planning LLM produces a structured plan upfront;
an executor runs it. Your `planner.py` does exactly this — builds the
entire DAG (which agents, dependencies, shape) in one call, before
`dag_executor.py` ever invokes a single agent. It doesn't interleave
"route one agent → observe → re-route" — it commits to the full
structure first.

**Why this composition, not one pattern throughout:** re-planning which
specialists to invoke on every step would be expensive and usually
unnecessary — a question's *shape* (does this need Market alone, or
Market+Sentiment+Filings) is typically knowable from the text alone,
without needing to see any tool result first. But *within* one
specialist, the next tool call genuinely depends on what the previous
one returned (stub vs. real content, empty vs. populated) — that's
exactly the case ReAct is built for. Using planner-executor for the
cheap-to-decide-upfront layer and ReAct for the must-adapt-as-you-go
layer is the efficient choice, not a compromise.

**One-liner for an interview:** *"Planner-executor at the orchestration
level, because the question's required specialists are usually knowable
upfront and re-planning per step would be wasteful. ReAct within each
specialist, because which tool to call next genuinely depends on what
the previous tool actually returned — I can't know in advance whether a
filing section will come back as real content or a stub needing a
fallback."*

---

## 2. Multi-agent coordination topology: supervisor, not peer-to-peer

**Your system is supervisor-based**, also called hierarchical/
orchestrator-pattern: one component (the planner) decides task
assignment and dependency structure; sub-agents don't negotiate with
each other or vote — they receive their slice of the question, do their
work, and the executor/synthesis step combines outputs.

**The alternative — peer-to-peer/collaborative** — agents communicate
directly, debate, and reach consensus (e.g. a "writer" and "editor"
agent iterating on a shared draft). Good for open-ended, creative,
consensus-needing tasks; more robust to single-agent failure but harder
to debug and more prone to going in circles.

**Why supervisor fits this system:** financial research questions
decompose cleanly into independent or sequentially-dependent
sub-questions (price data, macro context, filing content, sentiment
signal) with a natural "merge point" (synthesis). There's no need for
agents to negotiate or vote — MarketAgent never needs to convince
FilingsAgent of anything. The known cost of supervisor-pattern systems
(single point of failure, potential bottleneck at the orchestrator) is
real but acceptable here: a planner failure has a defined, observed
fallback (degraded single-agent routing on JSON parse failure — found
and partially exploited as a bug source, see the scope-boundary work in
the eval reference doc).

**Known, documented limitation worth citing if asked "what would you
change":** the planner's heuristic for deciding parallel-vs-sequential
appears to be "can each agent fetch independently," not "does answering
require one agent's conclusion to feed another" — these aren't always
the same question, and an early attempt to force a true reasoning-merge
DAG shape (not just data-dependency) repeatedly got planned as parallel
instead, with the verdict deferred to synthesis rather than to one
agent reasoning over another's output. This was decided to be a
real architectural property worth knowing about, not chased into a fix
— a good example of recognizing "the planner's actual semantics differ
from what I assumed I was testing," same epistemic honesty pattern as
elsewhere in this project.

---

## 3. Model tiering — real numbers, not a generic claim

| Role | Dev | Prod |
|---|---|---|
| Sub-agents (Market/Macro/Filings/Sentiment) | `claude-sonnet-4-6` | same |
| Planner / synthesis / reflexion critic | `claude-haiku-4-5-20251001` | `claude-sonnet-4-6` |

**Why this split:** Haiku is roughly 20x cheaper than Sonnet. Planning
(deciding a DAG shape) and critiquing (checking an answer against tool
outputs) are comparatively low-complexity reasoning tasks relative to
actually synthesizing financial analysis from retrieved data — cheaper
model, dev-time iteration speed, acceptable in dev where correctness
tolerance is higher. Prod switches planner/synthesis/critic to Sonnet
because routing and grounding mistakes are more costly once it's not
just you testing — same reasoning, different risk tolerance, not a
different architecture.

**Real cost data, for "how do you think about cost" questions:**
- Dev AWS infra (storage, crawlers, Athena, DynamoDB): ~$3/month total
- Anthropic API, Haiku dev testing: ~$0.003/query
- Anthropic API, Sonnet eval runs: ~$0.05–0.10/query
- Per-agent token budgets are tiered by expected task complexity, not
  uniform: Market/Macro 50,000 tokens, Sentiment 75,000, Filings
  150,000 (filings work involves the most tool-call iterations —
  multi-section batching, semantic search fallbacks).
- Tool definitions cost ~1,500 tokens on every single call regardless of
  question complexity (all registered tools' schemas are sent every
  time) — a fixed overhead worth knowing when asked about per-query
  cost floors.
- Price-data aggregation was deliberately changed from daily to weekly
  granularity for long date ranges specifically to cut token cost
  (~8,000 → ~1,500 tokens for a 1-year query) — a concrete example of
  "context engineering as a cost lever," not just a correctness lever.

---

## 4. Tool design philosophy

**Anthropic tool-use format, not plain Python functions called
directly, and not text-to-SQL.** Two separate decisions worth being
able to defend independently:

- **Tools over raw functions:** the LLM decides which tool to call and
  with what parameters, so novel question phrasing is handled
  automatically rather than needing a hardcoded intent-classifier in
  front of the system. Tool *descriptions* are explicitly treated as
  "the intelligence" — vague tool descriptions were a real, named
  source of routing/selection errors early in the project (see
  `TOOL-MARKET-002`'s sector-taxonomy gap in the eval doc — not a
  description problem, but the same general principle that the tool
  layer's documented behavior has to match its actual behavior).
- **Deterministic data access over text-to-SQL:** the LLM never
  generates SQL directly — it calls structured Python functions
  (`api.py`) that build parameterized, partition-aware queries
  internally. Reasoning happens in the LLM; reliable data access
  happens in code. This is a real, deliberate split, not laziness —
  partition-aware Athena queries require schema knowledge that's baked
  into the function layer once, rather than re-derived (and
  potentially gotten wrong) by the model on every call.

**Known, named gap, worth citing honestly:** SQL is currently built via
f-string interpolation of LLM-supplied arguments throughout `api.py`.
Currently low-risk because callers are constrained by Anthropic tool
schemas, not raw HTTP input — but this assumption breaks the moment any
HTTP-facing surface exists, since at that point the trust boundary
includes whatever a user types, filtered only by whatever the
planner/agent chooses to pass through. This is squarely the
"tool-call argument validation" guardrail category from the guardrails
doc — flagged, not yet fixed, a clear answer if asked "what's a known
risk you haven't closed yet."

---

## 5. Why not LangChain/LangGraph — a real, defensible answer

**Decision, stated plainly:** raw Anthropic API, not a framework — a
deliberate choice for a learning-focused project, not unfamiliarity.
The reasoning: frameworks abstract away exactly the mechanics worth
understanding firsthand (the ReAct loop's actual structure, how a DAG
gets resolved into execution rounds, how dependency context gets passed
between agents) — building it raw means failure modes are visible
rather than hidden inside framework internals.

**The honest tradeoff, worth stating unprompted rather than waiting to
be asked:** this means slower initial build time and reinventing some
solved problems (state management, checkpointing) that LangGraph
provides out of the box. The interview framing to use: *"I chose custom
because the goal was understanding the failure modes firsthand — for a
production team under deadline pressure, I'd evaluate LangGraph/CrewAI
on whether their state-graph or message-bus model fits the actual
coordination pattern needed, rather than defaulting to custom."* This
directly answers the "framework questions are a trap" pattern —
interviewers want a justified tradeoff, not a framework name-drop in
either direction.

---

## 6. Why Athena/S3, not RDS/Redshift, not a vector DB-first design

- **Decoupled storage (S3) and compute (Athena):** pay per query scan,
  not per provisioned hour — negligible at this project's scale (~$3/mo
  total AWS infra). Real architectural tradeoff for "when would this
  choice break down": Athena's per-query latency and lack of indexes
  make it a poor fit past a certain query-volume/latency-sensitivity
  threshold — a real production system at scale would likely need a
  warm-path cache (the project's own forward plan already names a
  DynamoDB query-result cache, SQL-hash keyed, with TTL tiered by data
  freshness needs — 1hr for prices, 24hr for filings — as a planned
  but not-yet-built mitigation).
- **Per-source Glue databases, not one unified schema:** consequence is
  cross-source queries (e.g. combining FRED and World Bank indicators)
  require a UNION or separate calls merged in Python
  (`get_indicator_multi` handles this transparently) — a real, accepted
  complexity cost in exchange for cleaner per-source IAM policies and
  independent lifecycle management.
- **S3 Vectors over a dedicated vector DB:** chosen specifically because
  it fits the existing S3-first architecture rather than introducing a
  new infrastructure dependency — a deliberate "don't add a new system
  if the existing one can do the job adequately" call, not a
  performance-maximizing choice.

---

## 7. Guardrail architecture, summarized (full detail in the guardrails doc)

Worth a one-paragraph version here since "agent questions are really
risk-management questions" comes up directly in interview prep
material: separation of concerns is the throughline — the LLM reasons
(sub-agents, planner), the orchestrator controls (DAG execution,
dependency resolution), and a growing set of dedicated judge calls
govern specific risk properties (grounding via Reflexion, prompt
injection via structural tagging + a gated provenance judge, advice and
scope boundaries via two more dedicated judges). No sandboxed execution
exists yet, because no action-taking tool exists yet — and that's
flagged explicitly as the dependency that would force this layer to be
built before any such tool ships, not an oversight.

---

## 8. Scaling bottlenecks — what would actually break first

Honest answer, not a deflection, for "how would you scale this":

1. **LLM round-trip latency, not infrastructure.** Each reasoning step
   costs 1-3+ seconds; a 3-agent sequential chain with reflexion retries
   has been observed at 60-115 seconds end-to-end. This is the dominant
   latency driver, not Athena or network — more agents/tool-calls
   multiplies wall-clock time directly, since most of the pipeline is
   currently sequential-dependent rather than maximally parallelized.
2. **Cost scales with the same multiplier** — a 3-agent merge query with
   reflexion retries costs meaningfully more than a single-agent
   question (real measured range: ~$0.003-0.10/query depending on
   complexity and which model tier).
3. **Athena has no real warm-path** today — every query re-scans,
   mitigated only by an in-process LRU result cache
   (`athena.py`'s `_cache`, max 100 entries, session-lifetime only, not
   persistent). The planned DynamoDB cache (see Section 6) is the real
   fix, not yet built.
4. **The planner's JSON-parse failure path is a real fragility point**
   under load or model variance — a single malformed plan currently
   degrades to a single-agent fallback rather than retrying or erroring
   loudly (this exact path was the proximate cause of the scope-boundary
   bug discovered live — see the guardrails reference doc).

---

## Quick-reference: "what pattern is this" cheat sheet

| If asked about... | Your answer |
|---|---|
| "Is this ReAct?" | Sub-agents: yes. Orchestration: no — that's planner-executor. Composed, not either-or. |
| "Single-agent or multi-agent?" | Multi-agent, supervisor-coordinated, not peer-to-peer/collaborative. |
| "Framework or custom?" | Custom, deliberately, for a learning-focused build — would evaluate LangGraph/CrewAI on fit for a production team under deadline pressure. |
| "How do you control cost?" | Two-tier model routing (Haiku/Sonnet by role), tiered per-agent token budgets, weekly (not daily) price aggregation for long ranges, in-process result caching. |
| "Biggest scaling risk?" | Sequential LLM round-trip latency, not infra — and Athena's lack of a real warm-path cache, mitigated only by a small in-memory LRU today. |
| "What's still unguarded?" | SQL built via f-string interpolation, safe only because callers are schema-constrained, not yet hardened for any HTTP-facing surface. No sandboxed execution, because no action-taking tool exists yet. |
