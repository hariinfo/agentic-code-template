---
name: gen-code
description: Generate Spring Boot 4.x microservice code from modernization analysis, functional spec, technical spec, and JIRA stories. Produces complete buildable Spring Boot projects with entity layer, repositories, services, controllers, messaging integration, security configuration, tests, and traceability manifest. Trigger on generate code, Spring Boot, microservice, modernize code, code generation, scaffold, modern code.
---

# Code Generator (gen-code)

## Overview

This skill generates a complete, buildable **Spring Boot 4.x microservice** from the analysis, functional specification, technical specification, and JIRA stories produced by earlier modernization steps. The generated code translates legacy business logic, data models, and integration patterns into idiomatic Spring Boot conventions while maintaining precise traceability back to the original legacy systems.

The primary goal is not to mechanically convert legacy code line-by-line, but to **understand intent from the analysis** and generate clean, testable Spring Boot code that preserves all business-critical behaviors.

## Inputs

You will receive a **slug** (the modernization-output directory) and a **service-name** (the microservice name). Load these files from `modernization-output/{slug}/`:

- **analysis.md** — component inventory (classes, JSPs, DAOs, PLSQL, Kafka topics), behavioral inventory (B-xxx entry point flows), data flow diagram
- **fingerprint.json** — legacy file list and signatures (for deduplication and change tracking)
- **funcspec.md** — functional requirements, acceptance criteria, data model, integration points, non-functional requirements
- **techspec.md** — technical specification with detailed API contracts, SQL logic, messaging schemas, error handling, transaction boundaries, security controls, caching strategies
- **jira-stories.md** — implementation stories with story IDs, title, description, legacy source files, traceability to funcspec and behaviors, estimated effort, acceptance criteria
- **contracts.json** — (optional) cross-service API contracts and message schemas to ensure compatibility

## Target Stack — Spring Boot 4.x

The generated code targets **Spring Boot 4.0+** with **Spring Framework 7.0**, Java 17+, and Jakarta EE 9+. Encode these platform decisions:

### Runtime & Language
- **Java 17+** (Java 25 idioms: records, sealed classes, pattern matching where beneficial)
- **Spring Boot 4.0+** with Spring Framework 7.0
- **Jakarta EE 9+** (jakarta.* namespace — NOT javax.*)
- **Build tool**: Maven (default) or Gradle (configurable)

### Web & API Layer
- Spring MVC (default for synchronous servlet-based legacy) or Spring WebFlux (if legacy is event-driven)
- **HTTP Interface Clients** (new in Spring Boot 4.x) for consuming external REST APIs — replaces RestTemplate/WebClient boilerplate
- API versioning support via `spring.mvc.apiversion.*` where applicable
- OpenAPI 3.x spec generation via springdoc-openapi

### Data Access
- **Spring Data JPA** with **Hibernate 7** (replaces legacy MyBatis mappers)
- Flyway or Liquibase for schema migration scripts (generated from PLSQL/DDL analysis)
- Repository pattern with custom query methods derived from legacy DAO SQL
- Native queries with `@Query` for complex legacy SQL that doesn't map cleanly to JPA

### Messaging
- **Spring Kafka** (if legacy uses Kafka) with proper serializer/deserializer configuration
- **Spring JMS** / **Spring AMQP** (if legacy uses JMS/RabbitMQ)
- CloudEvents format for event payloads where appropriate
- Proper error handling and dead-letter queue configuration

### Security
- **Spring Security 7.0** with simplified configuration in Boot 4.x
- Map legacy security filters/interceptors to Spring Security SecurityFilterChain
- OAuth2/OIDC resource server where legacy used custom auth
- CSRF, XSS, SQL injection prevention baked into framework

### Observability
- **OpenTelemetry starter** (`spring-boot-starter-opentelemetry`) for metrics, tracing, and profiling
- Structured logging with correlation IDs (MDC)
- Actuator health endpoints and custom metrics
- Distributed tracing compatible with Jaeger/Zipkin

### Testing
- **JUnit 5** + Spring Boot Test
- **RestTestClient** (new in Boot 4.x) for controller tests
- **Testcontainers** for database, Kafka, and messaging integration tests
- One test class per generated service/controller with tests derived from Behavioral Inventory

## Generation Phases

Execute code generation in **7 distinct phases**, maintaining story-by-story traceability:

