# Java + Spring Boot Interview Handbook (4+ YOE)
## Part 4: SQL, Design Patterns, System Design (LLD), Testing, Maven/Gradle, Git, Docker, Kafka/RabbitMQ

---

## 16. SQL & Database Fundamentals

### Q16.1 — Explain SQL Joins with examples, and the difference between `WHERE` and `HAVING`.

**Answer:**

```sql
-- INNER JOIN: only matching rows in both tables
SELECT o.id, c.name FROM orders o INNER JOIN customers c ON o.customer_id = c.id;

-- LEFT JOIN: all rows from left, matched rows from right (NULL if no match)
SELECT c.name, o.id FROM customers c LEFT JOIN orders o ON c.id = o.customer_id;

-- RIGHT JOIN: all rows from right, matched from left
-- FULL OUTER JOIN: all rows from both, matched where possible (not supported directly in MySQL — emulate via UNION of LEFT + RIGHT)
-- CROSS JOIN: Cartesian product (every row x every row) — rarely intentional, common accidental-bug source (missing ON clause)
-- SELF JOIN: table joined to itself (e.g., employee-manager hierarchy)
SELECT e.name AS employee, m.name AS manager FROM employees e LEFT JOIN employees m ON e.manager_id = m.id;
```

**`WHERE` vs `HAVING`:** `WHERE` filters rows **before** grouping/aggregation; `HAVING` filters groups **after** aggregation (can reference aggregate functions like `COUNT`, `SUM`).
```sql
SELECT department, COUNT(*) AS cnt FROM employees
WHERE status = 'ACTIVE'         -- filters individual rows first
GROUP BY department
HAVING COUNT(*) > 10;           -- filters resulting groups
```

**Follow-up:** *"Why can't you use an aggregate function in `WHERE`?"* → Because `WHERE` executes before `GROUP BY`/aggregation in logical query processing order — aggregate values don't exist yet at that stage.

---

### Q16.2 — SQL logical query processing order (very commonly probed for query-writing correctness).

**Answer:**
```
FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT/OFFSET
```
This explains several "gotchas": you can't reference a `SELECT` column alias in `WHERE` (WHERE runs before SELECT is evaluated) but you generally CAN in `ORDER BY` (runs after SELECT) in most databases; window functions run after `WHERE`/`GROUP BY`/`HAVING` but before the final `ORDER BY`/`SELECT DISTINCT` — so you cannot filter directly on a window function result in `WHERE` (must wrap in a subquery/CTE).

---

### Q16.3 — Explain Database Indexing — B-Tree internals, when indexes help/hurt, and composite index column order.

**Answer:** An index is a separate data structure (commonly a **B-Tree**, or **B+Tree** in most RDBMS implementations) that maintains a sorted structure over column(s), enabling `O(log n)` lookups instead of full table scans (`O(n)`).

**B+Tree structure:** Internal nodes hold only keys (routing), leaf nodes hold the actual indexed values + pointers to table rows (or the row data itself, for a clustered index), and leaf nodes are linked for efficient range scans.

**Clustered vs Non-Clustered index:**
- **Clustered:** table data itself is physically stored in index order (typically on the Primary Key) — only one per table, since data can only be sorted one way physically.
- **Non-clustered:** separate structure, leaf nodes store a pointer (row locator/PK value) back to the actual row — a table can have many.

**Composite index column order matters a lot (frequently asked):** An index on `(status, created_at)` can efficiently serve queries filtering on `status` alone, or `status AND created_at`, but **cannot** efficiently serve a query filtering on `created_at` alone — the index is only useful as a left-to-right prefix match (like a phone book sorted by last name then first name — useless for looking up by first name alone).

```sql
CREATE INDEX idx_status_created ON orders(status, created_at);
-- Efficient: WHERE status = 'PENDING'
-- Efficient: WHERE status = 'PENDING' AND created_at > '2024-01-01'
-- NOT efficient (index not used for filtering): WHERE created_at > '2024-01-01'  (status not specified)
```

**When indexes HURT:**
- Slow down `INSERT`/`UPDATE`/`DELETE` (every index must also be updated on write) — over-indexing is a real production anti-pattern.
- Low-cardinality columns (e.g., a boolean `is_active`) gain little benefit — the optimizer may ignore the index anyway if a large fraction of rows match.
- Extra storage overhead.

**Best Practices:**
- Index columns used in `WHERE`, `JOIN ON`, `ORDER BY`, foreign keys.
- Put the most selective (highest cardinality) / most frequently filtered column first in composite indexes (with care — also consider actual query patterns, not cardinality alone).
- Use `EXPLAIN`/`EXPLAIN ANALYZE` to verify the query planner actually uses your index (a "covering index" that includes all needed columns can let the engine avoid touching the table at all — "index-only scan").
- Periodically review unused indexes (most DBs expose index-usage stats) and drop them.

**Common Mistakes:**
- Applying a function to an indexed column in `WHERE` (`WHERE UPPER(name) = 'JOHN'`) — prevents index usage unless a functional index exists for that exact expression.
- Leading wildcard `LIKE '%something'` — can't use a standard B-Tree index (no fixed prefix to seek on); trailing wildcard `LIKE 'something%'` CAN use the index.
- Over-indexing a write-heavy table, tanking insert/update throughput.

**Follow-up:** *"What's the difference between a covering index and a regular index?"* → A covering index includes **all columns needed by a query** (in the SELECT, WHERE, ORDER BY) so the database can answer the query entirely from the index without a "lookup" back to the full table row — significantly faster for read-heavy hot queries.

