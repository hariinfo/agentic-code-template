---
name: gen-techspec
description: Generate comprehensive technical specification documenting every implementation detail of legacy system. Reads reorganized source-extracts.md to produce engineering blueprint with API contracts, data model, SQL logic, error handling, concurrency patterns, messaging contracts, security implementation, performance characteristics, configuration, and infrastructure dependencies. Cross-references to funcspec FRs and behavioral inventory. Output: techspec.md with 13 sections for modernization code generation.
---

# Technical Specification Writer (gen-techspec)

## Purpose

Transform raw technical analysis into a comprehensive **technical specification** — the engineering blueprint that translates every implementation detail of the legacy system into a structured reference for modernization developers.

The techspec bridges the gap between functional requirements (funcspec.md) and modernization code generation. Where the funcspec answers "what does the system do?" from a business perspective, the techspec answers "how does the legacy code actually implement it?" at an engineering level.

The techspec is built entirely from `source-extracts.md` — it does **not re-read** the legacy codebase. It reorganizes per-file technical details into per-concern sections: all API contracts together, all SQL together, all error handling together, etc. This gives developers a coherent technical reference instead of scattered file-by-file extracts.

## Inputs

You will receive a **slug** — the directory name under `modernization-output/` containing:

- `analysis.md` (required) — component inventory, inbound pathways, behavioral inventory, data flow
- `source-extracts.md` (required, **primary source**) — complete raw technical dump with every detail already extracted during analysis: SQL statements, API schemas, error handling, configuration, messaging, security, transactions, caching
- `funcspec.md` (required) — functional requirements to cross-reference

**Critical:** Do NOT re-read the legacy codebase. All source code details are in source-extracts.md. The skill's job is to reorganize and cross-reference those extracts into an engineering specification.

## Trigger

```
/gen-techspec <slug>
```

Example: `/gen-techspec order-jsp`

## Process

### Step 1: Load and Index All Inputs

Load the three input documents and build working indexes:

**From analysis.md:**
- Component inventory (roles and categories)
- Behavioral inventory with all behaviors, rules, and side effects
- Technology inventory (frameworks, databases, messaging systems)
- Inbound pathways (who calls what)
- Data flow overview

**From funcspec.md:**
- All functional requirements (FR-NNN) and their descriptions
- Acceptance criteria (Given/When/Then scenarios)
- Data entities and relationships
- Integration points
- Non-functional requirements

**From source-extracts.md:**
- Every file section with its role and content
- Class definitions with all fields and methods
- SQL queries and stored procedure bodies (COMPLETE TEXT)
- Validation annotations and constraints
- Configuration properties and environment variables
- Exception handling code and patterns
- Caching annotations and cache names
- Transaction boundaries and isolation levels
- Security checks and authorization decorators
- Test configurations and mocking setup

Index source-extracts.md by file role (API Controller, Service, DAO, Stored Procedure, Configuration, etc.) for rapid cross-referencing during reorganization.

### Step 2: Reorganize Source-Extracts by Technical Concern

The source-extracts.md is organized per-file. Reorganize its content into per-concern sections:

**Extract and group:**
- All API/REST endpoints → consolidate into unified API Contracts section
- All SQL queries, stored procedures, DDL → consolidate into Data Model & Storage section
- All Kafka topics, message producers/consumers → consolidate into Messaging & Event Contracts section
- All exception handling, retry logic → consolidate into Error Handling & Resilience section
- All @PreAuthorize, @Secured, security checks → consolidate into Security Implementation section
- All @Cacheable, cache managers, Redis configs → consolidate into Performance & Caching section
- All @Value, properties files, environment variables → consolidate into Configuration & Environment section
- All @Transactional, transaction propagation → consolidate into transaction section under Error Handling
- All @Aspect, filter chains, listeners → analyze for design patterns in Step 4

Keep track of source file and line numbers for every detail — developers need precise references back to legacy code.

### Step 3: Document API Contracts in Full

From source-extracts.md, extract all REST endpoint definitions. For each endpoint found:

**Extract from Controller sections:**
- HTTP method and path (`@RequestMapping`, `@GetMapping`, `@PostMapping`, etc.)
- All request parameters (path, query, body) with types and constraints
- All response status codes (200, 400, 401, 403, 404, 500) with schemas
- Request/response headers (Content-Type, Authorization, custom headers)
- Authentication mechanism (if any) — session, JWT, API key, OAuth
- Rate limiting rules (if any)
- Request body schema (from annotations: `@RequestBody`, validation annotations like `@NotNull`, `@Size`, etc.)
- Response body schema (from return type, Jackson annotations)
- Path parameters and their constraints
- Query parameters and defaults
- Pagination details (if this endpoint supports pagination: offset/limit or cursor-based)

For every API, trace backward to funcspec.md and note which FR(s) this API implements. Document the relationship in the "Funcspec Ref" field.

**Output format for each API:**

```markdown
### API-NNN: {Endpoint Name}
- **URL**: `{METHOD} {path}` (e.g., `POST /api/v1/orders`)
- **Authentication**: {mechanism or "None"}
- **Request Headers**:
  - {header-name}: {description, required/optional}
- **Request Body**:
  ```json
  {
    "field_name": {type, constraints, description},
    ...
  }
  ```
- **Response 200**:
  ```json
  {exact successful response schema}
  ```
- **Response 400**: {error condition and schema}
- **Response 401/403**: {authentication/authorization error and schema}
- **Response 404**: {not found error and schema}
- **Response 500**: {server error and schema}
- **Rate Limiting**: {if any — e.g., "10 requests per minute per user"}
- **Pagination**: {if applicable — e.g., "offset/limit", "0-based indexing"}
- **Funcspec Ref**: {FR-NNN}
- **Source**: {legacy file:line — e.g., "OrderController.java:45-120"}
```

### Step 4: Consolidate SQL and Data Logic

From source-extracts.md, extract all database-related information:

**Extract complete database schema:**
- Every table with all columns, types, nullability, defaults, constraints
- Primary keys, foreign keys (with exact target references)
- Unique constraints
- Check constraints
- Indexes (name, columns, type: unique/btree/etc.)
- Triggers (name, event — INSERT/UPDATE/DELETE, logic summary)
- Sequences (name, start value, increment, cache, cycle policy)
- Views (full SELECT statement)
- Partitioning (if any)

