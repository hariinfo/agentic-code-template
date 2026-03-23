---
name: code-analysis
description: |
  Comprehensive legacy code analysis and auto-discovery skill. Analyze and trace a single entry
  point (JSP, REST endpoint, service, DAO, PLSQL stored procedure, Kafka topic, Quartz job, etc.)
  and auto-discover ALL connected legacy components across every layer - presentation, business
  logic, data access, messaging, caching, security, configuration, and testing. Produces three
  artifacts: fingerprint.json (git commit + per-file content hashes + structural signatures for
  drift detection), analysis.md (component inventory, inbound pathways, behavioral inventory,
  technology stack), and source-extracts.md (exhaustive raw technical dump with ZERO
  summarization - every field, method, SQL statement, config property extracted in complete
  detail). Appends shared resources to contracts.json for cross-entry-point analysis.
---

# Code Analysis Skill (analyze-entry)

## Purpose

Scan a single legacy entry point and auto-discover **all** connected components regardless of technology layer or architectural depth. Produce three comprehensive artifacts:

1. **fingerprint.json** — Git commit info + file content hashes + structural fingerprints for downstream drift detection
2. **analysis.md** — Component inventory organized by discovered role, inbound pathways (who calls this entry point), behavioral inventory (complete checklist of every rule and behavior), technology inventory, and data flow
3. **source-extracts.md** — Complete raw technical details for every discovered file: every field, method, SQL statement, stored procedure body, annotation, configuration property, test method, JSP tag — ZERO summarization

Also appends any newly discovered shared resources to `contracts.json` for tracking cross-entry-point resource usage.

## Trigger

```
/analyze-entry <entry-point-path>
```

Where `<entry-point-path>` is any file within the legacy codebase:
- JSP file: `src/main/webapp/WEB-INF/views/order.jsp`
- Service class: `src/main/java/com/acme/service/OrderService.java`
- DAO/Repository: `src/main/java/com/acme/dao/OrderDAO.java`
- REST Controller: `src/main/java/com/acme/controller/OrderController.java`
- Kafka producer/consumer: `src/main/java/com/acme/kafka/OrderEventProducer.java`
- Stored procedure: `db/procedures/PROCESS_ORDER.sql`
- MyBatis mapper: `src/main/resources/mappers/OrderMapper.xml`
- Scheduled task: `src/main/java/com/acme/batch/OrderBatchJob.java`

## Auto-Discovery Logic (15 Steps)

The skill must follow this rigorous discovery process:

### Step 1: Read Entry Point
Read the provided file fully. Identify its type (JSP, Service, DAO, Controller, Mapper XML, stored procedure, test class, config file, etc.), its purpose, and all its direct dependencies.

### Step 2: Trace Outbound References (Depth 5)
Follow every explicit reference from the entry point:
- **Class imports** — what other classes does this file reference?
- **Dependency injection** — `@Autowired`, `@Inject`, constructor parameters
- **Method calls** — which methods are invoked and from which classes?
- **JSP directives** — `<%@ include %>`, `<%@ taglib %>`, `<jsp:include>`, `<jsp:forward>`
- **View template references** — `ModelAndView("viewName")`, `@RequestMapping` path references
- **Data access** — MyBatis mapper references, JPA repository calls, JDBC queries, stored procedure names
- **Messaging** — Kafka topics produced/consumed, JMS destinations, RabbitMQ exchanges
- **Configuration** — `@Value("${property.name}")`, XML bean references, property file imports
- **REST clients** — `RestTemplate`, `WebClient`, `@FeignClient` interfaces
- **Caching** — `@CacheConfig`, `@Cacheable` cache names, Redis keys
- **Security** — `@PreAuthorize`, `@Secured`, security context references
- **Scheduling** — `@Scheduled(cron="...")`, Quartz job references
- **Testing** — `@SpringBootTest`, `@DataJpaTest`, mock bean definitions

