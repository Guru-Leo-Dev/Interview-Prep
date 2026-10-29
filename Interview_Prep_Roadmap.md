# Interview Preparation Roadmap — 5 YOE Java/Spring Boot → 15+ LPA
### Backend/Full-Stack Role | 2-Month Plan | 4 hrs/day (~224 hours total)

---

## 0. How To Use This Document

- **Tier ratings (★)** tell you where to spend hours when time is short. ★★★★★ = asked in nearly every interview at this level; ★ = nice-to-have, cut first if behind schedule.
- **Your unique edge**: most 5 YOE Java/Spring candidates cannot talk credibly about RAG pipelines, vector databases, or NL2SQL architecture. You can. At 15+ LPA, companies building AI-adjacent products (and increasingly, everyone is) will pay a premium for someone who's both a solid backend engineer *and* has shipped LLM-based systems. Don't bury this — Week 8 is built around turning it into your strongest differentiator, not a side note.
- **Don't skip DSA** even with 5 YOE — most 15+ LPA product companies (and all FAANG-adjacent ones) still gate on 1-2 DSA rounds regardless of experience. The bar is "solid mediums," not "hard problems in 15 minutes."
- Use the **DSA Interview Guide** and **URL Shortener project** from earlier in this conversation as your DSA and System Design/HLD study companions respectively — this roadmap references both.

---

## 1. Master Tier List (Quick Reference)

| Topic | Priority | % of Prep Time | Why |
|---|---|---|---|
| Core Java (Collections, Streams, Java 17-21 features) | ★★★★★ | 8% | Baseline filter — weak here ends the interview regardless of everything else |
| Concurrency & Multithreading | ★★★★★ | 6% | The #1 place 5 YOE candidates get caught bluffing |
| DSA (Medium-level patterns) | ★★★★★ | 18% | Still gates almost every product company, FAANG or not |
| System Design (HLD) | ★★★★★ | 14% | THE deciding round at 5 YOE — this is where offers are won or lost |
| SQL & Database Design | ★★★★★ | 8% | Every backend role tests this; also feeds System Design answers |
| Spring Boot Core + Security + JPA | ★★★★★ | 10% | Your daily-driver stack — expected to be fluent, not just familiar |
| Microservices Patterns | ★★★★ | 6% | Near-mandatory at 15+ LPA product companies |
| Low-Level Design (LLD) / Machine Coding | ★★★★ | 6% | Common at senior levels — "design a parking lot in code" style rounds |
| AI/LLM Systems (RAG, NL2SQL, Vector DBs) | ★★★★ | 8% | Your differentiator — converts interviews into offers, not just passes |
| Design Patterns (GoF, practical) | ★★★ | 4% | Comes up inside LLD and code review discussions, rarely standalone |
| Caching & Messaging (Redis, Kafka) | ★★★ | 4% | Expected working knowledge, rarely deep-dived unless you claim expertise |
| Cloud & DevOps (Docker, K8s, AWS, CI/CD) | ★★★ | 4% | Working knowledge expected; deep expertise usually not required unless it's a DevOps-heavy role |
| React (Hooks, state, performance) | ★★★ | 4% | Matters because you're targeting full-stack — don't let this go rusty |
| Testing (JUnit, Mockito, TDD) | ★★ | 2% | Usually a "walk me through your testing approach" question, not a coding round |
| Behavioral / STAR stories | ★★★★ | 2% | Under-prepared by almost everyone — cheap points if you're ready |

---

## 2. 8-Week Schedule Overview

| Week | Focus | Hours |
|---|---|---|
| 1 | Core Java + Concurrency + JVM internals | 28 |
| 2 | DSA Foundations (Arrays → Trees) | 28 |
| 3 | DSA Advanced (Graphs, DP) + Spring Boot Core | 28 |
| 4 | Spring Boot Deep Dive + Design Patterns | 28 |
| 5 | Microservices + System Design Fundamentals | 28 |
| 6 | System Design Practice + LLD/Machine Coding | 28 |
| 7 | Databases + Cloud/DevOps + React Refresh | 28 |
| 8 | AI/LLM Specialization + Mock Interviews + Behavioral | 28 |

