# Java + Spring Boot Interview Handbook (4+ YOE)
## Part 2: Multithreading & Concurrency, Functional Programming, Stream API, Java 8/11/17/21 Features

---

## 6. Multithreading & Concurrency

### Q6.1 — Explain Thread lifecycle and the difference between `Runnable`, `Thread`, `Callable`, and `ExecutorService`.

**Answer:**

**Thread states (`Thread.State` enum):**
```
NEW → RUNNABLE → (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED
```
- `NEW`: created, not started.
- `RUNNABLE`: executing or ready to run (JVM doesn't distinguish "ready" vs "running" — that's OS scheduler's job).
- `BLOCKED`: waiting to acquire a monitor lock (`synchronized`).
- `WAITING`: waiting indefinitely (`Object.wait()`, `Thread.join()`, `LockSupport.park()`) for another thread's signal.
- `TIMED_WAITING`: `sleep(ms)`, `wait(ms)`, `join(ms)`.
- `TERMINATED`: run() completed or exited via exception.

**`Runnable` vs `Callable`:**

| | `Runnable` | `Callable<V>` |
|---|---|---|
| Method | `void run()` | `V call() throws Exception` |
| Return value | No | Yes |
| Checked exceptions | Cannot throw | Can throw |
| Used with | `Thread`, `ExecutorService.execute()` | `ExecutorService.submit()` → returns `Future<V>` |

**Why prefer `ExecutorService` over manually creating `Thread`s?**
- Manual thread creation is expensive (OS thread per object, no reuse) and unmanaged (no backpressure, no lifecycle control).
- `ExecutorService` (Thread Pool) reuses a bounded set of worker threads, queues excess tasks, and provides lifecycle (`shutdown()`, `awaitTermination()`), monitoring, and `Future`-based result handling.

```java
ExecutorService executor = Executors.newFixedThreadPool(10);
Future<Integer> future = executor.submit(() -> computeExpensive());
Integer result = future.get(5, TimeUnit.SECONDS); // blocks with timeout
executor.shutdown();
```

**Common `Executors` factory methods and their production pitfalls:**

| Factory | Backing pool | Pitfall |
|---|---|---|
| `newFixedThreadPool(n)` | Fixed threads, **unbounded `LinkedBlockingQueue`** | Under heavy load, tasks queue indefinitely → OOM risk; no backpressure |
| `newCachedThreadPool()` | Unbounded threads, `SynchronousQueue` | Can spawn unlimited threads under load → thread exhaustion |
| `newSingleThreadExecutor()` | 1 thread, unbounded queue | Same queue risk, serializes everything |
| `newScheduledThreadPool(n)` | For delayed/periodic tasks | — |

**Best Practice (why senior engineers avoid `Executors.*` factory methods in production):** Use `ThreadPoolExecutor` directly with a **bounded queue** and explicit `RejectedExecutionHandler`, so the system fails predictably (backpressure) instead of OOM-ing under load:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10, 20,                          // core, max pool size
    60L, TimeUnit.SECONDS,           // idle thread keep-alive
    new ArrayBlockingQueue<>(500),   // bounded queue
    new ThreadPoolExecutor.CallerRunsPolicy() // backpressure: caller thread runs task, slowing producer
);
```

**Real-world (Spring Boot):** `@Async` methods use a `ThreadPoolTaskExecutor` bean — always configure `corePoolSize`, `maxPoolSize`, `queueCapacity` explicitly in production rather than relying on defaults, and set a `RejectedExecutionHandler` to avoid silently dropping tasks.

**Follow-up:** *"What's the difference between `shutdown()` and `shutdownNow()`?"* → `shutdown()` stops accepting new tasks but lets queued/running tasks finish; `shutdownNow()` attempts to stop all actively executing tasks (via interrupt) and returns the list of tasks that never started.

---

### Q6.2 🎯 — Explain `synchronized`, intrinsic locks, and the difference vs `ReentrantLock`.

**Answer:** `synchronized` uses the object's **intrinsic monitor lock**. Only one thread can hold the monitor for a given object at a time — other threads attempting to enter a `synchronized` block on the same object move to `BLOCKED` state.

```java
public synchronized void increment() { count++; }         // locks 'this'
public static synchronized void staticMethod() { }        // locks the Class object
synchronized(lockObject) { /* critical section */ }        // locks explicit object
```

**Internals:** Every Java object has an associated monitor. Under the hood, JVM uses **biased locking → thin (lightweight, CAS-based) locking → fat (OS-level mutex) locking** escalation for performance (biased locking was deprecated/removed as default in Java 15 since modern workloads made its benefit marginal and it added complexity). Uncontended locks are cheap (CAS on object header); contended locks fall back to OS mutex (expensive context switches).

**`synchronized` vs `ReentrantLock`:**

| Feature | `synchronized` | `ReentrantLock` (`java.util.concurrent.locks`) |
|---|---|---|
| Acquisition | Implicit, block-structured | Explicit `lock()`/`unlock()` |
| Fairness | No fairness guarantee | Optional fair mode (`new ReentrantLock(true)`) — FIFO ordering, avoids starvation |
| Interruptible acquisition | No | Yes (`lockInterruptibly()`) |
| Try-lock with timeout | No | Yes (`tryLock(timeout)`) — avoids deadlock by backing off |
| Condition variables | One implicit (`wait`/`notify`/`notifyAll`) | Multiple `Condition` objects per lock (`newCondition()`) — finer-grained signaling |
| Must unlock manually | No (auto-released on block exit) | **Yes — must unlock in `finally`**, or lock leaks forever |

```java
private final ReentrantLock lock = new ReentrantLock();
public void update() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock(); // MANDATORY — common interview trap if forgotten
    }
}
```

**When to prefer `ReentrantLock`:** Need tryLock/timeout to avoid deadlock, need fairness, need multiple wait-conditions, or need to acquire/release lock across different methods (non-block-structured locking).

**Best Practice:** Prefer higher-level constructs (`java.util.concurrent` utilities, `AtomicX`, `ConcurrentHashMap`) over manual locking wherever possible — locking is error-prone (deadlocks, forgetting to unlock, lock ordering issues).

**Common Mistakes:**
- Forgetting `unlock()` in `finally` with `ReentrantLock` → permanent deadlock for future acquirers.
- Locking on mutable/changing objects (e.g., `Integer` autoboxed values, or `this` in a subclass with unrelated external synchronization) — leads to locks that don't actually protect the intended resource.
- Synchronizing on a `String` literal — string pool means unrelated code sharing that literal accidentally shares your lock.

**Follow-up:** *"What's a deadlock and how do you avoid it?"* → Circular wait where Thread A holds Lock1 waiting for Lock2, Thread B holds Lock2 waiting for Lock1. Avoid via: consistent lock ordering across the codebase, using `tryLock` with timeout, minimizing lock scope, or avoiding nested locks altogether.

---

### Q6.3 — `volatile` keyword — what does it actually guarantee (and NOT guarantee)?

**Answer:** `volatile` guarantees:
1. **Visibility** — writes to a `volatile` variable by one thread are immediately visible to other threads (reads always go to main memory, not a CPU-cache-local/thread-local copy).
2. **Ordering** — prevents instruction reordering around the volatile variable (establishes a **happens-before** relationship per the Java Memory Model — JSR-133).

**`volatile` does NOT guarantee:**
- **Atomicity** for compound operations. `count++` on a `volatile int count` is **NOT thread-safe** — it's read-modify-write (3 separate operations), and a race condition can still lose updates.

```java
private volatile boolean running = true; // correct use: simple flag, single writer/many readers
public void stop() { running = false; }
public void run() { while (running) { /* work */ } } // guaranteed to see the update