---

### Q16.4 — ACID properties and Transaction Isolation Levels — explain each with concrete anomaly examples.

**Answer:**

**ACID:**
- **Atomicity** — all operations in a transaction succeed or none do (all-or-nothing).
- **Consistency** — a transaction moves the DB from one valid state to another, respecting constraints/invariants.
- **Isolation** — concurrent transactions don't interfere with each other's intermediate states.
- **Durability** — once committed, changes survive crashes (via write-ahead logging/WAL).

**Isolation levels (weakest to strongest) — with the specific anomaly each one fixes:**

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| `READ UNCOMMITTED` | Possible | Possible | Possible |
| `READ COMMITTED` (default in PostgreSQL, Oracle, SQL Server) | Prevented | Possible | Possible |
| `REPEATABLE READ` (default in MySQL/InnoDB) | Prevented | Prevented | Possible (though InnoDB's implementation via MVCC + gap locks largely prevents phantoms too, beyond strict spec) |
| `SERIALIZABLE` | Prevented | Prevented | Prevented |

- **Dirty Read:** Transaction A reads uncommitted changes from Transaction B; if B rolls back, A read data that never "really" existed.
- **Non-Repeatable Read:** A reads a row twice within the same transaction, gets different values because B committed an update to that row in between.
- **Phantom Read:** A runs the same range query twice, gets a different **set of rows** because B inserted/deleted rows matching that range in between.

**Tradeoff:** Higher isolation = stronger consistency guarantees but lower concurrency (more locking/blocking or more transaction abort-and-retry under MVCC-based serializable implementations).

**Best Practice:** Use `READ COMMITTED` (most common default) for typical CRUD apps; reach for `SERIALIZABLE` only for genuinely critical invariants (e.g., financial ledger balance checks) since it can significantly hurt throughput and increase retry/rollback rates.

**Follow-up:** *"How does MVCC (Multi-Version Concurrency Control) relate to isolation levels?"* → Databases like PostgreSQL and MySQL/InnoDB implement most isolation levels using MVCC — instead of blocking readers with locks, each transaction sees a consistent **snapshot** of the data as of a certain point in time, and writers create new row versions rather than overwriting in place — this is why readers generally don't block writers (and vice versa) in these systems, unlike naive lock-based isolation.

---

### Q16.5 — Normalization vs Denormalization — when would you denormalize in a real production schema?

**Answer:** **Normalization** (1NF, 2NF, 3NF, BCNF) organizes data to eliminate redundancy and update anomalies — each fact stored once. **Denormalization** intentionally introduces redundancy for read performance, trading write complexity/storage for faster reads (fewer joins).

**Real-world denormalization example:** An `orders` table storing a `customer_name` snapshot column (in addition to `customer_id` FK) — avoids a join for order-listing screens, and preserves historical accuracy even if the customer later changes their name (since the order should reflect the name at time of purchase — a business/audit requirement in addition to a performance one). Similarly, materialized "summary"/"rollup" tables (e.g., `daily_sales_summary`) precomputed via batch jobs or triggers to avoid expensive aggregate queries on every dashboard page load.

**Best Practice:** Normalize first (for correctness/maintainability), then denormalize selectively and deliberately based on measured read-performance needs, keeping a clear strategy for how derived/denormalized data stays in sync (triggers, application-level dual-write, CDC/event-driven sync, or scheduled batch jobs) — never denormalize "by accident."

---

## 17. Design Patterns

### Q17.1 — Which design patterns have you actually used in a Spring Boot codebase, and why? (Extremely common open-ended question)

**Answer (structure your answer around real Spring usage — interviewers want proof you understand these beyond textbook definitions):**

| Pattern | Category | Where it shows up in Spring/enterprise code |
|---|---|---|
| **Singleton** | Creational | Default Spring bean scope — one shared instance per container |
| **Factory Method** | Creational | `BeanFactory`, `ApplicationContext.getBean()`; `@Bean` factory methods in `@Configuration` classes |
| **Builder** | Creational | `UriComponentsBuilder`, Lombok `@Builder`, `ResponseEntity.status(200).body(x).build()`-style fluent APIs |
| **Proxy** | Structural | Spring AOP (`@Transactional`, `@Async`, `@Cacheable`) implemented via JDK/CGLIB proxies |
| **Decorator** | Structural | `HttpServletRequestWrapper`, `BufferedReader` wrapping `FileReader` |
| **Adapter** | Structural | `HandlerAdapter` in Spring MVC adapts different controller styles to a uniform invocation interface |
| **Template Method** | Behavioral | `JdbcTemplate`, `RestTemplate`, `AbstractApplicationContext.refresh()` — defines algorithm skeleton, subclasses fill in specifics |
| **Strategy** | Behavioral | `PasswordEncoder` interface with swappable implementations (`BCryptPasswordEncoder`); `Comparator` |
| **Observer** | Behavioral | Spring's `ApplicationEvent`/`ApplicationListener`/`@EventListener` mechanism |
| **Chain of Responsibility** | Behavioral | Servlet Filter chain, Spring Security filter chain, `HandlerInterceptor` chain |
| **Dependency Injection** | (Enterprise pattern) | Spring's core IoC container |

**Code examples for the two most frequently deep-dived (Strategy & Template Method):**

```java
// STRATEGY — swap algorithm at runtime via interface
public interface DiscountStrategy { BigDecimal apply(BigDecimal price); }
public class FestiveDiscount implements DiscountStrategy {
    public BigDecimal apply(BigDecimal price) { return price.multiply(BigDecimal.valueOf(0.8)); }
}
@Service
public class PricingService {
    private final Map<String, DiscountStrategy> strategies; // Spring auto-wires all beans of this type into a Map keyed by bean name
    public PricingService(Map<String, DiscountStrategy> strategies) { this.strategies = strategies; }
    public BigDecimal price(String strategyName, BigDecimal base) {
        return strategies.get(strategyName).apply(base);
    }
}

// TEMPLATE METHOD
public abstract class ReportGenerator {
    public final void generate() { // skeleton fixed, steps vary
        fetchData();
        process();
        export();
    }
    protected abstract void fetchData();
    protected abstract void process();
    protected void export() { System.out.println("Default export as PDF"); } // hook, overridable
}
```

**Follow-up:** *"How does Spring inject a `Map<String, SomeInterface>` of all implementations?"* → Spring's autowiring resolves **all beans of a given type** into a `List`/`Map` (keyed by bean name) automatically when the constructor/field parameter type is `List<T>`/`Map<String,T>` — extremely useful for implementing Strategy pattern cleanly without a manual `switch`/`if-else` dispatcher.

---

### Q17.2 — Explain SOLID principles with a Java example for each.

**Answer:**

- **S — Single Responsibility Principle:** A class should have one reason to change. E.g., separate `OrderValidator`, `OrderRepository`, `OrderNotifier` rather than one `OrderService` doing validation + persistence + email.
- **O — Open/Closed Principle:** Open for extension, closed for modification. E.g., adding a new `DiscountStrategy` implementation shouldn't require modifying `PricingService`'s existing code.
- **L — Liskov Substitution Principle:** Subtypes must be substitutable for their base types without breaking correctness. Classic violation: `Square extends Rectangle` overriding `setWidth`/`setHeight` to keep them equal — breaks client code that assumes independent width/height mutation.
- **I — Interface Segregation Principle:** Prefer many small, specific interfaces over one fat interface — clients shouldn't be forced to depend on methods they don't use. E.g., splitting a bloated `Repository` interface into `Readable`/`Writable` if some consumers only need read access.
- **D — Dependency Inversion Principle:** High-level modules shouldn't depend on low-level modules; both depend on abstractions. This is exactly what Spring DI enables — `OrderService` depends on the `PaymentGateway` interface, not a concrete `StripePaymentGateway`.

**Follow-up 🎯:** *"Give a real LSP violation you've seen or could see in a Spring codebase."* → An `interface Repository { void save(T entity); }` where one implementation (`ReadOnlyRepository`) throws `UnsupportedOperationException` from `save()` — violates LSP because callers relying on the `Repository` contract can't safely substitute this implementation without breaking.

---

## 18. System Design & Low-Level Design (LLD) Basics

### Q18.1 — Design a Rate Limiter (classic LLD question).

**Answer:** Walk through algorithms and tradeoffs — interviewers want to see you compare options, not just code one blindly.

| Algorithm | How it works | Pros | Cons |
|---|---|---|---|
| **Fixed Window Counter** | Count requests in fixed time buckets (e.g., per minute), reset at boundary | Simple | Burst at window boundary can allow 2x limit (e.g., burst at end of one window + start of next) |
| **Sliding Window Log** | Store timestamp of every request, count requests within the trailing window | Accurate | Memory-heavy for high traffic (stores every timestamp) |
| **Sliding Window Counter** | Weighted average of current + previous fixed window | Good accuracy, low memory | Slightly approximate |
| **Token Bucket** | Bucket refills tokens at fixed rate; each request consumes a token; request rejected if empty | Allows controlled bursts, industry standard (used by AWS, Stripe) | Slightly more complex |
| **Leaky Bucket** | Requests processed at a constant fixed rate (queue-based) | Smooths bursts completely | Can add latency, less burst-tolerant than token bucket |

```java
public class TokenBucketRateLimiter {
    private final long capacity;
    private final long refillRatePerSecond;
    private double tokens;
    private long lastRefillTimestamp;

    public synchronized boolean allowRequest() {
        refill();
        if (tokens >= 1) { tokens -= 1; return true; }
        return false;
    }
    private void refill() {
        long now = System.currentTimeMillis();
        double tokensToAdd = (now - lastRefillTimestamp) / 1000.0 * refillRatePerSecond;
        tokens = Math.min(capacity, tokens + tokensToAdd);
        lastRefillTimestamp = now;
    }
}
```

**Distributed considerations:** Single-JVM in-memory rate limiting doesn't work across multiple service instances — use **Redis** (`INCR` + `EXPIRE`, or a Lua script for atomicity, or `Redisson`'s built-in `RRateLimiter`) as a shared, atomic counter store. Real production systems (Spring Cloud Gateway + Redis) implement this exact pattern via `RedisRateLimiter`.

