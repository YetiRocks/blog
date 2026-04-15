---
title: Zero-Code Auth in 30 Lines of GraphQL
description: "Basic, JWT, OAuth, and RBAC with field-level masking. No middleware, no token parsing, no session management — just a schema and a config file."
date: 2026-06-19
author: Jaxon Repp
category: Engineering
readingTime: 6 min read
linkedin_post: |
  How many lines of code is your auth system?

  Middleware for token validation. Session management. Password hashing. JWT generation. OAuth callback handlers. Role definitions. Permission checks on every endpoint. Field-level access control.

  In most backends, auth is 15-20% of the codebase. It's also the part most likely to have a security vulnerability.

  In Yeti, auth is a schema and a config file. No code.

  → Basic auth with Argon2id password hashing
  → JWT generation and validation
  → OAuth providers (Google, GitHub)
  → RBAC with field-level masking

  The admin sees salary, SSN, and home address. The viewer sees the same endpoint — those fields are stripped server-side before the response leaves the wire. Not filtered in your code. Stripped by the platform.

  30 lines of GraphQL. Zero application code. Full auth system.

  #security #auth #graphql #distributedsystems
linkedin_comment: |
  Full article: https://yetirocks.com/www/blog/zero-code-auth-in-30-lines-of-graphql

  Includes the schema, config.yaml, curl examples for all three auth methods, and a walkthrough of field-level RBAC in action.
---

Authentication is table stakes for any production application. It's also one of the most dangerous things to build yourself. Password hashing algorithms, token validation, OAuth flows, session management, RBAC enforcement — each one has well-documented pitfalls that turn into CVEs when implemented incorrectly.

Most frameworks provide auth middleware that you wire into your request pipeline. It's better than building from scratch, but it's still code you write, test, and maintain. And it's code that sits between every request and your business logic.

Yeti eliminates that code entirely. Authentication and authorization are configured declaratively — a schema defines the data, a config file defines the auth methods, and the platform enforces both automatically.

## The Schema

```graphql
type Employee @table @export {
  id: String!
  name: String!
  email: String!
  department: String!
  title: String!
  salary: Float
  ssn: String
  homeAddress: String
  personalEmail: String
}
```

That's the data model. Thirteen lines. Now add auth.

## The Config

```yaml
auth:
  basic:
    enabled: true
    users:
      - username: admin
        password: admin123
        roles: [admin]
      - username: user
        password: user123
        roles: [viewer]
  jwt:
    enabled: true
    secret: your-jwt-secret
    expiration: 3600
  oauth:
    providers:
      - name: google
        client_id: ${GOOGLE_CLIENT_ID}
        client_secret: ${GOOGLE_CLIENT_SECRET}
        role_mapping:
          google: admin
      - name: github
        client_id: ${GITHUB_CLIENT_ID}
        client_secret: ${GITHUB_CLIENT_SECRET}
        role_mapping:
          github: viewer

roles:
  admin:
    tables:
      Employee: [read, create, update, delete]
  viewer:
    tables:
      Employee:
        operations: [read]
        exclude_fields: [salary, ssn, homeAddress, personalEmail]
```

That configuration gives you three auth methods (Basic, JWT, OAuth), two roles, and field-level access control. The total: 30 lines of schema + config. Zero application code.

## What the Platform Enforces

**Basic auth** uses Argon2id for password hashing — the current recommendation from OWASP. Passwords are never stored in plaintext. The `Authorization: Basic` header is decoded, the password is verified against the Argon2id hash, and the associated roles are attached to the request context.

**JWT** tokens are generated via a login endpoint and validated on every subsequent request. The token contains the user's roles, expiration time, and a signature verified against the configured secret. No session table, no Redis, no server-side state.

**OAuth** redirects to the configured provider (Google, GitHub), handles the callback, exchanges the authorization code for user info, maps the provider to a role via `role_mapping`, and issues a JWT for subsequent requests. The entire flow is handled by `yeti-auth` — no callback handler code required.

**RBAC** is enforced at the serialization layer. The `viewer` role has `exclude_fields: [salary, ssn, homeAddress, personalEmail]`. When a viewer queries the Employee table, those fields are stripped from the response before it leaves the wire. Not filtered in application code. Not hidden by the frontend. Removed by the platform before serialization.

```bash
# Admin sees everything
curl -s https://localhost:9996/demo-authentication/Employee/?limit=1 \
  -u admin:admin123 | jq

{
  "id": "emp-001",
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "department": "Engineering",
  "title": "Senior Developer",
  "salary": 145000,
  "ssn": "123-45-6789",
  "homeAddress": "123 Oak Street, San Francisco, CA",
  "personalEmail": "alice.j@personal.com"
}

# Viewer sees the same endpoint — sensitive fields stripped
curl -s https://localhost:9996/demo-authentication/Employee/?limit=1 \
  -u user:user123 | jq

{
  "id": "emp-001",
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "department": "Engineering",
  "title": "Senior Developer"
}
```

Same endpoint. Same query. Different response based on the authenticated role. The viewer doesn't know the excluded fields exist — they're not returned as `null` or `"[REDACTED]"`, they're absent from the response entirely.

## Why Server-Side Masking Matters

Most RBAC implementations check permissions at the endpoint level: can this user access this table? Yeti goes further — can this user see this field?

This matters for compliance. HIPAA, SOC 2, and GDPR all have requirements around field-level access to sensitive data. If your auth system only gates at the table level, you need application code to strip sensitive fields. That code needs to be correct in every endpoint that returns the data. One missed endpoint is a compliance violation.

Yeti's approach is structural. The field exclusion is defined once in the role configuration and enforced by the platform on every response, from every transport — REST, GraphQL, SSE, WebSocket, MQTT, MCP. There is no code path that can accidentally return an excluded field.

## Combining Auth with Audit

Add `@audit` to the schema and every authenticated operation is logged:

```graphql
type Employee @table @export @audit {
  ...
}
```

The audit log captures who made the request (from the auth context), what they did, when, and optionally the before/after state of the record. Combined with RBAC, this gives you a complete compliance story: who can access what, who did access what, and exactly what they saw.

Try the [demo-authentication](https://yetirocks.com/developers/demos) to see all three auth methods and field-level RBAC in action.