Each week: **~3.5 hrs/day study + 0.5 hr/day active recall** (flashcards, teach-back, or writing your own summary — this is what actually makes it stick).

---

## 3. Detailed Weekly Plan

### Week 1 — Core Java, Concurrency, JVM (★★★★★)

| Day | Topics |
|---|---|
| 1 | Collections framework internals: HashMap (resize, treeification), ArrayList vs LinkedList, ConcurrentHashMap internals |
| 2 | Java 8-21 features: Streams API, Optional, var, records, sealed classes, pattern matching (switch), virtual threads (Project Loom) |
| 3 | Multithreading basics: Thread lifecycle, synchronized, wait/notify, volatile, thread safety |
| 4 | Concurrency utilities: ExecutorService, CompletableFuture, CountDownLatch, Semaphore, locks (ReentrantLock, ReadWriteLock) |
| 5 | JVM internals: memory model, heap/stack, garbage collection (G1, ZGC), class loading |
| 6 | Practice: write a thread-safe cache from scratch, explain CompletableFuture chaining out loud |
| 7 | Review + flashcards + mock Q&A with yourself (record answers, listen back) |

**Top Interview Questions:**
1. How does HashMap handle collisions, and what changed in Java 8 (treeification)?
2. Difference between `HashMap`, `ConcurrentHashMap`, and `Collections.synchronizedMap` — when would you use each?
3. Explain `volatile` vs `synchronized` — what does each actually guarantee?
4. Walk through what happens when you call `.get()` on a `CompletableFuture` that hasn't completed.
5. What's the difference between checked and unchecked exceptions, and how do you decide which to use in a Spring service layer?
6. Explain Java's memory model — heap generations, and how G1GC differs from the older CMS collector.
7. What are virtual threads (Java 21) and how do they change the concurrency story vs. platform threads?
8. Deep-copy vs shallow-copy — how would you implement `Cloneable` correctly, and why is it considered broken?
9. Explain `equals()` and `hashCode()` contract — what breaks if you override one but not the other?
10. What's a memory leak in Java, and how have you diagnosed one in production?

**Hands-on:** Build a simple thread-safe LRU cache using `LinkedHashMap` + explicit locking, then rebuild it with `ConcurrentHashMap` and compare trade-offs. This single exercise touches collections, concurrency, and design in one go.

**Resources:** *Java Concurrency in Practice* (Brian Goetz) for concurrency depth; Baeldung.com for quick reference on any Java topic; "Java Brains" and "Coding with John" (YouTube) for visual explanations of JVM/GC.

---

### Week 2 — DSA Foundations (★★★★★)

Use the **DSA Interview Guide** from earlier — follow its suggested study order: Arrays/Hashing → Two Pointers → Sliding Window → Stack → Binary Search → Linked List → Trees → Tries.

| Day | Focus | Target |
|---|---|---|
| 1 | Arrays & Hashing | 6-8 problems |
| 2 | Two Pointers + Sliding Window | 6-8 problems |
| 3 | Stack + Binary Search | 6-8 problems |
| 4 | Linked List | 5-6 problems |
| 5 | Trees (traversals, BST) | 6-8 problems |
| 6 | Trees (advanced) + Tries | 5-6 problems |
| 7 | Review: redo 5 problems you struggled with, untimed, from memory |

**Rule of thumb at 5 YOE:** interviewers care more about *clean code, edge-case handling, and clear communication of trade-offs* than raw speed. Narrate your thinking out loud while practicing — this is the actual skill being tested, not just "can you solve it."

**Resources:** LeetCode (filter by company + "medium" difficulty), NeetCode.io for pattern-based video explanations, Striver's SDE Sheet as an alternate curated list.

---

### Week 3 — DSA Advanced + Spring Boot Core (★★★★★)

