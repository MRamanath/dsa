# Sliding Window

## Learning Objectives

By the end of this topic, you should understand:

- how a sliding window reuses work across overlapping contiguous ranges
- the difference between fixed-size and variable-size windows
- how to define a window invariant before moving either boundary
- how adding on the right and removing on the left updates aggregate state
- when frequency maps, fixed arrays, sets, or scalar totals are appropriate
- why some shrinking rules require positive or nonnegative input
- how monotonic pointer movement leads to linear-time traversal
- how Unicode code-point contracts affect string windows and returned indexes
- how to generate bounded, meaningful window traces for a NestJS learning API

## Prerequisites

- contracts, invariants, and complexity reasoning from `00-dsa-fundamentals.md`
- array traversal and contiguous subarrays from `01-arrays.md`
- maps, sets, and frequency state from `02-hashing.md`
- code-point and normalization contracts from `03-strings.md`
- same-direction pointer movement from `04-two-pointers.md`
- TypeScript arrays, strings, maps, loops, functions, and tuple types

## Core Theory

### What is a sliding window?

A **sliding window** represents a contiguous range with two boundaries, commonly `[left, right]`. Instead of rebuilding each overlapping range, the algorithm updates state as values enter and leave.

```text
values: [2, 1, 5, 1, 3, 2]
             [-----]
             left right
```

Sliding window is useful only when the next range can reuse enough state from the current range.

### Why does the pattern exist?

For every length-`k` subarray, brute force may sum `k` values again:

```text
number of windows: O(n)
work per window:   O(k)
total:             O(nk)
```

A sliding sum removes the outgoing value and adds the incoming value:

```text
nextSum = currentSum - outgoing + incoming
```

Each value enters once and leaves at most once, reducing total work to `O(n)`.

### Fixed-size windows

A fixed-size window always contains exactly `k` items after initialization.

```text
[2, 1, 5] 1, 3, 2
 2,[1, 5, 1],3, 2
 2, 1,[5, 1, 3],2
```

Use fixed windows for:

- maximum or average over every size-`k` range
- counting a property in each size-`k` substring
- checking whether any range has exactly the same frequency signature as a pattern

The contract must define invalid `k` values such as `k <= 0` or `k > n`.

### Variable-size windows

A variable window expands right and shrinks left according to a validity rule.

```text
expand right -> window becomes invalid
shrink left  -> restore validity
record best valid window
```

Examples:

- no repeated characters
- at most `k` distinct characters
- enough characters to cover a target
- sum at least a target when all values are positive

### Window invariants

An invariant describes what the current range and its state mean. Examples include:

- `sum` equals the values in `[left, right]`
- every character in the window is unique
- `frequencies` exactly counts the current window
- `formed` is the number of target keys meeting their required count
- after shrinking, the window satisfies at most `k` distinct characters

Every boundary movement must update state in the same logical order as the invariant.

### Add and remove operations

Think of each iteration as two small state transitions:

```text
add input[right]
while invalid:
    remove input[left]
    left += 1
evaluate current window
```

Some optimization problems evaluate before shrinking instead. The problem's validity and objective determine the order.

### Choosing window state

| Question | Useful state |
|---|---|
| Sum or average | running numeric total |
| Number of vowels | running count |
| Unique characters | set or last-seen map |
| At most `k` distinct | frequency map plus distinct count |
| Required multiset covered | needed/window frequency maps plus matched-key count |
| Lowercase-English permutation | fixed 26-count arrays or difference state |

Store only what boundary updates need. Avoid rescanning the complete window after every movement.

### Longest versus shortest windows

For a **longest valid** range, shrink only while invalid, then maximize length.

For a **shortest covering** range, once valid, repeatedly shrink and minimize length until removing more would break validity.

```text
longest:  restore validity -> record maximum
shortest: reach validity   -> record minimum while shrinking
```

### Monotonicity and input restrictions

Window shrinking is not universally valid. For Minimum Size Subarray Sum, advancing `right` must never decrease the sum and advancing `left` must never increase it. That requires positive or nonnegative values, with details depending on the target contract.

With negative values, removing the leftmost value can increase the sum and adding a right value can decrease it. A different technique, such as prefix sums with a monotonic deque, may be required.

### Strings, code points, and indexes

JavaScript string indexes address UTF-16 code units. This chapter converts unrestricted strings with `[...text]` when windows operate on Unicode code points.

Returned positions then refer to code-point-array indexes, not UTF-16 offsets. APIs must label that distinction. Problems that explicitly restrict input to lowercase English letters may use a 26-element array instead.

## Visual Mental Model

### Fixed window rolls forward

```text
[2, 1, 5],1, 3, 2  sum 8
 2,[1, 5, 1],3, 2  sum 7 = 8 - 2 + 1
 2, 1,[5, 1, 3],2  sum 9 = 7 - 1 + 3
```

### Variable window restores validity

```text
"a b c a"
 [-----]       abc is unique
 [-------]     abca repeats a
   [-----]     bca is unique after left moves
```

### Frequency state follows boundaries

```text
window "aabc"

a -> 2
b -> 1
c -> 1

remove left a -> a becomes 1
```

### Minimum covering window

```text
target counts satisfied
[----------]  valid, record
  [--------]  still valid, record smaller
    [------]  invalid, expand right again
```

### Permutation signature

```text
pattern "ab": a1 b1
window  "ba": a1 b1 -> match
```

Order differs, but equal frequency signatures prove the window is a permutation.

## Complexity Concepts

Let `n` be the input length, `k` the fixed window size, and `u` the number of distinct keys.

| Pattern | Typical time | Extra space | Reason |
|---|---:|---:|---|
| Recompute every size-`k` range | `O(nk)` | `O(1)` | Each overlapping range is scanned again. |
| Fixed rolling aggregate | `O(n)` | `O(1)` | Each value is added and removed at most once. |
| Variable window with map | Average `O(n)` | `O(u)` | Both boundaries move only forward. |
| Fixed alphabet frequencies | `O(n)` | `O(1)` | Alphabet size is contractually bounded. |
| Materialize returned substring | `O(length)` | `O(length)` | Slicing/joining copies the result. |

### Why an inner while loop can still be linear

The right pointer advances `n` times. Across the whole algorithm, the left pointer also advances at most `n` times; it does not restart for each right position.

```text
right movements: at most n
left movements:  at most n
total:            O(2n) = O(n)
```

### Map operations are average constant time

`Map.get`, `Map.set`, and `Map.delete` are modeled as average `O(1)`. A frequency-based window is therefore average `O(n)`, not an unconditional worst-case guarantee for arbitrary hashing behavior.

### Frequency comparison costs matter

Comparing two frequency arrays on every step costs `O(a)` for alphabet size `a`:

```text
O(n * a)
```

When `a = 26`, this is `O(n)` under the fixed-alphabet contract. For an unbounded alphabet, maintain a mismatch or matched-key count to avoid full repeated comparisons.

### Output and trace costs

Returning all anagram start indexes uses `O(r)` output for `r` matches. A visualization that snapshots the entire window each step can use `O(n²)` serialized data even when the algorithm is `O(n)`.

Emit compact deltas such as boundary changes, entering/leaving values, aggregate updates, and best-window changes. Cap or sample traces for large inputs.

### Numeric safety

Sliding sums and averages use JavaScript `number`. Production contracts should validate finite safe integers and ensure running sums remain within safe range, or use another numeric representation consistently.

## Common Patterns

### Fixed rolling aggregate

**Pattern:** Build the first size-`k` window, then subtract the outgoing value and add the incoming value.

**Recognize it when:** every contiguous range has the same required size and the aggregate supports incremental updates.

**Similar problems:** maximum sum, averages, maximum vowels.

### Longest valid variable window

**Pattern:** Expand right, shrink while invalid, then record maximum length.

**Recognize it when:** the goal is a longest contiguous range under an at-most or uniqueness constraint.

**Similar problems:** no repeats, at most `k` distinct, character replacement.

### Minimum covering variable window

**Pattern:** Expand until requirements are met, then shrink while valid and record minimum length.

**Recognize it when:** the range must contain a required multiset or reach a monotonic threshold.

**Similar problems:** Minimum Window Substring and positive Minimum Size Subarray Sum.

### Fixed frequency signature

**Pattern:** Maintain frequencies for a pattern-sized window and compare them with target frequencies.

**Recognize it when:** a substring must be a permutation or anagram of a fixed pattern.

