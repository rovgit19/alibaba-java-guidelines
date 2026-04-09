# Modern Java (17+) Supplement for Spring Boot Projects

> Complements the Alibaba Java Coding Guidelines with modern Java features and practices.
> These rules apply alongside P3C, not as replacements.

> Severity levels follow P3C convention: **[Mandatory]** / **[Recommended]** / **[For Reference]**

---

## 1. Records & Immutability

**[Recommended]** Prefer records for DTOs and value objects. Records are immutable by design, eliminating boilerplate and preventing accidental mutation.

```java
// GOOD: Use record for DTO — auto-generates equals/hashCode/toString (aligns with P3C OOP Rule #13)
public record MarketDTO(Long id, String name, MarketStatus status) {}

// GOOD: Immutable entity fields with getters only
public class Market {
    private final Long id;
    private final String name;
    // getters only, no setters
}

// BAD: Mutable DTO with public setters — violates immutability principle
public class MarketDTO {
    private Long id;
    public void setId(Long id) { this.id = id; } // avoid
}
```

**When NOT to use records:**
- JPA/MyBatis entity classes (need mutable fields, no-arg constructor)
- Classes requiring inheritance
- Classes needing custom equals/hashCode logic

---

## 2. Optional Usage

**[Recommended]** Return `Optional` from query methods that may return null. Never use Optional as a field or parameter.

```java
// GOOD: Return Optional from find* methods
public Optional<Market> findBySlug(String slug) {
    return Optional.ofNullable(marketRepository.selectBySlug(slug));
}

// GOOD: Chain with map/flatMap, never call get() directly
return marketRepository.findBySlug(slug)
    .map(MarketResponse::from)
    .orElseThrow(() -> new EntityNotFoundException("Market", slug));

// GOOD: Provide default value
String name = optional.orElse("unknown");

// BAD: Using Optional as method parameter
public void process(Optional<String> name) {} // avoid

// BAD: Using Optional as class field
private Optional<String> nickname; // avoid — use @Nullable instead

// BAD: Calling get() without isPresent() check
String value = optional.get(); // may throw NoSuchElementException
```

---

## 3. Stream Best Practices

**[Recommended]** Use streams for transformations. Keep pipelines short (≤3-4 operations). Prefer loops for complex logic.

```java
// GOOD: Short, readable pipeline
List<String> activeNames = markets.stream()
    .filter(m -> m.getStatus() == MarketStatus.ACTIVE)
    .map(Market::getName)
    .filter(Objects::nonNull)
    .toList(); // Java 16+, replaces .collect(Collectors.toList())

// GOOD: Use toMap with merge function to avoid duplicate key exception
Map<Long, Market> marketMap = markets.stream()
    .collect(Collectors.toMap(Market::getId, Function.identity(), (a, b) -> a));

// BAD: Complex nested streams — use a for loop instead
markets.stream()
    .flatMap(m -> m.getItems().stream()
        .filter(i -> i.getPrice().compareTo(threshold) > 0)
        .map(i -> new Pair<>(m, i)))
    .collect(groupingBy(p -> p.getLeft().getCategory(),
        mapping(Pair::getRight, toList()))); // unreadable — refactor to loop

// GOOD: Equivalent for loop — explicit, debuggable, readable
Map<Category, List<Item>> result = new HashMap<>();
for (Market m : markets) {
    for (Item i : m.getItems()) {
        if (i.getPrice().compareTo(threshold) > 0) {
            result.computeIfAbsent(m.getCategory(), k -> new ArrayList<>()).add(i);
        }
    }
}

// CAUTION: Streams are single-use. Never store and reuse a stream reference.
// CAUTION: parallelStream() rarely helps. Profile before using.
```

---

## 4. Generics & Type Safety

**[Recommended]** Always parameterize generic types. Use bounded wildcards (PECS) for flexible APIs.