### Phase 1 — Project Scaffolding

Generate the project skeleton and Spring Boot 4.x configuration:

1. Create project structure:
   ```
   {service-name}/
   ├── pom.xml (or build.gradle)
   ├── Dockerfile
   ├── docker-compose.yml
   ├── .gitignore
   ├── README.md
   └── src/
   ```

2. Generate **pom.xml** (or build.gradle):
   - groupId: `com.acme`
   - artifactId: `{service-name}` (e.g., `order-service`)
   - Spring Boot 4.0.x parent POM
   - Dependencies: spring-boot-starter-web (or webflux), spring-boot-starter-data-jpa, spring-boot-starter-security, spring-kafka (if messaging), spring-boot-starter-opentelemetry, springdoc-openapi-starter-webmvc-api, junit-jupiter, testcontainers
   - Plugins: spring-boot-maven-plugin, maven-compiler-plugin (source/target Java 17+)

3. Generate **application.yml**:
   ```yaml
   spring:
     application:
       name: {service-name}
     jpa:
       hibernate:
         ddl-auto: validate
       properties:
         hibernate:
           dialect: org.hibernate.dialect.OracleDialect (or PostgreSQL if migrating)
     datasource:
       url: jdbc:oracle:thin:@localhost:1521:XE
       username: ${DB_USER:sa}
       password: ${DB_PASSWORD:password}
     kafka:
       bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}

   management:
     endpoints:
       web:
         exposure:
           include: health,metrics,prometheus
     tracing:
       sampling:
         probability: 1.0
   ```

4. Generate profile-specific YAMLs: `application-local.yml`, `application-dev.yml`, `application-staging.yml`, `application-prod.yml`

5. Generate **Dockerfile**:
   ```dockerfile
   FROM eclipse-temurin:17-jdk-alpine
   COPY target/order-service-1.0.0.jar app.jar
   ENTRYPOINT ["java", "-jar", "/app.jar"]
   ```

6. Generate **docker-compose.yml** with services for PostgreSQL, Kafka, Prometheus (local development)

7. Generate README.md explaining project structure, tech stack, how to build/run, and what was generated and from where

8. Record all Phase 1 outputs in `code-manifest.json`

### Phase 2 — Data Layer

For each JIRA story touching data access:

1. **Identify legacy data source**:
   - If legacy DAO exists, read the JDBC/MyBatis code and infer table structure
   - If PLSQL stored procedures exist, read the parameter types and INSERT/UPDATE/DELETE statements
   - If techspec has SQL extracts, use those as the authoritative schema

2. **Generate JPA Entity**:
   ```java
   @Entity
   @Table(name = "ORDERS")
   public class Order {
     @Id
     @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "orders_seq")
     @SequenceGenerator(name = "orders_seq", sequenceName = "ORDERS_SEQ")
     private Long id;

     @Column(name = "ORDER_NUMBER", nullable = false, unique = true)
     private String orderNumber;

     @Column(name = "ORDER_DATE")
     private LocalDateTime orderDate;

     @Column(name = "STATUS")
     @Enumerated(EnumType.STRING)
     private OrderStatus status;

     @ManyToOne(fetch = FetchType.LAZY)
     @JoinColumn(name = "CUSTOMER_ID")
     private Customer customer;

     @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
     private Set<OrderLine> lines;

     // Legacy: B-001 — orders created with PENDING status
     @CreationTimestamp
     @Column(name = "CREATED_AT", updatable = false)
     private LocalDateTime createdAt;

     // Getters, setters, equals, hashCode
   }
   ```
   - Use Java records for simple value objects (DTOs, events)
   - Include legacy comments linking to Behavioral Inventory (B-xxx)
   - Respect legacy column names and data types (inferred from analysis)
   - Add JPA annotations for relationships, sequences, audit columns

3. **Generate Spring Data Repository**:
   ```java
   @Repository
   public interface OrderRepository extends JpaRepository<Order, Long> {
     Optional<Order> findByOrderNumber(String orderNumber);
     List<Order> findByCustomerIdAndStatusOrderByOrderDateDesc(Long customerId, OrderStatus status);

     // Legacy: fetch orders created between dates (from OrderDAO.findByDateRange)
     @Query("SELECT o FROM Order o WHERE o.createdAt BETWEEN :startDate AND :endDate")
     List<Order> findByDateRange(
       @Param("startDate") LocalDateTime startDate,
       @Param("endDate") LocalDateTime endDate
     );
   }
   ```
   - Translate legacy SQL WHERE clauses to custom query methods
   - Use `@Query` for complex native SQL (document the legacy source file)
   - Include pagination and sorting support