**Similar problems:** Permutation in String and Find All Anagrams.

### Last-seen jump

**Pattern:** Store each character's latest index and jump `left` past a duplicate inside the window.

**Recognize it when:** only uniqueness matters and direct boundary jumps replace one-by-one removal.

**Similar problems:** longest substring without repeats and streaming deduplication ranges.

---

# Problems

## Problem 61 — Maximum Sum Subarray of Size K

### 1. Problem Statement
Given an array of finite safe integers and a positive size `k`, return the maximum sum among all contiguous subarrays of exactly `k` values, or `null` when no such window exists.

### 2. Example
```text
Input: values = [2, 1, 5, 1, 3, 2], k = 3
Output: 9
```

### 3. Understand the Problem
The range must be contiguous and have exactly `k` elements. This contract returns `null` when `k <= 0` or `k > values.length`, and assumes every running sum remains a safe integer.

#### In Simple Terms

Look at every continuous block containing exactly `k` values and return the largest block sum.

### 4. Real-World Analogy
Compare revenue across every block of exactly three consecutive days without recounting the two overlapping days each time.

### 5. Key Observation
Adjacent size-`k` windows differ by one outgoing value and one incoming value.

### 6. Pattern
```text
Pattern: Fixed-size rolling sum
Why this pattern applies: every candidate has the same size and sums update incrementally.
How to recognize it: optimize an aggregate over every contiguous range of exactly K items.
Similar problems: fixed-window averages, maximum vowels, rolling sensor totals.
```

### 7. Brute Force
Start at every valid index and sum the next `k` values from scratch.

#### Pseudocode
```text
best = negative infinity
for each valid start
    sum = 0
    for offset from 0 to k - 1
        sum += values[start + offset]
    best = max(best, sum)
return best
```

#### Brute-Force TypeScript

```typescript
function maximumFixedWindowSumByRecomputing(values: number[], k: number): number | null {
  if (!Number.isInteger(k) || k <= 0 || k > values.length) return null;
  let best = Number.NEGATIVE_INFINITY;
  for (let start = 0; start + k <= values.length; start += 1) {
    let sum = 0;
    for (let offset = 0; offset < k; offset += 1) sum += values[start + offset];
    best = Math.max(best, sum);
  }
  return best;
}
```

#### Visualization
```text
[2,1,5] -> 8
  [1,5,1] -> 7
    [5,1,3] -> 9
```

#### Dry Run
For `k=3`, recomputed sums are `8`, `7`, `9`, and `6`; the maximum is `9`.

#### Brute-Force Complexity
**Time: `O((n-k+1)k)`, worst `O(nk)`**, because each window is fully summed. **Space: `O(1)`**.

### 8. Why Brute Force Is Not Ideal
Neighboring windows recount `k-1` shared values.

### 9. Optimal Approach
Build the first sum once. For each later right boundary, add its value, subtract the value leaving at `right-k`, and update the maximum.

### 10. Visualization
```text
[2,1,5],1,3,2 sum 8
  [1,5,1],3,2 sum 8 - 2 + 1 = 7
    [5,1,3],2 sum 7 - 1 + 3 = 9
```

### 11. Step-by-Step Dry Run
| Window | Out | In | Sum | Best |
|---|---:|---:|---:|---:|
| `[2,1,5]` | - | - | 8 | 8 |
| `[1,5,1]` | 2 | 1 | 7 | 8 |
| `[5,1,3]` | 1 | 3 | 9 | 9 |
| `[1,3,2]` | 5 | 2 | 6 | 9 |

### 12. TypeScript Solution
```typescript
function maximumFixedWindowSum(values: number[], k: number): number | null {
  if (!Number.isInteger(k) || k <= 0 || k > values.length) return null;
  let windowSum = 0;
  for (let index = 0; index < k; index += 1) windowSum += values[index];
  let best = windowSum;
  for (let right = k; right < values.length; right += 1) {
    windowSum += values[right] - values[right - k];
    best = Math.max(best, windowSum);
  }
  return best;
}
```

### 13. Code Walkthrough
The initialization establishes one complete window. During each slide, `right-k` is exactly the outgoing index, preserving the invariant that `windowSum` covers the latest `k` values.

### 14. Complexity Analysis
**Time: `O(n)`** because initialization and sliding together process a linear number of values. **Extra space: `O(1)`** because only scalar state is stored.

### 15. Edge Cases
- Invalid `k` returns `null`.
- `k === values.length` returns the total sum.
- Negative values work because window size is fixed.
- Running sums must remain safe finite integers.

### 16. Common Mistakes
- Initializing `best` to zero when all sums may be negative.
- Removing the wrong outgoing index.
- Treating a subsequence as a subarray.
- Omitting invalid-`k` behavior.

### 17. Interview Follow-ups
1. Return the best start index. 2. Process a stream. 3. Support many `k` queries. 4. Return every tied window.

### 18. Variations
- Minimum fixed-window sum. - Maximum average. - Rolling weighted sum.

### 19. Pattern Connections
The same rolling update powers averages and property counts over fixed windows.

### 20. NestJS Project Connection
Expose `POST /api/dsa/sliding-window/maximum-sum` with `{ values, k }`. Return the maximum and steps `{ left, right, outgoing, incoming, sum, best }` after safe-number validation.

### 21. Mini Challenge
Return `{ sum, start, end }`, choosing the earliest window when several have the maximum sum.

## Problem 62 — Average of All Subarrays of Size K

### 1. Problem Statement
Return the arithmetic average of every contiguous subarray of exactly `k` finite numbers; return an empty array when `k` is invalid.

### 2. Example
```text
Input: values = [1, 3, 2, 6, -1, 4, 1, 8, 2], k = 5
Output: [2.2, 2.8, 2.4, 3.6, 2.8]
```

### 3. Understand the Problem
Results follow window start order. This contract returns `[]` for noninteger `k`, `k <= 0`, or `k > n`; numeric inputs and rolling sums must be finite.

#### In Simple Terms

For every continuous block of exactly `k` values, calculate its average and return all averages in start order.

### 4. Real-World Analogy
Calculate every five-day moving average by updating yesterday's total instead of rereading all five daily values.

### 5. Key Observation
Every average is a rolling sum divided by the same `k`, so only the sum requires incremental maintenance.

### 6. Pattern
```text
Pattern: Fixed-size rolling aggregate
Why this pattern applies: equal-size ranges overlap and division happens after each sum update.
How to recognize it: produce one statistic for every contiguous K-item range.
Similar problems: moving averages, rolling rates, fixed-window sums.
```

### 7. Brute Force
For each valid start, sum its `k` values and divide by `k`.

#### Pseudocode
```text
averages = empty
for each valid start
    sum next k values
    append sum / k
```

#### Brute-Force TypeScript

```typescript
function fixedWindowAveragesByRecomputing(values: number[], k: number): number[] {
  if (!Number.isInteger(k) || k <= 0 || k > values.length) return [];
  const averages: number[] = [];
  for (let start = 0; start + k <= values.length; start += 1) {
    let sum = 0;
    for (let offset = 0; offset < k; offset += 1) sum += values[start + offset];
    averages.push(sum / k);
  }
  return averages;
}
```

#### Visualization
```text
[1,3,2,6,-1] -> 11/5 = 2.2
  [3,2,6,-1,4] -> 14/5 = 2.8
```

#### Dry Run
The first two complete sums are recomputed as `11` and `14`, despite sharing four values.

#### Brute-Force Complexity
**Time: `O(nk)`** in the worst case. **Space: `O(n-k+1)`** for the required output.

### 8. Why Brute Force Is Not Ideal
It repeats additions for the overlapping portion of every adjacent range.

### 9. Optimal Approach
Add each new right value. Once the window reaches size `k`, emit `sum/k`, remove the left value, and advance left.

### 10. Visualization
```text
add until size 5 -> emit 11/5
remove 1, add 4 -> sum 14 -> emit 14/5
```

### 11. Step-by-Step Dry Run
| Window | Sum | Average |
|---|---:|---:|
| `[1,3,2,6,-1]` | 11 | 2.2 |
| `[3,2,6,-1,4]` | 14 | 2.8 |
| `[2,6,-1,4,1]` | 12 | 2.4 |

