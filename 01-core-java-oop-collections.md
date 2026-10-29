# Java + Spring Boot Interview Handbook (4+ YOE)
## Part 1: Core Java, OOP, Collections Framework, Generics, Exception Handling

> **How to use this handbook:** Read each answer as if you were saying it out loud in an interview. Every answer is structured as: **Core Answer → Internals → Real-world Example → Best Practices → Common Mistakes → Follow-ups**. Tricky questions are flagged with 🎯.

---

## 1. OOP Fundamentals

### Q1.1 — What are the four pillars of OOP? Explain each with a real enterprise example.

**Answer:**

| Pillar | Definition | Enterprise Example |
|---|---|---|
| **Encapsulation** | Bundling data + behavior, hiding internal state via access modifiers | `Account` class exposes `withdraw()`/`deposit()` but keeps `balance` private, enforcing business rules (no negative balance) |
| **Inheritance** | A class acquires fields/methods of another (`extends`) | `AbstractPaymentProcessor` base class with `UpiProcessor`, `CardProcessor` subclasses sharing retry/logging logic |
| **Polymorphism** | Same interface, different implementations (compile-time overloading, runtime overriding) | `NotificationService` interface with `EmailNotification`, `SmsNotification`, `PushNotification` — Spring injects the right bean via `@Qualifier` |
| **Abstraction** | Exposing *what* an object does, hiding *how* | JDBC `Connection` interface hides driver-specific implementation; `Repository` interface hides SQL |

**Internals:** Runtime polymorphism works via the **vtable (virtual method table)** mechanism. Each object carries a pointer to its class's method table; JVM resolves the actual method at runtime based on the object's real type — this is called **dynamic dispatch**. Compile-time (method overloading) is resolved by the compiler based on argument types/count — no vtable involved.

```java
abstract class PaymentProcessor {
    abstract void process(double amount);
    void logTransaction(double amount) { // shared behavior
        System.out.println("Processing: " + amount);
    }
}
class UpiProcessor extends PaymentProcessor {
    void process(double amount) { logTransaction(amount); /* UPI logic */ }
}
```

**Best Practices:**
- Favor **composition over inheritance** — inheritance creates tight coupling and fragile base class problems.
- Program to an interface, not an implementation (`List<String> list = new ArrayList<>();`).
- Keep classes following **SRP (Single Responsibility Principle)**.

**Common Mistakes:**
- Overusing inheritance for code reuse instead of using composition/delegation.
- Making fields `public` "for convenience" — breaks encapsulation.
- Confusing overloading (compile-time) with overriding (runtime) — asked in almost every interview.