private volatile int counter = 0;
public void increment() { counter++; } // WRONG — not atomic despite volatile; use AtomicInteger
```

**Best use case:** Boolean flags for stopping threads, double-checked locking singleton pattern (`volatile` is required there to prevent reordering that could expose a partially-constructed object):

```java
class Singleton {
    private static volatile Singleton instance;
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton(); // without volatile, this can be reordered,
                }                                // exposing a non-null but uninitialized reference to another thread
            }
        }
        return instance;
    }
}
```

**Follow-up:** *"When would you use `AtomicInteger` over `synchronized` for a counter?"* → `AtomicInteger` uses CAS (compare-and-swap, lock-free, hardware `cmpxchg` instruction) — much cheaper than lock acquisition under moderate contention, avoids context-switch overhead, and is non-blocking (no thread ever parks). Under *very* high contention, `LongAdder`/`DoubleAdder` (Java 8+) outperform `AtomicLong` by striping the counter across multiple cells to reduce CAS contention.

---

### Q6.4 — Explain the Java Memory Model's `happens-before` relationship.

**Answer:** JMM defines rules for when writes by one thread are guaranteed visible to reads by another, since without synchronization, compilers/CPUs are free to reorder instructions and cache values locally for performance.

**Key happens-before rules:**
1. **Program order rule:** within a single thread, each action happens-before every subsequent action in that thread (from that thread's own perspective).
2. **Monitor lock rule:** an unlock on a monitor happens-before every subsequent lock on that same monitor (by any thread).
3. **Volatile variable rule:** a write to a volatile field happens-before every subsequent read of that field.
4. **Thread start rule:** `Thread.start()` happens-before any action in the started thread.
5. **Thread termination rule:** all actions in a thread happen-before another thread successfully returns from `Thread.join()` on it.
6. **Transitivity:** if A happens-before B and B happens-before C, then A happens-before C.

**Why this matters practically:** Without a happens-before edge, thread A's write to a plain (non-volatile, non-synchronized) field might never become visible to thread B, or might appear in a different order than written — leading to subtle, hard-to-reproduce concurrency bugs.

**Follow-up:** *"Is a `final` field's value guaranteed visible to other threads without synchronization?"* → Yes, **if the object is properly constructed** (the `this` reference doesn't escape during construction) — JMM guarantees `final` fields are visible correctly-initialized to any thread that sees the reference, after the constructor completes.

---

### Q6.5 — Explain `CompletableFuture` and how it improves on `Future`.

**Answer:** `Future` (Java 5) only supports blocking `get()` — no way to compose, chain, or react to completion without blocking a thread. `CompletableFuture` (Java 8) supports **non-blocking, composable async pipelines**.

```java
CompletableFuture<UserDto> future = CompletableFuture
    .supplyAsync(() -> userRepository.findById(id), executor)
    .thenApply(user -> mapToDto(user))
    .thenCombine(fetchOrdersAsync(id), (user, orders) -> enrich(user, orders))
    .exceptionally(ex -> { log.error("failed", ex); return UserDto.empty(); });