### 12. TypeScript Solution
```typescript
function fixedWindowAverages(values: number[], k: number): number[] {
  if (!Number.isInteger(k) || k <= 0 || k > values.length) return [];
  const averages: number[] = [];
  let left = 0;
  let sum = 0;
  for (let right = 0; right < values.length; right += 1) {
    sum += values[right];
    if (right - left + 1 === k) {
      averages.push(sum / k);
      sum -= values[left];
      left += 1;
    }
  }
  return averages;
}
```

### 13. Code Walkthrough
The window grows only until size `k`. Emission occurs before removal, so each output uses exactly `k` values; removal prepares the next overlapping window.

### 14. Complexity Analysis
**Time: `O(n)`** because each value enters once and leaves once. **Space: `O(n-k+1)`** for the required averages; auxiliary state excluding output is `O(1)`.

### 15. Edge Cases
- Invalid `k` returns empty.
- `k=1` returns the input values as averages.
- Negative and fractional finite values are valid.
- Floating-point results may contain normal IEEE-754 rounding.

### 16. Common Mistakes
- Dividing before a complete window exists.
- Removing the outgoing value before recording the current average.
- Claiming constant total space while returning an output array.
- Expecting exact decimal arithmetic from binary floating point.

### 17. Interview Follow-ups
1. Stream moving averages. 2. Round to a requested precision. 3. Use exact decimal arithmetic. 4. Return rolling variance.

### 18. Variations
- Exponential moving average. - Weighted moving average. - Rolling median.

### 19. Pattern Connections
This is the fixed-sum invariant with one output emitted for every valid boundary position.

### 20. NestJS Project Connection
Use `POST /api/dsa/sliding-window/averages` with `{ values, k }`. Return averages and compact add/emit/remove trace events; reject nonfinite values at DTO validation.

### 21. Mini Challenge
Return fixed-window variance in linear time by maintaining both sum and sum of squares, and discuss numeric stability.

## Problem 63 — Maximum Number of Vowels in a Substring of Size K

### 1. Problem Statement
Given a string and positive code-point window size `k`, return the largest count of ASCII vowels in any contiguous substring of exactly `k` code points.

### 2. Example
```text
Input: text = "abciiidef", k = 3
Output: 3
```

### 3. Understand the Problem
Vowels are `a,e,i,o,u` in either ASCII case. Other code points are nonvowels. Invalid `k` throws `RangeError`, and positions are measured in code points.

#### In Simple Terms

Check every `k`-code-point piece of the string and return the largest number of ASCII vowels found in one piece.

### 4. Real-World Analogy
Slide a magnifier covering exactly three symbols and keep the largest number of highlighted vowel cards.

### 5. Key Observation
The vowel count changes only when the outgoing or incoming code point is a vowel.

### 6. Pattern
```text
Pattern: Fixed-size rolling property count
Why this pattern applies: each symbol contributes either 0 or 1 to a fixed window.
How to recognize it: maximize a countable property over every K-symbol substring.
Similar problems: maximum ones in a fixed range, rolling category count.
```

### 7. Brute Force
Build every size-`k` substring range and count its vowels independently.

#### Pseudocode
```text
best = 0
for each valid start
    count vowels across k code points
    best = max(best, count)
```

#### Brute-Force TypeScript

```typescript
function maximumVowelsByRecounting(text: string, k: number): number {
  const characters = [...text];
  if (!Number.isInteger(k) || k <= 0 || k > characters.length) {
    throw new RangeError("k must be between 1 and the code-point length");
  }
  let best = 0;
  for (let start = 0; start + k <= characters.length; start += 1) {
    let count = 0;
    for (let offset = 0; offset < k; offset += 1) {
      if ("aeiou".includes(characters[start + offset].toLowerCase())) count += 1;
    }
    best = Math.max(best, count);
  }
  return best;
}
```

#### Visualization
```text
"abc" -> 1
 "bci" -> 1
  "cii" -> 2
   "iii" -> 3
```

#### Dry Run
The windows ending in the three consecutive `i` characters increase the recomputed count from one to two to three.

#### Brute-Force Complexity
**Time: `O(nk)`** for repeated counting. **Space: `O(n)`** for the code-point representation used by this contract.

### 8. Why Brute Force Is Not Ideal
It recounts the shared `k-1` symbols after every one-position shift.

### 9. Optimal Approach
Convert to code points, count the first window, then subtract the outgoing vowel contribution and add the incoming contribution.

### 10. Visualization
```text
"a b c" i i i d e f -> 1
  "b c i"           -> 1 - vowel(a) + vowel(i) = 1
    "c i i"         -> 2
      "i i i"       -> 3
```

### 11. Step-by-Step Dry Run
| Window | Out | In | Vowels | Best |
|---|---|---|---:|---:|
| `abc` | - | - | 1 | 1 |
| `bci` | a | i | 1 | 1 |
| `cii` | b | i | 2 | 2 |
| `iii` | c | i | 3 | 3 |

### 12. TypeScript Solution
```typescript
function maximumVowels(text: string, k: number): number {
  const characters = [...text];
  if (!Number.isInteger(k) || k <= 0 || k > characters.length) {
    throw new RangeError("k must be between 1 and the code-point length");
  }
  const isVowel = (character: string): boolean => "aeiou".includes(character.toLowerCase());
  let count = 0;
  for (let index = 0; index < k; index += 1) count += Number(isVowel(characters[index]));
  let best = count;
  for (let right = k; right < characters.length; right += 1) {
    count += Number(isVowel(characters[right]));
    count -= Number(isVowel(characters[right - k]));
    best = Math.max(best, count);
  }
  return best;
}
```

### 13. Code Walkthrough
`Number(boolean)` converts each membership result to `0` or `1`. The rolling update preserves the exact vowel count for the latest `k` code points.

### 14. Complexity Analysis
**Time: `O(n)`** for code-point conversion and one rolling pass. **Space: `O(n)`** for the code-point array; rolling state is `O(1)`.

### 15. Edge Cases
- Invalid `k` throws.
- Uppercase ASCII vowels count.
- Accented vowels do not count under this contract.
- `k` equal to code-point length evaluates the whole string once.

### 16. Common Mistakes
- Using UTF-16 `.length` while claiming code-point windows.
- Recounting each complete window.
- Omitting uppercase behavior.
- Treating `y` or accented letters as vowels without a stated policy.

### 17. Interview Follow-ups
1. Return the earliest best substring. 2. Accept a configurable symbol set. 3. Count grapheme clusters. 4. Process streaming text.

### 18. Variations
- Minimum consonants in size `k`. - Fixed-window digit count. - Maximum weighted character score.

### 19. Pattern Connections
This replaces a numeric value with a Boolean contribution but otherwise matches a rolling sum.

### 20. NestJS Project Connection
Expose `POST /api/dsa/sliding-window/maximum-vowels` with `{ text, k }`. Return the maximum plus trace deltas `{ outgoing, incoming, count, best, left, right }` labeled as code-point indexes.

### 21. Mini Challenge
Return the earliest best substring and its code-point start index without slicing every candidate.

## Problem 64 — Longest Substring Without Repeating Characters

### 1. Problem Statement
Return the maximum code-point length of a contiguous substring containing no repeated code point.

### 2. Example
```text
Input: "abcabcbb"
Output: 3
```

### 3. Understand the Problem
The substring must be contiguous, comparison is exact and case-sensitive, and length is measured in Unicode code points rather than UTF-16 code units.

#### In Simple Terms

Find the longest continuous piece with no repeated code point and return its length.

### 4. Real-World Analogy
Keep a row of unique badges; when a badge repeats, discard through its previous occurrence before continuing.

### 5. Key Observation
The last index of the new right character tells us where `left` must jump. A previous occurrence outside the current window is irrelevant.

### 6. Pattern
```text
Pattern: Variable window with last-seen jumps
Why this pattern applies: validity requires every current character to be unique.
How to recognize it: longest contiguous range with no repeated key.
Similar problems: unique event spans, longest range after duplicate reset.
```

### 7. Brute Force
Start at every code-point index and extend a set-backed range until a duplicate appears.

#### Pseudocode
```text
for each start
    seen = empty set
    for end from start onward
        stop if character is already seen
        add character and update best
```

#### Brute-Force TypeScript

```typescript
function longestUniqueSubstringByEveryStart(text: string): number {
  const characters = [...text];
  let best = 0;
  for (let start = 0; start < characters.length; start += 1) {
    const seen = new Set<string>();
    for (let end = start; end < characters.length; end += 1) {
      if (seen.has(characters[end])) break;
      seen.add(characters[end]);
      best = Math.max(best, end - start + 1);
    }
  }
  return best;
}
```