Continue recursively to depth 5, adding newly discovered files to the working set until no new files are found.

### Step 3: Discover Inbound Pathways
Search the entire codebase for anything that invokes, routes to, or triggers this entry point:
- **Direct method calls** — grep for `className.methodName(` and class instantiation `new ClassName()`
- **Dependency injection** — grep for `@Autowired`, `@Bean(class=ClassName)`
- **URL mappings and routing** — grep for controller request paths that map to this handler
- **Navigation and menu links** — JSP/HTML `<a href>` or form actions pointing to this component
- **Message producers** — Kafka/JMS producers publishing to topics this entry point consumes
- **Event listeners** — `@EventListener`, `ApplicationListener` implementations that listen for events this produces
- **Scheduled triggers** — Quartz jobs, Scheduler beans that invoke methods in this component
- **Servlet filters and interceptors** — filter chains that apply to this component's URL patterns
- **AOP aspects** — `@Aspect` pointcuts that match this component's methods
- **Framework hooks** — `@ControllerAdvice` exception handlers that catch exceptions from this component
- **API gateway references** — reverse proxy routing to this endpoint
- **External system integrations** — webhooks, callbacks, or API calls from external systems

Document every pathway found with: who calls it, what mechanism (HTTP, method call, message, event, etc.), what data is passed, and under what conditions.

### Step 4: Discover Shared Code (Inheritance & Common Utilities)
For every class in the discovered set, trace its inheritance and dependencies on common utilities:
- **Superclasses** — `extends BaseService`, `extends AbstractDAO`, etc. Extract the full parent class
- **Interfaces** — what interfaces does this class implement? Extract each interface definition
- **Utility/helper classes** — `StringUtils`, `DateHelper`, `SecurityContext`, `MoneyFormatter` — any class referenced by multiple components
- **Common exception classes** — custom exception hierarchies (`BusinessException`, `DataAccessException`, etc.)
- **Shared DTOs and value objects** — classes used across multiple entry points
- **Common interceptors and filters** — `CorsFilter`, `LoggingInterceptor`, `SecurityFilter` that apply to multiple components
- **Base aspect classes** — common advice or pointcuts

Extract these fully even if they live outside the entry point's package — they carry critical logic that must be preserved.

### Step 5: Discover Tests
For every production file discovered, search for associated test files:
- **Test naming conventions** — `{ClassName}Test.java`, `{ClassName}Tests.java`, `{ClassName}IT.java` (integration tests), `{ClassName}Spec.groovy`
- **Test directories** — `src/test/java/`, `src/test/groovy/`, `src/test/resources/`, `src/integration-test/`
- **Test resources** — SQL setup scripts, test data files, mock XML configs, `test-beans.xml`

For each test class found, extract:
- Every test method with its purpose and assertions
- Test fixtures, setup methods (`@Before`, `@BeforeEach`, `@BeforeAll`), teardown methods
- Parameterized test data (`@ParameterizedTest` with `@ValueSource`, `@CsvSource`)
- Mock definitions and behavior setup
- Test-only beans and configurations (`@TestConfiguration`, `@SpyBean`, `@MockBean`)
- Integration test wiring (`@SpringBootTest`, `@DataJpaTest`, `@WebMvcTest` configurations)

Test files document edge cases and expected behaviors that may not be obvious from production code.

### Step 6: Discover Database Objects
For every table, view, or schema identified during code tracing, search the database scripts directory (search `db/`, `sql/`, `migration/`, `flyway/`, `src/main/resources/db/`, etc.) for associated objects:
- **Triggers** — on INSERT/UPDATE/DELETE; these contain business logic invisible to application code
- **Views and materialized views** — other systems may query these; capture full definitions
- **Database jobs** — DBMS_SCHEDULER, DBMS_JOB scheduled procedures
- **Synonyms and DB links** — cross-schema or cross-database dependencies
- **Sequences** — naming, start value, increment, caching — must be preserved in migration
- **Package bodies (Oracle)** — may contain procedures not called from app code
- **DDL for tables** — full column definitions, constraints, default values, partitioning, storage
- **User grants and permissions** — who/what can access what

