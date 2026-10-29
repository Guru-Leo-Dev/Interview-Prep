# Java Streams — Complete Interview Study Guide

---

## 1. What is a Stream?

A **Stream** (`java.util.stream`) is a sequence of elements supporting **functional-style operations** to process data declaratively. Introduced in **Java 8**.

Key characteristics:
- **Not a data structure** — it doesn't store elements; it conveys elements from a source (Collection, array, I/O channel) through a pipeline of computations.
- **Functional in nature** — operations produce a result without modifying the source.
- **Lazy** — intermediate operations are not executed until a terminal operation is invoked.
- **Consumable only once** — once a terminal operation runs, the stream is closed. Reusing it throws `IllegalStateException`.
- **Can be parallelized** easily via `parallelStream()`.

```java
List<String> names = List.of("Amit", "Bala", "Chitra", "Deepa");
names.stream()
     .filter(n -> n.startsWith("A") || n.startsWith("C"))
     .map(String::toUpperCase)
     .sorted()
     .forEach(System.out::println);
```

---

## 2. Stream Pipeline Anatomy

A pipeline has 3 parts:

| Stage | Description | Examples |
|---|---|---|
| **Source** | Where the stream is created from | `Collection.stream()`, `Stream.of()`, `Arrays.stream()`, `IntStream.range()` |
| **Intermediate Operations** | Return a new stream, lazily evaluated, can be chained | `map`, `filter`, `sorted`, `distinct`, `limit`, `skip`, `flatMap`, `peek` |
| **Terminal Operation** | Triggers execution, produces a result or side-effect | `collect`, `forEach`, `reduce`, `count`, `anyMatch`, `findFirst` |

---

## 3. Creating Streams

```java
// From a collection
List<Integer> list = List.of(1, 2, 3);
Stream<Integer> s1 = list.stream();

// From values
Stream<String> s2 = Stream.of("a", "b", "c");

// From an array
int[] arr = {1, 2, 3};
IntStream s3 = Arrays.stream(arr);

// Infinite streams
Stream<Integer> s4 = Stream.iterate(0, n -> n + 2).limit(5);   // 0,2,4,6,8
Stream<Double> s5 = Stream.generate(Math::random).limit(3);

// Ranges
IntStream s6 = IntStream.range(1, 5);      // 1,2,3,4
IntStream s7 = IntStream.rangeClosed(1, 5);// 1,2,3,4,5

// From a file
Stream<String> lines = Files.lines(Path.of("file.txt"));
```

---

## 4. Intermediate Operations (Lazy)

| Operation | Purpose |
|---|---|
| `filter(Predicate)` | Keep elements matching a condition |
| `map(Function)` | Transform each element |
| `flatMap(Function)` | Flatten nested streams/collections into one stream |
| `distinct()` | Remove duplicates (uses `equals()`) |
| `sorted()` / `sorted(Comparator)` | Sort elements |
| `limit(n)` | Truncate to first n elements |
| `skip(n)` | Skip first n elements |
| `peek(Consumer)` | Debug/inspect elements without altering stream |

```java
// flatMap example: List<List<Integer>> -> single stream
List<List<Integer>> nested = List.of(List.of(1,2), List.of(3,4));
List<Integer> flat = nested.stream()
        .flatMap(List::stream)
        .collect(Collectors.toList());
// [1, 2, 3, 4]
```

---

## 5. Terminal Operations (Eager)

| Operation | Purpose |
|---|---|
| `collect(Collector)` | Accumulate into List/Set/Map/String etc. |
| `forEach(Consumer)` | Perform an action per element |
| `reduce(...)` | Combine elements into a single result |
| `count()` | Count elements |
| `anyMatch/allMatch/noneMatch(Predicate)` | Boolean short-circuiting checks |
| `findFirst()/findAny()` | Return an `Optional` |
| `toArray()` | Convert to array |
| `min(Comparator)/max(Comparator)` | Return `Optional` extremum |

```java
int sum = Stream.of(1, 2, 3, 4).reduce(0, Integer::sum); // 10

Optional<Integer> max = Stream.of(1, 5, 3).max(Integer::compareTo);
```

---

## 6. Collectors (java.util.stream.Collectors)

