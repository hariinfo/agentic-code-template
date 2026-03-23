# Spring Boot 4.x Patterns & Conventions

Reference guide for generating Spring Boot 4.x code that follows framework conventions and leverages new Spring Framework 7.0 / Java 17+ features.

## 1. Project Structure & Naming Conventions

### Package Naming
```
com.acme.{service-name}
├── config/         — Spring configuration classes (@Configuration)
├── controller/     — REST controllers (@RestController)
├── service/        — Business logic (@Service)
├── repository/     — Data access (Spring Data JPA interfaces)
├── entity/         — JPA entities (@Entity)
├── dto/            — Data transfer objects (records, value objects)
├── event/          — Kafka/messaging event payloads (records)
├── client/         — HTTP Interface Client interfaces
├── aspect/         — AOP aspects (@Aspect)
├── exception/      — Domain-specific exceptions
├── mapper/         — Object mapping utilities
└── util/           — Utility/helper classes
```

### Artifact & Application Naming
- **Maven artifactId**: kebab-case (e.g., `order-service`, `payment-api`)
- **Package suffix**: camelCase (e.g., `com.acme.orderservice`, `com.acme.paymentapi`)
- **spring.application.name**: kebab-case (e.g., `order-service`)
- **Docker image**: kebab-case (e.g., `order-service:1.0.0`)

## 2. Spring Boot 4.x Dependencies

### Starter Dependencies (Common Set)
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-kafka</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-opentelemetry</artifactId>
</dependency>
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-api</artifactId>
  <version>2.0.0</version>
</dependency>
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-core</artifactId>
</dependency>
```

### Test Dependencies
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>testcontainers</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>testcontainers-bom</artifactId>
  <type>pom</type>
  <scope>import</scope>
</dependency>
```

### Java Language Level
```xml
<properties>
  <java.version>17</java.version>
  <maven.compiler.source>17</maven.compiler.source>
  <maven.compiler.target>17</maven.compiler.target>
</properties>
```

## 3. JPA Entity Best Practices

### Basic Entity with Audit Columns
```java
@Entity
@Table(name = "ORDERS")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = {"lines"})
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Order {
  @Id
  @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "orders_seq")
  @SequenceGenerator(name = "orders_seq", sequenceName = "ORDERS_SEQ", allocationSize = 1)
  @EqualsAndHashCode.Include
  private Long id;

  @Column(name = "ORDER_NUMBER", nullable = false, unique = true, length = 50)
  private String orderNumber;

  @Column(name = "CUSTOMER_ID", nullable = false)
  private Long customerId;

  @Enumerated(EnumType.STRING)
  @Column(name = "STATUS", nullable = false, length = 20)
  private OrderStatus status;

  @Column(name = "ORDER_DATE")
  private LocalDateTime orderDate;

  @CreationTimestamp
  @Column(name = "CREATED_AT", nullable = false, updatable = false)
  private LocalDateTime createdAt;

  @UpdateTimestamp
  @Column(name = "UPDATED_AT")
  private LocalDateTime updatedAt;

  @Column(name = "CREATED_BY", length = 100)
  private String createdBy;

  @Column(name = "UPDATED_BY", length = 100)
  private String updatedBy;

  @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
  private Set<OrderLine> lines = new HashSet<>();

  // Business method
  public void addLine(OrderLine line) {
    lines.add(line);
    line.setOrder(this);
  }
}
```

### Enum for Status
```java
public enum OrderStatus {
  PENDING("P", "Pending approval"),
  APPROVED("A", "Approved"),
  CANCELLED("C", "Cancelled"),
  SHIPPED("S", "Shipped");

  private final String code;
  private final String description;

  OrderStatus(String code, String description) {
    this.code = code;
    this.description = description;
  }

  public String getCode() { return code; }
  public String getDescription() { return description; }
}
```