// Fan-out / fan-in pattern — parallel calls to multiple downstream services
CompletableFuture<Void> all = CompletableFuture.allOf(call1(), call2(), call3());
```

**Key methods:**
- `thenApply` — transform result (sync, like `map`).
- `thenCompose` — chain another async op that itself returns a `CompletableFuture` (like `flatMap`) — avoids nested futures.
- `thenCombine` — combine results of two independent futures.
- `allOf`/`anyOf` — wait for all/any of multiple futures.
- `exceptionally`/`handle`/`whenComplete` — error handling without blocking.
- `*Async` variants (`thenApplyAsync`) — run the continuation on a separate thread pool (important: without specifying an executor, defaults to `ForkJoinPool.commonPool()`, which is shared JVM-wide — a common production pitfall when blocking calls starve the common pool).

**Real-world:** Microservice aggregating data from 3 downstream REST APIs (user-service, order-service, inventory-service) — calling them in parallel via `CompletableFuture` and combining results cuts total latency from sum-of-calls to max-of-calls.

**Best Practices:**
- Always supply a dedicated `Executor` for I/O-bound async work — don't rely on the shared `ForkJoinPool.commonPool()`, especially if you also do CPU-bound parallel streams (they share the same pool by default!).
- Always handle exceptions (`exceptionally`/`handle`) — unhandled exceptions in a `CompletableFuture` chain are silently swallowed unless you call `get()`/`join()` or attach a handler.

**Common Mistake:** Calling `.get()` immediately after `supplyAsync()` — defeats the purpose (blocks the calling thread, same as a plain synchronous call).

**Follow-up:** *"How does this relate to Reactive Programming / WebFlux?"* → `CompletableFuture` handles a single async value; `Mono`/`Flux` (Project Reactor, used in Spring WebFlux) handle 0..1 / 0..N async values with backpressure support — needed for streaming/reactive pipelines, not just one-shot async composition.

---

### Q6.6 — `ThreadLocal` — use cases, internal working, and the classic memory leak with thread pools.

**Answer:** `ThreadLocal<T>` provides per-thread isolated variable storage — each thread accessing a `ThreadLocal` sees its own independently initialized copy.

**Internals:** Each `Thread` object has a `ThreadLocalMap` field (a specialized hash map keyed by `ThreadLocal` instance, using **weak references** to the `ThreadLocal` key itself, but **strong references** to the value).

**Real-world use case:** Storing per-request context (e.g., `MDC` in logging frameworks for correlation/trace IDs, current authenticated user, `RequestContextHolder` in Spring MVC) without passing it through every method signature.

```java
public class RequestContext {
    private static final ThreadLocal<String> correlationId = new ThreadLocal<>();
    public static void set(String id) { correlationId.set(id); }
    public static String get() { return correlationId.get(); }
    public static void clear() { correlationId.remove(); } // CRITICAL in pooled-thread environments
}
```

**The classic leak:** In thread-pooled environments (Tomcat, `ExecutorService`), threads are **reused** across requests/tasks. If you `set()` a `ThreadLocal` value but never call `.remove()`, the value (and its class, and anything it references) stays attached to the pooled thread **forever**, causing a slow memory leak and potentially **leaking data between unrelated requests** (a security concern too — e.g., leaking one user's auth context into another user's request on the same reused thread).

**Best Practice:** Always `.remove()` in a `finally` block (Spring's `OncePerRequestFilter` pattern, or an interceptor's `afterCompletion`) at the end of a request lifecycle.

**Follow-up:** *"Why weak reference to the ThreadLocal key but not the value?"* → So that if the `ThreadLocal` variable itself is no longer referenced anywhere else, its map entry's *key* can be GC'd — but the corresponding *value* still needs an explicit `remove()` to be cleaned, since it's strongly referenced from the (now stale) map entry until the next `ThreadLocalMap` cleanup pass or an explicit remove.

---

### Q6.7 — Producer-Consumer pattern: `wait()/notify()` vs `BlockingQueue`.

**Answer:** Classic approach using intrinsic locks:
```java
class SharedBuffer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 10;
    synchronized void produce(int item) throws InterruptedException {
        while (queue.size() == capacity) wait();      // always use while, not if (guard against spurious wakeups)
        queue.add(item);
        notifyAll();                                    // prefer notifyAll over notify to avoid missed signals
    }
    synchronized int consume() throws InterruptedException {
        while (queue.isEmpty()) wait();
        int item = queue.poll();
        notifyAll();
        return item;
    }
}
```

**Modern approach — `BlockingQueue`** (preferred in real code, `wait/notify` is mostly interview theory now):
```java
BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);
// Producer: queue.put(item);   // blocks if full
// Consumer: int item = queue.take(); // blocks if empty
```

**Why `while` not `if` around `wait()`?** Spurious wakeups can occur (JVM/OS may wake a waiting thread without a corresponding `notify()`), and with `notifyAll()`, multiple threads wake up but only one should proceed — each must re-check the condition.

**Follow-up:** *"Difference between `notify()` and `notifyAll()`?"* → `notify()` wakes exactly one arbitrary waiting thread (risk: could wake the "wrong" thread type in mixed producer/consumer waiting sets, causing a missed-signal deadlock); `notifyAll()` wakes all, each rechecks its condition — safer default, slightly less efficient.

---

### Q6.8 — Fork/Join Framework and Parallel Streams.

**Answer:** `ForkJoinPool` (Java 7) implements **work-stealing**: idle worker threads "steal" tasks from busy threads' queues, maximizing CPU utilization for divide-and-conquer style parallel computation.

```java
class SumTask extends RecursiveTask<Long> {
    private final int[] arr; private final int start, end;
    protected Long compute() {
        if (end - start <= THRESHOLD) { /* sum sequentially */ }
        int mid = (start + end) / 2;
        SumTask left = new SumTask(arr, start, mid);
        SumTask right = new SumTask(arr, mid, end);
        left.fork();                       // async
        long rightResult = right.compute(); // sync in current thread
        return left.join() + rightResult;
    }
}
```

**Parallel Streams** (`list.parallelStream()`) are built on `ForkJoinPool.commonPool()` internally — great for **CPU-bound, stateless, non-blocking** operations on large in-memory datasets. **Bad fit** for I/O-bound work (blocking calls starve the shared common pool, affecting unrelated parallel streams / `CompletableFuture` async tasks elsewhere in the JVM) — use a custom `ForkJoinPool` or dedicated `ExecutorService` for I/O instead.

**Best Practice:** Benchmark before reaching for `parallelStream()` — for small collections, the overhead of splitting/merging/thread coordination often outweighs any gain; sequential streams are frequently faster for typical enterprise dataset sizes.

**Follow-up:** *"Why should you avoid blocking I/O in parallel streams?"* → All parallel streams share the single JVM-wide `ForkJoinPool.commonPool()` by default; a long blocking call in one parallel stream can starve threads needed by unrelated concurrent code (other parallel streams, or `CompletableFuture.*Async` calls without a custom executor).

---

## 7. Functional Programming, Lambdas & Stream API

### Q7.1 — Lambda expressions — what are they compiled to, and what's a Functional Interface?

**Answer:** A **functional interface** has exactly one abstract method (SAM — Single Abstract Method), optionally annotated `@FunctionalInterface` (compiler-enforced, not mandatory but best practice). Lambdas are essentially **anonymous implementations** of a functional interface's abstract method — but they are **not** compiled to anonymous inner classes.

**Internals:** Lambdas are compiled using `invokedynamic` bytecode instruction + `LambdaMetafactory`, which generates the implementation class **at runtime (first invocation)**, not at compile time like anonymous classes. This avoids the class-file bloat and startup cost of generating one `.class` file per lambda.

**Common built-in functional interfaces (`java.util.function`):**

| Interface | Method | Use |
|---|---|---|
| `Function<T,R>` | `R apply(T t)` | Transform T → R |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | Two inputs → output |
| `Predicate<T>` | `boolean test(T t)` | Boolean condition |
| `Consumer<T>` | `void accept(T t)` | Side-effect, no return |
| `Supplier<T>` | `T get()` | No input, produces value |
| `UnaryOperator<T>` | `T apply(T t)` | `Function<T,T>` |
| `BinaryOperator<T>` | `T apply(T t1, T t2)` | `BiFunction<T,T,T>` — used in `reduce()` |

**Lambda vs Anonymous class — key differences:**
1. `this` inside a lambda refers to the **enclosing class** instance; inside an anonymous class, `this` refers to the anonymous class instance itself.
2. Lambdas can't have their own state (no instance fields) — anonymous classes can.
3. Lambdas are lighter weight (no separate `.class` file generated at compile time for each; generated dynamically).
4. Lambdas can only implement functional interfaces (1 abstract method); anonymous classes can extend classes or implement interfaces with multiple methods.

**Effectively final requirement:** Variables captured by a lambda from the enclosing scope must be **final or effectively final** (never reassigned after initialization) — because the lambda may execute later/on another thread, and Java captures by **value** (for local variables), so it prevents the illusion of a live mutable reference that could cause race conditions/confusion.

**Follow-up:** *"Why can't lambdas modify local variables from the enclosing scope?"* → As above — effectively-final capture-by-value; use an `AtomicInteger`/array wrapper or an instance field if mutable shared state is genuinely needed.

---

### Q7.2 — Stream API deep dive: intermediate vs terminal operations, laziness, and common pitfalls.

**Answer:** A Stream represents a **pipeline** of computations over a data source (Collection, array, I/O channel), NOT a data structure itself — it doesn't store elements.

**Intermediate operations** (lazy, return a new Stream): `filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`, `skip`, `peek`.
**Terminal operations** (eager, trigger execution, produce a result or side-effect): `collect`, `forEach`, `reduce`, `count`, `anyMatch`/`allMatch`/`noneMatch`, `findFirst`/`findAny`, `toArray`.

**Laziness — critical concept:** Nothing executes until a terminal operation is invoked. Streams are processed **element-by-element through the whole pipeline** (vertically), not stage-by-stage over the whole collection (horizontally) — this enables short-circuiting.

```java
List<String> result = names.stream()
    .filter(n -> { System.out.println("filter " + n); return n.length() > 3; })
    .map(n -> { System.out.println("map " + n); return n.toUpperCase(); })
    .findFirst()   // short-circuits — processes ONE element through filter+map, not the whole list first
    .map(List::of)
    .orElse(List.of());
