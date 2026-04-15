# Search Systems — Architecture & Concepts

## 1. The Problem with Relational DB Search

```sql
-- This works for exact matches and prefix searches:
SELECT * FROM products WHERE name = 'iPhone 15';
SELECT * FROM products WHERE name LIKE 'iPhone%';

-- This is catastrophically slow at scale:
SELECT * FROM products WHERE name LIKE '%iPhone%';
-- Full table scan, no index used, O(N) per query
```

For full-text search — "find documents containing any of these words, ranked by relevance" — relational databases are fundamentally the wrong tool. A purpose-built **inverted index** is required.

---

## 2. Inverted Index

The core data structure behind all search engines.

**Forward index (naive):**
```
doc_1 → ["the", "quick", "brown", "fox"]
doc_2 → ["the", "lazy", "fox", "jumped"]
```
Finding documents containing "fox" requires scanning all documents — O(N).

**Inverted index:**
```
"the"    → [doc_1, doc_2]
"quick"  → [doc_1]
"brown"  → [doc_1]
"fox"    → [doc_1, doc_2]
"lazy"   → [doc_2]
"jumped" → [doc_2]
```
Finding documents containing "fox" → O(1) lookup, O(k) traversal where k = number of matching docs.

**Posting list:** The list of document IDs (plus positions, term frequencies) for each term.

```
"fox" → [(doc_1, freq=1, positions=[4]), (doc_2, freq=1, positions=[3])]
```

**Building the index:**
```
1. Tokenize text into terms (split on whitespace, punctuation)
2. Apply text analysis pipeline:
   - Lowercase: "Fox" → "fox"
   - Remove stop words: "the", "a", "is" → dropped
   - Stem or lemmatize: "running" → "run"
3. For each (term, doc_id, position): insert into the posting list
```

---

## 3. Text Analysis Pipeline

```
Input: "The Quick Brown Fox Jumped Over the Lazy Dog"
         │
         ▼
Tokenizer: ["The", "Quick", "Brown", "Fox", "Jumped", "Over", "the", "Lazy", "Dog"]
         │
         ▼
Lowercase filter: ["the", "quick", "brown", "fox", "jumped", "over", "the", "lazy", "dog"]
         │
         ▼
Stop word removal: ["quick", "brown", "fox", "jumped", "lazy", "dog"]  ← "the", "over" removed
         │
         ▼
Stemmer (Porter): ["quick", "brown", "fox", "jump", "lazi", "dog"]
         │
         ▼
Indexed terms: { "quick":1, "brown":1, "fox":1, "jump":1, "lazi":1, "dog":1 }
```

**Stemming vs. Lemmatization:**
- Stemmer (rule-based, fast): "running" → "run", "better" → "better"... but "dogs" → "dog", "argued" → "argu"
- Lemmatizer (dictionary-based, slower): "better" → "good", "running" → "run" — linguistically correct
- Elasticsearch default: Porter stemmer (fast, good enough)

---

## 4. TF-IDF (Term Frequency–Inverse Document Frequency)

How relevant is document D for query term T?

$$\text{TF-IDF}(t, d) = \text{TF}(t, d) \times \text{IDF}(t)$$

**TF (Term Frequency):** How often does term t appear in document d?
$$\text{TF}(t, d) = \frac{\text{count}(t \text{ in } d)}{\text{total terms in } d}$$

**IDF (Inverse Document Frequency):** How rare is term t across all documents?
$$\text{IDF}(t) = \log\left(\frac{N}{1 + \text{df}(t)}\right)$$

Where N = total documents, df(t) = documents containing t.

**Intuition:**
- "the" appears in every document → IDF ≈ 0 → near-zero score → stop words naturally de-ranked
- "elasticsearch" appears in 100 of 1M docs → high IDF → rare term → high score when matched
- A document with "elasticsearch" 10 times in a short text → high TF → ranked higher

---

## 5. BM25 (Best Match 25)

BM25 is the modern replacement for TF-IDF — Elasticsearch's default scoring since 5.x.

$$\text{BM25}(t, d) = \text{IDF}(t) \times \frac{\text{TF}(t,d) \times (k_1 + 1)}{\text{TF}(t,d) + k_1 \times (1 - b + b \times \frac{|d|}{\text{avgdl}})}$$

**Parameters:**
- $k_1 = 1.2$: controls TF saturation — beyond a point, more occurrences add diminishing returns
- $b = 0.75$: length normalization — long documents are penalized for having more term occurrences by chance

**BM25 vs TF-IDF improvements:**
- **TF saturation:** TF-IDF scores grow unboundedly with term frequency. BM25's TF contribution asymptotes at $k_1 + 1$ — mentioning a word 100 times isn't 100× better than mentioning it 10 times.
- **Length normalization:** TF-IDF doesn't account for document length. BM25 normalizes against average document length — a short document with 5 occurrences of "cat" is more relevant than a 100-page book with 5 occurrences.

---

