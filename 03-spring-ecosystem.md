# Java + Spring Boot Interview Handbook (4+ YOE)
## Part 3: Spring Core, Spring Boot, Spring MVC, Spring Data JPA/Hibernate, Spring Security, REST APIs, Microservices

---

## 9. Spring Core (IoC & DI)

### Q9.1 — Explain IoC and DI. What problem do they solve?

**Answer:** **Inversion of Control (IoC)** is a design principle where the control of object creation and lifecycle management is transferred from the application code to a framework/container. **Dependency Injection (DI)** is the specific technique Spring uses to implement IoC — dependencies are "injected" into a class from the outside rather than the class creating them itself.

**Problem it solves:** Without DI, classes tightly couple to concrete implementations (`new UserServiceImpl()` inside a controller) — hard to test (can't mock), hard to swap implementations, and violates the Dependency Inversion Principle (depend on abstractions, not concretions).

```java
// Without DI — tight coupling
class OrderController {
    private OrderService service = new OrderServiceImpl(new OrderRepositoryImpl());
}

// With DI — Spring wires the dependency, class only depends on the interface
@RestController
class OrderController {
    private final OrderService service;
    public OrderController(OrderService service) { this.service = service; } // constructor injection
}
```

**Types of DI in Spring:**
1. **Constructor Injection** (recommended) — dependencies passed via constructor; enables `final` fields, guarantees the object is fully initialized (no partially-constructed state), makes testing trivial (`new OrderController(mockService)`), and fails fast (missing bean = startup error, not a runtime NPE).
2. **Setter Injection** — used for optional dependencies.
3. **Field Injection** (`@Autowired` on field directly) — discouraged: hides dependencies (not visible in constructor signature), makes the class harder to unit test without Spring/reflection, can't use `final`, and allows circular dependencies to silently exist (constructor injection fails fast on circular deps, exposing design issues early).

**Best Practice (Spring team's own recommendation since Spring 4.3+):** Always use constructor injection; if a class has a single constructor, `@Autowired` is even optional (implicit).

**Follow-up 🎯:** *"How does Spring resolve circular dependencies?"* → With constructor injection, circular dependency **fails at startup** (`BeanCurrentlyInCreationException`) — forcing you to redesign (extract a shared interface, use setter injection, or refactor to break the cycle, which is usually a sign of poor separation of concerns). With field/setter injection, Spring can resolve simple circular dependencies using early-exposure of a partially-initialized bean via the third-level `singletonFactories` cache (part of the 3-level cache mechanism) — but this is generally considered a code smell, not something to design around.

---

### Q9.2 — Explain the Spring Bean Lifecycle in detail.

**Answer:**

```
1. Instantiate bean (constructor call)
2. Populate properties (dependency injection via setters/fields)
3. BeanNameAware.setBeanName()
4. BeanFactoryAware.setBeanFactory() / ApplicationContextAware.setApplicationContext()
5. BeanPostProcessor.postProcessBeforeInitialization() [for all registered post-processors]
6. @PostConstruct method / InitializingBean.afterPropertiesSet() / custom init-method
7. BeanPostProcessor.postProcessAfterInitialization() [e.g., AOP proxies created HERE]
8. Bean ready for use
   ... (application runs) ...
9. @PreDestroy method / DisposableBean.destroy() / custom destroy-method  (on container shutdown)
```

```java
@Component
class MyService implements InitializingBean, DisposableBean {
    @PostConstruct
    public void init() { System.out.println("PostConstruct — preferred, standard JSR-250 annotation"); }
    public void afterPropertiesSet() { System.out.println("InitializingBean — Spring-specific interface"); }
    @PreDestroy
    public void cleanup() { System.out.println("cleanup resources"); }
    public void destroy() { }
}
```

**Best Practice:** Prefer `@PostConstruct`/`@PreDestroy` (standard, framework-agnostic) over implementing `InitializingBean`/`DisposableBean` (couples code to Spring API directly).

**Real-world use:** `@PostConstruct` commonly used to warm up caches, validate configuration on startup, or initialize connection pools; `@PreDestroy` to gracefully close connections/flush buffers on shutdown (important for graceful shutdown in Kubernetes pods receiving `SIGTERM`).

**Follow-up:** *"What is a `BeanPostProcessor` and where is it used internally by Spring?"* → An extension point that intercepts every bean before/after initialization — Spring's own AOP proxy creation (`@Transactional`, `@Async`), `@Autowired` processing, and validation annotation processing are all implemented via `BeanPostProcessor`s internally.

---

### Q9.3 — Bean Scopes — explain each and a real bug caused by scope mismatch.

**Answer:**

| Scope | Lifecycle |
|---|---|
| `singleton` (default) | One instance per Spring container |
| `prototype` | New instance every time it's requested/injected |
| `request` | One instance per HTTP request (web-aware context only) |
| `session` | One instance per HTTP session |
| `application` | One instance per `ServletContext` |
| `websocket` | One instance per WebSocket session |

**Classic scope-mismatch bug 🎯:** Injecting a `prototype`-scoped bean into a `singleton`-scoped bean via normal field/constructor injection — Spring resolves the dependency **once at singleton creation time**, so you effectively get the SAME prototype instance forever, defeating the purpose of "new instance every time."

**Fix — Scoped Proxy or `ObjectProvider`/`ObjectFactory`:**
```java
@Component
@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE, proxyMode = ScopedProxyMode.TARGET_CLASS)
class Cart { }

@Component
class OrderService { // singleton
    @Autowired private ObjectProvider<Cart> cartProvider;
    public void process() {
        Cart cart = cartProvider.getObject(); // fresh prototype instance EVERY call
    }
}
```

**Follow-up:** *"Is `singleton` scope thread-safe by default?"* → No — Spring doesn't make singleton beans thread-safe automatically; it only guarantees one shared instance. If that bean has mutable instance state, you must synchronize access yourself, or better, keep singleton beans **stateless** (the standard, recommended pattern for `@Service`/`@Repository` beans).

---

### Q9.4 — `@Component` vs `@Service` vs `@Repository` vs `@Controller` — are they functionally different?

**Answer:** All are meta-annotated with `@Component`, so functionally (as far as component-scanning/bean registration) they're identical — but they carry **semantic meaning** and some add **extra behavior**:

- `@Repository` — additionally enables **exception translation**: a `PersistenceExceptionTranslationPostProcessor` wraps the bean so that platform-specific exceptions (e.g., `SQLException`, Hibernate exceptions) are translated into Spring's unified `DataAccessException` hierarchy — decoupling service-layer code from the specific persistence technology's exception types.
- `@Service` — purely semantic (marks business-logic layer); no extra framework behavior beyond `@Component`.
- `@Controller` — marks Spring MVC controllers; combined with `@ResponseBody` (or use `@RestController` = `@Controller` + `@ResponseBody`) to write REST APIs. Enables request-mapping method detection.
- `@Component` — generic stereotype for any Spring-managed bean.

**Best Practice:** Always use the most specific stereotype — improves readability, and tools/AOP pointcuts can target layers precisely (e.g., a logging aspect around all `@Repository` methods).

---

## 10. Spring AOP

### Q10.1 — Explain Spring AOP — concepts, proxy mechanism, and how `@Transactional` actually works internally.

**Answer:** AOP (Aspect-Oriented Programming) modularizes **cross-cutting concerns** (logging, security, transactions, caching) that would otherwise be scattered across many classes.

**Key terms:**
- **Aspect:** module encapsulating a cross-cutting concern (e.g., `LoggingAspect`).
- **Join point:** a point during execution (Spring AOP only supports **method execution** join points, unlike full AspectJ which supports field access, constructor calls, etc.).
- **Advice:** action taken at a join point — `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around`.
- **Pointcut:** expression matching which join points an advice applies to.
- **Weaving:** process of linking aspects with target objects. Spring AOP uses **runtime proxy-based weaving** (not compile-time, unlike full AspectJ).

**Proxy mechanism — the interview favorite:**
- If the target bean **implements an interface** → Spring uses **JDK Dynamic Proxy** (`java.lang.reflect.Proxy`) — proxy implements the same interface(s).
- If the target bean has **no interface** → Spring uses **CGLIB** (bytecode generation) — creates a runtime subclass of the target class.

**How `@Transactional` really works:**
1. `@EnableTransactionManagement` registers a `BeanPostProcessor` that wraps `@Transactional`-annotated beans in a proxy.
2. When you call a `@Transactional` method **from outside** the bean, the call goes through the proxy → proxy starts a transaction (via `PlatformTransactionManager`) → delegates to the real method → commits/rolls back based on outcome.
3. **Self-invocation problem 🎯:** if a method within the same class calls another `@Transactional` method on `this` (`this.otherMethod()`), the call **bypasses the proxy entirely** (it's a direct JVM method call, not going through the container-managed proxy) — so the transactional advice **does not apply**. This is one of THE most common `@Transactional` bugs in production code and a favorite trick question.

```java
@Service
class OrderService {
    public void placeOrder() {
        this.saveOrder(); // BUG: self-invocation bypasses proxy — @Transactional on saveOrder() is IGNORED
    }
    @Transactional
    public void saveOrder() { /* ... */ }
}
```
**Fix:** move the transactional method to a separate bean, inject a self-reference via `ApplicationContext`, or use `AopContext.currentProxy()` (requires exposing the proxy).

**Other `@Transactional` gotchas:**
- By default, only rolls back on **unchecked** exceptions (`RuntimeException`/`Error`); checked exceptions do NOT trigger rollback unless you specify `@Transactional(rollbackFor = Exception.class)`.
- `@Transactional` on a `private` method silently does nothing (Spring proxies can't intercept private/final methods since they can't be overridden in a subclass proxy — CGLIB requires overridable methods).
- Propagation defaults to `REQUIRED` (join existing transaction or create new one).

**Propagation types (asked frequently):**

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Join existing transaction, or create new if none exists |
| `REQUIRES_NEW` | Always suspend existing transaction and start a new independent one |
| `NESTED` | Creates a savepoint within the existing transaction; rollback can go back to savepoint only |
| `MANDATORY` | Must run within an existing transaction; throws exception if none exists |
| `SUPPORTS` | Runs within a transaction if one exists, else non-transactionally |
| `NOT_SUPPORTED` | Suspends existing transaction, runs without one |
| `NEVER` | Throws exception if a transaction exists |

**Real-world:** Audit logging that must persist even if the main business transaction rolls back — implemented with `REQUIRES_NEW` so the audit insert commits independently of the outer transaction's outcome.

**Best Practices:**
- Keep `@Transactional` methods focused and avoid long-running transactions (holds DB connections/locks longer than necessary).
- Never catch and swallow exceptions inside a `@Transactional` method without rethrowing — silently prevents rollback.
- Apply `@Transactional` at the service layer, not the repository/DAO layer, to correctly scope the unit of work across multiple repository calls.
- Set `readOnly = true` on read-only transactional methods — allows some JDBC drivers/Hibernate to optimize (e.g., skip dirty checking flush).

**Common Mistakes:**
- Self-invocation bug (above).
- Assuming checked exceptions trigger rollback by default (they don't).
- Wrapping the entire controller method in one giant transaction spanning external API calls — holds DB connections/locks during slow I/O, hurting throughput.

**Follow-up:** *"Why does Spring AOP only support method-level join points, unlike full AspectJ?"* → Because Spring AOP is proxy-based (works at runtime via generated subclasses/interface proxies) — it can only intercept calls that go **through the proxy object**, i.e., public method calls on the bean from outside. Full AspectJ uses compile-time or load-time bytecode weaving, enabling field-level and constructor join points too, but with more build complexity.

---

## 11. Spring Boot Essentials

### Q11.1 — What does `@SpringBootApplication` actually do, and explain Auto-Configuration internals.

**Answer:** `@SpringBootApplication` is a composite meta-annotation combining:
1. `@SpringBootConfiguration` (specialization of `@Configuration`) — marks the class as a source of bean definitions.
2. `@EnableAutoConfiguration` — triggers Spring Boot's auto-configuration mechanism.
3. `@ComponentScan` — scans the package (and sub-packages) of the annotated class for components.

**Auto-configuration internals:** Spring Boot ships with `spring-boot-autoconfigure.jar`, which contains hundreds of `@Configuration` classes guarded by **conditional annotations**:
- `@ConditionalOnClass` — only activate if a given class is on the classpath (e.g., `DataSourceAutoConfiguration` activates only if a JDBC driver is present).
- `@ConditionalOnMissingBean` — only activate if the user hasn't already defined their own bean of that type (lets you override defaults simply by defining your own `@Bean`).
- `@ConditionalOnProperty` — activate based on `application.properties`/`yml` values.
- `@ConditionalOnWebApplication` — only for web contexts.

Since Spring Boot 2.7+/3.x, these are registered via `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (older versions used `spring.factories`).

**Real-world:** Adding `spring-boot-starter-data-jpa` to your `pom.xml` → `DataSourceAutoConfiguration` + `HibernateJpaAutoConfiguration` detect the JDBC driver + `EntityManagerFactory` requirements and auto-wire a `DataSource`, `EntityManagerFactory`, and `PlatformTransactionManager` — zero manual XML/Java config needed, unlike classic Spring where you'd hand-configure all of this.

**Best Practices:**
- Use `@ConditionalOnMissingBean` pattern when writing your own starter/auto-config to let consumers override defaults.
- Use `spring-boot-configuration-processor` to generate metadata for custom `@ConfigurationProperties`, enabling IDE auto-complete.
- Exclude unwanted auto-configurations explicitly when needed: `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)`.

**Follow-up:** *"How would you debug which auto-configurations were applied/skipped?"* → Run with `--debug` flag or set `debug=true` in properties — Spring Boot logs a full **Conditions Evaluation Report** showing positive/negative matches for every auto-configuration class.

---

### Q11.2 — `application.properties` vs `application.yml`, Profiles, and externalized configuration order.

**Answer:** Both configure the same underlying `Environment`; YAML supports hierarchical structure more concisely and native lists/maps, while properties files are flatter (dot-notation). Functionally interchangeable for most purposes; YAML doesn't support `@PropertySource` multi-document import the same way, and Spring Boot has some quirks with YAML + relaxed binding on complex types.

**Profiles:** `application-{profile}.yml` overrides/extends `application.yml`, activated via `spring.profiles.active=dev` (env var, JVM arg, or `application.yml`). Enables environment-specific config (dev/test/staging/prod) without code changes. `@Profile("dev")` on beans conditionally registers them.

**Property resolution precedence (highest to lowest — commonly asked, MUST know for real debugging):**
1. Command-line arguments (`--server.port=8081`)
2. `SPRING_APPLICATION_JSON` (embedded JSON in env var)
3. Servlet parameters / JNDI attributes
4. `System.getProperties()`
5. OS environment variables
6. Profile-specific properties **outside** the packaged jar (`application-{profile}.yml` in `config/` dir)
7. Profile-specific properties **inside** the packaged jar
8. Application properties **outside** the jar
9. Application properties **inside** the jar (`src/main/resources/application.yml`)
10. `@PropertySource` annotations
11. Default properties (`SpringApplication.setDefaultProperties`)

**Real-world:** In Kubernetes, config is typically injected via environment variables (ConfigMaps/Secrets) or mounted volumes overriding `application.yml` inside the jar — this precedence order is exactly why that works without rebuilding the image.

**Best Practice:** Use `@ConfigurationProperties` (type-safe, validated config binding) over scattered `@Value("${...}")` for related groups of properties:
```java
@ConfigurationProperties(prefix = "app.payment")
@Validated
public record PaymentProperties(
    @NotBlank String gatewayUrl,
    @Min(1) int timeoutSeconds
) {}
```

**Common Mistake:** Committing environment-specific secrets (DB passwords, API keys) directly into `application.yml` in source control — use externalized secret managers (Vault, AWS Secrets Manager, Kubernetes Secrets) or at minimum environment variables.

---

### Q11.3 — Spring Boot Actuator — what is it, and which endpoints matter in production?

**Answer:** Actuator exposes production-ready monitoring/management endpoints out of the box.

| Endpoint | Purpose |
|---|---|
| `/actuator/health` | Liveness/readiness — used by Kubernetes probes, load balancers |
| `/actuator/metrics` | JVM, HTTP, custom metrics (integrates with Micrometer → Prometheus/Datadog) |
| `/actuator/info` | Build/version info |
| `/actuator/env` | Current environment properties (sensitive — should be secured) |
| `/actuator/loggers` | View/change log levels at runtime without redeployment |
| `/actuator/threaddump`, `/actuator/heapdump` | Diagnostics for production incident investigation |
| `/actuator/prometheus` | Prometheus-formatted metrics scrape endpoint (needs `micrometer-registry-prometheus`) |

**Real-world:** Kubernetes readiness probe hits `/actuator/health/readiness` (separate from liveness since Spring Boot 2.3+ with `management.endpoint.health.probes.enabled=true`) — readiness fails if DB connection pool isn't ready, preventing traffic routing to a not-yet-ready pod; liveness probe restarts the pod if the app is deadlocked/unresponsive.

**Best Practices:**
- Never expose all actuator endpoints publicly — restrict via Spring Security, expose only `health` externally, keep the rest on an internal management port (`management.server.port`).
- Implement custom `HealthIndicator` beans for critical downstream dependencies (DB, message broker, third-party APIs).
- Integrate with Micrometer for custom business metrics (`Counter`, `Timer`, `Gauge`) pushed to Prometheus/Grafana dashboards.

**Common Mistake:** Leaving `/actuator/env` or `/actuator/heapdump` publicly accessible in production — serious security/data-leak risk (can expose secrets/credentials in environment properties).

---

## 12. Spring MVC & REST APIs

### Q12.1 — Explain the DispatcherServlet request lifecycle (front controller pattern).

**Answer:**

```
Client Request
     |
     v
DispatcherServlet (Front Controller)
     |
     v
HandlerMapping — determines which controller/method handles this URL
     |
     v
HandlerAdapter — invokes the actual controller method
     |
     v
Controller method executes (business logic via Service layer)
     |
     v
Returns ModelAndView (traditional MVC) OR object (REST, serialized by HttpMessageConverter)
     |
     v
ViewResolver (for traditional MVC — resolves view name to actual View, e.g. JSP/Thymeleaf)
   OR
HttpMessageConverter (for @RestController — serializes return object to JSON/XML via Jackson)
     |
     v
Response sent to client
```

Additional cross-cutting layer: **`HandlerInterceptor`**s (`preHandle`, `postHandle`, `afterCompletion`) run before/after the controller — used for logging, auth checks, request timing, similar in spirit to AOP but Spring-MVC-specific (works at the servlet layer, not method-proxy layer).

**Follow-up:** *"Interceptor vs Filter — what's the difference?"* → `Filter` (Servlet spec) runs **before the DispatcherServlet**, container-managed, has no access to Spring's handler/controller metadata (just raw request/response), and works across the whole servlet context (not Spring-aware). `HandlerInterceptor` runs **inside** the DispatcherServlet's dispatch, Spring-managed, and has access to the resolved `HandlerMethod` (can inspect target controller method/annotations) — useful for controller-specific cross-cutting logic (e.g., `@RequiresAuth` custom annotation checks).

---

### Q12.2 — REST API design best practices interviewers expect a 4+ YOE candidate to articulate.

**Answer:**

**Resource naming & HTTP verbs:**
```
GET    /api/orders          -> list orders
GET    /api/orders/{id}     -> get one order
POST   /api/orders          -> create order
PUT    /api/orders/{id}     -> full update (idempotent)
PATCH  /api/orders/{id}     -> partial update
DELETE /api/orders/{id}     -> delete order
```
- Use **nouns**, not verbs, in URLs (`/orders`, not `/getOrders`).
- Use plural resource names consistently.
- Nest resources logically but avoid deep nesting (`/customers/{id}/orders`, not `/customers/{id}/orders/{oid}/items/{iid}/details/{did}`).

**HTTP status codes (know these cold):**

| Code | Meaning | Use case |
|---|---|---|
| 200 | OK | Successful GET/PUT/PATCH |
| 201 | Created | Successful POST — include `Location` header |
| 204 | No Content | Successful DELETE, nothing to return |
| 400 | Bad Request | Validation failure, malformed input |
| 401 | Unauthorized | Missing/invalid authentication |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Version conflict, duplicate resource |
| 422 | Unprocessable Entity | Semantically invalid data |
| 429 | Too Many Requests | Rate limiting |
| 500 | Internal Server Error | Unhandled exception |
| 503 | Service Unavailable | Downstream dependency down, circuit open |

**Idempotency:** `GET`, `PUT`, `DELETE` should be idempotent (same request repeated = same result/state); `POST` is not. Important for retry logic in distributed systems — clients/gateways can safely retry idempotent calls after timeouts.

**Versioning strategies:**
- URI versioning: `/api/v1/orders` (simplest, most common, but "pollutes" the URL).
- Header versioning: `Accept: application/vnd.company.v2+json` (cleaner URLs, harder to test manually).
- Query param: `/api/orders?version=2` (least preferred).

**Validation:**
```java
public record CreateOrderRequest(
    @NotNull @Positive Long customerId,
    @NotEmpty List<@Valid OrderItemRequest> items,
    @Email String contactEmail
) {}

@PostMapping("/orders")
public ResponseEntity<OrderResponse> create(@Valid @RequestBody CreateOrderRequest req) { ... }
// MethodArgumentNotValidException auto-thrown on failure -> handle in @ControllerAdvice -> 400
```

**Pagination & Filtering:**
```java
@GetMapping("/orders")
public Page<OrderResponse> list(@PageableDefault(size = 20, sort = "createdAt") Pageable pageable,
                                  @RequestParam Optional<String> status) { ... }
```

**HATEOAS (rarely used in practice but asked conceptually):** REST responses include hyperlinks to related actions/resources (`Spring HATEOAS` library) — theoretically the "purest" REST (Richardson Maturity Model Level 3), rarely fully adopted in enterprise due to added complexity vs benefit.

**Best Practices:**
- Consistent error response shape across the API (`{timestamp, status, error, message, path}`).
- Use DTOs, never expose JPA entities directly in REST responses (avoids lazy-loading serialization issues, over-exposure of internal fields, and tight coupling between API contract and DB schema).
- Version your API from day one, even if just `/v1/`.
- Use `ResponseEntity<T>` for full control over status codes/headers rather than relying solely on exception mapping.

**Common Mistakes:**
- Returning `200 OK` for errors with an error message in the body (breaks HTTP semantics, confuses clients/monitoring).
- Exposing JPA entities directly → causes `LazyInitializationException` when Jackson tries to serialize uninitialized lazy associations outside a transaction.
- Not validating input at the API boundary — pushes garbage data deeper into the system.
- Inconsistent naming/casing across endpoints (`camelCase` vs `snake_case` mixed).

**Follow-up:** *"How do you handle `LazyInitializationException` when serializing entities?"* → Best fix: never serialize entities directly — map to DTOs inside the (still-open) transactional service method. Alternative workarounds: `@Transactional` spanning the serialization (anti-pattern, keeps DB connection open longer), `Hibernate.initialize()` explicitly, or `FetchType.EAGER` (careful — can cause N+1 or over-fetching elsewhere).

---

## 13. Spring Data JPA & Hibernate

### Q13.1 — Explain the JPA/Hibernate architecture: `EntityManager`, Persistence Context, and entity states.

**Answer:**

**Entity lifecycle states:**
```
        new Entity()
             |
             v
        [TRANSIENT] ---persist()---> [MANAGED] ---remove()---> [REMOVED]
                                          |  ^
                                    detach()|  |merge()
                                          v  |
                                      [DETACHED]
```
- **Transient:** new object, not associated with any persistence context, no DB row.
- **Managed (Persistent):** associated with an active `EntityManager`/persistence context — changes are automatically tracked (**dirty checking**) and flushed to DB at transaction commit/flush time, no explicit `save()` call needed for updates.
- **Detached:** was managed, but the persistence context closed (e.g., transaction ended) — object still has data but changes are NOT tracked anymore.
- **Removed:** marked for deletion, will be deleted at flush/commit.

**Persistence Context** = first-level cache, scoped to an `EntityManager` (in Spring, typically scoped to a transaction). Ensures within one transaction, repeated `find()` calls for the same entity/ID return the **same Java object instance** (identity map pattern) — no redundant SQL for repeat lookups within the same context.

**Dirty Checking:** Hibernate compares a managed entity's current state against a snapshot taken when it was loaded; at flush time, generates `UPDATE` statements only for changed fields (or full field set depending on `@DynamicUpdate` config) — **you never need to call `save()` explicitly to persist changes to a managed entity within a transaction.**

```java
@Transactional
public void updatePrice(Long productId, BigDecimal newPrice) {
    Product p = productRepository.findById(productId).orElseThrow();
    p.setPrice(newPrice); // no explicit save() needed — dirty checking handles it at commit/flush
}
```

**Follow-up:** *"Why does `save()` sometimes still make sense even in JPA?"* → For NEW (transient) entities, `save()`/`persist()` is needed to transition them into managed state and get an assigned ID.

---

### Q13.2 🎯 — N+1 Select Problem — what is it, how do you detect it, and how do you fix it?

**Answer:** Classic and near-guaranteed interview question for JPA experience.

**The problem:** Fetching a list of parent entities (1 query) then lazily fetching each parent's associated collection/child individually inside a loop (N additional queries) — total N+1 queries instead of a single optimized join.

```java
List<Order> orders = orderRepository.findAll(); // 1 query
for (Order o : orders) {
    o.getItems().size(); // N queries — one per order, since items is LAZY by default for collections
}
```

**Detection:** Enable SQL logging (`spring.jpa.show-sql=true`, or better, `logging.level.org.hibernate.SQL=DEBUG` + `org.hibernate.type.descriptor.sql=TRACE` for bind params), or use a tool like **p6spy**/**datasource-proxy**, or Hibernate's statistics (`spring.jpa.properties.hibernate.generate_statistics=true`) to count actual queries executed vs expected.

**Fixes (in order of preference):**

1. **`JOIN FETCH` in JPQL** — explicit eager fetch for a specific query, avoids changing default fetch strategy globally:
   ```java
   @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
   List<Order> findWithItems(@Param("status") String status);
   ```
2. **`@EntityGraph`** — declarative alternative to JOIN FETCH, keeps repository method signatures clean:
   ```java
   @EntityGraph(attributePaths = {"items", "customer"})
   List<Order> findByStatus(String status);
   ```
3. **Batch fetching** (`@BatchSize(size = 20)` on the association, or `hibernate.default_batch_fetch_size` globally) — instead of N individual queries, Hibernate issues `queries/batchSize` queries using `WHERE id IN (...)` — good middle-ground when JOIN FETCH would cause cartesian-product blowup with multiple collections.
4. **DTO projections** — bypass entity fetching entirely for read-only views, selecting only needed columns via JPQL constructor expressions or Spring Data interface projections — most performant for read-heavy APIs.

```java
public interface OrderSummary {
    Long getId();
    BigDecimal getTotal();
}
List<OrderSummary> findByStatus(String status); // Spring Data generates optimized SELECT, no entity hydration
```

**Best Practices:**
- Default to `FetchType.LAZY` for all `@OneToMany`/`@ManyToMany` associations (JPA spec default is actually EAGER for `@ManyToOne`/`@OneToOne` — override these to LAZY too, a very common gotcha).
- Use `@EntityGraph` or `JOIN FETCH` per-query rather than globally switching to EAGER (EAGER everywhere causes over-fetching and its own N+1-like issues on unrelated queries).
- Watch for **multiple collection JOIN FETCH in one query** — causes a Cartesian product explosion (fetching 2 `@OneToMany` collections via JOIN FETCH in the same query multiplies rows); fetch one collection via JOIN FETCH and use batch-fetching or a second query for the other.

**Follow-up:** *"What is `FetchType.LAZY` vs `EAGER` default for each association type?"* → `@OneToMany`/`@ManyToMany` default to LAZY; `@ManyToOne`/`@OneToOne` default to EAGER (per JPA spec) — a commonly-missed detail that causes accidental N+1/over-fetching via seemingly innocent `@ManyToOne` associations.

---

### Q13.3 — Explain `@OneToMany`, `@ManyToOne`, `@ManyToMany`, cascade types, and `orphanRemoval`.

**Answer:**

```java
@Entity
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    private List<OrderItem> items = new ArrayList<>();
}

@Entity
class OrderItem {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order; // owning side — has the foreign key
}
```

**Owning side vs inverse side:** The side with `@JoinColumn` (or the join table owner in `@ManyToMany`) is the **owning side** — it controls what gets persisted to the FK column. `mappedBy` on the other side marks it as the **inverse (non-owning)** side, purely for object navigation — changes made only on the inverse side (without updating the owning side) are **not persisted**. This trips up many candidates:

```java
order.getItems().add(newItem); // WRONG alone — inverse side change, FK won't be set unless also:
newItem.setOrder(order);        // REQUIRED — owning side must be set for the FK to persist correctly
```

**Cascade types:**

| Cascade | Effect |
|---|---|
| `PERSIST` | Saving parent also saves new children |
| `MERGE` | Merging parent also merges children |
| `REMOVE` | Deleting parent also deletes children |
| `REFRESH` | Refreshing parent also refreshes children |
| `DETACH` | Detaching parent also detaches children |
| `ALL` | All of the above |

**`orphanRemoval = true`:** if a child is removed from the parent's collection (even without explicit delete), Hibernate deletes it from the DB at flush — models true parent-owns-child composition (e.g., `Order` → `OrderItem`; an item removed from the order's list should cease to exist).

**`@ManyToMany` real-world caveat:** Rarely used directly in mature systems — usually modeled as two `@OneToMany`/`@ManyToOne` relationships around an explicit join entity (e.g., `StudentCourseEnrollment` with its own `enrollmentDate`, `grade` fields) because pure `@ManyToMany` join tables can't hold extra attributes and are harder to evolve.

**Best Practices:**
- Always set both sides of a bidirectional relationship in a helper method:
  ```java
  public void addItem(OrderItem item) { items.add(item); item.setOrder(this); }
  ```
- Be cautious with `CascadeType.REMOVE`/`ALL` on `@ManyToOne` — accidentally cascading deletes up to shared parent entities (e.g., deleting an `OrderItem` shouldn't cascade-delete the shared `Product`).
- Use `equals()`/`hashCode()` carefully on entities in collections — prefer business key or just object identity, NOT the auto-generated `id` alone (before persist, `id` is null, causing `Set` membership issues).

**Follow-up:** *"What's the difference between `CascadeType.REMOVE` and `orphanRemoval`?"* → `CascadeType.REMOVE` deletes children **only when the parent itself is explicitly deleted**. `orphanRemoval` deletes a child **the moment it's disassociated from the parent's collection**, even if the parent isn't deleted — stronger ownership semantics.

---

### Q13.4 — Explain JPQL vs native queries vs Criteria API vs Spring Data derived query methods — tradeoffs.

**Answer:**

| Approach | Pros | Cons |
|---|---|---|
| **Derived query methods** (`findByStatusAndCustomerId`) | Zero boilerplate, type-safe, readable for simple queries | Gets unwieldy/unreadable for complex conditions; limited expressiveness |
| **JPQL (`@Query`)** | Full query flexibility, entity-object-based (portable across DBs), supports JOIN FETCH | String-based (errors caught at runtime unless using a compile-time checker plugin); no autocomplete natively |
| **Native SQL (`@Query(nativeQuery = true)`)** | Full DB-specific power (window functions, CTEs, DB-specific optimizations/hints) | Not portable across databases; bypasses JPQL entity mapping unless mapped carefully |
| **Criteria API** | Type-safe, programmatically composable (great for dynamic filters built at runtime) | Verbose, harder to read than JPQL |
| **Querydsl** (popular add-on) | Type-safe AND readable/fluent, great for dynamic queries | Extra dependency + code-gen step (Q-classes) |

```java
// Derived
List<Order> findByStatusAndCustomer_Id(String status, Long customerId);

// JPQL
@Query("SELECT o FROM Order o WHERE o.status = :status")
List<Order> findByStatus(@Param("status") String status);

// Native
@Query(value = "SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '7 days'", nativeQuery = true)
List<Order> findRecentOrders();

// Criteria API — good for dynamic search/filter screens
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Order> query = cb.createQuery(Order.class);
Root<Order> root = query.from(Order.class);
List<Predicate> predicates = new ArrayList<>();
if (status != null) predicates.add(cb.equal(root.get("status"), status));
query.where(predicates.toArray(new Predicate[0]));
```

**Best Practice:** Use derived methods for simple lookups, JPQL for moderately complex fixed queries, Criteria API/Querydsl for dynamic search filters (e.g., an admin panel with many optional filter fields), and native SQL sparingly for DB-specific performance-critical queries (with awareness this reduces portability).

**Follow-up:** *"What's the difference between JPQL and SQL?"* → JPQL operates on **entity object graphs** (`SELECT o FROM Order o`, referencing entity/field names, not table/column names) and is translated by Hibernate into the underlying dialect's SQL — portable across databases. Native SQL is the literal database-specific SQL.

---

### Q13.5 — Optimistic vs Pessimistic Locking — how do you prevent lost updates in concurrent transactions?

**Answer:**

**Optimistic Locking (`@Version`):** Assumes conflicts are rare. Each entity has a `@Version` column (int/long/timestamp); on `UPDATE`, Hibernate includes `WHERE version = :loadedVersion` — if 0 rows affected (someone else updated it first, incrementing the version), Hibernate throws `OptimisticLockException`. No DB locks held during the transaction — high concurrency, but requires retry logic on conflict.

```java
@Entity
class Account {
    @Version
    private Long version;
    private BigDecimal balance;
}
```

**Pessimistic Locking:** Acquires an actual DB row lock at read time, blocking other transactions from modifying (or even reading, depending on lock mode) the row until the lock is released.
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Account findByIdForUpdate(@Param("id") Long id); // issues SELECT ... FOR UPDATE
```

**Comparison:**

| | Optimistic | Pessimistic |
|---|---|---|
| Mechanism | Version check at commit | DB row lock held during transaction |
| Concurrency | High (no blocking) | Lower (blocks other transactions) |
| Conflict handling | Exception + retry logic needed | Prevented by blocking |
| Best for | Low-contention, high-read scenarios (most web apps) | High-contention critical sections (e.g., seat booking, inventory decrement, bank balance transfer) |
| Risk | Lost work on conflict (needs retry) | Deadlocks, reduced throughput, held connections |

**Real-world:** E-commerce inventory decrement during flash sales — pessimistic locking (`SELECT ... FOR UPDATE`) on the stock row prevents overselling; optimistic locking would cause a storm of retries under extreme contention, so pessimistic is chosen despite lower throughput, because correctness (no overselling) is critical.

**Follow-up:** *"How do you handle an `OptimisticLockException` gracefully?"* → Catch it (or a wrapping `ObjectOptimisticLockingFailureException` in Spring), retry the operation (re-fetch latest state, reapply the change) with a bounded retry count/backoff, or surface a "please retry, data changed" message to the end user for UI-driven conflicts.

---

## 14. Spring Security

### Q14.1 — Explain the Spring Security filter chain and how a request gets authenticated.

**Answer:** Spring Security integrates as a chain of **Servlet Filters** (`FilterChainProxy`), sitting before the `DispatcherServlet`.

```
Request → SecurityContextPersistenceFilter (loads SecurityContext, e.g. from session)
        → (various filters: CSRF, CORS, Logout, etc.)
        → UsernamePasswordAuthenticationFilter (form login) / BasicAuthenticationFilter / Custom JWT filter
        → AuthenticationManager → AuthenticationProvider → UserDetailsService.loadUserByUsername()
        → PasswordEncoder.matches() (verifies credentials)
        → On success: populates SecurityContextHolder with Authentication object
        → FilterSecurityInterceptor / AuthorizationFilter — checks authorization (roles/permissions) against the endpoint
        → DispatcherServlet → Controller
```

**Key components:**
- **`Authentication`** — represents the current user's credentials/principal + granted authorities.
- **`SecurityContext`/`SecurityContextHolder`** — holds the `Authentication` for the current thread (ThreadLocal-based by default — a common cause of "auth context lost" bugs when work is handed off to another thread, e.g., inside `@Async` methods, without propagating the context).
- **`UserDetailsService`** — loads user-specific data (typically from DB).
- **`AuthenticationProvider`** — performs the actual authentication logic (e.g., `DaoAuthenticationProvider` checks username/password against `UserDetailsService` + `PasswordEncoder`).
- **`AuthenticationManager`** — delegates to the appropriate `AuthenticationProvider`.

**JWT-based stateless authentication (very common in microservices, expect deep questions here):**
```java
@Component
class JwtAuthFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain) {
        String token = extractToken(req);
        if (token != null && jwtUtil.validateToken(token)) {
            String username = jwtUtil.extractUsername(token);
            UserDetails user = userDetailsService.loadUserByUsername(username);
            var auth = new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        chain.doFilter(req, res);
    }
}

@Configuration
@EnableWebSecurity
class SecurityConfig {
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable()) // typically disabled for stateless JWT APIs (no session/cookie CSRF risk)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
    @Bean PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }
}
```

**Why `BCryptPasswordEncoder`?** Adaptive hashing (configurable "strength"/work factor), includes a built-in random salt per hash (preventing rainbow-table attacks), and is intentionally slow (defends against brute-force) — never use plain MD5/SHA-256 for passwords (fast hashes are bad for password storage, good for data integrity checks only).

**Best Practices:**
- Stateless JWT for microservices (no server-side session, horizontally scalable) vs session-based auth for traditional monoliths.
- Store JWT secret/signing key securely (Vault/Secrets Manager), rotate periodically, use short expiry + refresh tokens.
- Always validate JWT signature AND expiry AND issuer/audience claims — not just decode-and-trust.
- Use method-level security (`@PreAuthorize("hasRole('ADMIN')")`) for fine-grained authorization beyond URL patterns.

**Common Mistakes:**
- Disabling CSRF protection for session-cookie-based (non-stateless) apps — reintroduces CSRF vulnerability.
- Storing JWT in `localStorage` (XSS-vulnerable) instead of an `HttpOnly` secure cookie for browser clients.
- Not propagating `SecurityContext` to `@Async`/new threads — leads to `null` authentication in async code (fix: `SecurityContextHolder.setStrategyName(MODE_INHERITABLETHREADLOCAL)` or explicitly pass context).

**Follow-up 🎯:** *"How would you prevent CSRF in a stateless JWT API — do you even need to?"* → CSRF exploits the browser's automatic inclusion of cookies in cross-site requests. If auth is via a JWT sent in an `Authorization` header (not a cookie), the browser won't auto-attach it to a forged cross-site request — so **CSRF risk is largely mitigated by design**, hence CSRF protection is commonly disabled for pure header-token APIs. If JWT is stored in a cookie instead, CSRF protection should stay enabled.

---

## 15. Microservices Architecture

### Q15.1 — Explain core microservices patterns: Service Discovery, Circuit Breaker, API Gateway, Config Server, Saga.

**Answer:**

**1. Service Discovery (Eureka / Consul / Kubernetes DNS):** Services register themselves and discover other services' network locations dynamically, instead of hardcoding IPs/ports — essential since instances scale up/down and get rescheduled to different hosts/pods.

**2. API Gateway (Spring Cloud Gateway / Netflix Zuul / Kong):** Single entry point for clients — handles routing, authentication, rate limiting, request/response transformation, and cross-cutting concerns centrally instead of duplicating in every service.

**3. Circuit Breaker (Resilience4j — Hystrix is deprecated):** Prevents cascading failures when a downstream service is slow/down — after a failure threshold, the circuit "opens" and fails fast (or falls back) instead of piling up threads waiting on a dying dependency.

```java
@CircuitBreaker(name = "inventoryService", fallbackMethod = "fallbackCheckStock")
@Retry(name = "inventoryService")
@TimeLimiter(name = "inventoryService")
public CompletableFuture<StockStatus> checkStock(String sku) {
    return CompletableFuture.supplyAsync(() -> inventoryClient.getStock(sku));
}
public CompletableFuture<StockStatus> fallbackCheckStock(String sku, Throwable t) {
    return CompletableFuture.completedFuture(StockStatus.unknown());
}
```

**Circuit breaker states:** `CLOSED` (normal, requests pass through, failures counted) → threshold exceeded → `OPEN` (requests fail immediately without calling downstream, for a configured wait duration) → after wait → `HALF_OPEN` (allows a few trial requests through) → success → `CLOSED`, failure → back to `OPEN`.

**4. Config Server (Spring Cloud Config):** Centralized, version-controlled (typically backed by Git) external configuration for all services, supporting dynamic refresh (`@RefreshScope`) without redeployment.

**5. Saga Pattern (distributed transactions):** Since microservices each own their own database (no distributed 2PC transaction spans them cleanly), a business process spanning multiple services is broken into a sequence of local transactions, each publishing an event that triggers the next step; if a step fails, **compensating transactions** undo prior steps.
- **Choreography-based:** each service listens for events and reacts independently (no central coordinator) — simpler but harder to trace/debug the overall flow.
- **Orchestration-based:** a central orchestrator explicitly directs each step and handles failure compensation — easier to reason about but introduces a coordination component.

**Real-world example (order placement Saga):** `OrderService` creates order (PENDING) → publishes `OrderCreated` → `PaymentService` charges card → publishes `PaymentCompleted`/`PaymentFailed` → on success, `InventoryService` reserves stock → publishes `StockReserved`/`StockUnavailable` → `OrderService` marks order `CONFIRMED`, or on any failure, triggers compensating actions (refund payment, cancel order).

**6. Database per Service:** Each microservice owns its data exclusively — no service reaches into another's database directly — enforces loose coupling but creates challenges for cross-service queries/joins (solved via API composition or CQRS/read-model replication).

**Best Practices:**
- Design for failure — assume any downstream call can fail/timeout; always have timeouts, retries (with backoff/jitter), and circuit breakers.
- Use asynchronous messaging (Kafka/RabbitMQ) for eventual consistency between services rather than synchronous chains for non-critical-path operations.
- Implement distributed tracing (Sleuth/Micrometer Tracing + Zipkin/Jaeger) — essential for debugging across service boundaries.
- Idempotent consumers — since messages can be redelivered (at-least-once delivery is common), consumers must handle duplicate processing safely.

**Common Mistakes:**
- "Distributed monolith" — services that are technically separate deployables but tightly coupled via synchronous chains and shared databases, getting the downsides of microservices (network complexity) without the benefits (independent deployability).
- No timeout on inter-service HTTP calls — one slow dependency exhausts caller's thread pool/connections, cascading the outage.
- Choosing microservices prematurely for a small team/simple domain — adds massive operational complexity (deployment, monitoring, distributed debugging) not justified by the problem size.

**Follow-up 🎯:** *"How do you handle data consistency across services without distributed transactions?"* → Saga pattern (above) for eventual consistency, or accept eventual consistency with idempotent, retryable event processing; avoid distributed 2PC (XA transactions) — poor fit for microservices due to blocking behavior and coordinator single-point-of-failure, and most modern message brokers/NoSQL stores don't support it well anyway.

**Follow-up:** *"Monolith vs Microservices — how do you decide?"* → Monolith: simpler to develop/deploy/debug initially, better for small teams/early-stage products, no network latency between modules. Microservices: independent scaling/deployment/tech-stack per service, better for large teams working in parallel, but adds operational complexity (service discovery, distributed tracing, network failure handling, data consistency challenges). Many successful systems start as a well-modularized monolith ("modular monolith") and extract services only when a clear scaling/team-boundary need emerges.

---

*(Continue to Part 4: SQL, Design Patterns, System Design/LLD, Testing, Maven/Gradle, Git, Docker, Kafka/RabbitMQ)*
