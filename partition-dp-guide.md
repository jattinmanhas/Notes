# Partition DP Guide

Partition DP is a dynamic programming pattern used when a problem asks you to split an array, string, expression, or interval into parts and optimize the result.

The key question is:

> If I make a cut somewhere, what is the best answer for the previous part plus the current segment?

Common signals:

- Split into parts
- Partition into groups
- Divide an array or string
- Cut at positions
- Parenthesize an expression
- Minimize or maximize cost after splitting
- Use exactly `k` parts
- Each segment contributes some cost

If the problem says "choose where to cut," think partition DP.

## The Three Families at a Glance

| Family | State | Transition | Complexity | Combine |
|---|---|---|---|---|
| Prefix | `dp[i]` (1D) | try every last cut `j < i` | **O(n²)** | `+` on `dp[j] + cost` |
| Interval | `dp[i][j]` (2D) | try every split `k` in `[i, j]` | **O(n³)** | `+` on two subintervals |
| K-partition | `dp[g][i]` (2D) | try every last-group start `j` | **O(k · n²)** | `+` **or** `max` (problem-dependent) |

All complexities assume O(1) segment cost (via precomputation). If computing a segment costs O(n) inline, multiply accordingly. **Knowing these signatures tells you when a pattern will TLE** — an O(n³) interval solution on `n = 2000` is dead on arrival.

## How to Derive Any of Them

Most people can't see the tabulation recurrence on a fresh problem. The reliable move is: **write the top-down recursion first** (the "what is the last cut?" question maps directly to a recursive call), confirm it's correct, then flip it to bottom-up. Each family below gives both.

---

## 1. Prefix Partition DP

Use this when you split the first `i` elements or characters into valid groups.

### State

```java
dp[i] = best answer for the prefix ending before index i   // covers 0 .. i-1
```

### Top-down (derive this first)

```java
Integer[] memo;   // size n+1, null = uncomputed

int solve(int i) {
    if (i == 0) return baseCase;            // often 0, or -1 for palindrome cuts
    if (memo[i] != null) return memo[i];

    int best = Integer.MAX_VALUE;           // or MIN_VALUE for maximization
    for (int j = 0; j < i; j++) {           // j..i-1 is the LAST segment
        if (isValidSegment(j, i - 1)) {
            best = Math.min(best, solve(j) + cost(j, i - 1));
        }
    }
    return memo[i] = best;
}
```

### Bottom-up

```java
int[] dp = new int[n + 1];
Arrays.fill(dp, Integer.MAX_VALUE);         // or MIN_VALUE for maximization
dp[0] = 0;                                  // base case, problem-dependent

for (int i = 1; i <= n; i++) {
    for (int j = 0; j < i; j++) {
        // guard against MAX_VALUE + cost overflow before adding
        if (isValidSegment(j, i - 1) && dp[j] != Integer.MAX_VALUE) {
            dp[i] = Math.min(dp[i], dp[j] + cost(j, i - 1));
        }
    }
}
```

### Meaning

The last partition is `j .. i-1`. Everything before `j` is already solved by `dp[j]`.

### Common Problems

- Word Break
- Perfect Squares
- Palindrome Partitioning II
- Partition Array for Maximum Sum

---

## 2. Interval Partition DP

Use this when the problem is about solving a range `[i..j]`, and a split point divides the interval into two smaller intervals.

### State

```java
dp[i][j] = best answer for interval i .. j
```

### ⚠️ Two different interval forms — do not mix them up

This is the single biggest source of bugs in interval DP. There are **two distinct mental models**, and they use different index arithmetic:

**Form A — Partition the elements** (`k` and `k+1`)
The split point `k` belongs to the *left* group. The two subproblems perfectly tile the interval with no overlap: `[i..k]` and `[k+1..j]`.
→ Matrix Chain Multiplication, Strange Printer, Boolean Parenthesization, Palindrome Partitioning (interval variant).

**Form B — Pivot around `k`** (`k` and `k`, shared boundary)
Here `k` is the element you fix / burst / remove **last**. The two subintervals sit on either side of it and *share* `k` as a boundary: `(i..k)` and `(k..j)`, an open interval. This is why you pad the array with sentinels (1s for Burst Balloons, 0 and n for Cut a Stick) — the boundaries `i` and `j` are walls that are never consumed.
→ Burst Balloons, Minimum Cost to Cut a Stick.

