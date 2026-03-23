# Modernization Prompt-Chain Architecture

## Overview

A nine-skill pipeline that takes a legacy entry point through analysis → functional spec → technical spec → JIRA stories → modernized code generation, with built-in traceability, drift detection, completeness verification, and codebase-wide orphan scanning. Each skill produces artifacts in `modernization-output/{slug}/`, and downstream skills consume the upstream output by reading those files.

```
                         PER ENTRY POINT (run for each legacy feature)
┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ analyze-entry │──▶│ gen-funcspec  │──▶│ gen-techspec  │──▶│  gen-jira     │──▶│  gen-code     │
│ (Step 1)      │   │ (Step 2)      │   │ (Step 3)      │   │ (Step 4)      │   │ (Step 5)      │
└──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
       │                   │                   │                   │                   │
       ▼                   ▼                   ▼                   ▼                   ▼
 analysis.md         funcspec.md         techspec.md         jira-stories.md    {service-name}/
 fingerprint.json                                            (traceability)     code-manifest.json
 source-extracts.md ─────────┴──────────────┘
       │
       ├──▶ contracts.json (appended — shared cross-entry-point registry)

                          ON DEMAND (per entry point)
    ┌─────────────────┐                         ┌─────────────────────┐
    │  check-drift     │                         │  verify-completeness │
    │  (Step 6)        │                         │  (Step 7)            │
    └─────────────────┘                         └─────────────────────┘
            │                                            │
            ▼                                            ▼
      drift-report.md                           completeness-report.md

                     CODEBASE-WIDE (run after all entry points analyzed)
                    ┌─────────────────┐
                    │  scan-orphans    │
                    │  (Step 8)        │
                    └─────────────────┘
                            │
                            ▼
                     orphan-report.md

┌──────────────────────────────────────────────────────────────────┐
│  modernize  (Step 0 — orchestrator)                              │
│  Runs Steps 1-5 end-to-end for a given entry point               │
│  + appends to contracts.json + runs verify-completeness          │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1. Directory & File Conventions

All artifacts for a single entry point live under a **slug directory**:

```
modernization-output/
  contracts.json               ← Shared cross-entry-point contract registry (Gap 6)
  orphan-report.md             ← Codebase-wide unclaimed code report (Gap 9)
  {slug}/
    analysis.md                ← Step 1 output (component inventory, behavioral inventory, data flow)
    fingerprint.json           ← Step 1 output (source code snapshot metadata)
    source-extracts.md         ← Step 1 output (raw technical details from every source file)
    funcspec.md                ← Step 2 output (business requirements — the "what")
    techspec.md                ← Step 3 output (engineering specification — the "how")
    jira-stories.md            ← Step 4 output
    {service-name}/            ← Step 5 output (Spring Boot 4.x microservice project)
    code-manifest.json         ← Step 5 output (maps legacy → generated files)
    drift-report.md            ← Step 6 output (on demand — legacy changed?)
    completeness-report.md     ← Step 7 output (on demand — did we miss anything?)