### Relationships
```java
// One-to-Many
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
private Set<OrderLine> lines;

// Many-to-One
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "CUSTOMER_ID")
private Customer customer;

// One-to-One
@OneToOne(cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
@JoinColumn(name = "INVOICE_ID", unique = true)
private Invoice invoice;

// Many-to-Many
@ManyToMany
@JoinTable(
  name = "ORDER_TAGS",
  joinColumns = @JoinColumn(name = "ORDER_ID"),
  inverseJoinColumns = @JoinColumn(name = "TAG_ID")
)
private Set<Tag> tags;
```

## 4. Spring Data JPA Repositories

### Repository Interface
```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long>, JpaSpecificationExecutor<Order> {
  Optional<Order> findByOrderNumber(String orderNumber);

  List<Order> findByCustomerIdAndStatusOrderByOrderDateDesc(Long customerId, OrderStatus status);

  // Custom query with parameters
  @Query("SELECT o FROM Order o WHERE o.createdAt BETWEEN :startDate AND :endDate")
  List<Order> findByDateRange(
    @Param("startDate") LocalDateTime startDate,
    @Param("endDate") LocalDateTime endDate
  );

  // Native query for complex logic
  @Query(
    value = "SELECT * FROM ORDERS WHERE CUSTOMER_ID = :customerId AND STATUS = :status ORDER BY CREATED_AT DESC",
    nativeQuery = true
  )
  List<Order> findOrdersByCustomerAndStatus(
    @Param("customerId") Long customerId,
    @Param("status") String status
  );

  // Count/exists queries
  long countByStatus(OrderStatus status);
  boolean existsByOrderNumber(String orderNumber);

  // Page-based queries
  Page<Order> findByStatus(OrderStatus status, Pageable pageable);
}
```

### Specifications (Dynamic Queries)
```java
public class OrderSpecifications {
  public static Specification<Order> byStatus(OrderStatus status) {
    return (root, query, cb) -> cb.equal(root.get("status"), status);
  }

  public static Specification<Order> byCustomerId(Long customerId) {
    return (root, query, cb) -> cb.equal(root.get("customerId"), customerId);
  }

  public static Specification<Order> createdAfter(LocalDateTime date) {
    return (root, query, cb) -> cb.greaterThanOrEqualTo(root.get("createdAt"), date);
  }
}

// Usage in service
List<Order> orders = orderRepo.findAll(
  byStatus(PENDING)
    .and(byCustomerId(123L))
    .and(createdAfter(LocalDateTime.now().minusDays(7)))
);
```

## 5. REST Controller Best Practices

### Basic Controller
```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Tag(name = "Orders", description = "Order management endpoints")
public class OrderController {
  private final OrderService service;
  private final OrderMapper mapper;

  @PostMapping
  @Operation(summary = "Create a new order", description = "Creates an order with PENDING status")
  @ApiResponse(responseCode = "201", description = "Order created")
  @ApiResponse(responseCode = "400", description = "Invalid request")
  @ApiResponse(responseCode = "409", description = "Order number exists")
  public ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody CreateOrderRequest req) {
    OrderResponse resp = service.createOrder(req);
    return ResponseEntity
      .created(URI.create("/api/v1/orders/" + resp.getId()))
      .body(resp);
  }

  @GetMapping("/{id}")
  @Operation(summary = "Get order by ID")
  public ResponseEntity<OrderResponse> getOrder(@PathVariable Long id) {
    return service.getOrder(id)
      .map(ResponseEntity::ok)
      .orElse(ResponseEntity.notFound().build());
  }

  @PutMapping("/{id}")
  @Operation(summary = "Update order")
  public ResponseEntity<OrderResponse> updateOrder(
    @PathVariable Long id,
    @Valid @RequestBody UpdateOrderRequest req
  ) {
    OrderResponse resp = service.updateOrder(id, req);
    return ResponseEntity.ok(resp);
  }

  @DeleteMapping("/{id}")
  @Operation(summary = "Delete order")
  @ResponseStatus(HttpStatus.NO_CONTENT)
  public void deleteOrder(@PathVariable Long id) {
    service.deleteOrder(id);
  }

  @ExceptionHandler(OrderNotFoundException.class)
  public ResponseEntity<ErrorResponse> handleNotFound(OrderNotFoundException e) {
    return ResponseEntity.status(404).body(
      new ErrorResponse("ORDER_NOT_FOUND", e.getMessage())
    );
  }

  @ExceptionHandler(ValidationException.class)
  public ResponseEntity<ErrorResponse> handleValidation(ValidationException e) {
    return ResponseEntity.status(400).body(
      new ErrorResponse("VALIDATION_ERROR", e.getMessage())
    );
  }
}
```

