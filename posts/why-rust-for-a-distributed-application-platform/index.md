---
title: Why Rust for a Distributed Application Platform
description: Garbage collection pauses, runtime overhead, and single-threaded bottlenecks. Here's why we chose Rust to build a platform where latency is bounded by physics.
date: 2026-05-02
author: Jaxon Repp
category: Engineering
readingTime: 7 min read
linkedin_post: |
  I spent 6.5 years selling a Node.js application platform to Fortune 500 companies.

  I watched garbage collection pauses ripple through real-time dashboards at 10,000 concurrent connections. I explained why cold starts took 45 seconds instead of 5. I sold deals the platform couldn't deliver — and had to walk them back.

  The architecture was right. Unified runtime. Database, cache, messaging, application logic in one process. That model eliminates network hops and simplifies everything.

  But Node.js was the wrong foundation for it.

  JavaScript is single-threaded. V8's garbage collector pauses your process to reclaim memory. The runtime carries overhead that becomes the ceiling when you need sub-millisecond p99 latency or local ML inference.

  So I rebuilt it in Rust.

  94,000 lines across 28 crates. HTTP server, RocksDB storage, real-time streaming, vector search, and ML inference — all in one immutable, signed binary. No garbage collector. No runtime. No npm install.

  The result:
  → 5x throughput
  → Half the latency
  → A quarter of the resources
  → No memory leaks. No GC pauses. Flat resource utilization under load.
  → Zero-copy data path from socket to storage
  → Immutable signed deployments — the binary you test is the binary you ship

  The full story of why we launched with Rust — and what we learned building a platform in it — is on the blog.

  #rust #distributedsystems #performance #startup
linkedin_comment: |
  Full article: https://yetirocks.com/www/blog/why-rust-for-a-distributed-application-platform

  Covers GC overhead, the dylib plugin boundary, schema-first design, and what 94K lines of Rust taught us about building platforms.
---

Every application platform makes the same architectural bet: unify the stack. Put your database, cache, application logic, and messaging into a single process. Eliminate network hops. Reduce operational complexity. Ship faster.

It's the right bet. I spent six and a half years proving it works — selling that exact architecture to Fortune 500 companies, closing $20M+ in enterprise deals, and achieving an 80% competitive win rate against AWS, Azure, and custom solutions.

But I kept running into a ceiling. Not an architectural ceiling — a runtime ceiling.

## The Runtime Is the Bottleneck

Most unified platforms are built on Node.js. There are good reasons for that. JavaScript is everywhere. The ecosystem is massive. Developers ship fast. For 90% of workloads, it's more than enough.

The other 10% is where things break down.

Node.js is single-threaded. V8's garbage collector periodically pauses execution to reclaim memory. At low concurrency this is invisible. At 10,000 concurrent WebSocket connections streaming real-time updates, a 50ms GC pause cascades into missed frames, stale data, and degraded user experience.

The runtime itself carries overhead. Cold starting a Node.js application means loading the V8 engine, parsing JavaScript, resolving npm dependencies. On a cloud instance, that's 30-45 seconds before your first request is served. On an edge device, it may not be viable at all.

And there's a hard constraint that no amount of optimization can fix: you cannot safely embed a machine learning inference engine inside a garbage-collected runtime. The memory management patterns are incompatible. Every major Node.js platform that offers "AI capabilities" is making HTTP calls to an external service. That's a network hop — the exact thing the unified architecture was supposed to eliminate.

## Why We Launched with Rust

Rust solves these problems at the language level, not the framework level. Here's what that means in practice:

**No memory leaks, no garbage collector.** Memory is managed through ownership and borrowing, enforced at compile time. There are no GC pauses. No slow memory growth over days that forces a restart. Resource utilization is stable and predictable — what you see in staging is what you get in production at 3 AM.

**Stable resource utilization.** A Node.js process under load is a moving target — memory climbs, GC kicks in, latency spikes, memory drops, repeat. A Rust process under the same load draws a flat line. Operations teams notice this immediately. Capacity planning goes from guesswork to arithmetic.