```

The **slug** is derived from the entry point name (e.g., `user-profile-jsp`, `order-service`, `payment-kafka-consumer`). The orchestrator (`modernize`) creates this directory and passes the path to each step.

### Cross-Entry-Point Contract Registry — `contracts.json`

When you modernize multiple entry points, they often share resources: Kafka topics, database tables, inter-service API calls. If one modernized service changes a shared message schema, it breaks the others. The contract registry tracks all shared resources across all analyzed entry points.

Each `/analyze-entry` run appends its shared resources to `contracts.json`. The `/gen-code` skill reads it before generating code to ensure it doesn't break existing contracts. The `/check-drift` skill reads it to detect contract-breaking changes.

```json
{
  "last_updated": "2026-03-22T15:00:00Z",
  "kafka_topics": {
    "order.created": {
      "producers": ["order-jsp"],
      "consumers": ["payment-service", "inventory-service"],
      "schema": {"orderId": "Long", "total": "BigDecimal", "ts": "ISO-8601"},
      "schema_source": "order-jsp/source-extracts.md §Messaging"
    }
  },
  "shared_tables": {
    "APP.ORDERS": {
      "read_by": ["order-jsp", "reporting-service"],
      "write_by": ["order-jsp"],
      "column_count": 18,
      "schema_source": "order-jsp/source-extracts.md §SQL"
    }
  },
  "api_contracts": {
    "POST /api/v1/orders": {
      "provided_by": "order-jsp",
      "consumed_by": ["mobile-app", "partner-gateway"],
      "request_schema_source": "order-jsp/source-extracts.md §API",
      "response_schema_source": "order-jsp/source-extracts.md §API"
    }
  },
  "shared_classes": {
    "com.acme.common.BaseTransactionalService": {
      "used_by": ["order-jsp", "payment-service", "inventory-service"],
      "source": "order-jsp/source-extracts.md §Shared Code"
    }
  }
}
```

When `/gen-code` generates a modernized service, it checks this registry:
- "Am I changing a Kafka schema that another service consumes?" → warn or generate a backward-compatible schema
- "Am I changing a shared DB table structure?" → generate a migration that won't break other services
- "Am I changing an API response format?" → generate versioned endpoints

### Orphan Scan — `orphan-report.md`

Run `/scan-orphans` after all entry points have been analyzed. It scans the entire codebase for files NOT claimed by any analysis and flags them.

This catches:
- REST endpoints nobody calls from the UI but external partners hit directly
- Scheduled jobs configured in the app server rather than in code
- Utility servlets/filters registered in web.xml but not referenced by any analyzed entry point
- `@KafkaListener` methods on topics no analyzed producer writes to
- Controllers, services, DAOs in the project that no entry-point trace reached

See Skill #8 (`scan-orphans`) below for full details.

---

## 2. Skill #1 — `analyze-entry`

### Purpose
Scan a legacy entry point and auto-discover **all** connected components — regardless of technology layer — producing a comprehensive analysis document. The discovery is not limited to any predefined set of layers; it follows every reference it finds.

### Trigger
`/analyze-entry <entry-point-path>`

User provides a single starting file or class. The skill traces outward through all discoverable references — imports, annotations, configuration files, SQL references, messaging bindings, API calls, scheduling, caching, security filters, etc.

### Input
- A file path or class name within the codebase
- The codebase itself (accessible on disk)

### Auto-Discovery Logic
The skill should instruct Claude to:
1. Read the entry point file
2. Follow **every** reference to another component, including but not limited to:
   - Class imports, `@Autowired`/`@Inject` dependencies, method calls
   - View templates (JSP, Thymeleaf, Freemarker, etc.)
   - Data access (MyBatis mappers, JPA repositories, JDBC calls, stored procedures)
   - Messaging (Kafka, JMS, RabbitMQ, SQS — producers and consumers)
   - REST/SOAP/gRPC endpoints and clients
   - Scheduled tasks (`@Scheduled`, Quartz, cron)
   - Caching layers (Redis, EhCache, Spring Cache annotations)
   - Security filters, interceptors, AOP aspects
   - Configuration files (Spring XML, properties, YAML, web.xml)
   - Build/deployment descriptors (pom.xml dependencies, Dockerfile, etc.)
   - Any other file or component referenced by the code
3. **Discover upstream callers (inbound pathways)** — search the entire codebase for anything that invokes, routes to, or triggers this entry point:
   - Direct method calls, class instantiation, or injection of this component
   - URL mappings, route configurations, or menu/navigation links pointing here
   - Message producers that publish to topics this entry point consumes
   - Scheduled jobs or batch processes that trigger this entry point
   - Other JSPs, controllers, or services that forward/redirect here
   - External system integrations or API gateways that call this endpoint
   - Event listeners, interceptors, or filters in the call chain
   - Any reference found via codebase-wide search (grep for class name, URL path, method name, topic name, etc.)
4. Recursively trace outbound references until no new files are discovered (with a depth limit of 5)

5. **Discover shared/common code** — for every class in the discovered set, trace its inheritance chain and shared dependencies:
   - Abstract superclasses and base classes (e.g., `BaseTransactionalService`, `AbstractDAO`, `BaseController`)
   - Utility/helper classes referenced by multiple components (`StringUtils`, `DateHelper`, `SecurityContext`)
   - Common exception classes and exception hierarchies
   - Shared DTOs, value objects, and constants classes used across layers
   - Common interceptors, filters, listeners that apply to multiple components
   - Interface definitions that multiple classes implement
   - Extract these fully even if they live outside the entry point's package — they carry critical shared logic (retry strategies, audit logging, connection management) that must be preserved in modernized code

6. **Discover legacy tests** — for each discovered production file, search for associated test files:
   - Naming conventions: `{ClassName}Test.java`, `{ClassName}Tests.java`, `{ClassName}IT.java` (integration tests), `{ClassName}Spec.groovy`
   - Test directories: `src/test/java/`, `src/test/groovy/`, `src/test/resources/`
   - Test resources: test data files, SQL scripts for test setup, mock configurations
   - For each test class, extract every test method with its assertions — these document edge cases, boundary conditions, and expected behaviors that may not be obvious from production code alone
   - Test fixtures and setup methods (`@Before`, `@BeforeEach`, `@BeforeAll`) — they reveal required preconditions
   - Integration test configurations — they show how components are wired together in testing

7. **Discover database objects not reachable from code** — for every table or schema identified during code tracing, search the database scripts directory (`/db/`, `/sql/`, `/migration/`, `/flyway/`, etc.) for associated objects:
   - **Triggers** — these fire on INSERT/UPDATE/DELETE and contain business logic invisible to application code (audit logging, cascade updates, data validation)
   - **Views and materialized views** — other systems may query these; their definitions reveal data contracts
   - **Database jobs** (DBMS_SCHEDULER, DBMS_JOB) — scheduled database-level operations
   - **Synonyms and DB links** — reveal cross-schema or cross-database dependencies
   - **Grants and permissions** — document who/what can access what
   - **Sequences** — naming, start values, increment, caching — must be preserved in migration
   - **Package bodies** (Oracle) — may contain additional procedures not directly called from app code
   - **DDL for tables** — full column definitions, constraints, default values, partitioning
   - Flag: "⚠ These objects were found in DB scripts but are NOT referenced from application code — they may be called by other systems, DB triggers, or scheduled DB jobs"
   - Flag: "⚠ If database objects are managed outside source control (by DBAs), request a full schema export"

8. **Discover implicit framework behavior** — search the entire codebase for framework hook points that affect the entry point but aren't discoverable through import/call tracing:
   - **Global exception handling**: `@ControllerAdvice`, `@ExceptionHandler`, `ErrorController` implementations
   - **Filter/interceptor chains**: `web.xml` filter ordering, `FilterRegistrationBean`, `HandlerInterceptor`, `WebMvcConfigurer` implementations, servlet listeners
   - **AOP aspects**: `@Aspect` classes — `@Before`, `@After`, `@Around` advice that wraps methods matching pointcut expressions (especially those using wildcards that silently match the entry point's methods)
   - **Bean lifecycle hooks**: `@PostConstruct`, `@PreDestroy`, `InitializingBean`, `DisposableBean`, `BeanPostProcessor`, `BeanFactoryPostProcessor`
   - **Event system**: `ApplicationListener`, `@EventListener`, `ApplicationEventPublisher` usage — events that are published or consumed as side effects
   - **Spring auto-configuration**: custom `@Configuration` classes, `@Conditional*` beans, `@Import`, `@EnableCaching`, `@EnableScheduling`, `@EnableAsync`, `@EnableTransactionManagement`
   - **Servlet container configuration**: `@WebServlet`, `@WebFilter`, `@WebListener`, `ServletContainerInitializer`
   - **Spring Security filter chain**: `SecurityFilterChain` bean definitions, custom authentication providers, custom filters inserted into the security chain
   - For each discovered hook, document: what it does, what components it affects (by pointcut/URL pattern/type), and its execution order

9. **Discover runtime/deployment configuration** — search for deployment-time settings that affect application behavior:
   - Application server configs: `server.xml`, `context.xml`, `standalone.xml` (JBoss/WildFly), `setenv.sh`
   - JNDI resource definitions (datasources, JMS connection factories, mail sessions)
   - JVM arguments and system properties
   - Container orchestration: Kubernetes manifests (Deployment, Service, ConfigMap, Secret, Ingress), Helm charts, Docker Compose files
   - CI/CD pipelines: Jenkinsfile, `.github/workflows/`, GitLab CI — reveals build steps, environment variables, deployment targets
   - External configuration: Spring Cloud Config references, Consul, Vault
   - Load balancer / reverse proxy configs (if in repo): nginx.conf, Apache httpd.conf, HAProxy
   - Environment variable references across all config files: compile a master list of every `${VAR_NAME}` / `%ENV_VAR%` found
   - Flag anything referenced but not found: "⚠ JNDI datasource `jdbc/OrdersDB` referenced in code but definition not found in repo — request from ops team"

10. **Discover conditional/profile-specific code paths** — scan for behavior that only activates under certain conditions:
    - `@Profile("production")`, `@Profile("dev")`, `@Profile("!test")` — capture the full bean/configuration for each profile
    - Profile-specific property files: `application-prod.properties`, `application-staging.yml`, etc. — capture ALL properties in each, noting what differs from the default
    - `@ConditionalOnProperty`, `@ConditionalOnBean`, `@ConditionalOnClass`, `@ConditionalOnMissingBean` — document the condition and what it activates/deactivates
    - Feature flags: programmatic checks like `if (featureFlags.isEnabled("ORDER_V2"))` — capture the full code path behind each flag
    - Environment-switching logic: `if (env.equals("PROD"))`, `System.getProperty("mode")` — capture both branches
    - Tag each of these in the Behavioral Inventory with its activation condition (e.g., "B-015 [PROD ONLY]: Send order confirmation SMS")

11. **Discover build-time code generation & external contracts** — check build configuration for generated code:
    - Scan `pom.xml` / `build.gradle` for code generation plugins: CXF (WSDL→Java), JAXB (XSD→Java), OpenAPI Generator, MapStruct, Lombok, QueryDSL, JOOQ, Protobuf, Avro
    - For each plugin found:
      - Document what it generates and from what input
      - Extract the input contract files in FULL (the WSDL, the XSD, the .proto, the .avsc, the OpenAPI spec) — these define data structures the application depends on
      - If generated source is available (in `target/generated-sources/` or checked in), extract it like any other source file
      - If generated source is NOT available, document exactly what would be generated based on the input contracts
    - Lombok: for every class using Lombok annotations (`@Data`, `@Builder`, `@Getter`, `@Setter`, `@ToString`, `@EqualsAndHashCode`, `@AllArgsConstructor`, `@NoArgsConstructor`, `@Slf4j`), expand what Lombok generates — document the effective fields, methods, constructors, builder class that exist at runtime even though they're not in the source
    - MapStruct: extract the mapper interfaces AND the generated implementations if available

12. **Exhaustive extraction (CRITICAL)** — for every discovered file (including files found in steps 5-11), perform a COMPLETE extraction with ZERO summarization. This is the only time source code is read; downstream skills rely entirely on what is captured here. Nothing should be paraphrased, abbreviated, or omitted. Specifically:

   **Classes, DTOs, Entities, Value Objects, Enums:**
   - Every field — name, type, access modifier, default value, annotations (with annotation parameters)
   - Every constructor — full parameter list with types
   - Every method — full signature (return type, name, parameters with types, throws clause, annotations)
   - Every getter/setter — especially those with custom logic beyond simple get/set (e.g., formatting, null coalescing, lazy init, computed values)
   - Class-level annotations with all parameters (e.g., `@Entity`, `@Table(name="ORDERS", schema="APP")`, `@JsonIgnoreProperties({"internalId"})`)
   - Inheritance chain — what it extends, what interfaces it implements
   - Inner classes and static nested classes — fully extracted as if they were top-level
   - Enum values with any associated fields, constructors, and methods
   - Constants — name, type, value, where referenced
   - Static initializer blocks and instance initializer blocks

   **Validation & Constraints:**
   - Every annotation on every field: `@NotNull`, `@Size(min=1, max=255)`, `@Pattern(regexp="...")`, `@Email`, custom validators — capture the exact parameters
   - Programmatic validation logic in methods (if/throw patterns, validate() methods)
   - Custom constraint validators — the full validation logic, not just the annotation

   **Serialization & Mapping:**
   - Jackson annotations: `@JsonProperty("wire_name")`, `@JsonFormat`, `@JsonIgnore`, `@JsonInclude`, custom serializers/deserializers
   - MyBatis type handlers, result maps, discriminators
   - Any manual mapping logic (e.g., `toDTO()`, `fromEntity()`, builder patterns)
   - XML binding annotations (JAXB) if present

   **SQL & Database:**
   - Complete SQL text for every query (not paraphrased)
   - Full stored procedure bodies (not summaries)
   - Every column in SELECT, INSERT, UPDATE statements
   - WHERE clause logic fully preserved
   - JOIN conditions in full
   - Subqueries and CTEs in full
   - Index definitions, constraint definitions
   - Trigger bodies in full
   - Sequence definitions

   **Configuration:**
   - Every property key-value pair from .properties/.yml files
   - Every `@Value("${...}")` injection with default values
   - Every web.xml entry (servlet, filter, listener)
   - Every Spring XML bean definition
   - Every environment variable reference

   **JSP / View Templates:**
   - All form fields with names, types, validation attributes
   - All display fields and their data bindings
   - Conditional rendering logic (JSTL `<c:if>`, `<c:choose>`)
   - JavaScript embedded in JSPs — function names, AJAX calls, event handlers
   - All included/imported fragments

   **Test Files:**
   - Every test method — name, annotations (`@Test`, `@ParameterizedTest`, `@Disabled`), assertions
   - Test data and fixtures — what scenario each test covers
   - Mock setups — what is mocked and what behavior is defined
   - Integration test wiring — `@SpringBootTest`, `@DataJpaTest`, `@WebMvcTest` configurations

   The rule is: **if it's in the source file, it's in source-extracts.md.** When in doubt, include it. A DTO with 45 fields gets all 45 fields captured. A stored procedure with 200 lines gets all 200 lines captured. A properties file with 80 entries gets all 80 entries. A test class with 30 test methods gets all 30 methods captured.

13. **Self-validation pass** — after completing the extraction, perform a verification sweep:
    a. For each file in the discovered set, confirm that source-extracts.md has a corresponding section
    b. For each class, confirm that every field and method is listed (compare count of fields/methods extracted vs. the actual count in the source file)
    c. For each SQL/PLSQL file, confirm the complete body is captured (not truncated)
    d. For each properties/config file, confirm every key is captured
    e. For each test file, confirm every test method is listed
    f. For each Lombok-annotated class, confirm generated methods are documented
    g. Verify that every shared/common class discovered in step 5 has a full extraction
    h. Verify that every DB object found in step 7 has its full body captured
    i. Verify that every framework hook found in step 8 has its full implementation captured
    j. Flag any file that appears to have incomplete extraction as `⚠ INCOMPLETE — {reason}`
    k. Report a summary: "{N} files extracted, {M} fully validated, {K} flagged incomplete"

14. **Categorize organically** — group discovered components by their actual role (presentation, business logic, data access, messaging, security, configuration, testing, shared/common, framework hooks, deployment, etc.) rather than forcing them into predetermined buckets
15. **Generate a fingerprint manifest** — for every source file touched during analysis, record a content hash and structural summary (see fingerprint.json below). This is the foundation for drift detection.

### Output 1 — `fingerprint.json`

Generated alongside analysis.md. This is the traceability anchor that all downstream artifacts reference and that drift detection uses to identify changes.

```json
{
  "generated_at": "2026-03-22T14:30:00Z",
  "git_commit": "a1b2c3d4e5f6...",
  "git_branch": "main",
  "git_repo_clean": false,
  "uncommitted_changes": ["src/main/java/com/acme/service/OrderService.java"],
  "entry_point": "src/main/webapp/WEB-INF/views/order.jsp",
  "slug": "order-jsp",
  "files": {
    "src/main/webapp/WEB-INF/views/order.jsp": {
      "sha256": "e3b0c44298fc1c149afbf4c8996fb924...",
      "last_modified": "2026-03-20T10:15:00Z",
      "role": "Presentation Layer",
      "key_signatures": ["form:submitOrder", "include:orderHeader.jsp"],
      "lines": 245
    },
    "src/main/java/com/acme/service/OrderService.java": {
      "sha256": "d7a8fbb307d7809469ca9abcb0082e4f...",
      "last_modified": "2026-03-19T08:22:00Z",
      "role": "Business Services",
      "key_signatures": ["method:createOrder(OrderDTO)", "method:validateOrder(Order)", "@Transactional"],
      "lines": 312
    }
  },
  "file_count": 14,
  "total_lines": 4230
}
```

Field details:

- **git_commit** — exact commit hash at generation time; lets you `git diff` against it later
- **git_branch** — the branch analyzed (important if you have feature branches diverging)
- **git_repo_clean** — whether there were uncommitted changes at analysis time (if true, the commit hash alone isn't a perfect snapshot)
- **uncommitted_changes** — lists any dirty files so you know the fingerprint includes unstaged work
- **files.{path}.sha256** — content hash of each file; the primary drift detection mechanism
- **files.{path}.key_signatures** — structural fingerprint: method names, annotations, form actions, SQL procedure calls, topic names, etc. When a file changes, comparing old vs new signatures tells you *what kind* of change it was (new method added? existing method renamed? annotation removed?)
- **files.{path}.role** — which category this file was grouped into in analysis.md, so drift reports can say "a Data Access file changed" not just "a file changed"

### Output 2 — `analysis.md`
```markdown
# Code Analysis: {entry-point-name}

