# REST & AI Agent API Design Standards

**Status:** Active  
**Version:** 1.0  
**Last updated:** 2026-10-28

This document establishes the architecture and design standards for all internal and external APIs. It aligns core REST principles with Microsoft REST API Guidelines and Azure Architecture Best Practices, optimized for both human developers and autonomous AI tool-calling workflows.

---

## 1. Resource-Oriented Design & URI Structure

* **Focus on Resources:** Structure paths around lowercase plural nouns representing entities (e.g., `/users`, `/orders`), never actions or RPC verbs (avoid `/getUsers`, `/createOrder`).
* **Hierarchy Depth:** Limit nested resource paths to a maximum of two levels:
  ```http
  /{collection}/{id}/{sub-resource}
  # Example:
  /customers/{customerId}/orders
  ```
  For relationships deeper than two levels, use query filtering on the top-level collection instead:
  ```http
  /order-items?orderId={id}
  ```
* **Non-Resource Operations (RPC Actions):** For operations that are strictly transactional, computational, or workflow-driven (e.g., triggering an AI evaluation or running a bulk calculation), expose a sub-resource `POST` action:
  ```http
  POST /evaluations/{id}/run-assessment
  POST /documents/{id}/vectorize
  ```

---

## 2. HTTP Verbs & Semantics

Adhere strictly to standard HTTP method semantics:

* `GET`: Read-only, safe, and idempotent. Must never mutate server state.
* `POST`: Create a new resource or execute an asynchronous task/action. Non-idempotent by default.
* `PUT`: Complete resource replacement. Idempotent.
* `PATCH`: Partial resource update. Must support `application/merge-patch+json` or `application/json-patch+json`.
* `DELETE`: Remove a resource. Idempotent (subsequent requests return `204 No Content` or `404 Not Found`).

---

## 3. Asynchronous Long-Running Operations (LRO)

Any operation expected to take more than 2 seconds (common in LLM synthesis, multi-step agent tool calling, file vectorization, or batch reporting) must not hold a synchronous HTTP connection open.

1. **Initial Response:** Return `202 Accepted` immediately.
2. **Monitoring Headers:**
   * `Location` or `Operation-Location`: URL to poll for job progress (e.g., `/operations/{operationId}`).
   * `Retry-After`: Recommended polling interval in seconds.
3. **Status Polling Contract:** The polling endpoint must return a structured lifecycle object:
   ```json
   {
     "operationId": "d3b07384-d113-4f44-9329-dc6d3e7d1234",
     "status": "running", // "notStarted" | "running" | "succeeded" | "failed"
     "createdDateTime": "2026-10-28T15:30:00Z",
     "percentComplete": 65,
     "resourceLocation": "https://api.example.com/v1/reports/rep_98765"
   }
   ```

---

## 4. Query Semantics & Agent Context Optimization

To minimize payload latency and avoid saturating AI model context windows, follow standardized query conventions:

* **Field Projection (`$select`):** Allow clients and AI agents to request only necessary attributes:
  ```http
  GET /orders?status=active&$select=id,status,totalAmount
  ```
* **Filtering & Sorting:** Use standardized query keys:
  * `$filter`: Logical filter expressions (e.g., `$filter=status eq 'Active' and totalAmount gt 100`).
  * `$orderby`: Field sort direction (e.g., `$orderby=createdDateTime desc`).
* **Continuation-Token Pagination:** Offset-based pagination (`?page=2`) is prohibited on large or fast-changing datasets. Use cursor-based continuation tokens:
  ```json
  {
    "value": [
      { "id": "ord_123", "status": "shipped" }
    ],
    "@nextLink": "https://api.example.com/v1/orders?continuationToken=eyJzZWFyY2hBZnRlciI6MTAwfQ=="
  }
  ```

---

## 5. Strict Data Types & Formatting

* **JSON Field Casing:** Use standard `camelCase` for all JSON properties and query parameters (except OData directives like `$select`, `$filter`).
* **Date & Time:** Always RFC 3339 / ISO 8601 formatted in UTC with an explicit `Z` suffix:
  `2026-10-28T15:30:00Z`.
