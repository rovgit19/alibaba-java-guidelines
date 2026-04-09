# Modern Java (17+) Supplement for Spring Boot Projects

> Complements the Alibaba Java Coding Guidelines with modern Java features and practices.
> These rules apply alongside P3C, not as replacements.

---

## 1. Records & Immutability

**Prefer records for DTOs and value objects.** Records are immutable by design, eliminating boilerplate and preventing accidental mutation.

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

**Return `Optional` from query methods that may return null. Never use Optional as a field or parameter.**

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

**Use streams for transformations. Keep pipelines short (≤3-4 operations). Prefer loops for complex logic.**

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

// CAUTION: Streams are single-use. Never store and reuse a stream reference.
// CAUTION: parallelStream() rarely helps. Profile before using.
```

---

## 4. Generics & Type Safety

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
// Base exception
public abstract class DomainException extends RuntimeException {
    private final String errorCode;
    protected DomainException(String errorCode, String message) {
        super(message);
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
class MarketServiceTest {

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

---

## 8. Logging Enhancement (supplement to P3C Logs section)

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
