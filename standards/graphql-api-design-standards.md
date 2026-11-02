# GraphQL & AI Agent API Design Standards

**Status:** Draft  
**Version:** 1.0  
**Last updated:** 2026-11-02

This document establishes the architecture and design standards for all internal and external GraphQL APIs. It conforms to the [GraphQL Specification](https://spec.graphql.org/draft/) and the [GraphQL over HTTP](https://graphql.github.io/graphql-over-http/draft/) draft, follows the [GraphQL.org Best Practices](https://graphql.org/learn/best-practices/), and is optimized for both human developers and autonomous AI tool-calling workflows.

It is the companion to the [REST & AI Agent API Design Standards](api-design-standards.md). Where the two differ, the difference is called out and explained. Where this standard and the GraphQL Specification conflict, the specification wins.

---

## 1. When to Use GraphQL vs. REST

Choose deliberately and record the decision in the solution architecture.

* **Prefer GraphQL** when many clients need different shapes of the same data, when UIs aggregate data from several back-end systems, or when a single unified graph across domains is wanted (federation).
* **Prefer REST** for simple resource-oriented CRUD, external partner integrations that expect OpenAPI, file transfer, webhooks, and machine-to-machine calls where HTTP caching matters.
* **Model the Domain, Not the Database:** A GraphQL schema must describe the business domain as a graph. It must never be a thin pass-through that mirrors database tables or REST endpoints one-to-one.
* **Shared Logic:** If the same capability is exposed through both REST and GraphQL, both must call the same business logic layer.

---

## 2. Endpoint & Transport (GraphQL over HTTP)

* **Single Endpoint:** Each service exposes one endpoint ending in `/graphql`. Entities are identified by `ID`, not by URL.
* **Methods:**
  * `POST`: Must be supported for queries and mutations.
  * `GET`: May be supported for **query** operations only (for example, to allow CDN caching of persisted operations). Mutations must never be executed over `GET`.
* **Request Headers:**
  * `Content-Type: application/json` is required on `POST`. A request without it must be rejected with a `4xx`.
  * `Accept: application/graphql-response+json` must be sent by clients (add `, application/json` when calling legacy servers).
* **Request Body:**
  ```json
  {
    "operationName": "GetCustomer",
    "query": "query GetCustomer($id: ID!) { customer(id: $id) { id name } }",
    "variables": { "id": "Q3VzdG9tZXI6MTIz" },
    "extensions": {}
  }
  ```
* **Response Body:** A JSON object with `data`, `errors` and `extensions`. The `errors` key must be omitted when there are no errors. When a request fails before execution (parse or validation error), `data` must be omitted.

### Standard Status Codes

Status codes follow GraphQL over HTTP with the `application/graphql-response+json` media type:

* `200 OK`: Operation executed and `data` is non-null, **even if `errors` is present** (partial success).
* `400 Bad Request`: Malformed JSON, missing `Content-Type`, parse or validation failure, or query too complex.
* `401 Unauthorized`: Missing or invalid authentication token.
* `406 Not Acceptable`: Client does not accept a supported media type.
* `429 Too Many Requests`: Request or cost rate limit hit (must include `Retry-After` header).
* `500` / `503` / `504`: The operation could not execute and `data` is `null`.

> **Difference from REST:** A field the caller may not see returns `200` with partial `data` and a `FORBIDDEN` entry in `errors`, not `403`. Business-rule failures return `200` with typed errors in the payload (section 8), not `409` or `422`.

---

## 3. Schema Design & Naming

* **Schema-First:** The SDL (`schema.graphql`) is the source of truth and must live in version control. Code-first frameworks are allowed only if the generated SDL is committed and reviewed.
* **Specification Naming Rules:** Names must match `[_A-Za-z][_0-9A-Za-z]*` (Latin ASCII only), are case-sensitive, and must never start with `__` (reserved for introspection).
* **Naming Conventions:**

  | Element | Convention | Example |
  |---|---|---|
  | Types, interfaces, unions, inputs, enums | `PascalCase` | `Customer`, `CreateOrderInput` |
  | Fields and arguments | `camelCase` | `createdDateTime`, `totalAmount` |
  | Enum values | `SCREAMING_SNAKE_CASE` | `IN_REVIEW` |
  | Queries | Noun (singular for one, plural for many) | `customer`, `customers` |
  | Mutations | `verbNoun` | `createOrder`, `cancelSubscription` |
  | Subscriptions | `nounPastTenseVerb` | `orderUpdated` |
  | Mutation input / payload | `<MutationName>Input` / `<MutationName>Payload` | `CreateOrderInput` |
  | Custom directives | `tkw_` prefix | `@tkw_requiresScope` |
  | Client operation names | `PascalCase`, unique per client | `GetCustomerOrders` |

  > **Difference from REST:** REST enum values are `camelCase` strings (`"inReview"`). GraphQL enum values are `SCREAMING_SNAKE_CASE` (`IN_REVIEW`), as recommended by the specification.

* **No Leaked Internals:** Database keys, table names and internal service names must not appear in the schema.
* **Global Identity:** Entities should implement the `Node` interface with an opaque, globally unique `ID`, and the schema should expose `node(id: ID!)` and `nodes(ids: [ID!]!)` root fields. IDs must be stable for the life of the entity.
* **Group Related Fields:** Use nested object types (`address: Address`) rather than flat prefixed fields (`addressLine1`, `addressCity`).

```graphql
"""
A person or organization that purchases services.
"""
type Customer implements Node {
  id: ID!
  "Display name of the customer."
  name: String!
  "Primary postal address, if known."
  address: Address
  "Orders placed by the customer, most recent first."
  orders(first: Int = 20, after: String): OrderConnection!
  "When the customer record was created (UTC)."
  createdDateTime: DateTime!
}
```

---

## 4. Strict Data Types & Formatting

* **Built-in Scalars:** `Int` is a signed **32-bit** integer only. Larger values must use the `BigInt` custom scalar. `Float` must never be used for money or exact decimals.
* **Standard Custom Scalars:** Use these instead of plain `String`. Each must declare `@specifiedBy(url:)` pointing to a stable specification, and that URL must never change once published.

  | Scalar | Format | Example |
  |---|---|---|
  | `DateTime` | RFC 3339 / ISO 8601 in UTC with `Z` suffix (same as REST) | `2026-11-02T15:30:00Z` |
  | `Date` | ISO 8601 calendar date | `2026-11-02` |
  | `UUID` | RFC 9562 | `d3b07384-d113-4f44-9329-dc6d3e7d1234` |
  | `Decimal` | Arbitrary-precision number serialized as a string | `"1299.95"` |
  | `BigInt` | Integer beyond 32 bits serialized as a string | `"9007199254740993"` |
  | `Email`, `URL` | RFC 5322 address / absolute RFC 3986 URL | `jane@example.com` |
  | `CurrencyCode`, `CountryCode` | ISO 4217 / ISO 3166-1 alpha-2 | `CAD`, `CA` |

  ```graphql
  "A UTC date-time, e.g. `2026-11-02T15:30:00Z`."
  scalar DateTime @specifiedBy(url: "https://scalars.graphql.org/andimarek/date-time")
  ```

* **Money:** Use a `Money` object (`amount: Decimal!`, `currency: CurrencyCode!`).
* **No Untyped JSON:** A `JSON` scalar should not be used. If it is, the reason must be stated in the field description.
* **Nullability:** All types are nullable by default; `!` makes them non-null.
  * An error on a non-null field nulls out its parent, up to the nearest nullable field. Make fields backed by remote services nullable so one failure doesn't wipe out the response.
  * Use non-null only when a value is guaranteed (`id`, `createdDateTime`).
  * Lists should be `[Item!]!`: the list always exists and holds no null items.
* **Null vs. Omitted Input:** The specification treats an input set to `null` differently from one left out. Update mutations must implement **omitted = leave unchanged, `null` = clear the value**, matching the REST `PATCH` rule.
* **One-of Inputs:** Use `@oneOf` when the caller must supply exactly one of several alternatives:
  ```graphql
  "Identify a customer by exactly one key."
  input CustomerLookup @oneOf {
    id: ID
    email: Email
    accountNumber: String
  }

  type Query {
    customer(by: CustomerLookup!): Customer
  }
  ```

---

## 5. Query Semantics & Agent Context Optimization

GraphQL's field selection is the built-in equivalent of REST `$select`: callers and AI agents request only the fields they need.

* **Read-Only Queries:** Queries must never mutate server state.
* **Not Found:** Single-entity lookups return a nullable type and return `null` when nothing is found, rather than an error.
* **Named Operations:** Clients must send named operations. Anonymous operations should be rejected in production, because operation names drive tracing and usage metrics.
* **Filtering & Sorting:** Use a dedicated input type per list, with explicit fields rather than a free-form expression string (the GraphQL equivalent of `$filter` and `$orderby`):
  ```graphql
  input OrderFilter {
    status: [OrderStatus!]
    createdAfter: DateTime
    createdBefore: DateTime
  }

  input OrderSort {
    field: OrderSortField!
    direction: SortDirection! = DESC
  }
  ```
* **Cursor Pagination:** Offset pagination is prohibited. Lists that can return more than about 50 items must use [Relay Cursor Connections](https://relay.dev/graphql/connections.htm) (the GraphQL equivalent of `@nextLink`):
  ```graphql
  type Query {
    orders(first: Int = 20, after: String, filter: OrderFilter, sort: [OrderSort!]): OrderConnection!
  }

  type OrderConnection {
    edges: [OrderEdge!]!
    nodes: [Order!]!
    pageInfo: PageInfo!
  }

  type PageInfo {
    hasNextPage: Boolean!
    endCursor: String
  }
  ```
  * The default page size is 20 and the maximum is 100.
  * Cursors are opaque; clients must never build or parse them.

---

## 6. Mutations & Idempotency

* **Business Actions:** Model mutations as business actions (`approveInvoice`, `cancelSubscription`), not generic updates (`updateInvoice(status: APPROVED)`). This is the GraphQL equivalent of the REST sub-resource `POST` action.
* **Single Input, Dedicated Payload:** Each mutation takes one argument named `input` and returns its own `<MutationName>Payload` type, even if it holds only one field today. Payloads should return the affected entity so clients can update their cache.
* **Idempotency:** State-changing mutations that may be retried must accept an `idempotencyKey` input field. As in the REST standard, servers must keep the result for at least 24 hours and return it for repeated calls with the same key.
* **Optimistic Concurrency:** Where REST uses `ETag` / `If-Match`, mutations accept an `expectedVersion` (or `etag`) input and return a `CONFLICT` error when it no longer matches.
* **Batch Mutations:** These may be offered, but each must say whether it is atomic.
* **File Uploads:** Use a pre-signed URL flow (a mutation returns an upload URL) rather than multipart GraphQL requests.

```graphql
input CreateOrderInput {
  "Caller-generated key. Repeated calls with the same key return the original result."
  idempotencyKey: String!
  customerId: ID!
  lines: [OrderLineInput!]!
}

type CreateOrderPayload {
  "The created order, or null if `userErrors` is not empty."
  order: Order
  userErrors: [CreateOrderError!]!
}

type Mutation {
  createOrder(input: CreateOrderInput!): CreateOrderPayload!
}
```

---

## 7. Asynchronous Long-Running Operations (LRO)

The REST 2-second rule applies here too. Any mutation expected to take more than 2 seconds must not keep the request open.

1. **Initial Response:** The mutation returns immediately with an `Operation` object.
2. **Monitoring:** Callers poll it with `node(id:)` (or `operation(id:)`), or subscribe to `operationUpdated` for live progress.
3. **Status Contract:** Same lifecycle as the REST polling contract:
   ```graphql
   enum OperationStatus { NOT_STARTED RUNNING SUCCEEDED FAILED }

   type Operation implements Node {
     id: ID!
     status: OperationStatus!
     createdDateTime: DateTime!
     percentComplete: Int
     "Suggested polling interval in seconds."
     retryAfterSeconds: Int
     "The resulting resource once status is SUCCEEDED."
     result: Node
   }

   type Mutation {
     runAssessment(input: RunAssessmentInput!): RunAssessmentPayload!
   }

   type RunAssessmentPayload {
     operation: Operation
     userErrors: [RunAssessmentError!]!
   }
   ```

---

## 8. Standardized Error Contract

GraphQL has two error channels. Use each for its purpose.

| Error type | Channel | Example |
|---|---|---|
| Infrastructure failure | Top-level `errors` | Database timeout |
| Invalid GraphQL | Top-level `errors` | Syntax error, unknown field |
| Authentication or field authorization | Top-level `errors` | Expired token, not allowed to read `Customer.taxId` |
| Business rule or input validation | Errors as data in the payload | Duplicate email, insufficient stock |

### Top-Level Errors

Error objects follow the specification: `message`, `locations`, `path` and `extensions`. All extra data goes inside `extensions`; custom top-level keys on an error object are not allowed. Every error must include `extensions.code`, and should include `traceId` and `timestamp`, matching the REST `innererror`.

```json
{
  "data": { "customer": { "id": "Q3VzdG9tZXI6MTIz", "taxId": null } },
  "errors": [
    {
      "message": "You are not authorized to view this field.",
      "path": ["customer", "taxId"],
      "locations": [{ "line": 4, "column": 5 }],
      "extensions": {
        "code": "FORBIDDEN",
        "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-00",
        "timestamp": "2026-11-02T15:30:00Z"
      }
    }
  ]
}
```

Standard `extensions.code` values:

| Code | Meaning | HTTP status |
|---|---|---|
| `GRAPHQL_PARSE_FAILED` | Document syntax is invalid | `400` |
| `GRAPHQL_VALIDATION_FAILED` | Operation does not validate against the schema | `400` |
| `BAD_USER_INPUT` | A variable value cannot be coerced to its type | `400` |
| `QUERY_TOO_COMPLEX` | Depth, cost or alias limit exceeded | `400` |
| `UNAUTHENTICATED` | Missing or invalid credentials | `401` |
| `FORBIDDEN` | Authenticated, but not allowed to see this field | `200` (partial data) |
| `RATE_LIMITED` | Request or cost rate limit hit | `429` |
| `INTERNAL_ERROR` | Unhandled server-side fault | `200` with partial data, otherwise `500` |

Error messages must never expose stack traces, SQL, hostnames or secrets in production.

### Errors as Data

Business and validation failures are returned in the mutation payload with an **enum** error code, so callers and AI agents can discover every possible error through the schema and self-correct their input. `field` plays the same role as `target` in the REST error envelope.

```graphql
type CreateOrderError {
  code: CreateOrderErrorCode!
  "Human-readable message, safe to display."
  message: String!
  "Path to the input field at fault, e.g. [\"lines\", \"0\", \"quantity\"]."
  field: [String!]
}

enum CreateOrderErrorCode {
  VALIDATION_FAILED
  CUSTOMER_NOT_FOUND
  INSUFFICIENT_STOCK
  DUPLICATE
  CONFLICT
}
```

---

## 9. Subscriptions

* **Use Sparingly:** Subscriptions should be used only for genuine real-time needs (live status, notifications, LRO progress). Otherwise use polling or webhooks.
* **One Root Field:** Each subscription operation has exactly one root field (specification rule).
* **Transport:** Use `graphql-ws` (the `graphql-transport-ws` protocol) or Server-Sent Events. The legacy `subscriptions-transport-ws` must not be used.
* **Security:** Authenticate when the connection starts and re-check authorization for each event. Limit concurrent subscriptions per client and close idle connections.

---

## 10. Security

* **Authentication First:** Authentication runs in middleware before the GraphQL request executes, using OAuth 2.0 / OIDC bearer tokens (Microsoft Entra ID for internal APIs).
* **Field-Level Authorization:** Authorization must be enforced in the business logic called by resolvers, at object and field level. It must never rely only on the gateway or on the client not asking for a field.
* **Operation Limits:** Every production API must enforce:

  | Control | Default |
  |---|---|
  | Maximum query depth | 10 |
  | Maximum query cost | Defined per API and documented |
  | Maximum aliases per operation | 15 |
  | Maximum operations per batch | 5, or batching disabled |
  | Maximum request size | 100 KB |
  | Execution timeout | 10 seconds |
  | Rate limiting | Per client and by query cost |

* **Introspection:** Must be disabled in production for external APIs. Internal APIs may allow it for authenticated developers. Field suggestions ("Did you mean…") should be disabled in production.
* **Trusted Operations:** Public-facing first-party clients (web, mobile) must use persisted (allow-listed) operations.
* **Input Validation:** Validate lengths, ranges and formats on the server. Scalar types alone are not enough.
* **Logging:** Never log variable values that contain personal or secret data.

---

## 11. Headers, Tracing & Observability

* `client-request-id`: Same as REST. A caller-generated UUID sent on every request.
* `traceparent`: Standard W3C distributed tracing header, propagated to every downstream call. Tracing uses OpenTelemetry.
* `x-client-name` / `x-client-version`: Identify the calling app or agent so usage can be tracked per client.
* **What to Log:** Every request logs the operation name, client, duration, error codes and trace ID.
* **Field Usage Metrics:** Usage must be recorded per field and per client, so deprecated fields can be removed safely.
* **N+1 Prevention:** Resolvers that load related data must batch and cache per request (the DataLoader pattern). Every downstream call needs a timeout and circuit breaker.

---

## 12. Versioning & Evolution

> **Difference from REST:** REST uses URI path versions (`/v1`, `/v2`). GraphQL schemas evolve continuously without versions. A versioned endpoint (`/graphql/v2`) needs Architecture Review Board approval.

* **Safe Changes:** Adding types, fields, queries or mutations; adding optional arguments whose default keeps the old behaviour; making a required input optional.
* **Dangerous Changes:** Adding an enum value, adding a union member or interface implementation, or changing a default value. These are allowed but must be announced, and clients and AI tools must be **tolerant readers** that handle unknown enum values and types.
* **Breaking Changes:** Removing or renaming anything, changing a type, making a non-null output nullable, or making an optional input required. These need the deprecation process below. CI must check every schema change against the production schema (for example with GraphQL Inspector) and block breaking changes.
* **Deprecation Process:**
  1. Add the replacement first.
  2. Mark the old element with `@deprecated(reason: "Use `newField`. Removal after 2027-05-01.")`. The reason must name the replacement and the removal date. A required argument or input field must first be made optional before it can be deprecated.
  3. Keep the deprecated element fully working.
  4. Announce it in the change log and to consuming teams.
  5. Track usage per client for at least 30 days, and remove it only after the deprecation window has passed and usage has stopped.
* **Change Log:** Each API keeps a dated change log with **Added / Deprecated / Removed / Changed** sections, naming schema elements by their path (for example `Customer.name`, `OrderStatus.SHIPPED`).

---

## 13. Schema Descriptions for Agent Tool Calling

The schema is the tool contract when GraphQL operations are consumed as AI agent tools (such as OpenAI Function Calling or Model Context Protocol / MCP tools).

* **Explicit Descriptions:** Every type, field, argument, enum value, input field and custom directive must have a description detailing expected units, formats and domain boundaries. Descriptions are Markdown (CommonMark).
* **Strict Schemas:** Use non-null, enums, `@oneOf` and custom scalars to express constraints in the type system rather than in prose.
* **Operations as Tools:** Expose agent tools as named, persisted operations with camelCase names (for example `getOrderById`, `cancelSubscription`), the equivalent of REST `operationId`s. Agents should call these, not compose arbitrary queries.
* **Custom Directives:** Prefix them `tkw_` (for example `@tkw_requiresScope`) so they never collide with future built-in directives.

---

## 14. Interactive Documentation & Testing UI

* **Schema Source:** The SDL with full descriptions is the primary documentation and must be published to the internal schema registry.
* **Explorer:** Non-production environments must host **GraphiQL** (MIT License) at `/graphiql`. It must not be exposed in production for external APIs.
* **README:** Each API documents its endpoints per environment, authentication, operation limits, custom directives and business error codes.
* **Federation:** In a multi-team graph, each subgraph has one owning team, each entity type has one owning subgraph, clients call only the gateway, and composition checks must pass in CI before deployment.

---

## 15. Microsoft Fabric API for GraphQL

These rules apply when GraphQL is served by [Microsoft Fabric API for GraphQL](https://blog.fabric.microsoft.com/blog/best-practices-for-fabric-api-for-graphql) over a Fabric Data Warehouse, Lakehouse or Mirrored Database.

* **Authentication:** Callers must authenticate with Microsoft Entra ID through a registered application (service principal). The app registration and its permissions must be set up before any calls are made.
* **Expose Deliberately:** Expose only the tables, views and stored procedures that consumers need. Prefer views over raw tables so the schema reflects the business domain (section 1) and is not tied to the physical model.
* **Mutations via Stored Procedures:** Where business logic or validation is needed on write, expose a stored procedure rather than direct table mutations.
* **Avoid Deep Nesting:** Keep queries shallow. Deeply nested queries are harder to debug and add overhead in Fabric.
* **Avoid Over- and Under-Fetching:** Request only the fields needed, and combine related reads into one operation instead of many small requests. Batch large data access.
* **Plan Schema Changes:** Changes to the underlying warehouse or lakehouse change the generated GraphQL schema. Run them through the evolution rules in section 12 and check the effect on query performance.
* **Monitor & Test:** Monitor query performance for slowdowns, and test queries and mutations regularly, including edge cases.
* **Generated Schema Deviations:** Fabric generates the schema, so some rules in this standard (for example custom scalars, payload types or enum-coded errors) may not be achievable. Each deviation must be recorded in the API README with the reason.