### REST Request/Response DTOs
```java
// Use records for immutable DTOs (Java 17+)
public record CreateOrderRequest(
  @NotBlank(message = "Order number required")
  String orderNumber,

  @Positive(message = "Customer ID must be positive")
  Long customerId,

  @NotNull @DecimalMin("0.01")
  BigDecimal orderTotal,

  LocalDateTime orderDate
) {}

public record OrderResponse(
  Long id,
  String orderNumber,
  Long customerId,
  OrderStatus status,
  LocalDateTime orderDate,
  LocalDateTime createdAt
) {}

public record ErrorResponse(
  String code,
  String message,
  LocalDateTime timestamp
) {
  public ErrorResponse(String code, String message) {
    this(code, message, LocalDateTime.now());
  }
}
```

## 6. Service Layer

### Service with Transactional Semantics
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderService {
  private final OrderRepository orderRepo;
  private final OrderValidator validator;
  private final OrderMapper mapper;
  private final OrderEventPublisher eventPublisher;
  private final CustomerClient customerClient;

  // Legacy: B-001 — Create order with PENDING status
  @Transactional
  public OrderResponse createOrder(CreateOrderRequest req) {
    validator.validate(req);

    // Verify customer exists
    customerClient.getCustomer(req.getCustomerId())
      .orElseThrow(() -> new CustomerNotFoundException(req.getCustomerId()));

    Order order = Order.builder()
      .orderNumber(req.getOrderNumber())
      .customerId(req.getCustomerId())
      .status(OrderStatus.PENDING)
      .orderDate(req.getOrderDate() != null ? req.getOrderDate() : LocalDateTime.now())
      .build();

    Order saved = orderRepo.save(order);
    log.info("Order created: {} (ID: {})", saved.getOrderNumber(), saved.getId());

    // Publish event asynchronously
    eventPublisher.publishOrderCreated(mapper.toEvent(saved));

    return mapper.toResponse(saved);
  }

  @Transactional(readOnly = true)
  public Optional<OrderResponse> getOrder(Long id) {
    return orderRepo.findById(id).map(mapper::toResponse);
  }

  // Legacy: B-003 — Approve order (only from PENDING)
  @Transactional(isolation = Isolation.SERIALIZABLE)
  public OrderResponse approveOrder(Long orderId) {
    Order order = orderRepo.findById(orderId)
      .orElseThrow(() -> new OrderNotFoundException(orderId));

    if (order.getStatus() != OrderStatus.PENDING) {
      throw new InvalidStateException(
        "Cannot approve order in " + order.getStatus() + " status"
      );
    }

    order.setStatus(OrderStatus.APPROVED);
    Order updated = orderRepo.save(order);
    log.info("Order approved: {} (ID: {})", updated.getOrderNumber(), updated.getId());

    eventPublisher.publishOrderApproved(mapper.toEvent(updated));

    return mapper.toResponse(updated);
  }

  @Transactional
  public void deleteOrder(Long id) {
    Order order = orderRepo.findById(id)
      .orElseThrow(() -> new OrderNotFoundException(id));
    orderRepo.delete(order);
    log.info("Order deleted: ID {}", id);
  }
}
```

### Validation Component
```java
@Component
@Slf4j
public class OrderValidator {
  private static final String ORDER_NUMBER_PATTERN = "^ORD-\\d{6}$";

