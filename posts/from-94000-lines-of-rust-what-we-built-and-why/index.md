---
title: "From 94,000 Lines of Rust: What We Built and Why"
description: A technical walkthrough of the 28 crates inside Yeti — what each one does, why it exists, and what we learned building them.
date: 2026-05-20
author: Jaxon Repp
category: Engineering
readingTime: 8 min read
linkedin_post: |
  94,000 lines of Rust. 28 crates. One binary.

  People ask what's actually inside Yeti. Here's the honest inventory:

  → yeti-http: Axum-based server, TLS termination, static file serving
  → yeti-store: RocksDB abstraction with column families per app
  → yeti-schema: GraphQL SDL parser with custom directives
  → yeti-table: Query execution, mutations, CRDT merge, streaming, vector search
  → yeti-auth: Argon2id, JWT, OAuth, RBAC with field-level masking
  → yeti-mqtt: Native MQTT 5.0 broker with WebSocket proxy
  → yeti-mcp: Model Context Protocol server generated from schema
  → yeti-ai: Candle-based ML inference, fastembed ONNX embeddings
  → yeti-compiler: Dynamic library compilation for application plugins
  → yeti-kafka: Kafka consumer bridge to tables
  → yeti-telemetry: OpenTelemetry traces, Prometheus metrics
  → yeti-audit: Immutable audit log with before/after state
  ...and 16 more.

  Every one of these would be a separate service in a traditional architecture. In Yeti, they're crates in a single workspace that compile into one static binary.

  The full walkthrough — what each crate does, the hard problems we solved, and the architectural decisions we'd make differently — is on the blog.

  #rust #architecture #distributedsystems #startup
linkedin_comment: |
  Full article: https://yetirocks.com/www/blog/from-94000-lines-of-rust-what-we-built-and-why

  If you're building anything at this scale in Rust, the dylib boundary section alone will save you weeks.
---

People ask what's inside Yeti. The short answer is "everything." The longer answer is 28 crates in a Cargo workspace that compile into a single static binary. Here's what each layer does and why it exists.

## The Workspace

Yeti is a Cargo workspace — a single `Cargo.toml` at the root that references 28 member crates. Each crate has a focused responsibility, explicit dependencies, and defined public API boundaries. The workspace compiles as a unit, which means the Rust compiler can optimize across crate boundaries with LTO (link-time optimization).

The crates fall into five layers: core infrastructure, data, protocol, extensions, and tooling.

## Core Infrastructure

**yeti-core** is the foundation. Configuration loading, lifecycle management, extension registration, and the shared type system that every other crate depends on. This is where the startup sequence lives — loading `yeti-config.yaml`, initializing extensions in dependency order, and coordinating graceful shutdown.

**yeti-types** defines the shared type system: scalar types, field definitions, table definitions, directive metadata. It's a leaf crate with no dependencies on the rest of the workspace, which keeps compile times fast for downstream crates.

**yeti-schema** parses GraphQL SDL files into `TableDefinition` structs. It handles every custom directive — `@table`, `@export`, `@primaryKey`, `@indexed`, `@relationship`, `@audit`, `@distribute`, `@vector`, and more. The parser is strict: invalid directives are compile errors, not runtime surprises.

**yeti-runtime** manages the Tokio async runtime, task spawning, signal handling, and process-level concerns. It's the thin layer between the OS and the application.

## Data Layer

**yeti-store** is the RocksDB abstraction. Each application gets its own column family, providing database-level isolation without separate RocksDB instances. The crate handles compaction tuning, TTL-based expiration, write-ahead log management, and the low-level get/put/delete/scan operations that everything else builds on.

**yeti-table** is where queries execute. FIQL parsing, secondary index lookups, full-text search, vector search (HNSW), CRDT merge operations for distributed writes, streaming fan-out to SSE/WebSocket/MQTT subscribers, and mutation validation. This is the largest and most complex crate in the workspace.

**yeti-data-loader** handles seed data and migration. Applications can define JSON seed files that load on first boot — the demo apps use this to populate sample data automatically.

**yeti-sync** manages cross-node replication for distributed deployments. Eventual consistency, conflict resolution via CRDTs, and the protocol for syncing table state between cluster members.

