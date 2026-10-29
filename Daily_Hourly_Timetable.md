# Daily Hour-by-Hour Timetable — 8-Week Interview Prep

**Default split:** Hour 1-2 = morning block, Hour 3-4 = evening block (adjust to your day — order matters more than clock time). Hour 4 is deliberately review/recall in most days — this is spaced repetition, don't skip it even when tempted to push forward.

Legend: 🧠 New concept · 💻 Practice/coding · 🔁 Active recall (no notes allowed)

---

## WEEK 1 — Core Java, Concurrency, JVM (★★★★★)

| Day | Hour 1 | Hour 2 | Hour 3 | Hour 4 |
|---|---|---|---|---|
| 1 | 🧠 HashMap internals: resize, treeification (Java 8+) | 🧠 ArrayList vs LinkedList internals; ConcurrentHashMap internals | 💻 Answer Q1-3 from Week 1 list in writing; code a tiny custom bucket-collision demo | 🔁 Recall: explain HashMap resize from memory |
| 2 | 🧠 Streams API: map/filter/reduce/collect + common pitfalls | 🧠 Optional, var, records, sealed classes, switch pattern matching | 💻 Code a virtual threads (Java 21) mini demo | 🔁 Recall Day 1 + Day 2 combined |
| 3 | 🧠 Thread lifecycle + `synchronized` deep dive | 🧠 `wait`/`notify`/`notifyAll` — build a producer-consumer example | 💻 Spot race conditions in 3 sample code snippets | 🔁 Recall: write a thread-safety checklist from memory |
| 4 | 🧠 ExecutorService + thread pool types | 🧠 `CompletableFuture` chaining (`thenApply`/`thenCompose`/`allOf`) | 💻 Code `CountDownLatch`, `Semaphore`, `ReentrantLock` demos | 🔁 Explain `CompletableFuture` chaining out loud, unprompted |
| 5 | 🧠 JVM memory model: heap generations, stack vs heap | 🧠 GC deep dive: G1 vs ZGC, when each applies | 🧠 Class loading process + classloader hierarchy | 🔁 Recall Days 1-5 (quiz yourself on all 10 top questions) |
| 6 | 💻 Build thread-safe LRU cache (LinkedHashMap + explicit locking) | 💻 Rebuild it using `ConcurrentHashMap`, compare trade-offs | 💻 Write answers to all 10 Week 1 questions from memory | 🔁 Record yourself answering 3 random Week 1 questions cold |
| 7 | 🔁 Full flashcard review, Week 1 | 💻 Redo your weakest coding exercise from the week | 📝 Write a 1-page "Core Java & Concurrency" cheat sheet from memory | 🔁 Preview Week 2 DSA patterns (skim only) |

---

## WEEK 2 — DSA Foundations (★★★★★)