#### Visualization
```text
start 0: abc then repeated a stops
start 1: bca then repeated b stops
```

#### Dry Run
For `abcabcbb`, starts at `a`, `b`, and `c` each rebuild overlapping unique sets and find length three.

#### Brute-Force Complexity
**Time: `O(n²)`** in the worst case when many starts extend far. **Space: `O(u)`** for up to `u` distinct code points plus `O(n)` representation.

### 8. Why Brute Force Is Not Ideal
It rescans and rebuilds state for heavily overlapping candidate substrings.

### 9. Optimal Approach
Move right once. When its last occurrence lies at or after `left`, jump `left` one position beyond it; then update the last index and best length.

### 10. Visualization
```text
abc -> best 3
abca -> repeated a at 0, left jumps to 1 -> bca
bcab -> repeated b at 1, left jumps to 2 -> cab
```

### 11. Step-by-Step Dry Run
| Right | Character | Previous | Left after | Length | Best |
|---:|---|---:|---:|---:|---:|
| 0 | a | - | 0 | 1 | 1 |
| 1 | b | - | 0 | 2 | 2 |
| 2 | c | - | 0 | 3 | 3 |
| 3 | a | 0 | 1 | 3 | 3 |
| 4 | b | 1 | 2 | 3 | 3 |

### 12. TypeScript Solution
```typescript
function longestUniqueSubstringLength(text: string): number {
  const characters = [...text];
  const lastIndex = new Map<string, number>();
  let left = 0;
  let best = 0;
  for (let right = 0; right < characters.length; right += 1) {
    const previous = lastIndex.get(characters[right]);
    if (previous !== undefined && previous >= left) left = previous + 1;
    lastIndex.set(characters[right], right);
    best = Math.max(best, right - left + 1);
  }
  return best;
}
```

### 13. Code Walkthrough
The `previous >= left` guard prevents `left` from moving backward because of stale occurrences. The map stores code-point-array indexes.

### 14. Complexity Analysis
**Time: average `O(n)`** for conversion and average constant-time map operations. **Space: `O(n)`** for code points and up to `O(u)` map entries.

### 15. Edge Cases
- Empty returns zero.
- All equal returns one.
- All unique returns code-point length.
- Case variants are distinct.

### 16. Common Mistakes
- Moving `left` backward.
- Checking a stored index by truthiness and losing index zero.
- Solving for a subsequence instead of substring.
- Mixing code-point indexes with UTF-16 offsets.

### 17. Interview Follow-ups
1. Return the earliest longest substring. 2. Use grapheme clusters. 3. Support at most one duplicate. 4. Process an event stream.

### 18. Variations
- Longest substring with at most `k` distinct. - Longest repeating replacement. - Maximum unique circular range.

### 19. Pattern Connections
This specialized jump window avoids one-by-one frequency removal because the exact duplicate location is known.

### 20. NestJS Project Connection
Use `POST /api/dsa/sliding-window/longest-unique` with `{ text }`. Return length and bounded events `{ right, character, previous, leftBefore, leftAfter, best }` using code-point indexes.

### 21. Mini Challenge
Return the earliest longest substring itself while avoiding a copy every time `best` changes.

## Problem 65 — Longest Substring With at Most K Distinct Characters

### 1. Problem Statement
Return the maximum code-point length of a contiguous substring containing at most `k` distinct code points.

### 2. Example
```text
Input: text = "eceba", k = 2
Output: 3
```

### 3. Understand the Problem
Comparison is exact and case-sensitive. For noninteger `k` or `k <= 0`, this contract returns `0`; length and indexes use code points.

#### In Simple Terms

Find the longest continuous piece that uses no more than `k` different code points.

### 4. Real-World Analogy
Keep the longest stretch of products containing no more than `k` categories, removing products from the left when a new category exceeds the limit.

### 5. Key Observation
A frequency map says both how many distinct characters are present and when removing a left character eliminates a category entirely.

### 6. Pattern
```text
Pattern: Longest variable window with an at-most constraint
Why this pattern applies: adding can violate the distinct-key limit and left removal restores it.
How to recognize it: longest contiguous range containing at most K categories.
Similar problems: fruit into baskets, at most K frequencies, exactly K distinct.
```

### 7. Brute Force
Start at every position, build frequencies to the right, and stop when distinct count exceeds `k`.

#### Pseudocode
```text
for each start
    frequencies = empty
    extend end
    stop after distinct count exceeds k
    update best for valid ranges
```

#### Brute-Force TypeScript

```typescript
function longestAtMostKDistinctByEveryStart(text: string, k: number): number {
  if (!Number.isInteger(k) || k <= 0) return 0;
  const characters = [...text];
  let best = 0;
  for (let start = 0; start < characters.length; start += 1) {
    const frequencies = new Map<string, number>();
    for (let end = start; end < characters.length; end += 1) {
      const character = characters[end];
      frequencies.set(character, (frequencies.get(character) ?? 0) + 1);
      if (frequencies.size > k) break;
      best = Math.max(best, end - start + 1);
    }
  }
  return best;
}
```

#### Visualization
```text
start e: e -> ec -> ece valid; eceb has 3 distinct, stop
```

#### Dry Run
Starting at index zero in `eceba` reaches `ece` with `{e,c}` and length three, then `b` exceeds two categories.

#### Brute-Force Complexity
**Time: `O(n²)`** worst case due to restarted extensions. **Space: `O(u)`** for frequencies plus `O(n)` code-point representation.

### 8. Why Brute Force Is Not Ideal
Every new start rebuilds frequencies for overlapping text.

### 9. Optimal Approach
Add each right character. While map size exceeds `k`, decrement the left character and delete it at zero. Once valid, maximize length.

### 10. Visualization
```text
ece -> {e:2,c:1}, valid, best 3
eceb -> {e:2,c:1,b:1}, invalid
shrink e, then c -> eb has 2 distinct, valid
```

### 11. Step-by-Step Dry Run
| Right | Added | Window after shrinking | Distinct | Best |
|---:|---|---|---:|---:|
| 0 | e | `e` | 1 | 1 |
| 1 | c | `ec` | 2 | 2 |
| 2 | e | `ece` | 2 | 3 |
| 3 | b | `eb` | 2 | 3 |
| 4 | a | `ba` | 2 | 3 |

### 12. TypeScript Solution
```typescript
function longestSubstringAtMostKDistinct(text: string, k: number): number {
  if (!Number.isInteger(k) || k <= 0) return 0;
  const characters = [...text];
  const frequencies = new Map<string, number>();
  let left = 0;
  let best = 0;
  for (let right = 0; right < characters.length; right += 1) {
    const added = characters[right];
    frequencies.set(added, (frequencies.get(added) ?? 0) + 1);
    while (frequencies.size > k) {
      const removed = characters[left];
      const nextCount = frequencies.get(removed)! - 1;
      if (nextCount === 0) frequencies.delete(removed);
      else frequencies.set(removed, nextCount);
      left += 1;
    }
    best = Math.max(best, right - left + 1);
  }
  return best;
}
```

### 13. Code Walkthrough
The map exactly counts `[left,right]`. Deleting zero-count keys makes `frequencies.size` equal the distinct count, so the shrink condition directly represents validity.

### 14. Complexity Analysis
**Time: average `O(n)`** because both boundaries move forward and map operations are average constant time. **Space: `O(n)`** for code points and up to `O(min(n,k+1))` frequency keys during adjustment.

### 15. Edge Cases
- Empty or nonpositive `k` returns zero.
- `k` above the distinct count returns full code-point length.
- Repeated one-character text needs one map key.
- Case variants count separately.

### 16. Common Mistakes
- Forgetting to delete zero-count keys.
- Shrinking only once when several categories may need removal.
- Recording length before restoring validity.
- Counting UTF-16 units under a code-point contract.

### 17. Interview Follow-ups
1. Return the substring and boundaries. 2. Require exactly `k` distinct. 3. Count substrings with exactly `k` distinct. 4. Process streaming symbols.

### 18. Variations
- Fruit Into Baskets (`k=2`). - At least `k` distinct. - At most `k` distinct array values.

### 19. Pattern Connections
Exactly-`k` substring counts can be derived from at-most counts: `atMost(k) - atMost(k-1)`.

