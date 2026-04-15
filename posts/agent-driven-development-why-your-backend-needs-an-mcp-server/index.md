---
title: "Agent-Driven Development: Why Your Backend Needs an MCP Server"
description: Every AI agent needs tools. MCP is the open standard for giving them. Here's why your backend should speak it natively.
date: 2026-05-09
author: Jaxon Repp
category: Engineering
readingTime: 7 min read
linkedin_post: |
  Your backend is about to become an AI endpoint.

  Not because you decided it should. Because your users' AI agents need to talk to it.

  MCP (Model Context Protocol) is the open standard for connecting AI agents to external tools and data. Claude, Cursor, Windsurf, and dozens of other AI products already speak it. When a developer asks their agent to "check the inventory" or "create a new user," MCP is what lets the agent discover your API, understand its schema, and make the call.

  The problem: building an MCP server from scratch means implementing JSON-RPC 2.0, session management, tool discovery, input schema generation, and audit logging. Hundreds of lines of boilerplate before you serve your first record.

  In Yeti, you add @export to a schema type. That's it.

  One directive generates a complete MCP endpoint — tools, resources, prompts, and sessions. Field types become JSON Schema automatically. Every call is audit-logged. Session management handles reconnects and cleanup.

  Agent-driven development isn't coming. It's here. The question is whether your backend is ready for it.

  #mcp #ai #agents #distributedsystems #rust
linkedin_comment: |
  Full article: https://yetirocks.com/www/blog/agent-driven-development-why-your-backend-needs-an-mcp-server

  Includes the full MCP schema, curl examples, and a walkthrough of how Yeti auto-generates tool discovery from GraphQL types.
---

The way developers build software is changing. Not in the way we predicted — not low-code platforms or visual builders or yet another JavaScript framework. The shift is agents.

AI agents don't browse your documentation. They don't read your OpenAPI spec and write fetch calls. They discover tools at runtime, understand their schemas, and invoke them through a protocol. That protocol is MCP — the Model Context Protocol — and it's already the standard. Claude, Cursor, Windsurf, and a growing list of AI products speak it natively.

This has a direct consequence for every backend in production: if an AI agent can't discover your API programmatically, it can't use your API at all.

## What MCP Actually Does

MCP is a JSON-RPC 2.0 protocol that gives AI agents three things: tools they can call, resources they can read, and prompts they can use. An MCP server advertises these capabilities through a discovery endpoint. The agent connects, learns what's available, and starts making calls.

A typical MCP interaction looks like this:

1. Agent sends `initialize` — establishes a session, exchanges capabilities
2. Agent calls `tools/list` — discovers what operations are available
3. Agent calls `tools/call` — executes an operation with typed parameters
4. Server returns structured results — the agent processes them and decides what to do next

The protocol handles session management, supports streaming via Server-Sent Events, and defines a clean schema for tool inputs and outputs. It's simple, well-specified, and already widely adopted.

## The Boilerplate Problem

Building an MCP server from scratch means implementing:

- JSON-RPC 2.0 request/response parsing
- Session management with `mcp-session-id` headers
- Tool discovery with JSON Schema input definitions
- Resource listing and content retrieval
- Prompt templates with argument substitution
- Error handling that follows the spec
- Audit logging for compliance

That's hundreds of lines of protocol code before you serve your first record. Most teams don't have time for this, so they don't build it. Their backend stays invisible to AI agents.

## Schema-Driven MCP

Yeti takes a different approach. Every table with `@export` automatically becomes an MCP endpoint. The schema defines the data model, and the platform generates the MCP server — tool definitions, input schemas, resource listings, and session management — with zero application code.

```graphql
type Product @table @export(mcp: true) {
  id: String! @primaryKey
  name: String!
  category: String!
  description: String!
  price: Float!
  inStock: Boolean!
}
```

That schema generates a complete MCP server. An AI agent connecting to `/demo-mcp/mcp` will discover tools for creating, reading, updating, deleting, and searching products. Each tool has a fully typed input schema derived from the GraphQL field definitions.

Here's what the agent sees when it calls `tools/list`:

```json
{
  "tools": [
    {
      "name": "create_Product",
      "description": "Create a new Product record",
      "inputSchema": {
        "type": "object",
        "properties": {
          "id": { "type": "string" },
          "name": { "type": "string" },
          "category": { "type": "string" },
          "description": { "type": "string" },
          "price": { "type": "number" },
          "inStock": { "type": "boolean" }
        },
        "required": ["id", "name", "category", "description", "price", "inStock"]
      }
    }
  ]
}
```

The agent doesn't need documentation. It doesn't need an SDK. It reads the tool schema, constructs a valid call, and gets structured results back.

## Every Call Is Auditable

MCP sessions generate audit records automatically. Every `tools/call`, `resources/read`, and `prompts/get` is captured with the session ID, client identity, timing, and response status. In a world where AI agents are making autonomous API calls, knowing exactly what they did and when is not optional — it's a compliance requirement.

Yeti's `@audit` directive extends this to the data layer. Combine `@export(mcp: true)` with `@audit` and every agent-initiated mutation is tracked with before/after state, user identity, and timestamps.

## Agent-Driven Development Is the Default

This isn't about building for a future use case. It's about recognizing what's already happening. Developers are using AI agents as their primary interface to systems. When a developer asks Claude to "add a product to the catalog," the agent needs a tool to call. MCP provides the protocol. Your backend needs to speak it.

The choice is build the MCP server by hand — JSON-RPC parsing, session management, schema generation, audit logging — or define your data model in a schema and let the platform generate it.

```bash
# Initialize a session
curl -X POST https://localhost:9996/demo-mcp/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-03-26",
      "clientInfo": { "name": "my-agent", "version": "1.0" }
    }
  }'

# Search products
curl -X POST https://localhost:9996/demo-mcp/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "search_Product",
      "arguments": { "query": "electronics under $500" }
    }
  }'
```

Every `@export` type is an MCP server. Every field becomes a typed tool parameter. Every call is audit-logged. The schema is the single source of truth — for your REST API, your GraphQL endpoint, your real-time streams, and now your AI agent interface.

If you're building a backend today and not thinking about MCP, you're building for a world that's already gone. The agents are here. Your backend needs to be ready.

Try the [demo-mcp](https://yetirocks.com/developers/demos) interactive playground to see it in action.