Flag any DB object not referenced from application code:
- "⚠ UNREFERENCED: Table `AUDIT_LOG` found in DB scripts but not queried from application — may be populated by triggers or external processes"
- "⚠ DB OBJECT NOT IN REPO: Stored procedure `PKG_ORDER.APPROVE_LEGACY` referenced in code but definition not found in repo — request from DBA"

### Step 7: Discover Implicit Framework Behavior
Search the entire codebase for framework hooks that affect the entry point but aren't discoverable through import/call tracing:
- **Global exception handling** — `@ControllerAdvice`, `@ExceptionHandler`, `ErrorController` implementations
- **Filter/interceptor chains** — order in `web.xml`, `FilterRegistrationBean`, `HandlerInterceptor`, `WebMvcConfigurer`, `ServletContextListener`
- **AOP aspects** — `@Aspect` classes with pointcut expressions that silently match this entry point's methods
- **Bean lifecycle hooks** — `@PostConstruct`, `@PreDestroy`, `InitializingBean`, `DisposableBean`, `BeanPostProcessor`, `BeanFactoryPostProcessor`
- **Event system** — `ApplicationListener`, `@EventListener`, `ApplicationEventPublisher` — events published/consumed as side effects
- **Spring auto-configuration** — `@Configuration` classes, `@Conditional*` beans, `@Import`, `@EnableCaching`, `@EnableScheduling`, `@EnableAsync`, `@EnableTransactionManagement`
- **Servlet container configuration** — `@WebServlet`, `@WebFilter`, `@WebListener`, `ServletContainerInitializer`
- **Spring Security filter chain** — `SecurityFilterChain` bean definitions, custom authentication providers, custom filters inserted into the chain

For each hook, document: what it does, what components it affects (by pointcut/URL pattern/type), and its execution order.

### Step 8: Discover Runtime/Deployment Configuration
Search for deployment-time settings that affect application behavior:
- **Application server configs** — `server.xml`, `context.xml`, `standalone.xml` (JBoss/WildFly), `setenv.sh`, `catalina.sh` (Tomcat)
- **JNDI resource definitions** — datasources, JMS connection factories, mail sessions
- **JVM arguments and system properties** — captured from startup scripts
- **Container orchestration** — Kubernetes manifests (Deployment, Service, ConfigMap, Secret, Ingress), Helm charts, Docker Compose files
- **CI/CD pipelines** — Jenkinsfile, `.github/workflows/`, GitLab CI, Azure Pipelines — reveals build steps, secrets, environment variables, deployment targets
- **External configuration** — Spring Cloud Config references, Consul, Vault, AWS Secrets Manager
- **Load balancer / reverse proxy configs** — nginx.conf, Apache httpd.conf, HAProxy (if in repo)

Compile a master list of every environment variable reference (`${VAR_NAME}`, `%ENV_VAR%`) found across all config files. Flag anything referenced but not defined: "⚠ `DB_POOL_SIZE` referenced in application.yml but not defined in deployment configs"

### Step 9: Discover Profile-Specific Code Paths
Scan for behavior that only activates under certain conditions:
- **Profile annotations** — `@Profile("production")`, `@Profile("dev")`, `@Profile("!test")` — capture the full bean/configuration for each profile
- **Profile-specific property files** — `application-prod.properties`, `application-staging.yml`, etc. — capture ALL properties in each, noting differences from default
- **Conditional annotations** — `@ConditionalOnProperty`, `@ConditionalOnBean`, `@ConditionalOnClass`, `@ConditionalOnMissingBean`
- **Programmatic feature flags** — `if (featureFlags.isEnabled("ORDER_V2"))` — capture the full code path behind each flag
- **Environment-switching logic** — `if (env.equals("PROD"))`, `System.getProperty("mode")` — capture both branches