```java
// GOOD: Bounded generics for reusable utilities
public <T extends Identifiable> Map<Long, T> indexById(Collection<T> items) {
    return items.stream()
        .collect(Collectors.toMap(Identifiable::getId, Function.identity()));
}

// GOOD: Wildcard for read-only parameters (PECS: Producer Extends, Consumer Super)
public void printAll(List<? extends BaseEntity> entities) {
    entities.forEach(e -> log.info("{}", e));
}

// BAD: Raw types — always parameterize (aligns with P3C Collection Rule #8)
List list = new ArrayList(); // avoid — use List<String>
```

---

## 5. Null Handling Strategy

**Layer-by-layer null defense:**

```java
// Layer 1: Bean Validation on Controller inputs
public ResponseEntity<?> create(@RequestBody @Valid CreateMarketRequest request) {}

public record CreateMarketRequest(
    @NotBlank String name,
    @NotNull MarketType type,
    @Size(max = 500) String description // nullable field is OK — no @NotNull
) {}

// Layer 2: Optional for Service query returns (see Section 2)

// Layer 3: Objects.requireNonNull for critical internal args
public MarketService(MarketRepository repo) {
    this.repo = Objects.requireNonNull(repo, "MarketRepository must not be null");
}

// Layer 4: @Nullable annotation for unavoidable null fields
public void process(@Nullable String optionalNote) {
    if (optionalNote != null) { ... }
}
```

---

## 6. Exception Design for Spring Boot

**Build a domain exception hierarchy aligned with HTTP semantics:**

```java
// Base exception — two constructors: one for originating exceptions, one for wrapping a cause
public abstract class DomainException extends RuntimeException {
    private final String errorCode;

    protected DomainException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    protected DomainException(String errorCode, String message, Throwable cause) {
        super(message, cause); // always chain cause for exception translation (Effective Java Item 73)
        this.errorCode = errorCode;
    }

    public String getErrorCode() { return errorCode; }
}

// Specific exceptions — name reflects business meaning (P3C: use custom exceptions, not raw RuntimeException)
public class EntityNotFoundException extends DomainException {
    public EntityNotFoundException(String entity, Object id) {
        super("NOT_FOUND", entity + " not found: " + id);
    }
}

public class BusinessRuleViolationException extends DomainException {
    public BusinessRuleViolationException(String rule) {
        super("BUSINESS_RULE_VIOLATION", rule);
    }
}

// Layer-specific exceptions for exception translation (see Section 12, Item 73)
public class RepositoryException extends DomainException {
    public RepositoryException(String message, Throwable cause) {
        super("REPOSITORY_ERROR", message, cause);
    }
}

public class ServiceException extends DomainException {
    public ServiceException(String message, Throwable cause) {
        super("SERVICE_ERROR", message, cause);
    }
}

// Global handler — single place for exception → HTTP mapping
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handle(EntityNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String msg = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining("; "));
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("VALIDATION_ERROR", msg));
    }
}

public record ErrorResponse(String code, String message) {}
```

---

## 7. Testing Conventions (JUnit 5 + Spring Boot)

```java
// Naming: method_scenario_expected (P3C: test class name = TestedClass + "Test")
@ExtendWith(MockitoExtension.class)
class MarketServiceTest {

    @Mock
    private MarketRepository repo;

    @InjectMocks
    private MarketService service;

    private Market testMarket = new Market(1L, "Test Market", MarketStatus.ACTIVE);

    @Test
    void findBySlug_existingMarket_returnsMarket() {
        // Given — setup
        when(repo.findBySlug("test")).thenReturn(Optional.of(testMarket));

        // When — execute
        MarketDTO result = service.findBySlug("test");

        // Then — verify (use AssertJ for fluent assertions)
        assertThat(result)
            .isNotNull()
            .extracting(MarketDTO::name)
            .isEqualTo("Test Market");
    }

    @Test
    void findBySlug_nonExistent_throwsNotFoundException() {
        when(repo.findBySlug("missing")).thenReturn(Optional.empty());

        assertThatThrownBy(() -> service.findBySlug("missing"))
            .isInstanceOf(EntityNotFoundException.class)
            .hasMessageContaining("missing");
    }
}

// Integration test with Spring context
@SpringBootTest
@Transactional // auto-rollback after each test
class MarketRepositoryIT {

    @Autowired
    private MarketRepository repo;

    @Test
    void save_validMarket_persistsSuccessfully() {
        Market market = new Market("test", MarketType.SPORTS);
        Market saved = repo.save(market);
        assertThat(saved.getId()).isNotNull();
    }
}
```