**Follow-up:** *"How do you rate-limit per-user vs globally?"* → Key the counter/bucket by a composite key (`userId + endpoint`, or `apiKey`), not a single global counter — Redis key like `ratelimit:{userId}:{endpoint}` with a TTL matching the window.

---

### Q18.2 — Design a URL Shortener (very common LLD/system design hybrid).

**Answer (key points to hit in an interview):**

1. **Core requirement:** `POST /shorten {longUrl} → shortCode`; `GET /{shortCode} → 302 redirect to longUrl`.
2. **Short code generation strategies:**
   - **Base62 encoding of an auto-increment ID** (using `[a-zA-Z0-9]`) — simple, guaranteed unique, but predictable/sequential (info leak: users can guess adjacent codes) and requires a centralized ID generator (bottleneck/single point of failure unless using a distributed ID generator like Snowflake or pre-allocated ID ranges per app instance).
   - **Hash-based (MD5/SHA-256 of the URL, truncated)** — need collision handling (append counter/salt on collision), naturally distributed (no central counter needed).
   - **Random string + DB uniqueness check** — simple, but need retry-on-collision logic; collision probability grows with scale (birthday paradox) — mitigate with sufficient code length (7 chars base62 ≈ 3.5 trillion combinations).
3. **Data model:** `short_code (PK), long_url, created_at, expiry, user_id, click_count`.
4. **Read-heavy system** (far more redirects than creations) → aggressive caching (Redis/CDN edge caching for hot URLs), read replicas.
5. **Scaling:** shard the DB by short_code hash if a single DB can't handle volume; use a distributed ID generator (Twitter Snowflake pattern: timestamp + machine ID + sequence) to avoid a single auto-increment bottleneck across multiple app servers.
6. **Analytics (click_count):** don't synchronously increment on every redirect (write contention on a hot row) — buffer clicks and batch-update asynchronously (via a message queue like Kafka), or use an approximate counter.