Tag each behavior in the Behavioral Inventory with its activation condition.

### Step 10: Discover Build-Time Code Generation
Check build configuration for generated code and external contracts:
- **Code generation plugins** — scan `pom.xml`, `build.gradle` for: CXF (WSDL→Java), JAXB (XSD→Java), OpenAPI Generator, MapStruct, Lombok, QueryDSL, JOOQ, Protobuf, Avro
- **For each plugin found**:
  - Document what it generates and from what input
  - Extract the input contract files in FULL (the WSDL, the XSD, the .proto, the .avsc, the OpenAPI spec)
  - If generated source is available (in `target/generated-sources/` or checked in), extract it like any other source file
  - If generated source is NOT available, document exactly what would be generated based on the input
- **Lombok expansion** — for every Lombok-annotated class (`@Data`, `@Builder`, `@Getter`, `@Setter`, `@ToString`, `@EqualsAndHashCode`, `@AllArgsConstructor`, `@NoArgsConstructor`, `@Slf4j`), expand what gets generated and document the effective fields, methods, constructors at runtime
- **MapStruct** — extract mapper interfaces AND generated implementations if available

### Step 11: Exhaustive Extraction (CRITICAL — ZERO SUMMARIZATION)
For every discovered file, perform COMPLETE extraction with NO summarization or abbreviation. This is the only time source code is read; downstream skills rely entirely on what is captured here.

**Classes, DTOs, Entities, Value Objects, Enums:**
- Every field — name, type, access modifier, default value, every annotation with exact parameters
- Every constructor — full parameter list with types
- Every method — full signature (return type, name, parameters with types, throws clause, annotations)
- Every getter/setter — especially those with custom logic (formatting, null coalescing, lazy init, computed values)
- Class-level annotations with all parameters (e.g., `@Entity`, `@Table(name="ORDERS", schema="APP")`, `@JsonIgnoreProperties({"internalId"})`)
- Inheritance chain — what it extends, what interfaces it implements
- Inner classes and static nested classes — fully extracted as if top-level
- Enum values with associated fields, constructors, and methods
- Constants — name, type, value, where referenced
- Static initializers and instance initializers

**Validation & Constraints:**
- Every annotation on every field: `@NotNull`, `@Size(min=1, max=255)`, `@Pattern(regexp="...")`, `@Email`, custom validators — exact parameters
- Programmatic validation logic in methods
- Custom constraint validators — the full validation logic

**Serialization & Mapping:**
- Jackson annotations: `@JsonProperty`, `@JsonFormat`, `@JsonIgnore`, `@JsonInclude`, custom serializers/deserializers
- MyBatis type handlers, result maps, discriminators
- Manual mapping logic (e.g., `toDTO()`, `fromEntity()`, builder patterns)
- JAXB annotations if present

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
- Every test method — name, annotations, assertions
- Test data and fixtures — what scenario each test covers
- Mock setups — what is mocked and what behavior is defined
- Integration test wiring — configurations

**Rule:** If it's in the source file, it goes in source-extracts.md. When in doubt, include it. A DTO with 45 fields gets all 45 fields. A stored procedure with 200 lines gets all 200 lines. A properties file with 80 entries gets all 80 entries.

### Step 12: Self-Validation Pass
After extraction, verify completeness:
- For each file in the discovered set, confirm source-extracts.md has a corresponding section
- For each class, confirm every field and method is listed (compare counts: extracted vs. actual)
- For each SQL/PLSQL file, confirm complete body is captured (not truncated)
- For each properties/config file, confirm every key is captured
- For each test file, confirm every test method is listed
- For each Lombok-annotated class, confirm generated methods are documented
- Verify every shared/common class discovered in Step 4 has a full extraction
- Verify every DB object found in Step 6 has its full body captured
- Verify every framework hook found in Step 7 has its full implementation captured