**Testing rules:**
- Unit tests: Mockito for dependencies, no Spring context, fast
- Integration tests: `@SpringBootTest` + `@Transactional`, test real DB interactions
- No `Thread.sleep()` in tests — use `Awaitility` for async assertions
- Test method naming: `methodName_scenario_expectedResult`

### Test Slices — prefer over full @SpringBootTest

**[Recommended]** Use targeted test slices to load only the relevant layer. `@SpringBootTest` loads the full context and is slow — reserve it for true end-to-end tests.

```java
// Controller layer only — loads Spring MVC infrastructure, no service/repo beans
@WebMvcTest(MarketController.class)
class MarketControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private MarketService marketService;  // inject mock, not real bean

    @Test
    void getMarket_validId_returns200() throws Exception {
        when(marketService.findById(1L)).thenReturn(new MarketDTO(1L, "Test", MarketStatus.ACTIVE));

        mockMvc.perform(get("/api/markets/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Test"));
    }
}

// Repository layer only — loads JPA/MyBatis + embedded DB, no web/service beans
@DataJpaTest  // or @MybatisTest for MyBatis
@Transactional  // auto-rollback after each test
class MarketRepositoryTest {

    @Autowired
    private MarketRepository repo;

    @Test
    void findBySlug_existingRecord_returnsMarket() {
        repo.save(new Market("test-slug", MarketType.SPORTS));
        Optional<Market> result = repo.findBySlug("test-slug");
        assertThat(result).isPresent();
    }
}
```

| Slice | What it loads | Use for |
|---|---|---|
| `@WebMvcTest` | MVC layer only | Controller + filter + validation |
| `@DataJpaTest` | JPA + embedded DB | Repository queries |
| `@JsonTest` | Jackson serialization | DTO serialization/deserialization |
| `@SpringBootTest` | Full context | End-to-end / integration smoke tests |

---

## 8. Spring Dependency Injection & Configuration

### Constructor Injection

**[Mandatory]** Use constructor injection. Never use `@Autowired` on fields.

```java
// GOOD: Constructor injection — immutable, testable, no reflection tricks needed
@Service
public class MarketService {
    private final MarketRepository repo;
    private final EventPublisher publisher;

    public MarketService(MarketRepository repo, EventPublisher publisher) {
        this.repo = repo;
        this.publisher = publisher;
    }
}

// GOOD: Lombok shortcut (same as above)
@Service
@RequiredArgsConstructor
public class MarketService {
    private final MarketRepository repo;
    private final EventPublisher publisher;
}

// BAD: Field injection — hides dependencies, prevents immutability, hard to test
@Service
public class MarketService {
    @Autowired
    private MarketRepository repo; // avoid
}
```

**Why constructor injection:**
- Dependencies are explicit and required at construction time — no partial object state
- Fields can be `final` — class is immutable after construction
- Unit tests can inject mocks directly with `new MarketService(mockRepo, mockPublisher)` — no Spring context needed

### @ConfigurationProperties

**[Recommended]** Use `@ConfigurationProperties` for structured config. Avoid scattered `@Value` for related properties.

