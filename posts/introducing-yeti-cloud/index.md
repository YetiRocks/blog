---
title: Introducing Yeti Cloud
description: "From self-hosted to managed. The same Rust-native platform — deployed, scaled, and monitored for you."
date: 2026-06-01
author: Jaxon Repp
category: Product
readingTime: 5 min read
linkedin_post: |
  One month ago, we launched Yeti self-hosted.

  Engineers ran it on bare metal, on EC2 instances, on Raspberry Pis. They benchmarked it against their existing stacks and sent us the results. They found bugs we missed and edge cases we hadn't considered.

  They also said the same thing, over and over:

  "I don't want to manage infrastructure. I want to deploy my schema and go."

  Today we're launching Yeti Cloud.

  Same platform. Same performance. Same single-binary architecture. But you push a schema and we handle the rest — deployment, TLS, monitoring, backups, scaling.

  Your data stays in your region. Your latency stays sub-millisecond. Your infrastructure bill stays a fraction of what you'd spend assembling the same capabilities from separate services.

  If you've been waiting for the managed version — it's here.

  → cloud.yetirocks.com

  #launch #cloud #rust #distributedsystems #startup
linkedin_comment: |
  Full announcement: https://yetirocks.com/www/blog/introducing-yeti-cloud

  Self-hosted isn't going anywhere — Cloud is just the managed option for teams that want to focus on their application, not their infrastructure.
---

A month ago, we launched Yeti as a self-hosted binary. Download it, run it, own your infrastructure. The response exceeded our expectations — engineers ran it on everything from bare metal servers to Raspberry Pis, benchmarked it against their existing stacks, and pushed it in ways we hadn't anticipated.

They also gave us consistent feedback: the platform is exactly what we want, but we don't want to manage infrastructure.

Today we're launching Yeti Cloud.

## What Yeti Cloud Is

Yeti Cloud is the same platform — the same Rust binary, the same RocksDB storage engine, the same schema-driven architecture — deployed and managed for you. You push a GraphQL schema, and Yeti Cloud handles deployment, TLS certificates, monitoring, backups, and scaling.

Your application gets a dedicated endpoint at `your-app.cloud.yetirocks.com`. Every feature that works in self-hosted works in Cloud: REST, GraphQL, SSE, WebSocket, MQTT, MCP, vector search, authentication, audit logging. The schema is the same. The API is the same. The performance is the same.

## What We Learned From Self-Hosted

The month between self-hosted GA and Cloud launch was intentional. We called it the teaching phase — but honestly, we learned as much as our users did.

**Deployment patterns vary wildly.** Some teams run Yeti on a single server behind nginx. Others run it in Kubernetes pods. One team embedded it in an edge computing pipeline on ARM devices. The self-hosted binary handles all of these because it's a single static executable with no dependencies — but each environment surfaced different operational questions.

**Schema iteration is fast, infrastructure iteration is slow.** Teams iterated on their schemas daily — adding fields, adjusting indexes, enabling new transports. The schema changes took seconds. The infrastructure changes (TLS renewal, backup configuration, monitoring setup) took hours. Cloud eliminates that asymmetry.

**Performance expectations are high.** When your platform promises sub-millisecond latency and 100K+ req/s, users verify it. The benchmark suite became one of our most-used features. Cloud runs the same binary on optimized infrastructure — the numbers you saw in self-hosted benchmarks translate directly.

## How It Works

**Deploy:** Push your schema files to Yeti Cloud via the CLI or web console. The platform validates the schema, compiles any application plugins, and deploys to your designated region.

**Scale:** Yeti Cloud monitors request volume and resource utilization. Single-node deployments handle most workloads. When you need more, horizontal scaling distributes tables across nodes using the `@distribute` directive in your schema.

**Monitor:** Built-in dashboards show request throughput, latency percentiles, storage utilization, and error rates. Alerts fire when thresholds are exceeded. All powered by the same yeti-telemetry crate that runs in self-hosted.

**Secure:** TLS is automatic. Authentication follows your schema — Basic, JWT, OAuth, RBAC with field-level masking. Data stays in your chosen region. Audit logs capture every mutation.

## Pricing

Yeti Cloud is priced on resource utilization — compute, storage, and bandwidth. No per-seat licensing. No per-request fees. No surprise bills for vector search queries or real-time streaming connections.

Self-hosted remains free and fully featured. Cloud is for teams that want managed infrastructure without giving up performance or control over their data model.

## Self-Hosted Isn't Going Anywhere

Yeti Cloud doesn't replace self-hosted — it complements it. The self-hosted binary continues to ship as a free download. Everything we build for Cloud ships in self-hosted too. If you want to own your infrastructure, run on edge devices, or deploy in air-gapped environments, self-hosted is the right choice.

Cloud is for the teams that told us: "I love the platform. I don't love managing servers."

We heard you.

Get started at [cloud.yetirocks.com](https://cloud.yetirocks.com).