**Follow-ups:**
- "Why prefer composition over inheritance?" → Inheritance breaks encapsulation (child depends on parent internals), causes fragile base class issues when parent changes, and Java only allows single inheritance of classes. Composition is flexible, testable (mock the composed object), and avoids deep hierarchies.
- "Can you achieve polymorphism without inheritance?" → Yes, via interfaces (Java doesn't require a common superclass, only a common contract).

---

### Q1.2 🎯 — Method Overloading vs Overriding — what really happens at compile time vs runtime?

**Answer:**

| Aspect | Overloading | Overriding |
|---|---|---|
| Binding | Static / compile-time | Dynamic / runtime |
| Where | Same class (or subclass with different signature) | Subclass, same signature |
| Return type | Can differ (with different params) | Must be same or covariant |
| Access modifier | No restriction | Cannot reduce visibility |
| `private`/`static`/`final` methods | Can be overloaded | **Cannot be overridden** — static methods are hidden, not overridden |
| Exceptions | Any exception can be thrown | Cannot throw broader checked exception than parent |

**Tricky part interviewers probe:** *"If a subclass defines a static method with same signature as parent's static method, is that overriding?"* → **No, it's method hiding.** The method called depends on the **reference type** at compile time, not the object type. This trips up many candidates.

```java
class Parent {
    static void greet() { System.out.println("Parent"); }
}
class Child extends Parent {
    static void greet() { System.out.println("Child"); }
}
Parent p = new Child();
p.greet(); // prints "Parent" — resolved at compile time via reference type
```

**Common Mistakes:** Believing `@Override` is mandatory (it's not — but best practice, since compiler catches signature mismatches).

---

### Q1.3 — Difference between Abstract Class and Interface? When would you choose one over the other (post Java 8)?

**Answer:** Since Java 8 introduced `default` and `static` methods in interfaces, and Java 9 added `private` interface methods, the line has blurred, but key differences remain:

| Feature | Abstract Class | Interface |
|---|---|---|
| State (instance fields) | Yes | Only `public static final` constants |
| Constructors | Yes | No |
| Multiple inheritance | No (single class inheritance) | Yes (a class can implement many interfaces) |
| Method types | Abstract + concrete | Abstract, `default`, `static`, `private` (Java 9+) |
| Access modifiers on methods | Any | `public` by default (except private helper methods) |
| Use case | "is-a" relationship with shared state/code | Contract/capability ("can-do" relationship) |

**When to choose:**
- Use **abstract class** when subclasses share common state and a chunk of implementation (template method pattern).
- Use **interface** when you're defining a capability multiple unrelated classes can implement (`Comparable`, `Serializable`) or you need multiple inheritance of type.

**Real-world:** Spring's `JpaRepository` is an interface hierarchy (multiple inheritance of behavior via default methods); `AbstractHttpMessageConverter` in Spring MVC is an abstract class providing template logic for message conversion, letting subclasses fill in `readInternal`/`writeInternal`.

**Follow-up:** *"Can an interface have a constructor? Why not?"* → No, because interfaces cannot be instantiated directly and hold no instance state to initialize.

**Follow-up 🎯:** *"What happens if a class implements two interfaces with the same default method signature?"* → Compile error ("diamond problem") — the class **must override** the method and explicitly resolve it (optionally calling `InterfaceA.super.method()`).

---

### Q1.4 — What is the difference between `==` and `.equals()`? How does `hashCode()` tie in?

**Answer:** `==` compares references (memory address) for objects, and values for primitives. `.equals()` is a method that, by default (in `Object`), also compares references — but classes like `String`, wrapper classes, and your own POJOs commonly override it for **logical/value equality**.

**The `equals`/`hashCode` contract (critical for HashMap/HashSet correctness):**
1. If `a.equals(b)` is true, then `a.hashCode() == b.hashCode()` **must** be true.
2. The reverse is NOT required (different objects can share a hash — collision).
3. Violating this breaks `HashMap`, `HashSet`, `HashTable` — objects "disappear" because they're stored in one bucket but searched for using a hash that lands in another.

```java
class Employee {
    int id; String name;
    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee)) return false;
        Employee e = (Employee) o;
        return id == e.id && Objects.equals(name, e.name);
    }
    @Override public int hashCode() { return Objects.hash(id, name); }
}
```

**Best Practices:**
- Always override both together, never one alone.
- Use `Objects.equals()` and `Objects.hash()` (Java 7+) to avoid NPEs and boilerplate.
- Since Java 14+, prefer **`record`** for immutable data carriers — `equals`/`hashCode`/`toString` are auto-generated.

**Common Mistakes:**
- Overriding `equals()` without `hashCode()` → silent bugs in Sets/Maps.
- Using mutable fields in `hashCode()` for objects stored as `HashMap` keys — if the field changes after insertion, the object becomes unreachable in the map (its bucket index changes).

**Follow-up:** *"Why is String immutable, and how does that relate to hashCode caching?"* → Since `String` is immutable, its `hashCode()` is computed once and cached (`hash` field), making it fast for repeated use as a `HashMap` key.

---

## 2. Core Java Language Mechanics

### Q2.1 — Explain `String`, `StringBuilder`, `StringBuffer` — internals and when to use which.

**Answer:**

| | `String` | `StringBuilder` | `StringBuffer` |
|---|---|---|---|
| Mutability | Immutable | Mutable | Mutable |
| Thread safety | Immutable → inherently safe | Not synchronized | Synchronized (all methods) |
| Performance | Slow for repeated concatenation (creates new objects) | Fast | Slower than StringBuilder due to sync overhead |
| Storage (pre Java 9) | String Pool (heap) | Heap | Heap |
| Storage (Java 9+) | Compact Strings: `byte[]` + coder flag (Latin-1 or UTF-16) instead of `char[]` — saves ~50% memory for Latin-1-only strings | — | — |

**Internals — String Pool:** String literals are interned in a special pool inside the heap (moved from PermGen to heap in Java 7). `new String("abc")` creates a new object outside the pool even if `"abc"` exists in the pool; `.intern()` forces pool lookup/insertion.

```java
String a = "hello";
String b = "hello";
System.out.println(a == b); // true — same pool reference

String c = new String("hello");
System.out.println(a == c); // false — different object
System.out.println(a == c.intern()); // true
```

**Why immutable?**
1. **Security** — used in class loading, file paths, network connections; mutability would allow tampering after validation.
2. **Thread-safety** — freely shared across threads with no synchronization.
3. **Hashcode caching** — safe to cache since value never changes, speeding up `HashMap` lookups.
4. **String pool reuse** — only possible because strings can't be modified underfoot.

**Best Practice:** Use `StringBuilder` inside loops doing concatenation; use `String` for fixed/rarely-changed text; `StringBuffer` only in legacy multi-threaded code (rare today — prefer external synchronization or `StringBuilder` per-thread).

**Common Mistake:** `String s = ""; for(...) s += x;` inside a loop — creates O(n²) garbage due to repeated new String allocation. Interviewers love asking you to spot this.

---

### Q2.2 — Explain the Java Memory Model at a language level: Stack vs Heap, and how does an object's lifecycle look?

**Answer:**
- **Stack:** Each thread has its own stack. Stores method call frames — local variables, primitive values, and **references** to objects. LIFO structure. Automatically reclaimed when a method returns. `StackOverflowError` occurs on deep/infinite recursion.
- **Heap:** Shared across all threads. Stores actual objects and arrays. Managed by the Garbage Collector. Divided into generations (Young: Eden + Survivor S0/S1, Old/Tenured) under most GCs.
- **Metaspace** (Java 8+, replaced PermGen): Stores class metadata, method bytecode, runtime constant pool. Grows into native memory, not bounded by heap size (configurable via `-XX:MaxMetaspaceSize`).

```
[Thread Stack]        [Heap]                          [Metaspace]
 frame: main()          Young Gen (Eden, S0, S1)        Class metadata
   ref -----------> obj Employee{...}                    Method bytecode
 frame: process()      Old Gen (tenured objects)          Constant pool
   ref -----------> obj List<...>
```

**Object lifecycle:**
1. `new Employee()` → allocated in Eden space (Young Gen).
2. Survives a Minor GC → moved to Survivor space; age counter incremented.
3. After crossing tenuring threshold (default 15, tunable) → promoted to Old Gen.
4. Large objects may go directly to Old Gen (`-XX:PretenureSizeThreshold`).
5. Unreachable objects reclaimed by GC (Minor GC for young, Major/Full GC for old).

**Follow-up:** *"What causes a StackOverflowError vs OutOfMemoryError?"* → SOE: stack space exhausted (deep recursion). OOME: heap exhausted (`java.lang.OutOfMemoryError: Java heap space`) or Metaspace exhausted (`Metaspace`) or too many threads (`unable to create new native thread`).

---

### Q2.3 — Explain Garbage Collection — algorithms, generational hypothesis, and how to choose a GC in production.

**Answer:** GC automatically reclaims heap memory occupied by objects with no live references, based on the **weak generational hypothesis**: most objects die young, so splitting the heap into generations lets the JVM collect young objects cheaply and frequently, while scanning the old generation rarely.

**Reachability, not reference counting:** Java doesn't use naive reference counting (which fails on cycles). GC roots (stack references, static fields, JNI references, active threads) are traced; anything unreachable is garbage — this correctly handles circular references (`A` refs `B`, `B` refs `A`, but neither reachable from a GC root → both collected).

**Common collectors (know differences for Spring Boot production tuning questions):**

| Collector | Type | Best For | Notes |
|---|---|---|---|
| **Serial GC** | Single-threaded, stop-the-world | Small heaps, single-core | `-XX:+UseSerialGC` |
| **Parallel GC** | Multi-threaded, stop-the-world | Throughput-focused batch jobs | Default in Java 8 |
| **G1 (Garbage First)** | Region-based, mostly concurrent | Default since Java 9; balances latency & throughput on heaps up to ~32GB+ | `-XX:+UseG1GC`, target pause via `-XX:MaxGCPauseMillis` |
| **ZGC** | Concurrent, region-based, colored pointers | Ultra-low latency (sub-millisecond pauses) even on multi-TB heaps | Production-ready since Java 15; ideal for latency-sensitive services (trading, payments) |
| **Shenandoah** | Concurrent, low pause | Similar goals to ZGC | Not shipped in Oracle JDK by default |

**Minor GC vs Major/Full GC:**
- **Minor GC:** cleans Young Gen only, fast, frequent, usually stop-the-world but short.
- **Full GC:** cleans entire heap (Young + Old + Metaspace), expensive, causes visible latency spikes — a major target of production tuning.

**Real-world scenario:** A Spring Boot microservice handling payment transactions saw p99 latency spikes every few minutes. Root cause: default Parallel GC causing multi-hundred-ms Full GC pauses under load. Fix: switched to G1GC with tuned `-XX:MaxGCPauseMillis=200`, and reduced short-lived object churn (e.g., avoided unnecessary DTO copies, used object pooling for hot-path buffers). Result: eliminated most Full GCs, more consistent p99.

**Best Practices:**
- Size Young Gen appropriately (`-Xmn`) to avoid premature promotion.
- Avoid creating unnecessary short-lived objects in hot paths (e.g., don't build new `SimpleDateFormat`/regex `Pattern` per request).
- Monitor with GC logs (`-Xlog:gc*`) and tools like GCViewer, or APM (New Relic, Datadog).
- Use `-XX:+HeapDumpOnOutOfMemoryError` in production for post-mortem analysis.

**Common Mistakes:**
- Assuming `System.gc()` forces GC — it's only a *hint*; JVM may ignore it. Never call it in production code.
- Setting `-Xms` and `-Xmx` to different values in latency-sensitive services, causing heap resize pauses — set them equal in production for predictability.
- Memory leaks via static collections (`static List<X> cache = new ArrayList<>();` that keeps growing) — GC can't reclaim objects that are still "reachable," even if logically unused.

**Follow-up 🎯:** *"Can you have a memory leak in Java despite GC?"* → Yes — GC only reclaims *unreachable* objects. If you keep unintended references alive (static caches, unclosed listeners, ThreadLocal not cleaned up, inner class holding outer reference), objects stay reachable forever → leak. Classic case: `ThreadLocal` in a thread-pooled environment (like Tomcat) not calling `.remove()`, causing memory to leak across requests since pooled threads are reused.

---

## 3. Collections Framework

### Q3.1 — Give the full Collections hierarchy and explain when to use each major implementation.

**Answer:**

```
                     Iterable
                        |
                    Collection
        ____________|___________________
       |             |                  |
     List           Set                Queue
   /  |   \      /   |    \          /   |   \
ArrayList       HashSet  LinkedHashSet  PriorityQueue ArrayDeque
LinkedList      TreeSet (SortedSet/NavigableSet)     LinkedList

                      Map (separate hierarchy, not a Collection)
             /         |           \
        HashMap   LinkedHashMap   TreeMap (SortedMap/NavigableMap)
          |
     ConcurrentHashMap (java.util.concurrent)
```

| Implementation | Ordering | Duplicates | Null | Thread-safe | Time Complexity (avg) | When to use |
|---|---|---|---|---|---|---|
| `ArrayList` | Insertion order | Yes | Yes | No | get O(1), add O(1) amortized, insert/remove middle O(n) | Random access, more reads than writes |
| `LinkedList` | Insertion order | Yes | Yes | No | get O(n), add/remove at ends O(1) | Frequent insert/delete at head/tail (also implements `Deque`) |
| `HashSet` | No guaranteed order | No | One null | No | O(1) | Fast membership check, don't care about order |
| `LinkedHashSet` | Insertion order | No | One null | No | O(1) | Need order + uniqueness |
| `TreeSet` | Sorted (natural/comparator) | No | No (NPE) | No | O(log n) | Need sorted unique elements, range queries |
| `HashMap` | No order | Keys unique | One null key, many null values | No | O(1) avg | General-purpose key-value store |
| `LinkedHashMap` | Insertion (or access, for LRU) order | Keys unique | Yes | No | O(1) | LRU caches, predictable iteration |
| `TreeMap` | Sorted by key | Keys unique | No null key | No | O(log n) | Sorted key iteration, `floorKey`/`ceilingKey` range queries |
| `ConcurrentHashMap` | No order | Keys unique | No null key/value | Yes | O(1) avg | Multi-threaded caches/counters |
| `CopyOnWriteArrayList` | Insertion order | Yes | Yes | Yes | read O(1), write O(n) copy | Read-heavy, rarely-written concurrent lists (listener lists) |

**Real-world:** LRU cache implementation using `LinkedHashMap`:
```java
class LruCache<K,V> extends LinkedHashMap<K,V> {
    private final int capacity;
    LruCache(int capacity) { super(16, 0.75f, true); this.capacity = capacity; }
    @Override protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return size() > capacity;
    }
}
```

---

### Q3.2 — Explain `HashMap` internal working in depth (bucket, hashing, treeification).

**Answer (this is one of the most-asked questions for 3+ YOE):**

1. **Structure:** Internally an array of `Node<K,V>[] table` (buckets). Default capacity 16, load factor 0.75 → resizes (doubles) when `size > capacity * loadFactor`.
2. **Hashing:** `hash(key)` = `key.hashCode() ^ (hashCode >>> 16)` — this **spreads** the higher bits into lower bits to reduce collisions from poor hash functions, since bucket index = `hash & (capacity - 1)` only uses lower bits.
3. **Bucket index:** `index = hash & (n - 1)` where `n` is table length (power of 2) — bitwise AND is faster than modulo and works correctly because n is always a power of 2.
4. **Collision handling:** Multiple keys hashing to the same bucket are stored as a **linked list** (chaining). Since **Java 8**, if a bucket's linked list grows beyond `TREEIFY_THRESHOLD` (8) **and** table capacity ≥ 64, it's converted to a **Red-Black Tree** for that bucket — improving worst-case lookup from O(n) to O(log n). If capacity < 64, it resizes instead of treeifying.
5. **Resize:** On resize, all entries are rehashed and redistributed — an expensive operation (O(n)), hence importance of sizing initial capacity if you know approximate size (`new HashMap<>(expectedSize)`).
6. **`put()` flow:** compute hash → find bucket → if empty, insert; if key exists (via `equals`), update value; else append to chain/tree → check treeify/resize conditions.

```
HashMap<K,V> table (capacity 16)
 index 0: null
 index 1: Node(k1,v1) -> Node(k5,v5)   // collision chain
 index 2: null
 ...
 index 7: [Red-Black Tree root] (>8 nodes, capacity>=64)
```

**Java 7 vs Java 8 HashMap differences (frequently asked):**
- Java 7: collision resolution purely via linked list; new colliding entries inserted at **head** (caused infamous infinite-loop bug during concurrent resize).
- Java 8: linked list → tree conversion for large buckets; new entries appended at **tail**; resize rehashing logic improved to avoid the Java 7 concurrent-modification infinite loop (though `HashMap` is still NOT thread-safe — this just reduced one specific failure mode).

**Best Practices:**
- Always override `equals`/`hashCode` correctly for custom key objects.
- Prefer immutable keys (or at least don't mutate fields used in hashCode after insertion).
- Set initial capacity when size is roughly known, to avoid multiple resizes.
- Use `ConcurrentHashMap` for multithreaded access, never synchronize `HashMap` manually unless legacy constraint.

**Common Mistakes:**
- Using mutable objects as keys and mutating them post-insertion → entry becomes "lost."
- Iterating and modifying a `HashMap` directly → `ConcurrentModificationException`; use `Iterator.remove()` or `entrySet().removeIf()`.
- Assuming `HashMap` preserves insertion order (it doesn't — use `LinkedHashMap`).

**Follow-up 🎯:** *"Why is capacity always a power of 2?"* → So that `hash & (n-1)` is equivalent to `hash % n` but much faster (bitwise op), and ensures a more uniform bit-mask distribution across buckets during resizing.

**Follow-up:** *"Is `HashMap` fail-fast? What does that mean?"* → Yes, its iterator throws `ConcurrentModificationException` if the map is structurally modified after iterator creation (except through the iterator itself) — detected via a `modCount` field, not a hard guarantee (best-effort).

---

### Q3.3 — `ConcurrentHashMap` internals — how is it thread-safe without global locking?

**Answer:**
- **Java 7:** Used **segment-based locking** — the map was divided into 16 segments (default concurrency level), each independently locked, allowing 16 concurrent writers.
- **Java 8+:** Segments removed. Uses **CAS (Compare-And-Swap)** operations for lock-free updates on empty buckets, and **synchronized blocks scoped to individual bin (bucket) heads** only when necessary (e.g., appending to a chain or treeifying) — far more granular than Java 7's segment locks, improving concurrency significantly.
- Reads are largely **lock-free** using `volatile` reads on `Node` fields — readers never block.
- Size tracking uses a `LongAdder`-like striped counter (`baseCount` + `CounterCell[]`) to avoid contention on a single counter under high write concurrency.

**Comparison table:**

| | `HashMap` | `Hashtable` | `Collections.synchronizedMap` | `ConcurrentHashMap` |
|---|---|---|---|---|
| Thread-safe | No | Yes (whole-method sync) | Yes (wrapper, whole-map lock) | Yes (fine-grained) |
| Null keys/values | Allowed | Not allowed | Depends on backing map | Not allowed |
| Performance under contention | N/A | Poor (single lock) | Poor (single lock) | Good (bucket-level) |
| Iterator | Fail-fast | Fail-fast | Fail-fast | **Weakly consistent** (won't throw CME, may or may not reflect concurrent updates) |

**Real-world:** Caching layer for a service catalog — `ConcurrentHashMap<String, ServiceConfig>` used for an in-memory config cache refreshed periodically by a background thread while being read by many request-handling threads concurrently, with no explicit locking needed.

**Follow-up:** *"Why doesn't ConcurrentHashMap allow null keys/values?"* → To avoid ambiguity: in a concurrent context, `map.get(key) == null` can't distinguish "key absent" from "key present with null value," which is dangerous when multiple threads might be racing to insert.

---

### Q3.4 — `ArrayList` vs `LinkedList` internals, and `Iterator` vs `ListIterator` vs `Fail-Fast` vs `Fail-Safe`.

**Answer:**
- **`ArrayList`:** backed by a dynamically resizing `Object[]`. Grows by 50% (`newCapacity = oldCapacity + (oldCapacity >> 1)`) when full, via `Arrays.copyOf`. Random access O(1); insert/delete at arbitrary index O(n) due to shifting.
- **`LinkedList`:** doubly-linked list of `Node` objects (`prev`, `item`, `next`). O(1) insert/delete at ends, O(n) random access, more memory overhead per element (extra pointers).

**Iterator types:**
- **Fail-fast** (`ArrayList`, `HashMap` default iterators): throws `ConcurrentModificationException` if the collection is structurally modified during iteration (detected via `modCount`).
- **Fail-safe** (`CopyOnWriteArrayList`, `ConcurrentHashMap`): iterates over a snapshot or is weakly consistent — no exception, but may not reflect the latest state.
- **`ListIterator`:** bidirectional (supports `hasPrevious()`/`previous()`), allows `set()` (replace element during iteration) in addition to `remove()`, which the plain `Iterator` doesn't support.

**Common Mistake:** `for (String s : list) { if (cond) list.remove(s); }` → CME. Correct: use `Iterator.remove()` or `list.removeIf(cond)`.

**Follow-up:** *"Is `Vector` still used?"* → Legacy, synchronized like `Hashtable`, generally avoided today in favor of `ArrayList` + explicit synchronization or `CopyOnWriteArrayList`/`Collections.synchronizedList`.

---

### Q3.5 — Comparable vs Comparator — with real code.

**Answer:**

| | `Comparable` | `Comparator` |
|---|---|---|
| Package | `java.lang` | `java.util` |
| Method | `compareTo(T o)` | `compare(T o1, T o2)` |
| Sorting logic location | Inside the class itself (natural ordering) | External, separate class/lambda |
| Number of sort sequences | One (natural order) | Many (as many comparators as needed) |
| Modifies class | Yes | No |

```java
class Employee implements Comparable<Employee> {
    int salary;
    public int compareTo(Employee o) { return Integer.compare(this.salary, o.salary); } // natural order
}
// External, flexible, multiple orderings, Java 8 style:
list.sort(Comparator.comparing(Employee::getSalary)
                     .thenComparing(Employee::getName)
                     .reversed());
```

**Best Practice:** Prefer `Comparator` with method references + `thenComparing` for readability and to avoid mutating domain classes just for sorting. Use `Comparator.nullsFirst`/`nullsLast` for null-safe sorting.

**Follow-up:** *"What does `compareTo` returning 0 mean for `TreeSet`?"* → `TreeSet`/`TreeMap` treat elements as **equal (and thus duplicates, only one retained)** if `compareTo`/`compare` returns 0 — even if `.equals()` would say otherwise. This is a classic gotcha: **`TreeSet` consistency should align `compareTo` with `equals`**, or you silently lose elements.

---

## 4. Generics

### Q4.1 — What are Generics, why do we need them, and explain Type Erasure.

**Answer:** Generics provide **compile-time type safety** and eliminate the need for explicit casting, catching type mismatches at compile time instead of `ClassCastException` at runtime (pre-Java 5, collections were raw `Object` containers).

**Type Erasure:** Java implements generics via erasure — generic type information exists **only at compile time**; the compiler:
1. Replaces type parameters with their bounds (`Object` if unbounded, or the bound type e.g. `T extends Number` → `Number`).
2. Inserts casts where needed.
3. Generates **bridge methods** to preserve polymorphism when overriding generic methods.

At runtime, `List<String>` and `List<Integer>` are the **same class** (`java.util.List`) — you cannot do `list instanceof List<String>` (compile error), and `list1.getClass() == list2.getClass()` is `true` regardless of type parameter.

```java
List<String> strings = new ArrayList<>();
List<Integer> ints = new ArrayList<>();
System.out.println(strings.getClass() == ints.getClass()); // true — type erased
```

**Why erasure (not reified generics like C#)?** Backward compatibility with pre-Java-5 bytecode/libraries — erasure lets generic and non-generic code interoperate.

**Bounded types & wildcards:**
```java
class Box<T extends Number> { } // T must be Number or subtype

// PECS: Producer Extends, Consumer Super
void copy(List<? extends Number> src, List<? super Number> dest) {
    for (Number n : src) dest.add(n);
}
```

**PECS explained:** 
- `? extends T` (upper bound) — use when you only **read/produce** from the structure (you can't safely `add()` to it, since actual type is unknown subtype).
- `? super T` (lower bound) — use when you only **write/consume** into the structure.
- This is directly used in `Collections.copy(List<? super T> dest, List<? extends T> src)`.

**Best Practices:**
- Prefer generic methods over raw types — never use raw `List`, always parameterize.
- Use bounded wildcards in public APIs for maximum flexibility (`Collections.max(Collection<? extends T>)`).

**Common Mistakes:**
- Trying to create generic arrays directly: `T[] arr = new T[10];` → compile error due to erasure (array covariance vs generic erasure conflict). Workaround: `@SuppressWarnings("unchecked") T[] arr = (T[]) new Object[10];`
- Overloading methods that erase to the same signature: `void foo(List<String> l)` and `void foo(List<Integer> l)` → compile error, both erase to `foo(List)`.

**Follow-up 🎯:** *"Can you overload a method based only on generic type parameter?"* → No, due to erasure — same erasure signature causes a compile-time clash.

---

## 5. Exception Handling

### Q5.1 — Checked vs Unchecked exceptions — design philosophy and when to use which in a Spring Boot service.

**Answer:**

| | Checked | Unchecked (Runtime) |
|---|---|---|
| Hierarchy | `Exception` (not `RuntimeException`) | `RuntimeException` and its subclasses |
| Compiler enforcement | Must be declared (`throws`) or caught | Not enforced |
| Examples | `IOException`, `SQLException`, `ClassNotFoundException` | `NullPointerException`, `IllegalArgumentException`, `IndexOutOfBoundsException` |
| Philosophy | Recoverable conditions the caller *should* handle | Programming errors / conditions typically not recoverable at the call site |
| Typical Spring Boot usage | Rare in service layer; often wrapped into unchecked | Preferred — keeps method signatures clean, use `@ControllerAdvice` for centralized handling |

**`Error` vs `Exception`:** `Error` (e.g., `OutOfMemoryError`, `StackOverflowError`) represents serious JVM-level problems applications generally should NOT try to catch/recover from.

**Real-world Spring Boot pattern:**
```java
// Custom unchecked exception for domain errors
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String msg) { super(msg); }
}

@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse(ex.getMessage(), HttpStatus.NOT_FOUND.value()));
    }
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unhandled exception", ex);
        return ResponseEntity.status(500).body(new ErrorResponse("Internal error", 500));
    }
}
```

**try-with-resources (Java 7+, enhanced Java 9):** Auto-closes any `AutoCloseable` resource, even on exception, avoiding `finally` boilerplate and resource leaks. Java 9+ allows using **effectively final variables declared outside** the try directly.

```java
// Java 9+ style — no need to redeclare inside try()
BufferedReader reader = new BufferedReader(new FileReader("file.txt"));
try (reader) {
    return reader.readLine();
} // auto-closed, and suppressed exceptions from close() are attached via getSuppressed()
```

**Best Practices:**
- Never swallow exceptions silently (`catch (Exception e) {}`) — at minimum log with context.
- Throw specific exceptions, not generic `Exception`/`RuntimeException`.
- Use `@ControllerAdvice`/`@ExceptionHandler` for centralized, consistent error responses instead of try-catch in every controller.
- Wrap checked exceptions from libraries (JDBC, IO) into meaningful unchecked domain exceptions rather than leaking `SQLException` up the stack.
- Include the original exception as `cause` when wrapping (`throw new ServiceException("msg", e);`) — preserves the stack trace for debugging.

**Common Mistakes:**
- Catching `Exception` broadly and losing the real cause.
- Using exceptions for normal control flow (expensive — stack trace capture has real cost).
- Not closing resources — pre-Java 7 code with manual `finally { conn.close(); }` easily leaks on early returns/exceptions.
- Catching then rethrowing without adding value ("catch-log-rethrow" ceremony that just adds noise).

**Follow-up 🎯:** *"What's the exception hierarchy for `NumberFormatException`?"* → `NumberFormatException extends IllegalArgumentException extends RuntimeException extends Exception extends Throwable`.

**Follow-up:** *"What happens if an exception is thrown in both `try` and `finally`?"* → The exception from `finally` **suppresses** the one from `try` (the try's exception is lost, unless you use try-with-resources, which attaches it as a *suppressed* exception via `Throwable.getSuppressed()`, preserving both).

---

*(Continue to Part 2: Multithreading, JVM Internals, Functional Programming, Java 8/11/17/21 Features)*