### 20. NestJS Project Connection
Expose `POST /api/dsa/sliding-window/at-most-k-distinct` with `{ text, k }`. Return the best length plus add/remove/map-size/boundary trace events labeled as code-point positions.

### 21. Mini Challenge
Return the number of substrings containing exactly `k` distinct code points by composing two at-most counters.

## Problem 66 — Longest Repeating Character Replacement

### 1. Problem Statement
Given a string and a nonnegative integer `k`, return the maximum code-point length of a substring that can be changed into one repeated code point by replacing at most `k` positions.

### 2. Example
```text
Input: text = "AABABBA", k = 1
Output: 4
```

### 3. Understand the Problem
The substring is contiguous, comparisons are exact and case-sensitive, and replacements may choose any code point. Invalid `k` returns `0`; the input itself is not modified.

#### In Simple Terms

Find the longest continuous piece that can become one repeated code point after changing at most `k` positions.

### 4. Real-World Analogy
Choose the longest row of colored tiles that can become one color after repainting at most `k` outliers.

### 5. Key Observation
For a window of length `w`, keep its most frequent character and replace the other `w - maxFrequency` positions. The window is feasible exactly when that cost is at most `k`.

### 6. Pattern
```text
Pattern: Longest variable window with replacement budget
Why this pattern applies: expand while replacement cost is affordable and shrink when it exceeds K.
How to recognize it: make a window uniform by changing a bounded number of outliers.
Similar problems: maximum consecutive ones with flips, frequency-budget windows.
```

### 7. Brute Force
Start at every position, extend right, maintain frequencies, and compute the maximum frequency for each candidate window.

#### Pseudocode
```text
for each start
    frequencies = empty
    for end from start onward
        add character
        recompute maximum frequency
        if window length - maximum <= k
            update best
```

#### Brute-Force TypeScript

```typescript
function longestRepeatingReplacementByEveryStart(text: string, k: number): number {
  if (!Number.isInteger(k) || k < 0) return 0;
  const characters = [...text];
  let best = 0;
  for (let start = 0; start < characters.length; start += 1) {
    const frequencies = new Map<string, number>();
    for (let end = start; end < characters.length; end += 1) {
      const character = characters[end];
      frequencies.set(character, (frequencies.get(character) ?? 0) + 1);
      let maximumFrequency = 0;
      for (const count of frequencies.values()) maximumFrequency = Math.max(maximumFrequency, count);
      const length = end - start + 1;
      if (length - maximumFrequency <= k) best = Math.max(best, length);
    }
  }
  return best;
}
```

#### Visualization
```text
"AABA": length 4, max frequency A=3, replacements=1 -> valid
```

#### Dry Run
For `AABABBA` with `k=1`, windows beginning at zero include `AABA`, which needs one `B` replacement and has length four.

#### Brute-Force Complexity
**Time: `O(n²u)`** if the maximum is rescanned across up to `u` frequency keys for every candidate; with careful per-start updates it remains at least `O(n²)`. **Space: `O(u)`** plus code-point representation.

### 8. Why Brute Force Is Not Ideal
It restarts overlapping ranges and repeatedly searches frequency state for a dominant character.

### 9. Optimal Approach
Expand right while tracking frequencies and the largest frequency observed in the current expansion history. Shrink whenever `windowLength - maxFrequency > k`, then record the longest feasible length.

### 10. Visualization
```text
A A B A -> length 4, max 3, cost 1, valid
A A B A B -> length 5, max 3, cost 2, invalid for k=1
  A B A B -> shrink left; continue
```

### 11. Step-by-Step Dry Run
| Right | Added | Left after | Max frequency | Window length | Best |
|---:|---|---:|---:|---:|---:|
| 0 | A | 0 | 1 | 1 | 1 |
| 1 | A | 0 | 2 | 2 | 2 |
| 2 | B | 0 | 2 | 3 | 3 |
| 3 | A | 0 | 3 | 4 | 4 |
| 4 | B | 1 | 3 | 4 | 4 |

### 12. TypeScript Solution
```typescript
function longestRepeatingReplacement(text: string, k: number): number {
  if (!Number.isInteger(k) || k < 0) return 0;
  const characters = [...text];
  const frequencies = new Map<string, number>();
  let left = 0;
  let maxFrequency = 0;
  let best = 0;
  for (let right = 0; right < characters.length; right += 1) {
    const added = characters[right];
    const addedCount = (frequencies.get(added) ?? 0) + 1;
    frequencies.set(added, addedCount);
    maxFrequency = Math.max(maxFrequency, addedCount);
    while (right - left + 1 - maxFrequency > k) {
      const removed = characters[left];
      frequencies.set(removed, frequencies.get(removed)! - 1);
      left += 1;
    }
    best = Math.max(best, right - left + 1);
  }
  return best;
}
```

### 13. Code Walkthrough
`maxFrequency` is intentionally not decreased during shrinking. It may become stale, but it only delays shrinking enough to preserve a previously achievable window length; it cannot create a larger reported length without a frequency at least that large having existed.

### 14. Complexity Analysis
**Time: average `O(n)`** because each boundary moves forward and map updates are average constant time. **Space: `O(n)`** for code points and up to `O(u)` frequency keys.

### 15. Edge Cases
- Empty returns zero.
- `k=0` finds the longest existing run.
- `k` at least `n-1` can make the whole nonempty string uniform.
- Case variants are different code points.

### 16. Common Mistakes
- Shrinking when `windowLength - maxFrequency >= k` instead of `> k`.
- Recomputing the maximum on every removal and losing the simple linear proof.
- Assuming the stale maximum equals the exact current-window maximum.
- Mutating the source string unnecessarily.

### 17. Interview Follow-ups
1. Return the chosen substring. 2. Return a replacement plan. 3. Restrict replacement to a target symbol. 4. Use weighted replacement costs.

### 18. Variations
- Max Consecutive Ones III. - Longest run after deletions. - Uniform window under a cost budget.

### 19. Pattern Connections
This is an at-most-budget window where the dominant frequency determines how many positions violate the desired uniform state.

### 20. NestJS Project Connection
Expose `POST /api/dsa/sliding-window/repeating-replacement` with `{ text, k }`. Return length and bounded steps `{ left, right, added, removed, maxFrequency, replacementCost, best }`.

### 21. Mini Challenge
Return the earliest longest window and a code point that can be used as its final repeated value.

## Problem 67 — Minimum Window Substring

### 1. Problem Statement
Return the shortest substring of `source` containing every code point from `target` with at least the target multiplicity; return `""` when no such window exists.

### 2. Example
```text
Input: source = "ADOBECODEBANC", target = "ABC"
Output: "BANC"
```

### 3. Understand the Problem
Comparison is exact and case-sensitive. Source and target are processed as code points; an empty target returns empty. On equal minimum lengths, return the earliest window.

#### In Simple Terms

Return the shortest continuous piece of `source` that contains every required code point from `target`, including repeated requirements.

### 4. Real-World Analogy
Find the shortest consecutive shelf segment containing the requested quantities of every product on a shopping list.

### 5. Key Observation
Track how many distinct target keys currently meet their required counts. Once all are satisfied, shrink from the left until further removal would break coverage.

### 6. Pattern
```text
Pattern: Minimum covering variable window
Why this pattern applies: the window must cover a required multiset, then discard excess from the left.
How to recognize it: shortest contiguous range containing all required keys with multiplicity.
Similar problems: smallest covering range, minimum positive-sum window, covering tokens.
```

### 7. Brute Force
Generate every source substring and compare its frequency map with the target requirements.

#### Pseudocode
```text
for each start
    frequencies = empty
    for each end at or after start
        add source[end]
        if all target counts are covered
            update shortest and stop extending this start
```

#### Brute-Force TypeScript

```typescript
function minimumWindowByCheckingEveryStart(source: string, target: string): string {
  const sourceCharacters = [...source];
  const targetCharacters = [...target];
  if (targetCharacters.length === 0) return "";
  const needed = new Map<string, number>();
  for (const character of targetCharacters) {
    needed.set(character, (needed.get(character) ?? 0) + 1);
  }
  let bestStart = 0;
  let bestLength = Number.POSITIVE_INFINITY;
  for (let start = 0; start < sourceCharacters.length; start += 1) {
    const frequencies = new Map<string, number>();
    for (let end = start; end < sourceCharacters.length; end += 1) {
      const character = sourceCharacters[end];
      frequencies.set(character, (frequencies.get(character) ?? 0) + 1);
      let coversTarget = true;
      for (const [required, count] of needed) {
        if ((frequencies.get(required) ?? 0) < count) {
          coversTarget = false;
          break;
        }
      }
      if (coversTarget) {
        const length = end - start + 1;
        if (length < bestLength) {
          bestStart = start;
          bestLength = length;
        }
        break;
      }
    }
  }
  return Number.isFinite(bestLength)
    ? sourceCharacters.slice(bestStart, bestStart + bestLength).join("")
    : "";
}
```

