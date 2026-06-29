# RAG Architecture — Personal Reference
*10-minute refresher. Built from first principles, anchored to a fintech / data engineering context.*

---

## The One-Line Mental Model

> RAG gives an LLM a library card — before answering, it fetches the relevant documents, reads them, and *then* answers. It separates two jobs: **retrieval** (finding the right information) and **generation** (articulating an answer from that information).

Without RAG, an LLM answers from training memory — it either doesn't know, or hallucinates. With RAG, it reads actual data at query time.

---

## The Core Flow

```
User question
      ↓
Embed the query          → convert question to a vector
      ↓
Search knowledge base    → find top-k most similar chunks
      ↓
Retrieved context        → actual text passages from your documents
      ↓
LLM generates answer     → reads question + context together
      ↓
Grounded, cited answer
```

The LLM doesn't need to "know" the answer — it just needs to be good at reading and summarizing, which it is. The retrieved context provides the facts. The LLM provides the articulation.

**Why this matters for your domain:** The DQ anomaly agent asks "which dimension breached PSI last week?" — RAG fetches the actual score history, the LLM narrates the findings. No hallucinated numbers.

---

## Why Each RAG Variant Exists

Every RAG variant fixes a specific failure mode of naive RAG. That's the right lens — not a taxonomy to memorize, but a set of targeted solutions to known problems.

| Failure mode | Fix |
|---|---|
| Question is vague or badly worded | Query improvement techniques |
| Wrong chunks come back | Retrieval improvement techniques |
| Knowledge base is flat or unstructured | Knowledge structure techniques |
| Model needs to decide, not just retrieve | Agentic loop techniques |

---

## The RAG Family

### Naive RAG — the baseline

**Mechanism:** Embed question as-is → vector search → retrieve top-k chunks → generate.

**Real example:** A law firm Q&A bot. User asks "what is the notice period in the MSA?" — clean question, structured document, answer in one paragraph. Naive RAG retrieves it perfectly.

**Your context:** "What is the PSI formula used in the DQ engine?" against your internal documentation. Clean, specific, one-hop answer.

**Good fit:** Single-hop factual questions, clean document sets, low-latency requirements, first iteration / prototyping.

**Bad fit:** Vague questions, multi-part analytical questions, when precision matters more than speed.

---

### 🟣 Query Improvement — "the question was bad"

#### Query rewriting

**Mechanism:** Before embedding, an LLM rewrites the raw question into a richer, more search-friendly form. The rewritten query is then embedded and searched.

**Real example:** Customer support bot. User types "cant login 2fa broken" → rewriter expands to "user unable to log in, two-factor authentication not working, possible SMS delivery failure or app token issue." Retrieval now finds the right troubleshooting guide.

**Your context:** User asks "why is CID tanking?" → rewriter expands to "CID dimension data quality score degradation, recent PSI breach or Z-score anomaly." Now the right score history chunks get retrieved.

**Good fit:** Chatbots where users type casually, internal tools with domain jargon, short or ambiguous queries.

**Bad fit:** Already well-formed precise queries (adds latency for no gain), high-volume pipelines where cost per query matters.

---

#### HyDE (Hypothetical Document Embedding)

**Mechanism:** Instead of embedding the question, ask the LLM to generate a plausible hypothetical answer first — then embed *that*. Real answers live closer to other real answers in vector space than short questions do.

**Real example:** Academic literature search. Student asks "how do SSRIs affect neuroplasticity?" — LLM generates a paragraph about synaptic remodeling and BDNF pathways. That paragraph retrieves far more relevant papers than the 6-word question would.

**Your context:** "Why might composite scores drop for high-cardinality dimensions?" → HyDE generates a paragraph about volume-tiered PSI sensitivity and outlier effects, retrieving your most relevant design notes.

**Good fit:** Scientific/technical search where questions are short but answers are dense, when question vocabulary doesn't match document vocabulary.

**Bad fit:** Factual lookups (IDs, dates, codes), when latency is critical, low-trust environments where retrieval auditability matters.

---

#### Query decomposition

**Mechanism:** A complex multi-part question is broken into 2-4 simpler sub-questions, each retrieved independently. Results are merged and fed to the LLM to synthesize a final answer. Can be parallel (all at once) or sequential (answer informs the next sub-question).

**Real example:** HR chatbot. Employee asks "how does my bonus compare to the London office, and how has it changed since 2021?" Decomposes into: (1) your bonus policy, (2) London office bonus policy, (3) historical bonus policy changes. Three retrievals, one synthesized answer.

**Your context:** "Compare PSI scores for CID vs PRODUCT_CODE in January and flag which crossed the threshold first." Decomposes into three sub-queries, merged into one answer.