4. **Generate Flyway migration script** if schema changes are needed:
   ```sql
   -- V2__Create_orders_table.sql
   -- Source: legacy ORDERS table with audit columns added
   CREATE TABLE ORDERS (
     ID NUMBER PRIMARY KEY,
     ORDER_NUMBER VARCHAR2(50) NOT NULL UNIQUE,
     CUSTOMER_ID NUMBER NOT NULL,
     STATUS VARCHAR2(20) DEFAULT 'PENDING',
     CREATED_AT TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
     UPDATED_AT TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
     CREATED_BY VARCHAR2(100),
     UPDATED_BY VARCHAR2(100),
     FOREIGN KEY (CUSTOMER_ID) REFERENCES CUSTOMERS(ID)
   );
   CREATE SEQUENCE ORDERS_SEQ START WITH 1;
   ```

5. **Generate Testcontainers integration test**:
   ```java
   @SpringBootTest
   @Testcontainers
   class OrderRepositoryIntegrationTest {
     @Container
     static OracleContainer db = new OracleContainer("gvenzl/oracle-free:latest");

     @Autowired
     private OrderRepository repo;

     @Test
     void testFindByOrderNumber() {
       Order order = new Order();
       order.setOrderNumber("ORD-001");
       repo.save(order);

       Optional<Order> found = repo.findByOrderNumber("ORD-001");
       assertTrue(found.isPresent());
     }
   }
   ```

6. Record mapping in `code-manifest.json`: generated file → legacy DAO/PLSQL → JIRA story → behaviors covered

### Phase 3 — Service Layer

For each JIRA story touching business logic:

1. **Read legacy service class** from the story's "Legacy Source Files" reference

2. **Generate Spring Service**:
   ```java
   @Service
   @RequiredArgsConstructor
   public class OrderService {
     private final OrderRepository orderRepo;
     private final OrderValidator validator;
     private final OrderMapper mapper;
     private final OrderEventPublisher eventPublisher;

     // Legacy: B-001, B-002 — Create order with validation and status PENDING
     public OrderResponse createOrder(CreateOrderRequest req) {
       validator.validate(req); // Validate per FR-001

       Order order = new Order();
       order.setOrderNumber(req.getOrderNumber());
       order.setStatus(OrderStatus.PENDING);
       order.setCustomerId(req.getCustomerId());

       Order saved = orderRepo.save(order);

       // Publish event for downstream systems (Kafka)
       eventPublisher.publishOrderCreated(mapper.toEvent(saved));

       return mapper.toResponse(saved);
     }

     // Legacy: B-003, B-004 — Approve order, transition to APPROVED
     @Transactional(isolation = Isolation.SERIALIZABLE)
     public OrderResponse approveOrder(Long orderId) {
       Order order = orderRepo.findById(orderId)
         .orElseThrow(() -> new OrderNotFoundException(orderId));

       if (order.getStatus() != OrderStatus.PENDING) {
         throw new InvalidStateException("Order must be PENDING to approve");
       }

       order.setStatus(OrderStatus.APPROVED);
       Order updated = orderRepo.save(order);

       eventPublisher.publishOrderApproved(mapper.toEvent(updated));

       return mapper.toResponse(updated);
     }
   }
   ```
   - Each method maps to one or more Behavioral Inventory items (B-xxx)
   - Include `// Legacy: B-xxx` comments linking code back to inventory
   - Use Spring annotations: `@Service`, `@Transactional`, `@RequiredArgsConstructor`
   - Respect isolation levels, transaction boundaries from techspec
   - Inject repositories, validators, event publishers, HTTP clients
   - Handle exceptions with domain-specific exceptions (OrderNotFoundException, etc.)

3. **Generate Validator classes** (if complex validation logic exists):
   ```java
   @Component
   public class OrderValidator {
     public void validate(CreateOrderRequest req) {
       if (req.getOrderNumber() == null || req.getOrderNumber().isBlank()) {
         throw new ValidationException("Order number is required");
       }
       if (req.getOrderTotal() == null || req.getOrderTotal().compareTo(BigDecimal.ZERO) <= 0) {
         throw new ValidationException("Order total must be > 0");
       }
     }
   }
   ```