Flag any file with incomplete extraction: `⚠ INCOMPLETE — {reason}`. Report summary: "{N} files extracted, {M} fully validated, {K} flagged incomplete"

### Step 13: Categorize Organically
Group discovered components by their actual role as found during discovery (not predetermined buckets):
- Presentation Layer / View Templates
- Business Services / Logic
- Data Access / Persistence
- Messaging / Event Producers / Event Consumers
- Caching Layer
- Configuration
- Testing / Test Fixtures
- Security / Authentication / Authorization
- Shared/Common Code
- Framework Hooks / AOP / Filters
- Deployment / Infrastructure
- Build Configuration

### Step 14: Generate Behavioral Inventory
Create an exhaustive checklist of every discrete behavior, business rule, code path, and side effect found in the legacy code:

| ID | Behavior / Rule | Source File(s) | Type | Details |
|----|----------------|---------------|------|---------|
| B-001 | {e.g., "Validate order total > 0 before submit"} | OrderService.java:142 | Validation | {logic description, edge cases} |
| B-002 | {e.g., "Send Kafka event on order creation"} | OrderService.java:198 | Side Effect | {topic, payload shape, conditions} |
| B-003 | {e.g., "Return 403 if user lacks ROLE_ADMIN"} | OrderController.java:55 | Security | {annotation, filter, or manual check} |

Types: Validation, Business Rule, Side Effect, Security, Error Handling, Performance, Data Transformation, UI Behavior, Integration, Scheduling, Logging/Audit, Configuration.

### Step 15: Generate Fingerprint & Append to Contracts
Create `fingerprint.json` with:
- Git commit hash, branch, repo clean status, uncommitted files list
- Per-file SHA-256 hash, last modified timestamp, role, key signatures (method names, annotations, SQL calls, topic names), line count
- File count, total lines

Then append any newly discovered shared resources (utility classes, base classes, common DTOs, shared DB tables, Kafka topics, Quartz jobs) to `contracts.json` under a new entry point slug.

## Output 1: fingerprint.json

Located at: `modernization-output/{slug}/fingerprint.json`

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
    }
  },
  "file_count": 14,
  "total_lines": 4230
}
```

**Field details:**
- **git_commit** — exact commit hash at generation time; enables `git diff` against it later
- **git_branch** — the branch analyzed (important if feature branches diverge)
- **git_repo_clean** — whether there were uncommitted changes (if true, the commit hash alone isn't perfect)
- **uncommitted_changes** — lists dirty files so you know the fingerprint includes unstaged work
- **files.{path}.sha256** — content hash of each file; primary drift detection mechanism
- **files.{path}.key_signatures** — structural fingerprint: method names, annotations, form actions, SQL calls, topic names. When a file changes, comparing old vs. new signatures reveals *what kind* of change occurred
- **files.{path}.role** — which category from analysis.md, so drift reports can say "a Data Access file changed"

## Output 2: analysis.md

Located at: `modernization-output/{slug}/analysis.md`

**Sections (dynamically created based on what is actually found):**

1. **Entry Point** — file, type, description
2. **Discovered Components** — one section per category (e.g., "Presentation Layer", "Business Services", "Data Access")
3. **Inbound Pathways** — detailed table of every caller: caller, type (HTTP, method call, message, etc.), mechanism, trigger condition, data passed, frequency, notes
4. **Data Flow Diagram** — Mermaid or textual description showing how data flows through all components including inbound pathways
5. **Business Logic Summary** — high-level description of what the feature/module does
6. **Technology Inventory** — all frameworks/technologies encountered (Spring MVC, Oracle PLSQL, Kafka, MyBatis, etc.)
7. **Behavioral Inventory** — exhaustive checklist of every behavior, rule, code path, side effect (see Step 14 above)
8. **Dependencies & Risks** — external systems, shared state, coupling, hardcoded values, undocumented dependencies

## Output 3: source-extracts.md

Located at: `modernization-output/{slug}/source-extracts.md`

A comprehensive technical dump of every raw detail found while reading source code. Format for each file:

```markdown
## File: {filepath}
**Role**: {category from analysis.md}
**Lines**: {count}
**Extraction Validated**: ✅ {field-count} fields, {method-count} methods captured (matches source)