  public void validate(CreateOrderRequest req) {
    if (req.orderNumber() == null || req.orderNumber().isBlank()) {
      throw new ValidationException("Order number is required");
    }

    if (!req.orderNumber().matches(ORDER_NUMBER_PATTERN)) {
      throw new ValidationException("Order number must match format ORD-XXXXXX");
    }

    if (req.customerId() == null || req.customerId() <= 0) {
      throw new ValidationException("Customer ID must be positive");
    }

    if (req.orderTotal() == null || req.orderTotal().compareTo(BigDecimal.ZERO) <= 0) {
      throw new ValidationException("Order total must be > 0");
    }
  }
}
```

## 7. HTTP Interface Clients (Spring Boot 4.x)

### Client Interface
```java
@HttpExchange("/api/v1/customers")
public interface CustomerClient {
  @GetExchange("/{id}")
  Optional<CustomerResponse> getCustomer(@PathVariable Long id);

  @PostExchange
  CustomerResponse createCustomer(@RequestBody CreateCustomerRequest req);

  @PutExchange("/{id}")
  CustomerResponse updateCustomer(@PathVariable Long id, @RequestBody UpdateCustomerRequest req);
}

public record CustomerResponse(Long id, String name, String email) {}
public record CreateCustomerRequest(@NotBlank String name, @Email String email) {}
public record UpdateCustomerRequest(String name, String email) {}
```

### Client Configuration
```java
@Configuration
public class HttpClientConfig {
  @Bean
  CustomerClient customerClient(RestClient.Builder builder) {
    return HttpServiceProxyFactory
      .builderFor(builder.baseUrl("${customer-service.url:http://customer-service:8080}").build())
      .build()
      .createClient(CustomerClient.class);
  }

  @Bean
  PaymentClient paymentClient(RestClient.Builder builder) {
    return HttpServiceProxyFactory
      .builderFor(builder.baseUrl("${payment-service.url:http://payment-service:8080}").build())
      .build()
      .createClient(PaymentClient.class);
  }
}
```

## 8. Kafka Messaging

### Kafka Producer
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventPublisher {
  private final KafkaTemplate<String, OrderCreatedEvent> createdTemplate;
  private final KafkaTemplate<String, OrderApprovedEvent> approvedTemplate;

  public void publishOrderCreated(OrderCreatedEvent event) {
    createdTemplate.send(
      new ProducerRecord<>("orders.created", event.orderId().toString(), event)
    ).whenComplete((result, ex) -> {
      if (ex != null) {
        log.error("Failed to publish OrderCreatedEvent for order {}", event.orderId(), ex);
      } else {
        log.info("OrderCreatedEvent published for order {}", event.orderId());
      }
    });
  }

  public void publishOrderApproved(OrderApprovedEvent event) {
    approvedTemplate.send("orders.approved", event.orderId().toString(), event);
  }
}
```

### Kafka Consumer
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class PaymentEventListener {
  private final OrderService orderService;

  // Legacy: B-010 — Listen for PaymentApproved and update order
  @KafkaListener(topics = "payments.approved", groupId = "order-service")
  public void handlePaymentApproved(PaymentApprovedEvent event) {
    log.info("Received PaymentApprovedEvent for order {}", event.orderId());
    orderService.markOrderAsApproved(event.orderId());
  }

  @KafkaListener(topics = "payments.failed", groupId = "order-service")
  public void handlePaymentFailed(PaymentFailedEvent event) {
    log.warn("Received PaymentFailedEvent for order {}", event.orderId());
    orderService.cancelOrder(event.orderId(), event.reason());
  }
}
```

### Kafka Configuration
```java
@Configuration
@EnableKafka
public class KafkaConfig {
  @Value("${spring.kafka.bootstrap-servers}")
  private String bootstrapServers;

  @Bean
  ProducerFactory<String, OrderCreatedEvent> producerFactory() {
    return new DefaultKafkaProducerFactory<>(producerConfigs());
  }