4. **Generate unit tests** covering each behavior:
   ```java
   @ExtendWith(MockitoExtension.class)
   class OrderServiceTest {
     @Mock
     private OrderRepository orderRepo;

     @Mock
     private OrderEventPublisher eventPublisher;

     @InjectMocks
     private OrderService service;

     @Test
     void testCreateOrderPendingStatus() {
       // Arrange
       CreateOrderRequest req = new CreateOrderRequest("ORD-001", 1L, new BigDecimal("100.00"));
       Order saved = new Order();
       saved.setId(1L);
       saved.setOrderNumber("ORD-001");
       saved.setStatus(OrderStatus.PENDING);
       when(orderRepo.save(any())).thenReturn(saved);

       // Act
       OrderResponse resp = service.createOrder(req);

       // Assert
       assertEquals(OrderStatus.PENDING, resp.getStatus());
       verify(eventPublisher).publishOrderCreated(any());
     }

     @Test
     void testApproveOrderInvalidState() {
       // Arrange
       Order order = new Order();
       order.setStatus(OrderStatus.APPROVED); // Already approved
       when(orderRepo.findById(1L)).thenReturn(Optional.of(order));

       // Act & Assert
       assertThrows(InvalidStateException.class, () -> service.approveOrder(1L));
     }
   }
   ```

5. Record mapping in `code-manifest.json`

### Phase 4 — API/Controller Layer

For each JIRA story touching REST endpoints or UI:

1. **Identify legacy endpoint** from JSP/controller/REST resource

2. **Generate REST Controller**:
   ```java
   @RestController
   @RequestMapping("/api/v1/orders")
   @RequiredArgsConstructor
   public class OrderController {
     private final OrderService service;
     private final OrderMapper mapper;

     // Legacy: B-001, B-002, B-005 — Create order endpoint
     @PostMapping
     public ResponseEntity<OrderResponse> createOrder(
       @Valid @RequestBody CreateOrderRequest req,
       HttpServletRequest httpReq
     ) {
       OrderResponse resp = service.createOrder(req);
       return ResponseEntity
         .created(URI.create("/api/v1/orders/" + resp.getId()))
         .body(resp);
     }

     // Legacy: B-003 — Approve order endpoint
     @PutMapping("/{id}/approve")
     public ResponseEntity<OrderResponse> approveOrder(@PathVariable Long id) {
       OrderResponse resp = service.approveOrder(id);
       return ResponseEntity.ok(resp);
     }

     @ExceptionHandler(OrderNotFoundException.class)
     public ResponseEntity<ErrorResponse> handleNotFound(OrderNotFoundException e) {
       return ResponseEntity.status(404).body(new ErrorResponse("Order not found", e.getMessage()));
     }
   }
   ```
   - Use `@RestController`, `@RequestMapping`, `@PostMapping`, `@GetMapping`, etc.
   - Include `// Legacy: B-xxx` comments linking endpoints to behavioral inventory
   - Use DTOs (preferably records) for request/response
   - Include `@Valid` annotations for validation
   - Include exception handlers with proper HTTP status codes
   - Add OpenAPI annotations for Swagger documentation

3. **Generate HTTP Interface Clients** for outbound REST calls:
   ```java
   @HttpExchange("/api/v1/customers")
   public interface CustomerClient {
     @GetExchange("/{id}")
     Optional<CustomerResponse> getCustomer(@PathVariable Long id);

     @PostExchange
     CustomerResponse createCustomer(@RequestBody CreateCustomerRequest req);
   }

   @Configuration
   public class HttpClientConfig {
     @Bean
     CustomerClient customerClient(RestClient.Builder builder) {
       return HttpServiceProxyFactory
         .builderFor(builder.baseUrl("http://customer-service:8080").build())
         .build()
         .createClient(CustomerClient.class);
     }
   }
   ```