**Follow-up:** *"How do you handle custom short codes and collisions?"* → Check DB uniqueness constraint on `short_code` before insert; on collision (race condition under concurrent requests), catch the unique-constraint-violation exception and retry with a new generated code, or use a DB-level `INSERT ... ON CONFLICT` upsert pattern.

---

### Q18.3 — Design a Distributed Cache / Explain Caching Strategies (Cache-Aside, Write-Through, Write-Behind).

**Answer:**

| Strategy | How it works | Tradeoff |
|---|---|---|
| **Cache-Aside (Lazy Loading)** | App checks cache first; on miss, reads DB, populates cache | Most common pattern (`@Cacheable` in Spring). Cache can go stale; first request after eviction is slow ("cache miss penalty") |
| **Write-Through** | Every write goes to cache AND DB synchronously | Cache always consistent with DB, but write latency = DB write + cache write |
| **Write-Behind (Write-Back)** | Write to cache immediately, asynchronously flush to DB later | Fast writes, but risk of data loss if cache fails before flush; more complex |
| **Read-Through** | Cache itself is responsible for loading from DB on miss (transparent to app) | Cleaner app code, needs cache-provider support |

```java
@Service
public class ProductService {
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) { return productRepository.findById(id).orElseThrow(); }

    @CacheEvict(value = "products", key = "#product.id")
    @CachePut(value = "products", key = "#result.id")
    public Product updateProduct(Product product) { return productRepository.save(product); }
}
```

**Cache eviction policies:** LRU (Least Recently Used — most common default), LFU (Least Frequently Used), FIFO, TTL-based expiry.

**Cache invalidation challenges (the classic "two hard things in CS" joke, but interviewers want a real answer):**
- **Stale data:** mitigate with reasonable TTLs + explicit eviction on writes (`@CacheEvict`).
- **Cache stampede/thundering herd:** many concurrent requests miss cache simultaneously (e.g., right after eviction of a hot key) and all hit the DB at once — mitigate with request coalescing/locking (only one thread fetches, others wait), or staggered TTL jitter to avoid mass simultaneous expiry.
- **Cache penetration:** repeated requests for a key that doesn't exist in DB either, bypassing cache every time — mitigate by caching the "not found" result too (with a short TTL), or a Bloom filter to quickly reject known-absent keys.

