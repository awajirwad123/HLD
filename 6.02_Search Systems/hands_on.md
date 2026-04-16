# Search Systems — Hands-On Exercises

## Exercise 1: Build an Inverted Index from Scratch

Understand the mechanics before using any search engine:

```python
"""
Goal: Build an in-memory inverted index that supports:
  - Document indexing with term analysis (tokenize, lowercase, stop words, stemming)
  - BM25 scoring
  - Boolean search (AND with minimum term frequency)
  - Phrase search (terms must be adjacent)
"""

import math
import re
from collections import defaultdict
from dataclasses import dataclass, field
from typing import Optional


# ─── Text Analysis ────────────────────────────────────────────────────────────

STOP_WORDS = {
    "the", "a", "an", "and", "or", "but", "in", "on", "at", "to",
    "for", "of", "with", "by", "is", "was", "are", "were", "be",
    "has", "have", "had", "do", "does", "did", "will", "would",
    "could", "should", "may", "might", "shall", "can", "not",
    "this", "that", "these", "those", "it", "its", "from", "as",
}


def simple_stem(word: str) -> str:
    """Very simple rule-based stemmer for demo purposes."""
    if word.endswith("ing") and len(word) > 6:
        return word[:-3]
    if word.endswith("tion") and len(word) > 7:
        return word[:-4]
    if word.endswith("ness") and len(word) > 6:
        return word[:-4]
    if word.endswith("ly") and len(word) > 4:
        return word[:-2]
    if word.endswith("ies") and len(word) > 5:
        return word[:-3] + "y"
    if word.endswith("s") and len(word) > 3 and not word.endswith("ss"):
        return word[:-1]
    return word


def analyze(text: str) -> list[str]:
    """
    Analysis pipeline: tokenize → lowercase → remove stop words → stem.
    Returns list of analyzed tokens (with positions preserved by index in the list).
    """
    tokens = re.findall(r"[a-zA-Z']+", text)
    result = []
    for tok in tokens:
        tok = tok.lower().strip("'")
        if len(tok) < 2 or tok in STOP_WORDS:
            continue
        result.append(simple_stem(tok))
    return result


# ─── Posting Entry ────────────────────────────────────────────────────────────

@dataclass
class Posting:
    doc_id: int
    term_freq: int
    positions: list[int]   # 0-based token positions in the document


# ─── Inverted Index ───────────────────────────────────────────────────────────

class InvertedIndex:
    def __init__(self):
        self.index: dict[str, list[Posting]] = defaultdict(list)
        self.docs: dict[int, str] = {}             # doc_id → original text
        self.doc_lengths: dict[int, int] = {}      # doc_id → number of tokens
        self.next_id: int = 0
        self.total_docs: int = 0
        self.avg_doc_length: float = 0.0

    def add_document(self, text: str) -> int:
        doc_id = self.next_id
        self.next_id += 1
        self.docs[doc_id] = text

        tokens = analyze(text)
        self.doc_lengths[doc_id] = len(tokens)

        # Build (term → positions) mapping
        term_positions: dict[str, list[int]] = defaultdict(list)
        for pos, token in enumerate(tokens):
            term_positions[token].append(pos)

        for term, positions in term_positions.items():
            self.index[term].append(Posting(
                doc_id=doc_id,
                term_freq=len(positions),
                positions=positions,
            ))

        self.total_docs += 1
        self.avg_doc_length = sum(self.doc_lengths.values()) / self.total_docs
        return doc_id

    def _idf(self, term: str) -> float:
        df = len(self.index.get(term, []))
        if df == 0:
            return 0.0
        return math.log((self.total_docs - df + 0.5) / (df + 0.5) + 1)

    def bm25_score(self, term: str, posting: Posting, k1: float = 1.2, b: float = 0.75) -> float:
        idf = self._idf(term)
        tf = posting.term_freq
        dl = self.doc_lengths[posting.doc_id]
        norm_tf = (tf * (k1 + 1)) / (tf + k1 * (1 - b + b * dl / self.avg_doc_length))
        return idf * norm_tf

    def search(self, query: str, top_k: int = 5) -> list[tuple[int, float, str]]:
        """Full-text search with BM25 scoring. Returns [(doc_id, score, text)]."""
        query_terms = analyze(query)
        if not query_terms:
            return []

        scores: dict[int, float] = defaultdict(float)
        for term in query_terms:
            for posting in self.index.get(term, []):
                scores[posting.doc_id] += self.bm25_score(term, posting)

        ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)[:top_k]
        return [(doc_id, score, self.docs[doc_id]) for doc_id, score in ranked]

    def phrase_search(self, phrase: str) -> list[int]:
        """Find documents where the analyzed phrase terms appear in sequence."""
        terms = analyze(phrase)
        if not terms:
            return []

        # Start with documents that contain all terms
        posting_map: dict[int, dict[str, list[int]]] = defaultdict(dict)
        for term in terms:
            for posting in self.index.get(term, []):
                posting_map[posting.doc_id][term] = posting.positions

        results = []
        for doc_id, term_positions in posting_map.items():
            if len(term_positions) < len(terms):
                continue   # Missing some terms

            # Check if terms appear consecutively
            first_term_positions = term_positions[terms[0]]
            for start_pos in first_term_positions:
                found = True
                for i, term in enumerate(terms[1:], 1):
                    if (start_pos + i) not in term_positions.get(term, []):
                        found = False
                        break
                if found:
                    results.append(doc_id)
                    break

        return results


# ─── Demo ─────────────────────────────────────────────────────────────────────

idx = InvertedIndex()

docs = [
    "Elasticsearch is a distributed search and analytics engine built on Apache Lucene",
    "The inverted index is the core data structure used in search engines",
    "BM25 scoring improves on TF-IDF with term frequency saturation and length normalization",
    "Python is a programming language widely used in machine learning and data science",
    "Distributed systems require careful consideration of consistency and availability",
    "Search engines like Elasticsearch and Solr use inverted indexes for fast full-text search",
    "Apache Lucene powers both Elasticsearch and Apache Solr search platforms",
    "Machine learning models can improve search relevance beyond traditional BM25 scoring",
]

for doc in docs:
    idx.add_document(doc)

print("=== Full-text search: 'elasticsearch distributed search' ===")
results = idx.search("elasticsearch distributed search", top_k=3)
for doc_id, score, text in results:
    print(f"  [{score:.3f}] doc_{doc_id}: {text[:70]}...")

print("\n=== Full-text search: 'BM25 scoring' ===")
results = idx.search("BM25 scoring", top_k=3)
for doc_id, score, text in results:
    print(f"  [{score:.3f}] doc_{doc_id}: {text[:70]}...")

print("\n=== Phrase search: 'inverted index' ===")
doc_ids = idx.phrase_search("inverted index")
for doc_id in doc_ids:
    print(f"  doc_{doc_id}: {idx.docs[doc_id][:70]}...")

print("\n=== Term statistics ===")
for term in ["search", "distribut", "lucen", "bm25"]:
    postings = idx.index.get(term, [])
    print(f"  '{term}': df={len(postings)}, idf={idx._idf(term):.3f}")
```

