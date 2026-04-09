# Alibaba Java Coding Guidelines (阿里巴巴Java开发手册)

> Source: https://github.com/alibaba/p3c (黄山版)
> Severity levels: **[Mandatory]** / **[Recommended]** / **[For Reference]**

---

## Table of Contents

1. [Programming Specification](#1-programming-specification)
   - Naming Conventions (L1-L15)
   - Constant Conventions (L16-L20)
   - Formatting Style (L21-L30)
   - OOP Rules (L31-L51)
   - Collection (L52-L66)
   - Concurrency (L67-L83)
   - Flow Control Statements (L84-L91)
   - Code Comments (L92-L102)
   - Other (L103-L110)
2. [Exception and Logs](#2-exception-and-logs)
   - Exception (L111-L123)
   - Logs (L124-L131)
3. [MySQL Rules](#3-mysql-rules)
   - Table Schema Rules
   - Index Rules
   - SQL Rules
   - ORM Rules
4. [Project Specification](#4-project-specification)
5. [Security Specification](#5-security-specification)

---

## 1. Programming Specification

### Naming Conventions

1. **[Mandatory]** Names should not start or end with `_` or `$`.
2. **[Mandatory]** No Chinese, Pinyin, or Pinyin-English mixed naming. Use accurate English. (Exception: established proper nouns like alibaba/taobao/hangzhou)
3. **[Mandatory]** Class names: UpperCamelCase nouns. Exception: DO, BO, DTO, VO. e.g. `MarcoPolo`, `UserDO`, `HtmlDTO`, `XmlService`
4. **[Mandatory]** Method/parameter/member/local variable names: lowerCamelCase. e.g. `localValue`, `getHttpMessage()`, `inputUserId`
5. **[Mandatory]** Constants: UPPER_SNAKE_CASE, semantically complete. e.g. `MAX_STOCK_COUNT` not `MAX_COUNT`
6. **[Mandatory]** Abstract class → `Abstract*` or `Base*`; Exception class → `*Exception`; Test class → `*Test`
7. **[Mandatory]** Array type: `String[] args` not `String args[]`
8. **[Mandatory]** Boolean variable: do NOT prefix with `is` (causes serialization issues in some frameworks). e.g. avoid `boolean isSuccess`
9. **[Mandatory]** Package names: all lowercase, one word per dot, singular. e.g. `com.alibaba.open.util`
10. **[Mandatory]** Avoid uncommon abbreviations. e.g. don't use `AbsClass` for `AbstractClass`
11. **[Recommended]** Include design pattern name in class name. e.g. `OrderFactory`, `LoginProxy`, `ResourceObserver`
12. **[Recommended]** No `public` modifier on interface methods. Add Javadoc. Only define application-wide constants in interfaces.
13. **[Mandatory]** Service/DAO must be interfaces (SOA). Impl class → `*Impl`. e.g. `CacheServiceImpl` implements `CacheService`
14. **[For Reference]** Enum name → `*Enum`; members → UPPER_SNAKE_CASE. e.g. `DealStatusEnum`, `SUCCESS`
15. **[For Reference]** Service/DAO method prefixes: `get` (single), `list` (multiple), `count` (stats), `save`/`insert`, `remove`/`delete`, `update`. Domain models: `*DO` (table), `*DTO` (domain), `*VO` (view). Never use `*POJO`.

### Constant Conventions

1. **[Mandatory]** No magic values. e.g. avoid `"Id#taobao_" + tradeId`
2. **[Mandatory]** Use `L` not `l` for Long literals. e.g. `2L` not `2l`
3. **[Recommended]** Separate constants by function into different classes. e.g. `CacheConsts`, `ConfigConsts`
4. **[Recommended]** 5 sharing levels: cross-app (client.jar) → app → sub-project → package → class (`private static final`)
5. **[Recommended]** Use enum for fixed-range values with attributes.

### Formatting Style

1. **[Mandatory]** Braces: no line break before `{`, line break after `{`, line break before `}`, line break after `}` (except before `else`/`,`). Empty → `{}` on same line.
2. **[Mandatory]** No space between `(` and its following char, or `)` and its preceding char.
3. **[Mandatory]** One space between keywords (`if`/`for`/`while`/`switch`) and `(`.
4. **[Mandatory]** One space around operators: `=`, `&&`, `+`, `-`, ternary, etc.
5. **[Mandatory]** 4-space indent (no tabs).
6. **[Mandatory]** 120-char column limit. Wrap rules: 4-space indent on continuation, operators move to next line, `.` moves with method, break after comma, never before `(`.
7. **[Mandatory]** One space after comma in parameter lists.
8. **[Mandatory]** UTF-8 encoding, Unix line breaks.
9. **[Recommended]** Don't align variables with extra spaces.
10. **[Recommended]** Single blank line between logical sections. No multiple blank lines.

### OOP Rules

1. **[Mandatory]** Static fields/methods → access via class name, not object.
2. **[Mandatory]** `@Override` on all overridden methods.
3. **[Mandatory]** Varargs only for same-type params. Avoid `Object...`. Varargs must be last param.
4. **[Mandatory]** Don't modify method signatures of existing interfaces. Use `@Deprecated` with description of replacement.
5. **[Mandatory]** Don't use deprecated classes/methods.
6. **[Mandatory]** Use `"constant".equals(object)` to avoid NPE. Or use `Objects.equals()` (JDK7+).
7. **[Mandatory]** Use `equals()` not `==` for wrapper class comparison (Integer cache only covers -128~127).
8. **[Mandatory]** Floating-point comparison: use `BigDecimal` or epsilon range (`1e-6f`), never `==` or `equals()`.
9. **[Mandatory]** POJO members → wrapper classes. RPC params/returns → wrapper classes. Local vars → prefer primitives.
10. **[Mandatory]** No default values in POJO class (DO/DTO/VO) member declarations.
11. **[Mandatory]** Don't change `serialVersionUID` unless fully incompatible update.
12. **[Mandatory]** No business logic in constructors. Use `init()` method.
13. **[Mandatory]** POJO must implement `toString()`. Call `super.toString()` if extending another POJO.
14. **[Recommended]** Check last separator for null when using `String.split()`.
15. **[Recommended]** Group overloaded methods together.
16. **[Recommended]** Method order: public/protected → private → getter/setter.
17. **[Recommended]** Setter arg name = field name. No business logic in getter/setter.
18. **[Recommended]** Use `StringBuilder.append()` in loops, not `str + "hello"`.
19. **[Recommended]** Use `final` for: non-inheritable classes, non-reassignable variables, non-modifiable params, non-overridable methods.
20. **[Recommended]** `Object.clone()` is shallow copy. Be cautious.
21. **[Recommended]** Minimize access levels. Private > protected > default > public.

### Collection

1. **[Mandatory]** Override `hashCode` when overriding `equals`. Required for Set elements and Map keys.
2. **[Mandatory]** Don't add to `keySet()`/`values()`/`entrySet()` results → `UnsupportedOperationException`.
3. **[Mandatory]** Don't modify `Collections.emptyList()`/`singletonList()` etc.
4. **[Mandatory]** Don't cast `ArrayList.subList()` to `ArrayList` → `ClassCastException`.
5. **[Mandatory]** Don't modify original list size after calling `subList()` → `ConcurrentModificationException`.
6. **[Mandatory]** Use `toArray(T[] array)` with correct size. Never `toArray()` without args.
7. **[Mandatory]** `Arrays.asList()` result doesn't support `add`/`remove`/`clear` → `UnsupportedOperationException`.
8. **[Mandatory]** PECS: `<? extends T>` → no `add`; `<? super T>` → no `get`.
9. **[Mandatory]** Don't `add`/`remove` in foreach loop. Use `Iterator.remove()`. Sync Iterator for concurrent ops.
10. **[Mandatory]** `Comparator` must be reflexive, transitive, symmetric → else `IllegalArgumentException` in JDK7+.
11. **[Recommended]** Set initial capacity for collections. e.g. `new ArrayList<>(initialCapacity)`.
12. **[Recommended]** Use `entrySet()` not `keySet()` to traverse maps (avoids double lookup).
13. **[Recommended]** Know which collections allow null keys/values:

| Collection | Key null? | Value null? | Thread-safe? |
|---|---|---|---|
| Hashtable | No | No | Yes |
| ConcurrentHashMap | No | No | Yes (segment lock) |
| TreeMap | No | Yes | No |
| HashMap | Yes | Yes | No |

14. **[For Reference]** Understand sorted vs ordered. ArrayList=ordered+unsorted, HashMap=unordered+unsorted, TreeSet=ordered+sorted.
15. **[For Reference]** Use Set for deduplication instead of `List.contains()`.

### Concurrency

1. **[Mandatory]** Singleton must be thread-safe, including all its methods.
2. **[Mandatory]** Name threads meaningfully. e.g. `super.setName("TimerTaskThread")`
3. **[Mandatory]** Use thread pools, don't create threads directly.
4. **[Mandatory]** Create thread pools via `ThreadPoolExecutor`, NOT `Executors` (FixedThreadPool/SingleThread → Integer.MAX_VALUE queue → OOM; CachedThreadPool/ScheduledThreadPool → Integer.MAX_VALUE threads → OOM).
5. **[Mandatory]** `SimpleDateFormat` is NOT thread-safe. Use `ThreadLocal<DateFormat>` or JDK8 `DateTimeFormatter`.
6. **[Mandatory]** `ThreadLocal.remove()` in finally block, especially with thread pools.
7. **[Mandatory]** Block lock > method lock > class lock for performance.
8. **[Mandatory]** Consistent lock ordering across threads to avoid deadlock.
9. **[Mandatory]** `lock()` OUTSIDE try block. No code between `lock()` and `try`.
10. **[Mandatory]** Use locks for concurrent record updates. Optimistic lock (version) if contention <20%, else pessimistic. Optimistic retry ≥ 3 times.
11. **[Mandatory]** Use `ScheduledExecutorService` not `Timer` (Timer kills all threads on uncaught exception).
12. **[Recommended]** `CountDownLatch`: always `countdown()` in finally; catch sub-thread exceptions.
13. **[Recommended]** Don't share `Random` across threads → use `ThreadLocalRandom` (JDK7+).
14. **[Recommended]** Double-checked locking → declare field as `volatile`.
15. **[For Reference]** `volatile` solves visibility, not atomicity. For `count++` use `AtomicInteger` or `LongAdder` (JDK8).
16. **[For Reference]** `HashMap` resize under concurrency → dead link / high CPU.
17. **[For Reference]** `ThreadLocal` doesn't solve shared-object updates. Use `static ThreadLocal`.

### Flow Control Statements

1. **[Mandatory]** Every `switch` case ends with `break`/`return` (or documented fall-through). Always include `default`.
2. **[Mandatory]** Always use braces with `if`/`else`/`for`/`do`/`while`.
3. **[Recommended]** Minimize `else`. Use early return / guard clauses. Max 3 levels of nesting.
4. **[Recommended]** Extract complex conditions into boolean variables.
5. **[Recommended]** Move object creation, DB connections, try-catch outside loops.
6. **[Recommended]** Validate input size for batch operations.
7. **[For Reference]** Validate params for: low-frequency methods, long-running methods, high-stability methods, open APIs, auth methods.
8. **[For Reference]** Skip validation for: hot loop internals, DAO when co-deployed with Service, private methods with controlled inputs.

### Code Comments

1. **[Mandatory]** Use `/** */` Javadoc for classes, class variables, and methods.
2. **[Mandatory]** Abstract/interface methods must have Javadoc with params, return, exceptions.
3. **[Mandatory]** Every class must have author and date.
4. **[Mandatory]** Single-line `//` above code; multi-line `/* */`. Align properly.
5. **[Mandatory]** All enum fields must have Javadoc.
6. **[Recommended]** Use local language in comments if English is unclear. Keep keywords in English.
7. **[Recommended]** Update comments when code logic changes.
8. **[For Reference]** Add explanation when commenting out code.
9. **[For Reference]** Comments should convey design ideas and business logic.
10. **[For Reference]** Don't over-comment self-explanatory code.
11. **[For Reference]** TODO/FIXME tags must include author and time. Handle promptly.

### Other

1. **[Mandatory]** Precompile regex patterns. Don't define `Pattern.compile()` inside method body.
2. **[Mandatory]** `Math.random()` returns double [0,1). For random int, use `Random.nextInt()`/`nextLong()`.
3. **[Mandatory]** Use `System.currentTimeMillis()` not `new Date().getTime()`. Use `System.nanoTime()` for precision. JDK8 → `Instant`.
4. **[Recommended]** Initialize data structures with known size.
5. **[Recommended]** Remove obsolete code/config promptly.

---

## 2. Exception and Logs

### Exception

1. **[Mandatory]** Pre-check instead of catching RuntimeException (NPE, IndexOutOfBounds). e.g. `if (obj != null)` not try-catch.
2. **[Mandatory]** Don't use exceptions for flow control.
3. **[Mandatory]** Don't try-catch big code blocks. Separate stable vs unstable code. Catch specific exceptions.
4. **[Mandatory]** Don't swallow exceptions. Re-throw if not handling.
5. **[Mandatory]** Rollback on exception.
6. **[Mandatory]** Close resources in finally. Use try-with-resources (Java 7+).
7. **[Mandatory]** Never `return` in finally block.
8. **[Mandatory]** Caught exception type must match or be superclass of thrown type.
9. **[Recommended]** Methods may return null. Document in Javadoc. Caller must null-check.
10. **[Recommended]** NPE-prone situations: primitive return from wrapper method, DB query results, collection elements, RPC returns, session data, method chaining. Use `Optional` (Java 8+).
11. **[Recommended]** APIs → error codes. Internal → throw exceptions. Cross-app RPC → `Result` with `isSuccess`/errorCode/message.
12. **[Recommended]** Don't throw raw `RuntimeException`/`Exception`/`Throwable`. Use custom exceptions: `DAOException`, `ServiceException`.
13. **[For Reference]** DRY — extract common validation logic.

### Logs

1. **[Mandatory]** Use SLF4J facade, not Log4j/Logback directly.
2. **[Mandatory]** Keep logs ≥ 15 days.
3. **[Mandatory]** Extended log naming: `appName_logType_logName.log`. Types: stats, desc, monitor, visit.
4. **[Mandatory]** TRACE/DEBUG/INFO → conditional output or placeholders `{}`. Never concatenate strings unconditionally.
5. **[Mandatory]** Log4j: `additivity=false` to avoid duplicate output.
6. **[Mandatory]** Log exceptions with context + full stack: `logger.error(context + "_" + e.getMessage(), e)`
7. **[Recommended]** Be selective with Info. No Debug in production. Watch Warn log volume.
8. **[Recommended]** Warn = invalid parameters / tracking. Error = system logic errors / critical issues only.

---

## 3. MySQL Rules

### Table Schema Rules

1. **[Mandatory]** Boolean columns → `is_xxx`, type `unsigned tinyint` (1=true, 0=false). Non-negative columns → `unsigned`.
2. **[Mandatory]** Table/column names: lowercase + digits + underscores. No digit-only segments between underscores.
3. **[Mandatory]** No plural table names.
4. **[Mandatory]** No reserved words for table/column names. e.g. avoid `desc`, `range`, `match`, `delayed`.
5. **[Mandatory]** Primary key → `id`, type `bigint unsigned`, auto-increment.
6. **[Mandatory]** Every table must have `gmt_create` (datetime) and `gmt_modified` (datetime).
7. **[Recommended]** Table name: `业务名_功能描述`. e.g. `alipay_task`, `force_project`.
8. **[Recommended]** DB name = app name.
9. **[Recommended]** Update `gmt_modified` on every row change.
10. **[Recommended]** Use `decimal` for money, not `float`/`double`. Or store as cents in `bigint`.
11. **[Recommended]** Use `varchar` (variable length) if length varies significantly. Max varchar = 5000. If >5000, use `text` with separate table.
12. **[For Reference]** Appropriate normalization when creating tables.

### Index Rules

1. **[Mandatory]** Unique business fields must have unique index.
2. **[Mandatory]** No join on 3+ tables. Joined columns must have same type and index.
3. **[Mandatory]** Create index on varchar: specify length. Usually 20 chars gets >90% selectivity.
4. **[Mandatory]** Avoid left-fuzzy or full-fuzzy search in pages. Use search engine if needed.
5. **[Recommended]** Order of composite index matters. Put most discriminating column first.
6. **[Recommended]** Create composite index instead of single-column indexes when possible.
7. **[For Reference]** Create index wisely. Don't blindly index or refuse to index.

### SQL Rules

1. **[Mandatory]** No `count(column)` or `count(constant)` as substitute for `count(*)`. `count(*)` is SQL92 standard, counts all rows regardless of NULL.
2. **[Mandatory]** `count(distinct col)` ignores NULL. `count(col)` also ignores NULL. But `count(NULL)` returns 0.
3. **[Mandatory]** Check if `count` result is 0 before using. Don't call `sum()` without null check.
4. **[Mandatory]** Use `ISNULL()` to check NULL, not `=`.
5. **[Mandatory]** No foreign keys in code. Referential integrity at application level.
6. **[Mandatory]** No `*` in stored procedures.
7. **[Recommended]** Use `IF EXISTS` when `IN` has potential large set.
8. **[For Reference]** If data to delete/update exceeds 10000, do it in batches.

### ORM Rules

1. **[Mandatory]** No `*` in queries. Enumerate required columns.
2. **[Mandatory]** POJO boolean → `isXxx`, DB column → `is_xxx`. ORM mapping must handle this mismatch.
3. **[Mandatory]** No `resultClass` as return for query in XML config. Use `resultMap` for column-field mapping.
4. **[Mandatory]** Beware of `#{}` vs `${}`. Use `#{}` to prevent SQL injection. `${}` is string replacement.
5. **[Mandatory]** Use physical pagination in SQL (e.g. `LIMIT #{offset}, #{pageSize}`). Never rely on in-memory pagination. For MyBatis 3.x, use PageHelper or hand-write `LIMIT`/`OFFSET` in the mapper XML.
6. **[Mandatory]** No `HashMap`/`Hashtable` as query result type.
7. **[Mandatory]** `gmt_modified` must be updated with data changes.
8. **[Recommended]** Don't write large complex SQL. Break into smaller queries. Don't compute in DB what app can compute.
9. **[Recommended]** Don't use `@Transactional` too broadly. Scope transactions to necessary operations only. Consider network calls, search engine, MQ interactions.
10. **[Mandatory]** Avoid N+1 queries. When loading a list of entities that each require a sub-query, use a single batch query or JOIN instead of querying inside a loop. e.g. fetch all related items in one `WHERE id IN (...)` query, not one query per parent row.

---

## 4. Project Specification

### Application Layers

- **Open API**: Exposed externally (RPC, HTTP)
- **Terminal Display**: Templates, pages
- **Web**: Request routing, parameter validation
- **Service**: Business logic
- **Manager**: Common business processing (cache, MQ, third-party)
- **DAO**: Data access, one DAO per table
- **External**: RPC wrappers for external services

Domain models per layer:
- DO (Data Object) → DAO ↔ DB
- DTO (Data Transfer Object) → Service/Manager
- BO (Business Object) → Service
- Query → complex queries beyond single DO
- VO (View Object) → Web/Terminal

### Library Specification

1. **[Mandatory]** GAV format: GroupId=com.{company/BU}.{project}; ArtifactId=product-module; Version=major.minor.patch
2. **[Mandatory]** Initial version = 1.0.0
3. **[Mandatory]** No SNAPSHOT deps in production (except security packages)
4. **[Recommended]** Manage versions centrally with `<dependencyManagement>`
5. **[Recommended]** Max 2 levels of sub-project nesting

### Server Specification

> Note: TCP tuning and OS-level file descriptor limits are infrastructure/ops concerns. Configure these at the deployment level (e.g., systemd unit files, container resource limits), not in application code.

---

## 5. Security Specification

1. **[Mandatory]** Pages/features with user data access must have authorization checks.
2. **[Mandatory]** Direct display of user sensitive data (phone, address) is prohibited. Mask data.
3. **[Mandatory]** User input SQL parameters must use parameter binding or whitelist-validated metadata, never string concatenation.
4. **[Mandatory]** User input must not contain executable code (no eval, no dynamic SQL).
5. **[Mandatory]** HTML/JS output must be encoded/escaped to prevent XSS.
6. **[Mandatory]** Forms must implement CSRF protection (CSRF token).
7. **[Mandatory]** Use platform-level text filtering against prohibited words. (Anti-content-policy, advertising law compliance)
8. **[Mandatory]** Server-side SMS/email sending must throttle (frequency, count limits) to prevent abuse.