```java
List<Employee> emps = ...;

// To List / Set / Map
List<String> names = emps.stream().map(Employee::getName).collect(Collectors.toList());
Set<String> deptSet = emps.stream().map(Employee::getDept).collect(Collectors.toSet());
Map<Integer, String> idToName = emps.stream()
        .collect(Collectors.toMap(Employee::getId, Employee::getName));

// Joining strings
String csv = names.stream().collect(Collectors.joining(", ", "[", "]"));

// Grouping
Map<String, List<Employee>> byDept = emps.stream()
        .collect(Collectors.groupingBy(Employee::getDept));

Map<String, Long> countByDept = emps.stream()
        .collect(Collectors.groupingBy(Employee::getDept, Collectors.counting()));

// Partitioning (boolean split)
Map<Boolean, List<Employee>> partitioned = emps.stream()
        .collect(Collectors.partitioningBy(e -> e.getSalary() > 50000));

// Summary statistics
DoubleSummaryStatistics stats = emps.stream()
        .collect(Collectors.summarizingDouble(Employee::getSalary));
// stats.getMax(), getMin(), getAverage(), getSum(), getCount()

// Averaging / Summing
double avgSalary = emps.stream().collect(Collectors.averagingDouble(Employee::getSalary));
```

---

## 7. Parallel Streams

```java
list.parallelStream()
    .filter(x -> x % 2 == 0)
    .forEach(System.out::println);
```