**Good fit:** Analytical multi-hop questions, comparison queries ("A vs B"), your DQ engine — almost every real question is compound.

**Bad fit:** Simple factual lookups, real-time / latency-sensitive applications.

---

### 🟢 Retrieval Improvement — "the wrong chunks came back"

#### Hybrid search

**Mechanism:** Two search systems run in parallel — vector/semantic search (finds meaning) and BM25/keyword search (finds exact matches). Scores combined using Reciprocal Rank Fusion (RRF).

**Real example:** E-commerce product search. "Waterproof hiking boots size 10" — semantic search finds boots matching "durable outdoor footwear," keyword search finds exact matches for "waterproof" and "size 10." Hybrid retrieves the right SKUs that purely semantic search would miss.

**Your context:** Searching DQ logs for "ATV_SCORE anomaly field_id=TXN_AMOUNT" — semantic finds relevant anomaly discussions, keyword finds the exact field name. Pure vector search frequently misses exact field identifiers in fintech pipelines.

**Good fit:** Any domain with exact identifiers (field names, IDs, codes), production RAG systems — this is usually the right default.

**Bad fit:** Simple prose-only document sets where keywords aren't meaningful, when you have a very small corpus.

---

#### Re-ranking

**Mechanism:** After retrieving top-20 candidates by fast vector search (bi-encoder), a slower but more accurate cross-encoder model re-scores each chunk in context with the query. Return only the top 3-5.

**Bi-encoder vs cross-encoder — the key distinction:**

| | Bi-encoder | Cross-encoder |
|---|---|---|
| How it works | Encodes query and document *separately*, compares vectors at the end | Reads query and document *together* from the start — full cross-attention |
| Speed | Fast — docs pre-encoded offline, O(1) lookup | Slow — must re-run for every (query, doc) pair |
| Accuracy | Good — misses negation, qualifications, subtle context | Better — catches nuance because query tokens attend to doc tokens |
| Role in pipeline | First pass: narrow millions to 50 | Second pass: narrow 50 to 5 |

**The pipeline:** Bi-encoder retrieves top-50 cheaply → cross-encoder re-ranks to top-5 accurately → LLM generates from top-5. Speed of one, accuracy of the other.

**Real example:** Legal discovery tool. Lawyer searches "indemnification clauses with carve-outs for gross negligence." Vector search returns 50 chunks mentioning indemnification. Cross-encoder re-ranker surfaces the 4 that specifically discuss gross negligence carve-outs.

**Your context:** Searching score history for a PSI alert — re-ranker surfaces the 3 chunks specifically about CID during the flagged date window, not just any PSI discussion.

**Good fit:** When precision matters more than recall, high-value low-volume queries, as a quality layer on any existing retrieval system.

**Bad fit:** High-throughput low-latency pipelines, when your first-pass retrieval is already excellent.

---

#### Iterative retrieval

**Mechanism:** Retrieval and generation alternate in a loop. Fetch → partial read → identify what's missing → fetch again with refined query. Unlike decomposition (which plans upfront), iterative retrieval *discovers* the next query from what it just read.

**Real example:** Medical diagnosis assistant. "Treatment options for a patient with Type 2 diabetes and stage 3 CKD." First fetch: diabetes treatment guidelines — finds metformin is contraindicated in CKD. Second fetch: CKD-safe diabetes medications. Third fetch: dosage adjustments for GFR < 45. Each hop discovered from the previous result.

**Your context:** DQ incident: fetch alert record → sees it involves CID → fetch CID PSI history → notices spike after a pipeline change → fetch that pipeline's change log → root cause identified. Each hop couldn't have been planned upfront.

**Good fit:** Deep investigative questions where the path isn't known upfront, your DQ anomaly agent.

**Bad fit:** Simple questions, when latency matters, risk of infinite loops without a good stopping condition.

---

### 🟡 Knowledge Structure — "the knowledge base was flat"

#### Hybrid search + Re-ranking already covered above.

#### GraphRAG

**Mechanism:** Documents are parsed into a knowledge graph — entities as nodes, relationships as edges — by an LLM extraction pass during ingestion. At query time, the system traverses the graph rather than doing similarity search. Can answer questions that require following chains of relationships across documents.

**Storage stack (three layers working together):**

```
Graph database (Neo4j, Neptune)   → entity nodes, relationships, traversal
Vector store (Pinecone, Weaviate) → node summaries, community summaries
Document store                    → raw source text chunks
```

