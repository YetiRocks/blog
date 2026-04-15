---
title: Why We Built Yeti
description: One binary, zero network hops, 100K req/s. The story behind a Rust application platform that embeds everything.
date: 2026-05-01
author: Jaxon Repp
category: Company
readingTime: 4 min read
linkedin_post: |
  Every application platform makes the same promise: fast, easy, scalable.

  Then you look under the hood. A Node.js process. A separate database. A message broker. A cache layer. Six network hops between your user and their data.

  For most apps, this is fine.

  For real-time dashboards at 10K concurrent connections, AI inference pipelines, or high-frequency data collection — the layers ARE the bottleneck.

  So we built Yeti: a single Rust binary that embeds everything. HTTP server, RocksDB storage, real-time streaming, vector search, ML inference, authentication — all in one process. No network hops between your code and your data.

  The result:
  → 100K+ req/s on a single node
  → Sub-millisecond p50 latency
  → Cold start under 10 seconds
  → No Docker, no Kubernetes, no infrastructure to manage

  You define your app in a GraphQL schema. Yeti generates REST, GraphQL, SSE, WebSocket, MQTT, and MCP endpoints automatically. Add @indexed for search, @vector for embeddings, @audit for tracking. Zero application code needed.

  We're launching today. The full story of why we built it — and what it can do — is on the blog.

  #rust #distributedsystems #performance #startup #ai
linkedin_comment: |
  Full article: https://yetirocks.com/www/blog/why-we-built-yeti

  If you're building something where latency matters — real-time streaming, AI pipelines, edge computing — I'd love to show you what Yeti can do.

  → yetirocks.com
---

Every application platform makes the same promise: fast, easy, scalable. But when you look under the hood, you find a Node.js process, a separate database, a message broker, a cache layer, and a deployment pipeline that takes an afternoon to configure. We asked: what if the platform itself was the performance?

## The Problem with Layers

Traditional application platforms are assembled from parts. A web framework talks to a database over the network. A cache sits between them to paper over the latency. A message broker handles real-time events. Each layer adds latency, complexity, and failure modes.

For most applications, this works fine. But for applications that need to be fast — real-time dashboards, AI inference pipelines, high-frequency data collection — the layers become the bottleneck. You're not slow because your code is slow. You're slow because your data crosses six network boundaries before it reaches the user.

## One Binary, Zero Network Hops

Yeti is a single Rust binary that embeds everything: HTTP server, storage engine (RocksDB), real-time streaming (SSE, WebSocket, MQTT), AI inference (Candle), authentication, and a schema-driven API layer. Your application code compiles into the binary as a dynamic library. When a request arrives, it goes from TCP socket to storage and back without ever crossing a network boundary.

The result: 100K+ requests per second on a single node. Sub-millisecond p50 latency. Cold start in under 10 seconds. No Docker, no Kubernetes, no infrastructure to manage.

## Schema-First, Not Code-First

You define your data model in a GraphQL schema file. Yeti reads it and generates REST, GraphQL, SSE, WebSocket, MQTT, and MCP endpoints automatically. Add `@indexed` to a field and it builds a secondary index. Declare a `Vector` field with `@indexed(source: "...")` and Yeti auto-embeds the source fields using a local ONNX model and builds an HNSW index — no external embedding API, no vector database. Add `@audit` and every mutation is tracked.

```graphql
type Product @table @export @audit {
  id: ID! @primaryKey
  name: String! @indexed
  description: String
  price: Float @default(value: 0)
  status: String @default(value: "draft")
  embedding: Vector @indexed(source: "name,description")
}
```

That schema gives you a full CRUD API, vector search with auto-generated embeddings, audit trail, and default values — with zero application code.

## Why Rust?

Not because it's trendy. Because garbage collection pauses are incompatible with real-time streaming. Because memory safety without a runtime means you can embed an ML inference engine in the same process as your HTTP server without worrying about memory corruption. Because a single static binary deploys to any Linux server with `scp`.

Rust lets us make a promise that no Node.js or Go platform can: your application's latency is bounded by physics, not by the runtime.

## What's Next

We're building Yeti in public. The codebase has 29 crates, 83 benchmarks, and a full audit system that verifies code quality on every change. Over the coming weeks, we'll be sharing deep dives into the architecture, performance comparisons, and tutorials for building real applications.

If you're building something that needs to be fast, we'd love to hear from you.