| Day | Focus |
|---|---|
| 1 | Graphs: BFS/DFS, Number of Islands, Course Schedule, Clone Graph |
| 2 | Graphs: Union-Find, Dijkstra, topological sort |
| 3 | Dynamic Programming: 1-D (climbing stairs, house robber, coin change) |
| 4 | Dynamic Programming: 2-D (LCS, edit distance, knapsack pattern) |
| 5 | Spring Boot Core: bean lifecycle, dependency injection, `@Component` vs `@Service` vs `@Repository`, auto-configuration internals |
| 6 | Spring Boot: REST API design, exception handling (`@ControllerAdvice`), validation, request/response DTOs |
| 7 | Review DSA weak spots + Spring Boot core concepts |

**Top Spring Boot Core Questions:**
1. Explain the Spring Bean lifecycle — from instantiation to destruction.
2. What's the difference between `@Autowired` field injection, constructor injection, and setter injection — why is constructor injection preferred?
3. How does Spring Boot auto-configuration actually work under the hood (`@EnableAutoConfiguration`, `spring.factories`/`AutoConfiguration.imports`)?
4. What's the difference between `@Component`, `@Service`, `@Repository`, `@Controller` — functionally, are they different?
5. How do you handle global exception handling in a Spring Boot REST API?
6. Explain `@Transactional` — propagation levels, isolation levels, and a scenario where the default behavior would bite you.
7. What's the difference between `@RequestParam`, `@PathVariable`, and `@RequestBody`?
8. How would you version a REST API, and what are the trade-offs of each approach (URL versioning vs header versioning)?
9. What's a Bean scope, and when would you use `prototype` over the default `singleton`?
10. How does Spring Boot's embedded Tomcat differ from deploying a WAR to an external server — why did the industry move this direction?

**Resources:** Spring's official docs (docs.spring.io) are unusually good — use them directly rather than tutorials for anything you're fuzzy on. Baeldung for quick how-tos.

---

### Week 4 — Spring Boot Deep Dive + Design Patterns (★★★★★ / ★★★)

| Day | Focus |
|---|---|
| 1 | Spring Security: authentication vs authorization, JWT flow, filter chain, `SecurityFilterChain` config |
| 2 | Spring Data JPA: entity mapping, `@OneToMany`/`@ManyToMany`, N+1 problem, lazy vs eager loading |
| 3 | Transactions deep dive: ACID, isolation levels, distributed transaction problems (segues into microservices week) |
| 4 | Design Patterns: Singleton, Factory, Builder, Strategy — where Spring itself uses each |
| 5 | Design Patterns: Observer, Decorator, Proxy, Adapter — with Spring AOP as the running example for Proxy |
| 6 | Hands-on: refactor a messy service class using 2-3 patterns from this week |
| 7 | Review + write your own cheat-sheet summary of Spring Security's filter chain from memory |

**Top Questions:**
1. Walk through what happens end-to-end when a JWT-authenticated request hits a Spring Boot API.
2. What's the N+1 query problem, and how do you fix it in JPA (`@EntityGraph`, `JOIN FETCH`, batch fetching)?
3. Explain isolation levels (Read Uncommitted → Serializable) and what problem each one prevents (dirty read, non-repeatable read, phantom read).
4. When would you choose `@Transactional(propagation = REQUIRES_NEW)` over the default `REQUIRED`?
5. How does Spring AOP work, and which design pattern is it fundamentally built on (Proxy)?
6. Explain the Singleton pattern's pitfalls in a multithreaded environment, and how Spring's singleton beans avoid them.
7. Give an example from your own projects where you used (or should have used) the Strategy pattern.
8. What's the difference between `@Modifying` queries and standard JPA repository methods — when do you need the former?
9. How do you secure sensitive fields (e.g., passwords, PII) at the entity/DB level in a Spring Boot app?
10. Optimistic vs pessimistic locking — when would each apply in an inventory/ERP context (relevant to your background)?