**Real example:** Microsoft's GraphRAG on Enron emails. "Which executives were aware of accounting irregularities?" — vector RAG finds emails mentioning accounting issues. GraphRAG traverses: executives → reported-to → CFO → communicated-with → auditors → flagged-issues. Surfaces the chain of awareness no individual document contained.

**Your context:** "Which pipelines feed CID and who owns them?" — traversal: CID → fed_by → ETL_LOAD_CID → owned_by → Data Eng Team A. No single document had the full chain.

**Ingestion complexity — the honest tradeoff:**
Full GraphRAG ingestion is expensive: LLM extraction pass over every document, entity resolution across documents, graph merging, community detection, summary generation. For structured metadata (your DQ world), skip LLM extraction — build the graph directly from existing pipeline metadata and Oracle tables. Same traversal power, fraction of the ingestion cost.

**Good fit:** Cross-document relationship questions, data lineage, org chart traversal, compliance mapping across multiple regulations, when the answer requires following a chain of unknown depth.

**Bad fit:** Frequently updated document sets (stateful updates are complex), small corpora where overhead isn't justified, heavy aggregation queries.

**Best real-world fit:** Legal contracts referencing each other, multi-regulation compliance mapping, enterprise knowledge graphs, data lineage — one-time or infrequent loads where relationship traversal is the primary value.

---

#### Hierarchical indexing (RAPTOR)

**Mechanism:** Documents indexed at multiple granularity levels — fine chunks at the bottom, section summaries in the middle, document summaries at the top. Query routes to the right level: broad questions → summaries, precise questions → chunks.

**Real example:** McKinsey internal knowledge base. "What is our firm's overall view on digital transformation in retail?" → retrieves topic-level summary across 200 reports. "What did the Target engagement in 2022 recommend on inventory digitization?" → drills to specific chunks. Same index, two levels.

**Your context:** "How does composite scoring work?" → document summary. "What are the exact weight parameters for timeliness?" → specific config chunk.

**GraphRAG vs Hierarchical — the honest comparison:**

| | Hierarchical RAG | GraphRAG |
|---|---|---|
| Relationship type | Containment (parent-child tree) | Any relationship type (web) |
| Update complexity | Low — updates contained within one branch | High — updates may touch nodes across the whole graph |
| Reasoning trace | Automatic — structural provenance | Requires edge bookkeeping |
| Best for | Documents with clear section hierarchy | Cross-document entity relationships |
| Infrastructure | Simpler — two vector indexes | Complex — graph DB + vector store + doc store |

**The key question:** Can the relationships in your domain be represented as a tree, or do they form a web? Tree → Hierarchical wins. Web → GraphRAG is necessary.

**Good fit:** Large structured document sets, mixed broad and narrow questions, company policy libraries, technical documentation.

**Bad fit:** Small corpora, when summaries might mislead, always-chunk-level queries.

---

#### Structured RAG

**Mechanism:** Natural language question → LLM writes a structured query (SQL, HQL, Cypher) → executes against actual database → result fed to LLM as context. The "retrieval" is a real database query, not a vector search.

**Real example:** Airbnb internal analytics. PM asks "how many Paris listings had occupancy over 80% in Q3 with Superhost status?" — LLM writes SQL, runs against bookings DB, gets count of 1,247. No hallucination possible.

**Your context:** "Which dimensions had a PSI breach in the last 30 days grouped by business unit?" → LLM writes HQL against score history QVD → real results → LLM narrates findings. Your DQ scoring engine is already the structured knowledge base.

**Good fit:** Any question answerable by data in a database, fintech analytics — exactly your domain, aggregations and time-series.

**Bad fit:** Unstructured prose questions, when NL-to-SQL reliability isn't production-grade, complex multi-table joins with intricate business logic.

---

### 🟠 Agentic Loop — "the model needs to decide, not just retrieve"

#### Agentic RAG

**Mechanism:** The LLM acts as an agent with access to multiple retrieval tools. It decides whether to retrieve, which tool to use, what to query, and whether the answer is good enough or another loop is needed.

**Real example:** Salesforce Einstein Copilot. Sales rep asks "should I offer a discount to Acme Corp?" — agent retrieves Acme deal history, current discount policy, Acme churn risk score. Synthesizes a recommendation. No single retrieval step pre-planned.

**Your context:** DQ alert fires for CID. Agent decides: retrieve Z-score history → reads it → decides to also check PSI → fetches PSI → sees spike after pipeline run → fetches pipeline run logs → generates root cause and recommended action. This is exactly what your MCP server architecture is building toward.

**Good fit:** Complex open-ended investigative questions, multi-tool environments (DB + docs + APIs + logs), your DQ incident responder.

