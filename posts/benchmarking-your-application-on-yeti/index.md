---
title: "Benchmarking Your Application on Yeti: A Guide"
description: "17 workloads, 6 transports, native Rust load generators. How to measure your Yeti application's performance and know exactly what you're getting."
date: 2026-05-23
author: Jaxon Repp
category: Product
readingTime: 6 min read
linkedin_post: |
  "How fast is it?"

  The honest answer to that question is: run the benchmarks yourself.

  That's why we built app-benchmarks — a self-contained performance lab that ships with Yeti. 17 workloads across 6 transports. Native Rust load generators (no k6, no wrk, no external tools). HdrHistogram latency tracking with p50, p95, p99, and p99.9 breakdowns.

  REST CRUD. GraphQL queries. WebSocket fan-out. SSE streaming. MQTT pub/sub. Vector embedding and search. Every path Yeti serves, measured.

  The best part: every run is stored with full metrics and per-second snapshots. The dashboard highlights personal bests and regressions. You know exactly what changed and when.

  Performance claims without reproducible benchmarks are marketing. Here's how to verify ours.

  #performance #benchmarking #rust #distributedsystems
linkedin_comment: |
  Full guide: https://yetirocks.com/www/blog/benchmarking-your-application-on-yeti

  Covers the benchmark suite, how to run it, how to read the results, and how to build your own workloads. No vendor trust required — measure it yourself.
---

Performance claims are easy to make and hard to verify. "100K+ requests per second" means nothing if you can't reproduce it on your own hardware with your own workload. That's why Yeti ships with a complete benchmarking suite — not as a marketing tool, but as a development tool you use to measure your own application.

## The Benchmark Suite

[app-benchmarks](https://yetirocks.com/developers/demos) is a Yeti application that measures throughput, latency percentiles, and scalability across every transport Yeti supports. It compiles native Rust load generators, orchestrates multi-process test runs, captures per-second time-series snapshots, and presents results in a real-time dashboard with historical comparison.

The suite includes 17 workloads across 6 transports:

**REST** — read, write, update, delete, and mixed CRUD operations. The bread-and-butter workloads that measure the fundamental request/response path.

**GraphQL** — queries and mutations through the GraphQL endpoint. Measures the overhead of schema introspection, query parsing, and resolver execution compared to REST.

**WebSocket** — fan-in (many writers, one reader) and fan-out (one writer, many readers). Measures connection management, message serialization, and pub/sub delivery under load.

**SSE** — server-sent event streaming. Measures subscriber fan-out latency and connection overhead at scale.

**MQTT** — publish/subscribe through the native MQTT broker. Measures topic routing, QoS delivery, and broker throughput.

**Vector** — embedding generation and HNSW search. Measures the cost of in-process ML inference and nearest-neighbor queries at various dataset sizes.

## Running a Benchmark

Install the benchmark application and start a test:

```bash
cd ~/yeti/applications
git clone https://github.com/yetirocks/app-benchmarks.git
# Restart yeti — first compilation takes ~6 minutes, cached starts ~10 seconds

# Start a REST read benchmark
curl -X POST https://localhost:9996/app-benchmarks/api/runner \
  -H "Content-Type: application/json" \
  -d '{ "test": "rest-read" }'
```

The runner spawns a native Rust load generator binary, seeds test data, warms up for 5 seconds, then measures for 30 seconds. Results are automatically stored in the TestRun table with full metrics and per-second snapshots.

## Reading the Results

Every test run captures:

- **Throughput**: requests per second, measured per-second over the test duration
- **Latency percentiles**: p50, p95, p99, p99.9 via HdrHistogram with microsecond precision
- **Error rate**: failed requests as a percentage of total
- **Per-second snapshots**: time-series data showing throughput and latency variation across the test

The dashboard at `https://localhost:9996/app-benchmarks` shows all 17 test cards with best results and run history. Click any test to see the per-second time series, latency distribution, and historical trend.

## What the Numbers Mean

On a modern server (8 cores, 32GB RAM, NVMe storage), typical results look like:

**REST read**: 100K+ req/s, sub-millisecond p50, <5ms p99
**REST write**: 50K+ req/s with fsync, higher without durability guarantees
**GraphQL query**: 80K+ req/s for simple queries, lower for deeply nested queries
**WebSocket fan-out**: 10K+ messages/second to 1,000 subscribers
**Vector search**: Sub-millisecond HNSW queries on 100K vectors

These numbers vary with hardware, dataset size, and query complexity. That's why the benchmarking tool exists — so you can measure your specific workload on your specific hardware.

## Multi-Process Scaling

For high virtual user counts, the runner automatically scales process count — one process per 5,000 virtual users. Each process runs independently against the target with staggered launches to avoid TLS handshake storms. This lets you push beyond the throughput limits of a single TCP connection pool.

For cluster benchmarking, configure comma-delimited target URLs and the runner distributes load round-robin across multiple Yeti instances.

## Telemetry Suppression

Active benchmarks send `x-yeti-suppress-telemetry` headers to prevent the observer effect from skewing results. When benchmarking is active, telemetry collection (OpenTelemetry traces, Prometheus scraping) is paused on the target. This ensures the numbers reflect application performance, not monitoring overhead.

## Building Custom Workloads

The benchmark suite is extensible. Each workload is a Rust module that compiles into a standalone binary via the Yeti plugin system. To add a custom workload:

1. Create a new module in the `modules/` directory
2. Implement the load generation logic using the benchmark harness
3. Register the test in the runner configuration
4. Rebuild — the new test appears in the dashboard automatically

This is useful for measuring workloads specific to your application — custom query patterns, specific data shapes, or mixed read/write ratios that match your production traffic.

## Performance Regressions

The stored test results serve a second purpose: regression detection. After a schema change, dependency update, or configuration change, re-run the relevant benchmarks and compare against previous results. The dashboard highlights regressions at a glance — if your REST read throughput dropped 20% after a schema migration, you'll see it immediately.

Performance claims without reproducible benchmarks are marketing. The benchmark suite is there so you never have to trust our numbers — you can verify them on your own hardware, with your own data, in your own environment.

Try it at [app-benchmarks](https://yetirocks.com/developers/demos).
