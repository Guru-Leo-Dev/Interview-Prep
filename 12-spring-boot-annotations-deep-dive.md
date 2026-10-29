# Spring Boot Annotations — Complete Deep-Dive Interview Handbook

> Structured as: **What it does → Internals (what Spring actually does with it) → Code → Real-world Use Case → Best Practices → Common Mistakes → Follow-up Questions**. Tricky questions flagged 🎯. This is organized as a complete annotation catalog, grouped by concern — the way Spring Boot itself is layered — so you can also use it as a reference, not just a linear read.

---

## Table of Contents
1. Bootstrapping Annotations (`@SpringBootApplication` and its parts)
2. Stereotype Annotations (`@Component` family)
3. Dependency Injection Annotations (`@Autowired`, `@Qualifier`, `@Bean`, `@Value`, etc.)
4. Bean Scope & Lifecycle Annotations
5. Web / Spring MVC Annotations
6. Validation Annotations
7. Spring Data JPA / Persistence Annotations
8. Transaction Annotations
9. Configuration & Conditional Annotations
10. Profile & Environment Annotations
11. AOP Annotations
12. Async, Scheduling & Caching Annotations
13. Spring Security Annotations
14. Testing Annotations
15. Lombok Annotations (commonly paired, worth knowing)
16. Annotation Comparison Tables
17. Rapid-Fire Tricky Q&A

---

## 1. Bootstrapping Annotations

### 1.1 `@SpringBootApplication`

**What it does:** The single entry-point annotation on your main class — a composite/meta-annotation bundling three others.

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

**Internals — what it's actually composed of:**
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@SpringBootConfiguration   // = @Configuration, marks this class itself as a bean-definition source
@EnableAutoConfiguration   // triggers the conditional auto-configuration mechanism
@ComponentScan(excludeFilters = { ... })  // scans the package of the annotated class + sub-packages
public @interface SpringBootApplication { ... }
```

**🎯 Frequently-tested consequence of `@ComponentScan`'s default behavior:** It scans the **package of the annotated class and all sub-packages** — nothing above or outside it. This is exactly why the conventional advice is to put your `@SpringBootApplication` class in the **root package** of your project — a bean defined in a sibling or parent package won't be picked up unless you explicitly widen `@ComponentScan`'s `basePackages`.

```java
com.example.myapp
├── MyApp.java                    // @SpringBootApplication here
├── controller/OrderController.java   // scanned — sub-package of com.example.myapp
└── ...
com.example.other.SomeService.java  // NOT scanned — different package tree entirely
```

**Best Practice:** Never move the main application class out of the root package "just to organize things" — it silently breaks component scanning for anything outside its new sub-tree, a very common real-world bug when a team reorganizes packages without realizing this dependency.

**Follow-up:** *"How would you scan an additional package outside the default tree without moving your main class?"* → `@SpringBootApplication(scanBasePackages = {"com.example.myapp", "com.example.other"})`, or a separate explicit `@ComponentScan` on a `@Configuration` class.

---

### 1.2 `@EnableAutoConfiguration`

**What it does:** Triggers Spring Boot's auto-configuration mechanism — scanning `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (or the older `spring.factories` in pre-2.7 Boot) for candidate `@Configuration` classes, each typically guarded by `@Conditional*` annotations (Section 9) so they only activate when relevant.

**Real-world use case:** This is what lets `spring-boot-starter-web` automatically configure an embedded Tomcat + `DispatcherServlet` + Jackson message converters with zero manual bean wiring — you add a starter dependency, and matching auto-configuration classes activate because their `@ConditionalOnClass` conditions are satisfied.

**Best Practice:** To debug what auto-configured and what didn't, run with `--debug` (or `debug=true` in properties) — Spring Boot logs a full **Conditions Evaluation Report**.

**Follow-up:** *"How do you exclude a specific auto-configuration?"* → `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)`, or `spring.autoconfigure.exclude` in properties — commonly done to disable a default datasource/security auto-config you're replacing with custom setup.

---

### 1.3 `@ComponentScan`

**What it does:** Tells Spring which package(s) to scan for classes annotated with stereotype annotations (`@Component`, `@Service`, `@Repository`, `@Controller`, and anything meta-annotated with them) to register as beans.

```java
@Configuration
@ComponentScan(basePackages = {"com.example.core", "com.example.shared"},
                excludeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Deprecated.class))
public class AppConfig { }
```

**Follow-up:** *"Is `@ComponentScan` needed if you're using `@SpringBootApplication`?"* → No — it's already included (Section 1.1); an explicit, separate `@ComponentScan` is only needed if you want to override the default scan package(s)/filters.

---

## 2. Stereotype Annotations

### 2.1 `@Component`, `@Service`, `@Repository`, `@Controller` / `@RestController`

**What they do:** Mark a class as a Spring-managed bean, discovered via component scanning. All except `@RestController` are directly meta-annotated with `@Component` (making them functionally interchangeable for bean-registration purposes) — but each carries **semantic meaning**, and some add real extra behavior.

```java
@Component
class AuditLogger { }               // generic bean, no special role

@Service
class OrderService { }              // business/service layer — purely semantic, no extra framework behavior

@Repository
class OrderRepository { }           // persistence layer — ADDS exception translation (see below)

@Controller
class ViewController { }            // Spring MVC controller, returns view names (traditional MVC)

@RestController                     // = @Controller + @ResponseBody
class OrderRestController { }       // every method's return value is written directly to the HTTP response body (JSON/XML), not resolved as a view name
```