**Bad fit:** Simple well-scoped Q&A, when every retrieval step must be pre-auditable, high-latency sensitivity.

---

#### Self-RAG

**Mechanism:** The model generates reflection tokens alongside its output — "Did I need retrieval here? Is this chunk relevant? Is my answer grounded?" Self-assessments guide whether to retrieve, use a chunk, or regenerate.

**Real example:** Medical Q&A. Model generates a drug dosage answer, self-reflects: "Is this grounded? No — I used training knowledge, not the retrieved formulary." Triggers re-retrieval. Final answer only goes out when the model certifies its own grounding.

**Your context:** DQ explanation to a stakeholder: "Your CID score dropped 12 points due to a PSI breach." Self-RAG checks: "Is the 12-point figure from retrieved data? Yes, chunk 3." Reduces hallucinated numbers in stakeholder-facing outputs.

**Good fit:** High-stakes high-accuracy requirements, stakeholder-facing outputs where numbers must be verifiable.

**Bad fit:** High-throughput pipelines, early-stage RAG systems — build the basic pipeline first.

---

### Modular RAG — the punchline

In production, systems pick and mix. Your DQ anomaly agent might use query decomposition + hybrid search + re-ranking + structured RAG + agentic loop. That composition is Modular RAG — not a single pattern, but a design philosophy.

---

## Chunking Strategies

Chunking is the step that breaks documents into indexable pieces. Bad chunking upstream breaks everything downstream regardless of retrieval quality.

**The core problem:** Meaning doesn't respect arbitrary boundaries. Split a paragraph mid-sentence and the second chunk becomes a dangling reference — "this is more conservative than the standard threshold" with no context for what "this" refers to.

### Fixed-size chunking
Split every N tokens with M token overlap between consecutive chunks. Overlap ensures sentences near boundaries appear in both chunks.

**Good fit:** Homogeneous documents — log files, transcripts, uniform reports.
**Bad fit:** Structured documents with natural section boundaries — chunks split mid-thought.

### Sentence and paragraph chunking
Split on linguistic boundaries — sentences, paragraphs, double newlines.

**Good fit:** Prose documents where paragraphs are coherent units.
**Bad fit:** Very long dense paragraphs, or very short fragmented paragraphs.

### Structure-aware / recursive chunking
Parse the document's actual structure — headings, sections, subsections — and use those as chunk boundaries. Falls back to smaller units only when sections are too large.

**Good fit:** Well-structured documents with clear headings — your DQ runbook, technical docs, legal contracts. **The right default for your documentation.**
**Bad fit:** Unstructured prose with no headings, scanned PDFs.

### Semantic chunking
Embed every sentence, find points where consecutive sentence embeddings have low cosine similarity — those are the semantic boundaries where the topic shifts. Split there.

**Good fit:** Documents where topics shift without structural markers — meeting notes, incident reports, research papers.
**Bad fit:** Short documents where every sentence is topically related, computationally expensive.

### Contextual chunking
Before indexing, prepend a generated context summary to every chunk:

```
[CONTEXT: This chunk is from the DQ Scoring Framework runbook,
section on PSI calculation for high-cardinality dimensions.
It explains why CID uses a volume-tiered PSI threshold of 0.25.]

The PSI threshold for high-cardinality dimensions like CID...
```

The chunk is unchanged but the embedding includes the context — retrieval finds it even when the question uses different vocabulary. The LLM also has context to correctly interpret the chunk.

**Good fit:** Almost any domain — this is an augmentation on top of any chunking strategy, not a replacement. High value for your DQ documentation.
**Bad fit:** Very large document sets where LLM context generation cost is prohibitive.

### Parent-document retrieval
Index small chunks for precise retrieval, but return the larger parent chunk to the LLM.

```
Parent (512 tokens): Full section on PSI — formula, thresholds, worked example
Children (128 tokens each):
  chunk_a: PSI formula definition
  chunk_b: threshold values by dimension
  chunk_c: worked example for CID
```

Vector index holds child chunks — small, precise retrieval. LLM receives parent chunk — rich surrounding context.

Solves the core chunking tension: small chunks retrieve precisely but provide thin context. Large chunks provide context but retrieve imprecisely.

**Good fit:** Technical runbooks, legal contracts — your DQ runbook is a strong fit.
**Bad fit:** Documents without natural parent-child structure.

### Chunking decision framework

| Document type | Recommended strategy |
|---|---|
| Technical runbooks, structured docs | Structure-aware + contextual |
| Legal contracts | Structure-aware + parent-document |
| Meeting notes, transcripts | Semantic chunking |
| Uniform reports, logs | Fixed-size with overlap |
| Mixed enterprise knowledge base | Parent-document + contextual |
| Your DQ documentation | Structure-aware + contextual |