---

## Exercise 2: Elasticsearch Index Design and CRUD

```python
"""
Goal: Design an Elasticsearch index for a product catalog.
      Implement: indexing, full-text search, filtered search,
      aggregations, and wildcard/autocomplete.
"""

from elasticsearch import Elasticsearch
import json

es = Elasticsearch("http://localhost:9200")
INDEX = "products"


# ─── Index mapping ────────────────────────────────────────────────────────────

def create_index():
    """
    mapping design decisions:
    - title: text (analyzed for full-text) + keyword (for sorting/exact match)
    - description: text only (too long for keyword)
    - category: keyword (exact match, aggregations)
    - price: float (range queries)
    - in_stock: boolean (filter)
    - brand: text + keyword (search + facets)
    - title_suggest: completion (autocomplete)
    """
    mapping = {
        "settings": {
            "number_of_shards": 3,
            "number_of_replicas": 1,
            "analysis": {
                "analyzer": {
                    "product_analyzer": {
                        "type": "custom",
                        "tokenizer": "standard",
                        "filter": ["lowercase", "stop", "porter_stem"],
                    }
                }
            },
        },
        "mappings": {
            "properties": {
                "title": {
                    "type": "text",
                    "analyzer": "product_analyzer",
                    "fields": {
                        "keyword": {"type": "keyword"},
                        "suggest": {"type": "completion"},  # autocomplete
                    },
                },
                "description": {"type": "text", "analyzer": "product_analyzer"},
                "category": {"type": "keyword"},
                "brand": {
                    "type": "text",
                    "analyzer": "product_analyzer",
                    "fields": {"keyword": {"type": "keyword"}},
                },
                "price": {"type": "float"},
                "rating": {"type": "float"},
                "review_count": {"type": "integer"},
                "in_stock": {"type": "boolean"},
                "tags": {"type": "keyword"},
                "created_at": {"type": "date"},
            }
        },
    }

    if es.indices.exists(index=INDEX):
        es.indices.delete(index=INDEX)
    es.indices.create(index=INDEX, body=mapping)
    print(f"Index '{INDEX}' created")


# ─── Index documents ──────────────────────────────────────────────────────────

def index_products():
    products = [
        {"id": "1", "title": "iPhone 15 Pro Max", "description": "Apple flagship smartphone with titanium design and Action button", "category": "smartphones", "brand": "Apple", "price": 1199.99, "rating": 4.8, "review_count": 15420, "in_stock": True, "tags": ["5g", "ios", "titanium"]},
        {"id": "2", "title": "Samsung Galaxy S24 Ultra", "description": "Android flagship with S Pen and 200MP camera system", "category": "smartphones", "brand": "Samsung", "price": 1299.99, "rating": 4.7, "review_count": 8930, "in_stock": True, "tags": ["5g", "android", "spen"]},
        {"id": "3", "title": "Sony WH-1000XM5 Headphones", "description": "Industry-leading noise canceling wireless headphones", "category": "audio", "brand": "Sony", "price": 349.99, "rating": 4.9, "review_count": 25000, "in_stock": True, "tags": ["wireless", "noise-canceling"]},
        {"id": "4", "title": "MacBook Pro 14-inch M3", "description": "Professional laptop with Apple M3 chip and Liquid Retina XDR display", "category": "laptops", "brand": "Apple", "price": 1999.99, "rating": 4.9, "review_count": 7200, "in_stock": False, "tags": ["m3", "macos", "retina"]},
        {"id": "5", "title": "Google Pixel 8", "description": "Pure Android experience with Google AI features and seven years of updates", "category": "smartphones", "brand": "Google", "price": 699.99, "rating": 4.6, "review_count": 4100, "in_stock": True, "tags": ["android", "ai", "5g"]},
        {"id": "6", "title": "Dell XPS 15", "description": "Premium Windows laptop with OLED display and discrete GPU", "category": "laptops", "brand": "Dell", "price": 1599.99, "rating": 4.5, "review_count": 3300, "in_stock": True, "tags": ["windows", "oled", "gpu"]},
    ]

    for product in products:
        pid = product.pop("id")
        es.index(index=INDEX, id=pid, document=product)

    es.indices.refresh(index=INDEX)
    print(f"Indexed {len(products)} products")


# ─── Queries ──────────────────────────────────────────────────────────────────

def search_full_text(query: str):
    """Multi-field full-text search with title boosted."""
    response = es.search(
        index=INDEX,
        body={
            "query": {
                "multi_match": {
                    "query": query,
                    "fields": ["title^3", "description^1", "brand^2"],
                    "type": "best_fields",
                    "operator": "or",
                }
            },
            "_source": ["title", "brand", "price", "rating"],
            "size": 5,
        },
    )
    print(f"\n=== Full-text: '{query}' ===")
    for hit in response["hits"]["hits"]:
        print(f"  [{hit['_score']:.2f}] {hit['_source']['title']} "
              f"— ${hit['_source']['price']}")


def search_with_filters(query: str, category: str = None,
                        min_price: float = None, max_price: float = None,
                        in_stock: bool = None):
    """Full-text search + filters (filters don't affect scoring)."""
    filters = []
    if category:
        filters.append({"term": {"category": category}})
    if min_price is not None or max_price is not None:
        price_range = {}
        if min_price is not None:
            price_range["gte"] = min_price
        if max_price is not None:
            price_range["lte"] = max_price
        filters.append({"range": {"price": price_range}})
    if in_stock is not None:
        filters.append({"term": {"in_stock": in_stock}})

    body = {
        "query": {
            "bool": {
                "must": [
                    {"multi_match": {
                        "query": query,
                        "fields": ["title^3", "description"],
                        "operator": "or",
                    }}
                ] if query else [{"match_all": {}}],
                "filter": filters,
            }
        },
        "_source": ["title", "brand", "price", "in_stock"],
        "size": 5,
    }

    response = es.search(index=INDEX, body=body)
    print(f"\n=== Filtered search: '{query}' | category={category} | "
          f"price={min_price}–{max_price} | in_stock={in_stock} ===")
    for hit in response["hits"]["hits"]:
        src = hit["_source"]
        print(f"  [{hit['_score']:.2f}] {src['title']} — ${src['price']}"
              f" {'✓' if src['in_stock'] else '✗'}")


def search_with_aggregations(query: str):
    """Search + facets (category counts, price ranges)."""
    response = es.search(
        index=INDEX,
        body={
            "query": {"multi_match": {"query": query, "fields": ["title", "description"]}},
            "aggs": {
                "categories": {
                    "terms": {"field": "category", "size": 10}
                },
                "brands": {
                    "terms": {"field": "brand.keyword", "size": 10}
                },
                "price_ranges": {
                    "range": {
                        "field": "price",
                        "ranges": [
                            {"to": 500},
                            {"from": 500, "to": 1000},
                            {"from": 1000},
                        ],
                    }
                },
                "avg_rating": {"avg": {"field": "rating"}},
            },
            "size": 3,
        },
    )

    print(f"\n=== Aggregations for '{query}' ===")
    print("Categories:", {b["key"]: b["doc_count"]
                          for b in response["aggregations"]["categories"]["buckets"]})
    print("Brands:", {b["key"]: b["doc_count"]
                      for b in response["aggregations"]["brands"]["buckets"]})
    print(f"Avg rating: {response['aggregations']['avg_rating']['value']:.2f}")


def autocomplete(prefix: str):
    """Completion suggester for autocomplete."""
    response = es.search(
        index=INDEX,
        body={
            "suggest": {
                "product_suggest": {
                    "prefix": prefix,
                    "completion": {
                        "field": "title.suggest",
                        "size": 5,
                        "skip_duplicates": True,
                    },
                }
            },
            "size": 0,
        },
    )
    suggestions = response["suggest"]["product_suggest"][0]["options"]
    print(f"\n=== Autocomplete: '{prefix}' ===")
    for s in suggestions:
        print(f"  {s['text']}")


# ─── Run ──────────────────────────────────────────────────────────────────────

create_index()
index_products()

search_full_text("Apple laptop")
search_full_text("noise canceling headphones")
search_with_filters("smartphone", category="smartphones", min_price=700, in_stock=True)
search_with_aggregations("laptop")
autocomplete("Sam")
```