  private Map<String, Object> producerConfigs() {
    return Map.ofEntries(
      Map.entry(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers),
      Map.entry(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class),
      Map.entry(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class),
      Map.entry(ProducerConfig.ACKS_CONFIG, "all"),
      Map.entry(ProducerConfig.RETRIES_CONFIG, 3)
    );
  }

  @Bean
  KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate() {
    return new KafkaTemplate<>(producerFactory());
  }

  @Bean
  ConsumerFactory<String, OrderCreatedEvent> consumerFactory() {
    return new DefaultKafkaConsumerFactory<>(
      Map.of(
        ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers,
        ConsumerConfig.GROUP_ID_CONFIG, "order-service",
        ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class,
        ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class,
        JsonDeserializer.VALUE_DEFAULT_TYPE, "com.acme.orderservice.event.OrderCreatedEvent"
      )
    );
  }
}
```

### Event DTOs
```java
public record OrderCreatedEvent(
  @NotNull Long orderId,
  @NotBlank String orderNumber,
  @NotNull Long customerId,
  @NotNull LocalDateTime createdAt
) {}

public record OrderApprovedEvent(
  @NotNull Long orderId,
  @NotBlank String orderNumber,
  @NotNull LocalDateTime approvedAt
) {}

public record PaymentApprovedEvent(
  @NotNull Long orderId,
  @NotNull BigDecimal amount,
  @NotNull LocalDateTime processedAt
) {}

public record PaymentFailedEvent(
  @NotNull Long orderId,
  @NotBlank String reason,
  @NotNull LocalDateTime failedAt
) {}
```

## 9. Spring Security 7.0

### Security Configuration
```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {
  private final JwtConverter jwtConverter;

  @Bean
  SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
      .authorizeHttpRequests(auth -> auth
        .requestMatchers("/actuator/health").permitAll()
        .requestMatchers("/api/v1/orders/**").hasRole("USER")
        .requestMatchers("/api/v1/orders/*/approve").hasRole("MANAGER")
        .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
        .anyRequest().authenticated()
      )
      .oauth2ResourceServer(oauth2 -> oauth2
        .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtConverter))
      )
      .csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))
      .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
      .exceptionHandling(ex -> ex
        .authenticationEntryPoint((req, resp, ex2) -> {
          resp.setContentType("application/json");
          resp.setStatus(401);
          resp.getWriter().write("{\"error\":\"Unauthorized\"}");
        })
      );
    return http.build();
  }

  @Bean
  JwtDecoder jwtDecoder() {
    return JwtDecoders.fromIssuerLocation("${spring.security.oauth2.resourceserver.jwt.issuer-uri}");
  }

  @Bean
  PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
  }
}
```

## 10. Testing with Spring Boot Test & Testcontainers

### Unit Test with Mocks
```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
  @Mock
  private OrderRepository orderRepo;

  @Mock
  private OrderValidator validator;

  @Mock
  private OrderEventPublisher eventPublisher;

  @InjectMocks
  private OrderService service;

  @Test
  void testCreateOrderSetsStatusToPending() {
    // Arrange
    CreateOrderRequest req = new CreateOrderRequest("ORD-000001", 1L, new BigDecimal("100.00"), null);
    Order saved = Order.builder()
      .id(1L)
      .orderNumber("ORD-000001")
      .customerId(1L)
      .status(OrderStatus.PENDING)
      .build();
    when(orderRepo.save(any())).thenReturn(saved);

    // Act
    OrderResponse resp = service.createOrder(req);

    // Assert
    assertEquals(OrderStatus.PENDING, resp.getStatus());
    verify(validator).validate(req);
    verify(eventPublisher).publishOrderCreated(any());
  }

  @Test
  void testApproveOrderInvalidStateThrows() {
    // Arrange
    Order order = Order.builder().id(1L).status(OrderStatus.APPROVED).build();
    when(orderRepo.findById(1L)).thenReturn(Optional.of(order));

    // Act & Assert
    assertThrows(InvalidStateException.class, () -> service.approveOrder(1L));
  }
}
```

### Integration Test with Testcontainers
```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {
  @Container
  static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>(DockerImageName.parse("postgres:15"))
    .withDatabaseName("testdb")
    .withUsername("testuser")
    .withPassword("testpass");

  @DynamicPropertySource
  static void registerProperties(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", postgres::getJdbcUrl);
    registry.add("spring.datasource.username", postgres::getUsername);
    registry.add("spring.datasource.password", postgres::getPassword);
  }

  @Autowired
  private OrderRepository orderRepo;

  @Test
  void testFindByOrderNumber() {
    // Arrange
    Order order = Order.builder()
      .orderNumber("ORD-000001")
      .customerId(1L)
      .status(OrderStatus.PENDING)
      .build();
    orderRepo.save(order);

    // Act
    Optional<Order> found = orderRepo.findByOrderNumber("ORD-000001");

    // Assert
    assertTrue(found.isPresent());
    assertEquals("ORD-000001", found.get().getOrderNumber());
  }
}
```

### Controller Test with RestTestClient
```java
@SpringBootTest
@AutoConfigureHttpMessageCodecs
class OrderControllerTest {
  @Autowired
  private ApplicationContext context;

