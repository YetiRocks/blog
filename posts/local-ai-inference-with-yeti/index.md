---
title: Local AI Inference with Yeti
description: "Embeddings, vector search, and ML inference — in-process, no external API, data never leaves your server."
date: 2026-06-09
author: Jaxon Repp
category: Engineering
readingTime: 6 min read
linkedin_post: |
  Every platform that advertises "AI capabilities" is doing the same thing: making an HTTP call to OpenAI.

  Your data leaves your server. Crosses the internet. Gets processed by a third party. Comes back. That's a network hop, a latency spike, a point of failure, and a compliance risk — for every single embedding.

  Yeti runs ML inference in-process.

  Candle-based models and fastembed ONNX run inside the same binary as your HTTP server and storage engine. When you write a record with a Vector field, the embedding is generated locally — in the same process, on the same hardware. No API key. No network call. No external dependency.

  The data never leaves your server. The latency is bounded by your hardware, not by someone else's API.

  For RAG pipelines, semantic search, content dedup, and recommendation engines — this changes the architecture entirely. You don't need an embedding service. You don't need a vector database. You need one schema field.

  Full deep dive on the blog.

  #ai #embeddings #rust #privacy #performance
linkedin_comment: |
  Full article: https://yetirocks.com/www/blog/local-ai-inference-with-yeti

  Covers why GC-based runtimes can't safely embed ML models, how fastembed ONNX works in-process, and what "no external API" means for compliance.
---

Every major application platform that offers "AI capabilities" is making HTTP calls to an external embedding service. Your text goes out over the network, gets processed by a third-party model, and the embedding comes back. For every write. For every query.

This creates four problems that no amount of caching can fix:

1. **Latency**: Every embedding generation adds a network round trip. At 50-100ms per call, this becomes the bottleneck in any write-heavy pipeline.
2. **Availability**: If the embedding API is down, your writes fail. Your application's uptime is now dependent on a service you don't control.
3. **Cost**: Embedding APIs charge per token. At scale, this becomes a significant line item — and it grows linearly with your write volume.
4. **Compliance**: Your data crosses a network boundary and is processed by a third party. For healthcare, finance, government, and any organization with data residency requirements, this may not be acceptable.

## In-Process Inference

Yeti takes a fundamentally different approach. The `yeti-ai` crate embeds ML inference directly into the platform binary using two frameworks:

**Candle** — Hugging Face's Rust-native ML framework. No Python dependency, no C++ build chain, no FFI boundary. Pure Rust tensor operations that compile into the same binary as the HTTP server.

**fastembed ONNX** — optimized ONNX runtime for embedding generation. The default model (`BAAI/bge-small-en-v1.5`) produces 384-dimensional embeddings with a ~90MB model file. It runs on CPU — no GPU required.

When you declare a `Vector` field in your schema:

```graphql
type Document @table @export {
    id: ID! @primaryKey
    title: String!
    content: String!
    embedding: Vector @indexed(source: "title,content")
}
```

Every write triggers this sequence:

1. The record arrives at the HTTP layer
2. `yeti-ai` extracts the `source` fields (`title` and `content`)
3. The text is tokenized and embedded in-process via ONNX
4. The 384-dimensional vector is stored alongside the record in RocksDB
5. The HNSW index is updated atomically

No network call. No external process. No API key. The embedding is generated in the same thread that handles the HTTP request, and the vector is stored in the same transaction as the record.

## Why Garbage-Collected Runtimes Can't Do This

This is often the question: why can't Node.js or Python do the same thing? The answer is memory management.

ML inference engines need precise control over memory allocation and deallocation. Tensor operations allocate large contiguous blocks, use them for computation, and release them immediately. A garbage collector interferes with this pattern — it delays deallocation, fragments the heap, and introduces unpredictable pauses during the exact operations that need deterministic timing.

You can technically run ONNX inside Node.js via native bindings. But you're fighting the runtime: V8's garbage collector will pause to reclaim memory from tensors that the inference engine already released. Under load, these pauses compound with the GC pauses from your application logic, creating latency spikes that are difficult to diagnose and impossible to eliminate.

Rust's ownership model solves this at the language level. Memory is allocated when needed and freed deterministically when the owning scope ends. The tensor operations in Candle and fastembed allocate, compute, and release without any garbage collector interference. Latency is bounded by the computation itself, not by memory management.

## Model Management

Yeti manages embedding models through the `yeti-vectors` extension:

```bash
# List available models
GET /yeti-vectors/models

# Download a new model
POST /yeti-vectors/models
{ "name": "BAAI/bge-small-en-v1.5" }

# Set the default model
PUT /yeti-vectors/models
{ "default": "BAAI/bge-small-en-v1.5" }
```

Models are cached locally in the `models/` directory. The first download fetches from Hugging Face; subsequent starts load from disk. The default model (`BAAI/bge-small-en-v1.5`) is small enough to run on a Raspberry Pi and fast enough to embed documents in single-digit milliseconds.

Per-field model selection is supported via the `model` parameter on `@indexed`:

```graphql
embedding: Vector @indexed(source: "content", model: "BAAI/bge-small-en-v1.5")
```

This lets you use different models for different tables — a lightweight model for short-text classification and a larger model for document-level semantic search.

## What This Means for RAG Pipelines

Retrieval-Augmented Generation (RAG) is the dominant pattern for grounding AI responses in real data. The standard RAG architecture requires three services: an embedding API to vectorize documents, a vector database to index and search them, and an LLM to generate responses.

With Yeti, the first two collapse into a schema declaration. Documents are embedded on write. Vector search is a REST query. The only external call is to the LLM for generation — and even that can be replaced with a local model via Candle for use cases where data can't leave the server.

```bash
# Write a document — embedding auto-generated
curl -X POST https://localhost:9996/myapp/Document \
  -H "Content-Type: application/json" \
  -d '{"id": "doc-1", "title": "Architecture Decision", "content": "We chose RocksDB for..."}'

# Retrieve relevant context — vector search
curl "https://localhost:9996/myapp/Document?search=storage+engine+decision&vector=true&limit=5"

# Pass context to your LLM of choice for generation
```

No embedding service. No vector database. No infrastructure to manage. The write path and the read path both execute in the same process with sub-millisecond latency.

## When Local Inference Is the Right Choice

Local inference is the right choice when:

- **Data residency matters**: Healthcare, finance, government, or any environment where data cannot leave the server
- **Latency matters**: Eliminating the embedding API round trip cuts write latency by 50-100ms per operation
- **Cost matters**: No per-token embedding charges, no matter how many documents you process
- **Availability matters**: No external dependency means no external point of failure
- **Offline environments**: Air-gapped deployments, edge devices, or any environment without reliable internet

For teams that need the absolute best embedding quality on frontier models, external APIs still have their place. But for the vast majority of semantic search, classification, and RAG use cases, local inference with a well-chosen ONNX model delivers equivalent quality with dramatically better latency, cost, and operational simplicity.

The embedding model runs in your process. The vectors live in your storage. The data never leaves your server.