## Protocol Layer

**yeti-http** is the Axum-based HTTP server. TLS termination, route generation from `@export` directives, static file serving for application frontends, and the middleware pipeline (auth, audit, telemetry, CORS). Every REST, GraphQL, SSE, and MCP endpoint flows through this crate.

**yeti-graphql** generates a GraphQL schema from the table definitions and handles query/mutation execution. The schema is derived from the same `TableDefinition` structs that drive REST endpoints — one source of truth, two query interfaces.

**yeti-mqtt** is a native MQTT 5.0 broker. Not a client library — a full broker running inside the Yeti process. Supports MQTTS on port 8883, WebSocket proxy for browser clients, topic-based routing, QoS 0/1/2, and session persistence. Tables with `@export(mqtt: true)` automatically publish mutations to their topic.

**yeti-mcp** generates Model Context Protocol servers from `@export` types. Tool discovery, resource listing, prompt templates, session management, and JSON-RPC 2.0 transport — all derived from the GraphQL schema.

**yeti-grpc** provides gRPC streaming endpoints for service-to-service communication. Protocol Buffer schemas are generated from the same table definitions.

**yeti-kafka** is a Kafka consumer bridge. Configure a broker and topic in `yeti-config.yaml`, and messages stream into a Yeti table automatically. Supports SASL authentication (PLAIN, SCRAM-SHA-256, SCRAM-SHA-512) and TLS.

## Extensions

**yeti-auth** handles authentication and authorization. Basic auth with Argon2id password hashing, JWT generation and validation, OAuth provider integration (Google, GitHub), and role-based access control with field-level masking. The `viewer` role can't see `salary` — the field is stripped before serialization, not after.

**yeti-ai** provides ML inference via Candle and embedding generation via fastembed ONNX. This is what powers the `Vector` type — when a record is written with a `Vector` field, this crate generates the embedding in-process without an external API call.

**yeti-audit** is an immutable audit log. Every mutation on `@audit`-annotated tables is captured with the operation type, user identity, timestamp, and optionally the before/after record state. Audit records have configurable retention (default 90 days) and are stored in their own column family.

**yeti-alerts** monitors table conditions and fires notifications. Configurable thresholds, rate limiting, and multiple delivery targets.

**yeti-telemetry** provides OpenTelemetry-compatible tracing and Prometheus-format metrics. Every request gets a trace ID. Latency histograms, throughput counters, and error rates are exported for monitoring.

## Tooling

**yeti-compiler** compiles application plugins from Rust source code into `.dylib` files loaded at runtime. This is where the sharp edges live — the dylib boundary creates constraints that are invisible until you hit them. `tokio::spawn` from a dynamic library crashes. `OnceLock` statics are duplicated. `reqwest::blocking::Client` deadlocks. Every one of these is encoded into the SDK so developers never encounter them.

**yeti-sdk** is the developer-facing API for writing application resources (custom endpoints, background tasks, lifecycle hooks). It's intentionally minimal — most applications don't need custom code because the schema generates everything.

**yeti-bin** is the CLI entry point. Command-line argument parsing, subcommands (start, init, version), and the main function that wires everything together.

**yeti-installer** handles first-run setup: creating the data directory, generating TLS certificates, downloading the default embedding model, and writing the initial configuration.

## What We Learned

**Crate boundaries are API contracts.** Once another crate depends on your public types, changing them requires coordinating across the workspace. We learned to be conservative with `pub` visibility early, because making something public is easy — making it private again is a breaking change.

**The dylib boundary is the hardest problem.** Everything else is conventional Rust engineering. The dynamic library boundary is where Rust's safety guarantees stop helping you. We spent more time debugging dylib issues than any other single category of bug.

**28 crates is not too many.** Each crate compiles independently, which means incremental builds only recompile what changed. The alternative — a monolithic crate — would mean every change triggers a full recompile. At 94,000 lines, that's the difference between a 10-second rebuild and a 10-minute rebuild.

The full crate dependency graph, build times, and architectural decisions are documented in the [Yeti documentation](https://yetirocks.com/developers/docs). If you're building anything at this scale in Rust, we hope this saves you some of the time it cost us to learn it.
