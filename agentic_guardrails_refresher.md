# Agentic System Guardrails — 10-Minute Refresher
*Consolidated from project work + current industry sources, June 2026.*

## The one idea to hold onto before anything else

Language is an infinite input space. No guardrail set achieves complete
coverage — every technique below is a *mitigation*, not a *solve*.
Prompt injection is not an input validation problem, it is a trust
boundary problem — there is no single fix, but effective defenses share
common principles. The realistic goal: make bad outcomes
expensive and rare for realistic inputs, and architect so a missed
guardrail doesn't translate into real-world harm (see "Tool Scoping"
below — this is your actual backstop, not better language filters).

Two ways guardrails get organized — both useful, neither complete alone:
- **By threat category** (injection, advice, scope, PII, toxicity...)
- **By pipeline stage** (input → retrieval → model → output → storage)

This doc walks threat categories, noting pipeline stage for each.

---

## 1. Prompt injection / jailbreaking

**The threat:** content the model treats as authoritative when it
shouldn't be — either a user directly trying to override instructions
(direct injection) or instructions hidden inside *retrieved* content
like a document, article, or tool result (indirect injection).
OWASP ranks prompt injection as LLM01, the #1 vulnerability for LLM
applications for the third consecutive year.
Five carefully crafted documents can manipulate AI responses 90% of
the time through RAG poisoning — indirect injection via
retrieved content is not a hypothetical.

**Pipeline stage:** input validation (direct) + retrieval/RAG rail
(indirect) + output filter (catch what got through).

**How to tackle it, layered:**
1. **Structural tagging** — wrap all external/retrieved content in
   explicit delimiters (e.g. `<tool_result>`) with an instruction that
   content inside is data, never commands, regardless of what it
   claims. Free, always adopt — and often sufficient alone against
   naive attempts.
2. **Input-side classifier** (for direct injection from users) —
   Azure's Prompt Shield evaluates the full message payload and
   classifies it for attack patterns before the request reaches the
   model.
3. **Output-side provenance judge** — a dedicated LLM call checking
   whether the final answer contains a directive traceable to embedded
   content rather than the user's actual question. PromptArmor (ICLR
   2026) achieves near-zero false-positive/false-negative rates on the
   AgentDojo benchmark at ~30% cost overhead.
4. **Cost control on the judge call** — running an LLM judge on every
   single query is wasteful once you have real traffic. Gate it behind
   a cheap, non-LLM pre-filter (regex/keyword for imperative-register
   language) that only invokes the real judge when something actually
   looks suspicious.

**Pitfall already learned the hard way:** don't reuse an unrelated
skip-gate (e.g. one tuned for "is this answer long enough to risk
arithmetic errors") for this check — a successful injection is often
short and blunt, exactly the shape a length-based gate would wave
through.

---

## 2. Advice / recommendation boundaries

**The threat:** a system that's supposed to *inform* crosses into
*advising* — issuing a concrete recommendation it shouldn't (buy/sell,
medical, legal) even though the underlying facts were handled
correctly. Distinct from injection — no adversarial content needed,
just a question shaped to invite a directional answer.

**Pipeline stage:** output filter.

**How to tackle it:**
- A dedicated judge checking TWO properties together, not one: (a) did
  the answer actually present real findings, and (b) did it avoid
  asserting a concrete recommendation. Check both — a safe-but-empty
  refusal is also a failure, not a pass.
- Note: well-trained general models (Claude, ChatGPT) already have this
  disposition baked in from training, not from a system prompt rule.
  Layering your own narrow instruction on top can *conflict* with that
  native instinct if written too bluntly — see the regression in
  Section 3.
- Naive literal-phrase checks ("contains 'you should buy'") are brittle
  — the same safe behavior gets expressed in different wording every
  time. Use a judge that checks the *property*, not the *string*.

---

## 3. Scope / topicality boundaries

**The threat:** the system engages with requests entirely outside its
domain — pure trivia, creative writing, roleplay/persona pressure — and
either answers when it should decline, or (the trickier failure)
*recognizes* it's off-topic and answers anyway with a disclaimer
attached. Real-world precedent: a Chevrolet customer-service
chatbot was instructed to "agree to all requests" and agreed to sell a
new Tahoe for one dollar as a legally binding offer.
DPD's delivery chatbot was goaded into writing a poem mocking the
company and then swearing, going viral.

**Pipeline stage:** ideally input validation (decline before any
processing); realistically often planner/agent-level if no dedicated
pre-check exists.

