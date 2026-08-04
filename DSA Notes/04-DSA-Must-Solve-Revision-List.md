# DSA Revision — 75 Must-Solve Problems, Topic-Wise

> **Who this is for:** someone who has already learned this material and is revising. So the value here isn't the problem list — it's the **recognition cues** (what in the problem statement tells you which pattern to reach for) and the **traps** (the specific thing you get wrong at 2 a.m.). Read the notes, then solve.

---

## How to use this

1. **Cover the notes and solve cold.** If you read the pattern first, you're testing recall of the solution, not recall of the *trigger*. The trigger is the skill that transfers.
2. **20-minute rule.** If you're stuck at 20 minutes, read the approach only — not the code — and try again. If still stuck at 35, read the solution, then **re-solve from scratch the next day**.
3. **Mark every problem** as ✅ (clean solve), 🟡 (solved with a hint / had bugs), or ❌ (needed the solution). Re-do 🟡 after 3 days and ❌ after 1 day, then again after a week.
4. **Say the complexity out loud before you code.** If you can't state the target complexity, you don't have an approach yet — you have a vibe.
5. **Write the pattern name in a comment at the top** of every solution. After 75 problems you want ~18 named patterns in your head, not 75 memorised solutions.

**Progress:** 0 / 75

---

## Table of Contents