**Immutable signed deployments.** Yeti ships as a single static binary. No `node_modules`, no dependency resolution at deploy time, no left-pad incidents. The binary you test is the binary you deploy. It can be signed, checksummed, and verified end-to-end.

**Zero-copy architecture.** Rust's ownership model lets us pass data from the TCP socket through the HTTP layer to the storage engine without copying it into intermediate buffers. In a garbage-collected runtime, every layer allocates, copies, and eventually needs cleanup. In Rust, the data moves.

**Smaller footprint.** No V8 engine, no runtime, no interpreted layer. Yeti's binary is a fraction of the memory footprint of an equivalent Node.js deployment. This matters on edge devices, in containers with tight resource limits, and when you're paying per-GB for cloud instances.

**Embeddable ML inference.** Because Rust manages memory explicitly, you can run a Candle-based ML model in the same process as your HTTP server and storage engine. Embeddings are generated locally, in-process, with no network call. Vector search queries hit an HNSW index backed by the same RocksDB instance that stores your application data.

The numbers tell the story: 5x the throughput, half the latency, a quarter of the resources — compared to the same architecture built on Node.js. These aren't theoretical advantages. They're the reason Yeti serves 100,000+ requests per second on a single node with sub-millisecond p50 latency.

## What 94,000 Lines of Rust Taught Us

Yeti is a single binary compiled from 28 crates. HTTP server (Axum), storage engine (RocksDB), real-time messaging (SSE, WebSocket, MQTT), authentication (Argon2id, JWT, OAuth), vector search (HNSW with fastembed ONNX), telemetry, audit logging, and a plugin compiler — all in one process.

Building this taught us a few things worth sharing:

**The dylib boundary is real.** Application plugins compile to `.dylib` files loaded at runtime. This creates hard constraints. You can't `tokio::spawn` from a dynamic library — it crashes with "cannot catch foreign exceptions." You can't use `reqwest::blocking::Client`. `OnceLock` statics are duplicated across the host/dylib boundary. Every one of these is a sharp edge we discovered in production and encoded into our SDK so developers never hit them.

**Schema-first beats code-first for platforms.** Yeti applications are defined by GraphQL schemas with custom directives. `@table` creates a RocksDB-backed table. `@export` generates REST, SSE, MQTT, and MCP endpoints. `@indexed` builds secondary indexes. `Vector` is a first-class type — declare a `Vector` field with `@indexed(source: "...")` and Yeti auto-embeds the source fields using a local ONNX model and builds an HNSW index. No external embedding API. The schema is the single source of truth — the platform generates everything else.

```graphql
type Product @table @export {
  id: ID! @primaryKey
  name: String! @indexed
  description: String
  price: Float @default(value: 0)
  createdAt: DateTime @createdTime
  embedding: Vector @indexed(source: "name,description")
}
```

That schema gives you a full CRUD API, vector search with auto-generated embeddings, timestamps, and default values — with zero application code.

**Compile times are the tax you pay.** First plugin compilation takes about 2 minutes. Cached restarts take 10 seconds. A full `cargo build` of the platform takes longer than we'd like. We mitigate this with a shared compilation cache across all plugins — they share a `target/` directory so dependencies compile once.

## The Performance Ceiling Is Gone

The unified architecture was always the right idea. Database, cache, application, and messaging in one process. No network hops between your data and your application logic.

The question was always: what do you build it on?

Node.js gave that architecture to the world. It proved the model works. But it also imposed a ceiling — GC pauses, memory leaks, single-threaded execution, no local inference, unpredictable resource utilization, and runtime cold starts.

Rust removes that ceiling. Not by being clever, but by being correct. The compiler enforces memory safety. The type system catches errors before deployment. The binary is immutable, signed, and runs without a runtime. You get 5x the throughput at a quarter of the infrastructure cost.

If you're building something where "fast enough" isn't fast enough — real-time streaming, AI inference pipelines, high-frequency data collection, edge computing — the runtime matters. And Rust is the only language that lets you build a unified platform where latency is bounded by physics, not by the garbage collector.

We're launching the self-hosted version of Yeti on May 1. If you want to see what a Rust-native application platform looks like in practice, check out [yetirocks.com](https://yetirocks.com).
