# Search Engines

## Elasticsearch

The most popular distributed search and analytics engine. Built on Apache Lucene.

### Core Concepts

| Concept | Description | Analogy (RDBMS) |
|---------|-------------|-----------------|
| **Index** | A collection of documents | Database table |
| **Document** | A JSON record | Table row |
| **Field** | A key-value pair in a document | Table column |
| **Shard** | A partition of an index (a Lucene index) | Table partition |
| **Replica** | A copy of a shard for redundancy | Read replica |

### How Search Works: Inverted Index

An inverted index maps terms to the documents that contain them.

**Document 1:** "the quick brown fox"
**Document 2:** "the quick rabbit"

**Inverted index:**

| Term | Documents |
|------|-----------|
| the | 1, 2 |
| quick | 1, 2 |
| brown | 1 |
| fox | 1 |
| rabbit | 2 |

A search for "quick fox" finds the intersection/union of document lists → Document 1.

### Text Analysis Pipeline

```
Raw text → Character filters → Tokenizer → Token filters → Indexed terms
```

| Stage | Purpose | Example |
|-------|---------|---------|
| **Character filter** | Clean up raw text | Strip HTML tags |
| **Tokenizer** | Split text into tokens | "New York" → ["New", "York"] |
| **Token filter** | Transform tokens | Lowercase, stemming ("running" → "run"), synonyms |

### Query Types

| Type | Purpose | Example |
|------|---------|---------|
| **Match** | Full-text search with analysis | "quick fox" matches "the quick brown fox" |
| **Term** | Exact match (no analysis) | Status = "published" |
| **Bool** | Combine queries (must, should, must_not) | Title matches "guide" AND status = "published" |
| **Range** | Numeric/date ranges | price >= 10 AND price <= 50 |
| **Fuzzy** | Typo-tolerant matching | "quik" matches "quick" |
| **Aggregation** | Analytics (group by, avg, histogram) | Average price by category |

### Architecture

```
Client → Coordinating node → Shard 1 (primary + replica)
                            → Shard 2 (primary + replica)
                            → Shard 3 (primary + replica)
```

- **Write path:** Document → routed to primary shard by ID hash → replicated to replica shards
- **Read path:** Query → scatter to all shards → each shard returns local results → coordinating node merges and sorts

### Performance Tuning

| Optimization | How |
|-------------|-----|
| **Shard sizing** | Target 10-50 GB per shard |
| **Index templates** | Consistent mappings and settings across indices |
| **Bulk indexing** | Batch writes for higher throughput |
| **Doc values** | Columnar storage for sorting/aggregations |
| **Index lifecycle** | Roll over time-series indices, delete old ones |
| **Caching** | Query cache, request cache, field data cache |

## Alternatives

| Engine | Differentiator |
|--------|---------------|
| **Apache Solr** | Mature, XML-heavy configuration, strong for traditional search |
| **OpenSearch** | AWS fork of Elasticsearch, fully open-source |
| **Meilisearch** | Typo-tolerant, easy to set up, great for small-medium search |
| **Typesense** | Low-latency, typo-tolerant, simpler operational model |
| **Algolia** | SaaS, instant search, great developer experience |

## Search in System Design

### Pattern: Database + Search Index

```
Primary data → PostgreSQL (source of truth)
                  → Change Data Capture (CDC) → Elasticsearch (search index)
```

**Write flow:** Application writes to the database. CDC or application-level events update the search index.
**Read flow:** Simple queries → database. Full-text search, faceted search, autocomplete → Elasticsearch.

### Common Use Cases

| Use Case | Features Needed |
|----------|----------------|
| **Product search** | Full-text, facets, filters, sorting, autocomplete |
| **Log aggregation** | Time-series indexing, retention policies, dashboards (ELK stack) |
| **Autocomplete** | Prefix matching, edge n-grams, completion suggester |
| **Geosearch** | Geo-point, geo-shape queries, distance sorting |
| **Analytics** | Aggregations, histograms, cardinality |

### Relevance Scoring

Elasticsearch uses **BM25** (an evolution of TF-IDF) to rank results:

- **Term Frequency (TF):** How often the term appears in the document
- **Inverse Document Frequency (IDF):** How rare the term is across all documents
- **Field length:** Shorter fields get higher scores (a match in a title is more relevant than in a body)

Boost scores with custom logic: recency, popularity, user preferences.

## Key Interview Talking Points

- Elasticsearch is a search index, not a primary database — always have a source of truth (PostgreSQL, etc.)
- Use CDC or event-driven updates to keep the search index in sync with the primary database
- Inverted indexes make full-text search fast but updates are expensive (immutable segments, merging)
- Shard count is set at index creation and hard to change — plan ahead
- For autocomplete, use edge n-grams or the completion suggester (not regular search)
- Search is eventually consistent — the index may lag behind the primary database
