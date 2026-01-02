# Anti-Money Laundering (AML): Detecting Mule Rings at Scale with Neo4j

## Overview

Money laundering schemes frequently rely on **money mule rings** — organized networks of accounts that move illicit funds through multiple layers of intermediaries (mules) to obscure the source. These rings often involve cycles or long chains where funds are passed forward, with each mule taking a small fee, resulting in slightly decreasing transaction amounts over time.

A classic suspicious pattern is a **cycle of transactions** returning to the originator (or forming a closed loop) with:
- Strictly increasing transaction dates (funds moved forward in time).
- Strictly decreasing amounts (typically 80–100% of the previous amount, reflecting ~10–20% fees per hop).
- Multiple hops (2 or more).

Traditional AML systems based on rules or relational databases struggle to detect such multi-hop, property-constrained cycles efficiently in large datasets containing millions of transactions.

**Neo4j** is uniquely suited for this because it treats relationships as first-class citizens and excels at variable-length path queries with complex conditions — enabling true scale for graph-based AML.

This collection of saved Cypher queries implements a complete **"needle in a haystack" testing workflow**:
1. **optional** -- Clean the database.
2. Load a large realistic dataset (~100K transactions) (the **haystack**).
3. **optional** -- Inject a known synthetic mule ring (the **needle**).
4. Detect mule ring patterns across the entire graph.

The synthetic ring acts as a **needle in a haystack**: a deliberately planted, detectable pattern hidden within a much larger, mostly benign transaction graph. This allows validation that the detection query actually finds real mule-like structures even at scale.

## Workflow and Queries Explained

### 1. Clean the Database
```cypher
// Drops all indexes and constraints
CALL apoc.schema.assert({},{});

// Deletes all data
MATCH (n)
CALL { WITH n DETACH DELETE n } IN TRANSACTIONS OF 100 ROWS
```

**Purpose**: Completely reset the database — useful for reproducible testing or demos.

### 2. Ingest Large Dataset (~100K Transactions)
```cypher
:param  file_path_root => 'https://raw.githubusercontent.com/halftermeyer/fraud-detection-training/main/data/data_csv/big/';
:param file_0 => 'accounts.csv';
:param file_1 => 'txs.csv';

// CONSTRAINT AND INDEX creation
// -------------------
//
// Create node uniqueness constraints, ensuring no duplicates for the given node label and ID property exist in the database. This also ensures no duplicates are introduced in future.
//
// NOTE: The following constraint creation syntax is generated based on the current connected database version 5.12-aura.
CREATE CONSTRAINT `imp_uniq_Account_a_id` IF NOT EXISTS
FOR (n: `Account`)
REQUIRE (n.`a_id`) IS UNIQUE;

CREATE INDEX transaction_date IF NOT EXISTS
    FOR ()-[r:TRANSACTIONS]->() 
    ON (r.date);

CREATE INDEX transaction_amount IF NOT EXISTS
    FOR ()-[r:TRANSACTIONS]->() 
    ON (r.amount);

:param idsToSkip => [];

// NODE load
// ---------
//
// Load nodes in batches, one node label at a time. Nodes will be created using a MERGE statement to ensure a node with the same label and ID property remains unique. Pre-existing nodes found by a MERGE statement will have their other properties set to the latest values encountered in a load file.
//
// NOTE: Any nodes with IDs in the 'idsToSkip' list parameter will not be loaded.
LOAD CSV WITH HEADERS FROM ($file_path_root + $file_0) AS row
WITH row
WHERE NOT row.`a_id` IN $idsToSkip AND NOT row.`a_id` IS NULL
CALL {
  WITH row
  MERGE (n: `Account` { `a_id`: row.`a_id` })
  SET n.`name` = row.`name`
  SET n.`email` = row.`email`
} IN TRANSACTIONS OF 10000 ROWS;


// RELATIONSHIP load
// -----------------
//
// Load relationships in batches, one relationship type at a time. Relationships are created using a MERGE statement, meaning only one relationship of a given type will ever be created between a pair of nodes.
LOAD CSV WITH HEADERS FROM ($file_path_root + $file_1) AS row
WITH row 
CALL {
  WITH row
  MATCH (source: `Account` { `a_id`: row.`from_id` })
  MATCH (target: `Account` { `a_id`: row.`to_id` })
  CREATE (source)-[r: `TRANSACTION`]->(target)
  SET r.`tx_id` = row.`tx_id`
  SET r.`amount` = toFloat(trim(row.`amount`))
  SET r.`date` = datetime(row.`date`)
  // Your script contains the datetime datatype. Our app attempts to convert dates to ISO 8601 date format before passing them to the Cypher function.
  // This conversion cannot be done in a Cypher script load. Please ensure that your CSV file columns are in ISO 8601 date format to ensure equivalent loads.
  // SET r.`date` = datetime(row.`date`)
} IN TRANSACTIONS OF 10000 ROWS;
```

**Purpose**: Creates a perfect cycle of 100 accounts forming a closed ring:
- Each step transfers money to the next account.
- Amounts decrease progressively (e.g., 99,999 → 99,998 → ... simulating fees).
- Dates increase along the path (older → newer as money moves forward).
- Marked with `test: true` for easy verification.

This synthetic ring is the **needle** — a known, detectable mule pattern deliberately hidden in the large random dataset.

### 3. Add Needle in Haystack

Adding a big fraud cycle

``` cypher
WITH 100 AS length
UNWIND range(1,length) AS ix
MERGE (a:Account {a_id:toString(ix)})
MERGE (b:Account {a_id:toString(CASE (ix+1)%length WHEN  0 THEN length ELSE (ix+1)%length END)})
CREATE (a)-[:TRANSACTION {test:true, amount: (1000*length)-ix, date: datetime()-duration({days: length - ix})}]->(b);
```

### 4. Find Mule Rings (Detection Query)
```cypher
CYPHER 25 runtime=parallel
MATCH path=(a:Account)-[txs:TRANSACTION]->{2,}(a)
WHERE allReduce(
  span = {},
  tx IN txs | {
    previous: span.current,
    current: tx{.amount, .date}
    },
    span.previous.date IS NULL OR
    (span.previous.date < span.current.date
    AND span.previous.amount
        > span.current.amount
        > 0.8 * span.previous.amount)
  )
RETURN path
```
<img width="3200" height="1253" alt="Capture d’écran 2025-12-19 à 16 42 42" src="https://github.com/user-attachments/assets/2441f59c-08d0-46f1-997f-956291f4a27c" />


**Purpose**: Finds all cycles (length ≥ 2) where:
- Transaction dates are strictly increasing.
- Amounts are strictly decreasing.
- Each amount is more than 80% of the previous (allowing up to ~20% fee per mule).

This query should reliably return the synthetic 100-node ring (the needle) while potentially flagging any similar real patterns in the haystack.

## Why This "Needle in a Haystack" Approach Matters

- **Validation at Scale**: Proves the detection logic works even when suspicious patterns are rare and buried in massive benign data.
- **Performance Testing**: Demonstrates Neo4j's ability to traverse and filter variable-length paths efficiently on large graphs.
- **Real-World Simulation**: Mirrors actual AML challenges — mule rings are infrequent but high-impact.
- **Extensibility**: Easily adjust thresholds (e.g., min cycle length, max fee %, amount floors) or combine with GDS algorithms (community detection, centrality).

This workflow provides a powerful, reproducible framework for developing and validating advanced graph-based AML detection of money mule rings at true financial-scale volumes.