The tell: if the recurrence reads `dp[i][k] + dp[k+1][j]`, you're in Form A. If it reads `dp[i][k] + dp[k][j]`, you're in Form B and you almost certainly needed sentinels.

### Top-down (Form A — partition the elements)

```java
int solve(int i, int j) {
    if (i >= j) return 0;                   // single element / empty range
    if (memo[i][j] != null) return memo[i][j];

    int best = Integer.MAX_VALUE;
    for (int k = i; k < j; k++) {           // k joins the LEFT half
        best = Math.min(best,
                solve(i, k) + solve(k + 1, j) + cost(i, k, j));
    }
    return memo[i][j] = best;
}
```

### Bottom-up (Form A)

```java
int[][] dp = new int[n][n];

for (int len = 2; len <= n; len++) {
    for (int i = 0; i + len - 1 < n; i++) {
        int j = i + len - 1;
        dp[i][j] = Integer.MAX_VALUE;

        for (int k = i; k < j; k++) {
            int cost = dp[i][k] + dp[k + 1][j] + combineCost(i, k, j);
            dp[i][j] = Math.min(dp[i][j], cost);
        }
    }
}
```

### Bottom-up (Form B — pivot around k, open interval)

```java
// array is padded with sentinels; the interval (i, j) is OPEN
int[][] dp = new int[n][n];

for (int len = 2; len < n; len++) {         // len = gap between the two walls
    for (int i = 0; i + len < n; i++) {
        int j = i + len;

        for (int k = i + 1; k < j; k++) {   // k is fixed/removed LAST
            int val = dp[i][k] + dp[k][j] + pivotCost(i, k, j);
            dp[i][j] = Math.max(dp[i][j], val);   // (max for Burst Balloons)
        }
    }
}
```

### Common Problems

- Matrix Chain Multiplication (Form A)
- Burst Balloons (Form B)
- Minimum Cost to Cut a Stick (Form B)
- Boolean Parenthesization (Form A, **needs extra state** — see below)
- Strange Printer (Form A)

---

## 3. K-Partition DP

Use this when you must split something into exactly `k` groups.

### State

```java
dp[g][i] = best answer for first i elements using exactly g groups
```

### ⚠️ The combine operator is problem-dependent

This is the second big trap. The template below shows an **additive** combine, which is correct for:

- Minimum Difficulty of a Job Schedule (sum of daily maxes)
- Palindrome Partitioning III (sum of edit costs)

But **Split Array Largest Sum** and **Painter's Partition** are *minimax* problems — you minimize the **largest** part, not the sum of parts. Their combine is:

```java
dp[g][i] = min over j of  Math.max(dp[g - 1][j], segmentSum(j, i - 1));
```

If you code the additive version for Split Array Largest Sum, you get wrong answers. Always ask: *am I summing the group costs, or minimizing the worst group?*

### Top-down

```java
int solve(int g, int i) {
    if (g == 0 && i == 0) return 0;         // used all groups, all elements
    if (g == 0 || i == 0) return INVALID;   // groups left but no elements, or vice versa
    if (memo[g][i] != null) return memo[g][i];

    int best = Integer.MAX_VALUE;
    for (int j = g - 1; j < i; j++) {       // need >= g-1 elements before for g-1 groups
        best = Math.min(best, solve(g - 1, j) + cost(j, i - 1));
        // minimax variant:
        // best = Math.min(best, Math.max(solve(g - 1, j), segmentCost(j, i - 1)));
    }
    return memo[g][i] = best;
}
```

### Bottom-up

```java
int[][] dp = new int[K + 1][n + 1];
for (int[] row : dp) Arrays.fill(row, Integer.MAX_VALUE);
dp[0][0] = 0;

for (int g = 1; g <= K; g++) {
    for (int i = 1; i <= n; i++) {
        for (int j = g - 1; j < i; j++) {       // last group is j .. i-1
            if (dp[g - 1][j] == Integer.MAX_VALUE) continue;

            // additive combine:
            dp[g][i] = Math.min(dp[g][i], dp[g - 1][j] + cost(j, i - 1));

            // minimax combine (Split Array Largest Sum / Painter's):
            // dp[g][i] = Math.min(dp[g][i],
            //                     Math.max(dp[g - 1][j], segmentSum(j, i - 1)));
        }
    }
}
```