#### Visualization
```text
all O(n^2) ranges, each checked against target requirements
```

#### Dry Run
In `ADOBECODEBANC`, broad valid ranges such as `ADOBEC` are found before the later shorter range `BANC`.

#### Brute-Force Complexity
**Time: `O(n²u)`** when up to `O(n²)` ranges check `u` target keys. **Space: `O(n+u)`** for code points and frequencies.

### 8. Why Brute Force Is Not Ideal
It rebuilds or repeatedly checks almost identical coverage state for overlapping ranges.

### 9. Optimal Approach
Build target counts, expand right to satisfy them, then repeatedly record and shrink valid windows. Update a matched-key count only when a key crosses its required threshold.

### 10. Visualization
```text
ADOBEC -> covers A,B,C, record length 6
shrink until A removed -> invalid
expand ... BANC -> valid, shrink and record length 4
```

### 11. Step-by-Step Dry Run
`ADOBEC` first makes `formed=3`, so length six is recorded. Removing `A` breaks coverage. Later the window reaches `CODEBANC`; shrinking excess symbols leaves `BANC`, which still covers all three keys and becomes the best.

### 12. TypeScript Solution
```typescript
function minimumWindowSubstring(source: string, target: string): string {
  const sourceCharacters = [...source];
  const targetCharacters = [...target];
  if (targetCharacters.length === 0) return "";

  const needed = new Map<string, number>();
  for (const character of targetCharacters) {
    needed.set(character, (needed.get(character) ?? 0) + 1);
  }

  const window = new Map<string, number>();
  let formed = 0;
  let left = 0;
  let bestStart = 0;
  let bestLength = Number.POSITIVE_INFINITY;

  for (let right = 0; right < sourceCharacters.length; right += 1) {
    const added = sourceCharacters[right];
    if (needed.has(added)) {
      const count = (window.get(added) ?? 0) + 1;
      window.set(added, count);
      if (count === needed.get(added)) formed += 1;
    }

    while (formed === needed.size) {
      const length = right - left + 1;
      if (length < bestLength) {
        bestStart = left;
        bestLength = length;
      }
      const removed = sourceCharacters[left];
      if (needed.has(removed)) {
        const count = window.get(removed)!;
        if (count === needed.get(removed)) formed -= 1;
        window.set(removed, count - 1);
      }
      left += 1;
    }
  }

  if (!Number.isFinite(bestLength)) return "";
  return sourceCharacters.slice(bestStart, bestStart + bestLength).join("");
}
```

### 13. Code Walkthrough
`formed` counts satisfied distinct requirements, not total matched characters. Strictly shorter updates preserve the earliest tie; final slicing occurs once instead of copying each candidate.

### 14. Complexity Analysis
**Time: average `O(n+m)`** for code-point conversion, target counting, and monotonic source boundaries. **Space: `O(n+m+u)`** for source/target arrays, maps, and the returned substring.

### 15. Edge Cases
- Empty target returns empty.
- Target longer than source cannot be covered.
- Repeated target characters require repeated source occurrences.
- Case variants are distinct.

### 16. Common Mistakes
- Tracking only membership instead of multiplicity.
- Incrementing `formed` for every extra copy beyond the required count.
- Recording only before shrinking and missing a smaller valid range.
- Slicing on every valid step and hiding extra copying costs.

### 17. Interview Follow-ups
1. Return code-point boundaries. 2. Support streamed source text. 3. Ignore case under normalization. 4. Find all minimum ties.

### 18. Variations
- Smallest range covering lists. - Minimum window subsequence. - Minimum token-span query.

### 19. Pattern Connections
Unlike longest-valid windows, this algorithm continues shrinking while valid because the objective is minimum coverage.

### 20. NestJS Project Connection
Use `POST /api/dsa/sliding-window/minimum-window` with `{ source, target }`. Return the result and capped events for add/remove, requirement thresholds, validity transitions, and best-range updates.

### 21. Mini Challenge
Return every earliest-to-latest window tied for minimum length without copying candidates during traversal.

## Problem 68 — Permutation in String

### 1. Problem Statement
Return whether `text` contains a contiguous substring that is a permutation of `pattern`, comparing exact Unicode code points and multiplicities.

### 2. Example
```text
Input: pattern = "ab", text = "eidbaooo"
Output: true
```

### 3. Understand the Problem
Any matching substring must have exactly the pattern's code-point length and frequency signature. Empty pattern returns true; pattern longer than text returns false.

#### In Simple Terms

Return `true` when some continuous piece of `text` uses exactly the same code points and counts as `pattern`, in any order.

### 4. Real-World Analogy
Slide a tray with the same number of letter tiles as the pattern and ask whether its tile inventory ever matches, regardless of order.

### 5. Key Observation
Only pattern-sized windows can be permutations. A `missing` count can track how many required occurrences are still absent without comparing entire maps each time.

### 6. Pattern
```text
Pattern: Fixed-size frequency window
Why this pattern applies: a permutation has fixed length and identical multiplicities.
How to recognize it: determine whether any substring rearranges to a pattern.
Similar problems: find all anagrams, DNA signature matching, inventory windows.
```

### 7. Brute Force
Extract every pattern-sized substring, sort its code points, and compare with the sorted pattern.

#### Pseudocode
```text
signature = sorted pattern
for each pattern-sized window
    if sorted window equals signature
        return true
return false
```

#### Brute-Force TypeScript

```typescript
function containsPermutationBySortingWindows(pattern: string, text: string): boolean {
  const patternCharacters = [...pattern];
  const textCharacters = [...text];
  if (patternCharacters.length === 0) return true;
  if (patternCharacters.length > textCharacters.length) return false;
  const signature = [...patternCharacters].sort().join("");
  for (let start = 0; start + patternCharacters.length <= textCharacters.length; start += 1) {
    const windowSignature = textCharacters
      .slice(start, start + patternCharacters.length)
      .sort()
      .join("");
    if (windowSignature === signature) return true;
  }
  return false;
}
```

#### Visualization
```text
"ei" -> "ei" != "ab"
"id" -> "di" != "ab"
"db" -> "bd" != "ab"
"ba" -> "ab" == "ab"
```

#### Dry Run
The fourth two-character window is `ba`; sorting it produces `ab`, so the answer is true.

#### Brute-Force Complexity
For pattern length `m`, **time is `O((n-m+1)m log m)`** due to sorting each window. **Space is `O(n+m)`** for code-point representations and sorting copies.

### 8. Why Brute Force Is Not Ideal
It rebuilds and sorts overlapping windows rather than updating two boundary contributions.

### 9. Optimal Approach
Count required pattern occurrences. Maintain only counts for required keys in a window of length `m`, adjusting `missing` as useful occurrences enter or leave.

### 10. Visualization
```text
pattern ab -> need a1,b1; missing 2
window ei -> missing 2
window db -> b contributes, missing 1
window ba -> a and b contribute, missing 0 -> true
```

### 11. Step-by-Step Dry Run
| Window | Useful counts | Missing | Match |
|---|---|---:|---|
| `ei` | none | 2 | no |
| `id` | none | 2 | no |
| `db` | b1 | 1 | no |
| `ba` | b1,a1 | 0 | yes |

### 12. TypeScript Solution
```typescript
function containsPermutation(pattern: string, text: string): boolean {
  const patternCharacters = [...pattern];
  const textCharacters = [...text];
  if (patternCharacters.length === 0) return true;
  if (patternCharacters.length > textCharacters.length) return false;

  const needed = new Map<string, number>();
  for (const character of patternCharacters) {
    needed.set(character, (needed.get(character) ?? 0) + 1);
  }

  const window = new Map<string, number>();
  let missing = patternCharacters.length;
  for (let right = 0; right < textCharacters.length; right += 1) {
    const added = textCharacters[right];
    if (needed.has(added)) {
      const count = window.get(added) ?? 0;
      if (count < needed.get(added)!) missing -= 1;
      window.set(added, count + 1);
    }

    if (right >= patternCharacters.length) {
      const removed = textCharacters[right - patternCharacters.length];
      if (needed.has(removed)) {
        const count = window.get(removed)!;
        if (count <= needed.get(removed)!) missing += 1;
        window.set(removed, count - 1);
      }
    }

    if (right + 1 >= patternCharacters.length && missing === 0) return true;
  }
  return false;
}
```