### Class / Interface / Enum Definition
```{language}
@Entity
@Table(name = "ORDERS", schema = "APP", uniqueConstraints = @UniqueConstraint(columnNames = {"order_number"}))
@JsonIgnoreProperties(ignoreUnknown = true)
public class Order extends BaseEntity implements Auditable {
```
```

- **Extends**: {parent classes}
- **Implements**: {interfaces}
- **Type**: Entity / DTO / Service / etc.

### Fields (ALL fields — no omissions)
| # | Field | Type | Access | Default | Annotations (with params) | Line | Notes |
|---|-------|------|--------|---------|--------------------------|------|-------|
| 1 | `id` | `Long` | private | — | `@Id`, `@GeneratedValue(...)`, `@Column(name="ORDER_ID")` | 12 | PK |
| ... (ALL fields) | ... | ... | ... | ... | ... | ... | ... |

### Methods (ALL methods — no omissions)
| # | Signature | Visibility | Annotations | Line | Purpose | Custom Logic? |
|---|-----------|-----------|------------|------|---------|--------------|
| 1 | `Long getId()` | public | — | 55 | Standard getter | No |
| ... (ALL methods) | ... | ... | ... | ... | ... | ... |

### SQL / Database Operations
```sql
{exact complete SQL statement}
```

### Stored Procedure Bodies
```sql
CREATE OR REPLACE PROCEDURE {name}(...) AS
  {complete procedure body with all logic}
END;
```

(Repeat structure for every discovered file. Include every detail — DTOs with 45 fields get all 45, procedures with 200 lines get all 200.)
```

## Output 4: Append to contracts.json

Located at: `modernization-output/contracts.json`

The skill appends a new entry for this entry point's slug:

```json
{
  "entry_points": {
    "order-jsp": {
      "entry_point": "src/main/webapp/WEB-INF/views/order.jsp",
      "analyzed_at": "2026-03-22T14:30:00Z",
      "shared_resources": {
        "utility_classes": ["com.acme.util.DateHelper", "com.acme.util.SecurityContext"],
        "base_classes": ["com.acme.service.BaseTransactionalService"],
        "shared_dtos": ["com.acme.dto.OrderDTO", "com.acme.dto.OrderItemDTO"],
        "kafka_topics": ["order.created", "order.approved"],
        "db_tables": ["ORDERS", "ORDER_ITEMS"],
        "common_exceptions": ["BusinessException", "DataAccessException"]
      }
    }
  }
}
```

This allows downstream analyses to track which resources are shared across multiple entry points.

## Implementation Notes

1. **Codebase root** — before starting, establish the absolute path to the project root (pom.xml / build.gradle location)
2. **Working slug** — derive a slug from the entry point path (e.g., `order.jsp` → `order-jsp`, `OrderService.java` → `order-service`). Create a working directory `modernization-output/{slug}/`
3. **Preserve all details** — during extraction, copy full code verbatim; never paraphrase or summarize
4. **Record source lines** — in every table, include the line number from the source file; this enables precise drift detection and error localization
5. **Use grep strategically** — searches for class names, method names, package names, URL patterns, topic names, table names find callers and framework hooks
6. **Build files matter** — read pom.xml / build.gradle for dependency versions, plugin configs, and generated source locations
7. **Validate completeness** — before closing, compare field/method counts in source-extracts.md against actual source files; flag any discrepancies
8. **No guessing** — if you can't find a referenced resource (e.g., a Kafka topic consumer, a called stored procedure, a property value), flag it as "⚠ NOT FOUND" and note the missing reference

---

**References:** See `references/output-templates.md` for detailed output format templates and schema documentation for fingerprint.json, analysis.md, and source-extracts.md sections.