```java
// GOOD: Typed, validated config group (Spring Boot 3.x — records supported natively)
@ConfigurationProperties(prefix = "market")
@Validated
public record MarketProperties(
    @NotNull Duration cacheTtl,
    @Min(1) @Max(100) int maxItemsPerPage,
    @NotBlank String defaultCurrency
) {}

// Spring Boot 2.x: use @ConstructorBinding on the record/class instead
// @ConstructorBinding
// @ConfigurationProperties(prefix = "market")

// Register in @SpringBootApplication or config class
@EnableConfigurationProperties(MarketProperties.class)

// application.yml
// market:
//   cache-ttl: PT10M
//   max-items-per-page: 20
//   default-currency: CNY

// BAD: Scattered @Value — no type safety, no validation, hard to refactor
@Value("${market.cache-ttl}")
private Duration cacheTtl;
@Value("${market.max-items-per-page}")
private int maxItemsPerPage;
```

---

## 9. Spring Transaction Propagation

**[Mandatory]** Understand propagation levels before using `@Transactional`. Wrong propagation causes silent data corruption or unnecessary performance overhead.

```java
// GOOD: Default REQUIRED — joins existing transaction or creates new one
// Use for most service methods that write to DB
@Transactional
public void placeOrder(OrderRequest req) { ... }

// GOOD: REQUIRES_NEW — always starts a NEW independent transaction
// Use for audit logs, notifications — must commit even if outer transaction rolls back
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAuditLog(AuditEvent event) { ... }

// GOOD: NOT_SUPPORTED — suspends current transaction, runs non-transactionally
// Use for read-only queries that should not hold a DB connection in a transaction
@Transactional(propagation = Propagation.NOT_SUPPORTED, readOnly = true)
public List<Market> listMarkets() { ... }

// BAD: @Transactional on private methods — Spring AOP proxy won't intercept them
@Transactional
private void internalUpdate() { ... } // annotation has no effect

// BAD: Calling @Transactional method from within the same class
// Self-invocation bypasses the proxy — transaction won't start
public void outerMethod() {
    this.transactionalMethod(); // proxy bypassed — no transaction
}
```