```

**`flatMap` vs `map`:** `map` transforms each element 1:1 (`Stream<Stream<X>>` if the function returns a stream); `flatMap` transforms and **flattens** nested structures into a single stream (`Stream<X>`) — essential for List<List<X>> → List<X>.

```java
List<List<String>> nested = List.of(List.of("a","b"), List.of("c"));
List<String> flat = nested.stream().flatMap(List::stream).collect(Collectors.toList());
```

**Collectors — most-used in enterprise code:**
```java
Map<Department, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));

Map<Department, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment, Collectors.averagingDouble(Employee::getSalary)));

String names = employees.stream().map(Employee::getName).collect(Collectors.joining(", ", "[", "]"));

Map<String,Employee> byId = employees.stream()
    .collect(Collectors.toMap(Employee::getId, Function.identity()));

// Java 12+ Collectors.teeing — combine two collectors into one result
var stats = employees.stream().collect(Collectors.teeing(
    Collectors.counting(), Collectors.summingDouble(Employee::getSalary),
    (count, totalSalary) -> totalSalary / count));
```

**Best Practices:**
- Keep stream pipelines **stateless and side-effect-free** (don't use `peek()` for business logic — it's meant for debugging only, and its execution isn't guaranteed for all stream sources).
- Prefer `Collectors.toList()`/`toUnmodifiableList()` (Java 10+) over manual `forEach` + external list mutation.
- Avoid overly long chained pipelines that hurt readability — break into named intermediate variables/methods when it aids clarity.
- Use parallel streams only for CPU-bound, large, stateless computations after benchmarking (see Q6.8).

**Common Mistakes:**
- Reusing a stream after a terminal operation → `IllegalStateException: stream has already been operated upon or closed`. Streams are single-use.
- Using streams for simple loops where a plain `for` is clearer and faster — streams aren't always "better," just more declarative.
- Mutating external state inside `map`/`forEach` (e.g., `list.stream().forEach(x -> externalList.add(x))`) — not thread-safe if parallel, and generally an anti-pattern; use `collect()`.
- Calling `.get()` on `Optional` without checking `isPresent()`/using `orElse` — defeats the purpose of `Optional`.

**Follow-up:** *"Difference between `findFirst()` and `findAny()`?"* → `findFirst()` guarantees encounter order (deterministic even in parallel streams, at the cost of some performance); `findAny()` may return any matching element, allowing better short-circuit performance in parallel streams since it doesn't have to respect ordering.

---

### Q7.3 — `Optional` — purpose, misuse, and best practices.

**Answer:** `Optional<T>` (Java 8) is a container object that may or may not hold a non-null value — designed to **make the possibility of "no value" explicit in the API/type signature**, reducing `NullPointerException`s and forcing callers to consciously handle absence.

```java
Optional<User> user = userRepository.findById(id);
String name = user.map(User::getName).orElse("Unknown");
user.ifPresentOrElse(
    u -> log.info("Found {}", u),
    () -> log.warn("Not found")
);
User u = user.orElseThrow(() -> new ResourceNotFoundException("User not found"));
```

**Best Practices:**
- Use as a **return type** for methods that might legitimately return "nothing" (e.g., repository lookups) — signals intent clearly.
- Never use `Optional` as a **field type**, **method parameter**, or in **collections** (`List<Optional<X>>`) — official Java guidance advises against this; it adds serialization complexity and doesn't compose well. Use `null` fields (with clear docs) or the Null Object pattern instead.
- Prefer `orElseThrow`, `map`, `filter`, `ifPresentOrElse` over imperative `isPresent()` + `get()` (defeats the functional style and reintroduces null-check-like verbosity).
- Use `Optional.ofNullable()` when wrapping a potentially-null value from legacy/external APIs; `Optional.of()` throws NPE immediately if the value is null (fail-fast, useful when you're certain it shouldn't be null).

**Common Mistakes:**
- `if (opt.isPresent()) { return opt.get(); } else { return null; }` — pointless; just `return opt.orElse(null);` or better, don't unwrap to null at all.
- Using `Optional` for every nullable field/parameter — adds needless wrapping overhead and violates JavaBean conventions used by many serialization frameworks (Jackson, JPA).

**Follow-up:** *"Why shouldn't Optional be used as a field in an Entity?"* → JPA/Hibernate entities need no-arg constructors and reflective field access for the persistence provider — `Optional` isn't `Serializable`, and wrapping entity fields adds needless complexity; better to keep fields nullable and expose `Optional` only at the getter/service boundary if desired.

---

## 8. Java Version Features (8 / 11 / 17 / 21) — Frequently Asked

### Q8.1 — What are the landmark features introduced in Java 8, and why was it such a pivotal release?

**Answer (rapid-fire list interviewers expect):**
1. **Lambda Expressions & Functional Interfaces** — functional-style programming.
2. **Stream API** — declarative collection processing.
3. **`Optional<T>`** — explicit absence handling.
4. **Default & static methods in interfaces** — allowed evolving interfaces (e.g., adding `stream()` to `Collection`) without breaking every implementer.
5. **New Date-Time API (`java.time`)** — replaced flawed, mutable, not-thread-safe `Date`/`Calendar` with immutable `LocalDate`, `LocalDateTime`, `ZonedDateTime`, `Duration`, `Period` (JSR-310, inspired by Joda-Time).
6. **Method References** (`Class::method`) — shorthand for lambdas that just call an existing method.
7. **`CompletableFuture`** — composable async programming.
8. **Nashorn JS engine** (later removed in Java 15).
9. **PermGen removed, replaced by Metaspace.**

```java
// Old Date API problems: mutable, not thread-safe, months 0-indexed (classic bug source)
Date d = new Date(); // mutable — bad for shared state
// New API: immutable, thread-safe, fluent
LocalDate today = LocalDate.now();
LocalDate nextWeek = today.plusDays(7);
Duration duration = Duration.between(start, end);
```

**Follow-up:** *"Why was `java.time` needed given `Date`/`Calendar` already existed?"* → `Date`/`Calendar` were mutable (unsafe to share across threads), had confusing APIs (0-indexed months, poor `equals`/`toString`), and mixed date/time/timezone concerns. `java.time` classes are immutable (thread-safe by default), clearly separate concerns (`LocalDate` vs `LocalTime` vs `ZonedDateTime` vs `Instant`), and follow ISO-8601.

---

### Q8.2 — Key Java 11 (LTS) features relevant to enterprise code.

**Answer:**
1. **`var` for local variables** (actually Java 10, but commonly grouped with 11 in interviews) — local-variable type inference; type is still static/compile-time, just inferred, NOT dynamic typing.
   ```java
   var list = new ArrayList<String>(); // inferred as ArrayList<String>
   ```
2. **New `String` methods:** `isBlank()`, `strip()`/`stripLeading()`/`stripTrailing()` (Unicode-aware, unlike `trim()`), `lines()` (splits into a `Stream<String>`), `repeat(n)`.
3. **`Files.readString()` / `Files.writeString()`** — simple one-liner file I/O without manual stream handling.
4. **`HttpClient` (standardized from incubator)** — modern, fluent, supports HTTP/2 and async (`sendAsync` returns `CompletableFuture`), replacing legacy `HttpURLConnection`.
5. **Running single-file source code directly:** `java HelloWorld.java` — no explicit compile step needed for quick scripts.
6. **Local-Variable Syntax for Lambda Parameters:** `(var x, var y) -> x + y` — allows annotating lambda parameters.
7. **Deprecation/removal of Java EE modules (`javax.xml.ws`, etc.)** — decoupled from JDK, must add as explicit dependency.

```java
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder(URI.create("https://api.example.com/data")).GET().build();
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
```

**Follow-up:** *"Is `var` the same as dynamic typing (like JavaScript's `var`)?"* → No — Java's `var` is resolved to a concrete static type **at compile time** by the compiler based on the initializer; you cannot reassign a `var` variable to an incompatible type later. It's purely syntactic sugar for type inference, not a change to Java's static type system.

---

### Q8.3 — Key Java 17 (LTS) features — Records, Sealed Classes, Pattern Matching.

**Answer:**

**1. Records (finalized Java 16, heavily used from 17 onward):** Concise, immutable data carrier classes — compiler auto-generates constructor, `equals()`, `hashCode()`, `toString()`, and accessor methods (named same as fields, no `get` prefix).

```java
public record Point(int x, int y) {
    // Compact constructor for validation
    public Point {
        if (x < 0 || y < 0) throw new IllegalArgumentException("Coordinates must be non-negative");
    }
    // Can add extra methods
    public double distanceFromOrigin() { return Math.sqrt(x*x + y*y); }
}
Point p = new Point(3, 4);
p.x(); // accessor, not getX()
```
Records are implicitly `final`, cannot extend other classes (but can implement interfaces), and all fields are implicitly `private final`. **Excellent for DTOs, value objects, immutable API responses.**

**2. Sealed Classes/Interfaces:** Restrict which classes can extend/implement a type — enables exhaustive pattern matching and models closed hierarchies explicitly (like algebraic data types).

```java
public sealed interface Shape permits Circle, Square, Triangle {}
public record Circle(double radius) implements Shape {}
public record Square(double side) implements Shape {}
public record Triangle(double base, double height) implements Shape {}

