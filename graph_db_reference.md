# Graph Databases — Personal Reference
*10-minute refresher. Built from first principles, anchored to a fintech / data engineering context.*

---

## The One-Line Mental Model

> A graph database stores **relationships as first-class objects** — not implied by foreign keys, not computed at query time, but physically stored with their own identity and properties.

If you remember nothing else: in RDBMS, relationships are **computed** (via JOIN). In a graph DB, relationships are **stored** (and traversed).

---

## Core Vocabulary

| Graph term | RDBMS equivalent | Key difference |
|---|---|---|
| Node | Row | Has a label (type), not a table |
| Node label | Table name | A node can have multiple labels |
| Node property | Column value | Schema is flexible — nodes of same label can have different properties |
| Relationship | FK + JOIN | Stored as an object, has direction, has its own properties |
| Relationship type | Junction table | Named, directed, carries data |
| Traversal | JOIN chain | Follows stored pointers — O(1) per hop |
| Variable-length traversal | Recursive CTE | Built-in, cycle-safe, one line of syntax |

---

## The Real-Life Example: Fraud Detection at a Fintech

**The problem a table can't solve cleanly:**

Account A sends money to Account B — looks clean.
Account B shares a device with Account C — different table.
Account C shares a phone number with Account D — another table.
Account D was flagged for fraud last month.

No single row contains this signal. In SQL: four joins, and you only find it if you knew to look exactly four hops deep. In a graph: one variable-length traversal finds the entire ring regardless of depth.

This pattern — a network of connected accounts that are individually clean but collectively fraudulent — is called a **fraud ring**. It's invisible in rows. Obvious in a graph.

---

## Domain Model: How to Design a Graph Schema

### Step 1 — Identify your nodes (the nouns)

Ask: what are the independent entities in my domain?

```
:Customer     { customer_id, name, kyc_status, risk_score }
:Account      { account_id, account_type, status, balance }
:Transaction  { transaction_id, amount, currency, timestamp, status }
:Device       { device_id, device_type, fingerprint_hash }
:Merchant     { merchant_id, name, category, risk_category }
:IP_Address   { ip_address, country, is_vpn, is_tor }
```

**Rule of thumb — node vs property:**
If other things need to independently connect to it → make it a **node**.
If it only describes one thing and nothing else links to it → make it a **property**.

*IP address: multiple transactions share the same IP → node.*
*Currency: just a label on a transaction, nothing connects to "USD" → property.*

### Step 2 — Identify your relationships (the verbs)

Ask: how do these things connect? Give every connection a name and a direction.

```
(:Customer)    -[:HAS_ACCOUNT]->   (:Account)
(:Account)     -[:INITIATED]->     (:Transaction)
(:Transaction) -[:TO]->            (:Account)
(:Transaction) -[:MADE_FROM]->     (:Device)
(:Transaction) -[:MADE_FROM]->     (:IP_Address)
(:Transaction) -[:AT]->            (:Merchant)
```

Read each line as a sentence. "A Customer HAS an Account." "A Transaction was MADE_FROM a Device."

### Step 3 — Add properties to relationships

This is the part with no clean relational equivalent. Relationships carry their own data:

```
[:HAS_ACCOUNT]   { since, is_primary }
[:MADE_FROM]     { first_seen, times_used }
[:SENT]          { amount, timestamp, channel }
```

`first_seen` on the `MADE_FROM` relationship means: "this transaction came from a device this account had never used before" — answerable directly from the edge, no aggregation query needed.

---

## Schema Definition (Constraints + Indexes)

Graph DBs are **permissive by default** — they accept data without a pre-defined schema. This is flexible but dangerous in production. You enforce structure explicitly.

**The mindset shift:**
- Oracle schema = the **floor** (nothing exists until defined)
- Graph DB schema = the **fence** (things can exist freely, constraints stop bad data)

### Constraints (your DDL equivalent)

```cypher
-- Uniqueness (also auto-creates an index — like a PK)
CREATE CONSTRAINT account_id_unique
FOR (a:Account) REQUIRE a.account_id IS UNIQUE;

-- Existence (NOT NULL equivalent)
CREATE CONSTRAINT account_status_exists
FOR (a:Account) REQUIRE a.status IS NOT NULL;

-- Relationship property existence
CREATE CONSTRAINT transaction_amount_exists
FOR ()-[t:SENT]-() REQUIRE t.amount IS NOT NULL;

-- Composite key (like a composite PK)
CREATE CONSTRAINT transaction_key
FOR (t:Transaction)
REQUIRE (t.account_id, t.timestamp) IS NODE KEY;
```

### Indexes (for entry-point lookups only)

```cypher
CREATE INDEX account_status_index
FOR (a:Account) ON (a.status);

CREATE INDEX transaction_timestamp_index
FOR (t:Transaction) ON (t.timestamp);
```

**Critical insight:** Indexes are only needed for the **entry point** — the first node you find to start traversal from. Once you're in the graph, traversal follows pointers (O(1) per hop) — no index needed for the hops themselves. This is different from RDBMS where every join benefits from indexes.

---

## Querying: Cypher Basics