**Rules:**
- **[Mandatory]** Do not call `@Transactional` methods via `this.` within the same class — inject self or extract to another bean
- **[Mandatory]** Set `readOnly = true` for pure read methods — enables DB optimizations and prevents accidental writes
- **[Recommended]** Keep transactions short. Move non-DB work (HTTP calls, MQ sends, file I/O) outside the transactional boundary (aligns with P3C ORM Rule #9)
- **[Recommended]** Always specify `rollbackFor` when catching checked exceptions: `@Transactional(rollbackFor = Exception.class)`

---

## 10. @Async Usage

**[Recommended]** Use `@Async` for fire-and-forget tasks (email, push notifications, audit logs). Requires explicit thread pool configuration — never rely on the default `SimpleAsyncTaskExecutor` in production.

```java
// GOOD: Configure a named thread pool bean
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean("asyncTaskExecutor")
    public Executor asyncTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

// GOOD: Always reference the named executor; return CompletableFuture for trackable tasks
@Async("asyncTaskExecutor")
public CompletableFuture<Void> sendWelcomeEmail(String userId) {
    try {
        emailService.send(userId);
        return CompletableFuture.completedFuture(null);
    } catch (Exception e) {
        log.error("send_email_failed userId={} reason={}", userId, e.getMessage(), e);
        return CompletableFuture.failedFuture(e);
    }
}

// BAD: @Async on a private method — proxy won't intercept
@Async
private void asyncInternal() { ... } // no effect

// BAD: Calling @Async method from the same class — proxy bypassed
public void trigger() {
    this.sendWelcomeEmail("123"); // runs synchronously
}
```

**Rules:**
- **[Mandatory]** Always configure a custom `ThreadPoolTaskExecutor` — `SimpleAsyncTaskExecutor` creates a new thread per call (no pooling, OOM risk)
- **[Mandatory]** `@Async` methods must be `public` and called from a different bean
- **[Recommended]** Return `CompletableFuture<T>` for tasks that callers may want to track or chain
- **[Recommended]** Always handle exceptions inside `@Async` methods — uncaught exceptions are silently swallowed unless you configure `AsyncUncaughtExceptionHandler`

---

## 11. Java 17 Language Features

### Sealed Classes

**[Recommended]** Use `sealed` classes for closed type hierarchies. Eliminates unchecked casting and makes exhaustive `switch` possible.

```java
// GOOD: Sealed hierarchy — compiler enforces exhaustiveness
public sealed interface PaymentResult
    permits PaymentResult.Success, PaymentResult.Failure, PaymentResult.Pending {}

public record Success(String transactionId) implements PaymentResult {}
public record Failure(String errorCode, String reason) implements PaymentResult {}
public record Pending(String referenceId) implements PaymentResult {}

// GOOD: Exhaustive switch expression — compiler warns if a case is missing
String message = switch (result) {
    case Success s  -> "Paid: " + s.transactionId();
    case Failure f  -> "Failed: " + f.reason();
    case Pending p  -> "Pending: " + p.referenceId();
};

// BAD: Pre-Java 17 approach — open hierarchy + instanceof chain + unchecked casts
if (result instanceof Success) {
    Success s = (Success) result; // verbose, error-prone
} else if (result instanceof Failure) { ... }
```

### Pattern Matching

**[Recommended]** Use pattern matching `instanceof` (Java 16+). Eliminates redundant casts.

```java
// GOOD: Pattern matching — bind and cast in one step
if (event instanceof OrderCreatedEvent e) {
    log.info("order_created orderId={}", e.orderId());
}

// BAD: Old style — check then cast
if (event instanceof OrderCreatedEvent) {
    OrderCreatedEvent e = (OrderCreatedEvent) event; // redundant cast
}
```

### Text Blocks

**[Recommended]** Use text blocks (Java 15+) for multiline strings: SQL, JSON, XML in tests.

```java
// GOOD: Text block for SQL — readable, no escape hell
String sql = """
    SELECT id, name, status
    FROM market
    WHERE status = 'ACTIVE'
      AND gmt_create > ?
    ORDER BY id DESC
    LIMIT 20
    """;

// GOOD: Text block for JSON in tests
mockMvc.perform(post("/api/markets")
    .contentType(MediaType.APPLICATION_JSON)
    .content("""
        {
            "name": "Test Market",
            "type": "SPORTS"
        }
        """))
    .andExpect(status().isCreated());

// BAD: String concatenation for multiline — unreadable
String sql = "SELECT id, name " +
             "FROM market " +
             "WHERE status = 'ACTIVE'";
```

### Switch Expressions

**[Recommended]** Use switch expressions (Java 14+) to replace verbose switch statements that assign a value.

```java
// GOOD: Switch expression — concise, exhaustive, no fall-through bugs
int discount = switch (userLevel) {
    case VIP      -> 20;
    case PREMIUM  -> 10;
    case STANDARD -> 0;
};

// GOOD: With arrow + block for complex logic
String label = switch (status) {
    case ACTIVE   -> "Active";
    case INACTIVE -> "Inactive";
    case PENDING  -> {
        log.warn("pending_status_encountered id={}", id);
        yield "Pending Review";
    }
};

// BAD: Traditional switch statement — fall-through risk, verbose
int discount;
switch (userLevel) {
    case VIP:     discount = 20; break;
    case PREMIUM: discount = 10; break;
    default:      discount = 0;
}
```

---

## 12. Effective Java Supplements

### Item 54 — Return Empty Collections/Arrays, Not Null

**[Mandatory]** Never return `null` for a method that conceptually returns a collection or array. Callers must always null-check, and forgetting causes NPE.

```java
// GOOD: Return empty collection — callers need no null check
public List<Item> findItemsByMarket(Long marketId) {
    List<Item> items = itemMapper.selectByMarketId(marketId);
    return items != null ? items : Collections.emptyList();
}

// GOOD: With Optional on the query layer, unwrap before returning list
public List<OrderDTO> listOrders(Long userId) {
    return orderMapper.selectByUserId(userId)
        .stream()
        .map(OrderDTO::from)
        .toList();
}

// BAD: Returning null — forces every caller to null-check
public List<Item> findItemsByMarket(Long marketId) {
    return itemMapper.selectByMarketId(marketId); // may return null from ORM
}
```

**Rules:**
- **[Mandatory]** Return `Collections.emptyList()` / `Collections.emptyMap()` / `new ArrayList<>()` — never `null` for collection return types
- **[Mandatory]** Return `new T[0]` — never `null` for array return types
- **[For Reference]** `Collections.emptyList()` returns a shared singleton — safe and allocation-free

---

### Item 2 — Builder for Objects with Many Parameters

**[Recommended]** When a class has 4+ parameters (especially optional ones), use the Builder pattern. Telescoping constructors and all-arg constructors with many parameters are hard to read and error-prone.

```java
// GOOD: Builder via Lombok — concise, readable call sites
@Builder
@Getter
public class CreateOrderCommand {
    private final Long userId;
    private final Long marketId;
    private final BigDecimal amount;
    private final String currency;
    private final String note;          // optional
    private final Instant scheduledAt;  // optional

    // Validation in builder — fail fast before object is constructed
    public static class CreateOrderCommandBuilder {
        public CreateOrderCommand build() {
            Objects.requireNonNull(userId, "userId required");
            Objects.requireNonNull(marketId, "marketId required");
            Objects.requireNonNull(amount, "amount required");
            if (amount.compareTo(BigDecimal.ZERO) <= 0) {
                throw new IllegalArgumentException("amount must be positive");
            }
            return new CreateOrderCommand(userId, marketId, amount,
                currency != null ? currency : "CNY", note, scheduledAt);
        }
    }
}

// Call site — self-documenting, no parameter position mistakes
CreateOrderCommand cmd = CreateOrderCommand.builder()
    .userId(123L)
    .marketId(456L)
    .amount(new BigDecimal("99.00"))
    .note("VIP order")
    .build();

// BAD: All-arg constructor with many params — which Long is which?
new CreateOrderCommand(123L, 456L, new BigDecimal("99.00"), "CNY", null, null);
```

**When to use Builder:**
- 4+ parameters, especially when several are optional
- Immutable objects that can't use setters
- When parameter combinations have constraints that should be validated at construction

---

### Item 18 — Composition over Inheritance

**[Recommended]** Prefer composition (wrapping) to inheritance when reusing behavior from another class. Inheritance breaks encapsulation — a subclass depends on implementation details of its superclass.

```java
// GOOD: Composition — wrap and delegate only what you need
public class InstrumentedList<E> implements List<E> {
    private final List<E> delegate;
    private int addCount = 0;

    public InstrumentedList(List<E> delegate) {
        this.delegate = delegate;
    }

    @Override
    public boolean add(E e) {
        addCount++;
        return delegate.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return delegate.addAll(c);
    }

    public int getAddCount() { return addCount; }

    // delegate all other List methods to this.delegate
}

// BAD: Inheritance — addAll internally calls add, so addCount gets double-counted
public class InstrumentedList<E> extends ArrayList<E> {
    private int addCount = 0;

    @Override
    public boolean add(E e) { addCount++; return super.add(e); }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c); // internally calls add() again — addCount doubled
    }
}
```

**Rules:**
- **[Recommended]** Extend only when there is a true is-a relationship AND you control the superclass
- **[Recommended]** Prefer wrapping (decorator pattern) when you want to add behavior to an existing class
- **[Recommended]** Mark classes `final` if not designed for inheritance (aligns with P3C OOP Rule #19)
- **[For Reference]** Abstract classes are appropriate for skeletal implementations of interfaces (Template Method pattern)

---

### Items 34-37 — Enum Best Practices

**[Recommended]** Use enums instead of int constants. Enums can carry data and behavior — don't reduce them to plain labels.

```java
// GOOD: Enum with data and behavior — self-contained, type-safe
public enum OrderStatus {
    PENDING("待支付", false),
    PAID("已支付", true),
    CANCELLED("已取消", false),
    REFUNDED("已退款", false);

    private final String label;
    private final boolean paid;

    OrderStatus(String label, boolean paid) {
        this.label = label;
        this.paid = paid;
    }

    public String getLabel() { return label; }
    public boolean isPaid() { return paid; }

    // Behavior in enum — avoids scattered instanceof/switch chains in service layer
    public boolean canRefund() {
        return this == PAID;
    }
}

// GOOD: EnumMap — O(1) lookup, more efficient than HashMap for enum keys
Map<OrderStatus, List<Order>> byStatus = new EnumMap<>(OrderStatus.class);

// GOOD: EnumSet — compact bitset implementation for enum subsets
Set<OrderStatus> activeStatuses = EnumSet.of(OrderStatus.PENDING, OrderStatus.PAID);

// BAD: int constants — no type safety, no behavior, no label
public static final int STATUS_PENDING   = 0; // avoid
public static final int STATUS_PAID      = 1;
public static final int STATUS_CANCELLED = 2;

// BAD: Don't use ordinal() as data — ordinal changes if enum order changes
int statusCode = status.ordinal(); // fragile — breaks if enum is reordered
```

**Rules:**
- **[Mandatory]** Never use `ordinal()` as a stored value — use an explicit field (Item 35)
- **[Mandatory]** Use `EnumSet` instead of bit-field `int` masks for enum subsets (Item 36)
- **[Recommended]** Use `EnumMap` instead of `HashMap` when keys are enums — more efficient (Item 37)
- **[Recommended]** Put behavior that depends on enum value into the enum itself, not in external switch statements

---

### Item 73 — Exception Translation

**[Mandatory]** Don't let lower-layer exceptions propagate to upper layers unchanged. Translate them into exceptions meaningful at the caller's abstraction level.

```java
// GOOD: DAO catches low-level exception, wraps in domain exception
public Optional<Market> findById(Long id) {
    try {
        return Optional.ofNullable(marketMapper.selectById(id));
    } catch (DataAccessException e) {
        // Translate: caller doesn't need to know about SQL details
        throw new RepositoryException("Failed to fetch market id=" + id, e); // always chain cause
    }
}

// GOOD: Service translates DAO exception into domain exception
public MarketDTO getMarket(Long id) {
    try {
        return marketRepo.findById(id)
            .map(MarketDTO::from)
            .orElseThrow(() -> new EntityNotFoundException("Market", id));
    } catch (RepositoryException e) {
        throw new ServiceException("market_fetch_failed id=" + id, e);
    }
}

// BAD: SQL exception leaks through service into controller
public MarketDTO getMarket(Long id) throws SQLException { // avoid — exposes persistence detail
    ...
}

// BAD: Exception swallowed — cause lost
catch (DataAccessException e) {
    throw new ServiceException("error"); // never drop the original cause
}
```

**Rules:**
- **[Mandatory]** Always chain the original exception as the `cause` when translating — never swallow it
- **[Mandatory]** Exception type at each layer must match that layer's abstraction: DAO → `RepositoryException`, Service → `ServiceException`, no raw `SQLException`/`DataAccessException` above the DAO layer
- **[Recommended]** If a lower-layer exception is unrecoverable at the current layer, re-throw without translation rather than catching and ignoring

---

## 13. Logging Enhancement (supplement to P3C Logs section)

```java
// Structured key=value format for easy log parsing/searching
log.info("market_created id={} slug={} type={}", market.getId(), market.getSlug(), market.getType());
log.warn("market_not_found slug={}", slug);
log.error("market_create_failed slug={} reason={}", slug, ex.getMessage(), ex);

// GOOD: Use MDC for request-scoped context (trace ID, user ID)
MDC.put("traceId", request.getTraceId());
MDC.put("userId", currentUser.getId().toString());
try {
    // all logs within this scope automatically include traceId and userId
    service.process(request);
} finally {
    MDC.clear(); // always clean up — same principle as ThreadLocal.remove()
}
```