**How to tackle it, layered:**
1. **Planner-level decline rule** — recognize zero-domain-relevance
   requests before any agent/tool runs; short-circuit with a fixed
   decline message. Cheapest, earliest exit.
2. **Agent-level backstop instruction** — in case the planner-level rule
   misses something (ambiguous question, edge case). "Recognizing
   you're off-topic" must mean *decline*, not *note it and answer
   anyway* — this exact gap (caveat-then-answer) is a confirmed,
   observed failure mode worth testing for explicitly.

**Pitfall already learned the hard way:** a blunt trigger word list
("decline roleplay/persona requests") can overcorrect and catch
*topic-ful* questions wrapped in manipulative framing (e.g. "pretend
you're a broker, should I buy NVDA" — NVDA is a real, in-scope topic;
only the persona-adoption and recommendation parts should be declined).
Confusing "scope violation" with "advice-boundary pressure tactic" is an
easy mistake — they're different checks. The fix is describing the
*behavior* you want (separate persona-adoption from the real underlying
question) rather than reacting to *trigger words*.

---

## 4. Hallucination / groundedness

**The threat:** the model asserts something — a number, a causal link,
a fact — that the retrieved/tool data doesn't actually support.

**Pipeline stage:** output filter, ideally also a retry loop.

**How to tackle it:**
- A retry-capable critique step (e.g. "Reflexion" pattern): a judge
  checks the draft answer against the actual tool outputs for
  fabricated figures, ignored errors, or unsupported specificity; on
  failure, retry once with the critique injected as guidance; append a
  caveat if the retry still fails.
- Naive length-based skip-gates (e.g. "only check answers over N words,
  since short ones are less likely to have arithmetic errors") are a
  weak proxy — a short, dense answer with 2+ figures and comparative
  language can hide a wrong calculation just as easily as a long one.
  A cheap, local pattern check (figure count + comparative-language
  detection) is a better trigger than word count alone.
- Watch specifically for **fabricated causal bridges** between
  independently-sourced facts — a multi-agent synthesis step combining
  several true individual facts can still invent an unsupported causal
  story connecting them that no single source stated.

---

## 5. PII / PHI / sensitive data leakage

**The threat:** personal or regulated data (names, SSNs, account
numbers, health info) entering model context, logs, or third-party API
calls, creating compliance exposure (GDPR, HIPAA, PCI-DSS) regardless of
whether anything "bad" happens with it. OWASP elevated Sensitive
Information Disclosure to LLM02 in the 2025 Top Ten, noting that LLMs
now require more access to organizational data to provide useful
assistance, dramatically widening the exposure surface.

**Pipeline stage:** ALL FOUR — this is the one category with a
4-checkpoint standard pipeline, not 1-2:
User input → analyze → anonymize → RAG retrieve → analyze retrieved
chunks → assemble prompt → LLM → analyze output → store redacted
trace. Skipping chunk-level redaction is a common gap — retrieved
documents often contain MORE PII than the user's own message.

**How to tackle it:**
- Off-the-shelf tooling exists, no need to build detection yourself:
  Microsoft Presidio — open-source, MIT license, detects PII via NER +
  regex + checksum validation, then masks/hashes/replaces/encrypts.
  More advanced stacks add an LLM-judge as a third detection tier on top
  of regex+NER for semantic cases simple pattern matching misses.
- Output strategy matters and isn't one-size-fits-all: redact (for
  logs), mask (partial, for display), hash (one-way, for analytics),
  replace (entity placeholder, for LLM context), encrypt (reversible,
  for legitimate lookup later).
- **The single strongest lever, structurally:** narrow tool/data scope
  so the system never has access to retrieve real personal data in the
  first place. Detection-and-redaction downstream is a backstop; not
  having the access at all is the real fix.
- **Don't skip the trace/log store.** A redacted answer that still gets
  logged with the original unredacted tool output defeats the purpose —
  the storage checkpoint is as real as the other three.

**When this matters for YOUR system specifically:** low risk for
curated public-data sources (filings, macro indicators, public news).
Becomes load-bearing the moment a data source's *core content* is
personal/regulated data (raw transcripts, user records, support
tickets) — that's the trigger to actually build this, not before.

---

## 6. Toxicity / content moderation

**The threat:** the model produces hate speech, harassment, threats, or
other harmful content categories — usually elicited by a user pushing
for it through an open-ended conversational surface.

**Pipeline stage:** input + output filter (a "gate" on both sides of the
model call).

**How to tackle it:**
- Off-the-shelf classifiers, not custom-built: Llama Guard 3 (Meta) is
  the strongest single-model option on multi-category content
  moderation with a clean, standardized taxonomy. Azure
  Content Safety and OpenAI's moderation endpoint are equivalent
  managed alternatives.