  private RestClient restClient;

  @MockBean
  private OrderService service;

  @BeforeEach
  void setUp() {
    restClient = RestClient.builder()
      .baseUrl("http://localhost:8080")
      .messageConverters(converters -> converters.add(new MappingJackson2HttpMessageConverter()))
      .build();
  }

  @Test
  void testCreateOrderReturns201() {
    // Arrange
    CreateOrderRequest req = new CreateOrderRequest("ORD-000001", 1L, new BigDecimal("100.00"), null);
    OrderResponse resp = new OrderResponse(1L, "ORD-000001", 1L, OrderStatus.PENDING, null, LocalDateTime.now());
    when(service.createOrder(any())).thenReturn(resp);

    // Act & Assert
    ResponseEntity<OrderResponse> result = restClient.post()
      .uri("/api/v1/orders")
      .body(req)
      .retrieve()
      .toEntity(OrderResponse.class);

    assertEquals(HttpStatus.CREATED, result.getStatusCode());
    assertEquals(1L, result.getBody().id());
  }
}
```

## 11. Application Configuration

### application.yml Structure
```yaml
spring:
  application:
    name: order-service
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQL15Dialect
        format_sql: true
        jdbc:
          time_zone: UTC
  datasource:
    url: jdbc:postgresql://localhost:5432/orderdb
    username: postgres
    password: postgres
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      acks: all
      retries: 3
    consumer:
      auto-offset-reset: earliest
      group-id: order-service
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth-server/.well-known/openid-configuration

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus,info
  endpoint:
    health:
      show-details: when-authorized
  tracing:
    sampling:
      probability: 1.0
  otlp:
    tracing:
      endpoint: http://jaeger:4317

logging:
  level:
    root: INFO
    com.acme: DEBUG
  pattern:
    console: "%d{ISO8601} [%thread] %-5level %logger{36} - %msg%n"
```

## 12. Flyway Database Migrations

### Naming Convention
```
V001__Create_orders_table.sql
V002__Add_audit_columns.sql
V003__Create_order_lines_table.sql
```

### Migration File Example
```sql
-- V001__Create_orders_table.sql
CREATE TABLE orders (
  id BIGSERIAL PRIMARY KEY,
  order_number VARCHAR(50) NOT NULL UNIQUE,
  customer_id BIGINT NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
  order_date TIMESTAMP,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP,
  created_by VARCHAR(100),
  updated_by VARCHAR(100),
  CONSTRAINT fk_customer FOREIGN KEY (customer_id) REFERENCES customers(id)
);

CREATE SEQUENCE orders_seq START WITH 1;
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at);
```