## Entry Point
- File: {path}
- Type: {detected type}
- Description: {what it does}

## Discovered Components

(One section per discovered layer/category — sections are created dynamically
based on what is actually found, not from a fixed list. Examples:)

### {Layer Name} (e.g., "Presentation Layer", "Business Services", "Data Access", etc.)
| File/Component | Purpose | Key Elements | Connected To |
|----------------|---------|-------------|-------------|
| ... | ... | ... | ... |

(Repeat for every distinct category of component discovered.
If only 2 layers exist, show 2. If 10 exist, show 10.)

## Inbound Pathways (Who Calls This Entry Point)

A detailed inventory of every pathway that leads into this entry point.

| # | Caller | Type | Mechanism | Trigger Condition | Data Passed | Frequency | Notes |
|---|--------|------|-----------|-------------------|-------------|-----------|-------|
| 1 | {file/component} | {e.g., JSP link, Controller forward, Kafka producer, Scheduler, API gateway, Menu nav} | {e.g., HTTP redirect, method call, message publish, cron trigger} | {When/why this pathway is invoked} | {Parameters, payload, or context passed in} | {High/Medium/Low or specific schedule} | {Any relevant context — conditional logic, feature flags, deprecated, etc.} |
| 2 | ... | ... | ... | ... | ... | ... | ... |

(Include EVERY caller found — direct and indirect. If a caller is itself called
by something else, note that in the Notes column to show the full chain.
If no callers are found, explicitly state that and flag it as a potential
orphaned component.)

## Data Flow Diagram
{Mermaid or textual description of how data flows through ALL discovered components, including inbound pathways}

## Business Logic Summary
{High-level description of what the feature/module does}

## Technology Inventory
| Technology | Usage | Components |
|-----------|-------|-----------|
| {e.g., Spring MVC} | {Controller routing} | {list of files} |
| {e.g., Oracle PLSQL} | {Stored procedures} | {list of files} |
(Captures every technology/framework encountered during discovery)

## Behavioral Inventory

An exhaustive checklist of every discrete behavior, business rule, code path,
and side effect found in the legacy code. This is the completeness baseline —
the verify-completeness skill checks the modernized code against this list.

| ID | Behavior / Rule | Source File(s) | Type | Details |
|----|----------------|---------------|------|---------|
| B-001 | {e.g., "Validate order total > 0 before submit"} | OrderService.java:142 | Validation | {logic description, edge cases} |
| B-002 | {e.g., "Send Kafka event on order creation"} | OrderService.java:198 | Side Effect | {topic, payload shape, conditions} |
| B-003 | {e.g., "Return 403 if user lacks ROLE_ADMIN"} | OrderController.java:55 | Security | {annotation, filter, or manual check} |
| B-004 | {e.g., "Retry DB insert up to 3 times on deadlock"} | OrderDAO.java:88 | Error Handling | {retry logic, exception types} |
| B-005 | {e.g., "Cache order lookup for 5 min"} | OrderService.java:67 | Performance | {cache key, TTL, eviction} |
| ... | ... | ... | ... | ... |

Types include but are not limited to: Validation, Business Rule, Side Effect,
Security, Error Handling, Performance, Data Transformation, UI Behavior,
Integration, Scheduling, Logging/Audit, Configuration.

Each entry should be specific enough that a developer can write a test for it.

## Dependencies & Risks
- External system dependencies
- Shared database tables / state
- Cross-module coupling
- Hardcoded values or configuration
- Undocumented or implicit dependencies
```

### Output 3 — `source-extracts.md`

A comprehensive dump of every raw technical detail found while reading the source code during discovery. This is the analyze step's "working notes" — all the engineering-level details captured in one pass so that downstream skills (funcspec, techspec) never need to go back to the source code.

The analyze step already reads every file while tracing components. Rather than discarding the technical details and only keeping summaries for analysis.md, it preserves everything here.

```markdown
# Source Extracts: {entry-point-name}

## File: {filepath}
**Role**: {category from analysis.md}
**Lines**: {count}
**Extraction Validated**: ✅ {field-count} fields, {method-count} methods captured (matches source)

### Class / Interface / Enum Definition
```
@Entity
@Table(name = "ORDERS", schema = "APP",
       uniqueConstraints = @UniqueConstraint(columnNames = {"order_number"}))
@JsonIgnoreProperties(ignoreUnknown = true)
public class Order extends BaseEntity implements Auditable {
```
- **Extends**: `BaseEntity`
- **Implements**: `Auditable`
- **Type**: Entity / DTO / Value Object / Service / etc.

### Fields (ALL fields — no omissions)
| # | Field | Type | Access | Default | Annotations (with params) | Line | Notes |
|---|-------|------|--------|---------|--------------------------|------|-------|
| 1 | `id` | `Long` | private | — | `@Id`, `@GeneratedValue(strategy=SEQUENCE, generator="order_seq")`, `@Column(name="ORDER_ID")` | 12 | PK |
| 2 | `orderNumber` | `String` | private | — | `@Column(name="ORDER_NUMBER", length=20, unique=true, nullable=false)`, `@NotBlank`, `@Pattern(regexp="ORD-[0-9]{6}")` | 15 | Business key |
| 3 | `customerId` | `Long` | private | — | `@Column(name="CUSTOMER_ID", nullable=false)`, `@NotNull` | 18 | FK to CUSTOMERS |
| 4 | `orderDate` | `Date` | private | `new Date()` | `@Column(name="ORDER_DATE")`, `@Temporal(TemporalType.TIMESTAMP)`, `@JsonFormat(pattern="yyyy-MM-dd'T'HH:mm:ss")` | 21 | Auto-set on creation |
| 5 | `status` | `OrderStatus` | private | `PENDING` | `@Enumerated(EnumType.STRING)`, `@Column(name="STATUS", length=30)` | 24 | See Enum: OrderStatus |
| 6 | `totalAmount` | `BigDecimal` | private | `BigDecimal.ZERO` | `@Column(name="TOTAL_AMOUNT", precision=12, scale=2)` | 27 | Computed in service |
| 7 | `items` | `List<OrderItem>` | private | `new ArrayList<>()` | `@OneToMany(mappedBy="order", cascade=CascadeType.ALL, orphanRemoval=true)`, `@JsonManagedReference` | 30 | Child collection |
| ... | (EVERY remaining field) | ... | ... | ... | ... | ... | ... |

**Total fields: {exact count} — verified against source**

### Enums (fully expanded)
```java
public enum OrderStatus {
    PENDING("P", "Pending Review"),
    PENDING_APPROVAL("PA", "Awaiting Manager Approval"),
    APPROVED("A", "Approved"),
    REJECTED("R", "Rejected"),
    CANCELLED("C", "Cancelled");

    private final String code;
    private final String displayName;

    OrderStatus(String code, String displayName) { ... }
    public String getCode() { return code; }
    public String getDisplayName() { return displayName; }

    public static OrderStatus fromCode(String code) {
        // reverse lookup logic
    }
}
```

### Constructors
| # | Parameters | Annotations | Line | Notes |
|---|-----------|------------|------|-------|
| 1 | `()` (no-arg) | — | 40 | Required by JPA |
| 2 | `(Long customerId, List<OrderItem> items)` | — | 42 | Business constructor |

### Methods (ALL methods — no omissions)
| # | Signature | Visibility | Annotations | Line | Purpose | Custom Logic? |
|---|-----------|-----------|------------|------|---------|--------------|
| 1 | `Long getId()` | public | — | 55 | Standard getter | No |
| 2 | `void setId(Long id)` | public | — | 56 | Standard setter | No |
| 3 | `BigDecimal getTotalAmount()` | public | `@JsonProperty("total")` | 62 | Getter with JSON rename | Yes — alias |
| 4 | `void addItem(OrderItem item)` | public | — | 70 | Add to collection | Yes — sets bidirectional reference: `item.setOrder(this)` then `items.add(item)` |
| 5 | `void removeItem(OrderItem item)` | public | — | 78 | Remove from collection | Yes — clears reference: `item.setOrder(null)` then `items.remove(item)` |
| 6 | `BigDecimal calculateTotal()` | public | — | 85 | Computes total | Yes — `items.stream().map(i -> i.getPrice().multiply(BigDecimal.valueOf(i.getQuantity()))).reduce(BigDecimal.ZERO, BigDecimal::add)` |
| 7 | `boolean isHighValue()` | public | — | 92 | Business rule | Yes — `totalAmount.compareTo(new BigDecimal("10000")) > 0` |
| ... | (EVERY remaining method) | ... | ... | ... | ... | ... |

**Total methods: {exact count} — verified against source**

### Inner Classes / Nested Types
(Fully extracted as if they were top-level classes — same field/method/annotation detail)

### Constants
| Name | Type | Value | Visibility | Used By |
|------|------|-------|-----------|---------|
| `MAX_ITEMS` | `int` | `50` | public static final | `validateOrder()` at line 120 |
| `STATUS_PENDING` | `String` | `"PENDING"` | public static final | Deprecated — use enum instead |

### Serialization / Mapping Notes
- Custom `@JsonSerialize(using = MoneySerializer.class)` on `totalAmount`
- `@JsonIgnore` on `internalNotes` field — not exposed in API response
- MyBatis resultMap `orderResultMap` maps `ORDER_ID → id`, `ORDER_NUMBER → orderNumber`, etc.
- Manual `toDTO()` method at line 150: maps entity to `OrderDTO`, excludes `internalNotes`, formats `orderDate` as ISO-8601

---

### SQL / Data Operations
```sql
-- Source: line 88, called via MyBatis mapper OrderMapper.insertOrder
INSERT INTO ORDERS (id, customer_id, total, status, created_at)
VALUES (#{id}, #{customerId}, #{total}, 'PENDING', SYSDATE)
```

```sql
-- Source: line 102, stored procedure call
{call PROCESS_ORDER(#{orderId, mode=IN}, #{result, mode=OUT, jdbcType=VARCHAR})}
```