- Uses the **ForkJoinPool.commonPool()** under the hood.
- Good for **large datasets** and **CPU-bound, stateless, independent** operations.
- Avoid for: small datasets (overhead > benefit), I/O-bound tasks, operations with shared mutable state, or when order matters (`forEach` order isn't guaranteed — use `forEachOrdered`).

---

## 8. Common Pitfalls

1. **Reusing a stream** after a terminal operation → `IllegalStateException`.
2. **Modifying the source collection** while streaming → `ConcurrentModificationException`.
3. Using **`peek()`** for side effects/business logic instead of debugging — discouraged.
4. Forgetting streams are **lazy** — intermediate ops won't run without a terminal op.
5. Using **parallelStream()** with **shared mutable state** → race conditions.
6. Overusing streams for simple loops — hurts readability/performance for trivial cases.

---

## 9. Stream vs Collection

| Aspect | Collection | Stream |
|---|---|---|
| Storage | Stores data in memory | No storage, pipes data |
| Traversal | Can traverse multiple times | Single traversal only |
| Evaluation | Eager | Lazy (intermediate ops) |
| Modification | Mutable | Doesn't modify source |
| Iteration | External (`for`, `iterator`) | Internal (`forEach`, pipeline) |

---

## 10. Practice Problems

### Problem 1: Find the sum of squares of even numbers
```java
List<Integer> nums = List.of(1, 2, 3, 4, 5, 6);
int result = nums.stream()
        .filter(n -> n % 2 == 0)
        .mapToInt(n -> n * n)
        .sum();
// Answer: 4 + 16 + 36 = 56
```

### Problem 2: Find the second-highest salary from a list of employees
```java
Optional<Double> secondHighest = emps.stream()
        .map(Employee::getSalary)
        .distinct()
        .sorted(Comparator.reverseOrder())
        .skip(1)
        .findFirst();
```

### Problem 3: Group words by their length
```java
List<String> words = List.of("cat", "dog", "apple", "bee", "kiwi");
Map<Integer, List<String>> byLength = words.stream()
        .collect(Collectors.groupingBy(String::length));
// {3=[cat, dog, bee], 5=[apple], 4=[kiwi]}
```

### Problem 4: Find frequency of each character in a string
```java
String str = "programming";
Map<Character, Long> freq = str.chars()
        .mapToObj(c -> (char) c)
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
```

### Problem 5: Find duplicate elements in a list
```java
List<Integer> nums = List.of(1, 2, 3, 2, 4, 5, 1);
Set<Integer> seen = new HashSet<>();
Set<Integer> duplicates = nums.stream()
        .filter(n -> !seen.add(n))
        .collect(Collectors.toSet());
// {1, 2}
```

### Problem 6: Flatten a list of lists and remove duplicates
```java
List<List<Integer>> lists = List.of(List.of(1,2,3), List.of(3,4,5), List.of(5,6));
List<Integer> result = lists.stream()
        .flatMap(List::stream)
        .distinct()
        .sorted()
        .collect(Collectors.toList());
// [1, 2, 3, 4, 5, 6]
```

### Problem 7: Convert a List<Employee> to Map<Dept, Double> (average salary per department)
```java
Map<String, Double> avgSalaryByDept = emps.stream()
        .collect(Collectors.groupingBy(Employee::getDept,
                  Collectors.averagingDouble(Employee::getSalary)));
```

### Problem 8: Find the first non-repeated character in a string
```java
String s = "swiss";
Optional<Character> firstNonRepeated = s.chars()
        .mapToObj(c -> (char) c)
        .collect(Collectors.groupingBy(Function.identity(), LinkedHashMap::new, Collectors.counting()))
        .entrySet().stream()
        .filter(e -> e.getValue() == 1)
        .map(Map.Entry::getKey)
        .findFirst();
// 'w'
```

### Problem 9: Sort a list of custom objects by multiple fields
```java
List<Employee> sorted = emps.stream()
        .sorted(Comparator.comparing(Employee::getDept)
                           .thenComparing(Employee::getSalary, Comparator.reverseOrder()))
        .collect(Collectors.toList());
```

### Problem 10: Implement your own "reduce" to concatenate strings
```java
List<String> words = List.of("Java", "Streams", "Rock");
String sentence = words.stream()
        .reduce("", (a, b) -> a.isEmpty() ? b : a + " " + b);
// "Java Streams Rock"
```

---

## 11. Interview Questions & Answers

**Q1. What is the difference between `map()` and `flatMap()`?**
> `map()` transforms each element to another object (1-to-1), producing a `Stream<Stream<T>>` if the mapper returns a stream. `flatMap()` transforms and then **flattens** nested streams into a single stream (1-to-many merged into one level). Use `flatMap` when each element maps to a collection/stream that should be merged.

**Q2. Why are Streams lazy? What does that mean in practice?**
> Intermediate operations (`map`, `filter`, etc.) just build a pipeline description — they don't execute until a terminal operation (`collect`, `forEach`, etc.) is called. This allows optimizations like short-circuiting (`findFirst`, `limit`) and avoids unnecessary computation. For example, `filter().map().findFirst()` may stop after processing just one matching element rather than the whole collection.

**Q3. Can a Stream be reused?**
> No. Once a terminal operation is invoked, the stream is considered "consumed" and closed. Any further operation on it throws `IllegalStateException: stream has already been operated upon or closed`. You must create a new stream from the source if you need to traverse again.

**Q4. What's the difference between `Collection` and `Stream`?**
> A `Collection` is an in-memory data structure supporting multiple traversals and external iteration. A `Stream` is a pipeline abstraction for computing on data — it doesn't store elements, supports only a single traversal, and uses internal iteration with lazy evaluation.

**Q5. What is the difference between intermediate and terminal operations?**
> Intermediate operations return a new `Stream` and are lazy (e.g., `filter`, `map`, `sorted`). Terminal operations trigger the actual traversal/computation and produce a non-stream result or side effect (e.g., `collect`, `reduce`, `forEach`). A pipeline needs exactly one terminal operation to execute.

**Q6. How does `reduce()` work? Explain its overloads.**
> `reduce()` combines stream elements into a single value using an associative accumulator function.
> - `Optional<T> reduce(BinaryOperator<T> accumulator)` — no identity, returns `Optional` (empty if stream is empty).
> - `T reduce(T identity, BinaryOperator<T> accumulator)` — has a starting value (identity), always returns `T`.
> - `<U> U reduce(U identity, BiFunction<U,T,U> accumulator, BinaryOperator<U> combiner)` — used mainly for parallel streams, where `combiner` merges partial results from different threads.

**Q7. What is the difference between `findFirst()` and `findAny()`?**
> `findFirst()` returns the first element encountered in encounter order — deterministic even in parallel streams. `findAny()` has no such guarantee and may return any matching element, which allows the JVM to short-circuit faster in parallel streams (better performance, non-deterministic result).

**Q8. When should you avoid parallel streams?**
> - Small datasets, where thread-management overhead outweighs benefits.
> - I/O-bound operations (parallel streams use a shared `ForkJoinPool` meant for CPU-bound work).
> - Operations involving shared mutable state (risk of race conditions).
> - When order of processing/output matters and hasn't been explicitly handled (`forEachOrdered`).

**Q9. What's the difference between `Collectors.toMap()` throwing an exception on duplicate keys, and how do you fix it?**
> `Collectors.toMap(keyMapper, valueMapper)` throws `IllegalStateException: Duplicate key` if two elements map to the same key. Fix by supplying a merge function: `Collectors.toMap(keyMapper, valueMapper, (v1, v2) -> v1)` (keep first) or accumulate differently (e.g., merge into a list).

**Q10. Explain `Collectors.groupingBy()` vs `Collectors.partitioningBy()`.**
> `groupingBy()` groups elements by a classifier function into a `Map<K, List<T>>` (K can be any type). `partitioningBy()` is a special case that splits elements into exactly two groups based on a `Predicate`, producing a `Map<Boolean, List<T>>`.

**Q11. What happens if you call `stream()` on a `null` collection or use `Optional` incorrectly with streams?**
> `null.stream()` throws `NullPointerException` since `stream()` is invoked on a null reference — always null-check the source, or use `Optional.ofNullable(collection).map(Collection::stream).orElseGet(Stream::empty)`. With `Optional`, calling `.get()` without checking `isPresent()`/`isEmpty()` risks `NoSuchElementException`; prefer `orElse`, `orElseGet`, `orElseThrow`, or `ifPresent`.

**Q12. How is short-circuiting achieved in Streams? Give examples.**
> Short-circuit operations don't need to process the entire stream to produce a result. Examples: `limit(n)`, `findFirst()`, `findAny()`, `anyMatch()`, `allMatch()`, `noneMatch()`. E.g., `anyMatch()` stops as soon as one match is found instead of scanning all elements.

**Q13. What is the difference between `Stream<Integer>` and `IntStream`?**
> `Stream<Integer>` boxes each `int` into an `Integer` object (autoboxing overhead). `IntStream` (and `LongStream`, `DoubleStream`) are specialized primitive streams avoiding boxing, and provide extra numeric methods like `sum()`, `average()`, `max()`, `min()` directly without needing a `Collector`.

**Q14. Can you explain a real scenario for using `flatMap()`?**
> Given a `List<Order>` where each `Order` has a `List<Item>`, and you want a single flat `List<Item>` across all orders:
> ```java
> List<Item> allItems = orders.stream()
>         .flatMap(order -> order.getItems().stream())
>         .collect(Collectors.toList());
> ```

**Q15. What's the difference between `peek()` and `map()`?**
> `map()` transforms elements and is meant to produce a new stream of transformed values. `peek()` is meant only for **debugging/inspecting** elements as they flow through the pipeline without altering them — it should not be used to mutate state or implement core logic, and JVM may even skip `peek()` calls if the result isn't used in some optimized cases.

**Q16. How would you count occurrences of words in a `List<String>` sentence list?**
```java
Map<String, Long> wordCount = sentences.stream()
        .flatMap(s -> Arrays.stream(s.split("\\s+")))
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
```

**Q17. What is a `Spliterator`?**
> `Spliterator` ("splittable iterator") is the underlying mechanism Streams use to traverse and **partition** source elements, enabling parallel processing. It supports `tryAdvance()` (sequential) and `trySplit()` (splitting the source for parallel execution).

**Q18. Difference between `sorted()` and `sorted(Comparator)`?**
> No-arg `sorted()` requires elements to implement `Comparable` (natural ordering) and throws `ClassCastException` otherwise. `sorted(Comparator)` allows custom ordering logic, including for objects that don't implement `Comparable`, or to override natural order.

**Q19. How do exceptions work inside a Stream pipeline (e.g. inside `map()`)?**
> Checked exceptions can't be thrown directly inside lambda expressions used in Stream operations since functional interfaces like `Function` don't declare checked exceptions. Common solutions: wrap the checked exception in a `RuntimeException` inside the lambda, use a helper method that catches and rethrows unchecked, or use libraries offering exception-friendly functional interfaces.

**Q20. What's the output and why?**
```java
Stream.of(1, 2, 3)
      .peek(System.out::println)
      .filter(n -> n > 5)
      .findFirst();
```
> Prints `1`, `2`, `3` but the pipeline runs element-by-element (not stage-by-stage): each element flows fully through `peek → filter` before the next one starts. Since `findFirst()` is a short-circuiting terminal op and no element satisfies `filter`, all 3 elements are still processed because none matched — demonstrating **vertical (per-element) execution** in stream pipelines, not horizontal batch execution.

---

## 12. Quick-Reference Cheat Sheet

```
Source          → stream(), Stream.of(), IntStream.range(), Files.lines()
Filter/Transform→ filter(), map(), flatMap(), distinct(), sorted(), limit(), skip()
Terminal        → collect(), forEach(), reduce(), count(), anyMatch(), findFirst()
Collectors      → toList(), toSet(), toMap(), joining(), groupingBy(), partitioningBy()
Parallel        → parallelStream(), .parallel()
Primitive       → IntStream, LongStream, DoubleStream (avoid boxing)
```

---

### Tips for the Interview
- Be ready to **write code on a whiteboard** for grouping, filtering, and reduce-based problems.
- Understand **lazy evaluation** and **short-circuiting** deeply — these are favorite "explain the output" trick questions.
- Know when **NOT** to use streams (readability, small loops, need for early `break`/`continue`-like control flow — use `anyMatch`/`findFirst` instead).
- Practice explaining **`reduce()` with 3 arguments** — commonly confuses candidates in parallel-stream contexts.