**Hands-on:** Add JWT-based auth + role-based access control to the URL Shortener project from earlier — this gives you a concrete "I built this" story that touches Security, JPA, and REST design together.

**Resources:** *Head First Design Patterns* for intuition-building (skim, don't deep-read); Spring Security reference docs; Baeldung's JPA N+1 articles.

---

### Week 5 — Microservices + System Design Fundamentals (★★★★ / ★★★★★)

| Day | Focus |
|---|---|
| 1 | Microservices patterns: API Gateway, Service Discovery, Circuit Breaker (Resilience4j), Config Server |
| 2 | Inter-service communication: REST vs gRPC vs async messaging, synchronous vs asynchronous trade-offs |
| 3 | Distributed transactions: Saga pattern (choreography vs orchestration), eventual consistency |
| 4 | System Design fundamentals: load balancing, horizontal vs vertical scaling, CAP theorem, consistent hashing |
| 5 | System Design fundamentals: caching strategies, database replication/sharding, message queues |
| 6 | Re-study the URL Shortener project's README as a worked HLD example — be ready to present it end-to-end unprompted |
| 7 | Review: write out the Saga pattern and Circuit Breaker pattern from memory, no notes |

**Top Microservices Questions:**
1. How do services discover each other in a microservices architecture (Eureka/Consul vs. Kubernetes-native DNS)?
2. What's the Circuit Breaker pattern, and how does Resilience4j implement it (closed/open/half-open states)?
3. Explain the Saga pattern — when would you use choreography vs. orchestration?
4. How do you handle distributed transactions when you can't use a single DB transaction across services?
5. What's the difference between API Gateway and a Load Balancer — do you need both?
6. How would you handle a service that's down when another service depends on it synchronously? (Retry, fallback, circuit breaker, async decoupling)
7. Explain eventual consistency with a concrete example from an e-commerce or inventory system.
8. What's the outbox pattern, and what problem does it solve with events + DB writes?
9. How do you version APIs across microservices without breaking existing consumers?
10. Monolith vs. microservices — when would you argue *against* microservices in an interview? (Shows maturity, not dogma.)

**Top System Design (HLD) Fundamentals Questions:**
1. Explain CAP theorem with a real system example — which two would you pick for a payment system vs. a social feed?
2. How does consistent hashing work, and why is it better than simple modulo-based sharding for a distributed cache?
3. Walk through cache-aside vs write-through vs write-behind caching strategies.
4. How would you design a system to handle 10x traffic growth without a rewrite?
5. What's the difference between horizontal and vertical scaling, and where does each hit a wall?

**Resources:** *Designing Data-Intensive Applications* (Martin Kleppmann) — the single best book for this; ByteByteGo (YouTube/newsletter) for visual system design explanations; Resilience4j docs.

---

### Week 6 — System Design Practice + LLD (★★★★★ / ★★★★)

Practice designing these end-to-end (verbally + on a whiteboard/paper), 1 per session:

| Day | System to Design |
|---|---|
| 1 | URL Shortener (you already have this — present it cold, then add: custom domains, analytics dashboard) |
| 2 | Rate Limiter (design the algorithm choices: token bucket vs sliding window vs fixed window) |
| 3 | E-commerce order/inventory system (leverage your actual domain background here — this is a strength) |
| 4 | Notification system (push/email/SMS, fan-out, retry logic) |
| 5 | LLD: Design a parking lot / elevator system in code (classic machine coding round) |
| 6 | LLD: Design an inventory management system in code (OOP, relevant to your background — strong differentiator) |
| 7 | Review: redo Day 3 or Day 6 from scratch without looking at your notes |

**Framework for every HLD answer** (say this structure out loud in interviews):
1. Clarify requirements (functional + non-functional) — **always start here, don't skip it**
2. Capacity estimation (back-of-envelope numbers)
3. API design
4. High-level architecture diagram
5. Deep dive into 1-2 hard parts (usually: data model + one scaling bottleneck)
6. Trade-offs and what you'd do differently at 10x scale

**Top LLD Questions:**
1. Design a parking lot system — classes, relationships, and how you'd handle different vehicle types.
2. Design an elevator system — how do you handle multiple elevators and request scheduling?
3. Design a library management system with book checkout/return and fines.
4. Design an inventory management system — how do you model stock across multiple warehouses (directly usable from your ERP background)?
5. Design a rate limiter as a reusable library/annotation.

**Resources:** *Grokking the System Design Interview* (Educative) for structured practice sets; "Gaurav Sen" and "System Design Interview" (YouTube) for walked examples; the URL Shortener project you already built is your strongest LLD+HLD combined artifact — know it cold.

---

### Week 7 — Databases + Cloud/DevOps + React (★★★★★ / ★★★ / ★★★)

| Day | Focus |
|---|---|
| 1 | SQL deep dive: joins, window functions, query optimization, EXPLAIN plans |
| 2 | Indexing: B-tree indexes, composite indexes, covering indexes, when indexes hurt write performance |
| 3 | NoSQL basics: when to choose document/key-value/column-store over relational, CAP trade-offs revisited |
| 4 | Cloud & DevOps: Docker fundamentals (images, layers, multi-stage builds), Kubernetes basics (pods, services, deployments) |
| 5 | CI/CD pipeline concepts, basic AWS services (EC2, S3, RDS, Lambda) at a working-knowledge level |
| 6 | React refresh: hooks (`useState`, `useEffect`, `useMemo`, `useCallback`), state management, performance (memoization, virtualization) |
| 7 | Review + hands-on: containerize the URL Shortener project with Docker (adds a concrete DevOps talking point) |

**Top Database Questions:**
1. How does a B-tree index actually speed up a query — walk through it.
2. When would a composite index not be used by the query planner (leftmost prefix rule)?
3. Explain the difference between clustered and non-clustered indexes.
4. Write a query using a window function to get, e.g., "top 3 orders per customer."
5. How would you optimize a slow query — what's your diagnostic process (EXPLAIN, index review, query rewrite)?
6. When would you denormalize a schema, and what do you trade away?
7. Explain database replication (leader-follower) and what problems replication lag causes.

**Top Cloud/DevOps Questions:**
1. What's the difference between a Docker image and a container?
2. Explain a Kubernetes Pod, Service, and Deployment — how do they relate?
3. What's a multi-stage Docker build, and why does it matter for image size?
4. Walk through a basic CI/CD pipeline you've worked with — what triggers deployment?
5. When would you choose serverless (Lambda) over a traditional deployed service?

**Top React Questions:**
1. Explain `useEffect`'s dependency array — what breaks if you get it wrong?
2. When would you use `useMemo` vs `useCallback`, and what performance problem does each solve?
3. How does React's reconciliation (virtual DOM diffing) actually work?
4. Controlled vs. uncontrolled components — trade-offs?
5. How would you handle global state — Context API vs. Redux/Zustand — and when is each overkill?

**Resources:** "Use The Index, Luke" (free online) for indexing depth; Docker/Kubernetes official docs for fundamentals; React official docs (react.dev) — significantly improved and now the best React resource available.

---

### Week 8 — AI/LLM Specialization + Mock Interviews + Behavioral (★★★★ / ★★★★)

This is where your background stops being "extra" and becomes your positioning. Companies at 15+ LPA building AI features want backend engineers who *get* LLM systems — not ML researchers, but engineers who can productionize them. That's exactly your last two roles.

| Day | Focus |
|---|---|
| 1 | RAG architecture deep dive: chunking strategies, embedding models, retrieval (dense vs. hybrid), re-ranking |
| 2 | Vector databases: how they index (HNSW, IVF), when to use pgvector vs. a dedicated vector DB (Pinecone/Weaviate/Milvus) |
| 3 | NL2SQL architecture: schema-aware prompting, handling ambiguity, validation/guardrails against bad generated SQL, execution safety |
| 4 | LLM system design: prompt engineering patterns, function calling/tool use, handling hallucination, cost/latency trade-offs, evaluation strategy |
| 5 | Prepare your own project narratives (STAR format) for both RAG chatbot and NL2SQL — architecture, challenges, trade-offs, metrics |
| 6 | Full mock interview #1: 1 DSA + 1 System Design (record yourself, review critically) |
| 7 | Full mock interview #2: 1 LLD + Behavioral round, plus resume/LinkedIn final pass |

**Top AI/LLM System Questions (increasingly common at senior backend interviews now):**
1. Walk me through the architecture of a RAG system you built — where does it break, and how did you handle that?
2. How do you decide chunk size and overlap when indexing documents for retrieval?
3. What's the difference between dense retrieval, sparse (BM25) retrieval, and hybrid — when would you combine them?
4. How would you design an NL2SQL system to prevent it from generating destructive or incorrect queries against a production database?
5. How do you evaluate RAG output quality — what metrics or approaches have you used (faithfulness, relevance, human eval)?
6. How do you handle a schema with hundreds of tables in an NL2SQL system without blowing the context window?
7. What's your approach to reducing hallucination in a system that must be factually grounded?
8. How would you architect an LLM-based feature to handle rate limits, timeouts, and provider fallback in production?
9. Vector DB indexing — explain HNSW at a high level, and why approximate nearest neighbor search is the right trade-off here.
10. How do you think about cost control when a feature depends on LLM API calls at scale?

**Hands-on:** Write up a 1-page architecture doc for your RAG chatbot and NL2SQL system as if explaining it to a new teammate — this becomes your interview script, and forces you to find the gaps in your own explanation before an interviewer does.

**Behavioral Prep — Build 5-6 STAR Stories Covering:**
- A time you resolved a production incident under pressure
- A time you disagreed with a technical decision and how you handled it
- A time you had to learn something completely new quickly (your AI-team transition is a strong one here)
- A time you mentored someone or influenced a team decision
- Your biggest technical mistake and what you changed afterward
- Why you're moving roles / what you're looking for now

**Resources:** Anthropic's and OpenAI's own engineering blogs for production LLM system patterns; "LLM System Design" content from ByteByteGo has started covering this well; your own project code/docs are your best resource this week — you lived it.

---

## 4. Domain Background — How To Use It

Your Supply Chain / Inventory / ERP domain knowledge is a real asset, not just resume filler:
- In **System Design rounds**, default to inventory/order/warehouse examples when given a free choice of system — you'll go deeper faster than with a generic example.
- In **behavioral rounds**, tie your AI-team move to business impact you understood because of your domain background (e.g., "I knew inventory forecasting was a bottleneck, which is why NL2SQL for ad-hoc supply chain queries mattered").
- In **LLD rounds**, an inventory/warehouse system is a strong default choice if the interviewer says "design anything you're familiar with."

---

## 5. Final Pre-Interview Checklist (Last 2-3 Days)

- [ ] Can you present the URL Shortener HLD cold, in under 5 minutes, without notes?
- [ ] Can you explain your RAG and NL2SQL architectures in under 3 minutes each?
- [ ] Do you have 5-6 STAR stories ready without hesitation?
- [ ] Have you done at least 2 full mock interviews (recorded, self-reviewed)?
- [ ] Resume: does every bullet point have a number (latency, scale, % improvement)?
- [ ] Are you prepared to ask 2-3 sharp questions back to each interviewer?

---

## 6. If You Fall Behind Schedule

Cut in this order, protecting the ★★★★★ items first:
1. Cut React depth to "can answer basic questions" only
2. Cut Cloud/DevOps to "conceptual understanding" only, skip hands-on Docker exercise
3. Cut Design Patterns study time in half — you'll still absorb some through LLD practice
4. **Never cut**: DSA practice, System Design practice, Core Java/Concurrency, or the AI/LLM week — these four are what separate 15+ LPA offers from lower ones at your experience level.