// Exhaustive switch — compiler enforces all permitted subtypes are handled (no default needed if exhaustive)
double area = switch (shape) {
    case Circle c -> Math.PI * c.radius() * c.radius();
    case Square s -> s.side() * s.side();
    case Triangle t -> 0.5 * t.base() * t.height();
};
```

**3. Pattern Matching for `instanceof`** (Java 16+): eliminates explicit casting.
```java
if (obj instanceof String s && s.length() > 5) { System.out.println(s.toUpperCase()); }
```

**4. Enhanced (switch expressions, Java 14+):** arrow syntax, no fall-through, `yield` for block bodies.
```java
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY -> 7;
    default -> { yield 0; }
};
```

**5. Text Blocks** (Java 15+): multi-line strings without escape-heavy concatenation — great for embedded SQL/JSON.
```java
String json = """
    {
      "name": "%s",
      "age": %d
    }
    """.formatted(name, age);
```

**6. Removed/deprecated:** Applets removed; Security Manager deprecated for removal.

**Follow-up:** *"How do records differ from Lombok's `@Data`?"* → Records are immutable by design (no setters, all-final fields), part of the language (better tooling/compiler support, pattern-matching friendly), while Lombok `@Data` generates getters/setters (mutable) via annotation processing — a build-time dependency, not a language feature. Records are preferred for true immutable value types; Lombok still useful for mutable JPA entities where records don't fit well (entities need no-arg constructors and mutable fields for the persistence provider).

---

### Q8.4 — Java 21 (LTS) — Virtual Threads, Pattern Matching for switch, Sequenced Collections.

**Answer (very hot topic in 2024-2026 interviews):**

**1. Virtual Threads (Project Loom, JEP 444) — the headline feature:**
- Lightweight threads managed by the **JVM**, not the OS — you can create **millions** of virtual threads (vs a few thousand platform/OS threads before memory/context-switch limits bite).
- Each virtual thread is mounted onto a small pool of **carrier (platform) threads**; when a virtual thread performs a blocking operation (I/O, `Thread.sleep`, blocking queue), it's **unmounted** from its carrier thread (which is then free to run other virtual threads), and remounted when the blocking operation completes — dramatically improving throughput for I/O-bound, thread-per-request style workloads (classic Spring MVC style) without rewriting to reactive/async code.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 100_000).forEach(i ->
        executor.submit(() -> {
            // blocking I/O call — cheap now, no OS thread pinned for the whole duration
            callDownstreamService();
        })
    );
} // auto-awaits completion on close()
```