**Size guidance:** Small chunks (128-256 tokens) = precise retrieval, thin context. Large chunks (512-1024 tokens) = rich context, imprecise retrieval. Most production systems: 256-512 tokens for indexed chunk + parent-document retrieval for LLM context.

---

## Retrieval Evaluation Metrics

How do you know if your RAG retrieval is actually working?

**Precision@K** — of the K chunks retrieved, what fraction were actually relevant?
High precision = what you retrieved was useful. Low precision = noise in the context window.

**Recall@K** — of all relevant chunks in the knowledge base, what fraction did you retrieve?
High recall = you found everything important. Low recall = you missed key information.

**MRR (Mean Reciprocal Rank)** — how high up was the first relevant chunk in your ranked results?
MRR = 1 if the first result was relevant, 0.5 if the second was first relevant, etc.
High MRR = relevant chunks surface near the top, which matters for re-ranking.

**The tradeoff:** Precision and recall pull against each other. Retrieving more chunks (higher K) improves recall but dilutes precision. Re-ranking helps by improving precision without sacrificing recall — cast a wide net, then sharpen.

---

## The Emerging Frontier: KAG

**Knowledge Augmented Generation** — the layer above GraphRAG.

KAG's critique of RAG: vector similarity finds things that *look* related but doesn't understand *how* things are logically connected. KAG combines a knowledge graph with **explicit symbolic reasoning** — the LLM doesn't just read retrieved context, it reasons along logical paths in the graph the way a human expert would.

Where GraphRAG assembles context by traversal, KAG uses that traversal as a **reasoning scaffold**:

```
RAG:      retrieve relevant chunks → generate
GraphRAG: traverse entity graph → assemble context → generate
KAG:      traverse entity graph → reason along logical paths → derive answer
```

**When KAG matters:** When GraphRAG gives you the right context but the LLM still struggles to reason through it correctly. Specifically: multi-hop logical inference chains ("if regulation A requires X, and X implies Y, and our policy satisfies Y, then we comply with A").

**Current state:** Actively developing (2024-2025), mostly research and early production in specialized domains — pharma, legal, financial compliance. Not yet mainstream. Worth knowing exists; not yet worth going deep on for most production builds.

---

## Connection to Your Stack

| Your asset | RAG role |
|---|---|
| Score history QVD (90 days) | Knowledge base for Structured RAG |
| Pipeline metadata (Oracle/Hive) | Direct graph construction — skip LLM extraction |
| DQ alert logs | Retrieval corpus for anomaly agent |
| Config CSVs | Structured retrieval source |
| Composite scoring runbook | Hierarchical RAG or structure-aware chunking |
| MCP server layer | Agentic RAG orchestration — tool calls per retrieval source |

**Recommended architecture for your DQ anomaly agent:**

```
Alert fires
      ↓
Query decomposition     → split "what broke and who owns it?" into sub-queries
      ↓
Structured RAG          → HQL against score history QVD for metrics
      ↓
Graph traversal         → pipeline → team ownership chain (built from metadata)
      ↓
Hybrid search + rerank  → retrieve relevant runbook sections for context
      ↓
Agentic loop            → iterate if root cause not yet clear
      ↓
Self-RAG check          → verify numbers are grounded before stakeholder output
      ↓
LLM generates summary   → root cause + recommended action + owner
```

---

## One-Line Summary of Every Technique

| Technique | One line |
|---|---|
| Naive RAG | Embed, search, generate — the baseline |
| Query rewriting | Fix the question before searching |
| HyDE | Hallucinate an answer, embed that instead of the question |
| Query decomposition | Split complex questions into independent sub-queries |
| Hybrid search | Keyword + semantic, scores blended via RRF |
| Re-ranking | Bi-encoder casts wide net, cross-encoder sharpens it |
| Iterative retrieval | Each fetch discovers what to fetch next |
| GraphRAG | Traverse entity relationships across documents |
| Hierarchical RAG | Summary layer + detail layer, route by question breadth |
| Structured RAG | NL → SQL/HQL → real database result → generate |
| Agentic RAG | LLM decides what to fetch, when, and whether to loop |
| Self-RAG | Model grades its own retrieval and groundedness |
| Contextual chunking | Prepend context summary to every chunk before indexing |
| Parent-document retrieval | Retrieve small chunks, return large parent to LLM |
| KAG | GraphRAG + explicit symbolic reasoning along logical paths |

---

*Built from a learning conversation — DQ scoring engine and fintech data engineering context. Companion document: Graph DB Reference.*
