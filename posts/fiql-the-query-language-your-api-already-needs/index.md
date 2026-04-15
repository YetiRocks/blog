---
title: "FIQL: The Query Language Your API Already Needs"
description: "Equality, ranges, regex, full-text search, joins, sorting, pagination — all in the URL. No custom endpoints, no query builders, no GraphQL required."
date: 2026-06-26
author: Jaxon Repp
category: Engineering
readingTime: 7 min read
linkedin_post: |
  Your REST API has a query problem.

  Simple CRUD is easy. But the moment a user needs "products under $50 in the electronics category that are in stock, sorted by price" — you're writing a custom endpoint or bolting on a query language.

  FIQL (Feed Item Query Language) solves this at the URL level. Every Yeti REST endpoint speaks it out of the box:

  /Products/?category==electronics;price=lt=50;inStock==true&sort=price&limit=10

  That's a URL. Not a POST body. Not a GraphQL query. A URL you can paste into a browser, share in a Slack message, or curl from the command line.

  It goes deeper: regex matching, full-text search, range queries, relationship joins, nested field access, and vector search — all in the same syntax.

  50+ live examples in the demo. Full breakdown on the blog.

  #api #rest #developer #distributedsystems
linkedin_comment: |
  Full article: https://yetirocks.com/www/blog/fiql-the-query-language-your-api-already-needs

  Every example works with curl. The demo-fiql app ships 50 products across 22 brands for hands-on testing.
---

REST APIs have a query problem. The CRUD operations are simple — GET, POST, PUT, DELETE map cleanly to read, create, update, delete. But the moment you need to filter, sort, paginate, search, or join, you're either writing custom endpoints or adopting a query language.

Most teams solve this ad hoc. Query parameters for simple filters (`?status=active`), request bodies for complex queries, or GraphQL for everything. Each approach has tradeoffs: custom endpoints don't compose, request-body queries aren't bookmarkable, and GraphQL adds a learning curve and tooling dependency.

FIQL (Feed Item Query Language, based on the RFC draft) takes a different approach: put the query in the URL.

## The Basics

Every Yeti REST endpoint with `@export` speaks FIQL automatically. No configuration, no middleware, no query builder.

**Exact match:**
```
GET /Products/?category==electronics
```

**Not equal:**
```
GET /Products/?category!=electronics
```

**Range queries:**
```
GET /Products/?price=gt=100;price=lt=500
```

**Boolean:**
```
GET /Products/?inStock==true
```

**Sorting:**
```
GET /Products/?sort=price&order=asc
```

**Pagination:**
```
GET /Products/?limit=10&offset=20
```

**Combined:**
```
GET /Products/?category==electronics;price=lt=500;inStock==true&sort=price&limit=10
```

That last URL says: "electronics products under $500 that are in stock, sorted by price, first 10 results." It's readable, bookmarkable, shareable, and works with any HTTP client.

## Beyond the Basics

FIQL gets interesting when you go past simple equality checks.

**Regex matching:**
```
GET /Products/?name=re="Ultra.*"
```

**Full-text search:**
```
GET /Products/?name=search="wireless headphones"
```

Full-text search uses the `@indexed(type: "fulltext")` directive on the field. It's not keyword matching — it's tokenized, stemmed search with relevance scoring.

**Relationship joins:**
```
GET /Products/?brand.country==Japan
```

This queries Products but filters on a field from the related Brand table. The `@relationship` directive in the schema defines the join:

```graphql
type Products @table @export(public: [read]) {
    id: ID! @primaryKey
    name: String! @indexed(type: "fulltext")
    price: Float! @indexed
    category: String! @indexed
    inStock: Boolean! @indexed
    brandId: ID @indexed
    brand: Brand @relationship(from: "brandId")
}

type Brand @table @export(public: [read]) {
    id: ID! @primaryKey
    name: String! @indexed
    country: String!
    foundedYear: Int
    products: [Products] @relationship(to: "brandId")
}
```

The join is resolved server-side. The client queries one endpoint with a dot-notation field path. No N+1 queries, no client-side join logic.

**Vector search:**
```
GET /Articles/?search=storage+engine+tradeoffs&vector=true&limit=5
```

Vector search is available on any table with a `Vector` field. The query text is embedded in-process and searched against the HNSW index. Same URL syntax, same HTTP client.

**Composite index queries:**
```
GET /Products/?category==electronics;price=gt=100
```

With `@compositeIndex(fields: "category,price")` on the type, this query uses a composite index for efficient two-field lookups without scanning.

## Why URL-Based Queries Matter

**Bookmarkable.** A FIQL URL is a saved query. Paste it in a browser, share it in documentation, store it in a monitoring dashboard. No request body required.

**Cacheable.** GET requests with query parameters are natively cacheable by HTTP proxies, CDNs, and browsers. POST-based query endpoints require custom cache keys.

**Debuggable.** When something goes wrong, the query is in the URL. It shows up in access logs, in monitoring tools, in error reports. No need to reconstruct a request body from a log entry.

**Composable.** FIQL queries compose with standard HTTP features. Add `Accept: text/event-stream` and the same filtered query becomes an SSE subscription. Add `$format=csv` and you get CSV export. The query language and the transport are orthogonal.

**Agent-friendly.** AI agents discover FIQL parameters through MCP tool schemas. The `search_Products` tool includes the query parameter with its schema, and the agent constructs valid FIQL expressions without documentation.

## The 50-Example Reference

The [demo-fiql](https://yetirocks.com/developers/demos) application ships a product catalog with 50 items across 6 categories and 22 brands. The React UI provides 50+ curated query examples organized from basics through advanced joins, each with:

- A plain-English description of what the query does
- The raw FIQL URL
- The parsed resource JSON showing how the server interprets the query
- Live execution against real data

Every example works with curl. The application uses `@export(public: [read])`, so no authentication is required for testing.

## Schema-Driven Query Capabilities

The key insight is that FIQL capabilities are derived from the schema. `@indexed` fields get equality and range queries. `@indexed(type: "fulltext")` fields get full-text search. `@relationship` fields get dot-notation joins. `Vector` fields get semantic search. `@compositeIndex` fields get efficient multi-field queries.

You don't configure a query engine. You declare your data model, and the query capabilities emerge from the field types and directives. Add an index, gain a query operator. It's the same schema-first principle that drives everything in Yeti — define the model, and the platform generates the interface.

Try the full query reference at [demo-fiql](https://yetirocks.com/developers/demos).