**Extract stored procedures and functions:**
- Exact procedure name (schema-qualified)
- All parameters: name, type, direction (IN/OUT/INOUT)
- Complete procedure body (copy verbatim from source-extracts)
- Step-by-step logic breakdown (what does this procedure do, in order?)
- Conditional branches (if-then-else within procedure)
- Error handling within procedure (exceptions caught, rollback logic)
- Called by (which DAO/service invokes this?)
- Transaction behavior (does it commit? rollback on error? savepoints?)

**Extract data transformation flows:**
Build a table showing how data transforms as it moves through the system:
- Raw user input → validation → DTO → mapping → entity → SQL → database
- For each step, document: where it happens (Controller, Service, DAO), what validation/mapping occurs, any custom logic

**Flag gaps in source-extracts:**
If a stored procedure is referenced in code but its body wasn't captured in source-extracts, flag it: `⚠ INCOMPLETE — Procedure {name} referenced but body not found in source-extracts.md`

### Step 5: Document Messaging and Event Contracts

From source-extracts.md, extract all messaging details:

**For each Kafka topic, JMS queue, or message channel:**
- Topic/queue name (exact, as it appears in config)
- Direction: Producer, Consumer, or Both
- Message serialization (JSON, Avro, Protobuf, Java serialization)
- Complete message schema (all fields, types, descriptions)
- Partition key (if Kafka — which field determines partitioning?)
- Consumer group ID (for Kafka consumers)
- Offset strategy (earliest, latest, specific offset)
- Dead letter queue name (if applicable) and routing logic
- Retry policy: max retries, backoff strategy (exponential, fixed, linear)
- Ordering guarantees (per-partition, global, or none)
- When is this message produced (on what action/event)?
- What consumes this message and what does it do with it?

**Output format:**

```markdown
### MSG-NNN: {Topic Name}
- **Topic**: `{exact-topic-name}`
- **Direction**: Produce / Consume / Both
- **Serialization**: {format}
- **Message Schema**:
  ```json
  {
    "field_name": {type, constraint, description},
    ...
  }
  ```
- **Partition Key**: {field name or strategy}
- **Consumer Group**: `{group-id}`
- **Offset Strategy**: {earliest/latest/specific}
- **DLQ**: {queue name and condition for routing to DLQ}
- **Retry Policy**: {max retries}, {backoff strategy}
- **Ordering Guarantees**: {description}
- **Produced By**: {legacy file:line}
- **Consumed By**: {legacy file:line}
- **Funcspec Ref**: {FR-NNN}
```

### Step 6: Document Error Handling and Resilience

From source-extracts.md, consolidate all error handling:

**Extract exception hierarchy:**
- Every custom exception class (extract from source-extracts)
- Parent classes for each exception
- Where each exception is caught (which layer catches what?)
- What action is taken when caught (logged, transformed, rethrown, etc.)
- HTTP status code mapped (if this is a web exception)
- User-facing error message (if any)

**Extract retry strategies:**
- What operations have retry logic?
- How many retries allowed?
- Backoff strategy (fixed delay, exponential, etc.)
- Which exceptions trigger a retry?
- What happens after max retries (fallback, propagate, circuit break)?

**Extract transaction management:**
- Where are `@Transactional` boundaries?
- Isolation levels used (READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE)
- Propagation rules (REQUIRED, REQUIRES_NEW, SUPPORTS, etc.)
- Rollback conditions (on which exceptions?)
- Transaction timeout values
- Pessimistic/optimistic locking strategies

**Extract circuit breaker and bulkhead patterns:**
- If any circuit breaker patterns exist (e.g., Hystrix, Resilience4j), document: threshold, half-open conditions, timeout
- If thread pool isolation or bulkheads exist, document pool sizes, queue depth, rejection policy

**Output format:**

```markdown
### 6.1 Exception Hierarchy
| Exception Type | Caught Where | Action | HTTP Status | User Message |
|---|---|---|---|---|
| {ExceptionName} | {Layer — Controller/Service/DAO} | {logged/transformed/rethrown/handled} | {status code} | {message} |
| ... | ... | ... | ... | ... |

### 6.2 Retry Strategies
| Operation | Max Retries | Backoff Strategy | Retriable Exceptions | Fallback |
|---|---|---|---|---|
| {operation name} | {number} | {exponential/fixed/linear} | {exception classes} | {fallback action} |
| ... | ... | ... | ... | ... |

### 6.3 Transaction Management
| Boundary | Isolation Level | Propagation | Rollback On | Timeout |
|---|---|---|---|---|
| {method/operation} | {SERIALIZABLE/REPEATABLE_READ/etc.} | {REQUIRED/REQUIRES_NEW/etc.} | {exception types} | {seconds or "none"} |
| ... | ... | ... | ... | ... |

### 6.4 Circuit Breakers / Bulkheads
{If applicable: threshold, half-open conditions, pool sizes, queue depth, rejection policy}
```

### Step 7: Document Security Implementation

From source-extracts.md, extract all security-related code:

**Authentication flow:**
- How does a user authenticate? (session cookie, JWT token, API key, OAuth, LDAP, etc.)
- Where is authentication enforced (filter, aspect, method-level annotation)?
- What happens on failed authentication (redirect, 401 response, exception)?
- What user/principal information is stored in the security context?
- How long does an authentication session last (timeout)?

**Authorization rules:**
- Every `@PreAuthorize`, `@Secured`, `@RolesAllowed` annotation found
- Every manual authorization check (if statements checking roles/permissions)
- What roles/permissions are required for each operation?
- Where is authorization checked (controller, service, DAO)?
- What roles and permissions are defined in the system?

**Input validation and sanitization:**
- Every validation rule applied to user input (from annotations: `@NotNull`, `@Size`, `@Pattern`, `@Email`, etc.)
- Programmatic validation logic (if-statements, custom validators)
- Sanitization (HTML encoding, SQL injection prevention, XSS protection, etc.)
- Which layer performs validation (Controller, Service, custom validators)?

**Sensitive data handling:**
- What data is considered sensitive? (PII, financial data, passwords, tokens, etc.)
- How is sensitive data encrypted at rest (if stored)?
- How is it encrypted in transit (HTTPS, etc.)?
- Is sensitive data logged? (should it be masked?)
- Audit logging for access to sensitive data
- Data retention policies

**Output format:**