- **Spring Boot 3.2+ integration:** set `spring.threads.virtual.enabled=true` to make Tomcat/embedded server handle each request on a virtual thread — massive throughput gains for I/O-bound (DB/HTTP call heavy) services **without switching to WebFlux/reactive**.
- **Important caveat:** `synchronized` blocks **pin** the virtual thread to its carrier thread (in Java 21; this was improved/mostly fixed in later JDKs) — under a long `synchronized` block with blocking I/O inside, you lose the benefit and can even exhaust carrier threads. Best practice: prefer `ReentrantLock` over `synchronized` in virtual-thread-heavy code, or keep synchronized blocks short and non-blocking.
- Virtual threads are **not for CPU-bound work** — they don't add more CPU cores; the benefit is for I/O-bound concurrency (waiting), not computation.

**2. Pattern Matching for `switch`** (finalized Java 21, JEP 441) + **Record Patterns** (JEP 440) — deconstruct records directly in `switch`/`instanceof`:
```java
record Point(int x, int y) {}
static String describe(Object obj) {
    return switch (obj) {
        case Point(int x, int y) when x == y -> "Diagonal point";
        case Point(int x, int y) -> "Point at " + x + "," + y;
        case null -> "null value";
        default -> "Unknown";
    };
}
```

**3. Sequenced Collections** (JEP 431) — new interfaces (`SequencedCollection`, `SequencedSet`, `SequencedMap`) providing uniform `getFirst()`, `getLast()`, `addFirst()`, `addLast()`, `reversed()` across `List`, `LinkedHashSet`, `LinkedHashMap`, etc. — no more inconsistent APIs for "get first/last element" across collection types.
```java
List<Integer> list = new ArrayList<>(List.of(1,2,3));
list.getFirst(); // 1 — no more list.get(0)
list.reversed(); // view, not a copy
```