**Best Practices:**
- Set sensible TTLs — never cache indefinitely without an eviction strategy.
- Use a distributed cache (Redis/Memcached) for multi-instance deployments — in-process caches (`Caffeine`, Spring's default `ConcurrentMapCacheManager`) aren't shared across instances, leading to inconsistent cache state per node.
- Monitor cache hit ratio — a consistently low hit ratio suggests wrong TTL/key strategy or that caching isn't actually helping for that access pattern.

**Follow-up:** *"What happens if the cache and DB get out of sync?"* → Depends on chosen consistency strategy; for cache-aside, always evict/update cache **after** the DB write commits (not before — avoids caching a value that then fails to persist), and prefer `@CacheEvict` over `@CachePut` when in doubt (forces a fresh read-through on next access rather than risking a stale/incorrect cached value).

---

## 19. Testing (JUnit, Mockito)

### Q19.1 — JUnit 5 essentials, test lifecycle annotations, and unit vs integration testing in Spring Boot.

**Answer:**

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock private OrderRepository orderRepository;
    @Mock private PaymentClient paymentClient;
    @InjectMocks private OrderService orderService; // Mockito injects mocks into this instance

    @BeforeEach void setup() { /* runs before each test */ }

    @Test
    void shouldThrowWhenOrderNotFound() {
        when(orderRepository.findById(1L)).thenReturn(Optional.empty());
        assertThrows(ResourceNotFoundException.class, () -> orderService.getOrder(1L));
    }

    @Test
    void shouldCalculateTotalCorrectly() {
        Order order = new Order(List.of(new OrderItem("SKU1", 2, BigDecimal.TEN)));
        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
        BigDecimal total = orderService.calculateTotal(1L);
        assertEquals(BigDecimal.valueOf(20), total);
    }

    @ParameterizedTest
    @ValueSource(strings = {"", " ", "invalid-email"})
    void shouldRejectInvalidEmails(String email) {
        assertFalse(EmailValidator.isValid(email));
    }
}
```

**Key JUnit 5 annotations:** `@Test`, `@BeforeEach`/`@AfterEach` (per-test), `@BeforeAll`/`@AfterAll` (once per class, must be `static` unless `@TestInstance(PER_CLASS)`), `@ParameterizedTest` + `@ValueSource`/`@CsvSource`/`@MethodSource`, `@Disabled`, `@Nested` (grouping related tests), `@Tag` (categorize/filter test runs, e.g., `@Tag("slow")`).

**Mockito essentials:**
- `@Mock` — creates a mock; all methods return default values (null/0/false) unless stubbed.
- `@Spy` — wraps a **real** object; unstubbed methods call the real implementation.
- `@InjectMocks` — Mockito attempts constructor/setter/field injection of mocks into the object under test.
- `when(...).thenReturn(...)` / `when(...).thenThrow(...)` — stubbing.
- `verify(mock).method(args)` — behavior verification (was this method called, how many times, with what args).
- `ArgumentCaptor` — capture the actual argument passed to a mocked method for assertions.

```java
@Test
void shouldSendCorrectPaymentAmount() {
    orderService.processOrder(order);
    ArgumentCaptor<BigDecimal> captor = ArgumentCaptor.forClass(BigDecimal.class);
    verify(paymentClient).charge(captor.capture());
    assertEquals(BigDecimal.valueOf(100), captor.getValue());
}
```

**Unit vs Integration testing in Spring Boot:**

| | Unit Test | Integration Test |
|---|---|---|
| Scope | Single class, dependencies mocked | Multiple layers, real Spring context (partial or full) |
| Speed | Fast (milliseconds) | Slower (seconds) |
| Spring context | Not loaded (`@ExtendWith(MockitoExtension.class)`) | Loaded (`@SpringBootTest`) |
| Example | `OrderServiceTest` with mocked repository | Full flow test hitting a real (test) DB via Testcontainers |

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderControllerIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @DynamicPropertySource
    static void configureProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
    }
    @Autowired TestRestTemplate restTemplate;

    @Test
    void shouldCreateOrderSuccessfully() {
        ResponseEntity<OrderResponse> response = restTemplate.postForEntity("/api/orders", request, OrderResponse.class);
        assertEquals(HttpStatus.CREATED, response.getStatusCode());
    }
}
```

**Other important Spring test slices:**
- `@WebMvcTest` — loads only the web layer (controllers, `@ControllerAdvice`, Jackson config), mocks service layer via `@MockBean` — fast, focused controller tests.
- `@DataJpaTest` — loads only JPA-related config, uses an in-memory/embedded DB by default, rolls back transactions after each test — good for repository-layer tests.
- `@MockBean`/`@SpyBean` (or `@MockitoBean` in Spring Boot 3.4+) — replace a bean in the Spring context with a mock.

**Best Practices:**
- Follow the **Test Pyramid**: many fast unit tests, fewer integration tests, very few end-to-end tests.
- Use **Testcontainers** (real Dockerized DB/Kafka/Redis) over H2/embedded fakes for integration tests — avoids subtle SQL-dialect/behavior mismatches between H2 and production Postgres/MySQL (a very common real-world gotcha — H2 "passes" tests that then fail against real Postgres due to dialect differences).
- Follow **AAA pattern** (Arrange-Act-Assert) for readable tests.
- Test behavior/outcomes, not implementation details — avoid over-mocking/over-verifying internal calls that make tests brittle to refactoring.
- Aim for meaningful coverage of business logic and edge cases, not just a coverage percentage target (100% coverage with weak assertions is a false sense of security).

**Common Mistakes:**
- Using `@SpringBootTest` (full context) for every test — slow test suites, discourages running tests frequently.
- Testing getters/setters or framework code instead of actual business logic.
- Not resetting/isolating state between tests — tests that pass/fail depending on execution order (shared mutable static state, non-rolled-back DB state).
- Overusing `@MockBean` in `@SpringBootTest` — reloads the Spring context per unique `@MockBean` combination, dramatically slowing the whole suite (context caching is invalidated).

**Follow-up:** *"What is the Spring Test Context Caching and why does it matter for test speed?"* → Spring caches the `ApplicationContext` across test classes with identical configuration, avoiding the expensive context-rebuild cost per test class. Introducing different `@MockBean`s or `@ActiveProfiles` per test class creates distinct context "keys," causing more context reloads — a common cause of slow CI pipelines.

---

## 20. Maven / Gradle

### Q20.1 — Maven lifecycle, dependency scopes, and Maven vs Gradle tradeoffs.

**Answer:**

