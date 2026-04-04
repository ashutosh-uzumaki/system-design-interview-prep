# REST API Design: Complete Guide (Beginner to Advanced)

---

## Table of Contents

1. [Introduction to REST](#1-introduction-to-rest)
2. [Core Principles of REST](#2-core-principles-of-rest)
3. [HTTP Methods](#3-http-methods)
4. [URL & Resource Naming](#4-url--resource-naming)
5. [HTTP Status Codes](#5-http-status-codes)
6. [Request & Response Design](#6-request--response-design)
7. [Filtering, Sorting & Pagination](#7-filtering-sorting--pagination)
8. [Authentication & Authorization](#8-authentication--authorization)
9. [Versioning](#9-versioning)
10. [Error Handling](#10-error-handling)
11. [HATEOAS](#11-hateoas)
12. [Rate Limiting & Throttling](#12-rate-limiting--throttling)
13. [Caching](#13-caching)
14. [Idempotency](#14-idempotency)
15. [API Security Best Practices](#15-api-security-best-practices)
16. [Advanced Patterns](#16-advanced-patterns)
17. [Practice Questions & Answers](#17-practice-questions--answers)

---

## 1. Introduction to REST

**REST** (Representational State Transfer) is an architectural style for designing networked applications. It was introduced by Roy Fielding in his 2000 doctoral dissertation. REST relies on a stateless, client-server communication protocol — almost always HTTP.

### What is a REST API?

A REST API (also called a RESTful API) is an interface that allows two systems to exchange information securely over the internet using HTTP. It treats server-side data as **resources** that can be created, read, updated, or deleted (CRUD).

### Why REST?

- Simple and lightweight
- Stateless communication
- Scalable architecture
- Language and platform agnostic
- Uses standard HTTP methods
- Easy to cache

### REST vs SOAP

| Feature        | REST                     | SOAP                        |
| -------------- | ------------------------ | --------------------------- |
| Protocol       | HTTP                     | HTTP, SMTP, TCP, etc.       |
| Data Format    | JSON, XML, YAML, etc.    | XML only                    |
| Complexity     | Simple                   | Complex (WSDL, envelopes)   |
| Performance    | Faster                   | Slower                      |
| Statefulness   | Stateless                | Can be stateful             |
| Error Handling | HTTP status codes        | SOAP fault element          |

---

## 2. Core Principles of REST

REST is defined by **six architectural constraints**:

### 2.1 Client-Server Separation

The client and server are independent. The client handles the UI/UX, and the server handles data storage and business logic. They communicate only through requests and responses.

### 2.2 Statelessness

Each request from the client must contain **all the information** needed to process it. The server does not store any client context between requests. Session state is kept entirely on the client.

```
# Bad (Stateful) — server remembers previous request
GET /next-page

# Good (Stateless) — everything is in the request
GET /articles?page=3&limit=20
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### 2.3 Cacheability

Responses must define themselves as **cacheable or non-cacheable**. Proper caching eliminates some client-server interactions and improves performance.

### 2.4 Uniform Interface

This is the most fundamental constraint. All API resources should be accessible through a consistent, standardized interface using:

- Resource identification via URIs
- Resource manipulation through representations
- Self-descriptive messages
- Hypermedia as the engine of application state (HATEOAS)

### 2.5 Layered System

The architecture can be composed of multiple layers (load balancers, proxies, gateways). The client does not know whether it's connected directly to the server or through an intermediary.

### 2.6 Code on Demand (Optional)

Servers can send executable code (e.g., JavaScript) to extend client functionality. This is the only optional constraint.

---

## 3. HTTP Methods

HTTP methods define the **action** to perform on a resource.

### 3.1 Method Overview

| Method  | CRUD     | Description                  | Idempotent | Safe |
| ------- | -------- | ---------------------------- | ---------- | ---- |
| GET     | Read     | Retrieve a resource          | Yes        | Yes  |
| POST    | Create   | Create a new resource        | No         | No   |
| PUT     | Update   | Replace a resource entirely  | Yes        | No   |
| PATCH   | Update   | Partially update a resource  | No*        | No   |
| DELETE  | Delete   | Remove a resource            | Yes        | No   |
| HEAD    | Read     | Same as GET but no body      | Yes        | Yes  |
| OPTIONS | Metadata | Get supported methods        | Yes        | Yes  |

> *PATCH can be made idempotent, but is not guaranteed to be by specification.

### 3.2 Examples

#### GET — Retrieve Resources

```http
# Get all users
GET /api/v1/users HTTP/1.1
Host: api.example.com
Accept: application/json

# Response: 200 OK
[
  { "id": 1, "name": "Alice", "email": "alice@example.com" },
  { "id": 2, "name": "Bob", "email": "bob@example.com" }
]
```

```http
# Get a single user
GET /api/v1/users/1 HTTP/1.1

# Response: 200 OK
{ "id": 1, "name": "Alice", "email": "alice@example.com" }
```

#### POST — Create a Resource

```http
POST /api/v1/users HTTP/1.1
Content-Type: application/json

{
  "name": "Charlie",
  "email": "charlie@example.com",
  "password": "securePass123"
}

# Response: 201 Created
# Location: /api/v1/users/3
{
  "id": 3,
  "name": "Charlie",
  "email": "charlie@example.com",
  "createdAt": "2026-03-31T10:00:00Z"
}
```

#### PUT — Replace a Resource Entirely

```http
PUT /api/v1/users/3 HTTP/1.1
Content-Type: application/json

{
  "name": "Charlie Brown",
  "email": "charliebrown@example.com",
  "password": "newSecurePass456"
}

# Response: 200 OK
{
  "id": 3,
  "name": "Charlie Brown",
  "email": "charliebrown@example.com",
  "updatedAt": "2026-03-31T11:00:00Z"
}
```

#### PATCH — Partially Update a Resource

```http
PATCH /api/v1/users/3 HTTP/1.1
Content-Type: application/json

{
  "email": "charlie.new@example.com"
}

# Response: 200 OK
{
  "id": 3,
  "name": "Charlie Brown",
  "email": "charlie.new@example.com",
  "updatedAt": "2026-03-31T12:00:00Z"
}
```

#### DELETE — Remove a Resource

```http
DELETE /api/v1/users/3 HTTP/1.1

# Response: 204 No Content
```

---

## 4. URL & Resource Naming

Good URL design is the backbone of a well-designed REST API.

### 4.1 Golden Rules

1. **Use nouns, not verbs** — resources are things, not actions
2. **Use plural nouns** for collections
3. **Use lowercase letters**
4. **Use hyphens (`-`)** for multi-word names, never underscores
5. **Never expose file extensions** in URLs
6. **Nest resources** to show relationships (but avoid deep nesting)

### 4.2 Examples

```
# GOOD
GET    /api/v1/users
GET    /api/v1/users/42
GET    /api/v1/users/42/orders
GET    /api/v1/users/42/orders/99
POST   /api/v1/users
PUT    /api/v1/users/42
DELETE /api/v1/users/42

# BAD
GET    /api/v1/getUsers          # verb in URL
GET    /api/v1/user/42           # singular noun
POST   /api/v1/createUser        # verb in URL
GET    /api/v1/Users             # uppercase
GET    /api/v1/user_orders       # underscores
DELETE /api/v1/deleteUser/42     # verb in URL
```

### 4.3 Handling Non-CRUD Operations

Sometimes you need actions that don't map cleanly to CRUD. Strategies include:

```
# Option 1: Treat the action as a sub-resource
POST /api/v1/users/42/activate
POST /api/v1/orders/99/cancel
POST /api/v1/emails/42/resend

# Option 2: Use a "controller" resource
POST /api/v1/searches
POST /api/v1/translations

# Option 3: Use query parameters for state changes (less preferred)
PATCH /api/v1/users/42?action=activate
```

### 4.4 Avoid Deep Nesting

```
# BAD — too deeply nested
GET /api/v1/countries/1/states/5/cities/12/neighborhoods/3/houses

# GOOD — flatten with query params or direct resource access
GET /api/v1/houses?neighborhood=3
GET /api/v1/neighborhoods/3/houses
```

---

## 5. HTTP Status Codes

Status codes communicate the result of a request. Use them correctly and consistently.

### 5.1 Categories

| Range | Category      | Meaning                          |
| ----- | ------------- | -------------------------------- |
| 1xx   | Informational | Request received, continuing     |
| 2xx   | Success       | Request was successful           |
| 3xx   | Redirection   | Further action needed            |
| 4xx   | Client Error  | Problem with the request         |
| 5xx   | Server Error  | Problem with the server          |

### 5.2 Most Commonly Used Codes

#### Success (2xx)

```
200 OK              — Standard success for GET, PUT, PATCH
201 Created         — Resource was created successfully (POST)
202 Accepted        — Request accepted, processing asynchronously
204 No Content      — Success but no response body (DELETE)
```

#### Client Errors (4xx)

```
400 Bad Request     — Malformed syntax, invalid input
401 Unauthorized    — Authentication required or failed
403 Forbidden       — Authenticated but not authorized
404 Not Found       — Resource does not exist
405 Method Not Allowed — HTTP method not supported on this resource
409 Conflict        — Conflict with current state (e.g., duplicate)
410 Gone            — Resource permanently deleted
415 Unsupported Media Type — Content-Type not supported
422 Unprocessable Entity — Validation errors
429 Too Many Requests — Rate limit exceeded
```

#### Server Errors (5xx)

```
500 Internal Server Error — Generic server error
502 Bad Gateway           — Invalid response from upstream
503 Service Unavailable   — Server temporarily down
504 Gateway Timeout       — Upstream server timeout
```

### 5.3 Choosing the Right Status Code

```
Created a new user?              → 201 Created
Updated an existing user?        → 200 OK
Deleted a user?                  → 204 No Content
User sent invalid JSON?          → 400 Bad Request
User not logged in?              → 401 Unauthorized
User logged in but no access?    → 403 Forbidden
User not found?                  → 404 Not Found
Duplicate email on signup?       → 409 Conflict
Validation failed?               → 422 Unprocessable Entity
Server crashed?                  → 500 Internal Server Error
```

---

## 6. Request & Response Design

### 6.1 Request Headers

```http
GET /api/v1/users HTTP/1.1
Host: api.example.com
Accept: application/json          # What format the client wants
Authorization: Bearer <token>     # Authentication credentials
Content-Type: application/json    # Format of request body
X-Request-ID: abc-123-def         # Correlation ID for tracing
Accept-Language: en-US            # Preferred language
Cache-Control: no-cache           # Caching directives
```

### 6.2 Response Structure (Consistent Envelope)

Use a consistent response structure across your entire API:

#### Success Response

```json
{
  "status": "success",
  "data": {
    "id": 42,
    "name": "Alice",
    "email": "alice@example.com",
    "role": "admin",
    "createdAt": "2026-01-15T08:30:00Z",
    "updatedAt": "2026-03-31T10:00:00Z"
  }
}
```

#### Collection Response

```json
{
  "status": "success",
  "data": [
    { "id": 1, "name": "Alice" },
    { "id": 2, "name": "Bob" }
  ],
  "meta": {
    "totalCount": 150,
    "page": 1,
    "perPage": 20,
    "totalPages": 8
  }
}
```

#### Error Response

```json
{
  "status": "error",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed for the request.",
    "details": [
      { "field": "email", "message": "must be a valid email address" },
      { "field": "name", "message": "must be between 2 and 100 characters" }
    ]
  }
}
```

### 6.3 Date/Time Format

Always use **ISO 8601** format with UTC timezone:

```json
{
  "createdAt": "2026-03-31T10:30:00Z",
  "scheduledFor": "2026-04-15T14:00:00+05:30"
}
```

### 6.4 Null vs. Absent Fields

Pick a strategy and be consistent:

```json
// Option A: Include null fields explicitly
{ "id": 1, "name": "Alice", "bio": null, "avatar": null }

// Option B: Omit null fields
{ "id": 1, "name": "Alice" }
```

---

## 7. Filtering, Sorting & Pagination

### 7.1 Filtering

Use query parameters for filtering:

```http
# Filter by status
GET /api/v1/orders?status=shipped

# Multiple filters
GET /api/v1/products?category=electronics&min_price=100&max_price=500

# Filter with operators
GET /api/v1/users?created_after=2026-01-01&role=admin

# Search
GET /api/v1/products?search=laptop
```

### 7.2 Sorting

```http
# Sort ascending
GET /api/v1/products?sort=price

# Sort descending (use - prefix)
GET /api/v1/products?sort=-price

# Multiple sort fields
GET /api/v1/products?sort=-rating,price
```

### 7.3 Pagination

#### Offset-Based Pagination (Simple)

```http
GET /api/v1/users?page=2&per_page=20

# Response
{
  "data": [...],
  "meta": {
    "page": 2,
    "perPage": 20,
    "totalCount": 150,
    "totalPages": 8
  },
  "links": {
    "self":  "/api/v1/users?page=2&per_page=20",
    "first": "/api/v1/users?page=1&per_page=20",
    "prev":  "/api/v1/users?page=1&per_page=20",
    "next":  "/api/v1/users?page=3&per_page=20",
    "last":  "/api/v1/users?page=8&per_page=20"
  }
}
```

#### Cursor-Based Pagination (Scalable)

Better for large datasets — avoids the "skip N rows" problem:

```http
GET /api/v1/users?limit=20&cursor=eyJpZCI6MTAwfQ==

# Response
{
  "data": [...],
  "meta": {
    "hasMore": true,
    "nextCursor": "eyJpZCI6MTIwfQ=="
  }
}
```

### 7.4 Field Selection (Sparse Fieldsets)

Allow clients to request only the fields they need:

```http
GET /api/v1/users?fields=id,name,email

# Response
[
  { "id": 1, "name": "Alice", "email": "alice@example.com" },
  { "id": 2, "name": "Bob", "email": "bob@example.com" }
]
```

---

## 8. Authentication & Authorization

### 8.1 API Keys

Simple but limited. Typically used for server-to-server communication:

```http
GET /api/v1/data HTTP/1.1
X-API-Key: sk_live_abc123def456ghi789
```

### 8.2 Basic Authentication

Base64-encoded username:password. Only use over HTTPS:

```http
GET /api/v1/users HTTP/1.1
Authorization: Basic YWxpY2U6cGFzc3dvcmQxMjM=
```

### 8.3 Bearer Token (JWT)

The most common modern approach:

```http
# Step 1: Login to get token
POST /api/v1/auth/login HTTP/1.1
Content-Type: application/json

{ "email": "alice@example.com", "password": "securePass123" }

# Response
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...",
  "expiresIn": 3600
}

# Step 2: Use token in subsequent requests
GET /api/v1/users/me HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### 8.4 OAuth 2.0

Used for third-party access delegation:

```
1. Client redirects user to → /oauth/authorize?client_id=XXX&redirect_uri=YYY&scope=read
2. User grants permission
3. Auth server redirects to → YYY?code=AUTH_CODE
4. Client exchanges code for token:
   POST /oauth/token
   { "grant_type": "authorization_code", "code": "AUTH_CODE", ... }
5. Client uses access token for API calls
```

### 8.5 Token Refresh Flow

```http
POST /api/v1/auth/refresh HTTP/1.1
Content-Type: application/json

{
  "refreshToken": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4..."
}

# Response: 200 OK
{
  "accessToken": "new.jwt.token",
  "expiresIn": 3600
}
```

---

## 9. Versioning

APIs evolve. Versioning prevents breaking changes for existing consumers.

### 9.1 Strategies

#### URI Path Versioning (Most Common)

```http
GET /api/v1/users
GET /api/v2/users
```

#### Query Parameter Versioning

```http
GET /api/users?version=1
GET /api/users?version=2
```

#### Header Versioning

```http
GET /api/users HTTP/1.1
Accept: application/vnd.myapi.v1+json
```

#### Content Negotiation

```http
GET /api/users HTTP/1.1
Accept: application/vnd.myapi+json;version=2
```

### 9.2 Best Practices

- Start with v1 from day one
- Only increment major versions for breaking changes
- Support at least N-1 version
- Provide a deprecation timeline (e.g., 6-12 months)
- Include deprecation headers:

```http
Deprecation: Sun, 01 Jan 2027 00:00:00 GMT
Sunset: Sun, 01 Jul 2027 00:00:00 GMT
Link: </api/v2/users>; rel="successor-version"
```

---

## 10. Error Handling

### 10.1 Consistent Error Format

```json
{
  "status": "error",
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "The requested user was not found.",
    "details": {
      "resource": "User",
      "id": 42
    },
    "timestamp": "2026-03-31T10:00:00Z",
    "path": "/api/v1/users/42",
    "traceId": "abc-123-def-456"
  }
}
```

### 10.2 Common Error Codes

```json
// 400 Bad Request — Malformed JSON
{
  "error": {
    "code": "INVALID_JSON",
    "message": "Request body contains invalid JSON at position 45."
  }
}

// 401 Unauthorized — Missing/invalid token
{
  "error": {
    "code": "TOKEN_EXPIRED",
    "message": "Your access token has expired. Please refresh your token."
  }
}

// 403 Forbidden — Insufficient permissions
{
  "error": {
    "code": "INSUFFICIENT_PERMISSIONS",
    "message": "You do not have permission to delete this resource."
  }
}

// 409 Conflict — Duplicate entry
{
  "error": {
    "code": "DUPLICATE_ENTRY",
    "message": "A user with this email already exists.",
    "details": { "field": "email", "value": "alice@example.com" }
  }
}

// 422 Unprocessable Entity — Validation error
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "One or more fields failed validation.",
    "details": [
      { "field": "email", "rule": "format", "message": "Invalid email format." },
      { "field": "age", "rule": "min", "message": "Must be at least 18." }
    ]
  }
}

// 429 Too Many Requests — Rate limited
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please retry after 30 seconds.",
    "retryAfter": 30
  }
}
```

### 10.3 Never Expose Internal Details

```json
// BAD — leaks stack trace and DB info
{
  "error": "NullPointerException at com.app.UserService.getUser(UserService.java:42)",
  "sql": "SELECT * FROM users WHERE id = 42"
}

// GOOD — safe generic message
{
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "An unexpected error occurred. Please try again later.",
    "traceId": "abc-123-def"
  }
}
```

---

## 11. HATEOAS

**HATEOAS** (Hypermedia As The Engine Of Application State) means that API responses include links to related actions and resources. The client doesn't need to hardcode URLs — it discovers them dynamically.

### 11.1 Example

```http
GET /api/v1/orders/99 HTTP/1.1

# Response: 200 OK
{
  "id": 99,
  "status": "pending",
  "total": 250.00,
  "customer": {
    "id": 42,
    "name": "Alice"
  },
  "links": [
    { "rel": "self",     "href": "/api/v1/orders/99",         "method": "GET" },
    { "rel": "cancel",   "href": "/api/v1/orders/99/cancel",  "method": "POST" },
    { "rel": "pay",      "href": "/api/v1/orders/99/pay",     "method": "POST" },
    { "rel": "customer", "href": "/api/v1/users/42",          "method": "GET" },
    { "rel": "items",    "href": "/api/v1/orders/99/items",   "method": "GET" }
  ]
}
```

Once the order is **shipped**, the available links change:

```json
{
  "id": 99,
  "status": "shipped",
  "links": [
    { "rel": "self",    "href": "/api/v1/orders/99",         "method": "GET" },
    { "rel": "track",   "href": "/api/v1/orders/99/tracking","method": "GET" },
    { "rel": "return",  "href": "/api/v1/orders/99/return",  "method": "POST" }
  ]
}
```

> Notice: "cancel" and "pay" are gone; "track" and "return" appeared. The API drives the workflow.

---

## 12. Rate Limiting & Throttling

### 12.1 Why Rate Limit?

- Prevent abuse and DDoS attacks
- Ensure fair usage
- Protect backend infrastructure
- Control costs

### 12.2 Common Strategies

| Strategy         | Description                                           |
| ---------------- | ----------------------------------------------------- |
| Fixed Window     | X requests per time window (e.g., 100/min)            |
| Sliding Window   | Rolling window for smoother rate control              |
| Token Bucket     | Tokens refill at a steady rate; each request costs 1  |
| Leaky Bucket     | Requests processed at a fixed rate, excess queued     |

### 12.3 Rate Limit Headers

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100          # Max requests per window
X-RateLimit-Remaining: 67       # Requests remaining
X-RateLimit-Reset: 1711872000   # Unix timestamp when window resets
Retry-After: 30                 # Seconds until retry (on 429)
```

### 12.4 Rate Limit Exceeded Response

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Try again in 30 seconds.",
    "limit": 100,
    "window": "1m",
    "retryAfter": 30
  }
}
```

---

## 13. Caching

### 13.1 Cache-Control Headers

```http
# Cache for 1 hour
Cache-Control: public, max-age=3600

# Private cache (only browser, not CDN)
Cache-Control: private, max-age=600

# No caching at all
Cache-Control: no-store

# Must revalidate with server before using cached copy
Cache-Control: no-cache
```

### 13.2 ETag-Based Caching (Conditional Requests)

```http
# First request
GET /api/v1/products/42 HTTP/1.1

# Response
HTTP/1.1 200 OK
ETag: "v1-abc123"
{ "id": 42, "name": "Laptop", "price": 999 }

# Subsequent request — client sends ETag back
GET /api/v1/products/42 HTTP/1.1
If-None-Match: "v1-abc123"

# If unchanged:
HTTP/1.1 304 Not Modified
# (No body — client uses cached version)

# If changed:
HTTP/1.1 200 OK
ETag: "v2-def456"
{ "id": 42, "name": "Laptop", "price": 899 }
```

### 13.3 Last-Modified Caching

```http
# Response includes Last-Modified
HTTP/1.1 200 OK
Last-Modified: Mon, 30 Mar 2026 10:00:00 GMT

# Client sends conditional request
GET /api/v1/products/42 HTTP/1.1
If-Modified-Since: Mon, 30 Mar 2026 10:00:00 GMT

# If unchanged → 304 Not Modified
# If changed   → 200 OK with new data
```

---

## 14. Idempotency

An operation is **idempotent** if performing it multiple times produces the same result as performing it once.

### 14.1 Why It Matters

Network failures happen. If a client retries a request, idempotency ensures no duplicate side effects (e.g., double charges, duplicate records).

### 14.2 Idempotency by Method

```
GET    → Always idempotent (read-only)
PUT    → Always idempotent (full replacement)
DELETE → Always idempotent (already gone = still gone)
PATCH  → NOT guaranteed idempotent (depends on implementation)
POST   → NOT idempotent (creates new resources each time)
```

### 14.3 Making POST Idempotent with Idempotency Keys

```http
POST /api/v1/payments HTTP/1.1
Content-Type: application/json
Idempotency-Key: pay_unique_abc123

{
  "amount": 100.00,
  "currency": "USD",
  "customerId": 42
}
```

The server stores the idempotency key. If the same key is sent again:
- If the original request is still processing → return `409 Conflict`
- If it already completed → return the cached original response
- Keys typically expire after 24 hours

---

## 15. API Security Best Practices

### 15.1 Checklist

```
✅  Always use HTTPS (TLS 1.2+)
✅  Authenticate every request (API keys, JWT, OAuth)
✅  Authorize at the resource level (RBAC, ABAC)
✅  Validate and sanitize ALL inputs
✅  Rate limit all endpoints
✅  Use parameterized queries (prevent SQL injection)
✅  Set CORS policies correctly
✅  Don't expose sensitive data in URLs
✅  Implement request size limits
✅  Log and monitor all access
✅  Use security headers
✅  Rotate secrets and tokens regularly
```

### 15.2 Security Headers

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
```

### 15.3 Input Validation Example

```json
// Request
POST /api/v1/users
{
  "name": "<script>alert('xss')</script>",
  "email": "not-an-email",
  "age": -5
}

// Response: 422 Unprocessable Entity
{
  "error": {
    "code": "VALIDATION_ERROR",
    "details": [
      { "field": "name", "message": "contains invalid characters" },
      { "field": "email", "message": "must be a valid email" },
      { "field": "age", "message": "must be a positive integer" }
    ]
  }
}
```

### 15.4 CORS Configuration

```
# Restrictive (recommended for production)
Access-Control-Allow-Origin: https://myapp.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400

# Open (ONLY for truly public APIs)
Access-Control-Allow-Origin: *
```

---

## 16. Advanced Patterns

### 16.1 Bulk Operations

```http
# Bulk create
POST /api/v1/users/bulk HTTP/1.1
Content-Type: application/json

{
  "users": [
    { "name": "Alice", "email": "alice@example.com" },
    { "name": "Bob", "email": "bob@example.com" },
    { "name": "Charlie", "email": "charlie@example.com" }
  ]
}

# Response: 207 Multi-Status
{
  "results": [
    { "index": 0, "status": 201, "data": { "id": 1, "name": "Alice" } },
    { "index": 1, "status": 201, "data": { "id": 2, "name": "Bob" } },
    { "index": 2, "status": 409, "error": { "code": "DUPLICATE_ENTRY", "message": "Email exists" } }
  ]
}
```

### 16.2 Long-Running Operations (Async)

```http
# Initiate an export
POST /api/v1/reports/export HTTP/1.1
Content-Type: application/json

{ "type": "sales", "dateRange": "2026-Q1" }

# Response: 202 Accepted
{
  "jobId": "job_abc123",
  "status": "processing",
  "statusUrl": "/api/v1/jobs/job_abc123",
  "estimatedCompletion": "2026-03-31T10:05:00Z"
}

# Poll for status
GET /api/v1/jobs/job_abc123

# Response (still processing)
{ "jobId": "job_abc123", "status": "processing", "progress": 65 }

# Response (completed)
{
  "jobId": "job_abc123",
  "status": "completed",
  "resultUrl": "/api/v1/reports/downloads/job_abc123",
  "expiresAt": "2026-04-01T10:00:00Z"
}
```

### 16.3 Webhooks (Event Notifications)

```http
# Register a webhook
POST /api/v1/webhooks HTTP/1.1
Content-Type: application/json

{
  "url": "https://myapp.com/hooks/orders",
  "events": ["order.created", "order.shipped", "order.delivered"],
  "secret": "whsec_abc123"
}

# Webhook payload sent by the API to your URL
POST https://myapp.com/hooks/orders
Content-Type: application/json
X-Webhook-Signature: sha256=abc123...

{
  "id": "evt_123",
  "type": "order.shipped",
  "timestamp": "2026-03-31T10:00:00Z",
  "data": {
    "orderId": 99,
    "trackingNumber": "1Z999AA10123456784"
  }
}
```

### 16.4 API Gateway Pattern

```
Client → API Gateway → Microservice A (Users)
                     → Microservice B (Orders)
                     → Microservice C (Payments)

Gateway handles: routing, auth, rate limiting, logging, load balancing
```

### 16.5 Content Negotiation

```http
# Request JSON
GET /api/v1/users/42 HTTP/1.1
Accept: application/json

# Request XML
GET /api/v1/users/42 HTTP/1.1
Accept: application/xml

# Request CSV
GET /api/v1/reports/sales HTTP/1.1
Accept: text/csv
```

### 16.6 Optimistic Concurrency Control

Prevent lost updates when multiple clients modify the same resource:

```http
# Client A fetches the resource
GET /api/v1/products/42 → ETag: "v5"

# Client A updates (includes ETag)
PUT /api/v1/products/42 HTTP/1.1
If-Match: "v5"
{ "name": "Updated Laptop", "price": 899 }

# If no one else changed it → 200 OK, ETag: "v6"
# If someone changed it first → 412 Precondition Failed
```

---

## 17. Practice Questions & Answers

### Beginner Level

---

**Q1.** What does REST stand for, and who introduced it?

**Answer:**
REST stands for **Representational State Transfer**. It was introduced by Roy Fielding in his doctoral dissertation in 2000. It's an architectural style, not a protocol or standard.

---

**Q2.** What are the six constraints of REST?

**Answer:**
1. Client-Server Separation
2. Statelessness
3. Cacheability
4. Uniform Interface
5. Layered System
6. Code on Demand (optional)

---

**Q3.** Which HTTP method would you use to create a new resource?

**Answer:**
**POST**. For example, `POST /api/v1/users` with a JSON body creates a new user. The server returns `201 Created` with the newly created resource and optionally a `Location` header pointing to the new resource's URI.

---

**Q4.** What is the difference between PUT and PATCH?

**Answer:**
- **PUT** replaces the entire resource. You must send all fields, even unchanged ones. If you omit a field, it may be set to null.
- **PATCH** partially updates a resource. You only send the fields you want to change.

Example:
```
PUT /users/1  → { "name": "Alice", "email": "a@b.com", "bio": "Dev" }  (all fields)
PATCH /users/1 → { "bio": "Senior Dev" }  (only the changed field)
```

---

**Q5.** What status code should you return after successfully deleting a resource?

**Answer:**
**204 No Content** — indicates success with no response body. Alternatively, **200 OK** with a confirmation message is also acceptable, but 204 is the most common convention.

---

**Q6.** Why should you use nouns (not verbs) in REST endpoint URLs?

**Answer:**
REST treats URLs as representations of **resources** (things), not actions. The HTTP method already specifies the action. Using verbs leads to redundancy:

```
BAD:  POST /createUser     → "create" is redundant with POST
GOOD: POST /users           → POST implies creation
```

---

**Q7.** What is the difference between 401 and 403 status codes?

**Answer:**
- **401 Unauthorized**: The client has not provided valid authentication credentials. "Who are you?"
- **403 Forbidden**: The client is authenticated but does not have permission. "I know who you are, but you can't do this."

---

**Q8.** Convert this bad API design into a good one:

```
GET /getAllUsers
POST /createNewUser
PUT /updateUserInfo/42
DELETE /removeUser/42
GET /fetchUserOrders/42
```

**Answer:**
```
GET    /api/v1/users
POST   /api/v1/users
PUT    /api/v1/users/42
DELETE /api/v1/users/42
GET    /api/v1/users/42/orders
```

---

### Intermediate Level

---

**Q9.** Design a complete REST API for a blog platform with users, posts, and comments.

**Answer:**

```
# Users
GET    /api/v1/users                    → List all users
POST   /api/v1/users                    → Create a user
GET    /api/v1/users/:id                → Get a user
PUT    /api/v1/users/:id                → Update a user
DELETE /api/v1/users/:id                → Delete a user

# Posts
GET    /api/v1/posts                    → List all posts
POST   /api/v1/posts                    → Create a post
GET    /api/v1/posts/:id                → Get a post
PUT    /api/v1/posts/:id                → Update a post
DELETE /api/v1/posts/:id                → Delete a post
GET    /api/v1/users/:id/posts          → Get all posts by a user

# Comments
GET    /api/v1/posts/:id/comments       → List comments on a post
POST   /api/v1/posts/:id/comments       → Add a comment to a post
GET    /api/v1/comments/:id             → Get a specific comment
PUT    /api/v1/comments/:id             → Update a comment
DELETE /api/v1/comments/:id             → Delete a comment

# Filtering / Sorting / Pagination
GET    /api/v1/posts?author=42&status=published&sort=-createdAt&page=1&per_page=10
GET    /api/v1/posts?search=rest+api&tags=tutorial,beginner
```

---

**Q10.** What is idempotency? Which HTTP methods are idempotent?

**Answer:**
An operation is **idempotent** if executing it multiple times has the same effect as executing it once.

- **Idempotent:** GET, PUT, DELETE, HEAD, OPTIONS
- **Non-idempotent:** POST, PATCH (typically)

Example: `DELETE /users/42` — calling it once deletes the user. Calling it again returns 404, but the server state hasn't changed further. Same outcome = idempotent.

`POST /users` — calling it twice creates two different users = not idempotent.

---

**Q11.** Explain the difference between offset-based and cursor-based pagination.

**Answer:**

**Offset-Based:**
- Uses `page` and `per_page` parameters
- Easy to implement; supports jumping to any page
- Performance degrades with large offsets (`OFFSET 1000000` is slow)
- Inconsistent if data changes between pages (items can be skipped or duplicated)

**Cursor-Based:**
- Uses an opaque cursor pointing to the last item seen
- Consistent results even when data changes
- Scales well for large datasets (no offset scanning)
- Cannot jump to arbitrary pages

Use offset for small datasets and admin dashboards. Use cursor for feeds, timelines, and large datasets.

---

**Q12.** How would you handle file uploads in a REST API?

**Answer:**

```http
# Option 1: multipart/form-data (most common)
POST /api/v1/users/42/avatar HTTP/1.1
Content-Type: multipart/form-data; boundary=----Boundary

------Boundary
Content-Disposition: form-data; name="file"; filename="avatar.jpg"
Content-Type: image/jpeg

(binary data)
------Boundary--

# Response: 200 OK
{
  "url": "https://cdn.example.com/avatars/42.jpg",
  "size": 204800,
  "mimeType": "image/jpeg"
}

# Option 2: Pre-signed URL (for large files)
# Step 1: Request upload URL
POST /api/v1/uploads/presign
{ "fileName": "report.pdf", "contentType": "application/pdf" }

# Response
{
  "uploadUrl": "https://s3.amazonaws.com/bucket/...",
  "fileId": "file_abc123",
  "expiresAt": "2026-03-31T11:00:00Z"
}

# Step 2: Client uploads directly to storage
PUT https://s3.amazonaws.com/bucket/...  (binary data)

# Step 3: Confirm upload
POST /api/v1/uploads/file_abc123/confirm
```

---

**Q13.** What is HATEOAS and why is it useful?

**Answer:**
HATEOAS (Hypermedia As The Engine Of Application State) means API responses include **links to related resources and available actions**. Benefits:

1. **Discoverability**: Clients can navigate the API by following links, no hardcoded URLs needed.
2. **Decoupling**: Server can change URLs without breaking clients.
3. **Self-documentation**: Responses tell clients what they can do next.
4. **Workflow enforcement**: Only valid actions for the current state are shown.

Example: A "pending" order shows "cancel" and "pay" links, but a "shipped" order shows "track" and "return" links.

---

**Q14.** How do you handle API versioning when introducing a breaking change?

**Answer:**

Steps:
1. Create the new version (`/api/v2/users`) with the breaking change
2. Keep the old version (`/api/v1/users`) running
3. Add deprecation headers to v1 responses:
   ```http
   Deprecation: Sun, 01 Jan 2027 00:00:00 GMT
   Sunset: Sun, 01 Jul 2027 00:00:00 GMT
   ```
4. Document the migration guide with clear before/after examples
5. Notify consumers via email, changelog, and developer portal
6. Monitor v1 usage and provide a grace period (6-12 months)
7. Eventually sunset v1

---

### Advanced Level

---

**Q15.** Design a REST API for a ride-sharing application (like Uber/Ola).

**Answer:**

```
# Authentication
POST   /api/v1/auth/register           → Register (rider or driver)
POST   /api/v1/auth/login              → Login
POST   /api/v1/auth/refresh            → Refresh token
POST   /api/v1/auth/logout             → Logout

# User Profile
GET    /api/v1/users/me                → Get current user profile
PUT    /api/v1/users/me                → Update profile
POST   /api/v1/users/me/documents      → Upload verification documents

# Rides (Rider)
POST   /api/v1/rides/estimate          → Get fare estimate
POST   /api/v1/rides                   → Request a ride
GET    /api/v1/rides/:id               → Get ride details
POST   /api/v1/rides/:id/cancel        → Cancel a ride
GET    /api/v1/rides/:id/track         → Live tracking
POST   /api/v1/rides/:id/rate          → Rate the ride
GET    /api/v1/rides?status=completed&sort=-createdAt&page=1  → Ride history

# Rides (Driver)
GET    /api/v1/drivers/me/requests     → See incoming ride requests
POST   /api/v1/rides/:id/accept        → Accept a ride
POST   /api/v1/rides/:id/arrive        → Mark arrived at pickup
POST   /api/v1/rides/:id/start         → Start the trip
POST   /api/v1/rides/:id/complete      → Complete the trip

# Payments
GET    /api/v1/payments/methods        → List payment methods
POST   /api/v1/payments/methods        → Add a payment method
GET    /api/v1/rides/:id/receipt       → Get receipt
POST   /api/v1/rides/:id/tip          → Add tip

# Real-time (WebSocket or SSE)
WS     /api/v1/rides/:id/live          → Real-time location updates

# Admin
GET    /api/v1/admin/rides?status=active   → Monitor active rides
GET    /api/v1/admin/drivers?status=online → Online drivers
GET    /api/v1/admin/analytics/revenue     → Revenue dashboard
```

---

**Q16.** How would you design rate limiting for an API with different tiers?

**Answer:**

```
# Tier definitions
Free:       100 requests/hour,  1,000/day
Pro:        1,000 requests/hour, 10,000/day
Enterprise: 10,000 requests/hour, unlimited/day

# Implementation approach: Token Bucket per user per tier

# Response headers (always include these)
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1711872000
X-RateLimit-Tier: pro

# Rate-limited response
HTTP/1.1 429 Too Many Requests
Retry-After: 42
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Pro tier limit of 1000 requests/hour exceeded.",
    "currentUsage": 1000,
    "limit": 1000,
    "resetAt": "2026-03-31T11:00:00Z",
    "upgradeUrl": "/api/v1/billing/upgrade"
  }
}
```

Design considerations:
- Use Redis for distributed counting
- Apply rate limits at the API Gateway layer
- Different limits per endpoint (e.g., search is more expensive)
- Include a "burst" allowance (e.g., 50 extra requests as buffer)
- Provide clear documentation for each tier's limits

---

**Q17.** How do you handle partial failures in bulk/batch API operations?

**Answer:**

Use HTTP **207 Multi-Status** to report individual outcomes:

```http
POST /api/v1/orders/bulk HTTP/1.1
Content-Type: application/json

{
  "orders": [
    { "productId": 1, "quantity": 2 },
    { "productId": 999, "quantity": 1 },
    { "productId": 3, "quantity": 5 }
  ]
}

# Response: 207 Multi-Status
{
  "summary": { "total": 3, "succeeded": 2, "failed": 1 },
  "results": [
    {
      "index": 0,
      "status": 201,
      "data": { "orderId": 501, "productId": 1, "quantity": 2 }
    },
    {
      "index": 1,
      "status": 404,
      "error": { "code": "PRODUCT_NOT_FOUND", "message": "Product 999 not found" }
    },
    {
      "index": 2,
      "status": 201,
      "data": { "orderId": 502, "productId": 3, "quantity": 5 }
    }
  ]
}
```

Design decision: Should bulk operations be **atomic** (all-or-nothing) or **partial** (best-effort)?
- Atomic → Use transactions; return 400 if any fail
- Partial → Use 207; report each result individually (more common in practice)

---

**Q18.** Design an API that supports real-time updates. Compare polling, SSE, and WebSockets.

**Answer:**

| Feature            | Polling           | SSE                    | WebSocket            |
| ------------------ | ----------------- | ---------------------- | -------------------- |
| Direction          | Client → Server   | Server → Client        | Bidirectional        |
| Connection         | New per request   | Persistent (HTTP)      | Persistent (WS)      |
| Overhead           | High              | Low                    | Very Low             |
| Complexity         | Simple            | Medium                 | High                 |
| Use Case           | Dashboards        | Notifications, feeds   | Chat, gaming, trading|
| Auto-reconnect     | N/A               | Built-in               | Manual               |
| Through firewalls  | Always works      | Usually works          | Sometimes blocked    |

**REST + Polling:**
```http
# Client polls every 5 seconds
GET /api/v1/rides/42/status
{ "status": "driver_en_route", "eta": 3, "driverLocation": {...} }
```

**REST + SSE (Server-Sent Events):**
```http
GET /api/v1/rides/42/stream
Accept: text/event-stream

# Server sends events
data: {"status":"driver_en_route","eta":3}

data: {"status":"driver_arrived"}

data: {"status":"trip_started"}
```

**REST + WebSocket (for bidirectional):**
```
# Initial ride request via REST
POST /api/v1/rides → { "rideId": 42 }

# Then upgrade to WebSocket for real-time
WS /api/v1/rides/42/live

# Server → Client
{ "type": "location_update", "lat": 12.97, "lng": 77.59 }
{ "type": "status_change", "status": "arrived" }

# Client → Server
{ "type": "message", "text": "I'm at gate 2" }
```

---

**Q19.** How would you design backward-compatible API changes without versioning?

**Answer:**

Strategies for **non-breaking (additive) changes**:

1. **Add new fields** — never remove or rename existing ones
   ```json
   // Before
   { "id": 1, "name": "Alice" }
   // After (non-breaking)
   { "id": 1, "name": "Alice", "avatar": "https://..." }
   ```

2. **Add new endpoints** — never remove old ones

3. **Use feature flags/headers**
   ```http
   GET /api/v1/users/42
   X-Features: include-analytics,new-scoring
   ```

4. **Expand enums, don't change them**
   ```
   // Before: status ∈ {active, inactive}
   // After:  status ∈ {active, inactive, suspended}
   // Clients should handle unknown values gracefully
   ```

5. **Tolerant Reader Pattern** — clients should ignore unknown fields

6. **Default values** — new required fields should have sensible defaults

7. **Deprecate before removing**
   ```http
   # Mark old field as deprecated in docs
   {
     "userName": "alice",           // deprecated
     "username": "alice",           // new preferred field
     "display_name": "Alice Smith"  // new field
   }
   ```

---

**Q20.** You're building a REST API that will be used by mobile apps with unreliable networks. How would you make it resilient?

**Answer:**

1. **Idempotency Keys** for all write operations:
   ```http
   POST /api/v1/payments
   Idempotency-Key: client_generated_uuid_abc123
   ```
   The server caches the response for this key. Retries return the cached response.

2. **ETags for Conflict Detection:**
   ```http
   PUT /api/v1/cart/items/5
   If-Match: "v3"
   ```
   Prevents overwriting changes from another session.

3. **Retry-Friendly Responses:**
   ```http
   HTTP/1.1 503 Service Unavailable
   Retry-After: 5
   ```

4. **Partial Response Support** to reduce payload size:
   ```http
   GET /api/v1/users/42?fields=id,name,avatar
   ```

5. **Compression:**
   ```http
   Accept-Encoding: gzip, br
   ```

6. **Conditional Requests** to avoid re-downloading unchanged data:
   ```http
   GET /api/v1/feed
   If-None-Match: "etag_abc"
   → 304 Not Modified (no body transferred)
   ```

7. **Request Queuing on Client** — queue failed POST/PUT requests and retry with exponential backoff:
   ```
   Retry 1: wait 1s
   Retry 2: wait 2s
   Retry 3: wait 4s
   Retry 4: wait 8s (max)
   ```

8. **Batch Endpoints** to reduce round trips:
   ```http
   POST /api/v1/batch
   {
     "requests": [
       { "method": "GET", "path": "/users/me" },
       { "method": "GET", "path": "/notifications?unread=true" },
       { "method": "GET", "path": "/feed?limit=10" }
     ]
   }
   ```

9. **Offline-Friendly Design:**
   - Return `Last-Modified` or `updatedAt` timestamps on all resources
   - Support sync endpoints: `GET /api/v1/sync?since=2026-03-31T00:00:00Z`
   - Use 409 Conflict with merge strategies for conflicting offline edits

---

**Q21.** Design the API error handling strategy for a payment processing system.

**Answer:**

Payment systems require extra precision in error handling:

```json
// Insufficient funds
HTTP 402 Payment Required
{
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "The card does not have enough funds.",
    "declineCode": "insufficient_funds",
    "param": "source",
    "paymentIntentId": "pi_abc123",
    "retryable": true,
    "suggestedAction": "Try a different payment method."
  }
}

// Card declined
HTTP 402 Payment Required
{
  "error": {
    "code": "CARD_DECLINED",
    "message": "The card was declined by the issuer.",
    "declineCode": "generic_decline",
    "retryable": false,
    "suggestedAction": "Contact your bank or try a different card."
  }
}

// Idempotency conflict
HTTP 409 Conflict
{
  "error": {
    "code": "IDEMPOTENCY_CONFLICT",
    "message": "A request with this idempotency key is already processing.",
    "idempotencyKey": "pay_abc123",
    "originalRequestStatus": "processing",
    "retryable": true,
    "retryAfter": 5
  }
}

// Transient failure (network issue with payment processor)
HTTP 503 Service Unavailable
{
  "error": {
    "code": "PROCESSOR_UNAVAILABLE",
    "message": "Payment processor temporarily unavailable.",
    "retryable": true,
    "retryAfter": 10
  }
}
```

Key principles:
- Always include `retryable` flag so clients know whether to retry
- Include `suggestedAction` for user-facing errors
- Include `declineCode` for programmatic handling
- Never expose raw processor errors (wrap them)
- Log full details server-side, send safe summaries to client
- Use idempotency keys for ALL payment operations

---

**Q22.** What are the trade-offs between REST, GraphQL, and gRPC? When would you choose each?

**Answer:**

| Aspect              | REST              | GraphQL            | gRPC               |
| ------------------- | ----------------- | ------------------ | ------------------- |
| Protocol            | HTTP/1.1+         | HTTP               | HTTP/2              |
| Data Format         | JSON (usually)    | JSON               | Protocol Buffers    |
| Schema              | OpenAPI (optional) | Strong (SDL)       | Strong (.proto)     |
| Over-fetching       | Common            | Solved             | Solved              |
| Under-fetching      | Common            | Solved             | N/A                 |
| Caching             | Native HTTP       | Complex            | Manual              |
| Real-time           | SSE/WS            | Subscriptions      | Bidirectional stream|
| File Upload         | Easy              | Complex            | Streaming           |
| Learning Curve      | Low               | Medium             | High                |
| Browser Support     | Native            | Native             | Needs grpc-web      |

**Choose REST when:**
- Building public APIs or simple CRUD apps
- HTTP caching is important
- Wide client compatibility needed
- Team is small or prefers simplicity

**Choose GraphQL when:**
- Multiple clients need different data shapes (mobile vs web)
- Deeply nested or relational data
- Reducing round trips is critical
- Rapid frontend iteration needed

**Choose gRPC when:**
- Microservice-to-microservice communication
- Low latency and high throughput required
- Bidirectional streaming needed
- Strong typing and code generation important

---

### Expert / System Design Level

---

**Q23.** You are the API architect for a global e-commerce platform serving 10M+ users. Design the API architecture.

**Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
│   Web App (React)  │  Mobile (iOS/Android)  │  Partner APIs     │
└──────────┬─────────┴──────────┬─────────────┴──────┬────────────┘
           │                    │                     │
           ▼                    ▼                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API GATEWAY (Kong/AWS API GW)              │
│  Rate Limiting │ Auth │ Routing │ SSL │ Logging │ Load Balance  │
└──────────┬──────────────────────────────────────────────────────┘
           │
     ┌─────┴─────┐
     ▼           ▼
┌─────────┐ ┌──────────┐
│ REST API│ │ GraphQL  │  ← BFF (Backend for Frontend)
│ (public)│ │ (mobile) │
└────┬────┘ └────┬─────┘
     │           │
     ▼           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MICROSERVICES (gRPC internally)               │
│                                                                 │
│  User Service  │  Product Service  │  Order Service              │
│  Cart Service  │  Payment Service  │  Inventory Service          │
│  Search Service│  Notification Svc │  Review Service             │
│  Shipping Svc  │  Analytics Svc    │  Recommendation Svc         │
└─────────────────────────────────────────────────────────────────┘
           │
     ┌─────┴─────────────┐
     ▼                   ▼
┌──────────┐   ┌──────────────────┐
│  Caching │   │  Message Queue   │
│  (Redis) │   │  (Kafka/RabbitMQ)│
└──────────┘   └──────────────────┘
```

API design decisions:
- **Public REST API** for partners with API keys and rate limiting by tier
- **GraphQL BFF** for mobile to minimize over-fetching and round trips
- **gRPC** for internal service-to-service communication (fast, typed)
- **Event-driven** architecture with Kafka for order processing, inventory updates, notifications
- **Redis** caching for product catalog, session data, rate limiting
- **CDN** for static assets and product images
- **Idempotency keys** on all write operations
- **Circuit breaker** pattern between services (if Payment is down, don't fail the whole checkout)
- **API versioning** via URI path for public API, header-based for internal

---

**Q24.** How do you design a REST API for multi-tenancy?

**Answer:**

Three approaches:

**Option A: Subdomain per Tenant**
```http
GET https://acme.api.example.com/v1/users
GET https://globex.api.example.com/v1/users
```

**Option B: Header-Based**
```http
GET /api/v1/users HTTP/1.1
X-Tenant-ID: acme
```

**Option C: Path-Based**
```http
GET /api/v1/tenants/acme/users
```

Recommended design:
- Use **Header-Based** (Option B) for SaaS APIs
- Tenant resolved at the API Gateway layer
- Every request is scoped to the tenant automatically
- Data isolation at the database level (schema-per-tenant or row-level security)
- Tenant-specific rate limits and quotas
- Tenant context injected into every service call
- Audit logs tagged with tenant ID

---

**Q25.** Write a comprehensive OpenAPI (Swagger) specification for a simple User CRUD API.

**Answer:**

```yaml
openapi: 3.0.3
info:
  title: User Management API
  description: A complete CRUD API for managing users
  version: 1.0.0
  contact:
    email: api@example.com

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://staging-api.example.com/v1
    description: Staging

paths:
  /users:
    get:
      summary: List all users
      operationId: listUsers
      tags: [Users]
      parameters:
        - name: page
          in: query
          schema: { type: integer, default: 1 }
        - name: per_page
          in: query
          schema: { type: integer, default: 20, maximum: 100 }
        - name: sort
          in: query
          schema: { type: string, enum: [name, -name, createdAt, -createdAt] }
        - name: search
          in: query
          schema: { type: string }
      responses:
        '200':
          description: A paginated list of users
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  meta:
                    $ref: '#/components/schemas/PaginationMeta'

    post:
      summary: Create a new user
      operationId: createUser
      tags: [Users]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: User created
          headers:
            Location:
              schema: { type: string }
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '409':
          description: Email already exists
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '422':
          description: Validation error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ValidationError'

  /users/{id}:
    get:
      summary: Get a user by ID
      operationId: getUser
      tags: [Users]
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: integer }
      responses:
        '200':
          description: User found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: User not found

    put:
      summary: Update a user (full replacement)
      operationId: updateUser
      tags: [Users]
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: integer }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateUserRequest'
      responses:
        '200':
          description: User updated
        '404':
          description: User not found
        '412':
          description: Precondition failed (ETag mismatch)

    delete:
      summary: Delete a user
      operationId: deleteUser
      tags: [Users]
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: integer }
      responses:
        '204':
          description: User deleted
        '404':
          description: User not found

components:
  schemas:
    User:
      type: object
      properties:
        id: { type: integer, example: 42 }
        name: { type: string, example: "Alice" }
        email: { type: string, format: email }
        role: { type: string, enum: [user, admin] }
        createdAt: { type: string, format: date-time }
        updatedAt: { type: string, format: date-time }

    CreateUserRequest:
      type: object
      required: [name, email, password]
      properties:
        name: { type: string, minLength: 2, maxLength: 100 }
        email: { type: string, format: email }
        password: { type: string, minLength: 8 }
        role: { type: string, enum: [user, admin], default: user }

    UpdateUserRequest:
      type: object
      required: [name, email]
      properties:
        name: { type: string, minLength: 2, maxLength: 100 }
        email: { type: string, format: email }
        role: { type: string, enum: [user, admin] }

    PaginationMeta:
      type: object
      properties:
        page: { type: integer }
        perPage: { type: integer }
        totalCount: { type: integer }
        totalPages: { type: integer }

    Error:
      type: object
      properties:
        status: { type: string, example: error }
        error:
          type: object
          properties:
            code: { type: string }
            message: { type: string }

    ValidationError:
      type: object
      properties:
        status: { type: string, example: error }
        error:
          type: object
          properties:
            code: { type: string, example: VALIDATION_ERROR }
            message: { type: string }
            details:
              type: array
              items:
                type: object
                properties:
                  field: { type: string }
                  message: { type: string }

  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - BearerAuth: []
```

---

## Quick Reference Cheat Sheet

```
┌──────────────────────────────────────────────────────────────┐
│                    REST API CHEAT SHEET                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  METHODS        URLS              STATUS CODES               │
│  ───────        ────              ────────────               │
│  GET   = Read   /resources        200 OK                     │
│  POST  = Create /resources        201 Created                │
│  PUT   = Replace/resources/:id    204 No Content             │
│  PATCH = Update /resources/:id    400 Bad Request            │
│  DELETE= Remove /resources/:id    401 Unauthorized           │
│                                   403 Forbidden              │
│  RULES                            404 Not Found              │
│  ─────                            409 Conflict               │
│  ✓ Use nouns, not verbs           422 Validation Error       │
│  ✓ Use plural names               429 Rate Limited           │
│  ✓ Use lowercase + hyphens        500 Server Error           │
│  ✓ Version your API                                          │
│  ✓ Use JSON                                                  │
│  ✓ Always use HTTPS                                          │
│  ✓ Paginate collections                                      │
│  ✓ Return proper status codes                                │
│  ✓ Handle errors consistently                                │
│  ✓ Document with OpenAPI                                     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

*Last updated: March 2026*
