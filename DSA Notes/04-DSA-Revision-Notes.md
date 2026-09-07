# DSA Revision Notes (Complete Master Sheet)

> Comprehensive revision notes across foundational topics, derived via the Socratic method in Java. Contains recognition cues, core derivations, complete code solutions, edge-case traps, and cross-cutting meta-lessons.

---

## Table of Contents

1. [Part A — Bonus Problems (Kadane & Array Optimizations)](#part-a--bonus-problems-not-on-the-75-sheet-but-high-value)
2. [Section 1 — Arrays & Hashing](#section-1--arrays--hashing-complete-all-66-)
3. [Section 2 — Two Pointers](#section-2--two-pointers)
4. [Section 3 — Sliding Window](#section-3--sliding-window-44-complete)
5. [Section 4 — Binary Search](#section-4--binary-search--revision-notes)
6. [Section 5 — Stack & Monotonic Stack](#section-5--stack--monotonic-stack--revision-notes)
7. [Section 6 — Linked List](#section-6--linked-list--revision-notes)
8. [Section 7 — Trees & BST](#section-7--trees--bst--revision-notes)

---

# Part A — Bonus Problems (not on the 75-sheet, but high-value)

### 1. Kadane's Algorithm (Maximum Subarray)
**State:** `dp[i]` = max subarray sum **ending exactly at index i** (not "up to i" — that loses contiguity and would allow non-adjacent jumps, turning it into a subsequence problem instead of subarray).

**Recurrence:** `dp[i] = max(dp[i-1] + nums[i], nums[i])` → equivalently `nums[i] + max(dp[i-1], 0)`. The `max(dp[i-1], 0)` framing makes the intuition explicit: carry forward the previous sum only if it's net-positive; otherwise treat it as dead weight and restart.

**Why no memoization needed:** it's a linear chain — `dp[i]` depends on exactly one prior state, computed once, never revisited via multiple paths. Contrast with Fibonacci-style branching recursion where the same subproblem is reached from multiple call paths.

**Space optimization:** O(1) — only need `dp[i-1]`, replace array with a running variable.

**The "aha":** Kadane's isn't really "DP that we got lucky with" — it's a case where the locally-greedy choice (extend if positive, restart if negative) provably yields the global optimum, because addition can never flip a negative running sum into something worth keeping just by adding one more element. This is NOT true in general DP (e.g. House Robber — skipping now can matter for options two steps ahead).

**Bug I hit:** initializing `maxValue = Integer.MIN_VALUE` and then never comparing it against the very first element before the loop starts — silently wrong for single-negative-element arrays like `[-5]`. Fix: seed `maxValue = nums[0]` (or `dp[0]`), not `MIN_VALUE`.

**Solution (Java):**
```java
public int maxSubArray(int[] nums) {
    int currentSum = nums[0];
    int maxSum = nums[0];

    for (int i = 1; i < nums.length; i++) {
        currentSum = Math.max(nums[i], currentSum + nums[i]);
        maxSum = Math.max(maxSum, currentSum);
    }
    return maxSum;
}
```

**TC:** O(n) · **SC:** O(1)

---

### 2. Maximum Product Subarray
**Why Kadane's breaks here:** multiplication (unlike addition) can flip sign. A very negative running product can become the best answer with just one more negative multiplier. Addition can never do this — a negative running sum stays "bad" no matter what you add.

**State:** track BOTH `maxDp[i]` and `minDp[i]` (max and min product of subarray ending at i) — because a very negative `minDp[i-1]`, multiplied by a negative `nums[i]`, can become the new max.

**Recurrence (unconditional — no sign-branching needed):**
```
candidate1 = nums[i] * maxDp[i-1]
candidate2 = nums[i] * minDp[i-1]
candidate3 = nums[i]
maxDp[i] = max(candidate1, candidate2, candidate3)
minDp[i] = min(candidate1, candidate2, candidate3)
```
Taking max/min over all 3 candidates unconditionally absorbs the sign logic — no need to branch on whether `nums[i]` is positive or negative. This is simpler and less bug-prone than conditional branching.

**Base case:** `maxDp[0] = minDp[0] = nums[0]`

**Key gotcha I initially got wrong:** thought I needed to check `maxDp` (already-updated) when computing `minDp` on the next line — but since `candidate1`/`candidate2` are pre-computed into separate local variables *before* either `maxDp`/`minDp` gets reassigned, there's no stale-value bug. Lesson: the danger isn't "using a variable you're about to overwrite," it's "using a variable *after* it's been reassigned." Precomputing candidates into locals sidesteps this entirely.

**Solution (Java):**
```java
public int maxProduct(int[] nums) {
    int maxProduct = nums[0];
    int maxDp = nums[0];
    int minDp = nums[0];

    for (int i = 1; i < nums.length; i++) {
        int candidate1 = nums[i] * maxDp;
        int candidate2 = nums[i] * minDp;
        int candidate3 = nums[i];

        maxDp = Math.max(candidate3, Math.max(candidate1, candidate2));
        minDp = Math.min(candidate3, Math.min(candidate1, candidate2));

        maxProduct = Math.max(maxProduct, maxDp);
    }
    return maxProduct;
}
```

**TC:** O(n) · **SC:** O(1)

---

### 3. Best Time to Buy and Sell Stock I
**Initially over-thought this** — reached for the general buy/sell state-machine DP (with a "holding" flag) that's needed for Stock II (multiple transactions). That machinery is overkill here.

**Key realization:** with only ONE transaction allowed, the problem reduces to: for every day `j`, "what's the best profit if I sell today?" = `prices[j] - min(prices[0..j-1])`. Only need to remember the **minimum price seen so far** — no day-tracking, no state machine.

**Recurrence (conceptually):** `dp[i] = prices[i] - minSoFar`, then `maxProfit = max(maxProfit, dp[i])`.

**Critical ordering bug I avoided (correctly, first try):** compute the day's potential profit using **yesterday's** `minSoFar` *before* updating `minSoFar` with today's price — otherwise you'd allow buying and selling on the same day using today's price as both, which is invalid.

**Solution (Java):**
```java
public int maxProfit(int[] prices) {
    int minSoFar = prices[0];
    int maxProfit = 0;

    for (int i = 1; i < prices.length; i++) {
        int profitToday = prices[i] - minSoFar;
        maxProfit = Math.max(maxProfit, profitToday);
        minSoFar = Math.min(minSoFar, prices[i]);
    }
    return maxProfit;
}
```

**TC:** O(n) · **SC:** O(1)

---

### 4. Next Permutation (not on sheet, pure array manipulation — no DP)
**Core insight — descending suffix = local maximum:** scan from the right to find the first index `i` where `arr[i] < arr[i+1]`. Everything from `i+1` to the end is the longest **descending** run, which is already at its maximum possible arrangement — like `1999` before rolling to `2000`.

**Algorithm:**
1. Find break index `i` scanning right-to-left where `arr[i] < arr[i+1]`. If none found (whole array descending, e.g. `[3,2,1]`), the array is the last permutation — just reverse the whole thing to get the smallest (first) permutation.
2. In the suffix, find the **smallest value greater than `arr[i]`** — scanning right-to-left, the *first* value `> arr[i]` you hit is guaranteed to be the smallest such value (since the suffix is descending, values increase as you move left through it).
3. Swap `arr[i]` with that value.
4. Reverse the suffix (`i+1` to end) — since it's still descending (max arrangement) after the swap, reversing gives the minimum arrangement, which is what you want for the *immediate* next permutation (smallest possible increase).

**Why smallest-greater-value, not any larger value:** minimizes the increase at position `i`, keeping the result as close as possible to the original — that's what "next" permutation means.

**Solution (Java):**
```java
public void nextPermutation(int[] nums) {
    int n = nums.length;
    int i = n - 2;

    // 1. Find the first break point from right where nums[i] < nums[i + 1]
    while (i >= 0 && nums[i] >= nums[i + 1]) {
        i--;
    }

    // 2. If a break point was found, find the smallest greater element in suffix to swap with
    if (i >= 0) {
        int j = n - 1;
        while (nums[j] <= nums[i]) {
            j--;
        }
        swap(nums, i, j);
    }

    // 3. Reverse the suffix from i + 1 to end
    reverse(nums, i + 1, n - 1);
}

private void swap(int[] nums, int i, int j) {
    int temp = nums[i];
    nums[i] = nums[j];
    nums[j] = temp;
}

private void reverse(int[] nums, int start, int end) {
    while (start < end) {
        swap(nums, start, end);
        start++;
        end--;
    }
}
```

**TC:** O(n) (three O(n) passes: find break point, find swap target, reverse) · **SC:** O(1), in-place

---

# Section 1 — Arrays & Hashing (complete, all 6/6 ✅)

### 1. Two Sum
Standard single-pass hashmap: `HashMap<value, index>`. Check `map.containsKey(target - nums[i])` **before** inserting `nums[i]` into the map on the same iteration — this ordering prevents matching an element with itself.

Why not `HashSet`: need to return **indices**, and a HashSet can't carry that value alongside membership.

**Solution (Java):**
```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>();

    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (map.containsKey(complement)) {
            return new int[]{ map.get(complement), i };
        }
        map.put(nums[i], i);
    }
    return new int[]{};
}
```

**TC:** O(n) · **SC:** O(n)

---

### 2. Group Anagrams
**Key design decision:** the "key" for grouping is the **sorted version** of each string — anagrams produce identical sorted strings. `HashMap<String sortedKey, List<String> originalWords>`.

**Bug to avoid:** append the **original** string to the group, not the sorted key — otherwise you lose the actual words.

**Solution (Java):**
```java
public List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> map = new HashMap<>();

    for (String s : strs) {
        char[] chars = s.toCharArray();
        Arrays.sort(chars);
        String sortedKey = new String(chars);

        if (!map.containsKey(sortedKey)) {
            map.put(sortedKey, new ArrayList<>());
        }
        map.get(sortedKey).add(s); // append original word, not key
    }
    return new ArrayList<>(map.values());
}
```

**TC:** O(n · L log L) — sort each of n strings, each costing O(L log L).
**SC:** O(n · L) — not O(n); must account for total characters stored (both keys and original strings), not just number of map entries.

---

### 3. Product of Array Except Self
**No division allowed** (division would crash on any `0` in the array) and **O(1) extra space** required (excluding the output array).

**Technique:** `output[i] = (product of everything left of i) × (product of everything right of i)`.
- Base cases: `prefix[0] = 1`, `suffix[n-1] = 1` (multiplicative identity — like `0` is for addition in prefix-sum problems; using `0` here would zero out everything).
- **Space collapse:** build prefix products directly into the `output[]` array (first left-to-right pass), then do a second right-to-left pass with a single running `suffixProduct` variable (init `1`), doing `output[i] *= suffixProduct; suffixProduct *= nums[i];` at each step.

**Solution (Java):**
```java
public int[] productExceptSelf(int[] nums) {
    int n = nums.length;
    int[] output = new int[n];

    // Left prefix pass: output[i] contains product of all elements to the left of i
    output[0] = 1;
    for (int i = 1; i < n; i++) {
        output[i] = output[i - 1] * nums[i - 1];
    }

    // Right suffix pass: collapse suffixProduct into output array in-place
    int suffixProduct = 1;
    for (int i = n - 1; i >= 0; i--) {
        output[i] *= suffixProduct;
        suffixProduct *= nums[i];
    }

    return output;
}
```

**TC:** O(n) · **SC:** O(1) extra (output array doesn't count against the constraint)

---

### 4. Longest Consecutive Sequence
**Naive trap:** for every element, walk forward counting the consecutive chain — but this re-walks overlapping parts of the same chain repeatedly (e.g. starting the count from `1`, `2`, `3`, `4` all separately when they're one chain) → O(n²) in the worst case.

**Fix — only count from true chain heads:** a number `num` is a chain head if `num - 1` is **NOT** in the set. This is an O(1) check per number. Only chain heads trigger the inner counting `while` loop.

**Amortized O(n) argument:** every element belongs to exactly one chain and gets visited by the inner while-loop **at most once total**, across the *entire* algorithm — not once per outer-loop iteration. Total work across all chains combined = O(n).

**Real TLE bug I hit on LeetCode:** iterating the outer loop over the raw `nums` array (not deduplicated) — with heavy duplicates (e.g. 10,000 copies of the same value), the "is this a head" check passes every single time for that duplicated value, and the full chain gets recounted from scratch 10,000 times → O(n·k), effectively O(n²) on adversarial input. **Fix:** iterate the outer loop over the `HashSet` (deduplicated), not the original array. Passed all small test cases *before* the fix — only adversarial/duplicate-heavy input exposed it.

**General lesson:** when "only process X once" logic depends on a *condition* rather than *structurally* guaranteeing uniqueness (like iterating a deduped collection), always ask: can the same value trigger this condition multiple times if duplicated in the input?

**Edge case bug:** `count` initialized to `1` returns wrong answer (`1`) for an **empty array** — should be `0`. Fixed by defaulting `count = 0`.

**Solution (Java):**
```java
public int longestConsecutive(int[] nums) {
    if (nums.length == 0) return 0;

    Set<Integer> set = new HashSet<>();
    for (int num : nums) {
        set.add(num);
    }

    int longestStreak = 0;

    // Iterate over the deduplicated HashSet, NOT the raw array, to prevent TLE on duplicates
    for (int num : set) {
        // Check if num is a true chain head
        if (!set.contains(num - 1)) {
            int currentNum = num;
            int currentStreak = 1;

            while (set.contains(currentNum + 1)) {
                currentNum++;
                currentStreak++;
            }

            longestStreak = Math.max(longestStreak, currentStreak);
        }
    }
    return longestStreak;
}
```

**TC:** O(n) · **SC:** O(n)

---

### 5. Subarray Sum Equals K
**Ruled out first:** sorting (destroys contiguity/index info — this problem needs contiguous subarrays) and sliding window (relies on sum changing monotonically as the window shrinks; negative numbers break that assumption entirely).

**Technique — prefix sum + hashmap:** `sum(i,j) = prefix[j] - prefix[i-1]`. Want `sum(i,j) == k` → rearranged: `prefix[i-1] = prefix[j] - k`. So at each index `j`, look up how many times the value `(prefix[j] - k)` has occurred as a prefix sum before.

**Critical map semantics (different from Two Sum!):** `HashMap<prefix sum value, count of occurrences>` — NOT value→index. You don't care *where* a prefix sum occurred, only *how many times*, since each occurrence is a valid starting point.

**`map.put(0, 1)` before the loop — why compulsory:** represents the "empty prefix" (sum of zero elements = 0), which accounts for subarrays that start at index 0. Without it, any subarray from index 0 summing exactly to `k` is silently undercounted (verified via `[1,1,1], k=2` — one of the two valid subarrays would be missed).

**Structural distinction vs. Kadane's** (same "subarray"+"sum" keywords, totally different technique): optimization problems (find the *single best*) can often collapse state into O(1) running variables, since only the best matters and everything else can be discarded. Exact-count/existence problems must remember **all** distinct prior values (and their frequencies) because you need to match an exact target, not just track a running extremum — hence hashmap, not a DP scalar.

**Solution (Java):**
```java
public int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> prefixCounts = new HashMap<>();
    prefixCounts.put(0, 1); // compulsory: represents empty prefix for subarrays starting at index 0

    int runningSum = 0;
    int count = 0;

    for (int num : nums) {
        runningSum += num;
        if (prefixCounts.containsKey(runningSum - k)) {
            count += prefixCounts.get(runningSum - k);
        }
        prefixCounts.put(runningSum, prefixCounts.getOrDefault(runningSum, 0) + 1);
    }
    return count;
}
```

**TC:** O(n) · **SC:** O(n)

---

### 6. First Missing Positive (Hard)
**Key range insight:** for an array of size `n`, the answer is always in `[1, n+1]` — bounded regardless of what garbage (negatives, zeros, huge numbers, duplicates) is in the array. If `nums` contains exactly `{1,...,n}` with no gaps, the answer is `n+1`. This bounded range is what makes O(1) space possible.

**Technique — index-as-hash (cyclic sort):** use the array itself as a hashtable. Place each value `v` (where `1 ≤ v ≤ n`) at index `v-1`, via **swapping** (not overwriting) so no information is lost — the displaced value moves to where the swap came from and gets its own chance to be placed correctly.

**Placement loop (per index i):**
```java
while (nums[i] >= 1 && nums[i] <= n && nums[nums[i]-1] != nums[i]) {
    swap(nums, i, nums[i]-1);
}
```
Two safeguards, both necessary:
- **Range check** (`1 ≤ nums[i] ≤ n`): ignore anything outside the possible answer range (negatives, zeros, values > n).
- **Equality check** (`nums[nums[i]-1] != nums[i]`): prevents infinite loop on duplicates — if the target position already holds the same value (e.g. `[1,1]`), swapping is a no-op that would loop forever without this guard.

**Scan phase:** first `i` where `nums[i] != i+1` → answer is `i+1`. If no mismatch found after full scan → answer is `n+1` (initially mis-answered this as just `n` — corrected via trace of `[1,2,3,4]`).

**Amortized O(n) proof:** every swap places exactly one value into its **permanent** final position (never moved again, due to the equality-check guard). With only `n` positions total, total swaps across the *entire* algorithm (summed over all outer-loop iterations) ≤ n. So despite the nested for+while structure, it's O(n), not O(n²).

**Solution (Java):**
```java
public int firstMissingPositive(int[] nums) {
    int n = nums.length;

    // Cyclic sort: place each value v in [1, n] at index v - 1
    for (int i = 0; i < n; i++) {
        while (nums[i] >= 1 && nums[i] <= n && nums[nums[i] - 1] != nums[i]) {
            swap(nums, i, nums[i] - 1);
        }
    }

    // Scan phase: find first index i where nums[i] != i + 1
    for (int i = 0; i < n; i++) {
        if (nums[i] != i + 1) {
            return i + 1;
        }
    }

    // If 1 through n are all present, the answer is n + 1
    return n + 1;
}

private void swap(int[] nums, int i, int j) {
    int temp = nums[i];
    nums[i] = nums[j];
    nums[j] = temp;
}
```

**TC:** O(n) · **SC:** O(1) extra

---

### Meta-lessons carried forward from Session 1

1. **State-definition sufficiency check:** before trusting a recurrence, ask "does this state capture everything the recurrence needs?" (Max Product Subarray — single `dp[i]` wasn't enough; needed max AND min.)
2. **Dependency direction dictates traversal order** — not intuition/"feels natural." Fill lower-index-dependent states before higher ones.
3. **Overlapping subproblems test:** ask "is this a linear chain (each state computed once, never revisited) or a branching tree (same subproblem reached via multiple paths)?" — determines whether memoization is even relevant.
4. **The "process once" trap:** a condition-based uniqueness check (e.g. "is this a chain head?") can fire multiple times if the *iteration space* isn't itself deduplicated. Structural guarantees (iterating a Set) are safer than relying purely on a runtime condition.
5. **Precompute-then-reassign pattern** avoids stale-variable bugs in simultaneous multi-state updates (Max Product Subarray candidates) — safer than trying to reason about assignment order.
6. **Always trace edge cases by hand before trusting a solution:** empty array, single element, all-duplicates, all-negative — these are exactly where subtle bugs (wrong sentinel init, infinite loops, off-by-one in n vs n+1) hide, and passing small test cases doesn't guarantee correctness on adversarial input (see: Longest Consecutive Sequence TLE).

---

# Section 2 — Two Pointers

**Status:** Section 2 (Two Pointers) — **4/4 complete and verified on LeetCode.**
✅ 3Sum (LC 15) · ✅ Container With Most Water (LC 11) · ✅ Sort Colors (LC 75) · ✅ Trapping Rain Water (LC 42)

---

## How to recognize "Two Pointers" as a family — the real unifying idea

Across all four problems, "two pointers" isn't one trick — it's four different flavors that all lean on the same underlying move: **prove, right now, that moving/eliminating one option can never produce something worse than what full information would give you.** If you can state that proof for a new problem, you've found the pattern. If you can't, it's probably not this.

**Decision checklist:**

| Signal in the problem | Flavor | Core mechanic |
|---|---|---|
| Array sorted (or safely sortable) + matching a target sum/relationship, values not indices | **3Sum-style** | Sort, fix one, two-pointer the rest; `sum <> target` drives movement |
| No target — maximizing something bounded by the *shorter* of two chosen values; moving the taller one is provably useless | **Container-style** | Eliminate the dominated option; pointers converge from both ends |
| Small fixed set of distinct values (e.g. only 0/1/2); need one-pass partitioning into regions | **Sort Colors-style** | Three-pointer region invariant — a partitioning scan, not really "converging" pointers |
| Per-index answer depends on max/min of everything to the left **and** right; one side's bound can be proven "locked in" by comparing two running trackers | **Trapping Rain Water-style** | Two running maxes; whichever tracker is smaller determines which side is safe to finalize |

---

## 1. 3Sum (LC 15) — ✅

### Recognition cue
"Find pair/triplet whose sum equals a target" + no positional/index constraint in the output (only values matter) → sort is "safe" (you don't lose anything the problem cares about) + two pointers becomes viable on the sorted array.

### Approach derived
1. **Sort** the array.
2. Outer loop `i` from `0` to `n-1`, fixing one element.
3. Inner two-pointer scan: `lo = i+1`, `hi = n-1`.
   - `sum == 0` → record triplet, then skip duplicates, then advance both.
   - `sum < 0` → `lo++` (need bigger).
   - `sum > 0` → `hi--` (need smaller).
4. **Early exit:** once `nums[i] > 0` (strictly greater, not `>=`), break — no triplet starting there can ever sum to 0, since everything after is ≥ `nums[i]`.

### Final code shape
```java
Arrays.sort(nums);
for (int i = 0; i < n; i++) {
    if (nums[i] > 0) break;                              // early exit
    if (i > 0 && nums[i] == nums[i-1]) continue;          // skip duplicate anchors

    int lo = i+1, hi = n-1;
    while (lo < hi) {
        int sum = nums[i] + nums[lo] + nums[hi];
        if (sum == 0) {
            res.add(Arrays.asList(nums[i], nums[lo], nums[hi]));
            while (lo < hi && nums[lo] == nums[lo+1]) lo++;
            while (lo < hi && nums[hi] == nums[hi-1]) hi--;
            lo++; hi--;
        } else if (sum < 0) lo++;
        else hi--;
    }
}
```

### Complexity
- **TC: O(N²)** — sort is O(N log N), outer+inner is O(N²); these are sequential phases (added, not multiplied), so N² dominates.
- **SC: O(log N)** auxiliary (Java's dual-pivot quicksort recursion stack on primitives) **+ O(N)** for mandatory output — state both, and note which is "algorithmic overhead" vs "required output."

### Traps hit this session
- **Duplicate-skip direction — THE recurring bug.** Initially wrote `if (i > 0 && nums[i] == nums[i+1]) skip` — compares to the element **ahead**, which never fires correctly. Must compare to `nums[i-1]` (the element **already processed**) with an `i > 0` guard. Mnemonic: *"skip if I've already handled this value as an anchor before"* — that's inherently a look-**back**, not look-ahead.
- **Early-exit condition:** first instinct was `nums[i] >= 0`, which incorrectly skips processing `i` when `nums[i] == 0` itself — but `[0,0,0]` is a valid triplet. Must be strictly `> 0`, so the zero anchor still gets its two-pointer scan.
- **Space complexity mix-up:** initially claimed Java's dual-pivot quicksort takes O(N) extra space — wrong, it's **O(log N)** (recursion stack). O(N) is what *merge sort* needs. `Arrays.sort()` on primitives uses dual-pivot quicksort (O(log N) space); on objects it uses a Timsort-like merge (O(N) space) — worth knowing which you're calling.

---

## 2. Container With Most Water (LC 11) — ✅

### Recognition cue
Two pointers at the extremes of an **unsorted** array (sorting would destroy the positional/width information you need), maximizing `width × min(height[lo], height[hi])`, with **no target value** — this is optimization via elimination-proof, not target-matching. Fundamentally different flavor from 3Sum despite both being "two pointers."

### The core proof (this IS the problem — be able to state it cold)
Given `lo`, `hi` with `height[lo] < height[hi]` (left is the shorter wall):
- **Moving `hi` inward (the taller side):** width strictly decreases; height cap is still bounded by `height[lo]` (unchanged) — so the new area can **never** exceed the old one. Provably useless.
- **Moving `lo` inward (the shorter side):** width still decreases, but the *next* `height[lo']` could be taller than the old one — height cap **could** increase. Not guaranteed to help, but it's the only branch that keeps the possibility of a better answer alive.
- Since moving the taller side is *provably* never better, it's always safe to discard that option and move the shorter side. This also never "jumps over" the true optimum — pointers move one index at a time, so every index is visited by the boundary that could still improve, meaning no candidate pair is skipped.

### Final code shape
```java
int lo = 0, hi = n - 1, maxArea = 0;
while (lo < hi) {
    int minHeight = Math.min(height[lo], height[hi]);
    int width = hi - lo;
    maxArea = Math.max(maxArea, minHeight * width);
    if (height[lo] > height[hi]) hi--; else lo++;   // always move the shorter side
}
```

### Complexity
- **TC: O(N)** — each pointer moves inward monotonically; total moves bounded by N.
- **SC: O(1)**.

### Traps hit this session
- None on implementation — this one went cleanly once the elimination proof was internalized. The trap is purely conceptual: **don't skip the proof.** "Move the shorter side" without being able to justify *why* the taller side is provably useless is a guess dressed up as an algorithm, and won't survive interviewer follow-ups.
- Minor: used a custom method name (`containerWithMaxWater` vs LeetCode's expected `maxArea`) for personal searchability — fine as a convention, just remember to match the actual function stub when submitting.

---

## 3. Sort Colors — Dutch National Flag (LC 75) — ✅

### Recognition cue
Array has a **small, fixed set of distinct values** (here: 0, 1, 2) + LeetCode follow-up explicitly demands **one-pass** and **O(1) extra space**. This combination is what rules out counting sort (which needs 2 passes: count, then overwrite) and points to **three-pointer region-partitioning** instead.

### Invariant (the whole problem, stated precisely)
At any point during the scan, the array is partitioned into four regions:

| Region | Meaning |
|---|---|
| `0 .. low-1` | confirmed `0`s |
| `low .. mid-1` | confirmed `1`s |
| `mid .. high` | **unexplored** |
| `high+1 .. n-1` | confirmed `2`s |

`low` and `mid` start at `0`; `high` starts at `n-1` (entire array unexplored initially).

### Transition rules
- `nums[mid] == 0`: swap `nums[low] ↔ nums[mid]`, then `low++` **and** `mid++`.
  - Safe to advance `mid` because the value swapped **into** `mid` from `low` is *always* a known `1` (or a `0` swapping with itself, when `low == mid`) — never something unexamined.
- `nums[mid] == 1`: just `mid++` (already in the right place).
- `nums[mid] == 2`: swap `nums[mid] ↔ nums[high]`, then `high--` **only** (do NOT advance `mid`).
  - `mid` must stay because the value swapped in from `high` is **unknown** — `high` was the boundary of the unexplored region, not a region with a guaranteed uniform value. It needs inspection on the next iteration.

### Final code shape
```java
int low = 0, mid = 0, high = n - 1;
while (mid <= high) {                 // ⭐ NOT mid < high — see trap below
    if (nums[mid] == 0) {
        swap(nums, low, mid);
        low++; mid++;
    } else if (nums[mid] == 1) {
        mid++;
    } else {
        swap(nums, mid, high);
        high--;
    }
}
```

### Complexity
- **TC: O(N)** — `low`, `mid`, `high` each move monotonically; total pointer movements bounded by ~2N.
- **SC: O(1)**.

### Traps hit this session
- **THE bug: loop condition `mid < high` instead of `mid <= high`.** Root cause: habit-bleed from binary search's `lo < hi` / `lo <= hi` conventions. Caught via manual trace on `[1,2,0]` — with `mid < high`, the loop exits while `mid == high` still points at an unprocessed index, silently leaving it unsorted (`[1,0,2]` instead of `[0,1,2]`).
  - **Why `<=` is correct:** the unexplored region is defined as `mid..high` **inclusive on both ends** — so when `mid == high`, that single index is still unexplored and still needs evaluation.
  - **Personal trap to flag explicitly:** *"Don't default to binary search's loop-boundary habits on other pointer problems — re-derive the boundary condition from the actual region invariant, every time."* This exact bug recurred in Trapping Rain Water later in the same session — it is the single highest-frequency mistake pattern right now.
- Initial region-boundary confusion: first pass had `mid..high` labeled as "confirmed 2s" and `high+1..n-1` as "unexplored" — backwards. Resolved by tracing which pointer moves in which direction (`high` retreats leftward as it *confirms* 2s, so the confirmed-2 region must be *after* `high`, not before).

---

## 4. Trapping Rain Water (LC 42) — ✅ (Hard)

### Recognition cue
Per-index answer depends on `min(leftMax, rightMax)` — i.e., you need to know the tallest wall on **both** sides of every index simultaneously. The "aha" that unlocks the O(1)-space version: you don't need the *array* of left/right maxes, just **two running trackers**, because whichever tracker is currently smaller has a value that's already "locked in" (proven below) — directly mirroring the Container With Most Water elimination proof.

### Core formula
For any index `i`: **`water[i] = min(leftMax, rightMax) - height[i]`** (single formula — not two separate left/right subtractions; there is exactly *one* water level per index, and leftMax/rightMax both just feed into computing that one number via `min`).

### Progression used (three stages, in order)

**Stage 1 — Brute force.** For each `i`, scan left and right independently to find `leftMax`/`rightMax`. **TC: O(N²)**, too slow for n up to 2×10⁴ (~4×10⁸ ops).

**Stage 2 — Precompute arrays.** One left-to-right pass builds `leftMax[]` (running max), one right-to-left pass builds `rightMax[]` (running max), one final pass sums `min(leftMax[i], rightMax[i]) - height[i]`.
- **TC: O(N)** (three sequential O(N) passes — constants drop in Big-O, don't say "O(3N)").
- **SC: O(N)** (two full auxiliary arrays — not O(1), regardless of whether you also materialize a `water[]` array).

**Stage 3 — Two pointers, O(1) space (the actual target for this section).**

### The proof (mirrors Container With Most Water — recognize the family resemblance)
Maintain **running** trackers `leftMax`, `rightMax` (not arrays) alongside pointers `lo`, `hi`.
- At any iteration, compare the two running trackers. Say `leftMax <= rightMax`.
- `leftMax` (max of `height[0..lo]`) could still grow as `lo` moves further right — it is *not* necessarily final yet.
- `rightMax` (max of `height[hi..n-1]` so far) can **only stay the same or grow** as `hi` moves further left (running max never shrinks) — so the *true, final* rightMax for this position is guaranteed to be **≥** the current tracked value.
- Since `leftMax <= (current) rightMax <= (true, final) rightMax`, you can conclude **`min(leftMax, true_rightMax) == leftMax`** for certain, no matter what `rightMax` grows to later.
- Therefore `water[lo] = leftMax - height[lo]` can be safely computed **right now**, using only the current tracker — no need to ever know the array's true global rightMax. Process `lo`, advance it.
- Symmetric logic when `rightMax <= leftMax`: process `hi` instead, using `rightMax`.
- **Important:** this comparison is re-evaluated **every iteration** — it's not "assume `leftMax < rightMax` for the whole run." Which side is smaller can flip iteration to iteration; whichever is smaller *at that moment* dictates which pointer moves *that* iteration.

### Update-order subtlety
Within the processed side (say `lo`), order matters:
1. **First** check if `height[lo]` is a new max: if `height[lo] >= leftMax`, update `leftMax = height[lo]` (implies water there is `0` — a new tallest-so-far wall cannot have water sitting on it — no separate branch needed, this falls out naturally).
2. **Else**, `leftMax` stays the old (larger) value, and compute `water[lo] += leftMax - height[lo]`.

Doing it in the opposite order (compute before checking for a new max) would use a stale `leftMax` and produce a wrong (positive) water value at a new-max index, which is physically impossible.

### Final code shape
```java
int lo = 0, hi = n - 1;
int leftMax = height[0], rightMax = height[n - 1];
int result = 0;

while (lo <= hi) {                              // ⭐ NOT lo < hi — see trap below
    if (leftMax <= rightMax) {
        if (height[lo] > leftMax) leftMax = height[lo];
        else result += leftMax - height[lo];
        lo++;
    } else {
        if (height[hi] > rightMax) rightMax = height[hi];
        else result += rightMax - height[hi];
        hi--;
    }
}
```

### Complexity
- **TC: O(N)**, **SC: O(1)**.

### Traps hit this session
- **THE bug (again): loop condition `lo < hi` instead of `lo <= hi`.** Same root cause as Sort Colors — assumed the boundary case didn't matter, tested empirically instead of proving it, and LeetCode's own test suite didn't catch it (passed with the buggy `<` version). Only caught by manually constructing a counterexample.
  - **Counterexample that exposed it:** `height = [5, 1, 6, 1, 5]`. With `lo < hi`, index `3` (`height[3]=1`) never gets processed as the loop exits one iteration early, silently losing its water contribution. Result: `4` (wrong) vs. `8` (correct, verified independently against the Stage 2 precompute-array method).
  - **Why `<=` is correct:** when `lo == hi`, that single shared index has **not** necessarily already had its water accounted for — `leftMax`/`rightMax` at that point represent maxes from the already-processed regions **strictly outside** that index, and the shared index can still independently trap water between those two bounds.
  - **Meta-lesson, worth internalizing hard:** *passing LeetCode's test suite is not proof of correctness* — it only means no counterexample in their suite exposed the bug. Cross-check against a slower, more obviously-correct method (like the Stage 2 array approach) whenever a boundary condition feels uncertain, rather than trusting "it passed."

---

## Cross-Problem Meta-Traps (Section 2 summary)

1. **Duplicate-skip direction is always "look backward, not forward"** (3Sum: `nums[i-1]`, not `nums[i+1]`). The rule: you're checking "have I already fully handled this value," which is inherently about the past, not the future.
2. **Loop boundary conditions (`<` vs `<=`) do not transfer between patterns.** Binary search's `lo < hi` habit bled into *both* Sort Colors and Trapping Rain Water this session, causing the exact same class of bug twice. **Going forward: for every new two/three-pointer problem, explicitly re-derive the boundary condition from the region/index invariant — never default to a remembered convention from a different pattern.**
3. **LeetCode passing ≠ correctness.** Two boundary bugs this session initially passed LeetCode's suite (Trapping Rain Water with `lo < hi`) or would have if not caught by hand-tracing first. When a boundary condition feels arbitrary or "seems to work either way," construct your own counterexample rather than trusting the online judge's test coverage.
4. **State Big-O properly — drop constants.** Said "O(4N)"/"O(3N)" more than once this session; both simplify to O(N). Interviewers expect the simplified class, not the raw pass count.
5. **The elimination-proof habit is the actual transferable skill.** Container With Most Water and Trapping Rain Water are structurally the same proof reused: "the smaller of two tracked bounds is already final w.r.t. this comparison, so it's safe to act on now." Recognizing this connection is what turns "75 memorized solutions" into "~18 patterns," per the sheet's own stated goal.


---

# Section 3 — Sliding Window (4/4 Complete)

**Problems:** Longest Substring Without Repeating Characters ✓ | Longest Repeating Character Replacement ✓ | Minimum Window Substring ✓ | Sliding Window Maximum ✓

---

## The Big Picture: Three Distinct Sub-Patterns in This Section

Don't lump all four problems into "sliding window" as one mental bucket — they use three genuinely different mechanisms:

| Mechanism | Problems | Validity check |
|---|---|---|
| **Direct membership check** | Longest Substring w/o Repeat | "Is this exact character already in the window?" |
| **Derived-quantity check** | Longest Repeating Char Replacement | `(windowLength - maxFreq) > k` |
| **Multi-requirement dual-counter** | Minimum Window Substring | `matched == required` (have/need maps) |
| **Monotonic deque (not a shrink window at all)** | Sliding Window Maximum | Front of deque = max; fixed-size window |

Recognizing *which* of these four a new problem needs is the actual skill — not memorizing four solutions.

---

## 1. Longest Substring Without Repeating Characters (LC 3)

### Recognition cue
"Longest **contiguous** substring/subarray with **no repeats** (distinctness constraint)." Contiguity + distinctness → sliding window with a hashmap of last-seen indices.

### Core mechanism
- `HashMap<Character, Integer> lastSeen` — maps char → most recent index seen.
- On each `right`, if char is in the map, `left = Math.max(left, lastSeen.get(char) + 1)`.
- The `Math.max` is what protects against a **stale** lastSeen index (one from *before* the current window) incorrectly yanking `left` backward.
- Record `maxLength` **after** the update (this is a "longest" problem → record after contracting/adjusting).

### Code
```java
public int lengthOfLongestSubstring(String s) {
    int n = s.length();
    HashMap<Character, Integer> map = new HashMap<>();
    int left = 0, maxLength = 0;

    for (int right = 0; right < n; right++) {
        if (map.containsKey(s.charAt(right))) {
            left = Math.max(left, map.get(s.charAt(right)) + 1);
        }
        int windowLength = right - left + 1;
        maxLength = Math.max(maxLength, windowLength);
        map.put(s.charAt(right), right);   // update AFTER using old value
    }
    return maxLength;
}
```

### TC / SC
- TC: O(N)
- SC: O(1) — bounded by charset size (~128), not string length. Don't just say O(N); say **O(min(N, charset size))**.

### Personal trap (yours)
> **Always check if the character is in the map BEFORE inserting it.** Insert-before-check makes every character look like a duplicate of itself on first sighting (`map.get(char)` returns its own current index), collapsing `left` past `right` and breaking the whole algorithm. Read-then-write, never write-then-read.

---

## 2. Longest Repeating Character Replacement (LC 424)

### Recognition cue
"At most **k** replacements/changes/removals allowed" is the tell — this is different from plain distinctness. Any time validity depends on a **budget against a computed count** (not a direct membership check), you need a derived quantity tracked alongside the window.

### Core mechanism
- `int[26]` (or HashMap) frequency count of the current window.
- `maxFreq` = running high-water mark of "most frequent character count seen **while expanding**." **Never recomputed on shrink.**
- Validity: window is invalid if `(windowLength - maxFreq) > k` (i.e., more non-majority characters than your replacement budget allows).
- Shrink is an `if`, not a `while` — because the window only ever becomes invalid by growing exactly one past the threshold, so one shrink step always restores validity.
- Record `maxLength` **after** the shrink check (longest-variant → record after contracting).

### The subtle part — why a stale `maxFreq` doesn't break correctness
`maxFreq` can overestimate the *current* window's true max frequency after a shrink (since it's never decremented). This makes the validity check **more lenient** than it should be — but:
- The window can only ever shrink by at most 1 per growth step (net `right - left` stays non-decreasing across the whole run in the cases that matter).
- Every window length you ever record was, at some earlier point, genuinely achievable with a truly accurate `maxFreq`.
- So a stale `maxFreq` can make the window "look valid" when it technically isn't for an instant, but it **never causes you to record a length that wasn't legitimately earned** — it just means the window doesn't shrink further than it "should," which doesn't produce a wrong (too-high) answer.

In short: `maxFreq` is not "max frequency in my current window" — it's "the max frequency of any character, in any window of this size or smaller, seen so far." Reframing it this way is what makes the correctness argument click.

### Code
```java
public int characterReplacement(String s, int k) {
    HashMap<Character, Integer> map = new HashMap<>();
    int n = s.length();
    int maxFreq = 0, maxLength = 0, left = 0;

    for (int right = 0; right < n; right++) {
        map.put(s.charAt(right), map.getOrDefault(s.charAt(right), 0) + 1);
        maxFreq = Math.max(maxFreq, map.get(s.charAt(right)));

        if (((right - left) + 1 - maxFreq) > k) {
            map.put(s.charAt(left), map.get(s.charAt(left)) - 1);
            left++;
        }
        maxLength = Math.max(maxLength, (right - left + 1));
    }
    return maxLength;
}
```

### TC / SC
- TC: O(N)
- SC: O(1) — at most 26 keys (uppercase English letters per LC constraints).

### Personal trap (yours)
> The "stale `maxFreq`" correctness argument is genuinely subtle — it took real time to internalize. **It cannot produce an incorrectly recorded `maxLength`; it only allows a technically-invalid window to persist an instant longer without harming the final answer.** Don't confuse "window is momentarily lenient" with "answer is wrong." This same style of argument (correctness despite a variable that isn't perfectly up to date) will reappear in monotonic stack and DP "return one thing, record another" problems — worth over-drilling now.

---

## 3. Minimum Window Substring (LC 76)

### Recognition cue
"Substring of `s` that contains **all** characters of `t`" — satisfying **multiple simultaneous count requirements at once** (not one derived scalar like `maxFreq`) is the tell for a **dual-counter (`have`/`need`) design with a `matched` scalar**.

Also: this is the **shortest**-variant sliding window, which flips *where* you record the answer compared to problems 1 and 2.

### Core mechanism
- `need[128]` — fixed requirement counts from `t`, set once, never mutated.
- `have[128]` — live counts of the current window, mutated as window grows/shrinks.
- `required` — total *distinct* characters `t` needs (not total character count).
- `matched` — how many distinct characters currently have `have[c] == need[c]` (i.e., their requirement is exactly satisfied). Increments only at the exact moment a character's count crosses from "not enough" to "exactly enough" on the way up.
- **Shrink-while-valid**: once `matched == required`, keep shrinking (a `while`, not an `if`) — record the candidate length **at every still-valid window size**, stopping the instant removing another character would break `matched`.
- This is why "record inside the shrink loop" applies here, unlike problems 1/2: you're hunting for the smallest valid window, so you must check every valid size as you shrink, not just the size after one adjustment.

### Code
```java
public String minWindow(String s, String t) {
    if (t.isEmpty() || s.length() < t.length()) return "";

    int[] need = new int[128];
    int matched = 0, required = 0;

    for (char c : t.toCharArray()) {
        if (need[c] == 0) required++;
        need[c]++;
    }

    int left = 0, minLength = Integer.MAX_VALUE, startIndex = -1;
    int[] have = new int[128];

    for (int right = 0; right < s.length(); right++) {
        have[s.charAt(right)]++;
        if (have[s.charAt(right)] == need[s.charAt(right)]) matched++;

        while (required == matched && left <= right) {
            int currentLength = right - left + 1;
            if (currentLength < minLength) {
                minLength = currentLength;
                startIndex = left;
            }
            have[s.charAt(left)]--;
            if (have[s.charAt(left)] == need[s.charAt(left)] - 1) matched--;
            left++;
        }
    }
    return startIndex != -1 ? s.substring(startIndex, startIndex + minLength) : "";
}
```

### TC / SC
- TC: O(N) — `left` and `right` each traverse the string at most once across the whole run (not multiplicatively).
- SC: O(1) — fixed-size 128-arrays, not proportional to input length.

### Personal traps (yours)
> 1. **Shrink-while-valid, record inside the loop.** Minimum/shortest window problems capture the answer *during* the shrink (checking every valid size as you contract), the mirror image of "longest" problems recording *after* contracting.
> 2. **Don't assume the charset from the problem's "vibe" — check the actual constraint.** Initially sized arrays at 26 (assumed uppercase-only) and failed on LeetCode; LC 76 allows both upper and lower case. Default to `int[128]` (or `int[256]` if symbols/extended ASCII are plausible) unless the constraint explicitly narrows it.

---

## 4. Sliding Window Maximum (LC 239)

### Recognition cue
"Max/min of **every fixed-size window**, efficiently" — specifically **max/min** (not sum — sums just use a running total) with a **fixed** window size (not variable/shrinking) is the tell for a **monotonic deque of indices**. This is a genuinely distinct tool from the shrink-based windows in problems 1–3 — not something you can derive by analogy from them.

### Core mechanism
- `Deque<Integer>` storing **indices**, not values (needed to detect staleness — a value alone can't tell you if it's still in the window, especially with duplicates).
- Deque is kept in **decreasing value order** front-to-back.
- **Back-eviction** (`while`): before pushing a new index, pop everything from the back whose value is `<= nums[right]` — those elements can never be the max again while the new, larger element is still in the window.
- **Front-eviction** (`if`, not `while`): evict the front if `deque.peekFirst() <= right - k` (index fell out of window `[right-k+1, right]`). Only ever **one** stale index can accumulate at the front per step, since indices are pushed in increasing order — hence `if` suffices, unlike the back's `while`.
- **Order independence**: front-eviction only reads *index/position*; back-eviction only reads *value*. Neither check's outcome depends on the other having run, so the order between them doesn't affect correctness — front-first is just conventional cleanliness.
- Start recording once `right >= k - 1` (the first index where a full window of size `k` exists).

### Code
```java
public int[] maxSlidingWindow(int[] nums, int k) {
    int n = nums.length;
    int[] result = new int[n - k + 1];
    Deque<Integer> deque = new ArrayDeque<>();
    int ansIndex = 0;

    for (int right = 0; right < n; right++) {
        if (!deque.isEmpty() && deque.peekFirst() <= right - k) {
            deque.pollFirst();
        }
        while (!deque.isEmpty() && nums[deque.peekLast()] <= nums[right]) {
            deque.pollLast();
        }
        deque.offerLast(right);

        if (right >= k - 1) {
            result[ansIndex++] = nums[deque.peekFirst()];
        }
    }
    return result;
}
```
*(Note: the `!deque.isEmpty()` guard on the recording `if` is dead code — the deque was just pushed to two lines above, so it can never be empty at that check.)*

### TC / SC
- TC: O(N) — each index is pushed once and popped at most once across the entire run, from either end.
- SC: O(N) worst case for the deque (strictly decreasing input never evicts), plus O(N - k + 1) for the output array. Overall O(N).

### Personal trap (yours)
> Front-eviction checks **index/position** → at most one stale entry per step → `if`. Back-eviction checks **value** → potentially many stale entries → `while`. These two checks read disjoint information and don't interact, so their relative order doesn't matter — but each individually needs the right conditional type. Confusing "index-vs-index" was the deque's realization moment: values alone can't answer "is this still in my window."

---

## Section-Wide Meta-Pattern

The record-location rule, stated once cleanly:

- **Longest/maximum-window problems** → record the answer **after** adjusting/contracting (window is guaranteed valid at that point).
- **Shortest/minimum-window problems** → record the answer **inside** the contraction loop, at every still-valid size, since you're actively searching for the smallest valid instance before it breaks.

This single distinction is the answer to "where do I put `maxLength = Math.max(...)` vs `minLength = Math.min(...)`" for any future sliding window problem — check whether you're maximizing or minimizing the window, and that tells you which side of the shrink to record on.


---

# Section 4 — Binary Search — Revision Notes

**Section 4 of the 75-problem sheet. Status: 6/6 ✅ (all LeetCode-verified)**

Problems covered: LC 34, LC 33, LC 81 (bonus), LC 153, LC 154 (bonus, untested on LC), LC 875, LC 4.

---

## Recognition Cues

Two distinct triggers — keep them separate in your head:

1. **Search a sorted / rotated-sorted array.** You're binary-searching *the input array itself*, looking for a target value or a boundary (first/last occurrence, minimum, pivot).
2. **Binary search on the answer.** The cue: *"minimise the maximum…"*, *"smallest k such that it's possible"*, plus a **monotone feasibility predicate** — if `k` works, every larger (or smaller) `k` also works. You're searching the *answer space*, not the input. Koko is the canonical drill.
3. **Partition-based search (LC 4).** A third, rarer flavour: binary search over "how many elements go left," not over values at all. Trigger: two sorted arrays, need combined median, target complexity is explicitly `O(log(min(m,n)))`.

**The half-open template `[lo, hi)` with `lo < hi`** is your default for "find a boundary" problems (34, 153). **The closed template `[lo, hi]` with `lo <= hi`** is your default for "find a target, or check every partition explicitly" problems (33, 81, 875, 4) — because in those, every iteration either returns/records or explicitly shrinks the range, and the value at `lo == hi` still needs to be checked.

---

## LC 34 — Find First and Last Position of Element in Sorted Array

**Derivation:** Standard binary search, but on hitting `nums[mid] == target`, don't stop — record the index and keep narrowing:
- First occurrence: record `ans = mid`, then `hi = mid - 1` (search left for something even earlier — `mid` is no longer useful info once recorded).
- Last occurrence: record `ans = mid`, then `lo = mid + 1` (search right).

**Final code:**
```java
private int firstOccurrence(int[] nums, int target) {
    int low = 0, high = nums.length - 1, ans = -1;
    while (low <= high) {
        int mid = low + (high - low) / 2;
        if (nums[mid] == target) { ans = mid; high = mid - 1; }
        else if (nums[mid] < target) low = mid + 1;
        else high = mid - 1;
    }
    return ans;
}

private int lastOccurrence(int[] nums, int target) {
    int low = 0, high = nums.length - 1, ans = -1;
    while (low <= high) {
        int mid = low + (high - low) / 2;
        if (nums[mid] == target) { ans = mid; low = mid + 1; }
        else if (nums[mid] < target) low = mid + 1;
        else high = mid - 1;
    }
    return ans;
}

public int[] searchRange(int[] nums, int target) {
    return new int[]{ firstOccurrence(nums, target), lastOccurrence(nums, target) };
}
```

**TC:** O(2 log n) → O(log n). **SC:** O(1).

**Empty array:** handled for free — `low > high` immediately, both helpers return `-1` untouched.

**Trap hit:** none this time — clean derivation once `hi = mid - 1` vs `hi = mid` was reasoned through explicitly (see LC 153 for where this distinction *does* bite).

---

## LC 33 — Search in Rotated Sorted Array

**Recognition:** array is sorted-then-rotated, so plain binary search breaks — but **exactly one of the two halves around any `mid` is always fully sorted.**

**Derivation:**
1. Check `if (nums[mid] == target) return mid;` first.
2. Determine which half is sorted: `nums[lo] <= nums[mid]` → left half `[lo, mid]` is sorted; else right half `[mid, hi]` is sorted.
3. If left half is sorted, check if `target` lies inside it: `nums[lo] <= target && target < nums[mid]`. If yes, search left (`hi = mid - 1`); else search right (`lo = mid + 1`). Mirror logic for the right-sorted case.

**Final code:**
```java
public int searchInRotatedSortedArray(int[] nums, int target) {
    int lo = 0, hi = nums.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] == target) return mid;

        if (nums[lo] <= nums[mid]) {
            if (nums[lo] <= target && target < nums[mid]) hi = mid - 1;
            else lo = mid + 1;
        } else {
            if (nums[mid] < target && target <= nums[hi]) lo = mid + 1;
            else hi = mid - 1;
        }
    }
    return -1;
}
```

**TC:** O(log n). **SC:** O(1).

**Why `nums[lo] <= nums[mid]` is safe here:** LC 33 guarantees **distinct** elements. In a rotated-sorted array of unique values, `nums[lo] == nums[mid]` with `lo != mid` is structurally impossible — so the "which half is sorted" check never degenerates. This safety disappears the moment duplicates are allowed (see LC 81).

---

## LC 81 — Search in Rotated Sorted Array II (bonus, duplicates allowed)

**Delta from LC 33:** when `nums[lo] == nums[mid] == nums[hi]`, the sortedness check gives **zero information** — both halves can look "sorted" by that comparison even when one is actually broken. Fallback: shrink from both ends by one.

**Final code:**
```java
public int searchInRotatedSortedArrayWithDuplicates(int[] nums, int target) {
    int lo = 0, hi = nums.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] == target) return mid;

        if (nums[lo] == nums[mid] && nums[mid] == nums[hi]) {
            lo++; hi--;
            continue;
        }

        if (nums[lo] <= nums[mid]) {
            if (nums[lo] <= target && target < nums[mid]) hi = mid - 1;
            else lo = mid + 1;
        } else {
            if (nums[mid] < target && target <= nums[hi]) lo = mid + 1;
            else hi = mid - 1;
        }
    }
    return -1;
}
```

**TC:** worst case O(n) — e.g. `[2,2,2,2]` forces the `lo++; hi--` fallback repeatedly (O(n/2) → simplify constants away → **O(n)**). Best/avg case still O(log n). **SC:** O(1).

**Meta-lesson reinforced:** don't leave intermediate forms like O(n/2) unsimplified — always reduce to the tightest standard Big-O form.

---

## LC 153 — Find Minimum in Rotated Sorted Array

**Recognition:** same rotated-array family, different question — you want the *minimum value*, not a target, and there's no target to compare against.

**Core trap (the one the sheet specifically flags):** compare `nums[mid]` to **`nums[hi]`**, never `nums[lo]`.

**Why:** `mid = lo + (hi - lo) / 2` (integer division, rounds down) guarantees `mid < hi` whenever `lo < hi`. But `mid` can equal `lo` (e.g. any 2-element window). When `mid == lo`, comparing `nums[lo]` to `nums[mid]` compares an element to itself — zero information. `nums[mid]` vs `nums[hi]` never degenerates this way.

**Derivation:**
- Loop invariant: `lo`/`hi` narrow down to the index of the minimum. Loop while `lo < hi` (half-open — converge to a single index, don't overshoot).
- If `nums[mid] > nums[hi]`: minimum is strictly to the right → `lo = mid + 1`.
- Else (`nums[mid] <= nums[hi]`): minimum is at `mid` or further left → `hi = mid` (**not** `mid - 1` — `mid` itself could be the answer, and `hi = mid - 1` risks discarding it entirely, e.g. on `[1,2]` you'd jump `hi` to `-1` and go out of bounds).
- Return `nums[hi]` (== `nums[lo]`) at termination.

**Final code:**
```java
public int findMin(int[] nums) {
    int low = 0, high = nums.length - 1;
    while (low < high) {
        int mid = low + (high - low) / 2;
        if (nums[mid] > nums[high]) low = mid + 1;
        else high = mid;
    }
    return nums[high];
}
```

**TC:** O(log n). **SC:** O(1).

**Note:** `>=` instead of `>` also passes here, but only because LC 153 guarantees unique elements — `nums[mid] == nums[high]` is only reachable when `mid == high`, which never happens inside the loop (`low < high` guarantees `mid < high`). This equivalence breaks the moment duplicates are allowed.

---

## LC 154 — Find Minimum in Rotated Sorted Array II (bonus, duplicates, self-initiated — not LeetCode-verified this session)

**Delta from LC 153:** add a third branch for `nums[mid] == nums[hi]` → can't tell which side the minimum is on, so just shrink `hi--` (safe because you're only ever discarding *the copy at `hi`*, and the minimum is guaranteed to survive somewhere in `[lo, hi-1]` since it's equal to `nums[mid]`).

```java
public int findMinWithDuplicates(int[] nums) {
    int low = 0, high = nums.length - 1;
    while (low < high) {
        int mid = low + (high - low) / 2;
        if (nums[mid] > nums[high]) low = mid + 1;
        else if (nums[mid] < nums[high]) high = mid;
        else high--;
    }
    return nums[high];
}
```
**TC:** worst case O(n) (same degeneration reasoning as LC 81). **SC:** O(1).
⚠️ **Not yet verified on LeetCode** — do this before fully trusting it.

---

## LC 875 — Koko Eating Bananas

**Recognition:** first true **"binary search on the answer"** problem. You are *not* searching the `piles` array — you're searching the space of possible eating speeds `k`, from `1` to `max(piles)`.

**Why binary search applies — the monotonicity argument:** as `k` increases, hours-required is non-increasing (monotonic). If speed `k` is feasible (finishes within `h` hours), every speed `> k` is also feasible. That "if it works, everything bigger works too" shape is *the* trigger for binary-search-on-the-answer.

**Feasibility check:** for a candidate speed `k`, hours needed = `Σ ceil(pile / k)` over all piles. Feasible iff that sum `<= h`.

**Ceiling via integer arithmetic:** `ceil(pile / k) == (pile + k - 1) / k` — **parenthesize carefully**; `pile + k - 1 / k` is wrong due to Java operator precedence (division binds tighter than subtraction).

**Two valid designs for the boundary:**
- Track a separate `ans` variable: on feasible, `ans = mid` then shrink freely (`hi = mid - 1` is safe since the answer is already banked).
- No separate variable (converge `lo`/`hi` like LC 153): must use `hi = mid`, never `mid - 1`, since `mid` might be the final answer.

**Final code:**
```java
private long kokoFeasibility(int[] piles, int speed) {
    long total = 0;
    for (int pile : piles) total += (pile + speed - 1) / speed;
    return total;
}

public int minEatingSpeed(int[] piles, int h) {
    int low = 1, high = Arrays.stream(piles).max().getAsInt();   // ⚠️ low starts at 1, NOT 0
    int ans = -1;
    while (low <= high) {
        int mid = low + (high - low) / 2;
        long hours = kokoFeasibility(piles, mid);
        if (hours <= h) { ans = mid; high = mid - 1; }
        else low = mid + 1;
    }
    return ans;
}
```

**TC:** O(n log(max(piles))) — brute force (try every speed linearly) would be O(n · max(piles)), which at `n ≤ 10⁴`, `max(piles) ≤ 10⁹` is ~10¹³ operations, completely infeasible. Binary search collapses the outer factor to `log(max(piles))`. **SC:** O(1).

**Trap hit and self-corrected:** initializing `low = 0` lets `mid` land on `0`, and `(pile + 0 - 1) / 0` is a division by zero. Caught before submission — `low` must start at `1` (minimum meaningful eating speed).

**Overflow note:** `n × max(piles)` can reach ~10¹³, so the hours accumulator must be `long`, not `int`.

---

## LC 4 — Median of Two Sorted Arrays (Hard)

**Recognition:** two sorted arrays, need the combined median, target complexity explicitly `O(log(min(m,n)))`. This complexity signature — `min(m,n)`, not `m+n` — is the tell that you must binary search *only the smaller array*, and that you're searching a **partition point**, not a value.

**Core idea — partitioning, not searching:**
- Let `half = (m + n + 1) / 2` — the count of elements that belong in the combined "left half." This single formula (verified by hand: total=7 → `half=4`, odd case, median is the last element of the left half; total=8 → `half=4`, even case, median is the average of the last-left and first-right elements) works uniformly for odd and even totals without branching.
- If `x` elements come from `nums1`'s left portion, `nums2` must supply exactly `half - x` — no freedom once `x` is picked.
- **Ensure `m <= n`** (swap if not) before binary searching on `x` — this guarantees `half - x` never exceeds `n` (never asks `nums2` for more elements than it has).

**Four boundary values per partition `x`:**
- `l1` = last element of `nums1`'s left part (`nums1[x-1]`, or `-∞` if `x == 0`)
- `r1` = first element of `nums1`'s right part (`nums1[x]`, or `+∞` if `x == m`)
- `l2` = last element of `nums2`'s left part (`nums2[y-1]`, or `-∞` if `y == 0`, where `y = half - x`)
- `r2` = first element of `nums2`'s right part (`nums2[y]`, or `+∞` if `y == n`)

**Correct partition condition:** `l1 <= r2 && l2 <= r1` — everything in the combined left half must be ≤ everything in the combined right half.

**Adjustment on failure:**
- `l1 > r2` → too many elements taken from `nums1`'s left → shrink `x` → `hi = mid - 1`.
- else (`l2 > r1`) → too many elements taken from `nums2`'s left, i.e. not enough from `nums1` → grow `x` → `lo = mid + 1`.

**Final code:**
```java
public double findMedianSortedArrays(int[] nums1, int[] nums2) {
    int m = nums1.length, n = nums2.length;
    if (m > n) return findMedianSortedArrays(nums2, nums1);   // always binary search the smaller array

    int low = 0, high = m;
    int half = (m + n + 1) / 2;

    while (low <= high) {
        int x = low + (high - low) / 2;
        int y = half - x;

        int l1 = Integer.MIN_VALUE, r1 = Integer.MAX_VALUE;
        int l2 = Integer.MIN_VALUE, r2 = Integer.MAX_VALUE;

        if (x > 0) l1 = nums1[x - 1];
        if (x < m) r1 = nums1[x];
        if (y > 0) l2 = nums2[y - 1];
        if (y < n) r2 = nums2[y];

        if (l1 <= r2 && l2 <= r1) {
            if ((m + n) % 2 == 1) return Math.max(l1, l2);
            else return (Math.max(l1, l2) + Math.min(r1, r2)) / 2.0;
        } else if (l1 > r2) {
            high = x - 1;
        } else {
            low = x + 1;
        }
    }
    return 0.0; // unreachable if inputs are valid sorted arrays
}
```

**TC:** O(log(min(m,n))). **SC:** O(1) — four sentinel variables only, no auxiliary structures.

**Why `low <= high`, not `low < high`:** unlike LC 153 (where the loop converges toward a single surviving index), here every iteration either **returns immediately** on a valid partition or **explicitly** moves `low`/`high`. The candidate `x = low = high` is a legitimate partition that must still be checked — switching to `low < high` would exit one step early and could skip the correct partition.

**Traps hit and self-corrected:**
1. Initially swapped the even/odd return conditions (`(m+n)%2 == 0` returning the odd-case formula and vice versa) — caught via a hand-trace on `nums1=[1,3], nums2=[2]`.
2. Initially mislabeled a sentinel as `r1` instead of `r2` while explaining the empty-partition cases verbally — code was correct, explanation had the typo.

---

## Cross-Cutting Traps for This Section

1. **`(lo + hi) / 2` overflows for large ints.** Always `lo + (hi - lo) / 2`. Used correctly throughout this session.
2. **Mixing inclusive/exclusive `hi` conventions across problems causes infinite loops or off-by-one errors.** This session used two templates deliberately:
   - Closed `[lo, hi]`, `while (lo <= hi)` → LC 33, 81, 875, 4 (every iteration checks and either returns or explicitly narrows).
   - Half-open convergence, `while (lo < hi)`, `hi = mid` (never `mid - 1`) → LC 153 (narrowing to a single surviving index).
3. **`hi = mid` vs `hi = mid - 1` is not a fixed rule — it depends on whether `mid` is still a live candidate for the final answer.** If you've already banked the answer in a separate variable, `mid - 1` is safe. If you're relying on `lo == hi` convergence to *be* the answer, you must never discard `mid` with `mid - 1`.
4. **Rotated array "which half is sorted" check (`nums[lo] <= nums[mid]`) is only trustworthy under uniqueness.** With duplicates, add an explicit equality fallback (`lo++; hi--`).
5. **Compare against `hi`, not `lo`, when hunting for the rotated minimum** — `mid` can equal `lo` on small windows, making an `lo`-based comparison uninformative; `mid < hi` always holds so this never degenerates.
6. **Binary-search-on-the-answer needs a correct starting `lo`.** `lo = 0` in Koko caused a division-by-zero; the true minimum meaningful answer for the domain (`1`, not `0`) must be reasoned out, not defaulted.
7. **Watch operator precedence when writing ceiling-division inline:** `pile + k - 1 / k` ≠ `(pile + k - 1) / k` in Java.
8. **Overflow:** any accumulator that can reach `n × max(value)` scale (Koko's total hours) needs `long`.
9. **Simplify Big-O fully.** O(n/2) → state it as O(n), don't leave the constant in.


---

# Section 5 — Stack & Monotonic Stack — Revision Notes

**Status:** ✅ All 4 problems solved & verified on LeetCode (Java)

---

## Recognition Cues (say these out loud before coding)

- **Plain stack:** matching/nesting, deferred evaluation ("compute this later once precedence is known"), "undo the most recent thing," or auxiliary state that must track the *current* min/max alongside a normal stack.
- **Monotonic stack:** the phrase "next greater/smaller" or "previous greater/smaller" — explicit or disguised ("how many days until warmer" = next greater; "how far can this bar extend" = previous smaller *and* next smaller, both sides).
- **The core invariant:** the stack holds indices whose answer is still *unresolved*. The current element resolves the top of the stack when it breaks the monotonic property. Each index is pushed once and popped at most once → **O(n)** even though there's a `while` inside a `for`.

---

## 1. LC 155 — Min Stack

**Pattern:** Auxiliary stack tracking running state (not monotonic — a design problem).

**Idea:** Maintain a second `minStack` in lockstep with the main stack. On `push(x)`: push `min(x, minStack.peek())` (or just `x` if `minStack` is empty). On `pop()`: pop both stacks together, so the min-stack "reverts" to the correct historical minimum automatically.

**Why it works:** Every index in `minStack` stores "the minimum of everything at-or-below this point in the stack," so popping the main stack and popping `minStack` in lockstep keeps them perfectly synced — no stale values possible.

```java
class MinStack {
    Stack<Integer> stack = new Stack<>();
    Stack<Integer> minStack = new Stack<>();

    public void push(int value) {
        stack.push(value);
        minStack.push(minStack.isEmpty() ? value : Math.min(minStack.peek(), value));
    }
    public void pop() { stack.pop(); minStack.pop(); }
    public int top() { return stack.peek(); }
    public int getMin() { return minStack.peek(); }
}
```

**TC/SC:** All ops O(1) time. O(N) space (two stacks, so O(2N) → O(N)).

**Space-optimized variant (bonus, discussed not implemented):** Only push to `minStack` when the new value is `<=` current min. On `pop()`, only pop `minStack` if its top equals the value being popped from the main stack.

**Traps hit:** None major — this one clicked fast once "maintain a parallel min-tracking stack" was internalized.

---

## 2. LC 739 — Daily Temperatures

**Pattern:** The base monotonic stack template — **next greater element**.

**Idea:** Stack holds indices with unresolved "next warmer day." Scan left to right; while the current temperature is greater than `heights[stack.peek()]`, pop and resolve `answer[popped] = i - popped`. Push current index.

```java
public int dailyTemperatures(int[] temperatures) {
    int n = temperatures.length;
    Stack<Integer> stack = new Stack<>();
    int[] result = new int[n];

    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && temperatures[stack.peek()] < temperatures[i]) {
            int top = stack.pop();
            result[top] = i - top;
        }
        stack.push(i);
    }
    return result;
}
```

**Stack invariant:** Monotonically **decreasing** top-to-bottom (only unresolved — i.e., "nothing warmer seen yet" — indices remain).

**Why `while` not `if`:** One new temperature can resolve multiple unresolved days at once (e.g., a big jump after several cooler days).

**TC:** O(N) — each index pushed once, popped at most once, despite the nested loop appearance.
**SC:** O(N) worst case (strictly decreasing input never pops until the end).

**Traps hit:** None — clean first implementation. Correctly recognized `answer[]` defaults to 0, handling "no warmer day ever" for free.

---

## 3. LC 227 — Basic Calculator II

**Pattern:** Plain stack for **deferred-operand arithmetic** (not monotonic — new pattern for this section).

**Idea:** Track `num` (number being built), `pendingOp` (the operator seen *before* the current number — starts as `'+'`), and a stack of signed values.

**Rule — whenever a number finishes parsing (hit an operator, a space that isn't trailing, or end of string):**
- `pendingOp == '+'` → push `num`
- `pendingOp == '-'` → push `-num`
- `pendingOp == '*'` → push `stack.pop() * num`
- `pendingOp == '/'` → push `stack.pop() / num`
- then `pendingOp = currentChar`, reset `num = 0`

Final answer = sum of everything left on the stack.

```java
public int basicCalculatorII(String s) {
    int n = s.length();
    int num = 0;
    char pendingOp = '+';
    Stack<Integer> stack = new Stack<>();

    for (int i = 0; i < n; i++) {
        char c = s.charAt(i);
        if (Character.isDigit(c)) num = num * 10 + (c - '0');

        if ((!Character.isDigit(c) && !Character.isWhitespace(c)) || i == n - 1) {
            if (pendingOp == '+') stack.push(num);
            else if (pendingOp == '-') stack.push(-num);
            else if (pendingOp == '*') stack.push(stack.pop() * num);
            else if (pendingOp == '/') stack.push(stack.pop() / num);
            pendingOp = c;
            num = 0;
        }
    }

    int result = 0;
    while (!stack.isEmpty()) result += stack.pop();
    return result;
}
```

**Why `*`/`/` apply immediately but `+`/`-` defer:** Precedence — multiplication/division must combine with the value already on top of the stack right away; addition/subtraction can safely wait since summing is commutative/associative regardless of order.

**TC:** O(N), single pass. **SC:** O(N) worst case (all `+`/`-` expression → stack holds ~N/2 values).

**Traps hit (real bug, not hypothetical):**
- Original code used `continue` to skip whitespace, which meant a **trailing space** (e.g. `" 3/2 "`) skipped past the final "flush the last number" block entirely, silently truncating the last operation.
- **Fix:** removed the early `continue`; folded the whitespace check directly into the trigger condition: `(!isDigit && !isWhitespace) || i == n-1`. This guarantees the final flush always fires on the true last character, regardless of what that character is.
- **Lesson:** any time a loop has an early `continue`/`break` for "skip this char," double check it can't accidentally skip past a boundary-condition check that needs to fire on the *last* iteration.

---

## 4. LC 84 — Largest Rectangle in Histogram

**Pattern:** Monotonic stack finding **previous smaller** and **next smaller** elements simultaneously — the hardest problem in the section, and the unlock for LC 85.

### Key reframe (this is the whole problem)
For bar `i`, if the rectangle's height is fixed at `heights[i]`, the rectangle can extend left/right only as far as bars that are **at least** `heights[i]` tall. So you need, for every `i`:
- `left[i]` = nearest index to the left with a **smaller** height (`-1` if none)
- `right[i]` = nearest index to the right with a **smaller** height (`n` if none)
- `width = right[i] - left[i] - 1`
- `area = heights[i] * width`

**Why sentinels `-1` and `n` (not `0` and `n-1`):** They make the subtraction formula naturally *include* the current bar when no smaller boundary exists. E.g. `left = -1, right = 5` → `width = 5 - (-1) - 1 = 5`, correctly counting indices `0..4`. Using `left = 0` instead would undercount by 1.

**Not the same as Trapping Rain Water** — TRW bounds water by the *tallest* walls (needs both sides ≥ current); this problem bounds a rectangle by the *shortest* neighbors (breaks on anything smaller). Easy to conflate since both are "look left and right" problems — they use opposite comparisons.

### Two-Pass Solution (find PSE and NSE separately)

```java
private int[] findPreviousSmallerElement(int[] nums, int n) {
    int[] pse = new int[n];
    Arrays.fill(pse, -1);
    Stack<Integer> stack = new Stack<>();
    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && nums[stack.peek()] >= nums[i]) stack.pop();
        if (!stack.isEmpty()) pse[i] = stack.peek();
        stack.push(i);
    }
    return pse;
}

private int[] findNextSmallerElement(int[] nums, int n) {
    int[] nse = new int[n];
    Arrays.fill(nse, n);
    Stack<Integer> stack = new Stack<>();
    for (int i = n - 1; i >= 0; i--) {
        while (!stack.isEmpty() && nums[stack.peek()] >= nums[i]) stack.pop();
        if (!stack.isEmpty()) nse[i] = stack.peek();
        stack.push(i);
    }
    return nse;
}

public int largestRectangleHistogram2Pass(int[] heights) {
    int n = heights.length;
    int[] nse = findNextSmallerElement(heights, n);
    int[] pse = findPreviousSmallerElement(heights, n);
    int area = 0;
    for (int i = 0; i < n; i++) {
        area = Math.max(area, heights[i] * (nse[i] - pse[i] - 1));
    }
    return area;
}
```

**TC:** O(N) (O(3N): two stack passes + one combining pass). **SC:** O(N) (two extra arrays + stack).

### Single-Pass Solution (the interview-preferred version)

**Core insight:** the *act of popping* resolves both boundaries at once. When index `i` causes a pop of `topIdx`:
- `right = i` (the element that caused the pop)
- `left` = whatever's now exposed on top of the stack after the pop (`-1` if empty) — because that element is still there precisely *because* it's smaller than `heights[topIdx]`, i.e., it already **is** the previous smaller element.

Elements that never get popped during the main scan (e.g. strictly increasing heights) need a cleanup pass at the end, where `right = n` for all of them (nothing smaller exists anywhere to their right).

```java
public int largestRectangleHistogram(int[] heights) {
    int n = heights.length;
    int maxArea = 0;
    Stack<Integer> stack = new Stack<>();

    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && heights[stack.peek()] >= heights[i]) {
            int topIdx = stack.pop();
            int left = stack.isEmpty() ? -1 : stack.peek();
            int right = i;
            maxArea = Math.max(maxArea, heights[topIdx] * (right - left - 1));
        }
        stack.push(i);
    }

    // Flush remaining indices — nothing smaller exists to their right in the array.
    while (!stack.isEmpty()) {
        int topIdx = stack.pop();
        int left = stack.isEmpty() ? -1 : stack.peek();
        int right = n;
        maxArea = Math.max(maxArea, heights[topIdx] * (right - left - 1));
    }

    return maxArea;
}
```

*(Equivalent alternative not implemented: fold the cleanup into the main loop by iterating `i` from `0` to `n` inclusive, treating `i == n` as a sentinel height of `0`, which guarantees every remaining stack element gets popped naturally.)*

**TC:** O(N) — each index pushed once, popped once, across both loops combined.
**SC:** O(N) — single stack, worst case holds all N indices (strictly increasing input).

**Traps hit:**
- Initially conflated this problem with Trapping Rain Water (two-pointer, tallest-wall bounding) before correctly distinguishing that this problem bounds by *shortest* neighbor, not tallest.
- First attempt at the width formula was off by one (`right - left` instead of `right - left - 1`) — caught and corrected by manually listing which indices are actually included in the rectangle for a concrete example.
- Needed the sentinel reasoning (`-1` / `n`) walked through explicitly with the subtraction formula before it "clicked" — good sign this deserves a cold re-solve in a few days to confirm it's internalized, not just recently explained.

---

## Section 5 Summary — Pattern Recognition Table

| Problem | Trigger phrase | Stack holds | Monotonic direction |
|---|---|---|---|
| Min Stack | "O(1) getMin" | Running minimum per push | N/A (parallel state, not monotonic) |
| Daily Temperatures | "days until warmer" | Indices with unresolved next-greater | Decreasing top→bottom |
| Basic Calculator II | Expression with `+ - * /` precedence | Signed operands to sum | N/A (deferred evaluation, not monotonic) |
| Largest Rectangle | "how far can this bar extend" | Indices with unresolved prev/next-smaller | Increasing top→bottom |

**Recurring bug pattern this section:** boundary/off-by-one errors — trailing whitespace skipping the final flush in Calculator II, and the `right - left - 1` width formula needing to be derived from a concrete included-indices list rather than guessed. Consistent with the `<` vs `<=` boundary risk already flagged as a recurring theme across sections.


---

# Section 6 — Linked List — Revision Notes

**Status:** ✅ Complete (5/5 verified on LeetCode)
**Problems:** LC 206, LC 142, LC 23, LC 25, LC 146

---

## LC 206 — Reverse Linked List

### Recognition cue
"Reverse the list" — no k-grouping, no cycle, just full reversal. Three-pointer iterative is the default; recursive variant also expected.

### Iterative — O(N) time, O(1) space

```java
public ListNode reverseList(ListNode head) {
    if (head == null) return null;          // guard: curr.next on null head → NPE

    ListNode prev = null;
    ListNode curr = head;

    while (curr != null) {
        ListNode next = curr.next;          // save BEFORE rewiring
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}
```

### Recursive — O(N) time, O(N) space (call stack)

```java
public ListNode reverseListRecursion(ListNode head) {
    if (head == null || head.next == null) return head;

    ListNode reversedHead = reverseListRecursion(head.next);
    head.next.next = head;
    head.next = null;

    return reversedHead;
}
```

### Traps
- **Save `next` before rewiring `curr.next`.** Rewire-first loses your only path forward.
- **Null-check `head` first** — `curr.next` on a null head throws NPE.
- Recursive `reversedHead` is assigned exactly once (in the base case) and passed through unchanged on every `return` during unwinding — that's *why* it "stays" at the new head. Not magic, just a pass-through value.

---

## LC 142 — Linked List Cycle II

### Recognition cue
"Find where the cycle begins" (not just "does a cycle exist"). Return type is `ListNode`, not `int`/`boolean`. O(1) space is the explicit ask — that rules out the `HashSet` approach as the final answer.

### Floyd's Cycle Detection — O(N) time, O(1) space

```java
public ListNode detectCycle(ListNode head) {
    ListNode slow = head, fast = head;

    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) {
            ListNode x = head;
            ListNode y = slow;
            while (x != y) {
                x = x.next;
                y = y.next;
            }
            return x;                       // cycle entry point
        }
    }
    return null;                            // no cycle
}
```

### The derivation (re-derive this cold — it's the actual skill, not the code)
Let `k` = distance from `head` to cycle entry `E`, `m` = distance from `E` to meeting point `M`, `C` = cycle length.

- `slow` travels `k + m`. `fast` travels `2(k + m)`, and also `k + m + nC` (n full extra laps).
- `2(k+m) = k + m + nC` → **`k = nC − m`**.
- So walking `k` steps from `M` lands at `m + k = m + nC − m = nC` → a whole number of laps from `E` → back at `E`.
- Walking `k` steps from `head` also lands at `E` (that's the definition of `k`).
- Same distance (`k`), same speed (1 step/iteration), different start points → **arrive at `E` simultaneously.**

### Traps
- Loop guard must be `fast != null && fast.next != null` — `fast != null` alone crashes on `fast.next.next`.
- Brute force (`HashSet<ListNode>`, check-before-add) is O(N)/O(N) — correct but doesn't meet the O(1) space bar.
- The "reset to head" step is **not** intuition to memorize — it's a direct consequence of the algebra above. If you can't re-derive `k = nC - m` cold, you don't actually own this yet.

---

## LC 23 — Merge k Sorted Lists

### Recognition cue
"k sorted lists/arrays, merge into one" → min-heap of size k, or divide-and-conquer pairwise merging. Both hit O(N log K).

### Approach A — Min-Heap — O(N log K) time, O(K) space

```java
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> pq = new PriorityQueue<>((a, b) -> Integer.compare(a.val, b.val));

    for (ListNode node : lists) {
        if (node != null) pq.offer(node);
    }

    ListNode dummy = new ListNode(-1);
    ListNode curr = dummy;

    while (!pq.isEmpty()) {
        ListNode smallest = pq.poll();
        curr.next = smallest;
        curr = curr.next;
        if (smallest.next != null) pq.offer(smallest.next);
    }
    return dummy.next;
}
```

### Approach B — Divide & Conquer — O(N log K) time, O(log K) space (recursion stack)

```java
public ListNode mergeKListsDC(ListNode[] lists) {
    if (lists == null || lists.length == 0) return null;
    return merge(lists, 0, lists.length - 1);
}

private ListNode merge(ListNode[] lists, int left, int right) {
    if (left > right) return null;
    if (left == right) return lists[left];

    int mid = left + (right - left) / 2;
    ListNode l1 = merge(lists, left, mid);
    ListNode l2 = merge(lists, mid + 1, right);
    return mergeTwoLists(l1, l2);
}

private ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(-1);
    ListNode curr = dummy;

    while (l1 != null && l2 != null) {
        if (l1.val <= l2.val) { curr.next = l1; l1 = l1.next; }
        else                  { curr.next = l2; l2 = l2.next; }
        curr = curr.next;
    }
    curr.next = (l1 != null) ? l1 : l2;
    return dummy.next;
}
```

### Traps
- **`a.val - b.val` in a comparator overflows** on extreme int values. Use `Integer.compare(a.val, b.val)`.
- Brute force "collect all nodes, sort, rebuild": O(N log N) time, O(N) space — works, but doesn't use the sortedness of individual lists.
- Sequential merging (merge list 1+2, then +3, then +4...) is **O(NK)**, worse than heap/D&C when K is large — the growing combined list gets re-touched on every merge.
- Heap space is O(K); D&C recursion space is O(log K) — a real, mentionable tradeoff even though both are O(N log K) time.

---

## LC 25 — Reverse Nodes in k-Group

### Recognition cue
"Reverse every k nodes" + explicit rule for leftover nodes (< k) staying unreversed. Combines LC 206's reversal mechanic with group-boundary bookkeeping.

### O(N) time, O(1) space

```java
public ListNode reverseKGroup(ListNode head, int k) {
    if (head == null || k == 1) return head;

    ListNode dummy = new ListNode(-1);
    dummy.next = head;
    ListNode prevTail = dummy;

    while (true) {
        ListNode groupStart = prevTail.next;
        ListNode nextGroupStart = groupStart;

        for (int i = 0; i < k; i++) {
            if (nextGroupStart == null) return dummy.next;   // fewer than k left — stop
            nextGroupStart = nextGroupStart.next;
        }

        ListNode prev = null;
        ListNode curr = groupStart;

        for (int i = 0; i < k; i++) {
            ListNode next = curr.next;
            curr.next = prev;
            prev = curr;
            curr = next;
        }

        prevTail.next = prev;              // new head of reversed group
        groupStart.next = nextGroupStart;   // old head (now tail) links to next group
        prevTail = groupStart;              // old head is new prevTail for next iteration
    }
}
```

### Traps
- **Precondition check before reversing:** walk `k` nodes ahead first; if you hit `null` before counting `k`, that group stays untouched. This must happen *before* any pointer rewiring.
- **Save `nextGroupStart` before reversing** — once you rewire inside the group, you lose the only link to the rest of the list.
- `groupStart` (the original head of the group) ends up as the **tail** after reversal — that's your anchor point for `groupStart.next = nextGroupStart` and for becoming the next `prevTail`.
- Inside the k-bounded reversal loop, a null-guard on `curr.next` (copied from LC 206 habit) is **dead code** — the loop is bounded by `i < k`, not by `curr != null`, so `next`/`curr` are never read past the k-th iteration. Don't carry over guards from a different loop-termination style without checking if they're still needed.

---

## LC 146 — LRU Cache (Design)

### Recognition cue
"O(1) get and put" + "evict least recently used" → HashMap (O(1) key lookup) + Doubly Linked List (O(1) reorder/evict), map stores `key → Node` (not `key → value`) so you can jump straight to a node without scanning.

### O(1) time (get & put), O(capacity) space

```java
class LRUCache {
    class Node {
        int key, val;
        Node next, prev;
        Node(int key, int val) { this.key = key; this.val = val; }
    }

    private final HashMap<Integer, Node> map;
    private final Node head, tail;       // sentinels — never hold real data
    private final int capacity;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>();
        head = new Node(-1, -1);
        tail = new Node(-1, -1);
        head.next = tail;
        tail.prev = head;
    }

    public int get(int key) {
        if (!map.containsKey(key)) return -1;
        Node node = map.get(key);
        removeNode(node);
        addToFront(node);
        return node.val;
    }

    public void put(int key, int value) {
        if (map.containsKey(key)) {
            Node node = map.get(key);
            node.val = value;
            removeNode(node);
            addToFront(node);
            return;
        }

        if (map.size() == capacity) {
            map.remove(tail.prev.key);      // read key BEFORE unlinking
            removeNode(tail.prev);
        }

        Node newNode = new Node(key, value);
        map.put(key, newNode);
        addToFront(newNode);
    }

    private void removeNode(Node node) {    // LIST-ONLY. Never touches map.
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void addToFront(Node node) {
        node.next = head.next;
        node.prev = head;
        node.next.prev = node;
        head.next = node;
    }
}
```

### Traps — this is the highest bug-density problem in the section
- **Sentinel `head`/`tail` must be linked to each other in the constructor** (`head.next = tail; tail.prev = head;`). Forgetting this means `head.next` is `null` on first use → NPE in `addToFront`.
- **`removeNode` must do exactly one job: unlink from the list.** It must never touch the `HashMap`. The bug that appeared here: `removeNode` originally called `map.remove(...)`, which meant calling it from `get()` (to reposition a node) silently evicted that key from the map — the *first* `get(key)` would still return the right value (read before eviction), but a *second* `get(key)` would wrongly return `-1`. This is dangerous specifically because casual single-call testing won't catch it.
- **Eviction order matters:** `map.remove(tail.prev.key)` must run *before* `removeNode(tail.prev)`. If you unlink first, `tail.prev` now points to a *different* node, and you'd evict the wrong key from the map — list and map would silently disagree about what was removed. Same family of bug as "save `next` before rewiring" in LC 206, just relocated.
- Map is keyed `key → Node`, never `key → value` — you need the node reference for O(1) list surgery, not just the stored value.
- `removeNode(Object)` bug variant to watch for: calling `map.remove(node.val)` instead of `node.key` — easy typo since both are `int` fields on the same object and the compiler won't catch it.

---

## Section 6 — Pattern Summary

| Ask | Tool | Problems |
|---|---|---|
| Reverse (full or grouped) | Three-pointer iterative reversal | 206, 25 |
| Cycle / entry point | Fast & slow pointers (Floyd's) | 142 |
| Merge k sorted structures | Min-heap **or** divide & conquer | 23 |
| O(1) get + put with eviction | HashMap + Doubly Linked List | 146 |

**Recurring meta-trap across the section:** save the pointer/key you need to read **before** you perform a rewiring or unlinking operation that would make it unreachable or stale. This showed up in LC 206 (`next = curr.next`), LC 25 (`nextGroupStart` saved pre-reversal), and LC 146 (`tail.prev.key` read before `removeNode`). Worth internalizing as a single named instinct rather than three separate lessons.


---

# Section 7 — Trees & BST — Revision Notes

**Status:** ✅ Complete — 8/8 problems verified on LeetCode (Java derivation)

---

## Recognition Table — The Four Tree Shapes

| Shape | Use when | Signature | Problems |
|---|---|---|---|
| **DFS, return a value up** | Answer depends on subtree results | `int/TreeNode dfs(node)` returning info to parent | 236, 105, 124 |
| **DFS with a global / persistent state** | Answer can be "centred" at any node — need one value returned, another recorded | `dfs` returns one thing, updates external state with another | 124 |
| **BFS by level** | "level", "row", "closest", "right side view" | Queue + `int size = queue.size()` snapshot per level | 102, 199, 297 |
| **In-order on a BST** | "k-th smallest", "validate", "sorted" | In-order traversal of a BST is sorted — that *is* the property | 98, 230 |

---

## LC 102 — Binary Tree Level Order Traversal

**Pattern:** BFS with frozen level-size snapshot

```java
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;

    Queue<TreeNode> que = new LinkedList<>();
    que.offer(root);

    while (!que.isEmpty()) {
        int size = que.size();               // ⭐ frozen BEFORE inner loop
        List<Integer> level = new ArrayList<>();

        for (int i = 0; i < size; i++) {
            TreeNode node = que.poll();
            level.add(node.val);
            if (node.left  != null) que.offer(node.left);
            if (node.right != null) que.offer(node.right);
        }
        result.add(level);
    }
    return result;
}
```

**TC:** O(N) · **SC:** O(N) (queue + output)

**Trap:** `int size = que.size()` must be captured **once**, outside the inner loop. If you instead re-check `que.size()` live inside the loop condition (e.g. `for (int i = 0; i < que.size(); i++)`), newly-enqueued next-level nodes inflate the live size mid-loop and levels bleed into each other.

---

## LC 199 — Binary Tree Right Side View

**Pattern:** BFS-last-in-level, or DFS-right-first with depth tracking (two valid approaches — know both)

```java
// Approach 1 — BFS: same skeleton as LC 102, record only the last node per level
public List<Integer> rightSideView(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result;

    Queue<TreeNode> que = new LinkedList<>();
    que.offer(root);

    while (!que.isEmpty()) {
        int size = que.size();
        for (int i = 0; i < size; i++) {
            TreeNode node = que.poll();
            if (i == size - 1) result.add(node.val);   // rightmost of this level
            if (node.left  != null) que.offer(node.left);
            if (node.right != null) que.offer(node.right);
        }
    }
    return result;
}
```

```java
// Approach 2 — DFS, right child before left. First arrival at a depth = rightmost node.
public List<Integer> rightSideView(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    helper(root, 0, result);
    return result;
}

private void helper(TreeNode root, int depth, List<Integer> result) {
    if (root == null) return;
    if (result.size() == depth) result.add(root.val);  // first time at this depth
    helper(root.right, depth + 1, result);
    helper(root.left,  depth + 1, result);
}
```

**TC:** O(N) both approaches · **SC:** O(N)

**Trap / proof:** Right-first DFS recursion structurally exhausts the *entire* right subtree before ever backing out to a left branch at the same depth — so `result.size() == depth` is a valid "first arrival = rightmost" signal. This only holds because standard recursive DFS cannot re-enter a shallower-then-already-passed depth from a different branch out of order.

---

## LC 98 — Validate Binary Search Tree

**Pattern:** Inherited (min, max) bounds — NOT parent-only comparison

```java
public boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}

private boolean validate(TreeNode root, long lowerBound, long upperBound) {
    if (root == null) return true;
    if (root.val <= lowerBound || root.val >= upperBound) return false;

    boolean left  = validate(root.left,  lowerBound, root.val);
    boolean right = validate(root.right, root.val,   upperBound);
    return left && right;
}
```

**TC:** O(N) · **SC:** O(H) → O(log N) balanced, O(N) skewed

**Trap:** Comparing each node only to its immediate parent is wrong — a right-child-of-left-child can still violate a constraint set two levels up. Use `long` bounds (not `int`) since node values can span the full `int` range, and `Integer.MIN_VALUE`/`MAX_VALUE` as sentinels would risk collision with an actual node value.

**Alternative (not implemented):** in-order traversal + track only the *previous* value seen (O(1) extra space beyond recursion) — if any current value ≤ previous, invalid.

---

## LC 230 — Kth Smallest Element in a BST

**Pattern:** In-order traversal = sorted; early-exit once k-th element found

```java
// Iterative — cleaner early-exit than recursion
public int kthSmallest(TreeNode root, int k) {
    Stack<TreeNode> stack = new Stack<>();
    TreeNode curr = root;
    int count = 0;

    while (true) {
        if (curr != null) {
            stack.push(curr);
            curr = curr.left;
        } else {
            if (stack.isEmpty()) return -1;   // k invalid — shouldn't happen per constraints
            TreeNode node = stack.pop();
            count++;
            if (count == k) return node.val;
            curr = node.right;
        }
    }
}
```

**TC:** O(H + k) best case, O(N) worst case · **SC:** O(H) → O(log N) balanced, O(N) skewed

**Trap:** A recursive version needs an explicit guard (`if (answer != -1) return;`) at the **top** of every call to actually short-circuit — a bare `return` inside one branch only unwinds that one call, not the sibling calls higher up the stack. The iterative stack-based version avoids this tangle entirely — once you find the answer, just `return` from the loop.

---

## LC 236 — Lowest Common Ancestor of a Binary Tree

**Pattern:** Post-order — "found in both subtrees ⇒ current node is LCA"

```java
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == p || root == q || root == null) return root;

    TreeNode left  = lowestCommonAncestor(root.left,  p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);

    if (left != null && right != null) return root;   // found on both sides → this is LCA
    return (left != null) ? left : right;              // bubble up whichever side found something
}
```

**TC:** O(N) · **SC:** O(H) → O(log N) balanced, O(N) skewed

**Trap / key insight:** A node can be **its own ancestor**. If `root == p`, you return `root` immediately without needing to also confirm `q` is somewhere below it — that's still a valid LCA candidate per the problem's definition. Missing this makes you overcomplicate the base case.

---

## LC 105 — Construct Binary Tree from Preorder and Inorder Traversal

**Pattern:** Hashmap index lookup + index-range recursion (no array copying)

```java
public TreeNode buildTree(int[] preorder, int[] inorder) {
    Map<Integer, Integer> map = new HashMap<>();
    for (int i = 0; i < inorder.length; i++) map.put(inorder[i], i);

    return buildTreeHelper(preorder, 0, preorder.length - 1,
                            inorder, 0, inorder.length - 1, map);
}

private TreeNode buildTreeHelper(int[] preorder, int preStart, int preEnd,
                                  int[] inorder, int inStart, int inEnd,
                                  Map<Integer, Integer> map) {
    if (preStart > preEnd || inStart > inEnd) return null;

    TreeNode root = new TreeNode(preorder[preStart]);
    int inRoot = map.get(root.val);
    int numsLeft = inRoot - inStart;

    root.left  = buildTreeHelper(preorder, preStart + 1, preStart + numsLeft,
                                  inorder, inStart, inRoot - 1, map);
    root.right = buildTreeHelper(preorder, preStart + numsLeft + 1, preEnd,
                                  inorder, inRoot + 1, inEnd, map);
    return root;
}
```

**TC:** O(N) · **SC:** O(N) (hashmap) + O(H) recursion → overall O(N)

**Trap:** Rebuilding subarrays via `Arrays.copyOfRange()` at each call costs O(N) *per call*, totaling O(N log N) balanced / O(N²) skewed. Passing index bounds into the **same original arrays** avoids all copying → true O(N). The hashmap (value → inorder index) turns each root-lookup into O(1) instead of an O(N) linear scan.

---

## LC 124 — Binary Tree Maximum Path Sum

**Pattern:** "Return one thing, record another" — the payoff trick of the section

```java
public int maxPathSum(TreeNode root) {
    int[] maxSum = new int[1];
    maxSum[0] = Integer.MIN_VALUE;
    maxPathHelper(root, maxSum);
    return maxSum[0];
}

private int maxPathHelper(TreeNode root, int[] maxSum) {
    if (root == null) return 0;

    int leftGain  = Math.max(maxPathHelper(root.left,  maxSum), 0);  // clamp negatives to 0
    int rightGain = Math.max(maxPathHelper(root.right, maxSum), 0);

    maxSum[0] = Math.max(maxSum[0], leftGain + rightGain + root.val);  // RECORD: bent path

    return root.val + Math.max(leftGain, rightGain);                  // RETURN: straight path
}
```

**TC:** O(N) · **SC:** O(H) → O(log N) balanced, O(N) skewed

**Trap / key insight:** A node with a parent-edge cannot *also* use both its children in the path it returns upward — that would require 3 edges meeting at one node, which isn't a valid (non-branching) path. So:
- **Returned value** (to parent): straight extension through **one** child only — `val + max(leftGain, rightGain)`.
- **Recorded value** (global candidate): the "bent" path using **both** children — `val + leftGain + rightGain`. This can only be a *final* answer, never extended further.
- Negative subtree gains are clamped to 0 via `Math.max(gain, 0)` — a negative contribution should simply be excluded, not subtracted.

---

## LC 297 — Serialize and Deserialize Binary Tree

**Pattern:** BFS (level-order) with explicit null markers

```java
public String serialize(TreeNode root) {
    if (root == null) return "";

    StringBuilder sb = new StringBuilder();
    Queue<TreeNode> que = new LinkedList<>();
    que.offer(root);

    while (!que.isEmpty()) {
        TreeNode node = que.poll();
        if (node == null) {
            sb.append("null,");
        } else {
            sb.append(node.val).append(",");
            que.offer(node.left);
            que.offer(node.right);
        }
    }
    return sb.toString();
}

public TreeNode deserialize(String data) {
    if (data.equals("")) return null;

    String[] values = data.split(",");
    TreeNode root = new TreeNode(Integer.parseInt(values[0]));
    Queue<TreeNode> que = new LinkedList<>();
    que.offer(root);

    int i = 1;
    while (!que.isEmpty() && i < values.length) {
        TreeNode node = que.poll();

        if (!values[i].equals("null")) {
            node.left = new TreeNode(Integer.parseInt(values[i]));
            que.offer(node.left);
        }
        i++;

        if (!values[i].equals("null")) {
            node.right = new TreeNode(Integer.parseInt(values[i]));
            que.offer(node.right);
        }
        i++;
    }
    return root;
}
```

**TC:** O(N) both directions · **SC:** O(N) both directions

**Trap / key insight:** A traversal **without** null markers is structurally ambiguous — two different-shaped trees can produce the identical flat value sequence (e.g. `root=1,left=2,right=3` vs. `root=1,left=2,2.right=3`, both would serialize to `"1,2,3"` without markers). Null markers remove the ambiguity by explicitly recording "no child here" at every step.

**Why preorder/level-order works but in-order doesn't (even with null markers):** preorder and level-order both place the **root at a deterministic, known position** (first token), so reconstruction can proceed sequentially from a known starting point. In-order buries the root in the *middle* of the sequence — even with null markers present, you have no way to know where to start reconstructing, since "middle" isn't a fixed position without already knowing the subtree sizes.

---

## Cross-Cutting Patterns Reinforced This Section

- **Frozen loop bound vs. live re-check** (LC 102): capturing `size` once before an inner loop vs. re-evaluating it live — the same class of bug as "save the pointer before an operation makes it stale," seen earlier in LC 206/25/146.
- **A node can be its own ancestor** (LC 236) — don't over-constrain the base case.
- **Inherited constraints ≠ local constraints** (LC 98): a node's validity depends on the full root-to-node path, not just its immediate parent.
- **Index-range recursion avoids copying** (LC 105) — same idea as passing `(lo, hi)` bounds instead of slicing arrays, seen conceptually in binary search.
- **Return vs. record duality** (LC 124) — the single most reusable "hard" tree trick; expect to see it again in Diameter of Tree, Longest Univalue Path, House Robber III if they come up later.
- **Traversal order determines reconstructability** (LC 105, LC 297) — root position within the traversal sequence (start vs. middle) is what makes a scheme reversible or not.

---