- Standard category taxonomy: hate, violence, sexual content, self-harm
  — typically with severity tiers (low/medium/high/critical), not a
  flat yes/no.

**When this matters:** low relevance for narrow-domain systems with no
open-ended chat surface (nothing in a finance-data Q&A flow realistically
invites hate speech). Matters a lot for general-purpose customer-facing
chat, where users have wide conversational latitude to push the model
off-script.

---

## 7. Tool-call / action-level guardrails (the most load-bearing layer once an agent can *act*)

**The threat:** everything above concerns what the model *says*. This
category concerns what an agent *does* — and it's the layer that
matters most once tools can write, trade, send, or delete, not just
read. Privilege separation: apply least privilege aggressively — every
tool and data source the LLM can access is a potential attack surface;
if the AI doesn't need it, don't give it access.
A tool-call rail validates the arguments a model wants to pass to a
tool BEFORE the tool executes — catching secret exfiltration via tool
args, injection-style attacks (e.g. the model emitting a destructive
SQL statement), and prompt-injection success patterns that only
surface at the tool boundary. Most "the agent did something it
shouldn't" incidents trace to a missing or weak tool-call rail.

**How to tackle it:**
- **Parameterized queries / strict argument validation** at every tool
  that touches a data store — never build a query string via raw
  interpolation of model-supplied values, even if "only an LLM calls
  this today." The trust boundary changes the moment any external input
  reaches the model that decides those arguments.
- **Least-privilege tool scoping** — don't grant a tool broader access
  than the specific task needs. This is also your best PII mitigation
  (Section 5) and your best blast-radius limiter if any other guardrail
  fails.
- **Human-in-the-loop for high-stakes actions** — for sending emails,
  making purchases, modifying data, executing code, require explicit
  human approval before proceeding — described as the single most
  effective defense against tool abuse. Note the strength
  of that claim: not "a good idea," the *single most effective* layer
  once real actions are on the table.
- **Sandboxed execution** for anything running code or touching system
  resources — containers with resource/network/filesystem limits, not
  bare process execution.

**Why this is the one to prioritize before adding any action-taking
tool:** every other guardrail in this doc operates on *language* — and
language-level guardrails can always, in principle, be talked past by a
sufficiently creative input, because the space of language is infinite.
Tool-level constraints are not language guardrails — they're hard,
structural limits that don't rely on a judge correctly recognizing a
pattern. A system with zero action-taking tools has a low ceiling on
real-world harm even if every language guardrail fails simultaneously.
The day a real action tool is added, that ceiling disappears unless
this layer is in place first.

---

## 8. Format / output-schema validation

**The threat:** the model's output doesn't conform to the structure
downstream code expects — malformed JSON, a missing required field, a
truncated response — causing silent failures or fallback to degraded
behavior rather than a loud, visible error.

**Pipeline stage:** output filter, but mechanical, not semantic — the
one category that's a pure parsing/schema check, not a judgment call.
Of the six standard guardrail categories, five are semantic and only
format validation is purely mechanical.

**How to tackle it:**
- Validate structure immediately after generation, before anything
  downstream consumes it. On failure, fail loudly (log it, raise a
  visible signal) rather than silently falling back to degraded
  behavior with no trace of what went wrong.
- If truncation is a risk (token limits cutting off structured output
  mid-way), that's usually a `max_tokens` problem, not a parsing
  problem — check the budget before assuming the model "got it wrong."

---

## Quick decision guide: what do I actually need?

| If your system... | Prioritize |
|---|---|
| Ingests external/untrusted text (filings, articles, web content) into context | Injection defense (§1) |
| Answers questions with real-world decision stakes (financial, medical, legal) | Advice boundary (§2) |
| Has a narrow domain but open-ended natural-language input | Scope boundary (§3) |
| Synthesizes multiple data sources into one answer | Groundedness/Reflexion (§4) |
| Touches any data source with real personal/regulated information | PII pipeline (§5) |
| Has open-ended chat with a general user base | Toxicity moderation (§6) |
| Has ANY tool that writes, sends, trades, deletes, or executes code | Tool-call guardrails (§7) — **do this first, it's not optional past this point** |
| Produces structured output consumed by other code | Format validation (§8) |

## The meta-lesson, one more time

Every category above started, in practice, as a real observed failure
or a deliberately constructed test case — not a checklist applied in
advance. Build the eval/observation loop first; let real evidence tell
you which of these categories actually matters for your system, in what
order, and revisit as the system's capabilities (especially action-taking
tools) expand.