| # | Topic | Problems |
|---|---|---|
| 1 | [Arrays & Hashing](#1-arrays--hashing-6) | 6 |
| 2 | [Two Pointers](#2-two-pointers-4) | 4 |
| 3 | [Sliding Window](#3-sliding-window-4) | 4 |
| 4 | [Binary Search](#4-binary-search-5) | 5 |
| 5 | [Stack & Monotonic Stack](#5-stack--monotonic-stack-4) | 4 |
| 6 | [Linked List](#6-linked-list-5) | 5 |
| 7 | [Trees & BST](#7-trees--bst-8) | 8 |
| 8 | [Heap / Priority Queue](#8-heap--priority-queue-3) | 3 |
| 9 | [Backtracking](#9-backtracking-4) | 4 |
| 10 | [Graphs — BFS / DFS / Dijkstra](#10-graphs--bfs--dfs--dijkstra-5) | 5 |
| 11 | [Topological Sort](#11-topological-sort-2) | 2 |
| 12 | [Union-Find](#12-union-find-2) | 2 |
| 13 | [1-D Dynamic Programming](#13-1-d-dynamic-programming-6) | 6 |
| 14 | [2-D / String DP](#14-2-d--string-dp-4) | 4 |
| 15 | [Greedy](#15-greedy-3) | 3 |
| 16 | [Intervals](#16-intervals-3) | 3 |
| 17 | [Bit Manipulation](#17-bit-manipulation-3) | 3 |
| 18 | [Tries](#18-tries-2) | 2 |
| 19 | [Matrix Manipulation](#19-matrix-manipulation-2) | 2 |
| | **Total** | **75** |

---

## 1. Arrays & Hashing (6)

### When you see this
"Find pairs/groups that…", "count occurrences", "does X exist", "subarray summing to K", **and** the brute force is O(n²) with a repeated lookup inside the inner loop. **The move: trade space for time — a `HashMap` turns the inner loop into O(1).**

The specific tell for **prefix sums**: the problem says *subarray* (contiguous) and *sum*. `sum(i..j) = prefix[j] - prefix[i-1]`, so "find a subarray with sum K" becomes "find a previously-seen prefix equal to `current - K`" — which is just Two Sum on prefixes.

### Template

```java
// Prefix sum + hashmap: count subarrays summing to k.  O(n) time, O(n) space.
Map<Integer, Integer> countByPrefix = new HashMap<>();
countByPrefix.put(0, 1);              // ← the empty prefix. Forgetting this is THE bug.
int prefix = 0, answer = 0;
for (int x : nums) {
    prefix += x;
    answer += countByPrefix.getOrDefault(prefix - k, 0);
    countByPrefix.merge(prefix, 1, Integer::sum);
}
```

### Traps
- **Forgetting `map.put(0, 1)`** in prefix-sum counting — you lose every subarray that starts at index 0.
- Using a `HashSet` when you need indices (`Two Sum` returns indices, not values).
- **Sorting when you didn't need to** — it's O(n log n) and often destroys the index information you need.
- For "no division allowed" product problems: the answer is prefix products from the left × suffix products from the right, in two passes.
- `41. First Missing Positive` is the odd one out — O(1) space forces **index-as-hash** (place value `v` at index `v-1`). Recognise "O(1) space + values bounded by n" as the cue for cyclic sort / index marking.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 1 | Two Sum | Easy | The base hashmap-lookup reflex |
| ☐ | 49 | Group Anagrams | Medium | Designing a canonical hash key |
| ☐ | 238 | Product of Array Except Self | Medium | Prefix/suffix passes, no division |
| ☐ | 128 | Longest Consecutive Sequence | Medium | O(n) with a set; only start counting at sequence heads |
| ☐ | 560 | Subarray Sum Equals K | Medium | **Prefix sum + hashmap** — the highest-yield pattern here |
| ☐ | 41 | First Missing Positive | Hard | Index-as-hash / cyclic sort under O(1) space |

---

## 2. Two Pointers (4)

### When you see this
The array is **sorted** (or you may sort it), and you're looking for a pair/triple with a target property. Or: you're shrinking a window from both ends, or partitioning in place.

**The core insight:** from `(lo, hi)`, if `sum < target` you can *prove* no valid pair uses `lo`, so `lo++` is safe. That proof is what makes it O(n) instead of O(n²) — and it's what the interviewer wants you to articulate.

### Template

```java
// Opposite-direction pointers on a sorted array.
Arrays.sort(nums);
int lo = 0, hi = nums.length - 1;
while (lo < hi) {
    int sum = nums[lo] + nums[hi];
    if (sum == target) { /* record */ lo++; hi--; }
    else if (sum < target) lo++;          // need bigger → only lo can grow
    else hi--;                            // need smaller → only hi can shrink
}
```

```java
// De-duplication in 3Sum — the part everyone forgets.
for (int i = 0; i < n - 2; i++) {
    if (i > 0 && nums[i] == nums[i - 1]) continue;   // skip duplicate anchors
    // ... two-pointer inner loop, and skip duplicates on BOTH pointers after a hit
}
```

### Traps
- **Duplicate handling in 3Sum.** Skip duplicates for the anchor *and* for both pointers after recording a triple. This is the single most common bug in the entire topic.
- `Container With Most Water`: always move the **shorter** line. Moving the taller one can never improve the answer (width shrinks, height is still capped by the shorter side).
- `Trapping Rain Water`: two-pointer version needs `leftMax`/`rightMax`; the rule is to advance the side with the **smaller** max, because that side's answer is then fully determined.
- Sorting destroys original indices — if the problem wants indices, this pattern is probably wrong.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 15 | 3Sum | Medium | Sort + two pointers + **duplicate skipping** |
| ☐ | 11 | Container With Most Water | Medium | The "move the shorter side" proof |
| ☐ | 75 | Sort Colors | Medium | Dutch national flag, 3-way partition, one pass |
| ☐ | 42 | Trapping Rain Water | Hard | Two pointers *and* monotonic stack — solve it both ways |

---

## 3. Sliding Window (4)

### When you see this
**"Longest / shortest / count of *contiguous* subarray or substring satisfying P."** Contiguity is the tell. If the problem says "subsequence", it is *not* a sliding window — that's usually DP.

Two flavours, and you should know which one you're in:
- **Variable window** — grow right, shrink left while the window is invalid. Use for "longest".
- **Fixed window** — right moves, left follows at a fixed offset. Use for "size k".

### Template

```java
// Variable-size window. The shape is identical for almost every problem in this topic.
Map<Character, Integer> window = new HashMap<>();
int left = 0, best = 0;
for (int right = 0; right < s.length(); right++) {
    window.merge(s.charAt(right), 1, Integer::sum);          // 1. expand

    while (/* window is INVALID */) {                        // 2. contract
        char c = s.charAt(left++);
        if (window.merge(c, -1, Integer::sum) == 0) window.remove(c);
    }

    best = Math.max(best, right - left + 1);                 // 3. record (window is valid here)
}
```

```java
// Monotonic deque for sliding-window maximum. Deque holds INDICES, values decreasing.
Deque<Integer> dq = new ArrayDeque<>();
for (int i = 0; i < n; i++) {
    if (!dq.isEmpty() && dq.peekFirst() <= i - k) dq.pollFirst();   // drop out-of-window
    while (!dq.isEmpty() && nums[dq.peekLast()] <= nums[i]) dq.pollLast();
    dq.offerLast(i);
    if (i >= k - 1) result[i - k + 1] = nums[dq.peekFirst()];
}
```

### Traps
- **Record the answer in the right place.** For "longest", record *after* contracting (window is valid). For "shortest", record *inside* the contract loop. Getting this backwards is the classic off-by-everything bug.
- Removing a key when its count hits 0 — otherwise `map.size()` lies to you about distinct characters.
- `424. Longest Repeating Character Replacement`: the window never actually shrinks, and you can get away with a stale `maxCount`. Understand *why* that's still correct — it's a favourite follow-up question.
- Sliding-window maximum needs a **deque of indices**, not values, so you can evict entries that fell out of the window.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 3 | Longest Substring Without Repeating Characters | Medium | The base variable-window template |
| ☐ | 424 | Longest Repeating Character Replacement | Medium | Window validity defined by a *derived* quantity |
| ☐ | 76 | Minimum Window Substring | Hard | "Shortest" variant + a `have`/`need` counter |
| ☐ | 239 | Sliding Window Maximum | Hard | **Monotonic deque** — a genuinely distinct tool |

---

## 4. Binary Search (5)

### When you see this
Two very different triggers — keep them separate in your head:

1. **Search a sorted (or rotated-sorted) array** — the textbook case.
2. **Binary search on the answer.** The cue: *"minimise the maximum…"*, *"maximum minimum…"*, *"smallest k such that it's possible"*, plus a **monotone feasibility predicate** — if `k` works, every larger `k` works. You're searching the answer space, not the input. `875. Koko` is the canonical drill.

### Template

```java
// LOWER BOUND — first index where predicate is true. Learn ONE template, use it always.
int lo = 0, hi = n;                       // hi is EXCLUSIVE
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;         // avoids int overflow; (lo+hi)/2 does not
    if (predicate(mid)) hi = mid;         // mid might be the answer → keep it
    else                lo = mid + 1;     // mid is definitely not → discard it
}
return lo;                                // == n means "no such index"
```

```java
// BINARY SEARCH ON THE ANSWER — same skeleton, different search space.
long lo = 1, hi = maxPossibleAnswer;
while (lo < hi) {
    long mid = lo + (hi - lo) / 2;
    if (isFeasible(mid)) hi = mid;        // feasible → try to do better
    else                 lo = mid + 1;
}
return lo;
```

### Traps
- `(lo + hi) / 2` **overflows** for large ints. Always `lo + (hi - lo) / 2`.
- Mixing inclusive and exclusive `hi` between problems → infinite loops. Pick the half-open `[lo, hi)` form above and never deviate.
- Rotated array: decide which half is sorted *first* (`nums[lo] <= nums[mid]`), then check whether the target lies inside that sorted half. Handle duplicates separately if asked.
- `153. Find Minimum in Rotated Sorted Array`: compare `nums[mid]` to **`nums[hi]`**, not `nums[lo]`. Comparing to `lo` breaks on non-rotated input.
- `4. Median of Two Sorted Arrays`: binary search the **partition point** of the smaller array, not the values. Always search the shorter array to keep the bounds valid.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 34 | Find First and Last Position in Sorted Array | Medium | Lower bound / upper bound — the template itself |
| ☐ | 33 | Search in Rotated Sorted Array | Medium | "Which half is sorted?" reasoning |
| ☐ | 153 | Find Minimum in Rotated Sorted Array | Medium | Compare against `hi`, not `lo` |
| ☐ | 875 | Koko Eating Bananas | Medium | **Binary search on the answer** — highest transfer value |
| ☐ | 4 | Median of Two Sorted Arrays | Hard | Partition-based search; O(log min(m,n)) |

---

## 5. Stack & Monotonic Stack (4)

### When you see this
- **Plain stack:** matching/nesting (parentheses, expression parsing), or "undo the most recent thing".
- **Monotonic stack:** the phrase **"next greater / previous smaller"** appears, explicitly or in disguise. `Daily Temperatures` says "how many days until warmer" — that's next-greater. `Largest Rectangle` says "how far can this bar extend" — that's previous-smaller *and* next-smaller.

**The invariant:** the stack holds indices whose answers are still unknown. When the current element resolves them, you pop and record. Each index is pushed and popped once → **O(n)**.

### Template

```java
// NEXT GREATER ELEMENT. Stack holds indices; values are decreasing bottom→top.
int[] answer = new int[n];
Deque<Integer> stack = new ArrayDeque<>();
for (int i = 0; i < n; i++) {
    while (!stack.isEmpty() && nums[i] > nums[stack.peek()]) {
        int idx = stack.pop();
        answer[idx] = i - idx;            // ← current element resolves that index
    }
    stack.push(i);
}
// Anything left on the stack has no next-greater element.
```

```java
// LARGEST RECTANGLE IN HISTOGRAM — increasing stack + a sentinel to flush the tail.
Deque<Integer> stack = new ArrayDeque<>();
int best = 0;
for (int i = 0; i <= n; i++) {
    int h = (i == n) ? 0 : heights[i];                 // sentinel: forces a full flush
    while (!stack.isEmpty() && heights[stack.peek()] >= h) {
        int height = heights[stack.pop()];
        int left = stack.isEmpty() ? -1 : stack.peek();
        best = Math.max(best, height * (i - left - 1));  // width spans (left, i)
    }
    stack.push(i);
}
```

### Traps
- **Push indices, not values** — you almost always need the distance.
- Forgetting the **sentinel** (a trailing `0` or `i == n` case) leaves bars unresolved on the stack.
- Width in `Largest Rectangle` is `i - stack.peek() - 1` *after* popping, not `i - poppedIndex`. Draw it once and it sticks.
- Use `ArrayDeque`, not `java.util.Stack` — `Stack` extends `Vector` and is synchronised and slow. Mentioning this in an interview is a cheap point.
- Expression parsing: handle multi-digit numbers, unary minus, and **apply `*` / `/` immediately** while deferring `+` / `-` to the stack.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 155 | Min Stack | Medium | Auxiliary state alongside the stack, O(1) min |
| ☐ | 739 | Daily Temperatures | Medium | The base monotonic-stack template |
| ☐ | 227 | Basic Calculator II | Medium | Precedence via a deferred-operand stack |
| ☐ | 84 | Largest Rectangle in Histogram | Hard | Monotonic stack + width arithmetic (unlocks LC 85) |

---

## 6. Linked List (5)

### When you see this
It's a linked list — the pattern is decided by what's asked:

| Ask | Tool |
|---|---|
| Reverse / reorder | Three-pointer iterative reversal |
| Find middle, detect cycle, find k-th from end | **Fast & slow pointers** |
| Merge / sort k lists | Heap, or divide and conquer |
| O(1) get *and* put | Doubly linked list + hashmap (LRU) |

### Template

```java
// ITERATIVE REVERSAL — write this until it's muscle memory. O(n) time, O(1) space.
ListNode prev = null, curr = head;
while (curr != null) {
    ListNode next = curr.next;   // 1. save
    curr.next = prev;            // 2. flip
    prev = curr;                 // 3. advance prev
    curr = next;                 // 4. advance curr
}
return prev;                     // new head
```

```java
// FLOYD'S CYCLE DETECTION + finding the entry point.
ListNode slow = head, fast = head;
while (fast != null && fast.next != null) {
    slow = slow.next;
    fast = fast.next.next;
    if (slow == fast) {                       // cycle exists
        ListNode p = head;                    // ⭐ reset one pointer to head
        while (p != slow) { p = p.next; slow = slow.next; }
        return p;                             // == cycle entry
    }
}
return null;
```

```java
// The DUMMY NODE idiom. Removes every "what if we delete the head" special case.
ListNode dummy = new ListNode(0, head);
// ... operate on dummy.next ...
return dummy.next;
```

### Traps
- **Use a dummy head** whenever the head might change. It deletes half your edge cases for one line.
- Losing the `next` pointer before reassigning — save it first, always.
- Floyd's entry-point step (reset one pointer to `head`, advance both by 1) is easy to half-remember. Be able to justify it: the meeting point is `k` steps from the cycle entry, where `k` is the distance from head to entry, mod cycle length.
- `25. Reverse Nodes in k-Group`: you must **check that k nodes exist before reversing**, and reconnect the tail of the previous group to the new head of this one.
- `146. LRU Cache`: use a **doubly** linked list — singly gives O(n) removal. `Node` needs both `prev` and `next`, and the map stores nodes, not values.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 206 | Reverse Linked List | Easy | The reversal template — do it iteratively *and* recursively |
| ☐ | 142 | Linked List Cycle II | Medium | Floyd's algorithm and the entry-point proof |
| ☐ | 23 | Merge k Sorted Lists | Hard | Min-heap (O(n log k)) vs. divide and conquer |
| ☐ | 25 | Reverse Nodes in k-Group | Hard | Reversal + group boundary bookkeeping |
| ☐ | 146 | LRU Cache | Medium | **Design** — hashmap + doubly linked list, O(1) both ops |

---

## 7. Trees & BST (8)

### When you see this
Almost every tree problem is one of four shapes. Identify the shape and the code writes itself:

| Shape | Use when | Signature |
|---|---|---|
| **DFS, return a value up** | Answer depends on subtree results | `int dfs(node)` returning info to the parent |
| **DFS with a global** | The answer can be "centred" at any node | `dfs` returns one thing, updates a field with another — **the key trick** |
| **BFS by level** | "level", "row", "closest", "right side view" | Queue + `int size = queue.size()` per level |
| **In-order on a BST** | "k-th smallest", "validate", "sorted" | In-order traversal of a BST is sorted — that *is* the property |

### Template

```java
// BFS BY LEVEL. The `size` snapshot is what separates the levels.
Queue<TreeNode> q = new LinkedList<>();
if (root != null) q.offer(root);
while (!q.isEmpty()) {
    int size = q.size();                       // ⭐ snapshot BEFORE the inner loop
    for (int i = 0; i < size; i++) {
        TreeNode node = q.poll();
        // i == size - 1  → this is the rightmost node of the level (LC 199)
        if (node.left  != null) q.offer(node.left);
        if (node.right != null) q.offer(node.right);
    }
}
```

```java
// "RETURN ONE THING, RECORD ANOTHER" — the single most reusable tree trick.
// Used by Max Path Sum, Diameter, Longest Univalue Path, Balanced Binary Tree...
private int best = Integer.MIN_VALUE;

private int gain(TreeNode node) {          // returns: best DOWNWARD path from node
    if (node == null) return 0;
    int left  = Math.max(gain(node.left),  0);   // negative contributions → drop them
    int right = Math.max(gain(node.right), 0);
    best = Math.max(best, node.val + left + right);   // record: path THROUGH node
    return node.val + Math.max(left, right);          // return: path DOWN one side
}
```

```java
// VALIDATE BST — bounds, not just parent comparison.
private boolean valid(TreeNode n, long min, long max) {
    if (n == null) return true;
    if (n.val <= min || n.val >= max) return false;
    return valid(n.left, min, n.val) && valid(n.right, n.val, max);
}
```

### Traps
- **`98. Validate BST`:** comparing each node only to its parent is wrong. You need an inherited `(min, max)` range. Use `long` bounds (or `Integer` objects) so `Integer.MIN_VALUE` as a node value doesn't break you.
- **`124. Max Path Sum`:** the returned value and the recorded value are *different*. A path through a node can use both children; a path returned to the parent can use only one. Clamp negative child gains to 0.
- `236. LCA`: the elegant version returns the node itself when found, and a node with two non-null child results is the LCA. Note the problem guarantees both nodes exist — ask if they might not.
- `105. Construct from Preorder + Inorder`: use a hashmap of value→inorder-index, and pass ranges. Rebuilding by `Arrays.copyOfRange` each call is O(n²).
- `297. Serialize/Deserialize`: preorder with explicit `null` markers is simplest. Level-order works too. Be able to say why in-order alone is **not** sufficient to reconstruct a tree.
- Recursion depth: a skewed tree of 10⁵ nodes will stack-overflow. Mention the iterative alternative.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 102 | Binary Tree Level Order Traversal | Medium | The BFS-by-level template |
| ☐ | 199 | Binary Tree Right Side View | Medium | Level BFS with a per-level selection |
| ☐ | 98 | Validate Binary Search Tree | Medium | Inherited bounds, not parent comparison |
| ☐ | 230 | Kth Smallest Element in a BST | Medium | In-order = sorted; iterative version with a stack |
| ☐ | 236 | Lowest Common Ancestor of a Binary Tree | Medium | Post-order "found in both subtrees" logic |
| ☐ | 105 | Construct Binary Tree from Preorder and Inorder | Medium | Index map + range recursion |
| ☐ | 124 | Binary Tree Maximum Path Sum | Hard | **Return one thing, record another** |
| ☐ | 297 | Serialize and Deserialize Binary Tree | Hard | Design + traversal choice justification |

---

## 8. Heap / Priority Queue (3)

### When you see this
**"Top k", "k-th largest/smallest", "median of a stream", "merge k sorted".** The word *k* next to *largest/smallest* is the tell.

The counter-intuitive bit worth internalising: **for the k largest, use a MIN-heap of size k.** You keep the k best seen so far, and the smallest of them sits at the top ready to be evicted. O(n log k), and O(k) space instead of O(n).

### Template

```java
// TOP K LARGEST — min-heap of size k. O(n log k).
PriorityQueue<Integer> minHeap = new PriorityQueue<>();     // natural order = min-heap
for (int x : nums) {
    minHeap.offer(x);
    if (minHeap.size() > k) minHeap.poll();                 // evict the smallest
}
// minHeap now holds the k largest; minHeap.peek() is the k-th largest.
```

```java
// TWO HEAPS for a streaming median. Invariant: lo.size() == hi.size() or hi.size() + 1.
PriorityQueue<Integer> lo = new PriorityQueue<>(Comparator.reverseOrder()); // max-heap
PriorityQueue<Integer> hi = new PriorityQueue<>();                          // min-heap

void add(int num) {
    lo.offer(num);
    hi.offer(lo.poll());                       // funnel through to keep them ordered
    if (hi.size() > lo.size()) lo.offer(hi.poll());
}
double median() {
    return lo.size() > hi.size() ? lo.peek() : (lo.peek() + hi.peek()) / 2.0;
}
```

### Traps
- Java's `PriorityQueue` is a **min-heap by default**. For a max-heap: `new PriorityQueue<>(Comparator.reverseOrder())`.
- `PriorityQueue.remove(Object)` is **O(n)**, not O(log n) — it scans. If you need arbitrary removal, you need an indexed heap or lazy deletion.
- **Don't reach for a heap when counting sort works.** `347. Top K Frequent` has an O(n) bucket-sort solution (bucket by frequency, 0..n). Mentioning it unprompted is a strong signal.
- `215. Kth Largest`: know **Quickselect** — O(n) average, O(1) extra space. Randomise the pivot or an adversarial (sorted) input makes it O(n²).
- The two-heap median: rebalance by always pushing into one heap and funnelling the top into the other. Trying to branch on comparisons directly is where bugs live.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 347 | Top K Frequent Elements | Medium | Heap vs. bucket sort — know both |
| ☐ | 215 | Kth Largest Element in an Array | Medium | Quickselect + the heap alternative |
| ☐ | 295 | Find Median from Data Stream | Hard | **Two heaps** with a balance invariant |

---

## 9. Backtracking (4)

### When you see this
**"All possible…", "generate every…", "count the ways" where you must *enumerate* rather than count.** The search space is exponential and you need to prune it.

Every backtracking problem is the same three lines: **choose → explore → un-choose.** What changes is only the loop bounds and the pruning condition.

### Template

```java
// UNIVERSAL BACKTRACKING SKELETON.
void backtrack(List<Integer> path, int start) {
    if (isComplete(path)) { result.add(new ArrayList<>(path)); return; }  // ⭐ COPY

    for (int i = start; i < candidates.length; i++) {
        if (!isValid(candidates[i])) continue;             // prune early — this is the win

        path.add(candidates[i]);                           // choose
        backtrack(path, i + 1);                            // explore  (i → reuse allowed)
        path.remove(path.size() - 1);                      // un-choose
    }
}
```

**The three loop shapes — know which is which:**

| Shape | Recursive call | Produces |
|---|---|---|
| Subsets / combinations | `backtrack(path, i + 1)` | Each element used at most once, order irrelevant |
| Combination sum (reuse allowed) | `backtrack(path, i)` | Same element repeatable |
| Permutations | loop from `0`, with a `used[]` array | Order matters |

### Traps
- **`result.add(new ArrayList<>(path))`** — adding `path` directly stores a reference that you then mutate. Every entry ends up empty. This is *the* backtracking bug.
- **De-duplication with duplicate inputs:** sort first, then `if (i > start && nums[i] == nums[i-1]) continue;`. Note it's `i > start`, not `i > 0`.
- Forgetting to un-choose (the `remove` line) silently corrupts every later branch.
- `51. N-Queens`: track columns and both diagonals as sets — `row + col` for one diagonal, `row - col` for the other. Scanning the board each time is O(n) per check for no reason.
- Prune **before** recursing, not after. A `sum > target` check at the top of the loop body is worth orders of magnitude.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 78 | Subsets | Medium | The base skeleton (also: the bitmask solution) |
| ☐ | 46 | Permutations | Medium | The `used[]` variant |
| ☐ | 39 | Combination Sum | Medium | Reuse-allowed loop (`i`, not `i + 1`) + pruning |
| ☐ | 51 | N-Queens | Hard | Constraint tracking with sets; the full pattern |

---

## 10. Graphs — BFS / DFS / Dijkstra (5)

### When you see this
Grids, networks, dependencies, "connected", "reachable", "shortest path". **A 2-D grid is a graph** — each cell is a node with up to 4 neighbours. Recognising that is half the topic.

**Choosing the traversal — this decision table is the whole game:**

| Need | Use |
|---|---|
| Shortest path, **unweighted** | **BFS** (first time you reach a node is optimal) |
| Just reachability / connectivity | DFS (less code, less memory) |
| Shortest path, **weighted, non-negative** | **Dijkstra** (BFS + priority queue) |
| Shortest path with **negative** weights | Bellman-Ford |
| Spread from many sources at once | **Multi-source BFS** — seed the queue with all sources |

### Template

```java
// GRID BFS. The directions array keeps the neighbour loop clean.
int[][] DIRS = {{1,0},{-1,0},{0,1},{0,-1}};
Queue<int[]> q = new ArrayDeque<>();
boolean[][] seen = new boolean[m][n];

q.offer(new int[]{sr, sc});
seen[sr][sc] = true;                       // ⭐ mark on ENQUEUE, not on dequeue

while (!q.isEmpty()) {
    int[] cell = q.poll();
    for (int[] d : DIRS) {
        int r = cell[0] + d[0], c = cell[1] + d[1];
        if (r < 0 || r >= m || c < 0 || c >= n) continue;
        if (seen[r][c] || grid[r][c] == '0') continue;
        seen[r][c] = true;
        q.offer(new int[]{r, c});
    }
}
```

```java
// DIJKSTRA — BFS with a priority queue. O(E log V).
PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[1]));
int[] dist = new int[n];
Arrays.fill(dist, Integer.MAX_VALUE);
dist[src] = 0;
pq.offer(new int[]{src, 0});

while (!pq.isEmpty()) {
    int[] top = pq.poll();
    int u = top[0], d = top[1];
    if (d > dist[u]) continue;                       // ⭐ stale entry — skip it
    for (int[] edge : adj.get(u)) {                  // edge = {v, weight}
        int nd = d + edge[1];
        if (nd < dist[edge[0]]) {
            dist[edge[0]] = nd;
            pq.offer(new int[]{edge[0], nd});
        }
    }
}
```

### Traps
- **Mark visited when you *enqueue*, not when you dequeue.** Otherwise a node gets queued many times and BFS degrades badly (and can blow memory).
- **Multi-source BFS:** push *all* sources before the loop starts. `994. Rotting Oranges` is exactly this — the natural instinct to loop one source at a time is O(n) times slower.
- Dijkstra with a stale-entry check (`if (d > dist[u]) continue;`) is the lazy-deletion idiom, since `PriorityQueue` has no decrease-key.
- **Dijkstra is wrong with negative edges.** Say so if the constraints allow them.
- `133. Clone Graph`: keep a `Map<Node, Node>` from original to clone, and check it **before** recursing — that's what handles cycles.
- Modifying the grid in place (`grid[r][c] = '0'`) instead of a `seen` array is fine and saves memory — but say out loud that you're mutating the input, and ask if that's acceptable.
- `127. Word Ladder`: build the adjacency implicitly via wildcard patterns (`h*t`), or you'll do O(n²) word comparisons. Bidirectional BFS is the strong follow-up.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 200 | Number of Islands | Medium | Grid DFS/BFS — the base case for everything here |
| ☐ | 133 | Clone Graph | Medium | Traversal with a visited **map**, cycle-safe |
| ☐ | 994 | Rotting Oranges | Medium | **Multi-source BFS** + level counting |
| ☐ | 127 | Word Ladder | Hard | Implicit graph construction; BFS for shortest path |
| ☐ | 743 | Network Delay Time | Medium | **Dijkstra** with a priority queue |

---

## 11. Topological Sort (2)

### When you see this
**"Prerequisites", "dependencies", "build order", "is there a cycle in a directed graph", "valid ordering".** Any time A must come before B.

Two implementations — Kahn's is easier to get right and gives cycle detection for free.

### Template

```java
// KAHN'S ALGORITHM (BFS). Cycle detection falls out of the final count.
int[] indegree = new int[n];
List<List<Integer>> adj = /* build adjacency: prerequisite -> dependent */;
for (List<Integer> nbrs : adj) for (int v : nbrs) indegree[v]++;

Queue<Integer> q = new ArrayDeque<>();
for (int i = 0; i < n; i++) if (indegree[i] == 0) q.offer(i);

List<Integer> order = new ArrayList<>();
while (!q.isEmpty()) {
    int u = q.poll();
    order.add(u);
    for (int v : adj.get(u)) {
        if (--indegree[v] == 0) q.offer(v);        // last dependency resolved
    }
}
// ⭐ order.size() < n  ⇒  a cycle exists  ⇒  no valid ordering
return order.size() == n ? order : new int[0];
```

### Traps
- **Edge direction.** "To take course `a` you need `b`" means the edge is `b → a`. Reversing it silently produces a plausible-looking wrong answer. Write the direction down before you code.
- The cycle check is `order.size() == n`, not a separate DFS. Forgetting it is the most common miss.
- DFS-based topo sort needs **three** colours (white/grey/black). A grey node encountered again is a back edge = cycle. A simple boolean `visited` cannot detect cycles.
- `269. Alien Dictionary` edge cases: compare **adjacent** words only, take the first differing character, and if `word1` is longer than `word2` and `word2` is a prefix of it → invalid input, return `""`. That prefix case is the one interviewers check for.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 210 | Course Schedule II | Medium | Kahn's algorithm + cycle detection |
| ☐ | 269 | Alien Dictionary | Hard | Building the graph from constraints — the hard half |

---

## 12. Union-Find (2)

### When you see this
**Dynamic connectivity:** edges arrive over time and you're repeatedly asked "are these two connected?". Also: counting connected components, detecting the edge that creates a cycle, and Kruskal's MST.

**The rule of thumb:** if edges are all known up front and you just traverse once, DFS/BFS is simpler. If connectivity queries are *interleaved* with edge additions, use Union-Find.

### Template

```java
// UNION-FIND with path compression + union by rank. ~O(α(n)) ≈ O(1) amortised.
class DSU {
    private final int[] parent, rank;
    private int components;

    DSU(int n) {
        parent = new int[n];
        rank   = new int[n];
        components = n;
        for (int i = 0; i < n; i++) parent[i] = i;
    }

    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);   // ⭐ path compression
        return parent[x];
    }

    boolean union(int a, int b) {
        int ra = find(a), rb = find(b);
        if (ra == rb) return false;                        // already connected → a cycle
        if (rank[ra] < rank[rb]) { int t = ra; ra = rb; rb = t; }
        parent[rb] = ra;
        if (rank[ra] == rank[rb]) rank[ra]++;
        components--;
        return true;
    }

    int components() { return components; }
}
```

### Traps
- **Both optimisations matter.** Path compression alone or union-by-rank alone gives O(log n); together they give the inverse-Ackermann bound. Interviewers ask which you used.
- `union` returning `false` means "already in the same set" — that's your cycle detector (`684. Redundant Connection`) and your Kruskal skip condition.
- Union the **roots**, not the nodes: `parent[find(b)] = find(a)`, never `parent[b] = a`.
- 2-D grids need index flattening: `id = row * numCols + col`.
- **Kruskal's MST** = sort all edges by weight, then union greedily, skipping edges whose union returns `false`. `1584` is exactly this on a complete graph of Manhattan distances.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 684 | Redundant Connection | Medium | Cycle detection via a failed union |
| ☐ | 1584 | Min Cost to Connect All Points | Medium | **Kruskal's MST** on top of DSU |

---

## 13. 1-D Dynamic Programming (6)

### When you see this
**"Maximum/minimum/count of ways"** + **overlapping subproblems** + **optimal substructure**. If greedy feels almost right but you can construct a counterexample, it's DP.

**The four questions — answer them out loud, in order, before writing any code:**

1. **What is the state?** What does `dp[i]` *mean*, in one English sentence?
2. **What is the recurrence?** How does `dp[i]` follow from earlier states?
3. **What are the base cases?**
4. **What is the iteration order?** (Every state a recurrence depends on must already be computed.)

If you can't answer #1 in one sentence, your state is wrong. That's the most common failure and it's fixable by slowing down.

### Template

```java
// The four canonical 1-D recurrences. Recognise the shape, recall the recurrence.

// 1. HOUSE ROBBER — take-or-skip with a gap
dp[i] = Math.max(dp[i-1],  dp[i-2] + nums[i]);

// 2. COIN CHANGE — unbounded knapsack (min coins).  Coin loop OUTER or INNER both work
//    for the min-coins variant; for COUNTING combinations, coin loop MUST be outer.
for (int coin : coins)
    for (int amt = coin; amt <= target; amt++)
        dp[amt] = Math.min(dp[amt], dp[amt - coin] + 1);

// 3. 0/1 KNAPSACK (e.g. Partition Equal Subset Sum) — iterate capacity DOWNWARD
for (int num : nums)
    for (int s = target; s >= num; s--)          // ⭐ reverse, so each item is used once
        dp[s] |= dp[s - num];

// 4. KADANE — max subarray
running = Math.max(nums[i], running + nums[i]);
best    = Math.max(best, running);
```

```java
// LIS in O(n log n) — patience sorting. `tails[i]` = smallest tail of an
// increasing subsequence of length i+1. NOT itself a valid subsequence.
List<Integer> tails = new ArrayList<>();
for (int x : nums) {
    int pos = Collections.binarySearch(tails, x);
    if (pos < 0) pos = -(pos + 1);                    // insertion point
    if (pos == tails.size()) tails.add(x);
    else tails.set(pos, x);
}
return tails.size();
```

### Traps
- **0/1 knapsack: iterate capacity DOWNWARD.** Forward iteration lets you reuse an item, which silently turns it into unbounded knapsack. This is the single highest-frequency DP bug.
- **Counting combinations vs. permutations** in coin change: coin loop outer → combinations; amount loop outer → permutations. Know which the problem wants.
- `152. Maximum Product Subarray` (a great extra rep): track **both** max and min running products, because a negative number swaps them.
- `139. Word Break`: `dp[i]` = "the prefix of length `i` is breakable". Off-by-one on the prefix length is the usual bug — be deliberate about whether `dp[i]` covers `i` characters or index `i`.
- Always state the space optimisation even if you don't implement it: House Robber needs two variables, not an array.
- Recursion + memo is fine and often clearer. But know the bottom-up form too — interviewers ask for it.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 53 | Maximum Subarray | Medium | Kadane (also: the divide-and-conquer version) |
| ☐ | 198 | House Robber | Medium | The base take-or-skip recurrence |
| ☐ | 322 | Coin Change | Medium | Unbounded knapsack, min variant |
| ☐ | 416 | Partition Equal Subset Sum | Medium | **0/1 knapsack + the reverse-iteration trap** |
| ☐ | 300 | Longest Increasing Subsequence | Medium | O(n²) DP **and** O(n log n) patience sorting |
| ☐ | 139 | Word Break | Medium | Prefix-state DP with a dictionary set |

---

## 14. 2-D / String DP (4)

### When you see this
**Two sequences** being compared or aligned (→ `dp[i][j]` = answer for the first `i` of A and first `j` of B), or **one sequence with a range** (→ `dp[i][j]` = answer for the subarray `i..j`).

The two sub-shapes:

| Shape | State | Iteration order | Examples |
|---|---|---|---|
| **Two sequences** | `dp[i][j]` over prefixes | Row by row, forward | LCS, Edit Distance |
| **Interval DP** | `dp[i][j]` over the range `i..j` | **By increasing length** | Burst Balloons, Matrix Chain |

### Template

```java
// TWO-SEQUENCE DP — LCS. Every alignment problem is a variation of this.
for (int i = 1; i <= m; i++) {
    for (int j = 1; j <= n; j++) {
        if (a.charAt(i-1) == b.charAt(j-1)) dp[i][j] = dp[i-1][j-1] + 1;
        else dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);
    }
}
```

```java
// EDIT DISTANCE — the three operations, made explicit.
if (a.charAt(i-1) == b.charAt(j-1)) dp[i][j] = dp[i-1][j-1];
else dp[i][j] = 1 + Math.min(dp[i-1][j-1],            // replace
                    Math.min(dp[i-1][j],              // delete from a
                             dp[i][j-1]));            // insert into a
// Base cases: dp[i][0] = i, dp[0][j] = j  ← converting to/from the empty string.
```

```java
// INTERVAL DP — iterate by LENGTH, not by index. This ordering IS the pattern.
for (int len = 2; len <= n; len++) {
    for (int i = 0; i + len < n + 1; i++) {
        int j = i + len - 1;
        for (int k = i; k <= j; k++) {              // last thing resolved in i..j
            dp[i][j] = Math.max(dp[i][j], dp[i][k-1] + value(k) + dp[k+1][j]);
        }
    }
}
```

### Traps
- **Off-by-one from 1-indexed DP over 0-indexed strings.** `dp[i][j]` uses `a.charAt(i-1)`. Pick the convention, write it in a comment, and stick to it.
- **Interval DP must iterate by increasing length.** Iterating `i` then `j` computes states before their sub-ranges exist.
- `312. Burst Balloons`: think of `k` as the **last** balloon burst in the range, not the first. That reframing is the entire trick — with "first" the subproblems aren't independent.
- `5. Longest Palindromic Substring`: expand-around-centre is O(n²) time / **O(1) space** and easier to write than the DP. Know Manacher's exists (O(n)); you won't be asked to write it.
- Space optimisation: two-sequence DP only ever needs the previous row → O(min(m, n)). Mention it.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 1143 | Longest Common Subsequence | Medium | The base two-sequence grid |
| ☐ | 72 | Edit Distance | Medium | Three transitions + base cases |
| ☐ | 5 | Longest Palindromic Substring | Medium | Expand-around-centre vs. DP |
| ☐ | 312 | Burst Balloons | Hard | **Interval DP** and the "last one" reframing |

---

## 15. Greedy (3)

### When you see this
A locally optimal choice looks like it leads to a global optimum. **The interviewer's real question is: can you justify it?** A greedy answer without an exchange argument is a guess.

**The proof pattern (exchange argument):** *"Take any optimal solution. If it differs from my greedy choice at the first decision, I can swap in the greedy choice without making the solution worse. Therefore a greedy-containing optimum exists."* Practise saying it — it's what separates greedy from "I guessed and the tests passed".

### Template

```java
// REACHABILITY GREEDY — Jump Game. Track the furthest index reachable so far.
int furthest = 0;
for (int i = 0; i < n; i++) {
    if (i > furthest) return false;              // there's a gap we can't cross
    furthest = Math.max(furthest, i + nums[i]);
}
return true;
```

```java
// "LAST OCCURRENCE" GREEDY — Partition Labels.
int[] last = new int[26];
for (int i = 0; i < s.length(); i++) last[s.charAt(i) - 'a'] = i;

int start = 0, end = 0;
for (int i = 0; i < s.length(); i++) {
    end = Math.max(end, last[s.charAt(i) - 'a']);
    if (i == end) { result.add(end - start + 1); start = i + 1; }   // partition closes
}
```

### Traps
- **Greedy is often just wrong.** Coin Change with arbitrary denominations is the standard counterexample: coins `{1, 3, 4}`, target `6` → greedy gives `4+1+1 = 3` coins, optimal is `3+3 = 2`. Have this example ready; it proves you know the boundary.
- `134. Gas Station`: if total gas ≥ total cost a solution exists, and the answer is the index right after the last point where the running tank went negative. Be able to justify both halves.
- Don't confuse greedy with DP. If a choice constrains *future* choices in a way you can't bound locally, you need DP.
- Sorting is usually step one — by end time, by start time, by ratio. **Choosing the sort key is the whole design decision**, so say why you picked it.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 55 | Jump Game | Medium | Reachability greedy + the correctness argument |
| ☐ | 134 | Gas Station | Medium | A two-part proof (existence + location) |
| ☐ | 763 | Partition Labels | Medium | Last-occurrence precomputation |

---

## 16. Intervals (3)

### When you see this
The input is a list of `[start, end]` pairs. **Step one is always: sort.** The only real decision is *by which endpoint*, and that decision follows from the goal:

| Goal | Sort by | Why |
|---|---|---|
| Merge overlapping | **start** | Process left to right, extend the current merged block |
| Maximise non-overlapping count | **end** | Finishing earliest leaves the most room — the classic activity-selection result |
| Count concurrent (rooms, CPUs) | **start**, with a min-heap of end times | The heap size *is* the concurrency |

### Template

```java
// MERGE INTERVALS — sort by start.
Arrays.sort(intervals, Comparator.comparingInt(a -> a[0]));
List<int[]> merged = new ArrayList<>();
for (int[] cur : intervals) {
    int[] last = merged.isEmpty() ? null : merged.get(merged.size() - 1);
    if (last != null && cur[0] <= last[1]) last[1] = Math.max(last[1], cur[1]);  // extend
    else merged.add(cur);
}
```

```java
// MAX CONCURRENT INTERVALS (Meeting Rooms II) — sort by start, min-heap of end times.
Arrays.sort(intervals, Comparator.comparingInt(a -> a[0]));
PriorityQueue<Integer> endTimes = new PriorityQueue<>();
for (int[] iv : intervals) {
    if (!endTimes.isEmpty() && endTimes.peek() <= iv[0]) endTimes.poll();  // room freed
    endTimes.offer(iv[1]);
}
return endTimes.size();          // ⭐ peak concurrency == rooms needed
```

```java
// SWEEP LINE — the alternative for counting concurrency. Often cleaner.
// Build events: (start, +1), (end, -1). Sort by time, ties: -1 before +1.
// Running sum's maximum is the answer.
```

### Traps
- **Sorting by the wrong endpoint** is the entire failure mode of this topic. Non-overlapping/activity-selection → sort by **end**. Get this wrong and you'll produce a confident wrong answer.
- **Touching intervals:** is `[1,2]` and `[2,3]` an overlap? Depends on whether endpoints are inclusive. **Ask.** For meeting rooms they typically don't conflict.
- `435. Non-overlapping Intervals`: count how many you *keep* (greedy by end time), then answer is `n - kept`. Trying to count removals directly is fiddlier.
- `253. Meeting Rooms II` is LeetCode Premium — free equivalents are **1094. Car Pooling** and **My Calendar** problems. The pattern is identical.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 56 | Merge Intervals | Medium | Sort by start; the merge loop |
| ☐ | 435 | Non-overlapping Intervals | Medium | **Sort by end** — activity selection |
| ☐ | 253 | Meeting Rooms II | Medium | Min-heap of end times / sweep line (or LC 1094) |

---

## 17. Bit Manipulation (3)

### When you see this
"Without extra space", "find the one that appears once", "count bits", "subsets via bitmask", or constraints mentioning powers of two.

### Template

```java
// The operations worth memorising.
x & 1              // is odd
x >> 1             // divide by 2   (use >>> for unsigned / when sign matters)
x & (x - 1)        // clear the lowest set bit   ⭐ the most useful identity here
x & (-x)           // isolate the lowest set bit
x ^ x == 0         // XOR self-cancels  ⭐ the second most useful
(x & (x-1)) == 0   // is a power of two (for x > 0)
1 << i             // the i-th bit
x |= (1 << i)      // set bit i
x &= ~(1 << i)     // clear bit i
```

```java
// COUNTING BITS in O(n) — DP on bits, two equivalent recurrences.
dp[i] = dp[i >> 1] + (i & 1);      // "same as i/2, plus the last bit"
dp[i] = dp[i & (i - 1)] + 1;       // "one more than i with its lowest set bit cleared"
```

```java
// SINGLE NUMBER II — every element appears 3× except one. Bit-count mod 3.
int result = 0;
for (int bit = 0; bit < 32; bit++) {
    int count = 0;
    for (int num : nums) count += (num >> bit) & 1;
    if (count % 3 != 0) result |= (1 << bit);
}
```

### Traps
- **`>>` vs `>>>`.** `>>` preserves the sign bit; `>>>` fills with zeros. Iterating bits of a possibly-negative int with `>>` loops forever.
- Operator precedence in Java: `&`, `^`, `|` bind **looser than** `==`. `if (x & 1 == 0)` does not do what you think — parenthesise.
- XOR gives you "the one that appears once" only when everything else appears an **even** number of times. For 3× use the bit-count-mod-3 approach.
- Bitmask subsets: `for (int mask = 0; mask < (1 << n); mask++)` — remember it's `1 << n`, and that this is only viable for n ≲ 20.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 136 | Single Number | Easy | XOR self-cancellation |
| ☐ | 338 | Counting Bits | Easy | DP on bit structure |
| ☐ | 137 | Single Number II | Medium | Per-bit counting mod k — generalises |

---

## 18. Tries (2)

### When you see this
**Prefix** anything: autocomplete, "words starting with…", or **searching a dictionary against a grid/string** where you'd otherwise re-scan the word list repeatedly.

The killer combination is **Trie + backtracking** (`212. Word Search II`): instead of running a DFS per word, run **one** DFS over the grid and walk the trie alongside it. That's the whole reason the trie exists in that problem.

### Template

```java
class TrieNode {
    TrieNode[] children = new TrieNode[26];
    String word;                        // ⭐ store the WORD (not just a boolean) —
}                                       //    lets DFS collect matches with no backtrack

void insert(TrieNode root, String word) {
    TrieNode node = root;
    for (char c : word.toCharArray()) {
        int i = c - 'a';
        if (node.children[i] == null) node.children[i] = new TrieNode();
        node = node.children[i];
    }
    node.word = word;
}
```

```java
// TRIE + GRID BACKTRACKING (Word Search II) — one DFS for ALL words.
void dfs(char[][] board, int r, int c, TrieNode node, List<String> out) {
    char ch = board[r][c];
    if (ch == '#' || node.children[ch - 'a'] == null) return;

    node = node.children[ch - 'a'];
    if (node.word != null) { out.add(node.word); node.word = null; }  // ⭐ de-dupe

    board[r][c] = '#';                                   // mark visited
    for (int[] d : DIRS) { /* bounds check, then dfs */ }
    board[r][c] = ch;                                    // restore
}
```

### Traps
- **Store the word at the terminal node**, not a boolean. Then a grid DFS can emit the match immediately with no path reconstruction.
- **Null the word after collecting it** — that's how you avoid duplicates without a set.
- `211. Design Add and Search Words` (a good extra rep): the `.` wildcard means you must try **all 26 children** at that position — the search becomes a DFS, not a walk.
- Memory: `TrieNode[26]` per node is wasteful for sparse alphabets; a `HashMap<Character, TrieNode>` trades speed for space. Mention the trade-off.
- Prune dead branches after a match to keep later DFS calls fast — a nice optimisation to volunteer.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 208 | Implement Trie (Prefix Tree) | Medium | The structure itself |
| ☐ | 212 | Word Search II | Hard | **Trie + backtracking** — the payoff problem |

---

## 19. Matrix Manipulation (2)

### When you see this
In-place rotation, spiral traversal, layer-by-layer processing. Low conceptual difficulty, **high bug density** — these are pure index-discipline problems and they show up in interviews precisely because they're easy to get subtly wrong under pressure.

### Template

```java
// ROTATE 90° CLOCKWISE, IN PLACE = transpose, then reverse each row.
for (int i = 0; i < n; i++)
    for (int j = i + 1; j < n; j++) {           // ⭐ j starts at i+1, or you undo the swap
        int t = m[i][j]; m[i][j] = m[j][i]; m[j][i] = t;
    }
for (int[] row : m) reverse(row);
// Counter-clockwise: transpose, then reverse the COLUMN order instead.
```

```java
// SPIRAL TRAVERSAL — four moving boundaries.
int top = 0, bottom = m - 1, left = 0, right = n - 1;
while (top <= bottom && left <= right) {
    for (int j = left; j <= right; j++) out.add(matrix[top][j]);
    top++;
    for (int i = top; i <= bottom; i++) out.add(matrix[i][right]);
    right--;
    if (top <= bottom) {                                  // ⭐ guard: single row left
        for (int j = right; j >= left; j--) out.add(matrix[bottom][j]);
        bottom--;
    }
    if (left <= right) {                                  // ⭐ guard: single column left
        for (int i = bottom; i >= top; i--) out.add(matrix[i][left]);
        left++;
    }
}
```

### Traps
- Transpose loop must start at `j = i + 1`. Starting at `0` swaps every pair twice and leaves the matrix unchanged.
- Spiral: the **two guards** before the bottom row and left column are mandatory. Without them a single remaining row or column gets emitted twice. This is the bug in ~90% of first attempts.
- Non-square matrices: rotation in place only works for square. For rectangular you need a new matrix — say so.

### Problems

| ✓ | # | Problem | Level | What it drills |
|---|---|---|---|---|
| ☐ | 48 | Rotate Image | Medium | Transpose + reverse |
| ☐ | 54 | Spiral Matrix | Medium | Boundary discipline and the guard conditions |

---

## Complexity Quick Reference

| Structure | Access | Search | Insert | Delete | Note |
|---|---|---|---|---|---|
| Array | O(1) | O(n) | O(n) | O(n) | O(log n) search if sorted |
| Dynamic array (`ArrayList`) | O(1) | O(n) | O(1)* | O(n) | *amortised |
| Linked list | O(n) | O(n) | O(1) | O(1) | given the node reference |
| `HashMap` / `HashSet` | — | O(1)* | O(1)* | O(1)* | *average; O(n) worst with collisions |
| `TreeMap` / `TreeSet` | — | O(log n) | O(log n) | O(log n) | sorted; gives `floorKey`/`ceilingKey` |
| Heap (`PriorityQueue`) | O(1) peek | O(n) | O(log n) | O(log n) | `remove(Object)` is O(n)! |
| Trie | — | O(L) | O(L) | O(L) | L = key length |
| Union-Find | — | O(α(n)) | O(α(n)) | — | with both optimisations |

| Algorithm | Time | Space |
|---|---|---|
| Binary search | O(log n) | O(1) |
| Merge sort / heap sort | O(n log n) | O(n) / O(1) |
| Quicksort | O(n log n) avg, O(n²) worst | O(log n) |
| Quickselect | O(n) avg | O(1) |
| BFS / DFS | O(V + E) | O(V) |
| Dijkstra (binary heap) | O(E log V) | O(V) |
| Topological sort | O(V + E) | O(V) |
| Kruskal MST | O(E log E) | O(V) |

**Inferring the intended solution from the constraints** — a genuinely useful trick:

| n up to | Target complexity | Likely approach |
|---|---|---|
| 10⁹ or more | O(log n) / O(1) | Binary search, math, bit tricks |
| 10⁶–10⁸ | O(n) | Single pass, hashmap, two pointers, sliding window |
| 10⁵ | O(n log n) | Sorting, heap, binary search on the answer |
| ~5000 | O(n²) | 2-D DP, nested loops |
| ~500 | O(n³) | Interval DP, Floyd-Warshall |
| ~20 | O(2ⁿ) | Bitmask DP, subset enumeration |
| ~10 | O(n!) | Permutation backtracking |

---

## The 18 Patterns, Compressed

If you remember nothing else, remember these triggers:

1. Repeated lookup in an inner loop → **HashMap**
2. Contiguous subarray + sum → **prefix sum + hashmap**
3. Sorted array + find a pair → **two pointers**
4. Longest/shortest *contiguous* window → **sliding window**
5. Sorted input, or "minimise the maximum" → **binary search (possibly on the answer)**
6. "Next greater/smaller" → **monotonic stack**
7. Cycle / middle / k-th from end in a list → **fast & slow pointers**
8. Level / closest / shortest unweighted → **BFS**
9. Weighted shortest path → **Dijkstra**
10. Dependencies / ordering / prerequisites → **topological sort**
11. Dynamic connectivity, MST → **union-find**
12. "Top k" / "k-th" / streaming median → **heap** (min-heap of size k)
13. "Generate all" → **backtracking**
14. Answer depends on subtree results → **DFS returning a value**
15. Answer can be centred anywhere → **return one thing, record another**
16. Overlapping subproblems + optimal substructure → **DP**
17. Two sequences compared → **2-D DP grid**
18. `[start, end]` pairs → **sort first; the endpoint you sort by is the design decision**

---

*Good luck. Revisit the ❌ list before anything else.*