**4. Structured Concurrency (preview, JEP 453)** — treats a group of related tasks running in different threads as a single unit of work, simplifying cancellation/error propagation:
```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<User> userTask = scope.fork(() -> fetchUser(id));
    Subtask<List<Order>> ordersTask = scope.fork(() -> fetchOrders(id));
    scope.join().throwIfFailed();
    return new UserProfile(userTask.get(), ordersTask.get());
}
```

**5. Other notables:** Generational ZGC (Java 21), improved `SecurityManager` deprecation finalization, deprecation of the 32-bit x86 port.

**Best Practices for Virtual Threads:**
- Great fit for classic Spring MVC controllers doing blocking DB/REST calls at scale — often simpler than migrating to WebFlux.
- Don't pool virtual threads (`Executors.newVirtualThreadPerTaskExecutor()` creates a new one per task — that's the intended usage, unlike platform thread pools).
- Audit code for long `synchronized` blocks around blocking calls — replace with `ReentrantLock` where pinning is a concern.
- Don't use virtual threads for CPU-bound parallel computation — stick with `ForkJoinPool`/parallel streams for that.

**Follow-up 🎯:** *"Does using virtual threads mean I should stop using WebFlux/reactive programming?"* → Not necessarily — WebFlux still offers built-in backpressure handling and a functional composition model useful for streaming/complex async pipelines. But for typical CRUD/blocking-I/O-heavy services, virtual threads let you get most of the throughput benefit while keeping simple, familiar imperative/blocking code — reducing the need to adopt reactive programming *purely* for scalability reasons.

**Follow-up:** *"How many virtual threads can you realistically create?"* → Millions (bounded mainly by heap memory for their stack frames, which are much smaller/growable compared to fixed ~1MB default OS thread stacks), vs typically a few thousand platform threads before the OS/JVM becomes unstable.

---

*(Continue to Part 3: Spring Core, Spring Boot, Spring MVC, Spring Data JPA/Hibernate, Spring Security, REST APIs, Microservices)*