### Meaning

The last group is `j .. i-1`. The first `j` elements are split into `g - 1` groups.

### Common Problems

- Split Array Largest Sum (minimax)
- Allocate Books / Painter's Partition (minimax)
- Minimum Difficulty of a Job Schedule (additive)
- Palindrome Partitioning III (additive)

---

## Useful Precomputation

Many partition DP problems become much easier if you precompute segment costs. This is often what turns an O(n³) solution into O(n²), or keeps the transition O(1).

Common precomputations:

```text
isPalindrome[i][j]
prefixSum[i]            // then segmentSum(l, r) = prefix[r+1] - prefix[l]
maxValue[i][j]
costToConvertToPalindrome[i][j]
```

Ask:

1. Can I calculate the cost of a segment quickly?
2. Can I avoid recomputing the same segment many times?
3. Can prefix sums, palindrome tables, or range maximums help?

---

## Important Recurrences

### Palindrome Partitioning II

Problem: Minimum cuts needed so every substring is a palindrome.

State: `dp[i]` = minimum cuts for `s[0 .. i-1]`.

```java
int[] dp = new int[n + 1];
dp[0] = -1;                                  // key trick, see below

for (int i = 1; i <= n; i++) {
    dp[i] = Integer.MAX_VALUE;
    for (int j = 0; j < i; j++) {
        if (isPalindrome[j][i - 1]) {
            dp[i] = Math.min(dp[i], dp[j] + 1);
        }
    }
}
return dp[n];
```

**Why `dp[0] = -1`?** If the whole prefix `s[0..i-1]` is already a palindrome, then `dp[i] = dp[0] + 1 = 0`, i.e. zero cuts. The `-1` cancels the `+1` so a single palindromic block costs nothing.

### Partition Array for Maximum Sum

Problem: Partition into groups of length at most `k`. Replace each group by (its max × its length). Maximize the total.

State: `dp[i]` = maximum sum for the first `i` elements.

```java
int[] dp = new int[n + 1];

for (int i = 1; i <= n; i++) {
    int mx = 0;
    for (int len = 1; len <= k && len <= i; len++) {
        mx = Math.max(mx, arr[i - len]);              // running max over the last group
        dp[i] = Math.max(dp[i], dp[i - len] + mx * len);
    }
}
return dp[n];
```

Note the running max: it lets each transition stay O(1) instead of rescanning the segment.

### Matrix Chain Multiplication (Interval, Form A)

Problem: Minimum cost to multiply a chain of matrices. Matrix `m` has dimensions `p[m-1] × p[m]`.

State: `dp[i][j]` = min cost to multiply matrices `i .. j`.

```java
// matrices indexed 1..n, dimension array p of length n+1
for (int len = 2; len <= n; len++) {
    for (int i = 1; i + len - 1 <= n; i++) {
        int j = i + len - 1;
        dp[i][j] = Integer.MAX_VALUE;

        for (int k = i; k < j; k++) {
            int cost = dp[i][k] + dp[k + 1][j] + p[i - 1] * p[k] * p[j];
            dp[i][j] = Math.min(dp[i][j], cost);
        }
    }
}
```

### Minimum Cost to Cut a Stick (Interval, Form B)

Problem: Given a stick of length `n` and cut positions, find the minimum total cutting cost. First sort the cuts and add boundaries `0` and `n`.

State: `dp[i][j]` = min cost to make all cuts strictly between `cuts[i]` and `cuts[j]`.

```java
// cuts sorted, with 0 prepended and n appended
for (int len = 2; len < m; len++) {          // m = cuts.length after padding
    for (int i = 0; i + len < m; i++) {
        int j = i + len;
        dp[i][j] = Integer.MAX_VALUE;

        for (int k = i + 1; k < j; k++) {    // k = the cut made LAST in this segment
            int cost = dp[i][k] + dp[k][j] + (cuts[j] - cuts[i]);
            dp[i][j] = Math.min(dp[i][j], cost);
        }
    }
}
```

The `cuts[j] - cuts[i]` term is the length of the current piece — the price you pay to cut it, no matter where `k` lands.

### Burst Balloons (Interval, Form B)

Problem: Burst balloons to maximize coins. Pad `nums` with `1` at both ends.

State: `dp[i][j]` = max coins from bursting every balloon in the **open** interval `(i, j)`.