```markdown
### 7.1 Authentication Flow
{Step-by-step description: how a user authenticates, where credentials are verified, what happens on success/failure}

### 7.2 Authorization Matrix
| Resource/Action | Required Roles/Permissions | Check Location | Enforcement Method |
|---|---|---|---|
| {resource or operation} | {list of required roles} | {Controller/Service/DAO} | {annotation/@PreAuthorize/manual check} |
| ... | ... | ... | ... |

### 7.3 Input Validation & Sanitization
| Input Field | Validation Rules | Sanitization | Location |
|---|---|---|---|
| {field name} | {constraints — @NotNull, @Size(max=255), @Pattern(regex="...")} | {HTML encode/SQL escape/XSS prevention} | {layer} |
| ... | ... | ... | ... |

### 7.4 Sensitive Data Handling
- **PII Fields**: {list of fields containing PII}
- **Encryption at Rest**: {yes/no — which fields, which algorithm?}
- **Encryption in Transit**: {TLS/SSL version, cipher suites}
- **Sensitive Data Logging**: {what is/is not logged? masked?}
- **Audit Logging**: {what sensitive operations are audited?}
- **Data Retention**: {how long is sensitive data retained?}
```

### Step 8: Document Performance and Caching

From source-extracts.md, extract all performance-related details:

**Caching strategy:**
- Every `@Cacheable`, `@CachePut`, `@CacheEvict` annotation and the cache name
- Cache backend (in-memory, Redis, Memcached, etc.)
- Cache key strategy (how is the cache key built?)
- TTL (time-to-live) — when does cache expire?
- Eviction policy (LRU, LFU, FIFO, etc.)
- When is cache invalidated (on what events)?

**Connection and thread pools:**
- Database connection pool: min size, max size, timeout, idle timeout
- HTTP client connection pool: size, timeout, idle timeout
- Thread pools (if using Executor/ThreadPoolTaskExecutor): core size, max size, queue size, rejection policy
- Kafka consumer thread count

**Batch processing:**
- Are there batch operations? (e.g., bulk insert, batch update, background jobs)
- Batch size (how many records per batch?)
- Frequency (how often does batch run — scheduled, event-driven?)
- Parallel execution (is batch processing parallelized?)
- Success/failure handling in batch

**Output format:**

```markdown
### 8.1 Caching Strategy
| Cache Name | Key Pattern | Backend | TTL | Eviction | Invalidation Trigger |
|---|---|---|---|---|---|
| {cache-name} | {key format, e.g., "user:{userId}"} | {Redis/in-memory/etc.} | {seconds or "never"} | {LRU/LFU/none} | {on what event?} |
| ... | ... | ... | ... | ... | ... |

### 8.2 Connection Pools & Thread Pools
| Pool Name | Type | Min | Max | Timeout | Idle Timeout | Configuration Source |
|---|---|---|---|---|---|---|
| {pool-name} | {DB/HTTP/Kafka} | {count} | {count} | {seconds} | {seconds} | {property or JNDI resource} |
| ... | ... | ... | ... | ... | ... | ... |

### 8.3 Batch Processing
{If applicable: batch size, frequency, parallelization, error handling}
```

### Step 9: Document Configuration and Environment

From source-extracts.md, extract all configuration and environment variables:

**Properties and environment variables:**
- Every property key from .properties/.yml files found in source-extracts
- Default value (if any)
- Is it sensitive? (password, API key, token, etc.)
- What is it used for?
- Where is it used (which component)?
- Can it be overridden at runtime (environment variable, system property)?

**JNDI resources:**
- Every JNDI resource (datasource, JMS connection factory, mail session, etc.)
- Type and purpose
- Where is it defined (application server config, Spring bean definition)
- How is it looked up in code

**Feature flags and toggles:**
- Every feature flag found in code (if statements checking a feature toggle)
- Default state (enabled/disabled)
- How is it controlled (property, database, feature management service)
- What components are affected by this flag

**Output format:**

```markdown
### 9.1 Properties & Environment Variables
| Property/Variable | Purpose | Default | Sensitive? | Where Used |
|---|---|---|---|---|
| {key} | {description} | {value or "none"} | {yes/no} | {component} |
| ... | ... | ... | ... | ... |

### 9.2 JNDI Resources
| Name | Type | Purpose | Legacy Configuration |
|---|---|---|---|
| {jndi-name} | {Datasource/JMS/Mail/etc.} | {description} | {server config location} |
| ... | ... | ... | ... |

### 9.3 Feature Flags & Toggles
| Flag Name | Purpose | Default State | Components Affected |
|---|---|---|---|
| {flag-name} | {what does it control?} | {enabled/disabled} | {list of components} |
| ... | ... | ... | ... |
```

### Step 10: Document Infrastructure Dependencies

From analysis.md and source-extracts.md, extract all external infrastructure:

- What databases does this system depend on? (version, dialect)
- What message brokers? (Kafka version, broker count, etc.)
- What caches? (Redis, Memcached, etc.)
- What external services/APIs does it call?
- LDAP/Active Directory for authentication?
- SMTP for email?
- File storage (S3, NFS, local filesystem)?
- Any other external dependency

For each dependency:
- Type and version
- Protocol (JDBC, HTTP, gRPC, etc.)
- Purpose (what does the system use it for?)
- What happens if it's unavailable? (fallback, graceful degradation, failure)

**Output format:**

```markdown
## 10. Infrastructure Dependencies
| Dependency | Type | Version | Protocol | Purpose | Fallback if Unavailable |
|---|---|---|---|---|---|
| {name} | {DB/Cache/Queue/Service/etc.} | {version} | {JDBC/HTTP/gRPC/etc.} | {purpose} | {graceful degradation, circuit break, or "hard failure"} |
| ... | ... | ... | ... | ... | ... |
```

### Step 11: Identify Design Patterns and Architecture Notes

Analyze the code structures captured in source-extracts.md to identify patterns:

**Look for patterns:**
- Inheritance hierarchies (Strategy, Template Method via base classes)
- Listeners and event systems (Observer pattern)
- Decorators or wrapping classes
- Aspect-oriented code (@Aspect annotations)
- Factory patterns (builder classes, factory methods)
- Singleton beans (Spring `@Singleton`)
- Lazy initialization patterns
- Proxy patterns (dynamic proxies, Spring proxies)