Every Cypher query follows the same structure:

```cypher
MATCH  <path pattern>    -- like FROM + JOIN
WHERE  <filter>          -- like WHERE
RETURN <output>          -- like SELECT
```

### Simple lookup (1 table)

```cypher
MATCH (a:Account {account_id: 'ACC_001'})
RETURN a
```
```sql
-- Oracle equivalent
SELECT * FROM account WHERE account_id = 'ACC_001'
```

### One hop (1 join)

```cypher
MATCH (a:Account {account_id: 'ACC_001'})-[:INITIATED]->(t:Transaction)
RETURN t.transaction_id, t.amount, t.timestamp
```
```sql
-- Oracle equivalent
SELECT t.transaction_id, t.amount, t.timestamp
FROM account a
JOIN transaction t ON a.account_id = t.from_account_id
WHERE a.account_id = 'ACC_001'
```

No JOIN keyword. No ON condition. You drew the path — the stored relationship knows what connects to what.

### Two hops (2 joins) — graphs start pulling ahead

```cypher
MATCH (a:Account {account_id: 'ACC_001'})
      -[:INITIATED]->(t:Transaction)
      -[:TO]->(dest:Account)
RETURN t.transaction_id, t.amount, dest.account_id
```

You just extended the path pattern one step. In Oracle, you'd introduce a second JOIN and alias the account table twice. At 5 hops, the Cypher query is still one readable sentence. The SQL is a nested join chain.

### Filtering on relationship properties

```cypher
MATCH (a:Account {account_id: 'ACC_001'})
      -[:INITIATED]->(t:Transaction)
      -[r:MADE_FROM]->(d:Device)
WHERE r.first_seen = t.timestamp   -- r is a variable bound to the relationship itself
RETURN t.transaction_id, t.amount, d.device_id
```

`r` is a variable holding the relationship object. `r.first_seen` is a property on that relationship. Filtering on relationship properties — no direct SQL equivalent without a junction table join.

---

## Variable-Length Traversal

The feature with no clean RDBMS equivalent. Used when the depth of the answer is **unknown or variable** upfront.

### Syntax

```cypher
-[:SENT*1..3]->     -- between 1 and 3 hops
-[:SENT*3]->        -- exactly 3 hops
-[:SENT*0..3]->     -- 0 to 3 hops (includes start node itself)
-[:SENT*2..]-       -- at least 2 hops, no upper limit (use carefully)
-[:SENT*]->         -- any depth — dangerous on large graphs
```

### Fraud ring detection

```cypher
MATCH (a:Account {account_id: 'ACC_001'})
      -[:SENT*1..3]->(suspicious:Account)
WHERE suspicious.account_id <> 'ACC_001'
RETURN DISTINCT suspicious.account_id, suspicious.status
```

```sql
-- Oracle equivalent (recursive CTE)
WITH RECURSIVE ring AS (
  SELECT to_account_id, 1 as depth
  FROM transaction WHERE from_account_id = 'ACC_001'
  UNION ALL
  SELECT t.to_account_id, r.depth + 1
  FROM transaction t
  JOIN ring r ON t.from_account_id = r.to_account_id
  WHERE r.depth < 3 AND t.to_account_id <> 'ACC_001'
)
SELECT DISTINCT to_account_id FROM ring;
```

In Cypher you expressed **intent**. In SQL you expressed **mechanics** — base case, recursive case, depth tracking, termination condition.

### Shortest path (no RDBMS equivalent)

```cypher
MATCH path = shortestPath(
  (a:Account {account_id: 'ACC_001'})
  -[:SENT*]->
  (f:Account {account_id: 'ACC_999'})
)
RETURN path, length(path) as hops
```

Find the minimum-hop chain between two nodes across any depth. The database runs BFS internally. You don't specify depth, track visited nodes, or rank results.

### Why depth limits matter

- **Cycles:** Graph DBs track visited nodes internally — no infinite loops. But...
- **Combinatorial explosion:** A node with average degree 5 has 5¹ neighbors at depth 1, 5² at depth 2, 5³ at depth 3. Unbounded traversal on a dense graph visits millions of nodes.
- **Business logic:** Most domains have a meaningful max depth. Pipeline lineage rarely exceeds 8 hops. Fraud rings rarely exceed 5-6 accounts. Set your limit from domain knowledge, not just as a safety cap.

---

## Multiple Relationships Between the Same Nodes

Both are valid and useful:

**Different relationship types:**
```
(Customer: John) -[USES_DEVICE]->    (Device: DEV_A)
(Customer: John) -[REPORTED_STOLEN]-> (Device: DEV_A)
```
Same two nodes, two relationship types. Both facts coexist. Schema change not required to add the second type.

**Multiple relationships of the same type:**
```
(ACC_001) -[SENT, amount: 200,  timestamp: Jan]-> (ACC_002)
(ACC_001) -[SENT, amount: 4500, timestamp: Mar]-> (ACC_002)
(ACC_001) -[SENT, amount: 50,   timestamp: Jun]-> (ACC_002)
```
Three SENT relationships, same two nodes. Each carries its own properties. The pattern (repeated same-amount transfers) is a fraud signal called **smurfing** — visible as a relationship pattern, invisible as a row.