### Stored Procedure Bodies (if accessible)
```sql
-- PROCESS_ORDER (from db/procedures/PROCESS_ORDER.sql)
CREATE OR REPLACE PROCEDURE PROCESS_ORDER(
  p_order_id IN NUMBER,
  p_result OUT VARCHAR2
) AS
  v_total NUMBER;
BEGIN
  SELECT total INTO v_total FROM ORDERS WHERE id = p_order_id;
  IF v_total > 10000 THEN
    -- requires manager approval
    UPDATE ORDERS SET status = 'PENDING_APPROVAL' WHERE id = p_order_id;
    p_result := 'PENDING_APPROVAL';
  ELSE
    UPDATE ORDERS SET status = 'APPROVED' WHERE id = p_order_id;
    INSERT INTO ORDER_HISTORY (order_id, action, ts) VALUES (p_order_id, 'AUTO_APPROVED', SYSDATE);
    p_result := 'APPROVED';
  END IF;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    p_result := 'ERROR_NOT_FOUND';
  WHEN OTHERS THEN
    ROLLBACK;
    p_result := 'ERROR_' || SQLERRM;
END;
```

### API Endpoint Details
| Method | Path | Consumes | Produces | Params | Headers |
|--------|------|----------|----------|--------|---------|
| POST | /api/orders | application/json | application/json | — | Authorization, X-Request-ID |

**Request Schema** (extracted from DTO class):
```json
{
  "customerId": "Long, required",
  "items": [{"productId": "Long", "quantity": "Integer, min=1"}],
  "shippingAddress": {"street": "String", "city": "String", "zip": "String, pattern=[0-9]{5}"}
}
```

**Response Schema**:
```json
{
  "orderId": "Long",
  "status": "String, enum=[PENDING, PENDING_APPROVAL, APPROVED]",
  "createdAt": "ISO-8601 timestamp"
}
```

### Error Handling
| Exception | Caught At | Action | HTTP Status | Message |
|-----------|----------|--------|-------------|---------|
| `ValidationException` | Controller:62 | Return error response | 400 | Field-level errors |
| `InsufficientInventoryException` | Service:95 | Rollback + return | 409 | "Item {id} out of stock" |
| `DeadlockLoserDataAccessException` | DAO:110 | Retry up to 3x, exponential backoff | 500 (if exhausted) | "Please retry" |

### Messaging
| Direction | Topic | Key | Schema | Consumer Group | DLQ |
|-----------|-------|-----|--------|---------------|-----|
| Produce | `order.created` | orderId | `{"orderId":Long,"total":BigDecimal,"ts":ISO-8601}` | — | — |
| Consume | `payment.completed` | orderId | `{"orderId":Long,"paymentId":String,"status":String}` | `order-service-group` | `payment.completed.dlq` |

### Security
| Check | Type | Location | Details |
|-------|------|----------|---------|
| `@PreAuthorize("hasRole('ORDER_CREATE')")` | Role-based | Controller:40 | Spring Security annotation |
| CSRF token validation | Filter | web.xml filter chain | Custom `CsrfFilter` class |

### Configuration
| Property | Value/Pattern | Source | Used At |
|----------|-------------|--------|---------|
| `order.max-items` | `50` | application.properties | Service:22 |
| `order.approval-threshold` | `10000` | application.properties | Service:78 |
| `spring.datasource.url` | `jdbc:oracle:thin:@${DB_HOST}:1521:${DB_SID}` | env variable | application.yml |

### Transaction Boundaries
| Method | Isolation | Propagation | Timeout | Rollback For |
|--------|-----------|-------------|---------|-------------|
| `createOrder()` | READ_COMMITTED | REQUIRED | 30s | `RuntimeException.class` |
| `processPayment()` | SERIALIZABLE | REQUIRES_NEW | 10s | `PaymentException.class` |

### Caching
| Cache Name | Key | TTL | Eviction | Backend |
|-----------|-----|-----|----------|---------|
| `orders` | `order:{id}` | 5 min | LRU | EhCache |

---