4. **Generate RestTestClient controller tests**:
   ```java
   @SpringBootTest
   @AutoConfigureRestClient
   class OrderControllerTest {
     @Autowired
     private RestClient.Builder restClient;

     @MockBean
     private OrderService service;

     @Test
     void testCreateOrderReturns201() {
       CreateOrderRequest req = new CreateOrderRequest("ORD-001", 1L, new BigDecimal("100.00"));
       OrderResponse resp = new OrderResponse(1L, "ORD-001", OrderStatus.PENDING, ...);
       when(service.createOrder(any())).thenReturn(resp);

       RestClient client = restClient.build();
       OrderResponse result = client.post()
         .uri("/api/v1/orders")
         .body(req)
         .retrieve()
         .body(OrderResponse.class);

       assertEquals(resp.getId(), result.getId());
     }
   }
   ```

5. **Generate OpenAPI annotations**:
   ```java
   @Operation(summary = "Create a new order", description = "Creates an order with PENDING status")
   @ApiResponses({
     @ApiResponse(responseCode = "201", description = "Order created successfully"),
     @ApiResponse(responseCode = "400", description = "Invalid request body"),
     @ApiResponse(responseCode = "409", description = "Order number already exists")
   })
   @PostMapping
   public ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody CreateOrderRequest req) { ... }
   ```

6. Record mapping in `code-manifest.json`

### Phase 5 — Messaging & Integration

For each JIRA story touching messaging or external system integration:

1. **Generate Kafka Producer** (if legacy uses Kafka):
   ```java
   @Component
   @RequiredArgsConstructor
   public class OrderEventPublisher {
     private final KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;

     // Legacy: B-001 — Publish OrderCreated event after order is persisted
     public void publishOrderCreated(OrderCreatedEvent event) {
       kafkaTemplate.send("orders.created", event.getOrderId().toString(), event);
     }
   }
   ```

2. **Generate Kafka Consumer** (if legacy consumes Kafka):
   ```java
   @Component
   @RequiredArgsConstructor
   public class OrderEventListener {
     private final OrderService service;

     // Legacy: B-010 — Listen for PaymentApproved event and update order status
     @KafkaListener(topics = "payments.approved", groupId = "order-service")
     public void handlePaymentApproved(PaymentApprovedEvent event) {
       service.markOrderAsApproved(event.getOrderId());
     }
   }
   ```

3. **Generate Event payload DTOs** using Java records:
   ```java
   public record OrderCreatedEvent(
     Long orderId,
     String orderNumber,
     Long customerId,
     LocalDateTime createdAt
   ) {}
   ```

4. **Generate Kafka configuration**:
   ```java
   @Configuration
   public class KafkaConfig {
     @Bean
     KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate(ProducerFactory<String, OrderCreatedEvent> pf) {
       return new KafkaTemplate<>(pf);
     }

     @Bean
     ProducerFactory<String, OrderCreatedEvent> producerFactory() {
       return new DefaultKafkaProducerFactory<>(
         Map.ofEntries(
           Map.entry(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "${spring.kafka.bootstrap-servers}"),
           Map.entry(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class)
         )
       );
     }
   }
   ```

5. **Generate Testcontainers integration tests** for messaging:
   ```java
   @SpringBootTest
   @Testcontainers
   class OrderEventListenerIntegrationTest {
     @Container
     static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.0.0"));

     @Autowired
     private KafkaTemplate<String, PaymentApprovedEvent> kafkaTemplate;

     @MockBean
     private OrderService orderService;

     @Test
     void testPaymentApprovedEventUpdatesOrderStatus() {
       PaymentApprovedEvent event = new PaymentApprovedEvent(1L, new BigDecimal("100.00"), ...);
       kafkaTemplate.send("payments.approved", event.getOrderId().toString(), event);

       await().atMost(Duration.ofSeconds(5))
         .untilAsserted(() -> verify(orderService).markOrderAsApproved(1L));
     }
   }
   ```

6. Record mapping in `code-manifest.json`

### Phase 6 — Security & Cross-Cutting Concerns

1. **Generate Spring Security configuration**:
   ```java
   @Configuration
   @EnableWebSecurity
   public class SecurityConfig {
     @Bean
     SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
       http
         .authorizeHttpRequests(auth -> auth
           .requestMatchers("/actuator/health").permitAll()
           .requestMatchers("/api/v1/orders").hasRole("USER")
           .requestMatchers("/api/v1/orders/*/approve").hasRole("MANAGER")
           .anyRequest().authenticated()
         )
         .oauth2ResourceServer(oauth2 -> oauth2.jwt(withDefaults()))
         .csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))
         .build();
       return http.build();
     }

     @Bean
     JwtDecoder jwtDecoder() {
       return JwtDecoders.fromIssuerLocation("${spring.security.oauth2.resourceserver.jwt.issuer-uri}");
     }
   }
   ```