### 13. Code Walkthrough
An entering required character reduces `missing` only while its count is still useful. A leaving required character increases `missing` only when it removes a useful occurrence; window width remains exactly the pattern length before testing.

### 14. Complexity Analysis
**Time: average `O(n+m)`** for conversions, pattern counting, and one text pass. **Space: `O(n+m+u)`** for code-point arrays and maps over `u` pattern keys.

### 15. Edge Cases
- Empty pattern returns true.
- Pattern longer than text returns false.
- Duplicate pattern symbols require matching multiplicity.
- Case variants remain distinct.

### 16. Common Mistakes
- Checking only whether every distinct pattern key is present.
- Letting the window exceed pattern length before evaluating.
- Updating `missing` for extra copies that do not satisfy requirements.
- Sorting every window while claiming linear time.

### 17. Interview Follow-ups
1. Return the first start index. 2. Return all starts. 3. Restrict input to lowercase English for fixed arrays. 4. Stream the text.

### 18. Variations
- Find All Anagrams. - Anagram occurrence count. - Approximate permutation with mismatch budget.

### 19. Pattern Connections
This is the Boolean form of Find All Anagrams: both maintain a pattern-sized frequency signature.

### 20. NestJS Project Connection
Expose `POST /api/dsa/sliding-window/contains-permutation` with `{ pattern, text }`. Return the Boolean and events `{ left, right, added, removed, missing, match }` using code-point indexes.

### 21. Mini Challenge
Return the first code-point start index of a matching permutation, or `-1` when none exists.

## Problem 69 — Find All Anagrams in a String

### 1. Problem Statement
Given lowercase English strings `text` and nonempty `pattern`, return every start index where a pattern-length substring is an anagram of `pattern`.

### 2. Example
```text
Input: text = "cbaebabacd", pattern = "abc"
Output: [0, 6]
```

### 3. Understand the Problem
Indexes are zero-based. Because input is restricted to lowercase ASCII, UTF-16 and code-point indexes coincide. An empty pattern returns no matches, and invalid alphabet input throws `RangeError`.

#### In Simple Terms

Return every position where a pattern-length piece of `text` contains exactly the same lowercase letters and counts as `pattern`.

### 4. Real-World Analogy
Move a tray of fixed width across letter tiles and record every position where its inventory matches the requested inventory.

### 5. Key Observation
Anagrams have equal length and equal counts for all 26 letters. Adjacent windows update exactly one incoming and one outgoing count.

### 6. Pattern
```text
Pattern: Fixed-size frequency-signature window
Why this pattern applies: every candidate has pattern length and must match its counts.
How to recognize it: return every substring position whose symbols rearrange to a pattern.
Similar problems: permutation inclusion, anagram count, fixed inventory matching.
```

### 7. Brute Force
Sort the pattern once, then extract and sort every pattern-length substring before comparing signatures.

#### Pseudocode
```text
target = sorted pattern
for each valid start
    candidate = sorted text[start .. start + pattern length)
    if candidate equals target
        append start
```

#### Brute-Force TypeScript

```typescript
function findAnagramStartsBySortingWindows(text: string, pattern: string): number[] {
  const isLowercaseEnglish = (value: string): boolean => {
    for (const character of value) {
      const codePoint = character.codePointAt(0)!;
      if (codePoint < 97 || codePoint > 122) return false;
    }
    return true;
  };
  if (!isLowercaseEnglish(text) || !isLowercaseEnglish(pattern)) {
    throw new RangeError("text and pattern must contain only lowercase English letters");
  }
  if (pattern.length === 0 || pattern.length > text.length) return [];
  const signature = [...pattern].sort().join("");
  const starts: number[] = [];
  for (let start = 0; start + pattern.length <= text.length; start += 1) {
    const windowSignature = text.slice(start, start + pattern.length).split("").sort().join("");
    if (windowSignature === signature) starts.push(start);
  }
  return starts;
}
```

#### Visualization
```text
cba -> abc, match at 0
bae -> abe, no
...
bac -> abc, match at 6
```

#### Dry Run
In `cbaebabacd`, sorted windows at indexes `0` and `6` equal sorted `abc`; all others differ.

#### Brute-Force Complexity
For `m = pattern.length`, **time is `O((n-m+1)m log m)`** and **space is `O(m+r)`** for sorting copies and `r` returned indexes.

### 8. Why Brute Force Is Not Ideal
It repeatedly copies and sorts windows that differ by only two boundary characters.

### 9. Optimal Approach
Build a 26-count target signature. Roll a second 26-count signature across `text`; after each complete window, compare the fixed-size arrays and record matching starts.

### 10. Visualization
```text
target abc: [a1,b1,c1]
window cba: [a1,b1,c1] -> record 0
slide: remove c, add e -> signatures differ
```

### 11. Step-by-Step Dry Run
| Start | Window | Signature matches | Output |
|---:|---|---|---|
| 0 | `cba` | yes | `[0]` |
| 1 | `bae` | no | `[0]` |
| 2 | `aeb` | no | `[0]` |
| 6 | `bac` | yes | `[0,6]` |

### 12. TypeScript Solution
```typescript
function findAnagramStarts(text: string, pattern: string): number[] {
  const isLowercaseEnglish = (value: string): boolean => {
    for (const character of value) {
      const codePoint = character.codePointAt(0)!;
      if (codePoint < 97 || codePoint > 122) return false;
    }
    return true;
  };
  if (!isLowercaseEnglish(text) || !isLowercaseEnglish(pattern)) {
    throw new RangeError("text and pattern must contain only lowercase English letters");
  }
  if (pattern.length === 0 || pattern.length > text.length) return [];

  const needed = Array<number>(26).fill(0);
  const window = Array<number>(26).fill(0);
  const alphabetIndex = (character: string): number => character.charCodeAt(0) - 97;
  for (const character of pattern) needed[alphabetIndex(character)] += 1;

  const starts: number[] = [];
  for (let right = 0; right < text.length; right += 1) {
    window[alphabetIndex(text[right])] += 1;
    if (right >= pattern.length) {
      window[alphabetIndex(text[right - pattern.length])] -= 1;
    }
    if (right + 1 >= pattern.length &&
        window.every((count, index) => count === needed[index])) {
      starts.push(right - pattern.length + 1);
    }
  }
  return starts;
}
```

### 13. Code Walkthrough
Alphabet validation makes constant-size array indexes safe. Comparing 26 entries per complete window is constant work under the fixed lowercase-English contract.

### 14. Complexity Analysis
**Time: `O(n+m+26n) = O(n+m)`** because alphabet size is fixed. **Space: `O(r)`** for `r` output indexes; the two 26-element arrays are `O(1)` auxiliary space.

### 15. Edge Cases
- Empty pattern or pattern longer than text returns empty.
- Repeated letters require equal counts.
- Overlapping matches are returned.
- Uppercase or non-English input throws.

### 16. Common Mistakes
- Checking only whether each pattern letter appears.
- Forgetting overlapping matches.
- Returning substring values instead of start indexes.
- Calling a 26-entry comparison constant without stating the alphabet restriction.

### 17. Interview Follow-ups
1. Use a mismatch counter instead of comparing 26 entries. 2. Support arbitrary Unicode. 3. Stream text chunks. 4. Return matched substrings.

### 18. Variations
- Permutation in String. - Anagram occurrence count. - Approximate anagrams with bounded differences.

### 19. Pattern Connections
This extends Problem 68 from early Boolean success to collecting every matching fixed window.

### 20. NestJS Project Connection
Expose `POST /api/dsa/sliding-window/anagram-starts` with `{ text, pattern }`. Validate lowercase input and return starts plus bounded signature-delta and match events.

### 21. Mini Challenge
Replace the 26-entry comparison with a maintained mismatch count while preserving all overlapping matches.

## Problem 70 — Minimum Size Subarray Sum