For each pattern found:
- Name the pattern
- Where is it used (which classes/components)?
- Why was it chosen (what problem does it solve)?
- Should modernized code preserve or replace it?

**Output format:**

```markdown
## 11. Design Patterns & Architecture Notes

### Pattern: {Pattern Name}
- **Location**: {which classes implement this}
- **Purpose**: {what problem does this pattern solve?}
- **Implementation**: {brief description of how it's implemented}
- **Modernization Recommendation**: {Preserve / Replace with {modern alternative}}

{Repeat for each pattern found}
```

### Step 12: Document Known Technical Debt and Legacy Workarounds

From behavioral inventory and source-extracts.md, identify technical debt:

- Hardcoded values that should be configurable
- Performance anti-patterns (N+1 queries, inefficient loops, etc.)
- Security vulnerabilities or weak practices
- Overly coupled components
- Code duplications
- Outdated patterns or frameworks
- Scaling bottlenecks
- Commented-out code or TODO comments indicating unfinished work

For each debt item:
- Description (what's the problem?)
- Where is it (file, line)?
- Impact (performance, security, maintainability, scalability?)
- Recommendation (should modernization carry it forward or fix it?)

**Output format:**

```markdown
## 12. Known Technical Debt & Legacy Workarounds
| ID | Description | Location | Impact | Recommendation |
|---|---|---|---|---|
| TD-001 | {e.g., "Hardcoded timezone conversion to EST"} | {file:line} | {which quality is impacted?} | {Carry forward / Fix in modernization / Investigate} |
| TD-002 | {e.g., "N+1 queries in order list pagination"} | {file:line} | Performance | Fix — use lazy fetch or batch loading |
| ... | ... | ... | ... | ... |
```

### Step 13: Build Traceability Cross-References

For every major component in the techspec, trace backward to:
- Which functional requirement(s) does it implement? (FR-NNN from funcspec.md)
- Which behavioral inventory items does it relate to? (B-NNN from analysis.md)
- Which legacy files contain the source? (file:line)

Build a comprehensive traceability matrix:

**Output format:**

```markdown
## 13. Traceability
| Techspec Section | Funcspec Ref | Behavioral Inventory | Legacy Files |
|---|---|---|---|
| API-001 | FR-001, FR-002 | B-001, B-003 | OrderController.java:45-120, order.jsp:78-95 |
| MSG-001 | FR-007 | B-002 | OrderKafkaProducer.java:34 |
| SQL: ORDERS table | FR-001, FR-003 | B-004, B-005 | OrderDAO.java, order-tables.sql |
| ... | ... | ... | ... |
```

## Output: techspec.md

Create a file at `modernization-output/{slug}/techspec.md` with this complete structure:

```markdown
# Technical Specification: {Feature Name}

**Status**: Draft
**Modernization Initiative**: {slug}
**Last Updated**: {today's date}
**Derived From**: {entry point path from analysis.md}

---

## 1. Overview

{2-3 sentences describing the technical architecture — how the legacy code implements the feature. This is the complement to funcspec Overview. Include major technology choices.}

**Technology Stack**:
- Frontend: {list of frameworks/languages}
- Backend: {frameworks, languages}
- Database: {DBMS, versions}
- Messaging: {broker type, versions}
- Cache: {backend}
- Other: {any other significant tech}

**Key Technical Challenges**: {What makes this system complex?}

---

## 2. Architecture Summary

{Draw or describe the component architecture — how the discovered components connect technically}

{Mermaid diagram or ASCII diagram showing:}
- {Layer 1: Presentation/Controllers}
- {Layer 2: Business Services}
- {Layer 3: Data Access / Persistence}
- {Layer 4: Infrastructure (DB, Cache, Messaging)}
- {Connections and data flows between layers}

### Key Flows (Sequence Diagrams)

{For the main user interactions or critical business flows, describe the sequence:}

**Flow: {Flow Name}**
1. User action / External event
2. Which component is invoked (method/API)
3. What does it call next (service, DAO, external service)
4. What data is read/written
5. Response back up the chain

{Repeat for 2-3 main flows}

---

## 3. API Contracts

{For every REST endpoint, document as per Step 3 above}

{Example:}

### API-001: Create Order
- **URL**: `POST /api/v1/orders`
- **Authentication**: Session cookie (JSESSIONID)
- **Request Headers**:
  - `Content-Type: application/json` (required)
  - `X-CSRF-Token: {token}` (required for CSRF protection)
- **Request Body**:
  ```json
  {
    "customerId": {"type": "long", "required": true, "description": "ID of customer placing order"},
    "items": {"type": "array of OrderItem", "required": true, "minItems": 1},
    "shippingAddress": {"type": "Address", "required": true},
    "billingAddress": {"type": "Address", "required": false, "description": "If omitted, same as shippingAddress"}
  }
  ```
- **Response 200**:
  ```json
  {
    "orderId": {"type": "long"},
    "status": {"type": "string", "enum": ["PENDING", "PROCESSING", "CONFIRMED"]},
    "createdAt": {"type": "ISO-8601 datetime"},
    "total": {"type": "decimal", "format": "currency with 2 decimals"}
  }
  ```
- **Response 400**:
  ```json
  {
    "error": "VALIDATION_ERROR",
    "message": "One or more validation errors",
    "details": [{"field": "customerId", "issue": "Customer not found"}]
  }
  ```
- **Response 401**: {Unauthorized — session expired or invalid CSRF token}
- **Response 403**: {Forbidden — user lacks ROLE_CUSTOMER role}
- **Response 500**: {Server error}
- **Funcspec Ref**: FR-001
- **Source**: OrderController.java:45-120

### API-002: ...
{Repeat for each endpoint}

---

## 4. Data Model & Storage

### 4.1 Database Schema

{For each table, document as per Step 4 above}

**Table**: `APP.ORDERS`
**Purpose**: Core order record
**Columns**:
| Column | Type | Nullable | Default | Constraints | Description |
|---|---|---|---|---|---|
| ORDER_ID | BIGINT | N | Sequence | PK | Unique order identifier |
| CUSTOMER_ID | BIGINT | N | — | FK → CUSTOMERS | Customer who placed order |
| ORDER_DATE | TIMESTAMP | N | SYSDATE | — | When order was created |
| TOTAL | DECIMAL(10,2) | N | — | CHECK > 0 | Order total amount |
| STATUS | VARCHAR(20) | N | PENDING | CHECK IN ('PENDING','PROCESSING','CONFIRMED','SHIPPED','DELIVERED','CANCELLED') | Order status |
| ... | ... | ... | ... | ... | ... |

**Indexes**:
- `IX_ORDERS_CUSTOMER_ID` on (CUSTOMER_ID) — used by "find orders by customer" query
- `IX_ORDERS_ORDER_DATE` on (ORDER_DATE) — used by reporting queries
- ... (others)

**Foreign Keys**:
- CUSTOMER_ID → CUSTOMERS(CUSTOMER_ID)
- ... (others)

**Triggers**:
- `ORDERS_AUDIT_TRG` on INSERT/UPDATE — populates ORDERS_AUDIT table with old/new values for audit trail
- ... (others)

**Table**: `APP.ORDER_ITEMS`
**Purpose**: Line items within an order
**Columns**: {list as above}
**Foreign Keys**:
- ORDER_ID → ORDERS(ORDER_ID)
- PRODUCT_ID → PRODUCTS(PRODUCT_ID)

{Repeat for every table found in source-extracts.md}

---

### 4.2 Stored Procedures & Functions

**Procedure**: `PKG_ORDER.CREATE_PENDING_ORDER`
**Parameters**:
| Name | Type | Direction | Description |
|---|---|---|---|
| P_CUSTOMER_ID | BIGINT | IN | Customer ID |
| P_ORDER_DATE | DATE | IN | Date to use for order (or SYSDATE) |
| P_ORDER_ID | BIGINT | OUT | Generated order ID |
| P_ERROR_CODE | VARCHAR(20) | OUT | Error code if procedure fails |

**Logic** (step-by-step from source-extracts.md):
1. Validate P_CUSTOMER_ID exists in CUSTOMERS table, raise exception if not
2. Insert row into ORDERS table with status = PENDING
3. Retrieve the generated ORDER_ID (via sequence or identity)
4. Set P_ORDER_ID output parameter to new order ID
5. Commit transaction (implicit for stored procedure invocation)

**Called By**: OrderDAO.createPendingOrder() — OrderService.java:67

**Key SQL Statements**:
```sql
INSERT INTO APP.ORDERS (ORDER_ID, CUSTOMER_ID, ORDER_DATE, STATUS)
VALUES (SEQ_ORDER_ID.NEXTVAL, P_CUSTOMER_ID, P_ORDER_DATE, 'PENDING');
```

**Transaction Behavior**: Commit implicit; if any exception, rollback and return error code to caller

**Procedure**: ... (repeat for each procedure)

---

### 4.3 Data Transformations

| Step | Input | Output | Transformation Logic | Location |
|---|---|---|---|---|
| 1 | Raw HTTP request body (JSON) | CreateOrderRequest DTO | Jackson deserialization + validation (annotations) | OrderController.java:78 |
| 2 | CreateOrderRequest DTO | Order entity | Manual mapping in OrderService.toOrderEntity() — validate business rules, set defaults | OrderService.java:115 |
| 3 | Order entity | SQL INSERT statement | MyBatis mapper converts entity to column values, prepares parameterized query | OrderMapper.xml |
| 4 | Database ORDERS row | Order entity (loaded) | MyBatis result map maps columns back to entity fields | OrderMapper.xml |
| 5 | Order entity | OrderDTO (response) | Manual mapping OrderService.toOrderDTO() — selectively expose fields, format dates | OrderService.java:210 |
| 6 | OrderDTO | JSON response body | Jackson serialization with format annotations | OrderController.java:100 |

---

## 5. Messaging & Event Contracts

{For every Kafka topic or message queue, document as per Step 5 above}

### MSG-001: order.created
- **Topic**: `order.created`
- **Direction**: Produce
- **Serialization**: JSON (Kafka default)
- **Message Schema**:
  ```json
  {
    "eventId": "UUID for idempotency tracking",
    "orderId": "Generated order ID",
    "customerId": "Customer who created order",
    "total": "Order total (numeric)",
    "createdAt": "ISO-8601 timestamp when order was created",
    "source": "Literal string 'ORDER_SERVICE'"
  }
  ```
- **Partition Key**: `orderId` — ensures all events for same order go to same partition
- **Consumer Group**: {any consumers, e.g., "reporting-service", "fulfillment-service"}
- **Offset Strategy**: latest (new consumers start from new messages)
- **DLQ**: `order.created.dlq` — if consumer throws exception after 3 retries, message is moved to DLQ
- **Retry Policy**: 3 max retries, exponential backoff (1s, 2s, 4s)
- **Ordering Guarantees**: Per-partition ordering — all events for same orderId go to same partition, so consumer sees them in order
- **Produced By**: OrderService.publishOrderCreatedEvent() — OrderService.java:198
- **Consumed By**: ReportingEventConsumer, FulfillmentEventConsumer
- **Funcspec Ref**: FR-007

### MSG-002: ... (repeat for each topic)

---

## 6. Error Handling & Resilience

### 6.1 Exception Hierarchy

| Exception Type | Caught Where | Action | HTTP Status | User Message |
|---|---|---|---|---|
| OrderValidationException (extends BusinessException) | OrderController advice | Log warning, transform to ErrorResponse | 400 Bad Request | "Invalid order data: {detail}" |
| CustomerNotFoundException (extends BusinessException) | OrderService | Propagate to controller advice | 404 Not Found | "Customer not found" |
| DataAccessException (Spring) | Service/DAO | Log error, wrap in ServiceException | 500 Internal Server Error | "A system error occurred" |
| ... | ... | ... | ... | ... |

**Exception Sources** (from source-extracts.md):
- OrderValidationException — defined in OrderValidationException.java:1-45
- Custom exceptions defined in {location}
- Spring's DataAccessException — from Spring Framework

### 6.2 Retry Strategies

| Operation | Max Retries | Backoff Strategy | Retriable Exceptions | Fallback |
|---|---|---|---|---|
| Insert order into database | 3 | Exponential (100ms, 200ms, 400ms) | DataAccessException, SQLRecoverableException | Log error, throw ServiceException |
| Call fulfillment service (HTTP) | 3 | Fixed 1 second | IOException, HttpClientErrorException (5xx only, not 4xx) | Use previously cached inventory, log warning |
| Kafka message publish | 3 | Exponential | ProducerException | Log error, store in outbox table for manual retry |

**Retry Configuration Sources** (from source-extracts.md):
- Retries defined via Spring Retry `@Retryable` annotations
- Kafka client config: `retries=3`, `retry.backoff.ms=1000`
- RestTemplate configured via bean definition with HttpComponentsClientHttpRequestFactory

### 6.3 Transaction Management

| Boundary | Isolation Level | Propagation | Rollback On | Timeout |
|---|---|---|---|---|
| OrderService.createOrder() | SERIALIZABLE | REQUIRED | RuntimeException, custom BusinessException | 30 seconds |
| OrderDAO.insertOrder() | READ_COMMITTED | REQUIRED | RuntimeException | 5 seconds |
| OrderService.publishOrderCreatedEvent() | READ_COMMITTED | REQUIRES_NEW | RuntimeException | 10 seconds (publishes independently) |

**Configuration** (from source-extracts.md):
```java
@Transactional(isolation=Isolation.SERIALIZABLE, propagation=Propagation.REQUIRED, timeout=30)
public Order createOrder(CreateOrderRequest req) { ... }
```

**Database Transaction Defaults**:
- Oracle JDBC connection: autoCommit=false
- Connection pool timeout: 30 seconds
- Idle transaction timeout: 15 minutes (config param DB_IDLE_TX_TIMEOUT)

### 6.4 Circuit Breakers / Bulkheads

{If any: document Hystrix, Resilience4j, or manual circuit breaker patterns}

Example (if found in source-extracts.md):
- **FulfillmentServiceClient** uses Hystrix circuit breaker with:
  - Failure threshold: 50% of last 10 requests
  - Timeout per request: 5 seconds
  - Half-open state: try 1 request every 30 seconds
  - Fallback: return empty inventory (pessimistic inventory assumption)

---

## 7. Security Implementation

### 7.1 Authentication Flow

1. User submits login form (username + password) to LoginController
2. Spring Security's UsernamePasswordAuthenticationFilter intercepts request
3. Credentials validated against LDAP directory (configured via ldapAuthenticationProvider bean)
4. On success: Authentication object created, stored in SecurityContext
5. Session cookie (JSESSIONID) issued by servlet container
6. For subsequent requests: cookie presents JSESSIONID, Spring session repository loads SecurityContext
7. On logout: session invalidated, cookie deleted

**Session Timeout**: 30 minutes of inactivity (configured in web.xml: `session-config.timeout=30`)

**Password Policy** (from config):
- Minimum 8 characters
- Must contain uppercase, lowercase, digit, special char
- Enforced by LDAP policy (external system)

### 7.2 Authorization Matrix

| Resource/Action | Required Roles/Permissions | Check Location | Enforcement |
|---|---|---|---|
| POST /api/v1/orders (create order) | ROLE_CUSTOMER | OrderController.createOrder() | @PreAuthorize("hasRole('CUSTOMER')") |
| GET /api/v1/orders/{id} (view order) | ROLE_CUSTOMER + must own order | OrderController.getOrder() | Custom check in method: `if (!order.getCustomerId().equals(currentUserId)) throw AccessDeniedException` |
| POST /admin/orders/{id}/approve (admin approval) | ROLE_ADMIN | OrderController.approveOrder() | @PreAuthorize("hasRole('ADMIN')") |
| ... | ... | ... | ... |

**Roles Defined** (from source-extracts.md):
- ROLE_CUSTOMER — regular users
- ROLE_ADMIN — system administrators
- ROLE_AUDIT — read-only access for audit team

### 7.3 Input Validation & Sanitization

| Input Field | Validation Rules | Sanitization | Location |
|---|---|---|---|
| customerId (in request body) | @NotNull, @Positive | N/A (integer type prevents injection) | OrderRequest.java + Spring validation |
| shippingAddress.street | @NotNull, @Size(max=255), @Pattern(regex="[a-zA-Z0-9\\s,.-]*") | HTML encode on output | OrderAddressValidator.java |
| orderNotes | @Size(max=1000) | HTML encode before storing in DB | OrderController via sanitizer bean |
| ... | ... | ... | ... |

**Validation Framework**: Spring Validation with Hibernate Validator annotations

**Sanitization Methods** (from source-extracts.md):
- HTML encoding: Apache Commons Lang `StringEscapeUtils.escapeHtml4()`
- SQL parameters: MyBatis parameterized queries (prevents SQL injection)
- XSS prevention: Spring Security X-XSS-Protection header set to "1; mode=block"

### 7.4 Sensitive Data Handling

- **PII Fields**: customerId, customer name, email, phone, addresses (SHIPPING_ADDRESS, BILLING_ADDRESS columns in ORDERS)
- **Financial Data**: TOTAL amount in ORDERS
- **Passwords**: Never stored in application database — validated against external LDAP; never logged

- **Encryption at Rest**: None currently (data stored in clear in Oracle)
  - Recommendation: Encrypt TOTAL, address columns using Oracle Transparent Data Encryption (TDE)

- **Encryption in Transit**: HTTPS enforced via web.xml `<security-constraint>` with `CONFIDENTIAL` transport guarantee

- **Sensitive Data Logging**:
  - OrderService logs order creation with ID and total (acceptable — not PII)
  - Never logs customer email or address
  - Never logs CSRF tokens

- **Audit Logging**:
  - ORDERS_AUDIT table captures every INSERT/UPDATE (via trigger) with old values, new values, timestamp, username
  - Accessible only to ROLE_AUDIT users

- **Data Retention**:
  - Active orders retained indefinitely
  - Canceled orders retained for 7 years (regulatory requirement — see JIRA ticket COMP-4521)
  - ORDERS_AUDIT records retained for 10 years

---

## 8. Performance & Caching

### 8.1 Caching Strategy

| Cache Name | Key Pattern | Backend | TTL | Eviction | Invalidation Trigger |
|---|---|---|---|---|---|
| customer-details | "customer:{customerId}" | Redis (in-memory fallback if Redis unavailable) | 1 hour | LRU eviction at 80% capacity | On customer profile update in admin panel |
| products | "product:{productId}" | Redis | 6 hours | LRU | On product catalog update (admin trigger) |
| order-status-counts | "order-status-counts" | Redis | 5 minutes | Manual | Whenever order status changes |

**Cache Configuration** (from source-extracts.md):
- Redis connection pool: min=5, max=20, timeout=2 seconds
- Fallback: Spring local cache (ConcurrentHashMap) if Redis unavailable
- Bean: CachingConfig.java defines CacheManager with Redis backing

### 8.2 Connection Pools & Thread Pools

| Pool Name | Type | Min | Max | Timeout | Idle Timeout | Configuration |
|---|---|---|---|---|---|---|
| Database Connection Pool (HikariCP) | JDBC | 10 | 30 | 30 seconds | 10 minutes | application.properties: `spring.datasource.hikari.*` |
| HTTP Client Connection Pool | HTTP | 10 | 100 | 5 seconds | 30 seconds | RestTemplate bean config, max per route = 10 |
| Kafka Consumer Thread Pool | Kafka | 1 | 10 | N/A (persistent) | N/A | `num.stream.threads=4` in kafka config |
| Servlet container thread pool | Servlet | 20 | 200 | N/A | 1 minute idle | Tomcat embedded (spring.server.tomcat.threads.*) |

**Database Pool Details**:
```
spring.datasource.hikari.minimum-idle=10
spring.datasource.hikari.maximum-pool-size=30
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
```

### 8.3 Batch Processing

- **Batch Job**: OrderExportBatch (scheduled daily at 2 AM)
  - Batch Size: 500 orders per chunk
  - Frequency: Daily
  - Parallelization: Single-threaded (no parallelization)
  - Error Handling: If a chunk fails, entire batch is retried once; after 1 retry failure, batch marked as failed and manual review required
  - Configuration: Quartz job defined in QuartzConfig.java, schedule expression: `0 0 2 * * *` (2 AM daily)

---

## 9. Configuration & Environment

### 9.1 Properties & Environment Variables

| Property/Variable | Purpose | Default | Sensitive | Where Used |
|---|---|---|---|---|
| spring.datasource.url | Database JDBC connection string | jdbc:oracle:thin:@localhost:1521:ORCL | No | DataSource bean, OrderDAO |
| spring.datasource.username | Database username | (none — must be set) | Yes | DataSource bean |
| spring.datasource.password | Database password | (none — must be set) | Yes | DataSource bean |
| app.fulfillment.api.url | Fulfillment service base URL | http://fulfillment:8080 | No | FulfillmentServiceClient |
| app.fulfillment.api.timeout | HTTP timeout to fulfillment service | 5000 (ms) | No | FulfillmentServiceClient |
| app.cache.redis.host | Redis server hostname | localhost | No | CachingConfig |
| app.cache.redis.port | Redis server port | 6379 | No | CachingConfig |
| app.order.approval.enabled | Feature flag: enable order approval workflow | false | No | OrderApprovalService |
| app.order.max.amount | Maximum order amount allowed without approval | 1000.00 | No | OrderValidationService |
| ... | ... | ... | ... | ... |

**Environment Variable Overrides** (Spring property resolution):
- System properties (highest priority)
- Environment variables (prefixed `SPRING_`)
- application.properties (lowest priority)

Example: `spring.datasource.password` can be overridden by environment variable `SPRING_DATASOURCE_PASSWORD`

### 9.2 JNDI Resources

| Name | Type | Purpose | Configuration |
|---|---|---|---|
| jdbc/OrdersDB | DataSource | Primary ORDERS database connection | Defined in Tomcat context.xml; production: Oracle RAC instance |
| jms/OrderTopic | Topic | Kafka topic for order events (legacy JMS adapter) | Defined in Apache ActiveMQ bridge config |
| java:comp/env/mail/default | Session | SMTP session for order confirmation emails | Defined in Tomcat configuration |

**Lookup Code** (from source-extracts.md):
```java
DataSource ds = (DataSource) new InitialContext().lookup("jdbc/OrdersDB");
Connection conn = ds.getConnection();
```

### 9.3 Feature Flags & Toggles

| Flag | Purpose | Default | Components Affected |
|---|---|---|---|
| order.approval.enabled | Controls whether orders require approval before fulfillment | false (disabled) | OrderService.createOrder() checks this; if true, order status set to PENDING_APPROVAL instead of PROCESSING |
| customer.ldap.enabled | Use LDAP for customer authentication vs. local database | true (enabled) | SecurityConfiguration, authentication provider selection |
| kafka.async.enabled | Send order events to Kafka asynchronously vs. synchronously | false (synchronous) | OrderService.publishOrderCreatedEvent() — if true, publishes in separate thread; if false, blocks until published |

---

## 10. Infrastructure Dependencies

| Dependency | Type | Version | Protocol | Purpose | Fallback if Unavailable |
|---|---|---|---|---|---|
| Oracle Database | Relational Database | 19c | JDBC (thin) | Stores orders, customers, products | Hard failure — application cannot function |
| Apache Kafka | Message Broker | 2.8.x | KAFKA protocol | Event streaming for order events to fulfillment/reporting services | Hard failure — order events lost; manual intervention required |
| Redis | Cache | 6.x | TCP (Jedis client) | Caches customer details, product catalog, status counts | Graceful degradation — fallback to in-memory cache; slower performance |
| LDAP Directory | Authentication | OpenLDAP 2.4 | LDAP protocol | User authentication and role lookup | Hardcoded fallback user (for emergency access); production should have HA LDAP |
| FulfillmentService (external microservice) | HTTP API | v2.0 | REST/HTTP | Retrieves inventory, creates shipments | Circuit breaker + fallback: assume inventory available, try again later |
| SMTP Mail Server | Email | Postfix | SMTP | Sends order confirmation emails | Graceful degradation — email not sent; logged as warning; manual resend later |

---

## 11. Design Patterns & Architecture Notes

### Pattern: Template Method (Base Classes)

- **Location**: BaseService, BaseDAO — all services/DAOs extend these
- **Purpose**: Enforce consistent transaction handling, logging, and error handling across all services and DAOs
- **Implementation**: BaseService.executeInTransaction() template method calls abstract execute() implemented by subclasses
- **Modernization Recommendation**: Preserve — this pattern is valuable for cross-cutting concerns; could modernize to use Spring aspects or functional decorators instead

### Pattern: Strategy (Validation Rules)

- **Location**: ValidationStrategy interface, multiple implementations (OrderValidationStrategy, AddressValidationStrategy, etc.)
- **Purpose**: Encapsulate different validation rules and select at runtime
- **Implementation**: OrderService receives ValidationStrategy dependency, calls strategy.validate()
- **Modernization Recommendation**: Preserve — this pattern is cleaner than if-else chains; migrate to Bean Validation (JSR-380) for modern approach

### Pattern: Observer (Events)

- **Location**: ApplicationListener, event publishing via Kafka topics
- **Purpose**: Decouple OrderService from downstream systems (fulfillment, reporting)
- **Implementation**: OrderService publishes OrderCreatedEvent; listeners subscribe via Kafka consumers
- **Modernization Recommendation**: Preserve — the Kafka-based event pattern is modern; continue with async event-driven architecture

### Pattern: DAO Pattern

- **Location**: OrderDAO, OrderItemDAO, CustomerDAO
- **Purpose**: Encapsulate database access and SQL queries
- **Implementation**: DAO classes use MyBatis mappers to execute SQL
- **Modernization Recommendation**: Consider migrating from MyBatis to Spring Data JPA for reduced boilerplate, but preserve DAO abstraction layer

---

## 12. Known Technical Debt & Legacy Workarounds

| ID | Description | Location | Impact | Recommendation |
|---|---|---|---|---|
| TD-001 | Hardcoded timezone conversion to EST in OrderService.calculateDueDate() — should respect user's timezone | OrderService.java:245 | Correctness issue for international customers | Fix in modernization — store timezone in customer profile |
| TD-002 | N+1 query problem in OrderService.loadOrdersWithItems() — loads order, then loops loading each item separately | OrderService.java:167 | Performance (1000 orders = 1001 queries) | Fix — use LEFT JOIN FETCH in DAO query |
| TD-003 | Outbox table (ORDER_OUTBOX) used for reliable event delivery to Kafka, but never cleaned up | OrderDAO.java:300 | Disk space / query performance degrades over months | Add scheduled cleanup job; modern approach: use Kafka transactions instead |
| TD-004 | Manual timestamp management: created_at, updated_at columns managed in application code instead of database triggers | Multiple DAOs | Maintainability: timestamp inconsistency possible if app crashes mid-insert | Modernize: use database-managed timestamps or JPA @CreationTimestamp |
| TD-005 | Enum-like pattern for order status using VARCHAR + CHECK constraint instead of Java enum | Order.java, ORDERS table | No type safety in queries; status changes require code + DB migration coordination | Migrate to Java enum + @Enumerated JPA annotation |

---

## 13. Traceability

| Techspec Section | Funcspec Ref | Behavioral Inventory | Legacy Files |
|---|---|---|---|
| API-001: Create Order | FR-001 | B-001 (validate order total), B-002 (send Kafka event), B-003 (check customer exists) | OrderController.java:45-120, CreateOrderRequest.java, OrderService.java:50-200 |
| API-002: Approve Order | FR-003, FR-004 | B-005 (authorization check), B-006 (status transition), B-007 (publish approval event) | OrderController.java:200-250, OrderApprovalService.java |
| DB Schema: ORDERS table | FR-001, FR-003 | B-004, B-008 (audit trail) | order-tables.sql, OrderEntity.java |
| Stored Proc: PKG_ORDER.CREATE_PENDING_ORDER | FR-001 | B-001 (validation), B-009 (sequence generation) | order-procs.sql:50-120 |
| MSG-001: order.created | FR-001, FR-007 | B-002 (event publishing), B-010 (fulfillment integration) | OrderService.java:198, OrderKafkaProducer.java |
| Security: @PreAuthorize | FR-003, FR-005 | B-005 (authorization) | OrderController.java:200 (annotation), SecurityConfig.java |
| Cache: customer-details | FR-002, FR-006 | B-011 (performance optimization) | CachingConfig.java, CustomerService.java:78 |
| Transaction: OrderService.createOrder() | FR-001, FR-008 | B-012 (transaction management), B-013 (rollback on error) | OrderService.java:50 (@Transactional annotation) |
| Kafka Retry: order.created DLQ | FR-007 | B-014 (error handling), B-015 (dead letter queue routing) | KafkaConsumerConfig.java, error-handlers.properties |
| TD-002: N+1 queries | FR-001 (performance concern) | B-001, B-016 (query optimization opportunity) | OrderService.java:167 |
```

## Implementation Guidelines

1. **Read source-extracts.md completely** — build a working index of all files and their roles before reorganizing
2. **Preserve line numbers** — every detail in techspec should trace back to a specific line in legacy code
3. **Flag gaps proactively** — if source-extracts.md is missing expected content (a stored procedure body, an exception class, API schemas), explicitly flag: `⚠ INCOMPLETE — {reason}`
4. **Cross-reference obsessively** — every API should link to funcspec FRs, every FR should link back to techspec sections
5. **Trace data flows** — follow a single order through the entire system: user input → validation → service → DAO → SQL → database → response. Document the whole journey
6. **Technology accuracy** — copy exact SQL, API schemas, configuration keys, exception class names from source-extracts (no paraphrasing)
7. **Practicality for modernization** — developers will read this to understand what modern code must replicate; be detailed and precise

## Output Checklist

- [ ] techspec.md created in `modernization-output/{slug}/`
- [ ] Section 1: Overview includes technology stack and key challenges
- [ ] Section 2: Architecture diagram or flow description present
- [ ] Section 3: All API endpoints documented with complete request/response schemas
- [ ] Section 4: All tables, stored procedures, and data transformations documented
- [ ] Section 5: All messaging topics and event schemas documented
- [ ] Section 6: Exception hierarchy, retry strategies, and transaction management documented
- [ ] Section 7: Authentication flow, authorization matrix, validation rules, and sensitive data handling documented
- [ ] Section 8: Caching strategy, connection pools, and batch processing documented
- [ ] Section 9: All configuration properties and environment variables documented
- [ ] Section 10: Infrastructure dependencies and fallback strategies documented
- [ ] Section 11: Design patterns identified and recommendations provided
- [ ] Section 12: Technical debt items documented with modernization recommendations
- [ ] Section 13: Traceability matrix cross-references funcspec FRs and behavioral inventory
- [ ] Every detail traces back to source file and line number
- [ ] No gaps flagged as `⚠ INCOMPLETE` without explanation
- [ ] SQL and code samples copied verbatim from source-extracts.md (not paraphrased)
- [ ] Markdown formatting is clean and readable (tables, code blocks, emphasis used appropriately)