* **Enums:** Represent enums as human-readable string values (e.g., `"inReview"`, `"completed"`), never raw numeric IDs.
* **Null Handling:** Omit optional or unpopulated keys from JSON payloads rather than sending `"property": null`, unless explicitly unsetting a value via `PATCH`.

---

## 6. Standardized Error Contract

All `4xx` and `5xx` responses must follow a predictable error envelope (Microsoft REST API Guidelines format). This allows automated agents and client libraries to parse failures and self-correct inputs.

```json
{
  "error": {
    "code": "InvalidInputParameter",
    "message": "The request payload failed validation against the target schema.",
    "target": "taxRate",
    "details": [
      {
        "code": "ValueOutOfRange",
        "target": "taxRate",
        "message": "The taxRate field must be a decimal between 0.00 and 1.00."
      }
    ],
    "innererror": {
      "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-00",
      "timestamp": "2026-10-28T15:30:00Z"
    }
  }
}
```

### Standard Status Codes
* `200 OK`: Successful read or update.
* `201 Created`: Successful creation (must include a `Location` header to the new resource).
* `202 Accepted`: Long-running task started.
* `204 No Content`: Successful request with no body returned.
* `400 Bad Request`: General malformed syntax.
* `401 Unauthorized`: Missing or invalid authentication token.
* `403 Forbidden`: Authenticated, but lacks required permissions.
* `404 Not Found`: Resource does not exist.
* `409 Conflict`: Resource state conflict (e.g., duplicate unique constraint).
* `422 Unprocessable Entity`: Syntactically correct JSON that violates domain rules.
* `429 Too Many Requests`: Rate limit hit (must include `Retry-After` header).
* `500 Internal Server Error`: Unhandled server-side fault.

---

## 7. Headers, Tracing, and Idempotency

* `client-request-id`: A caller-generated UUID transmitted on every request to track distributed tool executions and agent workflows across service boundaries.
* `traceparent`: Standard W3C distributed tracing identifier header.
* `Idempotency-Key`: Required for state-changing `POST` requests. Servers must cache idempotent request responses for at least 24 hours to prevent duplicate side effects during automatic network retries or agent re-prompts.
* `ETag` and `If-Match`: Use for optimistic concurrency control on `PUT` and `PATCH` updates to prevent mid-air collisions.

---

## 8. Versioning & Evolution

* **URI Path Versioning:** Apply major version indicators in the URL path (`/v1`, `/v2`).
* **Non-Breaking Evolution:** Field additions and optional parameter introductions must be additive:
  * **Tolerant Reader:** Clients and AI tools must be configured to ignore unknown fields.
  * Never rename or remove a property within an active API version.
* **Deprecation Notice:** Deprecated endpoints must return the standard `Sunset` and `Deprecation` HTTP headers before sunsetting.

---

## 9. OpenAPI Specification (OAS 3.1) for Agent Tool Calling

To ensure high reliability when endpoints are consumed as AI agent tools (such as OpenAI Function Calling or Model Context Protocol / MCP tools):

* **Operation IDs:** Every operation must declare an explicit, camelCase `operationId` (e.g., `getOrderById`, `cancelSubscription`).
* **Explicit Descriptions:** Every endpoint, parameter, and object property must include concise, unambiguous `description` metadata detailing expected units, formats, and domain boundaries.
* **Strict Schemas:** Always explicitly define `required` arrays and constraints (`minimum`, `maximum`, `pattern`, `enum`).

---

## 10. Interactive Documentation & Testing UI (Scalar)

All services exposing REST endpoints or agent tools must provide interactive reference documentation powered by **Scalar** (permissive MIT License, free for commercial use).