**🎯 The one stereotype with REAL extra behavior — `@Repository`:** It activates `PersistenceExceptionTranslationPostProcessor`, which wraps the bean so that platform-specific persistence exceptions (`SQLException`, Hibernate's `ConstraintViolationException`, etc.) are automatically translated into Spring's unified, unchecked `DataAccessException` hierarchy — decoupling your service layer's exception handling from the specific persistence technology underneath.

**`@RestController` internals:** `@ResponseBody` (which it includes) tells Spring MVC to skip view resolution and instead pass the returned object to an `HttpMessageConverter` (Jackson, by default, for JSON) to serialize it directly into the response body.

**Best Practice:** Always use the most specific stereotype available — beyond documentation value, it lets AOP pointcuts/logging aspects target a layer precisely (e.g., `@within(org.springframework.stereotype.Repository)` to apply a pointcut to every repository method).

**Follow-up:** *"If `@Service` has no extra framework behavior beyond `@Component`, why use it at all?"* → Semantic clarity for humans and tooling (IDEs, architecture-linting tools, and other developers reading the codebase) — it's a documentation/convention mechanism, and consistently applying the right stereotype is itself a best practice worth defending even without functional differences, since consistent semantics aid maintainability at scale.

---

## 3. Dependency Injection Annotations

### 3.1 `@Autowired`

**What it does:** Marks a constructor, field, or setter for Spring to inject a matching bean.

```java
@Service
class OrderService {
    private final PaymentGateway paymentGateway;

    @Autowired  // OPTIONAL here — implicit since Spring 4.3+ when there's exactly ONE constructor
    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

**🎯 The single most important, frequently-tested best-practice fact:** **Constructor injection is the Spring team's own recommended default**, over field injection (`@Autowired` directly on a field) — reasons: enables `final` fields (immutability), guarantees full initialization before the bean is usable (no window where the bean exists but a dependency is null), fails fast at context-startup time with a clear error rather than a runtime NPE deep in a request path, and makes unit testing trivial (`new OrderService(mockGateway)`, no reflection/Spring context needed).

```java
// DISCOURAGED — field injection
@Service
class OrderService {
    @Autowired
    private PaymentGateway paymentGateway;  // can't be final, hides the dependency from the constructor signature,
                                              // requires Spring/reflection to test, allows circular deps to silently exist
}
```

**Resolution when multiple beans of the same type exist:** By type first; if ambiguous, Spring falls back to matching by **field/parameter name** against bean names; if still ambiguous, throws `NoUniqueBeanDefinitionException` unless disambiguated via `@Qualifier` or `@Primary` (Section 3.2/3.3).

**`required = false`:** `@Autowired(required = false)` allows the bean to be missing without failing startup (the field/parameter stays null) — generally discouraged in favor of `Optional<T>` as the injected type, which more explicitly signals "this dependency might be absent" to code reading it later.

**Best Practices:**
- Default to constructor injection always; use field injection essentially never in production code (test classes are a common, accepted exception, though even there many teams now prefer constructor-based test setup).
- If a class needs 5+ constructor-injected dependencies, treat that as a code smell suggesting the class has too many responsibilities (Single Responsibility Principle violation) — constructor injection's visible-boilerplate nature is a deliberate feature that surfaces this, not just a style inconvenience.

**Follow-up 🎯:** *"How does Spring resolve a circular dependency with constructor injection vs field injection?"* → With **constructor injection, a circular dependency fails fast at startup** (`BeanCurrentlyInCreationException`) — Spring cannot construct bean A (which needs bean B in its constructor) before constructing bean B (which needs bean A) — forcing you to redesign. With **field/setter injection**, Spring can resolve *some* circular dependencies via early bean-reference exposure (the "third-level cache" in `DefaultSingletonBeanRegistry`) — but this is broadly considered a code smell to actually rely on, not a feature to design around; a circular dependency almost always indicates two classes should be refactored (e.g., extract a shared interface, or merge overlapping responsibilities).

---

### 3.2 `@Qualifier`

**What it does:** Disambiguates which specific bean to inject when multiple candidates of the same type exist.

```java
public interface NotificationService { void send(String msg); }

@Service("emailNotification")
class EmailNotificationService implements NotificationService { }

@Service("smsNotification")
class SmsNotificationService implements NotificationService { }

@Service
class OrderService {
    private final NotificationService notificationService;
    public OrderService(@Qualifier("emailNotification") NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}
```

**Follow-up:** *"What's the difference between using `@Qualifier` and just naming the constructor parameter to match a bean's name?"* → Without `@Qualifier`, Spring's byname fallback only kicks in if type-matching alone is ambiguous, and it relies on the parameter name matching the target bean's name **exactly** — fragile if someone renames a parameter during a refactor without realizing it silently breaks bean resolution. `@Qualifier` is explicit, refactor-safe (an IDE rename of the parameter doesn't break it), and immediately visible as an intentional wiring decision — the clearly preferred, more robust approach.

---

### 3.3 `@Primary`

**What it does:** Marks one bean among several candidates of the same type as the **default** choice when no `@Qualifier` is specified.

```java
@Service
@Primary
class DefaultPaymentGateway implements PaymentGateway { }

@Service
class FallbackPaymentGateway implements PaymentGateway { }
// Any @Autowired PaymentGateway without an explicit @Qualifier gets DefaultPaymentGateway
```

**Best Practice:** Use `@Primary` when there's a clear "usual" choice among alternatives and only occasional code needs to override it with `@Qualifier` — if usage is closer to 50/50 between implementations, explicit `@Qualifier` everywhere is clearer than relying on an implicit default.

**Follow-up:** *"What happens if two beans of the same type both have `@Primary`?"* → `NoUniqueBeanDefinitionException` at startup — `@Primary` only resolves ambiguity when there's exactly one "primary" candidate; more than one primary candidate is itself an unresolved ambiguity.

---

### 3.4 `@Bean`

**What it does:** Declares a bean explicitly inside a `@Configuration` class — used for wiring beans you don't control the source of (third-party libraries) or beans requiring custom construction logic beyond what component-scanning + constructor injection can express.

```java
@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder.setConnectTimeout(Duration.ofSeconds(5))
                      .setReadTimeout(Duration.ofSeconds(10))
                      .build();
    }

    @Bean
    @ConditionalOnMissingBean  // lets consumers override this default by defining their own bean of the same type
    public Clock clock() {
        return Clock.systemUTC();
    }
}
```

**🎯 `@Configuration` proxying — a genuinely important internals question:** By default (`proxyBeanMethods = true`), Spring generates a **CGLIB subclass proxy** of your `@Configuration` class, so that calling one `@Bean` method **from within another** `@Bean` method in the same class returns the **same singleton instance** rather than creating a new one — this is what lets you express bean-to-bean dependencies by direct Java method calls, not just method parameters.

```java
@Configuration
public class AppConfig {
    @Bean
    public ServiceA serviceA() { return new ServiceA(serviceB()); }  // calls serviceB() directly

    @Bean
    public ServiceB serviceB() { return new ServiceB(); }
    // Because of CGLIB proxying, serviceA() calling serviceB() gets the SAME singleton
    // Spring already created for the "serviceB" bean — NOT a fresh new ServiceB() each time.
}
```
Setting `@Configuration(proxyBeanMethods = false)` skips this proxying (faster startup, less CGLIB overhead — relevant for GraalVM native image compatibility too) — but then direct method-to-method calls **do** create fresh instances each time, breaking the above pattern; use `proxyBeanMethods = false` only when your `@Bean` methods don't call each other directly (pass dependencies as method parameters instead, which Spring still correctly injects regardless of proxying mode).

**Best Practice:** Prefer passing dependencies as `@Bean` method **parameters** (which Spring resolves from the context regardless of proxy mode) over calling other `@Bean` methods directly — more explicit, and compatible with `proxyBeanMethods = false` if you later want that startup-performance/native-image benefit.

**Follow-up:** *"When would you use `@Bean` instead of `@Component`?"* → When the class is third-party code you can't annotate directly (e.g., configuring a `RestTemplate`, `ObjectMapper`, or any external library's class), or when bean construction needs conditional/parameterized logic beyond a simple no-arg constructor that component scanning + `@Autowired` alone can express.

---

### 3.5 `@Value`

**What it does:** Injects a single property value (from `application.yml`/properties, environment variables, or a SpEL expression) directly into a field or constructor parameter.

```java
@Component
class PaymentConfig {
    @Value("${payment.gateway.timeout:5000}")  // ':5000' is a default if the property is missing
    private int timeoutMs;

    @Value("#{systemProperties['user.region'] ?: 'US'}")  // SpEL expression with a fallback
    private String region;
}
```

**Best Practice:** For more than a couple of related properties, prefer `@ConfigurationProperties` (Section 9) over scattering many individual `@Value` fields — type-safe, validated as a group, and far less repetitive than one `@Value` per field.

**Follow-up:** *"Why might `@Value` silently fail to inject the expected property in a unit test?"* → `@Value` resolution depends on Spring's `Environment`/`PropertySourcesPlaceholderConfigurer` being active in the loaded context — a plain unit test that doesn't load a Spring context at all (e.g., using `@ExtendWith(MockitoExtension.class)` rather than `@SpringBootTest`) won't process `@Value` at all, since there's no Spring container present to resolve it — a common source of "why is this field null in my test" confusion.

---

## 4. Bean Scope & Lifecycle Annotations

### 4.1 `@Scope`

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
class ShoppingCart { }

@Component
@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE, proxyMode = ScopedProxyMode.TARGET_CLASS)
class RequestScopedHelper { }  // needed when injecting a narrower-scoped bean into a wider-scoped (e.g., singleton) one
```

**Scopes:** `singleton` (default — one instance per container), `prototype` (new instance per injection/request), `request`/`session`/`application`/`websocket` (web-aware contexts only).