### 1. Problem Statement
Given a positive safe-integer target and an array of positive safe integers, return the minimum length of a contiguous subarray whose sum is at least the target, or `0` if none exists.

### 2. Example
```text
Input: target = 7, values = [2, 3, 1, 2, 4, 3]
Output: 2
```

### 3. Understand the Problem
All values are strictly positive, which is essential to the shrinking proof. Running sums must remain safe integers; the input is not mutated.

#### In Simple Terms

Among all continuous blocks of positive values whose sum reaches the target, return the smallest block length.

### 4. Real-World Analogy
Find the shortest consecutive run of daily deposits that reaches a savings goal; every additional day increases the total.

### 5. Key Observation
With positive values, expanding right only increases the sum. Once the target is met, removing left values is the only way to search for a shorter valid range.

### 6. Pattern
```text
Pattern: Minimum variable window with a monotonic sum
Why this pattern applies: positive values make sum changes predictable at both boundaries.
How to recognize it: shortest contiguous positive-value range reaching a threshold.
Similar problems: minimum covering window, shortest subarray with positive costs.
```

### 7. Brute Force
Start at each index, accumulate to the right, and stop at the first range reaching the target because further positive values only increase its length.

#### Pseudocode
```text
best = infinity
for each start
    sum = 0
    for end from start onward
        sum += values[end]
        if sum >= target
            best = min(best, end - start + 1)
            break
return 0 if no best else best
```

#### Brute-Force TypeScript

```typescript
function minimumPositiveSubarrayLengthByEveryStart(target: number, values: number[]): number {
  let best = Number.POSITIVE_INFINITY;
  for (let start = 0; start < values.length; start += 1) {
    let sum = 0;
    for (let end = start; end < values.length; end += 1) {
      sum += values[end];
      if (sum >= target) {
        best = Math.min(best, end - start + 1);
        break;
      }
    }
  }
  return Number.isFinite(best) ? best : 0;
}
```

#### Visualization
```text
start 0: 2 -> 5 -> 6 -> 8, length 4
start 1: 3 -> 4 -> 6 -> 10, length 4
...
start 4: 4 -> 7, length 2
```

#### Dry Run
Several starts reach the target with length four or three; starting at value `4` reaches `7` after adding `3`, giving length two.

#### Brute-Force Complexity
**Time: `O(n²)`** in the worst case when many starts scan most of the suffix. **Space: `O(1)`**.

### 8. Why Brute Force Is Not Ideal
It recomputes overlapping positive sums instead of retaining the current range total.

### 9. Optimal Approach
Expand right and add each value. While the sum meets the target, record the current length, subtract the left value, and advance left to test shorter ranges.

### 10. Visualization
```text
[2,3,1,2] sum 8 -> record 4, remove 2 -> sum 6
  [3,1,2,4] sum 10 -> record 4, remove 3 -> sum 7
    [1,2,4] sum 7 -> record 3, remove 1
...
        [4,3] sum 7 -> record 2
```

### 11. Step-by-Step Dry Run
| Right added | Sum before shrinking | Valid lengths considered | Best |
|---:|---:|---|---:|
| 2 | 2 | none | infinity |
| 3 | 5 | none | infinity |
| 1 | 6 | none | infinity |
| 2 | 8 | 4 | 4 |
| 4 | 10 | 4, 3 | 3 |
| 3 | 9 | 3, 2 | 2 |

### 12. TypeScript Solution
```typescript
function minimumPositiveSubarrayLength(target: number, values: number[]): number {
  let left = 0;
  let sum = 0;
  let best = Number.POSITIVE_INFINITY;
  for (let right = 0; right < values.length; right += 1) {
    sum += values[right];
    while (sum >= target) {
      best = Math.min(best, right - left + 1);
      sum -= values[left];
      left += 1;
    }
  }
  return Number.isFinite(best) ? best : 0;
}
```

### 13. Code Walkthrough
The sum always represents `[left,right]`. The inner loop enumerates every valid left boundary for the current right endpoint, stopping exactly when positive-value removal makes the range invalid.

### 14. Complexity Analysis
**Time: `O(n)`** because each value is added once and removed at most once. **Space: `O(1)`** because the algorithm stores only boundaries, sum, and best length.

### 15. Edge Cases
- Empty or insufficient arrays return zero.
- One value meeting the target returns one.
- Target and every value must be positive safe integers.
- Negative or zero values violate this contract and require reconsidering the proof.

### 16. Common Mistakes
- Applying this shrink rule when negative values are allowed.
- Shrinking only once instead of while valid.
- Recording after removal and missing the current valid range.
- Returning infinity rather than zero when no window exists.

### 17. Interview Follow-ups
1. Allow negative values using prefix sums and a monotonic deque. 2. Return boundaries. 3. Count qualifying subarrays. 4. Support a data stream.

### 18. Variations
- Minimum length with sum greater than target. - Shortest Subarray With Sum at Least K. - Minimum product threshold for positive values.

### 19. Pattern Connections
Like Minimum Window Substring, this shrinks while valid; here scalar sum replaces multiset coverage and positivity supplies monotonicity.

### 20. NestJS Project Connection
Use `POST /api/dsa/sliding-window/minimum-size-sum` with `{ target, values }`. Validate positive safe integers and return length with events `{ left, right, added, removed, sum, best }`.

### 21. Mini Challenge
Return the earliest shortest range boundaries and explain why the same algorithm fails after adding one negative value.

---

# Topic Summary

## What You Learned

- Sliding windows reuse state across overlapping contiguous ranges.
- Fixed windows remove one outgoing contribution and add one incoming contribution.
- Variable windows need an explicit validity or coverage invariant.
- Longest-valid problems shrink until valid, while shortest-covering problems shrink while valid.
- Frequency maps preserve multiplicity; sets alone are insufficient for anagrams and coverage.
- Both boundaries can move inside nested loops while total work remains linear.
- Some sum windows require positive values so boundary movement changes sums monotonically.
- Code-point windows need clearly labeled indexes and may allocate code-point arrays.
- Compact trace deltas preserve the algorithm's complexity better than full snapshots.

## Patterns You Should Recognize

- exactly `k` contiguous items -> fixed rolling aggregate
- longest range under an at-most rule -> expand, shrink while invalid, maximize
- shortest range covering requirements -> expand to valid, shrink while valid, minimize
- no repeated characters -> last-seen boundary jump
- uniform range after replacements -> dominant-frequency budget
- permutation or anagram -> pattern-sized frequency signature
- positive-sum threshold -> monotonic variable window

## Complexity Cheat Sheet

| Operation or pattern | Time | Extra space |
|---|---:|---:|
| Recompute each size-`k` window | `O(nk)` | Often `O(1)` |
| Fixed rolling sum/count | `O(n)` | `O(1)` |
| Last-seen unique window | Average `O(n)` | `O(u)` plus representation |
| Frequency-map variable window | Average `O(n)` | `O(u)` plus representation |
| Fixed 26-letter signature comparison | `O(26n) = O(n)` | `O(1)` plus output |
| Minimum covering window | Average `O(n+m)` | `O(u)` plus representations/output |
| Positive minimum-sum window | `O(n)` | `O(1)` |

## Common Interview Traps

- Using a sliding window for a noncontiguous subsequence problem.
- Leaving invalid `k`, empty pattern, case, alphabet, or Unicode behavior undefined.
- Recomputing a sum or frequency table for every overlapping range.
- Updating state after moving a boundary instead of for the value actually entering or leaving.
- Recording a longest window before restoring validity.
- Stopping after one shrink when several removals are needed.
- Treating target coverage as membership and ignoring duplicate requirements.
- Assuming a stale maximum frequency is the exact current maximum in Problem 66.
- Claiming constant space while returning indexes or materializing code points.
- Applying positive-sum shrinking when negative values are allowed.

## Practice Checklist

- [ ] Problem 61 — Maximum Sum Subarray of Size K
- [ ] Problem 62 — Average of All Subarrays of Size K
- [ ] Problem 63 — Maximum Number of Vowels in a Substring of Size K
- [ ] Problem 64 — Longest Substring Without Repeating Characters
- [ ] Problem 65 — Longest Substring With at Most K Distinct Characters
- [ ] Problem 66 — Longest Repeating Character Replacement
- [ ] Problem 67 — Minimum Window Substring
- [ ] Problem 68 — Permutation in String
- [ ] Problem 69 — Find All Anagrams in a String
- [ ] Problem 70 — Minimum Size Subarray Sum