* **Standard UI Route:** The Scalar reference interface must be hosted at `/scalar/v1` or `/docs`.
* **Specification Source:** Scalar must bind directly to the OpenAPI 3.1 specification exposed at `/openapi/v1.json`.
* **Feature Requirements:**
  * Interactive API Client enabled for sandbox/development environments.
  * Multi-language code snippets enabled (cURL, Python, TypeScript, C#).
  * Dark and light theme support adhering to company branding.
# REST & AI Agent API Design Standards

**Status:** Active  
**Version:** 1.0  
**Last updated:** 2026-10-28

This document establishes the architecture and design standards for all internal and external APIs. It aligns core REST principles with Microsoft REST API Guidelines and Azure Architecture Best Practices, optimized for both human developers and autonomous AI tool-calling workflows.

---

## 1. Resource-Oriented Design & URI Structure

* **Focus on Resources:** Structure paths around lowercase plural nouns representing entities (e.g., `/users`, `/orders`), never actions or RPC verbs (avoid `/getUsers`, `/createOrder`).
* **Hierarchy Depth:** Limit nested resource paths to a maximum of two levels:
  ```http
  /{collection}/{id}/{sub-resource}
  # Example:
  /customers/{customerId}/orders
  ```
  For relationships deeper than two levels, use query filtering on the top-level collection instead:
  ```http
  /order-items?orderId={id}
  ```
* **Non-Resource Operations (RPC Actions):** For operations that are strictly transactional, computational, or workflow-driven (e.g., triggering an AI evaluation or running a bulk calculation), expose a sub-resource `POST` action:
  ```http
  POST /evaluations/{id}/run-assessment
  POST /documents/{id}/vectorize
  ```

---

## 2. HTTP Verbs & Semantics

Adhere strictly to standard HTTP method semantics:

* `GET`: Read-only, safe, and idempotent. Must never mutate server state.
* `POST`: Create a new resource or execute an asynchronous task/action. Non-idempotent by default.
* `PUT`: Complete resource replacement. Idempotent.
* `PATCH`: Partial resource update. Must support `application/merge-patch+json` or `application/json-patch+json`.
* `DELETE`: Remove a resource. Idempotent (subsequent requests return `204 No Content` or `404 Not Found`).

---

## 3. Asynchronous Long-Running Operations (LRO)

Any operation expected to take more than 2 seconds (common in LLM synthesis, multi-step agent tool calling, file vectorization, or batch reporting) must not hold a synchronous HTTP connection open.

1. **Initial Response:** Return `202 Accepted` immediately.
2. **Monitoring Headers:**
   * `Location` or `Operation-Location`: URL to poll for job progress (e.g., `/operations/{operationId}`).
   * `Retry-After`: Recommended polling interval in seconds.
3. **Status Polling Contract:** The polling endpoint must return a structured lifecycle object:
   ```json
   {
     "operationId": "d3b07384-d113-4f44-9329-dc6d3e7d1234",
     "status": "running", // "notStarted" | "running" | "succeeded" | "failed"
     "createdDateTime": "2026-10-28T15:30:00Z",
     "percentComplete": 65,
     "resourceLocation": "https://api.example.com/v1/reports/rep_98765"
   }
   ```

---

## 4. Query Semantics & Agent Context Optimization

To minimize payload latency and avoid saturating AI model context windows, follow standardized query conventions:

* **Field Projection (`$select`):** Allow clients and AI agents to request only necessary attributes:
  ```http
  GET /orders?status=active&$select=id,status,totalAmount
  ```
* **Filtering & Sorting:** Use standardized query keys:
  * `$filter`: Logical filter expressions (e.g., `$filter=status eq 'Active' and totalAmount gt 100`).
  * `$orderby`: Field sort direction (e.g., `$orderby=createdDateTime desc`).
* **Continuation-Token Pagination:** Offset-based pagination (`?page=2`) is prohibited on large or fast-changing datasets. Use cursor-based continuation tokens:
  ```json
  {
    "value": [
      { "id": "ord_123", "status": "shipped" }
    ],
    "@nextLink": "https://api.example.com/v1/orders?continuationToken=eyJzZWFyY2hBZnRlciI6MTAwfQ=="
  }
  ```

---

## 5. Strict Data Types & Formatting

* **JSON Field Casing:** Use standard `camelCase` for all JSON properties and query parameters (except OData directives like `$select`, `$filter`).
* **Date & Time:** Always RFC 3339 / ISO 8601 formatted in UTC with a explicit `Z` suffix:
  `2026-10-28T15:30:00Z`.
* **Enums:** Represent enums as human-readable string values (e.g., `"inReview"`, `"completed"`), never raw numeric IDs.
* **Null Handling:** Omit optional or unpopulated keys from JSON payloads rather than sending `"property": null`, unless explicitly unsetting a value via `PATCH`.

---

## 6. Standardized Error Contract

All `4xx` and `5xx` responses must follow a predictable error envelope (RFC 7807 / Microsoft API format). This allows automated agents and client libraries to parse failures and self-correct inputs.

```json
{
  "error": {
    "code": "InvalidInputParameter",
    "message": "The request payload failed validation against the target schema.",
    "target": "taxRate",
    "details": [
      {
        "code": "ValueOutOfRange",
        "target": "taxRate",
        "message": "The taxRate field must be a decimal between 0.00 and 1.00."
      }
    ],
    "innererror": {
      "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-00",
      "timestamp": "2026-10-28T15:30:00Z"
    }
  }
}
```

### Standard Status Codes
* `200 OK`: Successful read or update.
* `201 Created`: Successful creation (must include a `Location` header to the new resource).
* `202 Accepted`: Long-running task started.
* `204 No Content`: Successful request with no body returned.
* `400 Bad Request`: General malformed syntax.
* `401 Unauthorized`: Missing or invalid authentication token.
* `403 Forbidden`: Authenticated, but lacks required permissions.
* `404 Not Found`: Resource does not exist.
* `409 Conflict`: Resource state conflict (e.g., duplicate unique constraint).
* `422 Unprocessable Entity`: Syntactically correct JSON that violates domain rules.
* `429 Too Many Requests`: Rate limit hit (must include `Retry-After` header).
* `500 Internal Server Error`: Unhandled server-side fault.

---

## 7. Headers, Tracing, and Idempotency

* `client-request-id`: A caller-generated UUID transmitted on every request to track distributed tool executions and agent workflows across service boundaries.
* `traceparent`: Standard W3C distributed tracing identifier header.
* `Idempotency-Key`: Required for state-changing `POST` requests. Servers must cache idempotent request responses for at least 24 hours to prevent duplicate side effects during automatic network retries or agent re-prompts.
* `ETag` and `If-Match`: Use for optimistic concurrency control on `PUT` and `PATCH` updates to prevent mid-air collisions.

---

## 8. Versioning & Evolution

* **URI Path Versioning:** Apply major version indicators in the URL path (`/v1`, `/v2`).
* **Non-Breaking Evolution:** Field additions and optional parameter introductions must be additive:
  * **Tolerant Reader:** Clients and AI tools must be configured to ignore unknown fields.
  * Never rename or remove a property within an active API version.
* **Deprecation Notice:** Deprecated endpoints must return the standard `Sunset` and `Deprecation` HTTP headers before sunsetting.

---

## 9. OpenAPI Specification (OAS 3.1) for Agent Tool Calling

To ensure high reliability when endpoints are consumed as AI agent tools (such as OpenAI Function Calling or Model Context Protocol / MCP tools):

* **Operation IDs:** Every operation must declare an explicit, camelCase `operationId` (e.g., `getOrderById`, `cancelSubscription`).
* **Explicit Descriptions:** Every endpoint, parameter, and object property must include concise, unambiguous `description` metadata detailing expected units, formats, and domain boundaries.
* **Strict Schemas:** Always explicitly define `required` arrays and constraints (`minimum`, `maximum`, `pattern`, `enum`).

---

## 10. Interactive Documentation & Testing UI (Scalar)

All services exposing REST endpoints or agent tools must provide interactive reference documentation powered by **Scalar** (permissive MIT License, free for commercial use).

* **Standard UI Route:** The Scalar reference interface must be hosted at `/scalar/v1` or `/docs`.
* **Specification Source:** Scalar must bind directly to the OpenAPI 3.1 specification exposed at `/openapi/v1.json`.
* **Feature Requirements:**
  * Interactive API Client enabled for sandbox/development environments.
  * Multi-language code snippets enabled (cURL, Python, TypeScript, C#).
  * Dark and light theme support adhering to company branding.