---

## Exercise 3: TF-IDF and BM25 Scorer Comparison

```python
"""
Goal: Implement both TF-IDF and BM25 scoring from scratch
      and compare their behavior on a collection of documents.
      Visualize why BM25's TF saturation and length normalization matter.
"""

import math
from collections import Counter


def compute_tf_idf(query_terms: list[str], doc_term_counts: Counter,
                   doc_length: int, df_map: dict[str, int], N: int) -> float:
    score = 0.0
    for term in set(query_terms):
        tf = doc_term_counts[term] / doc_length if doc_length > 0 else 0
        df = df_map.get(term, 0)
        idf = math.log(N / (1 + df)) if df > 0 else 0
        score += tf * idf
    return score


def compute_bm25(query_terms: list[str], doc_term_counts: Counter,
                 doc_length: int, df_map: dict[str, int], N: int,
                 avg_dl: float, k1: float = 1.2, b: float = 0.75) -> float:
    score = 0.0
    for term in set(query_terms):
        tf = doc_term_counts[term]
        df = df_map.get(term, 0)
        if df == 0:
            continue
        idf = math.log((N - df + 0.5) / (df + 0.5) + 1)
        norm_tf = tf * (k1 + 1) / (tf + k1 * (1 - b + b * doc_length / avg_dl))
        score += idf * norm_tf
    return score


# ─── Documents to score ───────────────────────────────────────────────────────

documents = {
    "doc_a": "cat cat cat cat cat",                                    # 5× cat, short
    "doc_b": "cat dog bird fish turtle hamster parrot lizard frog snake cat",  # 2× cat, long
    "doc_c": "the quick brown fox jumped over the lazy dog near the cat",      # 1× cat, medium
    "doc_d": "cat",                                                    # 1× cat, very short
}

query = "cat"
query_terms = [query]

term_counts = {name: Counter(text.split()) for name, text in documents.items()}
lengths = {name: len(text.split()) for name, text in documents.items()}
avg_dl = sum(lengths.values()) / len(lengths)
N = len(documents)

# Document frequency
df_map: dict[str, int] = {}
for term in set(query_terms):
    df_map[term] = sum(1 for tc in term_counts.values() if tc.get(term, 0) > 0)

print("=== Scoring 'cat' across 4 documents ===\n")
print(f"{'Doc':<10} {'Length':>8} {'TF':>6} {'TF-IDF':>10} {'BM25':>10}")
print("-" * 50)

for name, text in documents.items():
    tc = term_counts[name]
    dl = lengths[name]
    tf = tc.get("cat", 0)
    tfidf = compute_tf_idf(query_terms, tc, dl, df_map, N)
    bm25  = compute_bm25(query_terms, tc, dl, df_map, N, avg_dl)
    print(f"{name:<10} {dl:>8} {tf:>6} {tfidf:>10.4f} {bm25:>10.4f}")

print("""
Key observations:
- TF-IDF: doc_a (5× cat, short) scores much higher than doc_d (1× cat, shorter)
  — TF grows unboundedly; no length normalization penalizes long docs
- BM25: doc_a still high but capped (TF saturation); doc_d scores similarly to doc_c
  — shorter documents with the same term frequency rank higher (length norm)
- BM25 is generally considered more aligned with human relevance judgments
""")

# ─── TF saturation visualization ─────────────────────────────────────────────
print("\n=== TF saturation: BM25 vs TF-IDF as term frequency increases ===")
print(f"{'TF':>5} {'TF-IDF (raw TF)':>20} {'BM25 norm_tf':>20}")
k1 = 1.2
for tf in [1, 2, 5, 10, 20, 50, 100]:
    tfidf_tf = tf
    bm25_tf = tf * (k1 + 1) / (tf + k1)
    print(f"{tf:>5} {tfidf_tf:>20.2f} {bm25_tf:>20.4f}")
print("(BM25 asymptotes at k1+1 = 2.20; TF-IDF grows without bound)")
```

