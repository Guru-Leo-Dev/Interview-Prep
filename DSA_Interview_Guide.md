# DSA Interview Guide — Patterns, Problems & Approaches

Organized by **pattern**, not topic, because that's how interviewers actually test you now — they want to see if you recognize *which technique applies*, not whether you memorized a solution. Each entry has: the problem, the core idea, and complexity.

---

## 1. Arrays & Hashing

**Pattern:** Use a hash map/set to trade space for O(1) lookups, turning O(n²) brute force into O(n).

| Problem | Approach | Complexity |
|---|---|---|
| **Two Sum** | Hash map storing `value → index`; check `target - num` as you iterate | O(n) / O(n) |
| **Contains Duplicate** | Insert into a set; if size ≠ array length, duplicates exist | O(n) / O(n) |
| **Group Anagrams** | Key each word by sorted string or 26-char frequency tuple, group in hash map | O(n·k log k) |
| **Top K Frequent Elements** | Count frequencies, then bucket sort by frequency (avoids full sort) | O(n) |
| **Product of Array Except Self** | Prefix product pass + suffix product pass, no division | O(n) / O(1) extra |
| **Valid Sudoku** | Track seen values per row/col/box using sets or bitmasks | O(1) (fixed 9x9) |
| **Longest Consecutive Sequence** | Put all in a set; only start counting from numbers with no `num-1` present | O(n) |

---

## 2. Two Pointers

**Pattern:** For sorted arrays or palindrome-style checks, converge two pointers from ends (or move at different speeds) instead of nested loops.