(Repeat the above structure for EVERY file discovered during analysis.
Not every file will have all sections — include only what's found.)

## Extraction Validation Summary
| File | Fields Found | Fields Extracted | Methods Found | Methods Extracted | SQL Statements | Config Props | Status |
|------|-------------|-----------------|--------------|------------------|---------------|-------------|--------|
| OrderService.java | 5 | 5 | 12 | 12 | 3 | — | ✅ Complete |
| OrderDTO.java | 45 | 45 | 48 | 48 | — | — | ✅ Complete |
| OrderMapper.xml | — | — | — | — | 8 | — | ✅ Complete |
| application.properties | — | — | — | — | — | 80 | ✅ Complete |
| OrderDAO.java | 3 | 3 | 8 | 7 | 5 | — | ⚠ INCOMPLETE — method `bulkInsert()` at line 201 contains dynamic SQL that could not be fully resolved |

**Overall: {N} files extracted, {M} complete, {K} incomplete**
If any file is marked incomplete, the reason is documented so that gen-techspec and gen-code
know where they may need to request manual clarification.
```

This file can be large — that's expected. It's a reference document, not something that needs to fit in a single prompt. Downstream skills (funcspec, techspec) read the sections they need.

---

## 3. Skill #2 — `gen-funcspec`

### Purpose
Read the analysis.md and source-extracts.md from Step 1 and generate a functional specification / requirements document for the modernized version of the feature.

### Trigger
`/gen-funcspec <slug>`

Where `<slug>` is the directory name under `modernization-output/`.

### Input
- `modernization-output/{slug}/analysis.md` (required — component inventory, behavioral inventory, data flow)
- `modernization-output/{slug}/source-extracts.md` (required — raw technical details; funcspec reads this for business rules, data schemas, and integration contracts but translates them into technology-agnostic requirements)

### What It Does
The skill reads the analysis and translates legacy implementation details into forward-looking functional requirements. It should:

1. Extract business rules embedded in the legacy code (from the analysis)
2. Separate concerns: what the feature *does* vs. how the legacy code *implements* it
3. Define functional requirements in a technology-agnostic way
4. Identify data entities and their relationships
5. Note integration points (APIs, events, external systems)
6. Call out edge cases and error handling observed in the legacy code
7. Flag areas where modernization could improve UX, performance, or maintainability

### Output — `funcspec.md`
```markdown
# Functional Specification: {feature-name}

## 1. Overview
{What this feature does from a business perspective}

## 2. Actors & Personas
{Who uses this feature and in what context}

## 3. Functional Requirements

### FR-001: {Requirement Title}
- **Description**: ...
- **Business Rule**: ...
- **Acceptance Criteria**:
  - Given ... When ... Then ...
- **Source**: {which legacy component this was derived from}

### FR-002: ...
(repeat)

## 4. Data Model
{Entities, attributes, relationships — derived from DB tables/PLSQL params}

## 5. Integration Points
| System | Direction | Protocol | Purpose |
|--------|-----------|----------|---------|
| ... | ... | ... | ... |

## 6. Non-Functional Considerations
- Performance expectations
- Security requirements
- Audit/logging needs

## 7. Migration Notes
- Data migration considerations
- Feature parity gaps
- Suggested phasing (what can be modernized incrementally)

## 8. Open Questions
{Anything ambiguous in the legacy code that needs stakeholder input}
```

---

## 4. Skill #3 — `gen-techspec`

### Purpose
Read the analysis.md and funcspec.md and generate a deep technical specification that documents every implementation detail of the legacy system — data structures, algorithms, SQL logic, API contracts, error handling strategies, concurrency patterns, configuration, and infrastructure dependencies. This is the engineering blueprint that gen-code uses to produce accurate modernized code.

### Why This Exists (Funcspec vs Techspec)
The funcspec answers **"what does the system do?"** from a business perspective — it's technology-agnostic and written for product owners and architects. The techspec answers **"how does the legacy code actually implement it?"** at an engineering level — it's written for developers who need to understand every technical decision before they can write equivalent modern code.

Without the techspec, gen-code would have to guess at implementation details: How does the connection pooling work? What isolation level do the transactions use? What's the retry strategy? What exact SQL does the stored procedure run? The techspec removes that guesswork.

### Trigger
`/gen-techspec <slug>`

### Input
- `modernization-output/{slug}/analysis.md` (required — component inventory + behavioral inventory)
- `modernization-output/{slug}/source-extracts.md` (required — **primary source**; contains all raw technical details already extracted during analysis: SQL, API schemas, error handling, config, caching, messaging, security, transactions)
- `modernization-output/{slug}/funcspec.md` (required — to cross-reference functional requirements)
- The legacy codebase is **NOT re-read** — source-extracts.md already captured everything during the analyze step. This avoids redundant source code scanning.

### What It Does
The skill reads source-extracts.md (the raw technical dump) and organizes, cross-references, and enriches it into a structured engineering specification. Unlike the analyze step which extracts data file-by-file, the techspec reorganizes by concern — all API contracts together, all SQL together, all error handling together — giving developers a coherent technical reference.

It should:

1. **Reorganize source-extracts.md by technical domain** — the extracts are organized per-file; the techspec reorganizes by concern (all APIs, all SQL, all messaging, etc.)
2. **Document API contracts in full** — consolidate all endpoint details from source-extracts into a complete API reference with request/response schemas, status codes, headers, pagination patterns
3. **Consolidate SQL/data logic** — bring together all queries, stored procedure bodies, DDL, indexes, constraints, triggers from across multiple files into a unified data model section
4. **Map data flows end-to-end** — using the per-file extracts, trace from user input to database write and back, connecting the dots across files that source-extracts presents separately
5. **Document error handling exhaustively** — consolidate all exception handling from source-extracts into a unified error strategy view
6. **Capture concurrency and transaction patterns** — consolidate transaction boundaries, isolation levels, locking strategies from source-extracts into a coherent picture
7. **Document messaging contracts** — consolidate all messaging details into a unified events/messaging reference
8. **Consolidate security implementation** — bring together all security checks, auth flows, sanitization from source-extracts into a unified security section
9. **Catalog configuration and environment dependencies** — merge all properties, environment variables, JNDI resources, feature flags from source-extracts into a single reference
10. **Document performance characteristics** — consolidate caching, connection pools, thread pools, batch sizes, timeouts from source-extracts
11. **Identify design patterns in use** — analyze the code structures captured in source-extracts to identify architectural patterns (decorator, strategy, observer, etc.) and document why they exist
12. **Cross-reference to funcspec** — every technical detail should trace back to the functional requirement it supports
13. **Flag gaps in source-extracts** — if the techspec identifies missing details that source-extracts should have captured (e.g., a referenced stored procedure whose body wasn't found), document them as open items

### Output — `techspec.md`
```markdown
# Technical Specification: {feature-name}

## 1. Overview
{Brief technical summary — what this system does and how, at an engineering level}

## 2. Architecture Summary
{Component diagram showing how the legacy pieces connect technically}
{Sequence diagrams for key flows}

## 3. API Contracts

### API-001: {Endpoint Name}
- **URL**: `{method} {path}` (e.g., `POST /api/v1/orders`)
- **Authentication**: {mechanism — e.g., session cookie, JWT Bearer, API key}
- **Request Headers**: {required/optional headers}
- **Request Body**:
  ```json
  {exact schema with field types, constraints, and descriptions}
  ```
- **Response 200**:
  ```json
  {exact response schema}
  ```
- **Response 400/403/404/500**: {each error response with schema and conditions}
- **Rate Limiting**: {if any}
- **Funcspec Ref**: FR-001
- **Source**: {legacy file:line}

### API-002: ...
(repeat for every endpoint)

## 4. Data Model & Storage

### 4.1 Database Schema
For each table:
- **Table**: `{SCHEMA.TABLE_NAME}`
- **Columns**:
  | Column | Type | Nullable | Default | Constraints | Description |
  |--------|------|----------|---------|------------|-------------|
  | ... | ... | ... | ... | ... | ... |
- **Indexes**: {name, columns, type (unique/btree/etc.)}
- **Foreign Keys**: {source → target}
- **Triggers**: {name, event, logic summary}

### 4.2 Stored Procedures / Functions
For each procedure:
- **Name**: `{SCHEMA.PROCEDURE_NAME}`
- **Parameters**: {name, type, direction (IN/OUT/INOUT)}
- **Logic** (step by step):
  1. {what the procedure does, in order}
  2. {including conditional branches}
  3. {and error handling}
- **Called By**: {which DAO/service invokes this}
- **SQL Statements**: {the key queries it executes, paraphrased with logic explained}
- **Transaction Behavior**: {commits, rollbacks, savepoints}

### 4.3 Data Transformations
| Step | Input | Output | Transformation Logic | Location |
|------|-------|--------|---------------------|----------|
| 1 | {raw user input} | {validated DTO} | {validation rules applied} | Controller |
| 2 | {validated DTO} | {entity} | {mapping logic} | Service |
| ... | ... | ... | ... | ... |

## 5. Messaging & Event Contracts

### MSG-001: {Topic/Queue Name}
- **Topic**: `{exact topic name}`
- **Direction**: Produce / Consume / Both
- **Serialization**: {JSON, Avro, Protobuf}
- **Message Schema**:
  ```json
  {exact payload structure with field types}
  ```
- **Partition Key**: {field or strategy}
- **Consumer Group**: `{group-id}`
- **Offset Strategy**: {earliest, latest, specific}
- **DLQ**: {dead letter queue name and routing logic}
- **Retry Policy**: {max retries, backoff strategy}
- **Ordering Guarantees**: {within partition, global, none}
- **Source**: {legacy file:line}

## 6. Error Handling & Resilience

### 6.1 Exception Hierarchy
| Exception Type | Caught Where | Action | HTTP Status | User Message |
|---------------|-------------|--------|-------------|-------------|
| ... | ... | ... | ... | ... |

### 6.2 Retry Strategies
| Operation | Max Retries | Backoff | Retriable Exceptions | Fallback |
|-----------|------------|---------|---------------------|----------|
| ... | ... | ... | ... | ... |

### 6.3 Circuit Breakers / Bulkheads
{If any patterns exist for fault isolation}

### 6.4 Transaction Management
| Boundary | Isolation Level | Propagation | Rollback On | Timeout |
|----------|----------------|-------------|-------------|---------|
| ... | ... | ... | ... | ... |

## 7. Security Implementation

### 7.1 Authentication Flow
{Step-by-step: how a user/service authenticates}

### 7.2 Authorization Matrix
| Resource/Action | Required Roles/Permissions | Check Location | Enforcement |
|----------------|--------------------------|----------------|-------------|
| ... | ... | ... | {annotation, filter, manual check} |

### 7.3 Input Validation & Sanitization
| Input Field | Validation Rules | Sanitization | Location |
|------------|-----------------|-------------|----------|
| ... | ... | ... | ... |

### 7.4 Sensitive Data Handling
{Encryption at rest, in transit, PII masking, audit logging for sensitive access}

## 8. Performance & Caching

### 8.1 Caching Strategy
| Cache | Key Pattern | TTL | Eviction | Backend | Invalidation Trigger |
|-------|-----------|-----|----------|---------|---------------------|
| ... | ... | ... | ... | ... | ... |

### 8.2 Connection Pools & Thread Pools
| Pool | Min | Max | Timeout | Idle Timeout | Configuration Source |
|------|-----|-----|---------|-------------|---------------------|
| ... | ... | ... | ... | ... | ... |

### 8.3 Batch Processing
{Batch sizes, chunk processing, parallel execution, scheduling}

## 9. Configuration & Environment

### 9.1 Properties & Environment Variables
| Property/Variable | Purpose | Default | Sensitive | Where Used |
|------------------|---------|---------|-----------|-----------|
| ... | ... | ... | yes/no | ... |

### 9.2 JNDI Resources
| Name | Type | Purpose | Legacy Configuration |
|------|------|---------|---------------------|
| ... | ... | ... | ... |

### 9.3 Feature Flags & Toggles
| Flag | Purpose | Default State | Components Affected |
|------|---------|--------------|-------------------|
| ... | ... | ... | ... |

## 10. Infrastructure Dependencies
| Dependency | Type | Version | Protocol | Purpose | Fallback if Unavailable |
|-----------|------|---------|----------|---------|------------------------|
| ... | {DB, cache, queue, service, LDAP, SMTP, etc.} | ... | ... | ... | ... |

## 11. Design Patterns & Architecture Notes
{Document any specific patterns in use: Strategy for validation, Observer for events,
Template Method in base classes, etc. Explain WHY the pattern was chosen and whether
the modernized code should preserve or replace it.}

## 12. Known Technical Debt & Legacy Workarounds
| Item | Description | Location | Recommendation |
|------|------------|----------|---------------|
| TD-001 | {e.g., "Hardcoded timezone conversion to EST"} | OrderService.java:245 | {Carry forward / Fix in modernization} |
| TD-002 | {e.g., "N+1 query in order list pagination"} | OrderDAO.java:88 | {Fix — use JPA fetch join} |
| ... | ... | ... | ... |

## 13. Traceability
| Techspec Section | Funcspec Ref | Behavioral Inventory | Legacy Files |
|-----------------|-------------|---------------------|-------------|
| API-001 | FR-001 | B-001, B-003 | OrderController.java, order.jsp |
| MSG-001 | FR-007 | B-002 | OrderKafkaProducer.java |
| ... | ... | ... | ... |
```

---

## 5. Skill #4 — `gen-jira`

### Purpose
Take the analysis.md (Step 1), funcspec.md (Step 2), and techspec.md (Step 3) and produce well-structured JIRA stories for a modernization implementation.

### Trigger
`/gen-jira <slug>`

### Input
- `modernization-output/{slug}/analysis.md` (required)
- `modernization-output/{slug}/funcspec.md` (required)
- `modernization-output/{slug}/techspec.md` (required)

### What It Does
The skill reads all three upstream artifacts and generates stories that:

1. Map functional requirements to implementable stories
2. Include deep technical context from the techspec (API contracts, SQL logic, messaging schemas) so developers have precise implementation guidance — not just "modernize the order service" but "replace PROCESS_ORDER stored procedure with JPA repository using these exact queries and transaction boundaries"
3. Create an Epic → Story hierarchy
4. Estimate relative complexity (S/M/L/XL) informed by the techspec's detail on transaction complexity, integration points, and technical debt
5. Define acceptance criteria tied to both the functional spec AND the technical spec
6. Include migration/cutover stories (not just net-new development)
7. Add dependencies between stories where ordering matters
8. Reference specific techspec sections (API-001, MSG-001, TD-001) in each story's technical notes

### Output — `jira-stories.md`
```markdown
# JIRA Stories: {feature-name} Modernization

## Traceability Header
- **Generated**: {timestamp}
- **Git Commit**: `{hash}` (branch: `{branch}`)
- **Fingerprint**: See `fingerprint.json` for full file-level hashes
- **Analysis Source**: `modernization-output/{slug}/analysis.md`
- **Funcspec Source**: `modernization-output/{slug}/funcspec.md`
- **⚠ Drift Check**: Run `/check-drift {slug}` to verify legacy code hasn't changed since generation

## Epic: {Feature Name} Modernization
**Description**: Modernize {feature} from legacy stack to {target stack}.
**Labels**: modernization, {module-name}

---

### Story 1: {Title}
- **Type**: Story
- **Size**: M
- **Description**: ...
- **Acceptance Criteria**:
  - [ ] ...
  - [ ] ...
- **Technical Notes**: Replaces legacy {component} — see analysis.md §{section}
- **Depends On**: (none | Story N)
- **Functional Spec Ref**: FR-001, FR-002
- **Legacy Source Files**: {list of specific files from fingerprint.json this story touches}
  - `src/main/java/com/acme/service/OrderService.java` (sha256: `d7a8fbb3...`)
  - `src/main/java/com/acme/dao/OrderMapper.java` (sha256: `9f86d081...`)

### Story 2: {Title}
...

---

## Suggested Sprint Breakdown
| Sprint | Stories | Theme |
|--------|---------|-------|
| 1 | Stories 1-3 | Data layer + API foundation |
| 2 | Stories 4-6 | Core business logic |
| 3 | Stories 7-9 | UI + integration |
| 4 | Story 10 | Migration + cutover |

## Legacy File → Story Mapping
(Reverse index: for each legacy file, which stories are affected if it changes)

| Legacy File | Stories | Role |
|------------|---------|------|
| `src/.../OrderService.java` | Story 1, Story 3 | Business Services |
| `src/.../OrderMapper.java` | Story 2 | Data Access |
| ... | ... | ... |
```

---

## 6. Skill #5 — `gen-code`

### Purpose
Generate a modernized Spring Boot 4.x codebase from the analysis, functional specification, technical specification, and JIRA stories. The techspec is the primary driver for implementation details — it tells gen-code exactly how the legacy system works at an engineering level so the modernized code is precise rather than inferred.

### Trigger
`/gen-code <slug> <service-name>`

- **slug** — the modernization-output directory for this entry point (e.g., `order-jsp`)
- **service-name** — the name of the microservice to generate (e.g., `order-service`, `payment-api`, `inventory-ms`). This becomes the project folder name, the Maven/Gradle artifact ID, the base package suffix, and the Docker image name.

Run this after JIRA stories have been reviewed and approved, or as part of the full `/modernize` pipeline.

### Input
- **service-name** (required) — drives all naming conventions:
  - Project folder: `modernization-output/{slug}/{service-name}/`
  - Maven artifactId: `{service-name}`
  - Base package: `com.acme.{service-name-as-package}` (e.g., `com.acme.orderservice`)
  - Docker image: `{service-name}:latest`
  - Spring application name: `{service-name}`
- `modernization-output/{slug}/analysis.md` (legacy component inventory + behavioral inventory)
- `modernization-output/{slug}/fingerprint.json` (legacy file list and signatures)
- `modernization-output/{slug}/funcspec.md` (functional requirements — the "what")
- `modernization-output/{slug}/techspec.md` (technical specification — the "how"; **primary source** for implementation details: API contracts, SQL logic, messaging schemas, error handling, transaction boundaries, security, caching, configuration)
- `modernization-output/{slug}/jira-stories.md` (implementation stories with traceability — defines scope and ordering)

### Target Stack — Spring Boot 4.x

The generated code targets **Spring Boot 4.0+** running on **Spring Framework 7.0**. The skill should encode the following platform decisions:

**Runtime & Language**
- Java 17+ (with Java 25 idioms where beneficial — records, sealed classes, pattern matching)
- Spring Boot 4.0 with Spring Framework 7.0
- Jakarta EE 9+ (jakarta.* namespace — NOT javax.*)
- Build: Maven or Gradle (configurable, default Maven)

**Web & API Layer**
- Spring MVC or Spring WebFlux (skill should decide based on legacy patterns — if legacy is synchronous servlet-based, default to MVC; if event-driven, consider WebFlux)
- HTTP Interface Clients (new in Spring Boot 4.x) for consuming external REST APIs — replaces RestTemplate/WebClient boilerplate
- API Versioning support via `spring.mvc.apiversion.*` properties where applicable
- OpenAPI 3.x spec generation via springdoc

**Data Access**
- Spring Data JPA with Hibernate 7 (replaces MyBatis mappers)
- Flyway or Liquibase for schema migration scripts (generated from PLSQL/DDL analysis)
- Repository pattern with custom query methods derived from legacy DAO SQL
- For complex queries that don't map cleanly to JPA: keep as native queries with `@Query`, document the legacy SQL source

**Messaging**
- Spring Kafka (if legacy uses Kafka) with proper serializer/deserializer configuration
- Spring JMS / Spring AMQP (if legacy uses JMS/RabbitMQ)
- CloudEvents format for event payloads where appropriate

**Security**
- Spring Security 7.0 (consolidated support in Boot 4.x)
- Map legacy security filters/interceptors to Spring Security filter chain
- OAuth2/OIDC resource server where legacy used custom auth

**Observability**
- OpenTelemetry starter (`spring-boot-starter-opentelemetry`) for metrics and tracing
- Structured logging with correlation IDs
- Actuator health endpoints

**Testing**
- JUnit 5 + Spring Boot Test
- RestTestClient (new in Boot 4.x) for controller tests
- Testcontainers for database and messaging integration tests
- One test class per generated service/controller, with tests derived from the Behavioral Inventory

### Generation Logic

The skill instructs Claude to generate code story-by-story, maintaining traceability:

**Phase 1 — Project Scaffolding**
1. Generate the project skeleton: `pom.xml` (or `build.gradle`), `application.yml`, package structure
2. Configure Spring Boot 4.x dependencies matching the technology inventory from analysis.md
3. Set up profiles: `local`, `dev`, `staging`, `prod`
4. Generate Dockerfile and docker-compose.yml for local development

**Phase 2 — Data Layer (per story)**
For each JIRA story touching data access:
1. Read the legacy DAO/mapper/PLSQL referenced in the story's `Legacy Source Files`
2. Generate JPA entity from the table structure (inferred from legacy SQL or PLSQL params)
3. Generate Spring Data repository interface with custom query methods
4. Generate Flyway migration script if schema changes are needed
5. Generate integration test using Testcontainers
6. Record mapping in `code-manifest.json`

**Phase 3 — Service Layer (per story)**
For each JIRA story touching business logic:
1. Read the legacy service class referenced in the story
2. Generate Spring `@Service` class implementing equivalent business logic
3. Map each behavior from the Behavioral Inventory (B-xxx) to a method or code block
4. Add `// Legacy: B-001 — Validate order total > 0` comments linking back to inventory
5. Generate unit tests covering each mapped behavior
6. Record mapping in `code-manifest.json`

**Phase 4 — API/Controller Layer (per story)**
For each JIRA story touching REST endpoints or UI:
1. Read the legacy controller/JSP/REST resource
2. Generate Spring MVC `@RestController` with proper request/response DTOs
3. Use HTTP Interface Clients for any outbound REST calls
4. Apply API versioning if the funcspec calls for it
5. Generate RestTestClient-based controller tests
6. Generate OpenAPI annotations
7. Record mapping in `code-manifest.json`

**Phase 5 — Messaging & Integration (per story)**
For each JIRA story touching messaging or external systems:
1. Generate Kafka listener/producer beans (or JMS/AMQP equivalents)
2. Define event payload DTOs using records
3. Generate integration tests with embedded broker (Testcontainers)
4. Record mapping in `code-manifest.json`

**Phase 6 — Security & Cross-Cutting**
1. Generate Spring Security 7.0 configuration from legacy security analysis
2. Map interceptors/filters to the SecurityFilterChain
3. Generate AOP aspects for cross-cutting concerns (logging, audit)
4. Configure OpenTelemetry and Actuator

**Phase 7 — Verification Hooks**
1. Generate a `BehaviorCoverageTest` — a test class that has one `@Test` method for every Behavioral Inventory item (B-001 through B-NNN), initially as `@Disabled` stubs with the behavior description. Developers enable each test as they verify the implementation.
2. Add a Gradle/Maven task or test tag that reports behavioral coverage percentage

### Output 1 — `{service-name}/`

A complete, buildable Spring Boot 4.x microservice project, named after the service:

```
modernization-output/{slug}/{service-name}/     ← e.g., order-service/
├── pom.xml (or build.gradle)
│   ├── groupId: com.acme
│   ├── artifactId: {service-name}              ← e.g., order-service
│   └── name: {service-name}
├── Dockerfile
├── docker-compose.yml
├── src/
│   ├── main/
│   │   ├── java/com/acme/{servicepkg}/         ← e.g., com.acme.orderservice
│   │   │   ├── config/          ← Security, Kafka, OpenTelemetry config
│   │   │   ├── controller/      ← @RestController classes
│   │   │   ├── service/         ← @Service classes
│   │   │   ├── repository/      ← Spring Data JPA repositories
│   │   │   ├── entity/          ← JPA entities
│   │   │   ├── dto/             ← Request/response DTOs (Java records)
│   │   │   ├── event/           ← Kafka/messaging event classes
│   │   │   ├── client/          ← HTTP Interface Client interfaces
│   │   │   └── aspect/          ← AOP cross-cutting concerns
│   │   └── resources/
│   │       ├── application.yml
│   │       │   └── spring.application.name: {service-name}
│   │       ├── application-local.yml
│   │       └── db/migration/    ← Flyway scripts
│   └── test/
│       └── java/com/acme/{servicepkg}/
│           ├── controller/      ← RestTestClient tests
│           ├── service/         ← Unit tests
│           ├── repository/      ← Testcontainers integration tests
│           └── BehaviorCoverageTest.java  ← Inventory-linked test stubs
└── README.md                    ← Generated — explains what was generated and from where

{servicepkg} = service-name with hyphens removed (e.g., "order-service" → "orderservice")
```

### Output 2 — `code-manifest.json`

Maps every generated file back to its legacy source, JIRA story, funcspec requirement, and behavioral inventory items. This is the key traceability artifact that `verify-completeness` uses.

```json
{
  "generated_at": "2026-03-22T15:00:00Z",
  "target_stack": "Spring Boot 4.0.x / Spring Framework 7.0 / Java 17+",
  "slug": "order-jsp",
  "service_name": "order-service",
  "base_package": "com.acme.orderservice",
  "artifact_id": "order-service",
  "mappings": [
    {
      "generated_file": "order-service/src/main/java/com/acme/orderservice/service/OrderService.java",
      "legacy_files": [
        "src/main/java/com/acme/service/OrderService.java",
        "db/procedures/PROCESS_ORDER.sql"
      ],
      "stories": ["Story 1", "Story 3"],
      "funcspec_refs": ["FR-001", "FR-003"],
      "behaviors_covered": ["B-001", "B-002", "B-004"],
      "generation_phase": "Phase 3 — Service Layer"
    },
    {
      "generated_file": "order-service/src/main/java/com/acme/orderservice/controller/OrderController.java",
      "legacy_files": [
        "src/main/webapp/WEB-INF/views/order.jsp",
        "src/main/java/com/acme/controller/OrderController.java"
      ],
      "stories": ["Story 4"],
      "funcspec_refs": ["FR-005", "FR-006"],
      "behaviors_covered": ["B-003", "B-008"],
      "generation_phase": "Phase 4 — API/Controller Layer"
    }
  ],
  "behaviors_not_mapped": ["B-012"],
  "stories_not_generated": [],
  "total_generated_files": 24,
  "total_test_files": 12,
  "behavioral_coverage": "95%"
}
```

### Generation Principles

The skill should encode these principles to avoid common code-generation pitfalls:

1. **Translate intent, not syntax** — don't mechanically convert Java → Java. Understand what the legacy code achieves and write idiomatic Spring Boot 4.x code that achieves the same thing. A 200-line JSP might become a 30-line REST controller + a React component.

2. **Favor Spring Boot conventions** — use auto-configuration, starters, and convention-over-configuration rather than manual wiring. If Spring Boot 4.x has a built-in way to do something (like HTTP Interface Clients for REST calls), use it instead of replicating legacy patterns.

3. **Keep legacy comments** — every generated method/class should have a comment linking back to the Behavioral Inventory ID and legacy source file, so developers can trace decisions back to their origin.

4. **Generate tests from behaviors, not from code** — tests should validate the behaviors in the Behavioral Inventory, not mirror the structure of the generated code. This ensures tests catch regressions in *what the code does*, not *how it's structured*.

5. **Don't over-generate** — if a JIRA story covers infrastructure (CI/CD pipeline, monitoring dashboards), generate a TODO placeholder rather than guessing. Only generate application code, tests, and configuration.

6. **Make it buildable** — the generated project should compile and pass its tests (at least the enabled ones) out of the box. If something requires manual configuration (database credentials, Kafka broker addresses), use placeholders with clear `TODO` markers in application.yml.

---

## 7. Skill #6 — `check-drift`

### Purpose
Compare the current state of legacy source files against the fingerprint.json snapshot to detect what has changed since the modernization artifacts were generated. Produces an actionable drift report that tells you which JIRA stories need attention.

### Trigger
`/check-drift <slug>`

Run this anytime — after a sprint, before starting a story, as part of a CI check, or on a schedule.

### Input
- `modernization-output/{slug}/fingerprint.json` (required)
- `modernization-output/{slug}/jira-stories.md` (required — for the file→story reverse mapping)
- The current codebase on disk
- The git repository (for `git diff` and `git log`)

### Detection Logic
The skill should instruct Claude to:
1. Read `fingerprint.json` to get the baseline commit, file list, and per-file hashes
2. For each file in the fingerprint:
   a. Check if the file still exists (deletion = critical drift)
   b. Compute the current SHA-256 and compare against the stored hash
   c. If changed: run `git diff {baseline-commit} HEAD -- {filepath}` to get the actual diff
   d. Compare current `key_signatures` against stored ones to classify the change
3. Search for **new files** that weren't in the original fingerprint but are in the same packages/directories (a new DAO in the same package = potential missed scope)
4. Check `git log {baseline-commit}..HEAD -- {filepath}` for each changed file to get commit messages explaining why it changed
5. Cross-reference changes against the **Legacy File → Story Mapping** from jira-stories.md to identify impacted stories
6. Classify each change by impact level

### Impact Classification

| Level | Meaning | Examples | Action Needed |
|-------|---------|----------|--------------|
| **CRITICAL** | Breaks assumptions in JIRA stories | New/removed method that a story depends on, table schema change, deleted file, changed API contract | Story must be updated before implementation |
| **SIGNIFICANT** | Alters business logic but stories may still be valid | Modified validation rules, changed SQL query logic, new error handling | Review story; may need AC updates |
| **MINOR** | Cosmetic or additive changes unlikely to affect modernization | Logging changes, comment updates, formatting, added metrics | Acknowledge but likely no action |
| **NEW SCOPE** | New files/components appeared in the same module | New service class, new Kafka consumer, new stored procedure in the same package | Evaluate whether modernization scope should expand |

### Output — `drift-report.md`
```markdown
# Drift Report: {slug}

## Summary
- **Baseline**: commit `{hash}` on `{branch}` at {timestamp}
- **Current**: commit `{current-hash}` at {current-timestamp}
- **Commits since baseline**: {count}
- **Files analyzed**: {count}
- **Files changed**: {count}
- **Files deleted**: {count}
- **New files detected**: {count}

## Overall Drift Status: {🟢 CLEAN | 🟡 MINOR DRIFT | 🟠 SIGNIFICANT DRIFT | 🔴 CRITICAL DRIFT}

## Changed Files Detail

### 🔴 CRITICAL Changes

#### {filepath}
- **What changed**: {description of the change}
- **Git commits**: {commit hash + message for each relevant commit}
- **Signature diff**: {what was added/removed/modified in key_signatures}
- **Impacted stories**: Story 1, Story 3
- **Recommended action**: {specific guidance — e.g., "Story 1 acceptance criteria #2 references OrderService.validateOrder() which now has a new required parameter"}

### 🟠 SIGNIFICANT Changes
(same structure)

### 🟡 MINOR Changes
(same structure, abbreviated)

### 🆕 New Scope
| New File | Location | Probable Role | Recommended Action |
|----------|----------|--------------|-------------------|
| ... | ... | ... | {Expand scope / Ignore / Investigate} |

## Story Impact Summary

| Story | Status | Changed Files | Highest Impact | Action |
|-------|--------|--------------|---------------|--------|
| Story 1 | 🔴 Affected | OrderService.java, OrderMapper.java | CRITICAL | Must revise before implementation |
| Story 2 | 🟢 Clean | (none) | — | Ready to implement |
| Story 3 | 🟡 Minor | OrderController.java | MINOR | Review but likely OK |
| ... | ... | ... | ... | ... |

## Recommended Next Steps
1. {Prioritized list of actions based on findings}
2. {e.g., "Re-run /analyze-entry and /gen-jira for this slug to regenerate stories"}
3. {e.g., "Only Story 1 needs regeneration — others are still valid"}
```

---

## 8. Skill #7 — `verify-completeness`

### Purpose
After modernized code has been written (either manually or generated), verify that it covers every behavior, business rule, integration, and code path from the legacy system. Produces a completeness report with gap analysis.

### The Problem This Solves
Modernization failures typically happen in three ways:
1. **Discovery gaps** — the analysis missed a legacy component entirely (a hidden stored procedure, an undocumented scheduled job)
2. **Translation gaps** — the analysis found it, the funcspec described it, but the JIRA story didn't capture it (or captured it incorrectly)
3. **Implementation gaps** — the story was correct but the modernized code doesn't fully implement it (missed an edge case, forgot an error handler, dropped a side effect)

This skill catches all three by working from the ground truth — the actual legacy code — and checking the actual modernized code against it.

### Trigger
`/verify-completeness <slug> <path-to-modernized-code>`

Run this after implementing some or all of the JIRA stories, or as a final gate before decommissioning legacy code.

### Input
- `modernization-output/{slug}/analysis.md` (the Behavioral Inventory)
- `modernization-output/{slug}/fingerprint.json` (the file list and signatures)
- `modernization-output/{slug}/funcspec.md` (the functional requirements)
- `modernization-output/{slug}/jira-stories.md` (the stories with legacy file mappings)
- The legacy codebase (original source files)
- The modernized codebase (new source files at the provided path)

### Verification Logic
The skill instructs Claude to perform a multi-pass verification:

**Pass 1 — Behavioral Coverage (Legacy → Modern)**
For each entry in the Behavioral Inventory (B-001, B-002, ...):
1. Read the legacy source at the cited location to confirm the behavior exists
2. Search the modernized codebase for equivalent implementation
3. Classify: COVERED (found equivalent), PARTIAL (found but incomplete), MISSING (not found), INTENTIONALLY SKIPPED (documented as out of scope)
4. For COVERED items, note where in the modernized code it lives

**Pass 2 — Structural Coverage (Files → Modern)**
For each file in fingerprint.json:
1. Extract all public methods, endpoints, SQL operations, message handlers
2. Check whether each has a modernized counterpart
3. Flag any legacy method/endpoint that has no equivalent in the new code

**Pass 3 — Requirement Traceability (Funcspec → Stories → Code)**
For each functional requirement (FR-001, FR-002, ...):
1. Verify it maps to at least one JIRA story
2. Verify that story maps to actual modernized code
3. Check that acceptance criteria from the story are testable against the new code

**Pass 4 — Reverse Check (Modern → Legacy)**
Search the modernized code for logic that has NO corresponding legacy behavior:
1. This catches accidental additions, hallucinated requirements, or scope creep
2. New behaviors should be intentional and documented — flag anything unexplained

**Pass 5 — Re-scan Legacy for Missed Behaviors**
Re-read each legacy source file looking for behaviors NOT already in the Behavioral Inventory:
1. Hidden conditional branches, catch blocks with business logic, default cases
2. Configuration-driven behavior (properties files, feature flags)
3. Implicit behaviors (framework conventions, annotation side effects)
4. If new behaviors are found, flag them as Discovery Gaps

### Output — `completeness-report.md`
```markdown
# Completeness Report: {slug}

## Summary
- **Generated**: {timestamp}
- **Legacy behaviors catalogued**: {count from Behavioral Inventory}
- **Behaviors covered in modern code**: {count} ({percentage}%)
- **Behaviors partially covered**: {count}
- **Behaviors missing**: {count}
- **Discovery gaps (missed by analysis)**: {count}
- **Unexplained new behaviors in modern code**: {count}

## Overall Completeness: {🟢 >95% | 🟡 80-95% | 🟠 60-80% | 🔴 <60%}

## Traceability Matrix

| Behavior ID | Legacy Description | Legacy Location | Funcspec Ref | Story Ref | Modern Location | Status | Notes |
|------------|-------------------|----------------|-------------|-----------|----------------|--------|-------|
| B-001 | Validate order total > 0 | OrderService.java:142 | FR-003 | Story 2 | NewOrderService.ts:88 | 🟢 COVERED | — |
| B-002 | Send Kafka event on creation | OrderService.java:198 | FR-007 | Story 5 | OrderEventPublisher.ts:34 | 🟢 COVERED | Now uses CloudEvents format |
| B-003 | Return 403 without ROLE_ADMIN | OrderController.java:55 | FR-012 | Story 4 | middleware/auth.ts:22 | 🟢 COVERED | — |
| B-004 | Retry DB insert 3x on deadlock | OrderDAO.java:88 | FR-003 | Story 2 | — | 🔴 MISSING | No retry logic found in modern code |
| B-005 | Cache order lookup 5 min | OrderService.java:67 | — | — | — | 🟡 PARTIAL | Caching exists but TTL is 15min, not 5min |
| ... | ... | ... | ... | ... | ... | ... | ... |

## Discovery Gaps (Behaviors Missed by Original Analysis)

| Gap ID | Behavior Found | Legacy Location | Severity | Recommendation |
|--------|---------------|----------------|----------|---------------|
| DG-001 | {e.g., "Audit log written to AUDIT_LOG table on every order update"} | OrderDAO.java:201 | HIGH | Add to funcspec, create new story |
| DG-002 | {e.g., "Feature flag ORDER_V2_ENABLED gates new validation path"} | OrderService.java:150 | MEDIUM | Decide if new code needs equivalent toggle |
| ... | ... | ... | ... | ... |

## Structural Coverage

### Legacy Methods Without Modern Equivalent
| Legacy File | Method/Endpoint | Purpose | Risk |
|------------|----------------|---------|------|
| OrderService.java | cancelOrder(Long id) | Cancels order and reverses inventory | HIGH — user-facing action not modernized |
| ... | ... | ... | ... |

### Modern Code Without Legacy Equivalent
| Modern File | Method/Endpoint | Explanation Needed |
|------------|----------------|-------------------|
| NewOrderService.ts | archiveOrder() | Not in legacy — is this intentional new scope? |
| ... | ... | ... |

## Requirement Traceability Gaps

| Funcspec Req | Has Story? | Has Modern Code? | Gap Type |
|-------------|-----------|-----------------|----------|
| FR-008 | ❌ No | — | Translation gap — requirement never became a story |
| FR-011 | ✅ Story 7 | ❌ No | Implementation gap — story exists but not implemented |
| ... | ... | ... | ... |

## Recommended Actions
1. {Prioritized list — e.g., "Implement retry logic for B-004 — this is a data integrity concern"}
2. {e.g., "Create new JIRA story for DG-001 (audit logging) — compliance requirement"}
3. {e.g., "Verify archiveOrder() in modern code is intentional with product owner"}
4. {e.g., "Re-run /analyze-entry to update Behavioral Inventory with newly discovered gaps"}
```

---

## 9. Skill #8 — `scan-orphans`

### Purpose
After all entry points have been individually analyzed, scan the entire codebase for files and components that were NOT discovered by any entry-point analysis. These "orphans" are code that's still deployed and potentially still in use — by external systems, DB triggers, app server configurations, or manual processes — but invisible to the entry-point tracing approach.

This is the safety net that catches everything the entry-point-driven analysis model inherently cannot reach.

### Trigger
`/scan-orphans`

Run this after you've completed `/analyze-entry` for all known entry points. It's a codebase-wide sweep, not per-entry-point.

### Input
- All existing `modernization-output/{slug}/fingerprint.json` files (to know which files have already been claimed)
- The complete codebase on disk

### Detection Logic
1. Build the "claimed files" set — union of all files across all fingerprint.json manifests
2. Scan the entire codebase for Java files, JSP files, SQL files, config files, XML files
3. For every file NOT in the claimed set, analyze it to determine if it's active or truly dead:

   **Indicators of active orphaned code:**
   - Has `@RestController`, `@Controller`, `@Path` → uncovered REST/web endpoint
   - Has `@KafkaListener`, `@JmsListener`, `@RabbitListener` → uncovered message consumer
   - Has `@Scheduled`, `@Quartz` → uncovered scheduled task
   - Registered in `web.xml` as a servlet, filter, or listener
   - Referenced in Spring XML config but not in any analyzed Java code
   - Has `public static void main(String[])` → standalone utility/batch program
   - Is a stored procedure/function in `/db/` that isn't called from any analyzed DAO
   - Has recent git commits (actively maintained = likely still used)

   **Indicators of truly dead code:**
   - No git commits in 2+ years
   - Has `@Deprecated` annotation
   - Commented out or in a `_deprecated/` or `_old/` directory
   - No references from any file in the codebase (including unclaimed files)

4. For active orphans, perform the same exhaustive extraction as analyze-entry step 12

### Output — `orphan-report.md`
```markdown
# Orphan Scan Report

## Summary
- **Total codebase files**: {count}
- **Files claimed by entry-point analyses**: {count} across {N} slugs
- **Unclaimed files**: {count}
- **Active orphans (likely still in use)**: {count}
- **Dead code (likely safe to ignore)**: {count}
- **Uncertain (needs manual review)**: {count}

## 🔴 Active Orphans — Likely Still In Use

### {filepath}
- **Type**: {REST endpoint | Kafka consumer | Scheduled job | Servlet | Stored procedure | ...}
- **Evidence of use**: {recent git commits, web.xml registration, topic has messages, etc.}
- **What it does**: {brief analysis}
- **Risk if missed**: {what breaks if this isn't modernized}
- **Recommended action**: {Create a new /analyze-entry for this as its own entry point | Add to existing slug X | Consult with team}

### ...
(repeat for each active orphan)

## 🟡 Uncertain — Needs Manual Review

| File | Last Modified | Indicators | Possible Purpose |
|------|-------------|-----------|-----------------|
| ... | ... | ... | ... |

## 🟢 Dead Code — Likely Safe to Ignore

| File | Last Modified | Reason Classified as Dead |
|------|-------------|--------------------------|
| ... | ... | {No references, deprecated, no git activity since 2022, etc.} |

## Recommended Next Steps
1. For each active orphan: run `/analyze-entry` to bring it into the pipeline
2. For uncertain files: review with team leads who know the history
3. For dead code: consider removing from the codebase (separate cleanup effort)
4. After addressing active orphans: re-run `/scan-orphans` to confirm full coverage
```

---

## 10. Orchestrator — `modernize`

### Purpose
Run all five core steps end-to-end for a given entry point, creating the slug directory and chaining the outputs through to generated code.

### Trigger
`/modernize <entry-point-path> <service-name>`

- **entry-point-path** — the legacy file to start from
- **service-name** — the microservice name for the generated project (e.g., `order-service`)

### Execution Flow
```
1. Derive slug from entry-point name
2. Create modernization-output/{slug}/
3. Run analyze-entry → saves analysis.md + fingerprint.json + source-extracts.md
   (includes: shared code, test discovery, DB objects, framework hooks,
    deployment config, profile-specific paths, build-time codegen)
4. Append shared resources to modernization-output/contracts.json
5. Run gen-funcspec → reads analysis.md + source-extracts.md, saves funcspec.md
6. Run gen-techspec → reads analysis.md + source-extracts.md + funcspec.md (NO re-reading of source code), saves techspec.md
7. Run gen-jira → reads analysis.md + funcspec.md + techspec.md, saves jira-stories.md
8. Run gen-code with {service-name} → reads all upstream artifacts + contracts.json (for cross-service compatibility), saves {service-name}/ + code-manifest.json
9. Run verify-completeness → reads everything including {service-name}/, saves completeness-report.md
10. Report completion with links to all artifacts and completeness summary
```

### Post-Pipeline: Codebase-Wide Checks
After running `/modernize` for ALL entry points:
```
11. Run scan-orphans → reads all fingerprint.json files, scans full codebase, saves orphan-report.md
12. For each active orphan found → run /modernize on it as a new entry point
13. Re-run scan-orphans to confirm full coverage
```

### Error Handling
- If no code files are found at the entry point → stop with a clear message
- If analysis discovers no connected components → warn but continue
- If funcspec.md would be empty → stop and ask user for clarification
- If contracts.json shows a shared resource conflict → warn and require confirmation before proceeding

---

## 11. Implementation Plan: Skills vs Commands vs Agents

### What Goes Where

| Component | Type | Location | Why |
|-----------|------|----------|-----|
| `analyze-entry` | **Skill** | `.claude/skills/code-analysis/SKILL.md` | Complex 15-step discovery + exhaustive extraction + fingerprinting. The heaviest skill in the pipeline. |
| `gen-funcspec` | **Skill** | `.claude/skills/funcspec-writer/SKILL.md` | Needs detailed output templates and domain knowledge about writing good functional specs. |
| `gen-techspec` | **Skill** | `.claude/skills/techspec-writer/SKILL.md` | Reorganizes source-extracts.md by concern — needs heuristics for cross-referencing and pattern recognition. |
| `gen-jira` | **Skill** | `.claude/skills/jira-modernize/SKILL.md` | Needs templates for story structure, sizing heuristics, traceability fields, and sprint planning guidance. |
| `gen-code` | **Skill** | `.claude/skills/code-generator/SKILL.md` + `references/` | Complex code generation requiring Spring Boot 4.x patterns, phased generation logic, contract registry awareness, and traceability. |
| `check-drift` | **Skill** | `.claude/skills/drift-detector/SKILL.md` | Complex comparison logic, impact classification heuristics, and cross-referencing against stories. |
| `verify-completeness` | **Skill** | `.claude/skills/completeness-checker/SKILL.md` | Multi-pass verification with behavioral matching, structural comparison, and traceability analysis. |
| `scan-orphans` | **Skill** | `.claude/skills/orphan-scanner/SKILL.md` | Codebase-wide scan requiring dead-code heuristics and classification logic. |
| `modernize` | **Command** | `.claude/commands/modernize.md` | Orchestrator — chains the core skills per entry point. |
| `scan-orphans` | **Command** | `.claude/commands/scan-orphans.md` | Codebase-wide orchestrator — runs after all entry points are analyzed. |

### Why Skills for the Core Eight

Skills are the right choice because each step involves substantial domain expertise — tracing code, extracting details, writing specs, decomposing into stories, generating code, detecting drift, verifying completeness, classifying orphans — that benefits from persistent, detailed instructions. The skill body acts as a "playbook" that Claude follows each time, and the progressive disclosure model (metadata → SKILL.md → reference files) keeps things efficient.

### Why Commands for Orchestrators

The orchestrators are procedural glue: "run A, then B, then C." They don't need deep domain instructions — they just need the sequence, file paths, and error handling. Commands are simpler to write and maintain for this pattern.

### How Chaining Works

Claude Code doesn't have a built-in "pipeline" primitive, but the chain works naturally through file I/O:

1. Each skill writes its output to a known path (`modernization-output/{slug}/{filename}`)
2. The next skill reads from that same path
3. Shared state goes in `modernization-output/contracts.json` (cross-entry-point)
4. The orchestrator command tells Claude to run each skill in sequence

### Agent Usage

For the orchestrator, if running in Claude Code (not Cowork), each step can be spawned as a **subagent** for isolation. In practice, the chain is sequential (each step depends on the prior), so subagents are mainly useful for:
- Running multiple entry points in parallel (`/modernize` across several features simultaneously)
- Running `scan-orphans` as a background task after all entry points complete
- The eval/test loop (running with-skill vs without-skill baselines)

---

## 12. Next Steps

1. **Build the eight skills** — Write SKILL.md for `code-analysis`, `funcspec-writer`, `techspec-writer`, `jira-modernize`, `code-generator`, `drift-detector`, `completeness-checker`, and `orphan-scanner`
2. **Build code-generator reference files** — Create `references/spring-boot-4-patterns.md`, `references/project-templates.md` for the gen-code skill
3. **Build the orchestrator commands** — Write `modernize.md` and `scan-orphans.md` in `.claude/commands/`
4. **Create skill directories** — `.claude/skills/techspec-writer/`, `.claude/skills/code-generator/`, `.claude/skills/drift-detector/`, `.claude/skills/completeness-checker/`, `.claude/skills/orphan-scanner/`
5. **Create test cases** — Pick 2-3 representative entry points from the codebase (when available) and run the chain
6. **Test exhaustive extraction** — Verify source-extracts.md captures ALL fields from a complex DTO, ALL SQL from stored procedures, ALL properties from config files
7. **Test code generation** — Verify generated project compiles, tests pass, and code-manifest.json accurately maps legacy → modern
8. **Test drift detection** — Make a small change to a legacy file, run `/check-drift`, verify it correctly identifies the change and impacted stories
9. **Test completeness** — Run `/verify-completeness` against generated code and verify it catches intentionally omitted behaviors
10. **Test orphan scan** — Intentionally exclude an entry point, run `/scan-orphans`, verify it flags the unclaimed code
11. **Iterate** — Use the skill-creator eval loop to refine each skill based on output quality
12. **Optional: CI integration** — Run `/check-drift` on a schedule or git hook; run `/verify-completeness` as a PR gate; run `/scan-orphans` periodically to catch new orphaned code