---

## Exercise 4: Elasticsearch Aggregations for Search Facets

```python
"""
Goal: Build a faceted search interface — filter by category, brand, price range
      and show matching counts for each facet value (like Amazon's sidebar).
"""

from elasticsearch import Elasticsearch

es = Elasticsearch("http://localhost:9200")
INDEX = "products"   # Assumes products from Exercise 2 are indexed


def faceted_search(
    query: str,
    selected_category: str = None,
    selected_brand: str = None,
    price_min: float = None,
    price_max: float = None,
):
    """
    Returns search results with facet counts.
    Active filters are applied to results but NOT to their own aggregation
    (so you still see all categories even when one is selected).
    """
    # Base text query
    text_query = {"multi_match": {"query": query, "fields": ["title^3", "description"]}}

    # Build active filters
    active_filters = []
    if selected_category:
        active_filters.append({"term": {"category": selected_category}})
    if selected_brand:
        active_filters.append({"term": {"brand.keyword": selected_brand}})
    if price_min is not None:
        active_filters.append({"range": {"price": {"gte": price_min}}})
    if price_max is not None:
        active_filters.append({"range": {"price": {"lte": price_max}}})

    # Main query: text search + all active filters
    main_query = {
        "bool": {
            "must": [text_query],
            "filter": active_filters,
        }
    }

    response = es.search(
        index=INDEX,
        body={
            "query": main_query,
            "aggs": {
                # Category facet: apply brand + price filters but NOT category filter
                # (so we show all category options, not just the selected one)
                "category_facet": {
                    "filter": {
                        "bool": {
                            "must": [text_query],
                            "filter": [f for f in active_filters
                                       if "category" not in str(f)],
                        }
                    },
                    "aggs": {
                        "values": {"terms": {"field": "category", "size": 20}}
                    },
                },
                # Brand facet: apply category + price but NOT brand filter
                "brand_facet": {
                    "filter": {
                        "bool": {
                            "must": [text_query],
                            "filter": [f for f in active_filters
                                       if "brand" not in str(f)],
                        }
                    },
                    "aggs": {
                        "values": {"terms": {"field": "brand.keyword", "size": 20}}
                    },
                },
                "price_stats": {"stats": {"field": "price"}},
            },
            "_source": ["title", "brand", "price", "rating", "in_stock"],
            "size": 10,
        },
    )

    hits = response["hits"]["hits"]
    cat_buckets = response["aggregations"]["category_facet"]["values"]["buckets"]
    brand_buckets = response["aggregations"]["brand_facet"]["values"]["buckets"]
    price_stats = response["aggregations"]["price_stats"]

    print(f"\n=== Faceted Search: '{query}' ===")
    print(f"Results ({response['hits']['total']['value']} total):")
    for hit in hits:
        s = hit["_source"]
        print(f"  [{hit['_score']:.2f}] {s['title']} — ${s['price']:.2f} ⭐{s['rating']}")

    print("\nCategories:")
    for b in cat_buckets:
        selected = " ✓" if b["key"] == selected_category else ""
        print(f"  {b['key']}: {b['doc_count']}{selected}")

    print("Brands:")
    for b in brand_buckets:
        selected = " ✓" if b["key"] == selected_brand else ""
        print(f"  {b['key']}: {b['doc_count']}{selected}")

    print(f"Price range: ${price_stats['min']:.0f} – ${price_stats['max']:.0f}")


faceted_search("phone")
faceted_search("phone", selected_category="smartphones", price_max=800)
```