```java
// nums padded: nums[0] = nums[n+1] = 1, real balloons at 1..n
for (int len = 2; len < N; len++) {          // N = nums.length after padding
    for (int i = 0; i + len < N; i++) {
        int j = i + len;

        for (int k = i + 1; k < j; k++) {    // k = the balloon burst LAST
            int coins = dp[i][k] + dp[k][j] + nums[i] * nums[k] * nums[j];
            dp[i][j] = Math.max(dp[i][j], coins);
        }
    }
}
```

The insight: bursting `k` **last** means its neighbors are still the walls `i` and `j`, so the gain is `nums[i] * nums[k] * nums[j]`.

### Boolean Parenthesization (Interval, Form A — needs extra state)

Problem: Count ways to parenthesize a boolean expression so it evaluates to a given value.

**This does not fit a single-value `dp[i][j]`.** Each interval must carry a *pair*: the number of ways it evaluates to true and to false, because combining two intervals across an operator (`&`, `|`, `^`) mixes both counts.

```java
// carry two tables (or a small class):
long[][] trueWays  = new long[n][n];
long[][] falseWays = new long[n][n];

// for each interval [i..j], split at every OPERATOR position k:
//   combine (trueWays[i][k-1], falseWays[i][k-1])
//      with (trueWays[k+1][j], falseWays[k+1][j])
//   according to the operator at k, accumulating into trueWays[i][j] / falseWays[i][j]
```

Same "extra state per interval" idea as Remove Boxes below — the single scalar isn't enough to reconstruct the answer.

---

## Must-Solve Partition DP Questions

### Beginner / Foundation

1. **Word Break** — Prefix. `dp[i] = true` if `s[0..i-1]` can be segmented.
2. **Perfect Squares** — Prefix. `dp[n]` = min number of perfect squares summing to `n`.
3. **Partition Equal Subset Sum** — Subset partition (not cut-based, but trains partition-style thinking).
4. **Palindrome Partitioning II** — Prefix. `dp[i]` = min cuts for `s[0..i-1]`.
5. **Partition Array for Maximum Sum** — Prefix. Last group has length at most `k`.

### Intermediate

6. **Matrix Chain Multiplication** — Interval, Form A. The classic.
7. **Minimum Cost to Cut a Stick** — Interval, Form B. Choose which cut to perform last.
8. **Burst Balloons** — Interval, Form B. Choose the last balloon to burst.
9. **Boolean Parenthesization** — Interval, Form A, extra state (true/false counts).
10. **Scramble String** — Interval / string partition. Split both strings, compare swapped and non-swapped.

### Advanced

11. **Split Array Largest Sum** — K-partition (**minimax**) or binary search.
12. **Painter's Partition / Allocate Books** — same minimax family as #11.
13. **Minimum Difficulty of a Job Schedule** — K-partition (**additive**). Each day's cost is the max difficulty that day.
14. **Palindrome Partitioning III** — K-partition (additive). Split into `k` palindromic substrings with min edits.
15. **Strange Printer** — Interval, Form A. Matching characters merge printing operations.
16. **Remove Boxes** — Advanced interval DP with **extra state** (`dp[i][j][k]`, carrying a count of trailing equal boxes). Final-boss tier.

---

## Recommended Practice Order

1. Word Break
2. Perfect Squares
3. Palindrome Partitioning II
4. Partition Array for Maximum Sum
5. Matrix Chain Multiplication
6. Minimum Cost to Cut a Stick
7. Burst Balloons
8. Split Array Largest Sum
9. Minimum Difficulty of Job Schedule
10. Palindrome Partitioning III
11. Boolean Parenthesization
12. Strange Printer
13. Remove Boxes

---

## Mental Checklist

For every partition DP problem, ask:

1. What does one segment contribute? (the `cost(...)` function)
2. Am I solving a **prefix** (1D `dp[i]`) or an **interval** (2D `dp[i][j]`)?
3. If interval — is it **Form A** (`k+1`, tile the elements) or **Form B** (`k`, pivot with sentinels)?
4. Is the number of parts fixed? If so, add the `g` dimension → K-partition.
5. Is the combine **additive** or **minimax** (or does it need extra state like true/false counts)?
6. Can I precompute the segment cost to keep the transition O(1)?

Answer these six and the recurrence usually writes itself. When in doubt, write the top-down recursion first and flip it.
