---
title: "Streaming Kafka into Your Application in 10 Lines"
description: "Connect a Kafka topic to a Yeti table with a config snippet. Messages stream in, SSE streams out, your data stays queryable."
date: 2026-07-03
author: Jaxon Repp
category: Engineering
readingTime: 5 min read
linkedin_post: |
  "We need to consume Kafka and make the data queryable."

  Typical solution: Write a consumer service. Parse messages. Transform schemas. Insert into a database. Build an API. Add monitoring. Deploy. Maintain.

  That's 6 services and weeks of work for what should be a configuration change.

  In Yeti, it's 10 lines of YAML:

  kafka:
    events:
      - topic: my-topic
        brokers: kafka:9092
        table: KafkaMessage
        group_id: yeti-consumer

  That config snippet connects a Kafka broker, consumes messages from a topic, and streams them into a Yeti table. The table has automatic REST, SSE, WebSocket, MQTT, and MCP endpoints. The data is queryable via FIQL. Messages arrive in real time via SSE.

  No consumer code. No transformation logic. No separate API layer.

  Full walkthrough on the blog.

  #kafka #streaming #distributedsystems #dataengineering
linkedin_comment: |
  Full article: https://yetirocks.com/www/blog/streaming-kafka-into-your-application-in-10-lines

  Supports SASL auth (PLAIN, SCRAM-SHA-256/512) and TLS. The demo-kafka app includes an interactive bridge configuration tool.
---

Kafka is the backbone of event streaming in most enterprises. But consuming Kafka and making the data useful to applications is still harder than it should be. The standard pattern: write a consumer service in Java or Python, parse and validate messages, transform them into your application's schema, insert them into a database, build an API on top of that database, add monitoring, and deploy the whole pipeline.

That's a lot of moving parts for "read messages from a topic and let people query them."

## 10 Lines of Configuration

Yeti's Kafka bridge connects a Kafka broker to a Yeti table with a config snippet:

```yaml
kafka:
  events:
    - topic: my-events
      brokers: kafka-broker:9092
      table: KafkaMessage
      group_id: yeti-consumer
      auth:
        mechanism: PLAIN
        username: ${KAFKA_USER}
        password: ${KAFKA_PASS}
      tls: true
```

That's it. When Yeti starts, the `yeti-kafka` crate connects to the broker, subscribes to the topic, and streams messages into the specified table. Each message becomes a record with the topic, key, payload, and timestamp.

The table schema:

```graphql
type KafkaMessage @table @export(
    rest: true
    sse: true
    public: [read, create, delete, subscribe]
) {
    id: ID! @primaryKey
    topic: String! @indexed
    key: String
    payload: String!
    __createdAt__: String
}
```

That schema gives the Kafka data:

- **REST API** — query, filter, and paginate messages with FIQL
- **SSE stream** — subscribe and receive messages in real time as they arrive from Kafka
- **WebSocket** — bidirectional streaming for interactive dashboards
- **MQTT** — republish to IoT devices or other subscribers
- **MCP** — AI agents can discover and query the Kafka data automatically

No consumer code. No transformation service. No separate API layer.

## Real-Time Fan-Out

The most powerful aspect of the Kafka bridge is what happens after ingestion. Messages that arrive from Kafka are written to the Yeti table, which triggers automatic fan-out to every active subscriber on every transport.

Open an SSE connection:

```bash
curl -N "https://localhost:9996/demo-kafka/message/?$format=sse"
```

Publish a message to the Kafka topic, and it appears in the SSE stream within milliseconds. The path is: Kafka → `yeti-kafka` consumer → RocksDB write → SSE/WebSocket/MQTT fan-out. One process, no intermediate broker.

This turns Kafka from a backend infrastructure component into a real-time data source for frontends, dashboards, and monitoring UIs — without writing a single line of transformation or routing logic.

## Querying Kafka Data

Once messages are in a Yeti table, they're queryable with FIQL:

```bash
# All messages from a specific topic
curl "https://localhost:9996/demo-kafka/message/?topic==user-events"

# Messages from the last hour
curl "https://localhost:9996/demo-kafka/message/?__createdAt__=gt=2026-07-03T12:00:00Z"

# Filter by topic and payload content
curl "https://localhost:9996/demo-kafka/message/?topic==orders;payload=search=refund"

# Paginate through history
curl "https://localhost:9996/demo-kafka/message/?sort=__createdAt__&order=desc&limit=50&offset=0"
```

Every query that works on any Yeti table works on Kafka data. Full-text search, range queries, pagination, sorting — it's all available because the data lives in the same storage engine as everything else.

## Authentication Support

Enterprise Kafka deployments require authentication. The bridge supports:

- **SASL/PLAIN** — username/password authentication
- **SASL/SCRAM-SHA-256** — challenge-response authentication
- **SASL/SCRAM-SHA-512** — stronger hash variant
- **TLS** — encrypted connections to the broker

Credentials are configured in `yeti-config.yaml` and support environment variable substitution for secrets management. No hardcoded credentials in config files.

## Multiple Topics, Multiple Tables

You can configure multiple bridges to consume from different topics into different tables:

```yaml
kafka:
  events:
    - topic: user-events
      brokers: kafka:9092
      table: UserEvent
      group_id: yeti-users
    - topic: order-events
      brokers: kafka:9092
      table: OrderEvent
      group_id: yeti-orders
    - topic: sensor-data
      brokers: iot-kafka:9092
      table: SensorReading
      group_id: yeti-sensors
      tls: true
```

Each topic gets its own table, its own schema, its own API endpoints, and its own SSE/WebSocket/MQTT subscribers. The tables can have different schemas — `UserEvent` might have `userId` and `action` fields while `SensorReading` has `deviceId` and `temperature`.

## When to Use the Kafka Bridge

The bridge is designed for teams that already run Kafka and want to make that data accessible to applications without building a custom consumer pipeline. Common use cases:

**Dashboards** — stream Kafka events into a Yeti table and subscribe via SSE for a real-time dashboard. No intermediate database or API layer.

**Audit trails** — consume audit events from Kafka and store them in a queryable, indexed table with automatic retention.

**Data lakes to APIs** — turn Kafka topics into REST APIs that frontend teams can consume without access to the Kafka cluster.

**Cross-system sync** — bridge events from one system's Kafka topics into Yeti tables that power a different application's UI.

The [demo-kafka](https://yetirocks.com/developers/demos) application includes an interactive bridge configuration tool — generate the YAML snippet, test the connection, and watch messages stream in real time.