| Day | Hour 1 | Hour 2 | Hour 3 | Hour 4 |
|---|---|---|---|---|
| 1 | 🧠 Arrays/Hashing pattern + 1 warm-up problem | 💻 Two Sum, Contains Duplicate, Group Anagrams | 💻 Top K Frequent, Product Except Self, Longest Consecutive Sequence | 🔁 Redo hardest problem from memory; note the "trigger words" that signal this pattern |
| 2 | 🧠 Two Pointers pattern + Valid Palindrome, Two Sum II | 💻 3Sum, Container With Most Water | 🧠 Sliding Window pattern + Best Time to Buy/Sell Stock | 💻+🔁 Longest Substring w/o Repeat, Longest Repeating Char Replace; redo yesterday's hardest cold |
| 3 | 🧠 Stack pattern + Valid Parentheses, Min Stack | 💻 Monotonic stack: Daily Temperatures, Largest Rectangle in Histogram | 🧠 Binary Search pattern + classic + Search in Rotated Sorted Array | 💻+🔁 Koko Eating Bananas; recall Day 1-3 |
| 4 | 💻 Reverse Linked List, Merge Two Sorted Lists | 💻 Linked List Cycle (Floyd's), Reorder List | 💻 Merge K Sorted Lists, Add Two Numbers | 🔁 Redraw pointer diagrams for 2 problems from memory |
| 5 | 🧠 DFS/BFS traversal + Invert Binary Tree, Max Depth | 💻 Validate BST, Lowest Common Ancestor | 💻 Level Order Traversal, Kth Smallest in BST | 🔁 Explain BST invariant out loud, no notes |
| 6 | 💻 Construct Tree from Preorder/Inorder | 💻 Binary Tree Max Path Sum, Serialize/Deserialize Tree | 🧠 Trie implementation + Word Search II | 🔁 Full week recall quiz |
| 7 | 💻 Redo 2 struggled problems, untimed | 💻 Redo 2 more struggled problems, untimed | 💻 Redo 1 final struggled problem | 📝 Write Week 2 pattern cheat sheet from memory |

---

## WEEK 3 — DSA Advanced + Spring Boot Core (★★★★★)

| Day | Hour 1 | Hour 2 | Hour 3 | Hour 4 |
|---|---|---|---|---|
| 1 | 🧠 BFS/DFS graph theory + Number of Islands | 💻 Course Schedule (cycle detection) | 💻 Clone Graph | 🔁 Redo Number of Islands from memory |
| 2 | 🧠 Union-Find theory + Number of Connected Components | 💻 Graph Valid Tree | 🧠 Topological sort (Kahn's BFS) + Dijkstra's algorithm | 🔁 Recall Day 1-2 graph patterns |
| 3 | 🧠 DP framework (state/recurrence/base case) + Climbing Stairs, House Robber | 💻 Longest Increasing Subsequence (O(n²) then O(n log n)) | 💻 Coin Change, Word Break | 🔁 Redo Coin Change from memory |
| 4 | 💻 Unique Paths, Longest Common Subsequence | 💻 Edit Distance | 💻 Partition Equal Subset Sum, Max Product Subarray | 🔁 Recall full DP week |
| 5 | 🧠 Bean lifecycle + DI types (why constructor injection wins) | 🧠 `@Component` vs `@Service` vs `@Repository` vs `@Controller` — write comparison table | 🧠 Auto-configuration internals (`AutoConfiguration.imports`) | 🔁 Answer Spring Q1-4 from memory |
| 6 | 🧠 REST API design best practices, DTOs | 💻 Global exception handling with `@ControllerAdvice` — code it | 🧠 Validation (`@Valid`, custom validators) | 🔁 Redo Spring core Q1-10 unprompted |
| 7 | 💻 Redo weakest DSA problem from Weeks 2-3 | 💻 Redo second weakest DSA problem | 🔁 Redo Spring Boot core Q&A cold | 📝 Cheat sheet: "Bean Lifecycle + Graph/DP patterns" |

---

## WEEK 4 — Spring Boot Deep Dive + Design Patterns (★★★★★ / ★★★)

| Day | Hour 1 | Hour 2 | Hour 3 | Hour 4 |
|---|---|---|---|---|
| 1 | 🧠 AuthN vs AuthZ + JWT flow diagram | 🧠 `SecurityFilterChain` config — study a real example | 💻 Code a JWT auth skeleton in a sample project | 🔁 Explain full JWT flow end-to-end from memory |
| 2 | 🧠 Entity mapping, `@OneToMany`/`@ManyToMany` | 💻 Reproduce the N+1 problem in a sample project | 💻 Fix it with `@EntityGraph` / `JOIN FETCH` / batch fetching | 🔁 Explain N+1 problem + fix out loud |
| 3 | 🧠 ACID properties + isolation levels theory | 💻 Practice dirty read / non-repeatable read / phantom read scenarios | 🧠 Distributed transaction problems (primes you for Week 5) | 🔁 Write the isolation-level table from memory |
| 4 | 🧠 Singleton (+ multithreading pitfalls), Factory | 💻 Builder pattern — code it | 🧠 Strategy pattern — map to a real example from your projects | 🔁 Write pattern-to-use-case mapping from memory |
| 5 | 🧠 Observer, Decorator — code examples | 🧠 Proxy pattern — connect it to Spring AOP mechanics | 💻 Adapter pattern — code it | 🔁 Recall Day 4-5 patterns |
| 6 | 💻 Refactor a messy service class using 2-3 patterns (use URL Shortener or your own code) | 💻 Continue refactor | 💻 Continue refactor | 🔁 Explain your refactor decisions out loud |
| 7 | 💻 Add JWT + RBAC to URL Shortener project | 💻 Continue JWT/RBAC integration | 🔁 Redo N+1 fix demo from memory | 📝 Draw Spring Security filter chain from memory |

---

## WEEK 5 — Microservices + System Design Fundamentals (★★★★ / ★★★★★)

| Day | Hour 1 | Hour 2 | Hour 3 | Hour 4 |
|---|---|---|---|---|
| 1 | 🧠 API Gateway + Service Discovery (Eureka/Consul vs K8s DNS) | 🧠 Circuit Breaker pattern — Resilience4j states | 💻 Code a simple Resilience4j circuit breaker demo | 🔁 Explain closed/open/half-open states from memory |
| 2 | 🧠 REST vs gRPC vs async messaging trade-offs | 🧠 Sync vs async — failure mode analysis | 💻 Design communication for an order↔inventory service pair | 🔁 Recall Day 1-2 |
| 3 | 🧠 Saga pattern: choreography vs orchestration | 🧠 Eventual consistency + outbox pattern | 💻 Design a Saga for an e-commerce order flow (use your domain background) | 🔁 Draw the Saga flow diagram from memory |
| 4 | 🧠 Load balancing algorithms, horizontal vs vertical scaling | 🧠 CAP theorem with real examples (payments vs. social feed) | 🧠 Consistent hashing — why it beats modulo sharding | 🔁 Recall CAP trade-offs from memory |
| 5 | 🧠 Caching strategies: cache-aside, write-through, write-behind | 🧠 DB replication + sharding strategies | 🧠 Message queues: when/why | 🔁 Recall Day 4-5 |
| 6 | 📖 Re-read URL Shortener README fully | 📝 Rebuild its architecture diagram from memory | 🔁 Present it out loud end-to-end, record yourself, time it (<5 min target) | 🔁 Identify weak spots, redo those sections |
| 7 | 🔁 Write Saga pattern from memory, no notes | 🔁 Write Circuit Breaker states from memory | 🔁 Full week self-quiz | 📖 Preview Week 6 system design problem list |

---

## WEEK 6 — System Design Practice + LLD (★★★★★ / ★★★★)

| Day | Hour 1 | Hour 2 | Hour 3 | Hour 4 |
|---|---|---|---|---|
| 1 | 🔁 Present URL Shortener cold, timed | 💻 Design extension: custom domains | 💻 Design extension: analytics dashboard | 🔁 Refine weak points from Hour 1 |
| 2 | 💻 Rate Limiter: requirements + capacity estimation | 🧠 Token bucket vs sliding window vs fixed window — compare all 3 | 💻 Design it as a reusable library/annotation | 🔁 Explain all 3 algorithms from memory |
| 3 | 💻 E-commerce order/inventory: requirements + capacity estimation | 💻 Data model design (lean on your domain background) | 💻 Scaling deep dive: inventory consistency at scale | 🔁 Present the design cold |
| 4 | 💻 Notification system: requirements + fan-out architecture | 💻 Push/email/SMS provider abstraction, retry logic | 💻 Full end-to-end design practice | 🔁 Recall + self-critique |
| 5 | 🧠 Parking lot: class design, OOP relationships | 💻 Code the core classes in Java | 🧠 Elevator system: scheduling algorithm discussion | 🔁 Explain class diagram from memory |
| 6 | 💻 Inventory management: requirements + class design (multi-warehouse) | 💻 Code the core classes | 🧠 Discuss edge cases: concurrent stock updates, reservations | 🔁 Present design cold |
| 7 | 💻 Redo Day 3 or Day 6 from scratch, no notes | 💻 Continue redo | 💻 Finish redo | 📝 Compare to original, log gaps, update cheat sheet |

---

## WEEK 7 — Databases + Cloud/DevOps + React (★★★★★ / ★★★ / ★★★)

| Day | Hour 1 | Hour 2 | Hour 3 | Hour 4 |
|---|---|---|---|---|
| 1 | 🧠 Joins deep dive (inner/outer/cross) + practice queries | 💻 Window functions — write 3-4 queries | 🧠 Query optimization + reading an EXPLAIN plan | 🔁 Write a window function query from memory |
| 2 | 🧠 B-tree index mechanics | 🧠 Composite indexes + leftmost prefix rule | 🧠 Covering indexes; when indexes hurt writes | 🔁 Explain B-tree speed-up from memory |
| 3 | 🧠 Document/key-value/column-store — when to choose each | 🧠 CAP trade-offs revisited in NoSQL context | 💻 Practice: DynamoDB/Cassandra vs Postgres scenario decisions | 🔁 Recall Day 1-3 DB week |
| 4 | 🧠 Docker fundamentals: images, layers, multi-stage builds | 🧠 Kubernetes basics: pods, services, deployments | 💻 Containerize a small Spring Boot app | 🔁 Explain Pod/Service/Deployment relationship from memory |
| 5 | 🧠 CI/CD pipeline concepts: stages, triggers | 🧠 AWS basics: EC2, S3, RDS | 🧠 AWS Lambda — when serverless makes sense | 🔁 Recall Day 4-5 |
| 6 | 🧠 Hooks: `useState`, `useEffect` deep dive (dependency array pitfalls) | 🧠 `useMemo`/`useCallback` — performance use cases | 🧠 Context API vs Redux/Zustand trade-offs | 🔁 Explain virtual DOM reconciliation from memory |
| 7 | 💻 Dockerize URL Shortener (multi-stage build) | 💻 Continue Docker setup | 💻 Verify it runs end-to-end in a container | 🔁 Full week recall quiz |

---

## WEEK 8 — AI/LLM Specialization + Mock Interviews + Behavioral (★★★★ / ★★★★)

| Day | Hour 1 | Hour 2 | Hour 3 | Hour 4 |
|---|---|---|---|---|
| 1 | 🧠 Chunking strategies, embedding models | 🧠 Dense vs hybrid (BM25 + dense) retrieval | 🧠 Re-ranking — map every concept to your actual RAG chatbot project | 🔁 Explain the full RAG pipeline end-to-end from memory |
| 2 | 🧠 HNSW / IVF indexing — how approximate nearest neighbor works | 🧠 pgvector vs Pinecone/Weaviate/Milvus trade-offs | 💻 Practice explaining your own project's vector DB choice and why | 🔁 Recall Day 1-2 |
| 3 | 🧠 Schema-aware prompting techniques | 🧠 Handling ambiguity in natural language queries | 🧠 Guardrails against destructive/incorrect generated SQL | 🔁 Explain your NL2SQL safety guardrails from memory |
| 4 | 🧠 Prompt engineering patterns, function calling/tool use | 🧠 Handling hallucination; cost/latency trade-offs | 🧠 Evaluation strategy (faithfulness, relevance, human eval) | 🔁 Explain your cost-control approach from memory |
| 5 | 📝 Write RAG chatbot STAR narrative (architecture, challenges, trade-offs, metrics) | 📝 Write NL2SQL STAR narrative | 🔁 Practice presenting both, timed (<3 min each) | 🔁 Refine based on self-review |
| 6 | 🔁 Mock: 1 DSA problem, timed, recorded | 📝 Review DSA performance, note gaps | 🔁 Mock: 1 System Design problem, recorded | 📝 Review System Design performance, note gaps |
| 7 | 🔁 Mock: 1 LLD problem, recorded | 🔁 Mock: Behavioral round (run through all 5-6 STAR stories) | 📝 Final resume pass — every bullet gets a number | 📝 Final LinkedIn pass + confidence check |

---

## Notes on Using This Table

- **Hour 4 is not optional.** It's the highest-leverage hour in the whole plan — spaced recall is what moves things from "I read about this" to "I can answer this cold under pressure," which is the actual bar in an interview.
- If a topic runs long, let it eat into Hour 4 rather than skip the next day's Hour 1-3 — better to truly finish a concept than half-cover two.
- Every Day 7 is intentionally lighter on new material and heavier on consolidation — don't fill that slack with new topics, your retention depends on it.
- Swap any 🧠 block for a video/article and any 💻 block for LeetCode/your own IDE — the format (read → practice → recall) matters more than the specific resource.