**🎯 The classic scope-mismatch bug:** Injecting a `prototype` bean into a `singleton` bean via normal `@Autowired` resolves the dependency **once**, at singleton-construction time — you get the same prototype instance forever after, defeating the point of "new instance each time." Fix: inject an `ObjectProvider<T>`/`ObjectFactory<T>` (call `.getObject()` per use for a fresh instance) or use `proxyMode` on the prototype bean's `@Scope` (injects a proxy that resolves a fresh target on every method call).

**Follow-up:** *"Is a singleton-scoped bean thread-safe by default?"* → No — Spring only guarantees one shared instance; if it holds mutable instance state, you must handle thread safety yourself. The standard convention is to keep `@Service`/`@Repository` singleton beans **stateless**.

---

### 4.2 `@PostConstruct` / `@PreDestroy`

```java
@Component
class ConnectionPoolManager {
    @PostConstruct
    public void init() { warmUpPool(); }   // runs after dependency injection completes, before the bean is used

    @PreDestroy
    public void cleanup() { drainPool(); } // runs during graceful container shutdown, before the bean is destroyed
}
```

**Note:** These are standard `jakarta.annotation` (JSR-250) annotations, not Spring-specific — preferred over implementing Spring's own `InitializingBean`/`DisposableBean` interfaces, since they don't couple your code directly to the Spring API.

**Real-world use case:** `@PostConstruct` for cache warm-up or config validation at startup; `@PreDestroy` for gracefully closing connections/flushing buffers on shutdown — important for clean pod termination in Kubernetes (`SIGTERM` handling).

**Follow-up:** *"Does `@PostConstruct` run before or after constructor injection completes?"* → After — the full bean lifecycle order is: constructor call → dependency injection (fields/setters) → `@PostConstruct` → bean ready for use. This guarantees all `@Autowired` dependencies are already set by the time `@PostConstruct` logic runs.

---

## 5. Web / Spring MVC Annotations

### 5.1 Request Mapping Family

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @GetMapping("/{id}")
    public OrderResponse getOrder(@PathVariable Long id) { ... }

    @GetMapping
    public Page<OrderResponse> listOrders(@RequestParam(required = false) String status,
                                           @PageableDefault(size = 20) Pageable pageable) { ... }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse createOrder(@Valid @RequestBody CreateOrderRequest request) { ... }

    @PutMapping("/{id}")
    public OrderResponse updateOrder(@PathVariable Long id, @Valid @RequestBody UpdateOrderRequest request) { ... }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteOrder(@PathVariable Long id) { ... }
}
```

- `@RequestMapping` — the general-purpose original annotation (`method = RequestMethod.GET` etc.); `@GetMapping`/`@PostMapping`/`@PutMapping`/`@DeleteMapping`/`@PatchMapping` are Spring 4.3+ shorthand meta-annotations for the common verb-specific cases — always prefer the specific shorthand for readability.
- Class-level `@RequestMapping("/api/v1/orders")` establishes a common path prefix for all methods in the controller.

**`@PathVariable` vs `@RequestParam` — the core distinction:**
```java
// /api/v1/orders/123          -> @PathVariable captures "123" as part of the URL path itself
// /api/v1/orders?status=PAID  -> @RequestParam captures "PAID" from the query string
```

**`@RequestBody` vs `@ResponseBody`:**
- `@RequestBody` — deserializes the incoming HTTP request body (typically JSON) into a Java object, via a registered `HttpMessageConverter` (Jackson by default).
- `@ResponseBody` — serializes the returned Java object into the HTTP response body — implicitly included on every method when the class is `@RestController`, so it's rarely written explicitly in modern code (mostly seen on individual methods within a plain `@Controller` that mixes view-returning and JSON-returning endpoints).

**Best Practices:**
- Use the HTTP-verb-specific shorthand annotations (`@GetMapping`, etc.) instead of `@RequestMapping(method = ...)`.
- Always validate `@RequestBody` inputs with `@Valid` (Section 6).
- Use `@ResponseStatus` to set the correct success status code (`201 CREATED` for POST, `204 NO_CONTENT` for DELETE) rather than defaulting to `200 OK` for everything.

**Follow-up 🎯:** *"What happens if a required `@RequestParam` is missing from the request?"* → Spring throws `MissingServletRequestParameterException`, resulting in a `400 Bad Request` by default (unless you've registered a custom `@ExceptionHandler` for it) — `@RequestParam` is **required by default**; use `@RequestParam(required = false)` or, more idiomatically, `@RequestParam Optional<String> status` to make absence explicit and safely handled in code.

---

### 5.2 `@ControllerAdvice` / `@RestControllerAdvice` and `@ExceptionHandler`

**What they do:** Centralize exception handling across (by default) **all** controllers, rather than repeating try-catch blocks in every controller method.

```java
@RestControllerAdvice   // = @ControllerAdvice + @ResponseBody
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse(ex.getMessage(), 404));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)  // thrown by @Valid failures
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest().body(new ErrorResponse(message, 400));
    }

    @ExceptionHandler(Exception.class)   // catch-all — should log and never leak internal details to the client
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unhandled exception", ex);
        return ResponseEntity.status(500).body(new ErrorResponse("Internal error", 500));
    }
}
```

**Scoping `@ControllerAdvice` to specific controllers/packages** (rather than globally):
```java
@RestControllerAdvice(basePackages = "com.example.api.admin")
public class AdminExceptionHandler { }

@RestControllerAdvice(assignableTypes = {OrderController.class})
public class OrderExceptionHandler { }
```

**Best Practice:** Order `@ExceptionHandler` methods from most-specific to least-specific exception type within the same advice class — Spring picks the most specific matching handler for a thrown exception's actual type, so a generic `Exception.class` handler acts as a safety-net catch-all, never masking more specific handlers.

**Follow-up:** *"How does an `@ExceptionHandler` inside a specific `@Controller` class interact with a global `@RestControllerAdvice` handler for the same exception type?"* → A controller-local `@ExceptionHandler` (defined directly inside the controller class itself) takes **precedence** over a global advice-class handler for exceptions thrown by that specific controller — Spring resolves the "closest"/most locally-scoped applicable handler first.

---

### 5.3 `@CrossOrigin`

```java
@RestController
@CrossOrigin(origins = "https://myfrontend.com", maxAge = 3600)
public class OrderController { }
```

**What it does:** Configures CORS (Cross-Origin Resource Sharing) headers for the annotated controller/method — allowing a frontend served from a different origin to call this API from a browser.

**Best Practice:** For anything beyond a single controller, configure CORS globally via a `WebMvcConfigurer` bean rather than annotating every controller individually — more maintainable and consistent.

---

## 6. Validation Annotations

### 6.1 `@Valid` / `@Validated` and Bean Validation constraints

```java
public record CreateOrderRequest(
    @NotNull @Positive Long customerId,
    @NotEmpty List<@Valid OrderItemRequest> items,       // @Valid on the generic type -> validates each item in the list too
    @Email String contactEmail,
    @Size(min = 1, max = 500) String notes,
    @Pattern(regexp = "^[A-Z]{2}\\d{6}$") String promoCode,
    @Min(0) @Max(100) int discountPercent
) {}