2. **Generate AOP aspects** for cross-cutting concerns (logging, audit, validation):
   ```java
   @Aspect
   @Component
   @RequiredArgsConstructor
   public class AuditAspect {
     private final AuditLog auditLog;

     @Around("@annotation(audited)")
     public Object auditServiceCall(ProceedingJoinPoint pjp, Audited audited) throws Throwable {
       String user = SecurityContextHolder.getContext().getAuthentication().getName();
       String method = pjp.getSignature().getName();

       try {
         Object result = pjp.proceed();
         auditLog.log(user, method, "SUCCESS", null);
         return result;
       } catch (Exception e) {
         auditLog.log(user, method, "FAILURE", e.getMessage());
         throw e;
       }
     }
   }
   ```

3. **Configure OpenTelemetry for observability**:
   ```yaml
   management:
     tracing:
       sampling:
         probability: 1.0
     otlp:
       tracing:
         endpoint: http://jaeger:4317
   ```

4. **Generate health check endpoint** and custom metrics:
   ```java
   @Component
   public class OrderServiceMetrics {
     private final MeterRegistry meterRegistry;

     public void recordOrderCreated() {
       meterRegistry.counter("orders.created.total").increment();
     }
   }
   ```

### Phase 7 — Verification Hooks

1. **Generate BehaviorCoverageTest**:
   ```java
   @SpringBootTest
   class BehaviorCoverageTest {
     @Autowired
     private OrderService service;

     @Test
     @Disabled("B-001: Orders created with PENDING status")
     void testB001_OrderCreationSetsStatusToPending() {
       // TODO: Enable once OrderService.createOrder is verified
     }

     @Test
     @Disabled("B-002: Order number must be unique")
     void testB002_OrderNumberUniquenessEnforced() {
       // TODO: Enable once duplicate detection is verified
     }

     @Test
     @Disabled("B-003: Only PENDING orders can be approved")
     void testB003_ApprovalValidatesStatus() {
       // TODO: Enable once approval logic is verified
     }
   }
   ```

2. Add Maven/Gradle task to report behavioral coverage:
   ```xml
   <plugin>
     <groupId>org.apache.maven.plugins</groupId>
     <artifactId>maven-surefire-plugin</artifactId>
     <configuration>
       <groups>behaviorCoverage</groups>
     </configuration>
   </plugin>
   ```

## Outputs

### Output 1: `{service-name}/` — Complete Spring Boot Project

A fully buildable microservice directory:

```
modernization-output/{slug}/{service-name}/
├── pom.xml
├── Dockerfile
├── docker-compose.yml
├── .gitignore
├── README.md
└── src/
    ├── main/
    │   ├── java/com/acme/{servicepkg}/
    │   │   ├── config/           (Security, Kafka, OpenTelemetry, etc.)
    │   │   ├── controller/       (RestControllers)
    │   │   ├── service/          (Business logic)
    │   │   ├── repository/       (Spring Data JPA)
    │   │   ├── entity/           (JPA entities)
    │   │   ├── dto/              (Request/response DTOs — records)
    │   │   ├── event/            (Kafka event payload DTOs)
    │   │   ├── client/           (HTTP Interface Clients)
    │   │   ├── aspect/           (AOP cross-cutting)
    │   │   ├── exception/        (Domain exceptions)
    │   │   └── mapper/           (Object mappers — MapStruct or manual)
    │   └── resources/
    │       ├── application.yml
    │       ├── application-local.yml
    │       ├── application-dev.yml
    │       └── db/migration/     (Flyway V-versioned scripts)
    └── test/
        └── java/com/acme/{servicepkg}/
            ├── controller/       (RestTestClient tests)
            ├── service/          (Unit tests)
            ├── repository/       (Testcontainers integration tests)
            └── BehaviorCoverageTest.java
```

**Must be buildable and testable out of the box**:
- `mvn clean package` produces a JAR
- `mvn test` runs all unit tests
- Disabled tests in BehaviorCoverageTest can be enabled incrementally