**Watch out:** traversal returns one row per relationship, not per node pair. Aggregate explicitly when needed:
```cypher
MATCH (a:Account)-[t:SENT]->(b:Account)
RETURN a.account_id, b.account_id, count(t) as transfers, sum(t.amount) as total
```

---

## Self-Referential Relationships

**Self-loop (node relates to itself):**
```
(ACC_001) -[SENT, amount: 5000, timestamp: ...]-> (ACC_001)
```
Valid when the connection is an event with its own data (amount, timestamp). A fraudulent internal transfer earns being a self-loop. A flag saying "this account has done this before" should just be a property.

**Same-label relationships (more common and important):**
```
(Pipeline: ETL_RAW)    -[FEEDS]-> (Pipeline: ETL_TRANSFORM)
(Pipeline: ETL_TRANSFORM) -[FEEDS]-> (Pipeline: ETL_LOAD_DQ)
```
All Pipeline nodes, relating to each other. This is your data lineage graph. Variable-length traversal across same-label relationships is the natural way to answer "what is the full upstream dependency chain of this pipeline?"

---

## Physical Storage: Why Graph DBs Are Fast at Traversal

**Native graph storage (Neo4j, Amazon Neptune):**
Each node record contains a direct pointer to its first edge. Each edge record contains pointers to the next edge from the same node. Finding all neighbors = follow a chain of pointers. **O(1) per hop, regardless of graph size.**

**Simulated graph in RDBMS:**
Finding neighbors requires `SELECT * FROM edges WHERE from_id = X`. Even with an index, that's O(log n). Compounds with every hop. 5 hops = 5 index lookups = latency multiplies.

This property of native graph storage is called **index-free adjacency** — traversal never touches an index. Indexes are only for the initial node lookup.

---

## When Graph DB Wins vs When RDBMS Wins

**Use a graph when:**
- The question involves variable or unknown depth ("what is the full upstream lineage?")
- The answer IS the path, not just the endpoint
- Multiple relationship types need to be traversed in one query
- Highly connected data where SQL joins compound badly
- Relationship properties matter for filtering

**Stay with RDBMS / Hadoop when:**
- Fixed, known depth with no expectation of change
- Heavy aggregations (GROUP BY, SUM, COUNT over millions of rows)
- Simple entity lookups with no traversal
- Time-series data or BI reporting
- Your team has no graph expertise and the operational cost isn't justified

**Real domains that consistently favor graphs:**
- Fraud detection and AML (fraud rings, smurfing patterns)
- Data lineage and impact analysis (your DQ world)
- Social networks and recommendation engines
- Knowledge graphs and enterprise search
- IT/network infrastructure dependency mapping
- Supply chain impact analysis
- Identity resolution (deduplicating accounts through shared identifiers)

---

## Connection to Your DQ World

Your DQ engine is already a graph in disguise:

```
(Alert) -[TRIGGERED_ON]-> (Dimension: CID)
        -[FED_BY]->        (Pipeline: ETL_LOAD_CID)
        -[OWNED_BY]->      (Team: Data Eng A)
        -[REPORTS_TO]->    (Manager: ...)
```

Questions your current stack answers badly but a graph handles naturally:
- "What is the full upstream lineage of this failing dimension?"
- "Which team owns the pipeline that caused this PSI breach?"
- "If I change this source table, what DQ scores are at risk downstream?"
- "Which dimensions tend to co-breach? Are they fed by the same pipeline?"

The last one — co-breach clusters — is a graph community detection problem. Dimensions that frequently fail together form a cluster, and the cluster usually points to a shared upstream cause.

---

## Quick Reference: Cypher Cheat Sheet

```cypher
-- Find a node
MATCH (n:Label {property: 'value'}) RETURN n

-- Follow a relationship
MATCH (a:Account)-[:SENT]->(b:Account) RETURN a, b

-- Filter on relationship property
MATCH (a)-[r:SENT]->(b) WHERE r.amount > 10000 RETURN a, b, r.amount

-- Variable length traversal
MATCH (a)-[:SENT*1..4]->(b) RETURN b

-- Shortest path
MATCH p = shortestPath((a)-[*]->(b)) RETURN p, length(p)

-- Count relationships between node pairs
MATCH (a)-[r:SENT]->(b)
RETURN a.account_id, b.account_id, count(r), sum(r.amount)

-- Create a node
CREATE (a:Account {account_id: 'ACC_001', status: 'active'})

-- Create a relationship
MATCH (a:Account {account_id: 'ACC_001'})
MATCH (b:Account {account_id: 'ACC_002'})
CREATE (a)-[:SENT {amount: 500, timestamp: datetime()}]->(b)

-- Create constraint
CREATE CONSTRAINT FOR (a:Account) REQUIRE a.account_id IS UNIQUE

-- Create index
CREATE INDEX FOR (a:Account) ON (a.status)
```

---

*Built from a learning conversation — fraud detection domain, fintech data engineering context. Companion document: RAG Architecture Reference.*