@PostMapping
public OrderResponse create(@Valid @RequestBody CreateOrderRequest request) { ... }
// If validation fails -> Spring throws MethodArgumentNotValidException -> handled via @ExceptionHandler (Section 5.2)
```

**Common constraint annotations:** `@NotNull`, `@NotEmpty` (not null AND not empty — for strings/collections), `@NotBlank` (not null, not empty, AND not just whitespace — strings only), `@Size`, `@Min`/`@Max`, `@Positive`/`@Negative`, `@Email`, `@Pattern`, `@Past`/`@Future` (for dates).

**🎯 `@Valid` vs `@Validated` — a genuinely important, frequently-tested distinction:**
- `@Valid` (`jakarta.validation`, standard Bean Validation spec) — works on method parameters/fields, triggers cascading validation, but does **NOT** support validation groups.
- `@Validated` (Spring-specific, `org.springframework.validation.annotation`) — a superset that **adds support for validation groups**, and is required (instead of `@Valid`) when you want to trigger **method-level parameter validation** on a `@Service`-layer bean (not just controller `@RequestBody`) via AOP.

```java
// Validation GROUPS — only @Validated supports this
public interface OnCreate {}
public interface OnUpdate {}

public record UserRequest(
    @Null(groups = OnCreate.class) @NotNull(groups = OnUpdate.class) Long id,  // id must be absent on create, present on update
    @NotBlank String name
) {}

@PostMapping
public void create(@Validated(OnCreate.class) @RequestBody UserRequest request) { }
@PutMapping
public void update(@Validated(OnUpdate.class) @RequestBody UserRequest request) { }
```

```java
// METHOD-LEVEL validation on a @Service bean — requires @Validated at the CLASS level
@Service
@Validated
public class OrderService {
    public void processOrder(@NotNull @Positive Long orderId) { ... }
    // calling processOrder(null) or processOrder(-1) throws ConstraintViolationException,
    // enforced via an AOP proxy Spring wraps around @Validated-annotated beans
}
```

**Best Practices:**
- Validate at the API boundary (`@RequestBody` + `@Valid`) as the first line of defense — never trust that data reaching the service layer is already valid just because it passed through a controller once, if that service method could be called from elsewhere too.
- Use `@Validated` + validation groups when the same DTO shape is legitimately reused for both create and update flows with different required-field rules — clearer than maintaining two near-duplicate DTOs.

**Follow-up:** *"Why doesn't `@Valid` alone trigger validation for method parameters on a plain `@Service` bean the way it does for `@RequestBody`?"* → `@RequestBody` validation is handled directly by Spring MVC's argument-resolution machinery, which explicitly checks for `@Valid`/`@Validated` on that specific parameter. For a general bean method (not a controller argument), triggering validation requires an **AOP proxy** wrapping the bean to intercept the method call and run validation before/around the actual invocation — that proxy is only created when the class itself is annotated `@Validated`, which is exactly why the class-level annotation is required for this case specifically.

---

## 7. Spring Data JPA / Persistence Annotations

### 7.1 Core JPA Entity Annotations

```java
@Entity
@Table(name = "orders", indexes = @Index(name = "idx_status", columnList = "status"))
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "order_status", nullable = false, length = 20)
    private String status;

    @Column(precision = 19, scale = 2)
    private BigDecimal totalAmount;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    private List<OrderItem> items = new ArrayList<>();

    @Version   // enables optimistic locking (see the SQL/PLSQL handbook for the full locking discussion)
    private Long version;

    @CreationTimestamp   // Hibernate-specific — auto-populated on insert
    private Instant createdAt;

    @UpdateTimestamp      // Hibernate-specific — auto-updated on every update
    private Instant updatedAt;

    @Enumerated(EnumType.STRING)  // ALWAYS prefer STRING over ORDINAL (default) — see gotcha below
    private OrderStatus statusEnum;

    @Transient   // excluded from persistence entirely — a computed/derived field, not a DB column
    private BigDecimal displayTotal;
}
```

**🎯 `@Enumerated(EnumType.ORDINAL)` vs `EnumType.STRING` — a classic, high-value gotcha:** `ORDINAL` (the default if `@Enumerated` is omitted or unspecified) stores the enum's **integer position** in its declaration order — if you ever reorder, insert, or remove a value in the enum declaration, **all previously stored ordinal values silently become wrong**, pointing to different enum constants than originally intended, with no error raised anywhere. `STRING` stores the enum constant's name as text — more storage, but immune to reordering issues (only broken by actually renaming a constant, which is a much rarer, more deliberate change). **Always use `EnumType.STRING`** for any enum you're not 100% certain will never be reordered — which in practice means essentially always.

```java
enum OrderStatus { PENDING, CONFIRMED, SHIPPED }  // ordinals: PENDING=0, CONFIRMED=1, SHIPPED=2
// If someone later inserts CANCELLED between PENDING and CONFIRMED:
enum OrderStatus { PENDING, CANCELLED, CONFIRMED, SHIPPED }  // now CONFIRMED=2, SHIPPED=3 — existing DB rows with ordinal 1 now mean CANCELLED instead of CONFIRMED!
```

**`@GeneratedValue` strategies:** `IDENTITY` (DB auto-increment column, simple but can't batch inserts efficiently since the ID isn't known until after insert), `SEQUENCE` (DB sequence object, supports batching/pre-allocation, generally preferred for Postgres/Oracle), `AUTO` (let the provider pick based on the DB dialect), `TABLE` (a table simulating a sequence — portable but slow, rarely used).

**Best Practices:**
- Always default associations (`@OneToMany`, `@ManyToOne`, `@OneToOne`) to `FetchType.LAZY` explicitly — JPA's spec default for `@ManyToOne`/`@OneToOne` is actually **EAGER**, a very commonly missed detail that causes accidental over-fetching/N+1 issues (full N+1 discussion is in the Java/Spring handbook).
- Use `@Enumerated(EnumType.STRING)` always.
- Use `@Version` for optimistic locking on any entity subject to concurrent updates.

**Follow-up:** *"Why is `@Transient` here different from Java's own `transient` keyword (used for Java serialization)?"* → They solve analogous problems in different serialization contexts but are **not the same annotation** — JPA's `@Transient` (`jakarta.persistence.Transient`) tells the **persistence provider** (Hibernate) to skip this field when mapping to/from the database; the Java language keyword `transient` tells the **JVM's built-in serialization mechanism** (`ObjectOutputStream`) to skip the field when serializing to a byte stream — a field could need one, both, or neither depending on context, and conflating them is a common point of confusion.

---

### 7.2 Spring Data Repository Annotations

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    List<Order> findByStatusAndCustomerId(String status, Long customerId);  // derived query — no annotation needed

    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
    List<Order> findWithItemsByStatus(@Param("status") String status);

    @Query(value = "SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '7 days'", nativeQuery = true)
    List<Order> findRecentOrders();

    @Modifying
    @Query("UPDATE Order o SET o.status = :status WHERE o.id = :id")
    int updateStatus(@Param("id") Long id, @Param("status") String status);

    @EntityGraph(attributePaths = {"items", "customer"})
    List<Order> findByStatus(String status);

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT o FROM Order o WHERE o.id = :id")
    Optional<Order> findByIdForUpdate(@Param("id") Long id);
}
```

**🎯 `@Modifying` — a mandatory, frequently-forgotten pairing:** Any `@Query` that performs `UPDATE`/`DELETE` (not a `SELECT`) **must** also be annotated `@Modifying`, or Spring Data throws an exception at execution time — `@Query` alone assumes a `SELECT`-style result-returning query by default. `@Modifying(clearAutomatically = true)` additionally clears the persistence context after execution, important because bulk `UPDATE`/`DELETE` via JPQL **bypasses the persistence context entirely** (executes directly against the DB) — any already-loaded managed entities in memory become stale/out of sync with the DB unless the context is cleared.

**`@EntityGraph`** — a declarative alternative to writing `JOIN FETCH` manually, letting you specify which associations to eagerly fetch for a specific query without changing the entity's default `FetchType` globally (directly addressing the N+1 problem covered in depth in the Java/Spring handbook).

