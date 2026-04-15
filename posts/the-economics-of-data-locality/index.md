---
title: The Economics of Data Locality
description: "Every network hop has a cost. Every millisecond of latency has a price. Here's the math behind keeping data close to computation."
date: 2026-06-16
author: Jaxon Repp
category: Engineering
readingTime: 7 min read
linkedin_post: |
  Your cloud bill is a latency bill.

  Every network hop between your application and its data costs money. Not just bandwidth — compute time waiting for responses, connection pool overhead, retry logic, timeout handling, and the infrastructure to make it all "fast enough."

  The typical web application makes 6-12 network calls per request. Database query. Cache lookup. Auth service. Message broker. Search index. Embedding API. Each one adds latency, infrastructure cost, and failure modes.

  What if your data was in the same process as your application?

  No network hops. No connection pools. No serialization overhead. A function call instead of an HTTP request.

  That's what data locality means in practice: 5x throughput at 1/4 the resources. Not because the code is faster — because the architecture eliminated the tax.

  The full economics breakdown — with real numbers — is on the blog.

  #cloud #infrastructure #distributedsystems #performance #economics
linkedin_comment: |
  Full article: https://yetirocks.com/www/blog/the-economics-of-data-locality

  Based on research from the Data Locality series. Covers the physics of distance, the cost of network hops, and why unified architectures change the math.
---

Every architecture discussion eventually becomes a cost discussion. The question isn't just "how fast is it?" — it's "how much does that speed cost?"

The answer, for most modern web applications, is more than it should. Not because cloud providers are expensive (though they are), but because the standard architecture multiplies infrastructure requirements at every layer.

## The Network Hop Tax

A typical web request in a microservices architecture touches 6-12 network boundaries:

1. Load balancer → application server
2. Application server → authentication service
3. Application server → primary database
4. Application server → cache layer
5. Application server → search index
6. Application server → message broker
7. Application server → embedding API (if AI-enabled)
8. Application server → vector database (if AI-enabled)

Each hop adds latency (1-10ms on a fast network, 50-200ms for external APIs), compute cost (CPU cycles serializing and deserializing data), memory cost (connection pools, buffers, retry queues), and operational cost (monitoring, alerting, deployment, configuration for each service).

The infrastructure to support this isn't just the services themselves — it's the connective tissue between them. Service meshes, API gateways, connection pool managers, circuit breakers, retry policies, timeout configurations. Each one exists because the architecture created a problem that needs solving.

## The Physics of Distance

There's a hard physical limit that no optimization can overcome: the speed of light. A network round trip within a data center takes 0.5-1ms. Between availability zones, 1-5ms. Between regions, 20-100ms. Between continents, 100-300ms.

When your application makes 6 network calls per request, the theoretical minimum latency is 6x the round-trip time — before any computation happens. In a single-AZ deployment, that's 3-6ms of pure network latency. In a multi-region deployment, it's 120-600ms.

This isn't a software problem. It's a physics problem. And the only way to solve a physics problem is to reduce the distance.

## Data Locality: The Economic Argument

Data locality means putting data and computation in the same process. Not the same server — the same process. A function call instead of a network request. Shared memory instead of serialization.

The economic impact is straightforward:

**Fewer servers.** When your application doesn't need a separate database server, cache server, message broker, search index, and embedding service, you need fewer servers. Yeti runs all of these in a single process on a single machine. The infrastructure cost is one server instead of six.

**Less bandwidth.** Internal network traffic between microservices is often the largest bandwidth consumer in a cloud deployment. When the database is in-process, that traffic drops to zero.

**Less compute waste.** Serialization and deserialization consume CPU cycles. JSON parsing, Protocol Buffer encoding, TLS handshakes — all of this work exists only because data crosses network boundaries. Eliminate the boundaries, eliminate the work.

**Less operational overhead.** Each service in a microservices architecture needs monitoring, alerting, deployment pipelines, configuration management, and on-call rotation. A single binary needs one of each.

**Predictable scaling.** When your application is a single binary with predictable resource utilization, capacity planning is arithmetic. You measure throughput and latency on one node, then multiply. No complex interaction effects between services under load.

## The Numbers

Yeti's unified architecture — database, cache, HTTP server, message broker, vector search, ML inference in one process — delivers measurable economic advantages over an equivalent distributed architecture:

**5x throughput** — eliminating network hops and serialization overhead means each request completes faster, so the same hardware handles more requests.

**Half the latency** — a function call to RocksDB takes microseconds. A network call to PostgreSQL takes milliseconds. The difference compounds across every database interaction in a request.

**Quarter the resources** — one server running Yeti replaces the database server, cache server, message broker, search index, and application server that a traditional architecture requires.

**Stable resource utilization** — no garbage collection means no memory spikes. No connection pool exhaustion under load. Resource usage is predictable and flat, which means you provision for your actual workload, not for the worst-case GC pause.

## When Locality Breaks Down

Data locality isn't the right answer for every workload. It breaks down when:

**Your data exceeds a single node's storage.** RocksDB is fast, but it runs on one machine. For petabyte-scale datasets, you need distributed storage. Yeti's `@distribute` directive handles horizontal scaling across nodes, but the economics shift — network hops reappear between cluster members.

**Your write volume exceeds a single node's throughput.** At some scale, you need to shard writes across machines. The unified architecture still wins on the read path (local cache, local index, local inference), but distributed writes introduce coordination costs.

**Your compliance requirements mandate service isolation.** Some security frameworks require network-level isolation between application tiers. In those environments, the network hop isn't optional — it's a requirement.

For the vast majority of applications — those serving thousands to millions of requests per day on datasets that fit on a single server with NVMe storage — data locality eliminates infrastructure cost that exists only because the traditional architecture chose to separate things that don't need to be separate.

## The Bottom Line

Cloud bills grow with architectural complexity. Every service you add creates costs: compute, bandwidth, operational overhead, and the engineering time to keep it all running. The question isn't whether you can afford the cloud — it's whether you can afford the architecture.

Data locality changes the math. Not by being cheaper per unit of compute, but by needing dramatically less of it.

One binary. One server. One bill.
