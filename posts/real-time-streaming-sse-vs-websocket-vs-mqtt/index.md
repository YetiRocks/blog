---
title: "Real-Time Streaming: SSE vs WebSocket vs MQTT"
description: Three protocols, one schema, zero glue code. A practical comparison with latency, throughput, and use case recommendations.
date: 2026-06-12
author: Jaxon Repp
category: Engineering
readingTime: 7 min read
linkedin_post: |
  SSE, WebSocket, or MQTT?

  Wrong question. The right question is: why are you choosing one?

  Most real-time applications need multiple transports. Your browser wants SSE for server-push notifications. Your mobile app wants WebSocket for bidirectional chat. Your IoT fleet wants MQTT for lightweight pub/sub over constrained networks.

  Building all three means three server implementations, three connection managers, three sets of reconnect logic. Or you pick one and tell the other use cases to adapt.

  In Yeti, you declare one schema:

  type Message @table @export(sse: true, ws: true, mqtt: true) {
    id: ID! @primaryKey
    title: String!
    content: String!
  }

  That generates all three transports from the same data. POST a message via REST, and it propagates to every SSE, WebSocket, and MQTT subscriber simultaneously. One schema, five protocols, zero glue code.

  Full comparison with latency numbers and use case recommendations on the blog.

  #realtime #websocket #mqtt #sse #distributedsystems
linkedin_comment: |
  Full article: https://yetirocks.com/www/blog/real-time-streaming-sse-vs-websocket-vs-mqtt

  Covers connection semantics, reconnect patterns, and when to use each protocol — with a live demo you can try yourself.
---

Every real-time application starts with the same question: which protocol? SSE for simplicity. WebSocket for bidirectional communication. MQTT for IoT and constrained networks. The usual answer is to pick one and build around it.

That's the wrong answer. Different clients have different needs, and forcing a browser dashboard and an IoT sensor onto the same protocol means one of them is working harder than it should.

## The Three Protocols

**Server-Sent Events (SSE)** is HTTP-native server push. The client opens a GET request, the server holds it open and sends events as they happen. It's unidirectional (server to client only), works through proxies and load balancers without special configuration, and reconnects automatically via the `Last-Event-ID` header. Use it for dashboards, notification feeds, and any scenario where the client just needs to listen.

**WebSocket** is a bidirectional, full-duplex channel over a single TCP connection. Both sides can send messages at any time. It's the right choice for chat, collaborative editing, and interactive applications where the client needs to send data back frequently. The tradeoff is more complex connection management — WebSocket connections don't survive proxy timeouts as gracefully as SSE, and you need to handle reconnection manually.

**MQTT** is a publish/subscribe protocol designed for constrained networks. Messages are routed through topics, clients subscribe to patterns, and the broker handles delivery. QoS levels (0, 1, 2) let you trade latency for delivery guarantees. It's the standard for IoT, sensor networks, and any scenario where bandwidth is limited or connections are unreliable.

## One Schema, All Three

In Yeti, you don't choose. You declare which transports you want, and the platform generates all of them from the same data model:

```graphql
type Message @table(database: "demo-realtime") @export(
    rest: true
    sse: true
    ws: true
    mqtt: true
    public: [read, create, delete, subscribe, connect]
) {
    id: ID! @primaryKey
    title: String!
    content: String!
}
```

That single type declaration creates:

- **REST endpoint** — `POST /demo-realtime/message` for writes, `GET` for reads
- **SSE stream** — `GET /demo-realtime/message/?$format=sse` for server-push
- **WebSocket** — `wss://localhost:9996/demo-realtime/message` for bidirectional messaging
- **MQTT topic** — `demo-realtime/message` on the built-in MQTTS broker (port 8883)
- **gRPC stream** — for high-performance service-to-service streaming

POST a message through any transport, and it propagates to every subscriber on every protocol simultaneously. The React dashboard in [demo-realtime](https://yetirocks.com/developers/demos) shows all five side by side — you can watch a single REST POST arrive on SSE, WebSocket, MQTT, and gRPC in real time.

## When to Use Which

**SSE** when:
- Your client only needs to receive updates (dashboards, feeds, notifications)
- You need automatic reconnection with event replay
- You're working behind corporate proxies that block WebSocket upgrades
- You want the simplest possible client implementation (it's just a GET request)

**WebSocket** when:
- The client needs to send messages frequently (chat, collaborative editing)
- You need low-latency bidirectional communication
- You're building interactive applications with real-time user input

**MQTT** when:
- You're connecting IoT devices or sensors with limited bandwidth
- You need QoS guarantees (at-most-once, at-least-once, exactly-once delivery)
- Your clients subscribe to topic patterns (e.g., `sensors/building-a/+/temperature`)
- You need a lightweight protocol that works over constrained networks

**gRPC** when:
- You're doing service-to-service streaming between backend services
- You need strongly typed contracts with Protocol Buffer schemas
- You want multiplexed streaming over HTTP/2

## No External Broker

The critical difference in Yeti's approach is that there's no external message broker. No Redis pub/sub. No RabbitMQ. No Kafka for the transport layer. The MQTT broker runs inside the Yeti binary. SSE and WebSocket connections are managed by the same process that handles storage.

This means a message goes from the write path to every subscriber without crossing a network boundary. The storage write and the fan-out happen in the same process, atomically. If the write succeeds, every subscriber gets the message. If it fails, nobody does.

For most real-time applications — dashboards with hundreds to thousands of concurrent connections, chat systems, IoT fleets with tens of thousands of devices — this single-process model is faster and simpler than a distributed broker. You trade horizontal fan-out scaling for zero-latency delivery and operational simplicity.

## Connection Management

Production real-time systems need to handle disconnects gracefully. Each protocol has different semantics:

**SSE** reconnects automatically. The browser's `EventSource` API retries with exponential backoff and sends `Last-Event-ID` so the server can replay missed events. This is built into the protocol — you get it for free.

**WebSocket** requires manual reconnect logic. The demo-realtime frontend implements configurable exponential backoff with jitter. When a WebSocket reconnects, it re-subscribes and the server begins streaming current events.

**MQTT** handles reconnection at the protocol level with configurable clean/persistent sessions. A persistent session (clean session = false) preserves subscriptions across reconnects. The broker queues messages for disconnected clients up to the configured QoS level.

All three work out of the box with `@export`. The protocol-specific connection management happens inside Yeti — your schema stays the same regardless of which transports your clients use.

Try the [demo-realtime](https://yetirocks.com/developers/demos) dashboard to see all three protocols side by side with a live message stream.