### Output 2: `code-manifest.json` — Traceability Map

Maps every generated file back to legacy sources, JIRA stories, funcspec requirements, and behavioral inventory:

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
      "generated_file": "order-service/src/main/java/com/acme/orderservice/entity/Order.java",
      "legacy_files": ["db/schema/orders_table.sql"],
      "stories": ["STORY-001"],
      "funcspec_refs": ["FR-001"],
      "behaviors_covered": ["B-001"],
      "generation_phase": "Phase 2 — Data Layer",
      "notes": "JPA entity mapped from legacy ORDERS table"
    },
    {
      "generated_file": "order-service/src/main/java/com/acme/orderservice/service/OrderService.java",
      "legacy_files": [
        "src/main/java/com/acme/service/OrderService.java",
        "db/procedures/PROCESS_ORDER.sql"
      ],
      "stories": ["STORY-001", "STORY-002"],
      "funcspec_refs": ["FR-001", "FR-002"],
      "behaviors_covered": ["B-001", "B-002", "B-003"],
      "generation_phase": "Phase 3 — Service Layer",
      "notes": "Business logic translated from legacy service; PLSQL procedures inlined as Java methods"
    }
  ],
  "behaviors_not_mapped": [],
  "stories_not_generated": [],
  "total_generated_files": 28,
  "total_test_files": 15,
  "behavioral_coverage": "100%"
}
```

## Generation Principles

Encode these principles to avoid code generation pitfalls:

1. **Translate intent, not syntax** — don't mechanically convert PLSQL → Java or JSP → REST controller. Understand what the legacy code achieves and write idiomatic Spring Boot code that achieves the same thing. A 200-line JSP with embedded SQL might become a 30-line REST controller + a service + a repository.

2. **Favor Spring Boot conventions** — use auto-configuration, starters, and convention-over-configuration. If Spring Boot 4.x has a built-in way (HTTP Interface Clients, Spring Data JPA), use it instead of replicating legacy patterns.

3. **Keep legacy comments** — every generated method/class should include a comment like `// Legacy: B-001 — Order status set to PENDING on creation` so developers can trace decisions back to their origin. This is crucial for compliance and maintenance.

4. **Generate tests from behaviors, not from code** — tests should validate the behaviors in the Behavioral Inventory (B-xxx), not mirror the structure of generated code. Tests catch regressions in *what the code does*, not *how it's structured*.

5. **Don't over-generate** — if a JIRA story covers infrastructure (CI/CD pipeline, monitoring dashboards, runbooks), generate a TODO placeholder rather than guessing. Only generate application code, tests, and configuration.

6. **Make it buildable** — the generated project must compile and pass its tests out of the box. If something requires manual configuration (database credentials, Kafka broker), use placeholders with clear `TODO` markers in application.yml.

7. **Check contracts.json** — before generating HTTP clients or Kafka producers, verify compatibility with contracts.json. If a consumer expects a schema but contracts.json differs, flag it as a warning and generate a TODO comment.

8. **Preserve error handling** — legacy code often has detailed error handling and validation. Translate this into Spring exception handlers, validation annotations, and domain-specific exceptions, not generic RuntimeExceptions.

## Execution Checklist

Before marking generation complete, verify:

- [ ] All 7 phases executed in order
- [ ] Project scaffolding creates buildable pom.xml and application.yml
- [ ] Every JIRA story has generated code (service, controller, test, or event producer/consumer)
- [ ] Every generated class includes `// Legacy: B-xxx` comments linking to behavioral inventory
- [ ] All entities have tests (Testcontainers integration tests)
- [ ] All services have unit tests (with mocks for repositories and dependencies)
- [ ] All controllers have RestTestClient tests
- [ ] All Kafka producers/consumers have integration tests
- [ ] BehaviorCoverageTest exists with one `@Disabled` test per Behavioral Inventory item
- [ ] code-manifest.json complete with full traceability
- [ ] `mvn clean package` succeeds (project compiles without errors)
- [ ] `mvn test` runs all unit tests and reports coverage
- [ ] README.md explains the project, how to build, run, and what was generated
- [ ] Behavioral coverage reported in code-manifest.json is >= 95%
- [ ] All dependencies in pom.xml align with Spring Boot 4.x targets