**Best Practices:**
- Always pair `@Query` with `@Modifying` for non-SELECT queries.
- Use `@Param` to bind named parameters explicitly (matching `:name` placeholders) rather than relying on positional `?1`-style parameters, which are more fragile to reorder.
- Prefer derived query methods for simple lookups; reach for `@Query`/`@EntityGraph` only when derived-method naming becomes unwieldy or you need `JOIN FETCH`/native SQL.

**Follow-up:** *"What does `@Query`'s `nativeQuery = true` cost you compared to JPQL?"* → Loses database portability (the SQL is tied to your specific DB's dialect) and loses JPQL's entity-graph-aware translation — but gains access to DB-specific features (window functions, CTEs, dialect-specific functions) that JPQL can't express — a deliberate tradeoff, not a free upgrade.

---

## 8. Transaction Annotations

### 8.1 `@Transactional`

```java
@Service
public class OrderService {

    @Transactional(readOnly = true)   // hint for optimization — some drivers/Hibernate skip dirty-checking flush for read-only txns
    public OrderResponse getOrder(Long id) { ... }

    @Transactional(rollbackFor = Exception.class)   // by default, only UNCHECKED exceptions trigger rollback — this widens it
    public void placeOrder(OrderRequest request) throws PaymentException { ... }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAudit(String message) { ... }  // commits independently, even if the caller's transaction later rolls back

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void transferFunds(Long fromId, Long toId, BigDecimal amount) { ... }
}
```

**🎯 The single most-tested `@Transactional` internals fact — self-invocation:** `@Transactional` works via a **Spring AOP proxy** wrapping the bean. Calling a `@Transactional` method **from outside** the bean goes through the proxy (transaction behavior applies correctly). Calling it via **self-invocation** (`this.otherMethod()` from within the same class) **bypasses the proxy entirely** — it's a direct JVM method call — so the transactional advice is silently **ignored**.

```java
@Service
class OrderService {
    public void placeOrder() {
        this.saveOrder();  // BUG: self-invocation — bypasses the proxy, @Transactional on saveOrder() has NO EFFECT
    }
    @Transactional
    public void saveOrder() { ... }
}
```

**Default rollback behavior — a second frequently-missed fact:** By default, `@Transactional` only rolls back on **unchecked** exceptions (`RuntimeException`/`Error`) — a checked exception propagating out of a `@Transactional` method does **NOT** trigger rollback unless you explicitly widen it with `rollbackFor = Exception.class` (or the specific checked exception type).

**Propagation types (full table also in the Java/Spring handbook — quick reference here):** `REQUIRED` (default — join existing or create new), `REQUIRES_NEW` (always suspend and start independent), `NESTED` (savepoint within existing transaction), `MANDATORY`, `SUPPORTS`, `NOT_SUPPORTED`, `NEVER`.

**Best Practices:**
- Apply `@Transactional` at the service layer (not repository/DAO layer) to correctly scope a unit of work spanning multiple repository calls.
- Set `readOnly = true` explicitly for read-only methods.
- Never rely on the default rollback-only-on-unchecked behavior without deliberately confirming it's what you want — widen with `rollbackFor` when checked exceptions should also trigger rollback.
- Never call a `@Transactional` method via `this.` from within the same class expecting transactional behavior to apply.

**Follow-up:** *"How would you fix the self-invocation problem shown above without restructuring into two separate classes?"* → Inject a self-reference via `ApplicationContext.getBean(OrderService.class)` (functional but a bit unusual-looking), or enable `@EnableAspectJAutoProxy(exposeProxy = true)` and use `((OrderService) AopContext.currentProxy()).saveOrder()` — but the cleanest, most broadly recommended fix is simply extracting `saveOrder()` into a separate, dedicated bean, since needing to work around self-invocation is often itself a sign the method boundary should be a distinct component.

---

## 9. Configuration & Conditional Annotations

### 9.1 `@ConfigurationProperties`

```java
@ConfigurationProperties(prefix = "app.payment")
@Validated
public record PaymentProperties(
    @NotBlank String gatewayUrl,
    @Min(1) int timeoutSeconds,
    List<String> supportedCurrencies
) {}

@Configuration
@EnableConfigurationProperties(PaymentProperties.class)   // needed if PaymentProperties isn't itself a @Component
public class PaymentConfig { }
```
```yaml
app:
  payment:
    gateway-url: https://payments.example.com
    timeout-seconds: 10
    supported-currencies: [USD, EUR, GBP]
```

**Best Practice:** Prefer `@ConfigurationProperties` over many scattered `@Value` fields for any related group of config — type-safe, validated as a unit, supports relaxed binding (`gateway-url` in YAML maps to `gatewayUrl` in Java automatically), and generates IDE autocomplete metadata via the `spring-boot-configuration-processor` annotation processor.

**Follow-up:** *"What's 'relaxed binding' and why does it matter?"* → Spring Boot tolerates several naming variations between YAML/properties keys and Java field names (kebab-case, camelCase, snake_case, and environment-variable SCREAMING_SNAKE_CASE all bind to the same property) — this is exactly why a Kubernetes environment variable like `APP_PAYMENT_TIMEOUT_SECONDS` correctly overrides `app.payment.timeout-seconds` from `application.yml` without any special mapping code.

---

### 9.2 `@Conditional` Family

```java
@Configuration
public class CacheConfig {

    @Bean
    @ConditionalOnClass(RedisConnectionFactory.class)      // only if this class is on the classpath
    @ConditionalOnProperty(name = "app.cache.type", havingValue = "redis")
    public CacheManager redisCacheManager() { ... }

    @Bean
    @ConditionalOnMissingBean(CacheManager.class)          // fallback if no other CacheManager bean was defined
    public CacheManager inMemoryCacheManager() { ... }

    @Bean
    @ConditionalOnWebApplication
    public WebSpecificBean webBean() { ... }

    @Bean
    @ConditionalOnExpression("${app.feature.enabled:false} and ${app.feature.beta:false}")
    public BetaFeatureBean betaFeature() { ... }
}
```

**What they do:** These are the exact mechanism Spring Boot's own auto-configuration classes are built from (Section 1.2) — and they're equally usable in your own `@Configuration` classes to build conditional bean wiring (e.g., writing your own internal "starter"-style conditional configuration).

**Real-world use case:** A shared internal library providing default beans that consuming applications can override — `@ConditionalOnMissingBean` on the library's default bean means any application defining its own bean of that type transparently takes precedence, with zero coordination needed.

**Follow-up:** *"How do you debug why a `@Conditional`-guarded bean didn't get created?"* → Same mechanism as Section 1.2's auto-configuration debugging — run with `--debug` to get the Conditions Evaluation Report, which lists every conditional bean and whether its condition matched or not, with the specific reason.

---

## 10. Profile & Environment Annotations

### 10.1 `@Profile`

```java
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource dataSource() { return new EmbeddedDatabaseBuilder().build(); }  // H2 in-memory for local dev
}

@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource dataSource() { return buildProductionDataSource(); }
}

@Component
@Profile("!prod")   // negation — active in every profile EXCEPT prod
class DebugToolbarFilter { }
```

**Activation:** `spring.profiles.active=dev` (env var, JVM arg, or `application.yml`), or `@ActiveProfiles("dev")` in tests.

**Best Practice:** Use profiles for genuinely environment-specific bean **wiring** (different `DataSource`, different external service endpoints) — not as a general-purpose feature-flagging mechanism (a dedicated feature-flag system, or `@ConditionalOnProperty`, is usually a better fit for runtime-toggleable behavior that isn't tied to a deployment environment).

**Follow-up:** *"Can multiple profiles be active simultaneously?"* → Yes — `spring.profiles.active=dev,debug` activates both; beans conditioned on either profile become active, useful for composing orthogonal concerns (environment + a debug-tooling overlay) rather than needing one profile per every combination.

---

## 11. AOP Annotations

### 11.1 `@Aspect`, `@Before`, `@After`, `@Around`, `@Pointcut`

```java
@Aspect
@Component
public class LoggingAspect {

    @Pointcut("within(com.example.service..*)")   // reusable pointcut expression
    public void serviceLayer() {}

    @Before("serviceLayer()")
    public void logBefore(JoinPoint joinPoint) {
        log.info("Entering {}", joinPoint.getSignature());
    }

    @AfterReturning(pointcut = "serviceLayer()", returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        log.info("{} returned {}", joinPoint.getSignature(), result);
    }

    @AfterThrowing(pointcut = "serviceLayer()", throwing = "ex")
    public void logAfterThrowing(JoinPoint joinPoint, Exception ex) {
        log.error("{} threw {}", joinPoint.getSignature(), ex.getMessage());
    }

    @Around("@annotation(Timed)")   // matches any method annotated with a custom @Timed annotation
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();   // MUST call this — actually invokes the target method
        } finally {
            log.info("{} took {}ms", pjp.getSignature(), System.currentTimeMillis() - start);
        }
    }
}
```

**Advice types:** `@Before` (runs before the join point, can't affect/skip the target call), `@After` (runs after, regardless of outcome — like `finally`), `@AfterReturning` (only on successful return), `@AfterThrowing` (only on exception), `@Around` (wraps the call entirely — the most powerful, only type that can skip/modify the invocation or its result, but requires explicitly calling `pjp.proceed()`).

**🎯 Critical gotcha shared with `@Transactional`:** Since Spring AOP is proxy-based (same mechanism underlying `@Transactional`, covered in Section 8.1), **self-invocation bypasses aspects too** — calling an aspect-advised method via `this.` from within the same class skips the aspect's advice entirely, for the exact same reason.

**Best Practice:** Keep pointcut expressions centralized and named (`@Pointcut` methods) rather than repeating long expression strings across multiple advice annotations — easier to maintain and refactor.

**Follow-up:** *"What's required for `@Aspect` to actually work in a Spring Boot app?"* → `spring-boot-starter-aop` on the classpath (brings in AspectJ's annotation support + Spring AOP), and `@EnableAspectJAutoProxy` — though Spring Boot auto-configures this automatically once the AOP starter is present, so it's rarely written explicitly in Boot apps (a good "know your auto-configuration" detail to mention).

---

## 12. Async, Scheduling & Caching Annotations

### 12.1 `@Async` / `@EnableAsync`

```java
@SpringBootApplication
@EnableAsync
public class MyApp { }

@Service
public class NotificationService {
    @Async
    public CompletableFuture<Void> sendEmailAsync(String to, String body) {
        emailClient.send(to, body);
        return CompletableFuture.completedFuture(null);
    }
}
```

**🎯 The self-invocation issue AGAIN (a pattern worth recognizing across `@Transactional`/`@Aspect`/`@Async` — all are proxy-based):** Calling an `@Async` method via `this.` from within the same class runs it **synchronously**, on the calling thread — the proxy is bypassed, so the method executes normally, blocking the caller, silently defeating the entire purpose of `@Async`.

**Return type matters:** `@Async` methods should return `void`, `Future<T>`, `CompletableFuture<T>`, or `ListenableFuture<T>` (deprecated in favor of `CompletableFuture`) — returning any other type means the caller can never observe the result or any exception thrown asynchronously (a `void`-returning `@Async` method's exceptions are simply logged by an `AsyncUncaughtExceptionHandler`, never propagated back to the original caller).

**Best Practice:** Always configure a dedicated `ThreadPoolTaskExecutor` bean (`corePoolSize`, `maxPoolSize`, `queueCapacity`, and a `RejectedExecutionHandler`) rather than relying on Spring's default `SimpleAsyncTaskExecutor`, which creates a **new thread per invocation** with no pooling/bounding at all — a serious production risk under load.

```java
@Bean(name = "notificationExecutor")
public Executor notificationExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(20);
    executor.setQueueCapacity(500);
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    return executor;
}

@Async("notificationExecutor")   // reference the named executor explicitly
public void sendEmailAsync(...) { }
```

**Follow-up:** *"What happens if you forget `@EnableAsync`?"* → `@Async` annotations are silently ignored — methods run synchronously with no error or warning, a genuinely easy-to-miss configuration gap that can go unnoticed until someone questions why an "async" operation is blocking the caller.

---

### 12.2 `@Scheduled` / `@EnableScheduling`

```java
@SpringBootApplication
@EnableScheduling
public class MyApp { }

@Component
public class ReportScheduler {

    @Scheduled(cron = "0 0 2 * * *")    // every day at 2 AM
    public void generateDailyReport() { ... }

    @Scheduled(fixedRate = 60000)        // every 60 seconds, measured from the START of the previous execution
    public void pollQueue() { ... }

    @Scheduled(fixedDelay = 30000)       // 30 seconds AFTER the previous execution COMPLETES
    public void cleanupTempFiles() { ... }
}
```

**`fixedRate` vs `fixedDelay` — a frequently-tested distinction:** `fixedRate` schedules the next execution based on the **start time** of the previous one (if a task takes longer than the rate, executions can back up/overlap depending on executor configuration); `fixedDelay` waits for the specified delay **after the previous execution finishes**, guaranteeing no overlap.

**🎯 Critical production gotcha:** By default, `@Scheduled` tasks run on a **single-threaded** scheduler — if you have multiple `@Scheduled` methods, or a task that occasionally runs long, they can **block each other**. Fix: configure a dedicated `TaskScheduler` bean with a larger thread pool.

```java
@Bean
public TaskScheduler taskScheduler() {
    ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
    scheduler.setPoolSize(10);
    return scheduler;
}
```

**Follow-up:** *"How would you run scheduled tasks safely across multiple instances of a horizontally-scaled service without duplicate execution?"* → Plain `@Scheduled` has no built-in awareness of other instances — every replica would run the task independently, causing duplicate execution. Use a **distributed lock** (e.g., ShedLock, a popular library specifically for this) to ensure only one instance actually executes a given scheduled task at a time across a cluster, or externalize scheduling entirely to a dedicated job scheduler (Quartz with a shared DB-backed job store, or a cloud-native cron/job system) designed for distributed coordination.

---

### 12.3 `@Cacheable` / `@CacheEvict` / `@CachePut` / `@EnableCaching`

```java
@SpringBootApplication
@EnableCaching
public class MyApp { }

@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElseThrow();
    }

    @CachePut(value = "products", key = "#result.id")   // always executes the method, then updates the cache with the result
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }

    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }

    @Caching(evict = { @CacheEvict("products"), @CacheEvict("productSummaries") })
    public void bulkInvalidate() { }

    @CacheEvict(value = "products", allEntries = true)   // clear the entire cache region
    public void clearAllProducts() { }
}
```

**🎯 `@Cacheable` vs `@CachePut` — a commonly-confused pair:** `@Cacheable` **skips** the method body entirely if a cached value already exists for the computed key (cache-aside read pattern). `@CachePut` **always executes** the method, then unconditionally updates the cache with the result — used for methods that must always run (like an update operation) but should also refresh the cache, rather than the read-only short-circuit behavior of `@Cacheable`.

**SpEL key expressions:** `key = "#id"` refers to the method parameter named `id`; `key = "#result.id"` (only valid in `@CachePut`/`@CacheEvict`'s context, referring to the method's return value) refers to the returned object's `id` field; `condition`/`unless` attributes allow conditional caching (`@Cacheable(condition = "#id > 0", unless = "#result == null")`).

**Best Practices:**
- Use `@CacheEvict` on the corresponding update/delete operations for any `@Cacheable` data — forgetting this is one of the most common cache-consistency bugs (serving stale data indefinitely after an update).
- Be deliberate about cache key design when methods have multiple parameters — an ambiguous/incomplete key can cause cache collisions between logically distinct calls.
- Remember that the default `ConcurrentMapCacheManager` (Spring's simplest built-in cache) is **in-process only** — not shared across multiple instances of a horizontally-scaled service; use Redis/a distributed cache manager for multi-instance consistency.

**Follow-up:** *"Does `@Cacheable` work if you call the annotated method via `this.` from within the same class?"* → No — same proxy-based self-invocation limitation as `@Transactional`/`@Async`/`@Aspect` (a genuinely recurring theme across essentially all Spring's declarative, annotation-driven cross-cutting behaviors) — self-invocation bypasses the caching proxy, so the method always executes fully, never hitting the cache.

---

## 13. Spring Security Annotations

### 13.1 `@EnableWebSecurity` and Method Security (`@PreAuthorize`, `@PostAuthorize`, `@Secured`)

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity   // needed for @PreAuthorize/@PostAuthorize/@Secured to work at all
public class SecurityConfig {
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
        return http.build();
    }
}

@Service
public class OrderService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(Long id) { ... }

    @PreAuthorize("#customerId == authentication.principal.id or hasRole('ADMIN')")
    public Order getOrder(Long customerId) { ... }   // SpEL referencing method parameters + the current Authentication

    @PostAuthorize("returnObject.customerId == authentication.principal.id")
    public Order getOrderById(Long orderId) { ... }   // checked AFTER the method runs, against the RETURNED object

    @Secured("ROLE_ADMIN")   // older, simpler alternative — no SpEL support, just role-name checking
    public void archiveOrder(Long id) { ... }
}
```

**🎯 `@PreAuthorize` vs `@PostAuthorize` — the key distinction:** `@PreAuthorize` evaluates its expression **before** the method executes (can prevent the call entirely). `@PostAuthorize` evaluates **after** the method has already run, checking the **returned value** — necessary when the authorization decision genuinely depends on data only available in the result (e.g., "can this user see this specific order" where the order's ownership is only known after fetching it) — but note the method's side effects (if any) have already happened by the time `@PostAuthorize` denies access, so it's unsuitable for authorizing mutating operations on its own.

**`@Secured` vs `@PreAuthorize`:** `@Secured` is older, simpler, role-name-only (no SpEL expressions, no method-parameter references) — `@PreAuthorize`/`@PostAuthorize` (from Spring Security's expression-based method security) are far more flexible and are the modern standard choice.

**Best Practices:**
- Use `@PreAuthorize` for the vast majority of method-security needs — most expressive, most widely used in current code.
- Combine URL-based security (`SecurityFilterChain`) for coarse-grained endpoint access control with method-level security (`@PreAuthorize`) for fine-grained, data-dependent authorization decisions — defense in depth, not either/or.
- Never forget `@EnableMethodSecurity` — without it, `@PreAuthorize`/`@PostAuthorize`/`@Secured` annotations are silently non-functional (present in code, doing nothing) — a genuinely dangerous, easy-to-miss security gap.

**Follow-up 🎯:** *"If `@PreAuthorize` is proxy-based like `@Transactional`, does the self-invocation bypass apply here too?"* → Yes — exactly the same underlying AOP-proxy mechanism, so calling a `@PreAuthorize`-protected method via `this.` from within the same class bypasses the authorization check entirely — a genuinely serious security bug if not understood, since it can silently create an authorization bypass path that looks secure in code review (the annotation is right there) but doesn't actually enforce anything for that particular call path.

---

## 14. Testing Annotations

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderIntegrationTest {
    @Autowired TestRestTemplate restTemplate;
    @MockitoBean PaymentClient paymentClient;   // replaces the real bean with a mock IN the Spring context (Spring Boot 3.4+ name; @MockBean in older versions)
}

@WebMvcTest(OrderController.class)   // loads ONLY the web layer — controllers, @ControllerAdvice, Jackson config
class OrderControllerTest {
    @Autowired MockMvc mockMvc;
    @MockitoBean OrderService orderService;   // service layer mocked, since @WebMvcTest doesn't load it

    @Test
    void shouldReturn404WhenOrderMissing() throws Exception {
        when(orderService.getOrder(1L)).thenThrow(new ResourceNotFoundException("not found"));
        mockMvc.perform(get("/api/v1/orders/1"))
               .andExpect(status().isNotFound());
    }
}

@DataJpaTest   // loads ONLY JPA-related config, uses an embedded DB by default, rolls back after each test
class OrderRepositoryTest {
    @Autowired OrderRepository orderRepository;
    @Autowired TestEntityManager entityManager;   // convenience helper for setting up test data directly
}
```

**The three Boot test "slices" — a very commonly-asked comparison:**

| Annotation | Loads | Use for |
|---|---|---|
| `@SpringBootTest` | Full application context (or a substantial slice, depending on config) | End-to-end integration tests |
| `@WebMvcTest` | Only the web layer (controllers, filters, `@ControllerAdvice`) | Fast, focused controller tests — mock the service layer |
| `@DataJpaTest` | Only JPA/repository-related config | Fast, focused repository/query tests |

**Best Practice:** Prefer test slices (`@WebMvcTest`, `@DataJpaTest`) over `@SpringBootTest` wherever the test's actual scope allows it — dramatically faster test suites, since a full `@SpringBootTest` context load is expensive and Spring's context-caching benefits are undermined by unnecessary `@MockBean`/profile variation across test classes (full discussion in the Java/Spring handbook's testing section).

**Follow-up:** *"Why does adding a `@MockitoBean`/`@MockBean` sometimes noticeably slow down an entire test suite?"* → Spring caches the `ApplicationContext` across test classes with an **identical** configuration signature — introducing a different `@MockBean`/`@MockitoBean` combination in one test class creates a distinct context "key," forcing Spring to build an entirely new context for it rather than reusing a cached one — widespread inconsistent mock usage across many test classes can multiply the number of contexts Spring must build and cache, directly slowing down CI.

---

## 15. Lombok Annotations (Commonly Paired, Worth Knowing)

*(Not part of Spring itself, but appears in the overwhelming majority of real-world Spring Boot codebases — interviewers frequently expect familiarity.)*

```java
@Data                    // = @Getter + @Setter + @ToString + @EqualsAndHashCode + @RequiredArgsConstructor
@NoArgsConstructor        // required by JPA — entities need a no-arg constructor for the persistence provider
@AllArgsConstructor
@Builder                  // generates a fluent builder pattern
@Entity
public class Order {
    @Id private Long id;
    private String status;
}

@Slf4j   // injects a `log` field (SLF4J Logger) — avoids manually writing `private static final Logger log = ...`
@Service
@RequiredArgsConstructor   // generates a constructor for all FINAL fields — the standard modern pattern for constructor injection with minimal boilerplate
public class OrderService {
    private final OrderRepository orderRepository;  // constructor auto-generated by Lombok, @Autowired implicit (single constructor)
}
```

**🎯 Why `@Data` is generally discouraged specifically on JPA `@Entity` classes:** `@Data`'s generated `equals()`/`hashCode()` (via `@EqualsAndHashCode`) includes **all** fields by default — for a JPA entity, this often includes lazily-loaded association fields, which can trigger unwanted lazy-loading during an `equals()`/`hashCode()` call (e.g., inside a `HashSet`), and including a mutable, auto-generated `@Id` field in `equals()`/`hashCode()` causes exactly the "hash changes after insertion" bug covered in the Collections handbook (Q "mutable HashMap key" gotcha) — since the ID is null before persistence and populated after, an entity's hash code changes the moment it's saved, corrupting any `HashSet`/`HashMap` it was already placed into beforehand. Best practice: use `@Getter`/`@Setter` individually on entities, and write a deliberate, minimal `equals()`/`hashCode()` based on a stable business key (or just object identity) instead of `@Data`'s all-fields default.

**Best Practices:**
- `@RequiredArgsConstructor` + `final` fields is the standard modern idiom for constructor injection with minimal boilerplate — widely used alongside Spring's own constructor-injection best practice (Section 3.1).
- Avoid `@Data` on JPA entities specifically; prefer it freely on plain DTOs/value objects with no persistence concerns.
- `@Builder` pairs well with immutable DTOs/request-response objects for readable object construction, especially with many optional fields.

**Follow-up:** *"Why is `@RequiredArgsConstructor` particularly well-suited to Spring's constructor-injection recommendation?"* → It generates a constructor for exactly the `final` (and any explicitly `@NonNull`) fields — which is precisely the shape Spring's implicit single-constructor `@Autowired` inference expects — giving you fully boilerplate-free, immutable, constructor-injected beans without writing the constructor by hand at all, while still getting all the benefits (fail-fast startup, testability, immutability) discussed in Section 3.1.

---

## 16. Annotation Comparison Tables

### `@Component` vs `@Bean`
| | `@Component` (+ stereotypes) | `@Bean` |
|---|---|---|
| Applied to | A class you own/control | A method (typically inside `@Configuration`) |
| Discovery | Component scanning | Explicit registration |
| Use for | Your own application classes | Third-party classes, or beans needing custom construction logic |

### `@Autowired` vs `@Resource` vs `@Inject`
| | `@Autowired` (Spring) | `@Resource` (JSR-250) | `@Inject` (JSR-330) |
|---|---|---|---|
| Resolution order | By type first, then name | By **name** first, then type | By type (like `@Autowired`, no Spring-specific extras like `required`) |
| Spring-specific features (`required`, etc.) | Yes | No | No |
| Portability outside Spring | No | Yes (standard Java annotation) | Yes (standard Java annotation) |

### `@Controller` vs `@RestController`
| | `@Controller` | `@RestController` |
|---|---|---|
| Default return handling | Resolved as a view name (template engine) | Written directly to response body via `HttpMessageConverter` |
| Needs `@ResponseBody` per method for JSON? | Yes | No — implicit |

### `@Cacheable` vs `@CachePut` vs `@CacheEvict`
| | `@Cacheable` | `@CachePut` | `@CacheEvict` |
|---|---|---|---|
| Method always executes? | No — skipped on cache hit | Yes — always | Yes — always |
| Effect on cache | Populates on miss | Always overwrites | Removes entry |

### `fixedRate` vs `fixedDelay` (`@Scheduled`)
| | `fixedRate` | `fixedDelay` |
|---|---|---|
| Timing reference | From the **start** of the previous execution | From the **end** (completion) of the previous execution |
| Can overlap? | Potentially, if execution exceeds the rate | Never |

---

## 17. Rapid-Fire Tricky Q&A

**Q: Does `@Transactional`, `@Async`, `@Cacheable`, and `@PreAuthorize` all share the exact same "self-invocation breaks it" limitation — why?**
A: Yes — all four (and `@Aspect`-based custom advice) are implemented via **Spring AOP proxies** wrapping the bean. Any of these annotations only take effect when the method is called **through the proxy** (i.e., from outside the bean, or via an injected reference to the bean). Calling the annotated method via `this.` from within the same class is a direct JVM call that never touches the proxy, so none of the associated cross-cutting behavior (transaction management, async dispatch, caching, authorization) applies — recognizing this as **one single underlying mechanism** behind four seemingly different annotations is a strong signal of real Spring internals understanding.

**Q: If a class has both `@Service` and a custom-defined `@Component`-meta-annotated annotation, does Spring register it as two separate beans?**
A: No — a class is registered as exactly one bean regardless of how many stereotype (or stereotype-composed custom) annotations are stacked on it; the annotations combine to describe that single bean's characteristics, they don't multiply bean registrations.

**Q: Why does `@Value("${some.property}")` throw an error at startup if the property is genuinely missing, but `@ConfigurationProperties` binding for the same missing property might not (depending on setup)?**
A: `@Value` with no default value and no matching property throws `IllegalArgumentException`/`PropertyReferenceException` at bean-creation time by default, since Spring can't resolve the placeholder. `@ConfigurationProperties` binds available properties into whatever fields it can and simply leaves unmatched fields at their default (often `null`) **unless** you've added `@Validated` + Bean Validation constraints (`@NotNull`, etc.) on those fields — in which case it fails at startup too, but only if validation is explicitly wired in; without it, a missing property silently results in a null field rather than a startup failure, a genuinely important distinction for catching configuration errors early.

**Q: Can `@Bean` methods have parameters, and how does Spring resolve them?**
A: Yes — parameters of a `@Bean` method are resolved exactly like `@Autowired` constructor parameters (by type, then by name if ambiguous, `@Qualifier` for explicit disambiguation) — this is the recommended way to express bean-to-bean dependencies within `@Configuration` classes (Section 3.4), preferable to calling another `@Bean` method directly when you want compatibility with `proxyBeanMethods = false`.

**Q: Why might `@Transactional(propagation = Propagation.REQUIRES_NEW)` used incorrectly cause a deadlock?**
A: If the outer transaction holds a row-level lock and the inner `REQUIRES_NEW` transaction (a genuinely separate, concurrently-running transaction from the database's perspective) attempts to acquire a lock on the **same row** before the outer transaction commits/releases its lock, the inner transaction can block waiting on the outer one — but if they're running on the **same thread** and the outer transaction is itself waiting for the inner one to complete before it can proceed (e.g., the inner call is synchronous, awaited directly in the outer method's flow), you get a classic self-inflicted deadlock: the outer transaction can't finish until the inner one does, but the inner one is blocked waiting for a lock the outer transaction holds and won't release until it finishes. `REQUIRES_NEW` should be used deliberately and sparingly, with a clear understanding that it's genuinely a separate, concurrently-committing transaction, not just a "nested" scope.

**Q: Does `@Order` (Spring's ordering annotation) affect the order `@Component`-scanned beans are instantiated in, generally?**
A: No — `@Order` (and `Ordered`/`PriorityOrdered`) primarily affects ordering in specific contexts where Spring explicitly collects and orders a **list of same-type beans** (e.g., multiple `HandlerInterceptor`s, multiple `AsyncUncaughtExceptionHandler`s, multiple `@Aspect` advice applying to the same join point) — it does NOT generally control the arbitrary instantiation order of unrelated singleton beans during context startup, a common misconception.

---

## Quick Self-Check Before Your Interview

- [ ] Can you name every annotation `@SpringBootApplication` is composed of, and explain what each contributes?
- [ ] Can you explain the self-invocation proxy limitation and name all 4-5 annotations it affects?
- [ ] Can you explain exactly why `@Repository` has real extra behavior while `@Service` doesn't?
- [ ] Can you explain `@Valid` vs `@Validated`, including why method-level validation needs the class-level annotation?
- [ ] Can you explain `@Modifying`'s requirement and why forgetting it causes a runtime failure?
- [ ] Can you explain `EnumType.ORDINAL` vs `STRING` and construct the exact failure scenario for ORDINAL?
- [ ] Can you explain `@Cacheable` vs `@CachePut` precisely, including when the method body actually executes?
- [ ] Can you explain why `@EnableMethodSecurity` being missing is a silent, dangerous security gap rather than an obvious error?
- [ ] Do you have a real project story involving an annotation-related bug you diagnosed (self-invocation, missing `@Modifying`, ORDINAL enum reordering, or similar)?