**Maven build lifecycle (phases run in order):**
```
validate → compile → test → package → verify → install → deploy
```
Running `mvn install` runs everything up through `install` (compiles, tests, packages, installs to local `.m2` repo). `mvn package` stops at packaging (produces jar/war, doesn't install/deploy).

**Dependency scopes:**

| Scope | Available at compile | Available at runtime | Included in final artifact | Example |
|---|---|---|---|---|
| `compile` (default) | Yes | Yes | Yes | Spring core libs |
| `provided` | Yes | No (assumed provided by container/JDK) | No | Servlet API (provided by Tomcat) |
| `runtime` | No | Yes | Yes | JDBC driver |
| `test` | Test only | Test only | No | JUnit, Mockito |
| `system` | Yes (path-based, deprecated pattern) | Yes | No | Rare/legacy |

**Transitive dependency conflict resolution:** Maven uses **"nearest wins"** (shortest dependency path from your project) when multiple versions of the same artifact appear transitively; ties broken by declaration order. Use `mvn dependency:tree` to diagnose version conflicts, and `<dependencyManagement>` / explicit `<exclusions>` to force specific versions.

**Maven vs Gradle:**

| | Maven | Gradle |
|---|---|---|
| Config | Declarative XML (`pom.xml`) | Groovy/Kotlin DSL (`build.gradle`/`.kts`) — more flexible/programmable |
| Build speed | Slower (no incremental build by default pre-recent versions) | Faster — incremental builds, build cache, parallel execution by default |
| Learning curve | Simpler, more standardized/opinionated | More powerful but more complex |
| Ecosystem | Extremely mature, huge plugin ecosystem | Modern default for Android, growing in enterprise |

**Follow-up:** *"How do you handle a dependency version conflict in Maven?"* → Run `mvn dependency:tree -Dverbose` to see conflicting versions and their source; resolve via explicit `<dependencyManagement>` entry to pin the version, or `<exclusions>` on the offending transitive dependency.

---

## 21. Git

### Q21.1 — Git branching strategies, rebase vs merge, and resolving conflicts.

**Answer:**

**`git merge` vs `git rebase`:**
- **Merge:** combines two branches' histories with a new merge commit — preserves full, accurate history (including exactly when branches diverged/joined), but can create a "noisy" non-linear history with many merge commits.
- **Rebase:** replays your branch's commits on top of the target branch's latest tip — produces a clean, linear history, but **rewrites commit hashes** (dangerous on shared/pushed branches — never rebase commits others have already pulled, causes painful history divergence for collaborators).

```bash
git checkout feature-branch
git rebase main         # replay feature commits on top of latest main
# resolve conflicts if any, then:
git rebase --continue
```

**Common branching strategies:**
- **Git Flow:** `main`, `develop`, `feature/*`, `release/*`, `hotfix/*` — structured, good for scheduled releases, but heavier process.
- **GitHub Flow / Trunk-Based Development:** short-lived feature branches off `main`, frequent small PRs, continuous deployment — favored by most modern product teams for faster iteration.

**Interactive rebase (`git rebase -i`)** — commonly used to squash/reorder/edit commits before merging a feature branch, keeping history clean:
```bash
git rebase -i HEAD~5   # squash/reorder last 5 commits
```

**`git cherry-pick`** — apply a specific commit from one branch onto another (e.g., a hotfix needed on both `main` and a `release` branch).

**Best Practices:**
- Small, frequent, atomic commits with clear messages.
- Never force-push (`git push --force`) to a shared branch without team coordination — use `--force-with-lease` at minimum (fails if remote has commits you don't have locally, preventing accidental overwrite of others' work).
- Squash noisy WIP commits before merging to keep `main`'s history meaningful.

**Follow-up:** *"How do you undo a commit that's already pushed?"* → `git revert <commit>` (creates a new commit undoing the changes — safe for shared branches, preserves history) vs `git reset --hard` (rewrites history — only safe for local/unpushed commits, or requires force-push + team coordination if already pushed).

---

## 22. Docker Basics

### Q22.1 — Explain Docker fundamentals for a Spring Boot app: image vs container, Dockerfile best practices, multi-stage builds.

**Answer:**

**Image vs Container:** An **image** is a read-only, layered template (built from a `Dockerfile`); a **container** is a running (or stopped) instance of an image, with its own writable layer on top.

**Multi-stage Dockerfile for a Spring Boot app (best practice, reduces final image size significantly):**
```dockerfile
# Stage 1: build
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline           # cache dependencies in a separate layer
COPY src ./src
RUN mvn package -DskipTests

# Stage 2: minimal runtime image
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Why multi-stage?** The build stage includes Maven, the full JDK, and source — all unnecessary bloat in the final runtime image. The final stage only copies the built jar into a minimal JRE base image — smaller image = faster pulls/deploys, smaller attack surface.

**Layer caching best practice:** Copy `pom.xml`/`build.gradle` and run dependency resolution **before** copying source code — Docker caches layers, so if only source changes (not dependencies), the dependency-download layer is reused, speeding up rebuilds significantly.

**Key concepts:**
- **`docker-compose`** — define multi-container local dev environments (app + Postgres + Redis + Kafka) declaratively in one YAML file.
- **Volumes** — persist data beyond container lifecycle (e.g., DB data directory) — without a volume, container filesystem changes are lost when the container is removed.
- **Networks** — containers on the same Docker network can reach each other by service/container name (DNS-based discovery) — no need for hardcoded IPs in local dev.
- **`.dockerignore`** — exclude `target/`, `.git/`, etc. from the build context, speeding up builds.

**Best Practices:**
- Run containers as a **non-root user** for security.
- Use specific version tags, not `latest`, for reproducible builds.
- Externalize configuration via environment variables (12-factor app principle), not baked into the image.
- Set JVM container-awareness flags/limits appropriately (`-XX:MaxRAMPercentage`) so the JVM respects container memory limits instead of assuming the full host's memory (modern JDKs are container-aware by default since Java 10+, but explicit tuning is still common in production).
- Implement graceful shutdown — handle `SIGTERM` (Spring Boot does this by default) so in-flight requests complete before the container stops (important for zero-downtime deploys in Kubernetes).

**Follow-up:** *"How does the JVM know how much memory it can use inside a container with a memory limit?"* → Since Java 10+ (backported to 8u191+), the JVM is container-aware and reads cgroup limits automatically to size the default heap (`-XX:MaxRAMPercentage` controls what % of the container's memory limit the heap can use, default 25% — often bumped up explicitly for memory-heavy services).

---

## 23. Kafka / RabbitMQ (Messaging)

### Q23.1 — Kafka core concepts: topics, partitions, consumer groups, offsets — and delivery guarantees.

**Answer:**

**Core concepts:**
- **Topic** — a named stream of records/events (e.g., `order-events`).
- **Partition** — a topic is split into partitions for parallelism; each partition is an ordered, immutable, append-only log. **Ordering is only guaranteed within a single partition**, not across the whole topic.
- **Producer** — writes records to a topic (optionally with a partition key — records with the same key always go to the same partition, preserving order for that key, e.g., all events for one `orderId`).
- **Consumer Group** — a set of consumers that share the work of consuming a topic; each partition is consumed by exactly **one** consumer within a group at a time (so max useful parallelism = number of partitions — more consumers than partitions means some sit idle).
- **Offset** — a per-partition, monotonically increasing position marker; consumers track offsets (committed to an internal Kafka topic `__consumer_offsets`) to know where they left off.
- **Broker** — a Kafka server; a cluster consists of multiple brokers. Each partition has one **leader** broker (handles all reads/writes) and zero or more **replica** followers (for fault tolerance).

```
Topic: order-events (3 partitions)
Partition 0: [msg1][msg4][msg7]...
Partition 1: [msg2][msg5][msg8]...
Partition 2: [msg3][msg6][msg9]...

Consumer Group "inventory-service":
 Consumer A -> Partition 0
 Consumer B -> Partition 1
 Consumer C -> Partition 2
```

**Delivery semantics:**

| Guarantee | Meaning | How achieved |
|---|---|---|
| **At-most-once** | Message may be lost, never duplicated | Commit offset before processing |
| **At-least-once** (most common default) | Message never lost, may be duplicated on retry/rebalance | Commit offset after successful processing — requires **idempotent consumers** |
| **Exactly-once** | No loss, no duplication | Kafka transactions (`enable.idempotence`, transactional producer/consumer API) — more overhead, used for critical financial-grade pipelines |

**Spring Boot Kafka producer/consumer:**
```java
@Service
public class OrderEventPublisher {
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    public void publish(OrderEvent event) {
        kafkaTemplate.send("order-events", event.getOrderId(), event); // key = orderId -> ordering per order preserved
    }
}

@Component
public class OrderEventConsumer {
    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    public void consume(OrderEvent event, Acknowledgment ack) {
        processIdempotently(event); // must be safe to run twice (at-least-once semantics)
        ack.acknowledge(); // manual commit after successful processing
    }
}
```

**Why idempotent consumers matter:** Because at-least-once delivery is the practical default, consumers can receive duplicate messages (e.g., after a rebalance where offset wasn't committed before a crash) — design consumers to be safe against reprocessing (e.g., check `processed_events` table by message ID before acting, use DB unique constraints, or make the operation naturally idempotent like `UPSERT`/`SET status = X` rather than `INCREMENT`).

**Real-world:** Order-processing pipeline: `order-service` publishes `OrderCreated` events keyed by `orderId`; `inventory-service`, `notification-service`, and `analytics-service` each run their own consumer group, independently consuming the same topic at their own pace — decoupled, scalable, and resilient (a slow/down consumer group doesn't block others or the producer).

**Best Practices:**
- Choose a partition key that gives good load distribution AND preserves necessary ordering (e.g., `orderId`, not a constant or fully random value).
- Set consumer count ≤ partition count for full parallelism utilization.
- Use manual offset commit (`ack.acknowledge()`) with idempotent processing over auto-commit for critical pipelines — auto-commit can lose or duplicate messages around crashes/rebalances.
- Monitor **consumer lag** (difference between latest produced offset and consumer's committed offset) — a growing lag signals consumers can't keep up.

**Common Mistakes:**
- Assuming global ordering across a topic (only per-partition ordering is guaranteed).
- Not handling rebalances gracefully — long-running message processing without proper `max.poll.interval.ms` tuning can trigger unwanted rebalances mid-processing.
- Treating Kafka like a traditional queue (message deletion/ack-and-remove) — Kafka retains messages per its retention policy regardless of consumption; consumers can replay from any offset, which is a feature (replay/reprocessing capability), not a bug.

**Follow-up:** *"Kafka vs RabbitMQ — when would you choose one over the other?"* → 

| | Kafka | RabbitMQ |
|---|---|---|
| Model | Distributed commit log (pull-based, retains messages) | Traditional message broker (push-based, AMQP) |
| Throughput | Very high (millions/sec), built for streaming/event-sourcing | Lower, but very flexible routing |
| Message retention | Configurable retention (time/size), consumers control their own offset/replay | Messages typically deleted after ack (unless using streams plugin) |
| Routing flexibility | Simple (topic + partition key) | Rich (exchanges: direct/topic/fanout/headers, complex routing rules) |
| Ordering | Per-partition | Per-queue (with a single consumer) |
| Best for | Event streaming, event sourcing, high-throughput pipelines, log aggregation, replay-able event history | Complex routing needs, task queues, RPC-style patterns, lower-latency small messages, priority queues |

**Follow-up:** *"What happens during a consumer group rebalance?"* → When a consumer joins/leaves a group (or crashes/is deemed dead via missed heartbeats), Kafka's group coordinator reassigns partitions among the remaining consumers — briefly pausing consumption for affected partitions. Frequent/unnecessary rebalances (e.g., due to slow processing exceeding `max.poll.interval.ms`) hurt throughput — tune poll intervals and consider using `CooperativeStickyAssignor` (incremental rebalancing, Kafka 2.4+) to minimize disruption compared to the older "stop-the-world" eager rebalance protocol.

---

## 24. Rapid-Fire Tricky Questions Bank (with model answers)

**Q: Why is `String[]` covariant but generics like `List<String>` are invariant?**
A: Arrays are covariant (`Object[] arr = new String[5];` compiles) for historical reasons (predates generics), but this is unsafe — `arr[0] = 42;` compiles but throws `ArrayStoreException` at runtime. Generics were deliberately designed **invariant** (`List<Object> l = new ArrayList<String>();` does NOT compile) specifically to catch this class of error at **compile time** instead of runtime — a key generics design improvement over arrays.

**Q: Can a `static` method be `abstract`?**
A: No — `abstract` requires overriding via dynamic dispatch on an instance; `static` methods belong to the class, not resolved via runtime polymorphism, so the combination is contradictory and a compile error.

**Q: Why does `Integer` caching (`-128` to `127`) cause a subtle `==` bug?**
A: `Integer.valueOf()` caches boxed values in range `-128..127` (`IntegerCache`) for performance — so `Integer a = 100, b = 100; a == b` is `true` (same cached object), but `Integer a = 200, b = 200; a == b` is `false` (different objects, outside cache range). Always use `.equals()` for wrapper comparison, never `==`.

**Q: What is the diamond problem in Java, and how do default interface methods handle it?**
A: If a class implements two interfaces with the same default method signature, Java **forces** the class to explicitly override and resolve it (can call `InterfaceA.super.method()` to pick one), avoiding C++-style ambiguous diamond inheritance — the compiler refuses to silently pick one for you.

**Q: Why shouldn't you use `finalize()` (and what replaced it)?**
A: `finalize()` (deprecated since Java 9, marked for removal) has unpredictable timing (GC-dependent, may never run), can resurrect objects, hurts GC performance, and can silently swallow exceptions. Use **`try-with-resources`** + `AutoCloseable`, or `java.lang.ref.Cleaner` (Java 9+) for last-resort cleanup tied to object reachability — deterministic resource management via explicit `close()` is always preferred.

**Q: Is Java "pass by value" or "pass by reference"?**
A: **Always pass-by-value.** For objects, the *value of the reference* (the memory address/pointer) is copied — so you can mutate the object the reference points to (visible to the caller), but reassigning the parameter inside the method to a new object does NOT affect the caller's original reference. This is a very common point of confusion/trick question.
```java
void reassign(StringBuilder sb) { sb = new StringBuilder("new"); } // caller's reference unaffected
void mutate(StringBuilder sb) { sb.append("!"); }                   // caller's object IS mutated
```

**Q: Why is `HashMap` not thread-safe, specifically — what breaks?**
A: Concurrent structural modification (resize, bucket chain manipulation) without synchronization can corrupt internal linked-list/tree structures, cause lost updates, or (in Java 7) create an infinite loop during concurrent resize due to pointer manipulation races. Java 8 improved some failure modes but `HashMap` is still fundamentally not safe for concurrent writes.

**Q: What's the difference between `this()` and `super()` in constructors — can you use both in one constructor?**
A: No — both must be the **first statement** in a constructor, so only one can be used per constructor (either delegate to another constructor in the same class via `this(...)`, or the parent class via `super(...)`, not both from the same constructor body). If neither is explicitly written, the compiler inserts an implicit `super()` call.

**Q: Why does Spring recommend constructor injection over field injection, beyond "just testability"?**
A: Immutability (`final` fields), fail-fast startup (missing dependency = immediate `NoSuchBeanDefinitionException` at context startup, not a `NullPointerException` deep in some request path later), guaranteed complete initialization (no window where the bean exists but dependencies aren't set), and it surfaces excessive dependencies (a long constructor signature is a visible code smell prompting a refactor, whereas field injection hides "too many responsibilities" bloat).

---

## Final Prep Checklist Before Your Interview

- [ ] Can you explain `HashMap` internals (hashing, treeification, resize) without notes?
- [ ] Can you draw the Spring Bean lifecycle and explain the AOP proxy self-invocation trap?
- [ ] Can you explain N+1 and demonstrate 2 fix strategies with code?
- [ ] Can you explain `happens-before`, `volatile`, and why `count++` isn't atomic even with `volatile`?
- [ ] Can you explain Virtual Threads and when they help/don't help?
- [ ] Can you whiteboard a rate limiter or LRU cache from scratch?
- [ ] Can you explain Kafka partitioning/consumer groups and idempotent consumer design?
- [ ] Can you articulate SOLID principles with your own project examples, not textbook ones?
- [ ] Do you have 2-3 real production war stories ready (a bug you diagnosed, a performance issue you fixed, a design tradeoff you made) — these are gold for "tell me about a challenge you faced" questions?

Good luck — walk in ready to have a technical conversation, not recite definitions.
