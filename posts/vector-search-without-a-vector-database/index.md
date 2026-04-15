---
title: Vector Search Without a Vector Database
description: Declare a Vector field. Yeti auto-embeds, indexes, and searches — no Pinecone, no external API, no infrastructure.
date: 2026-06-05
author: Jaxon Repp
category: Engineering
readingTime: 7 min read
linkedin_post: |
  "You need a vector database for semantic search."

  No. You don't.

  You need:
  1. An embedding model
  2. An HNSW index
  3. A way to keep vectors and data in sync

  That's it. The "vector database" is just those three things packaged as a separate service — with its own deployment, its own scaling, its own bill, and a network hop between your data and your embeddings.

  In Yeti, you declare a Vector field:

  embedding: Vector @indexed(source: "content")

  That's the whole integration. The platform auto-embeds on write using a local ONNX model, builds an HNSW index in the same RocksDB instance as your data, and serves nearest-neighbor queries with sub-millisecond latency. No Pinecone. No external API. No infrastructure.

  "Storage engine tradeoffs" finds the database article. "Baking temperature control" finds the French pastry article. Pure semantic similarity, zero keyword matching.

  Full deep dive on the blog.

  #vectorsearch #ai #rust #distributedsystems
linkedin_comment: |
  Full article: https://yetirocks.com/www/blog/vector-search-without-a-vector-database

  Includes the schema, curl examples, and a walkthrough of how 100 articles across 10 categories get semantically indexed with one field declaration.
---

Every tutorial about semantic search starts the same way: choose an embedding provider, choose a vector database, write the glue code. Three services, three bills, three things that can break independently. And a network hop between every query and every result.

We asked: why can't the application platform just do this?

## The Vector Database Tax

The standard architecture for semantic search looks like this:

1. Your application writes a record to its primary database
2. A background job sends the text to an embedding API (OpenAI, Cohere, etc.)
3. The embedding comes back over the network
4. Another write sends the vector to a separate vector database (Pinecone, Weaviate, etc.)
5. At query time, you embed the search text, query the vector database, then join results with your primary database

That's five steps, two network round trips, two databases to keep in sync, and an external API dependency for every write. If the embedding API is down, writes fail. If the vector database drifts out of sync with your primary store, search returns stale results.

For prototypes, this is fine. For production systems that need to be fast and reliable, it's unnecessary complexity.

## One Field, Zero Infrastructure

In Yeti, vector search is a schema declaration:

```graphql
type Article @table @export {
    id: ID! @primaryKey
    title: String!
    author: String!
    category: String!
    content: String!
    embedding: Vector @indexed(source: "content")
}
```

The `Vector` type and `@indexed(source: "content")` directive do everything:

**On write:** When you POST an article, the `yeti-vectors` extension intercepts the mutation, generates a 384-dimensional embedding from the `content` field using a local ONNX model (`BAAI/bge-small-en-v1.5`), and stores the vector alongside the record in the same RocksDB instance. One write, one storage engine, no network call.

**On index:** The embedding is inserted into a native Rust HNSW index with cosine similarity. The index lives in process memory, backed by the same storage engine as your data. No separate service.

**On query:** Search by meaning with a single HTTP request:

```bash
# Semantic search via URL
curl "https://localhost:9996/demo-vector/Article?search=storage+engine+tradeoffs&vector=true&limit=5"

# Semantic search via JSON query
curl -X POST "https://localhost:9996/demo-vector/Article?limit=5" \
  -H "Content-Type: application/json" \
  -d '{"conditions":[{"field":"embedding","op":"vector","value":"storage engine tradeoffs"}]}'
```

The query text is embedded in-process using the same model, searched against the HNSW index, and results are returned with similarity scores. Sub-millisecond latency on the vector lookup. No network hop to an external service.

## Semantic Search in Practice

The [demo-vector](https://yetirocks.com/developers/demos) application ships 100 articles spanning machine learning, French pastry, volcanic geology, jazz theory, coral reef biology, Renaissance architecture, and more. The search results demonstrate pure semantic similarity — no keyword matching:

- "storage engine tradeoffs" → finds the database internals article
- "baking temperature control" → finds the French pastry article
- "underwater ecosystems" → finds the coral reef biology article
- "improvisation in music" → finds the jazz theory article

The queries work because the embeddings capture meaning, not keywords. And because the embedding model runs locally, every query is fully offline — no API keys, no external calls, no internet required.

## Real-Time Vector Updates via SSE

Because `@export` generates SSE endpoints automatically, you can subscribe to vector-enriched records in real time:

```bash
curl -N "https://localhost:9996/demo-vector/Article/?$format=sse"
```

POST a new article, and the SSE stream delivers the complete record — including the auto-generated embedding — to every subscriber. This is useful for building live search interfaces that update as new content arrives.

## The Architecture Advantage

Traditional vector search architectures separate the embedding, indexing, and storage concerns into different services. This makes each piece independently scalable, but it also means:

- Vectors and data can drift out of sync
- Every write requires coordinating two databases
- Every query requires a network round trip to the vector store
- Embedding failures create orphaned records

Yeti's approach embeds all three concerns into the same process. The embedding model runs in-process via ONNX. The HNSW index is backed by the same RocksDB instance as the record data. Writes are atomic — the vector and the record are stored together or not at all.

The tradeoff is that you're running the embedding model on the same hardware as your application. For the `BAAI/bge-small-en-v1.5` model (384 dimensions, ~90MB), this is lightweight enough to run on any modern server or even a Raspberry Pi. For larger models, Yeti supports configuring the model per field.

## When You Don't Need a Vector Database

You don't need a vector database when:

- Your dataset fits on a single node (millions of vectors, not billions)
- You need vectors and data to be atomically consistent
- You want zero external API dependencies
- You're running on edge devices or air-gapped environments
- You want sub-millisecond query latency without a network hop

You might still want a dedicated vector database when you need to index billions of vectors across a distributed cluster with specialized quantization. But for most applications — semantic search, recommendation engines, RAG pipelines, content deduplication — the vector database is overhead you don't need.

One field. Zero infrastructure. Search by meaning.