## 6. Elasticsearch Architecture

Elasticsearch is a distributed search engine built on Apache Lucene.

### Core Concepts

| Concept | Description | Analogy |
|---|---|---|
| **Index** | Collection of documents with a common schema | Database |
| **Document** | JSON object with searchable fields | Row |
| **Shard** | Lucene index instance; unit of distribution | Partition |
| **Replica** | Copy of a shard; for HA and read scaling | Replica |
| **Node** | JVM process; hosts shards | Server |
| **Cluster** | Collection of nodes with one master | Database cluster |

### Sharding
```
Index: "products" with 3 primary shards
Primary shard assignment: shard = hash(document_id) % num_primary_shards

Node 1: Primary-0, Replica-1
Node 2: Primary-1, Replica-2
Node 3: Primary-2, Replica-0

Reads: can go to any primary or replica shard (round-robin)
Writes: go to primary shard → replicated to replicas synchronously
```

**Immutable shards:** Once a Lucene segment is written, it's immutable. Updates are implemented as: write new version + mark old version deleted. Periodic **segment merge** cleans up deleted docs and compacts smaller segments.

### Query flow
```
1. Client sends query to any node (coordinate node)
2. Coordinate node fans out to all shards (or relevant shards with routing)
3. Each shard executes Lucene query → returns top-K local results (doc_id + score)
4. Coordinate node merges all top-K results across shards → global top-K
5. Coordinate node fetches source documents for the final top-K from their shards
6. Returns result to client
```

**The "scatter-gather" pattern:** Phase 1 (query) fetches just IDs + scores from all shards; Phase 2 (fetch) retrieves full documents only for the final page.

### Near Real-Time (NRT) vs. Real-Time
- Documents indexed to an in-memory buffer (not yet searchable)
- Default **refresh interval: 1 second** — buffer flushed to a new Lucene segment → becomes searchable
- This is **near real-time**: up to 1 second lag between indexing and visibility
- `refresh=true` on individual requests for immediate visibility (expensive)

---

## 7. Inverted Index Data Structures

### Finite State Transducer (FST)
The term dictionary in Lucene is stored as an FST — a compact finite automaton mapping terms to their posting list offsets. Allows O(log N) term lookup with minimal memory overhead.

### Posting List Compression
Posting lists use:
- **Delta encoding:** store differences between consecutive doc IDs (small numbers = fewer bits)
- **Variable-byte encoding:** use fewer bytes for small numbers
- **PFOR (Patched Frame Of Reference):** batch encode groups of 128 doc IDs using minimal bits per block

A posting list for "the" in a million-document index might have 999,000 entries — compression is essential.

---

## 8. Full-Text Search Design Patterns

### Query Types in Elasticsearch

| Query | What it does | When to use |
|---|---|---|
| `match` | Analyzes query text, searches inverted index | Standard full-text search |
| `match_phrase` | Requires terms in exact order and proximity | "quick brown fox" |
| `multi_match` | `match` across multiple fields | Search title + description |
| `term` | Exact match, no analysis (case-sensitive) | Filter by status="active" |
| `range` | Numeric/date range | price ≥ 100 AND ≤ 500 |
| `bool` | Combine must/should/must_not/filter clauses | Compound queries |

### The filter vs. query distinction
- **Query context:** contributes to relevance score (BM25)
- **Filter context:** binary yes/no, no scoring, results cached by Elasticsearch

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "elasticsearch" }}    ← scored (relevance)
      ],
      "filter": [
        { "term": { "status": "published" }},        ← not scored, cached
        { "range": { "published_at": { "gte": "2024-01-01" }}}
      ]
    }
  }
}
```

### Indexing strategy
- Store only what needs to be searched in Elasticsearch; use a relational DB for full record
- Denormalize: embed author name, category name directly in the document (avoid joins)
- Use `_source: false` on fields not needed in the response to save disk I/O

---

## 9. Relevance Tuning Techniques

| Technique | How | When |
|---|---|---|
| **Field boosting** | `"title^3, description^1"` | Title match more important than body |
| **Function score** | Multiply BM25 by a field value (e.g., popularity) | Boost popular products |
| **Synonyms** | Token filter: "car" → ["car", "automobile"] | Domain-specific vocabulary |
| **Shingles** | N-gram tokens: "quick brown" as single term | Phrase matching performance |
| **Completion suggester** | FST-based prefix suggestion | Autocomplete / type-ahead |

### Hybrid search (modern approach)
Combine BM25 with vector similarity (dense retrieval) using **Reciprocal Rank Fusion (RRF)**:

```
1. BM25 score: keyword relevance → rank list A
2. Vector cosine similarity (embedding): semantic relevance → rank list B
3. RRF(A, B) = sum(1 / (k + rank_in_A), 1 / (k + rank_in_B))
4. Final ranking by RRF score
```

Elasticsearch 8.x supports HNSW-based vector search natively. Hybrid search is now production-standard for e-commerce and document retrieval.