| Problem | Approach | Complexity |
|---|---|---|
| **Valid Palindrome** | Pointers from both ends, skip non-alphanumeric, compare | O(n) |
| **Two Sum II (sorted array)** | Move left/right pointer based on whether sum is too big/small | O(n) |
| **3Sum** | Sort array, fix one element, two-pointer on the rest, skip duplicates | O(n²) |
| **Container With Most Water** | Move the pointer at the shorter wall inward (moving the taller one can't help) | O(n) |
| **Trapping Rain Water** | Two pointers tracking max height seen from left/right | O(n) / O(1) |

---

## 3. Sliding Window

**Pattern:** Maintain a window with two pointers, expand right, shrink left when a constraint is violated. Avoids recomputation.

| Problem | Approach | Complexity |
|---|---|---|
| **Best Time to Buy/Sell Stock** | Track min price seen so far, update max profit as you scan | O(n) |
| **Longest Substring Without Repeating Characters** | Expand window, shrink left when duplicate found (use set/map of last index) | O(n) |
| **Longest Repeating Character Replacement** | Window valid while `(window size - max freq char) <= k` | O(n) |
| **Permutation in String** | Fixed-size window + frequency count comparison | O(n) |
| **Minimum Window Substring** | Expand until window contains all target chars, then shrink to minimize | O(n) |
| **Sliding Window Maximum** | Monotonic deque holding indices of decreasing values | O(n) |

---

## 4. Stack

**Pattern:** LIFO order fits problems with nested/matching structure or "next greater/smaller" relationships (monotonic stack).

| Problem | Approach | Complexity |
|---|---|---|
| **Valid Parentheses** | Push opens, pop and match on close, fail on mismatch | O(n) |
| **Min Stack** | Auxiliary stack tracking running minimum alongside main stack | O(1) per op |
| **Evaluate Reverse Polish Notation** | Push numbers, pop two on operator, push result | O(n) |
| **Daily Temperatures** | Monotonic decreasing stack of indices; pop when a warmer day is found | O(n) |
| **Largest Rectangle in Histogram** | Monotonic increasing stack; on pop, compute area with popped bar as height | O(n) |

---

## 5. Binary Search

**Pattern:** Any time the search space is monotonic (sorted, or "can I achieve X?" is a yes/no threshold), binary search the answer — not just the array.

| Problem | Approach | Complexity |
|---|---|---|
| **Binary Search (classic)** | Standard `lo/hi/mid` shrink | O(log n) |
| **Search in Rotated Sorted Array** | Determine which half is sorted, decide which half to recurse into | O(log n) |
| **Find Minimum in Rotated Sorted Array** | Compare `mid` to `hi` to decide rotation side | O(log n) |
| **Koko Eating Bananas** | Binary search on the *answer* (eating speed), check feasibility each time | O(n log m) |
| **Median of Two Sorted Arrays** | Binary search on the smaller array's partition point | O(log(min(n,m))) |

---

## 6. Linked List

**Pattern:** Fast/slow pointers for cycle & midpoint problems; dummy nodes to simplify edge cases; reversal via pointer rewiring.

| Problem | Approach | Complexity |
|---|---|---|
| **Reverse Linked List** | Iteratively rewire `next` pointers with `prev/curr` | O(n) / O(1) |
| **Merge Two Sorted Lists** | Dummy head, compare and attach smaller node each step | O(n+m) |
| **Linked List Cycle** | Floyd's fast/slow pointer — they meet iff a cycle exists | O(n) / O(1) |
| **Reorder List** | Find middle (fast/slow) → reverse second half → merge alternately | O(n) |
| **Merge K Sorted Lists** | Min-heap of head nodes, or divide-and-conquer pairwise merge | O(n log k) |
| **Add Two Numbers** | Simulate digit-by-digit addition with carry, dummy head | O(max(n,m)) |
| **Copy List with Random Pointer** | Hash map old→new node, or interweave copies into original list | O(n) |

---

## 7. Trees

**Pattern:** DFS (recursive, often with helper functions returning extra info) for most tree problems; BFS for level-order needs.

| Problem | Approach | Complexity |
|---|---|---|
| **Invert Binary Tree** | Recursively swap left/right children | O(n) |
| **Maximum Depth of Binary Tree** | `1 + max(depth(left), depth(right))` | O(n) |
| **Same Tree / Subtree of Another Tree** | Recursive structural + value comparison | O(n) |
| **Lowest Common Ancestor (BST)** | Use BST ordering to go left/right toward split point | O(h) |
| **Binary Tree Level Order Traversal** | BFS with a queue, process level by level | O(n) |
| **Validate Binary Search Tree** | DFS passing down `(min, max)` bounds | O(n) |
| **Kth Smallest Element in BST** | In-order traversal (gives sorted order), stop at kth | O(h+k) |
| **Construct Tree from Preorder/Inorder** | Preorder gives root; inorder splits left/right subtrees; recurse | O(n) |
| **Binary Tree Maximum Path Sum** | DFS returning max downward path, track global max including "split" paths | O(n) |
| **Serialize/Deserialize Binary Tree** | Preorder DFS with null markers, rebuild recursively | O(n) |

---

## 8. Tries

**Pattern:** Prefix-tree for word storage/lookup problems — much faster than hashing every prefix.

| Problem | Approach | Complexity |
|---|---|---|
| **Implement Trie (Prefix Tree)** | Node with children map + `isEnd` flag | O(k) per op |
| **Design Add and Search Words** | Trie + DFS to handle wildcard `.` characters | O(k) avg |
| **Word Search II** | Build trie of all words, DFS the grid pruning branches not in trie | O(m·n·4^L) worst case |

---

## 9. Heap / Priority Queue

**Pattern:** Need repeated access to min/max, or "k-th" something — heaps beat sorting when you don't need full order.

| Problem | Approach | Complexity |
|---|---|---|
| **Kth Largest Element in Array** | Min-heap of size k, or Quickselect for O(n) average | O(n log k) |
| **Task Scheduler** | Max-heap by frequency + cooldown simulation | O(n log 26) |
| **Find Median from Data Stream** | Two heaps: max-heap for lower half, min-heap for upper half | O(log n) per insert |
| **Merge K Sorted Lists** | (see Linked List) min-heap of current heads | O(n log k) |

---

## 10. Backtracking

**Pattern:** Explore choices via DFS, undo ("backtrack") when a branch fails or completes — for combinations, permutations, constraint satisfaction.

| Problem | Approach | Complexity |
|---|---|---|
| **Subsets** | At each element, branch into include/exclude | O(2^n) |
| **Combination Sum** | DFS choosing candidates, allow reuse, prune when sum exceeds target | O(2^n) |
| **Permutations** | Swap-based or "used[]" tracked DFS building each ordering | O(n!) |
| **Word Search** | DFS/backtrack through grid, mark visited, unmark on backtrack | O(m·n·4^L) |
| **Palindrome Partitioning** | DFS trying every prefix that's a palindrome, recurse on remainder | O(2^n) |
| **N-Queens** | DFS placing queens row by row, prune via column/diagonal sets | O(n!) |

---

## 11. Graphs

**Pattern:** BFS for shortest path/levels, DFS for connectivity/cycles, Union-Find for dynamic connectivity, topological sort for dependency ordering.

| Problem | Approach | Complexity |
|---|---|---|
| **Number of Islands** | DFS/BFS flood-fill, mark visited, count triggers | O(m·n) |
| **Clone Graph** | DFS/BFS with a hash map of original→clone node | O(n) |
| **Course Schedule (cycle detection)** | Topological sort via DFS with 3-color state, or Kahn's BFS algorithm | O(V+E) |
| **Pacific Atlantic Water Flow** | Multi-source BFS/DFS from both borders inward, intersect reachable sets | O(m·n) |
| **Number of Connected Components** | Union-Find with path compression + union by rank | O(E·α(n)) |
| **Graph Valid Tree** | Union-Find: must have exactly n-1 edges and no cycles | O(E·α(n)) |
| **Word Ladder** | BFS over word graph, transform one letter at a time | O(n·26·L) |
| **Dijkstra's Algorithm** | Min-heap of `(distance, node)`, relax edges greedily | O(E log V) |

---

## 12. Dynamic Programming

**Pattern:** Optimal substructure + overlapping subproblems. Identify state, recurrence, base case; solve bottom-up or top-down with memoization.

### 1-D DP
| Problem | Approach | Complexity |
|---|---|---|
| **Climbing Stairs** | `dp[i] = dp[i-1] + dp[i-2]` (Fibonacci pattern) | O(n) |
| **House Robber** | `dp[i] = max(dp[i-1], dp[i-2] + nums[i])` | O(n) |
| **Longest Increasing Subsequence** | `dp[i]` = LIS ending at i, or patience sorting + binary search for O(n log n) | O(n²) or O(n log n) |
| **Coin Change** | `dp[amt] = min(dp[amt - coin] + 1)` over all coins | O(amount · coins) |
| **Word Break** | `dp[i]` = can we segment `s[0:i]`; check all valid `dp[j]` + dictionary word | O(n²) |
| **Decode Ways** | Similar to climbing stairs but with validity checks on 1 or 2-digit codes | O(n) |

### 2-D DP
| Problem | Approach | Complexity |
|---|---|---|
| **Unique Paths** | `dp[i][j] = dp[i-1][j] + dp[i][j-1]` | O(m·n) |
| **Longest Common Subsequence** | `dp[i][j]` matches chars or takes max of skipping one | O(m·n) |
| **Edit Distance** | `dp[i][j]` = min ops; branch on insert/delete/replace | O(m·n) |
| **0/1 Knapsack pattern (Partition Equal Subset Sum)** | `dp[sum]` boolean reachable, iterate items, iterate sum backwards | O(n·sum) |
| **Maximum Product Subarray** | Track both running max **and** min (negatives flip them) | O(n) |

---

## 13. Greedy

**Pattern:** Make the locally optimal choice at each step; requires proving greedy-choice property holds (interviewers often ask "why does greedy work here?").

| Problem | Approach | Complexity |
|---|---|---|
| **Jump Game** | Track farthest reachable index while scanning | O(n) |
| **Gas Station** | If total gas ≥ total cost, a solution exists; track running tank, reset start on deficit | O(n) |
| **Non-overlapping Intervals** | Sort by end time, greedily keep earliest-ending non-overlapping intervals | O(n log n) |

---

## 14. Intervals

**Pattern:** Sort by start (or end) time first — this single step unlocks a linear scan solution for almost all interval problems.

| Problem | Approach | Complexity |
|---|---|---|
| **Insert Interval** | Add all intervals ending before new one, merge overlaps, add rest | O(n) |
| **Merge Intervals** | Sort by start, merge when `current.start <= prev.end` | O(n log n) |
| **Meeting Rooms II** | Sort start/end times separately, sweep line counting concurrent meetings | O(n log n) |

---

## 15. Math & Bit Manipulation

**Pattern:** Constant-space tricks using XOR, bit shifts, and number properties — increasingly common as a way to test if you avoid brute force.

| Problem | Approach | Complexity |
|---|---|---|
| **Single Number** | XOR all elements — pairs cancel out, single value remains | O(n) / O(1) |
| **Number of 1 Bits** | `n & (n-1)` clears lowest set bit; count iterations | O(1) (32 bits) |
| **Counting Bits** | `dp[i] = dp[i >> 1] + (i & 1)` | O(n) |
| **Reverse Bits** | Shift and mask bit by bit into result | O(1) |
| **Missing Number** | XOR of `0..n` with all array elements, or sum formula difference | O(n) |

---

## What's Changed Recently in Interview Trends (2025–2026)

- **Pattern recognition over memorization**: Companies increasingly use novel problems that map to the same underlying pattern (e.g., a "new" sliding window problem you haven't seen) rather than reused LeetCode classics, to filter out pure memorizers.
- **System-design-adjacent DSA**: Problems like LRU Cache, rate limiters, and consistent hashing blend DSA with real system design — increasingly common at mid/senior levels.
- **Follow-up scaling questions**: Interviewers commonly push past the base solution — "what if the input doesn't fit in memory?" or "what if it's a live stream?" — so understanding *why* a data structure works, not just the code, matters more.
- **Practical over puzzle-y**: Companies (especially outside FAANG) have shifted somewhat toward more applied problems — parsing, simulation, and OOP-flavored design (e.g., "design a parking lot") — alongside classic DSA.
- **AI-assisted coding rounds**: Some companies now allow or even expect candidates to use AI tools during live coding but grade on debugging ability, edge-case handling, and explaining trade-offs rather than raw problem-solving speed.

---

## Suggested Study Order

1. Arrays & Hashing → Two Pointers → Sliding Window (fastest ROI, appear everywhere)
2. Binary Search → Stack (unlock a lot of "medium" problems)
3. Linked List → Trees → Tries (core data structure fluency)
4. Heap → Backtracking → Graphs (mid-to-senior level staples)
5. Intervals → Greedy → 1-D DP → 2-D DP (highest difficulty ceiling, worth the most prep time)
6. Bit Manipulation & Math (quick wins, often asked as warm-ups or OA filters)
