# Binary Search

## Learning Objectives

By the end of this topic, you should understand:

- why binary search requires ordered data or a monotonic decision rule
- how to maintain closed and half-open search intervals
- how to prove that each iteration discards a valid half of the search space
- how lower-bound and upper-bound searches handle duplicates
- how to derive first occurrence, last occurrence, counts, floor, and ceiling
- how rotated arrays preserve enough local order for logarithmic search
- how neighboring values guide peak search without global sorting
- how to binary-search a numeric answer using a feasibility predicate
- how to choose safe integer bounds and midpoint formulas
- how partition search finds the median of two sorted arrays
- why validating a sorted precondition can cost more than the search itself

## Prerequisites

- contracts, loop invariants, and logarithmic complexity from `00-dsa-fundamentals.md`
- sorted arrays and index-based traversal from `01-arrays.md`
- two-pointer interval reasoning from `04-two-pointers.md`
- prefix sums and safe accumulation from `06-prefix-sum-kadane.md`
- TypeScript arrays, loops, functions, integer arithmetic, and tuple types

## Core Theory

### What binary search really needs

Binary search does not merely need an array. It needs a search space where one comparison or predicate divides candidates into two regions and proves one region cannot contain the answer.

For an ascending array:

```text
value < target  -> discard this index and everything left of it
value > target  -> discard this index and everything right of it
```

For answer-space problems, the same idea appears as a monotonic predicate:

```text
false false false true true true
                  ^ first feasible answer
```

### Closed interval invariant

Many examples in this chapter use a closed candidate interval `[low, high]`.

```text
initially: low = 0, high = n - 1
continue:  while low <= high
midpoint:  floor(low + (high - low) / 2)
```

When `values[mid]` is too small, set `low = mid + 1`. When it is too large, set `high = mid - 1`. The `+1` and `-1` guarantee progress because `mid` has already been ruled out.

### Half-open interval invariant

Boundary searches often use `[low, high)`, where `high` is excluded.

```text
initially: low = 0, high = n
continue:  while low < high
```

The interval can represent an insertion boundary from `0` through `n`, which makes lower-bound searches natural. Closed and half-open forms are both correct; mixing their update rules is not.

### Safe midpoint calculation

Use:

```text
mid = low + floor((high - low) / 2)
```

instead of mathematically writing `(low + high) / 2`. JavaScript array indexes cannot approach the safe-integer limit in practice, but the safer formula states the transferable invariant and avoids overflow in fixed-width languages.

### Why the time is logarithmic

Each iteration keeps at most half of the candidates:

```text
n
n / 2
n / 4
n / 8
...
1
```

After `k` halvings, about `n / 2^k` candidates remain. Setting that value to `1` gives `k = log2(n)`, so time is `O(log n)`.

### Exact search versus boundary search

An exact search may return immediately when it finds a target. With duplicates, that matching index is not necessarily the first or last.

A boundary search records or converges on a match, then continues toward the required edge:

- lower bound: first index whose value is at least the target
- upper bound: first index whose value is greater than the target
- first occurrence: lower bound, if it equals the target
- last occurrence: upper bound minus one, if it equals the target
- count: `upperBound - lowerBound`

### Floor and ceiling

For a sorted numeric array:

- floor is the greatest value less than or equal to the target
- ceiling is the smallest value greater than or equal to the target

These are values, not indexes, unless the contract says otherwise. A missing side needs an explicit representation such as `null`.

### Rotated sorted arrays

A strictly increasing array rotated once has two sorted portions:

```text
[4, 5, 6, 7, 0, 1, 2]
 ^ sorted left ^  ^ sorted right ^
```

At least one side around `mid` is sorted. Determine which side, check whether the target lies inside its value range, and discard the other side. Duplicate values can hide which side is sorted, so problems in this chapter state when distinct values are required.

### Peak search

A peak is greater than its immediate neighbors, with outside boundaries treated as negative infinity. If `values[mid] < values[mid + 1]`, an ascent exists to the right and some peak must lie there. Otherwise, a peak exists at `mid` or to its left.

This is binary search on a directional property, not on globally sorted values.

### Binary search on the answer

Some problems ask for the smallest capacity, speed, or maximum allocation that satisfies a constraint. Candidate answers are numeric and feasibility is monotonic:

```text
capacity too small -> infeasible
larger capacity    -> may become feasible and stays feasible
```

Search the smallest feasible value. Bounds must be proven:

- lower bound: no valid answer can be smaller
- upper bound: definitely feasible

The feasibility check usually scans all inputs, so total time is `O(n log R)`, where `R` is the numeric answer range.

### Partition search for two sorted arrays

To find a median without merging, partition both arrays so the left side contains half the combined elements. Binary-search the partition in the shorter array; the other partition follows from the required left-side size.

A partition is valid when:

```text
leftA <= rightB and leftB <= rightA
```

Then the largest left value and smallest right value determine the median.

### Preconditions and validation cost

Checking that an entire array is sorted costs `O(n)`. A standalone function that performs that check cannot honestly claim total `O(log n)` time.

Therefore, logarithmic search solutions in this chapter document sortedness and element shape as caller guarantees. A NestJS boundary may validate once when data is ingested, cache trusted sorted data, or report validation separately from search time.

## Visual Mental Model

### Eliminate half

```text
target = 13

[1, 4, 7, 9, 13, 18, 22]
 low      mid           high
9 < 13 -> discard indexes 0..3

[13, 18, 22]
 low mid high
```

### Search for a boundary

```text
[1, 2, 2, 2, 5]
    ^ first 2
          ^ first value greater than 2
```

### Search a monotonic answer space

```text
candidate:   5   6   7   8   9  10
feasible:    F   F   F   T   T   T
                         ^ smallest feasible
```

### Partition two arrays

```text
A: [1, 3 | 8]
B: [2, 7 | 9, 10]

left side:  1, 3, 2, 7
right side: 8, 9, 10
```

Move the partition until every left value is no greater than every right value.

## Complexity Concepts

### Search cost versus validation cost

Binary search is `O(log n)` only when its ordered or monotonic precondition is already established. Validating `n` elements inside the call makes end-to-end time `O(n)`.

### Boundary searches stay logarithmic

Continuing after a match does not scan duplicates. It still halves the remaining interval, so first occurrence, last occurrence, floor, and ceiling remain `O(log n)`.

### Answer-range complexity

If candidates range from `low` through `high`, binary search performs `O(log(high - low + 1))` feasibility checks. If each check costs `O(n)`, total time is:

```text
O(n log(high - low + 1))
```

This depends on the magnitude of candidate values, not just the number of inputs.

### Recursive versus iterative space

Iterative binary search uses `O(1)` auxiliary space. A recursive version has `O(log n)` call-stack space even though its time remains `O(log n)`.

### Numeric safety

Answer-space problems sum loads, hours, or pages. Safe integer inputs can still produce unsafe totals. Bounds, ceiling division, feasibility accumulators, and final arithmetic must preserve the stated numeric contract.

## Common Patterns

### Exact match

```text
while low <= high:
    mid = safe midpoint
    if value equals target: return mid
    discard the impossible half
return not found
```

### Lower bound

```text
while low < high:
    mid = midpoint in [low, high)
    if values[mid] < target: low = mid + 1
    else: high = mid
return low
```

### Candidate answer

```text
while low < high:
    mid = lower midpoint
    if feasible(mid): high = mid
    else: low = mid + 1
return low
```

### Rotated-array side detection

```text
if left through mid is sorted:
    decide whether target lies there
else:
    right side is sorted; decide whether target lies there
```

### Neighbor-guided convergence

```text
if values[mid] < values[mid + 1]:
    move right
else:
    keep mid and move left
```

# Problems

## Problem 81 — Binary Search

### 1. Problem Statement
Given an array of safe integers sorted in nondecreasing order and a safe-integer target, return an index containing the target or `-1` if absent. If duplicates exist, any matching index is valid. The caller guarantees the array contract; the function does not mutate it.

### 2. Example
```text
Input:  values = [-1, 0, 3, 5, 9, 12], target = 9
Output: 4
```

### 3. Understand the Problem
The ascending order is the reason half the remaining indexes can be discarded. This exact-match contract does not promise the first or last duplicate.

#### In Simple Terms
Find the target in an already sorted list. Check the middle number: if it is too small, continue only on the right; if it is too large, continue only on the left. Return a matching position or `-1`.

### 4. Real-World Analogy
To find a name in an alphabetized directory, open near the middle and decide which half can still contain that name.

### 5. Key Observation
Comparing the middle value with the target proves that one entire ordered half cannot contain the target.

### 6. Pattern
```text
Pattern: Exact-Match Binary Search
Why this pattern applies: sorted order makes one half impossible after each comparison
How to recognize it: find a target in ordered random-access data
Similar problems: dictionary lookup, first occurrence, rotated-array search
```

### 7. Brute Force
Scan every value from left to right and return the first matching index. Although this chooses the first duplicate, it still satisfies the any-match contract.

#### Pseudocode
```text
validate target
for index from 0 through n - 1:
    if values[index] equals target:
        return index
return -1
```

#### Brute-Force TypeScript
```typescript
function binarySearchBruteForce(values: readonly number[], target: number): number {
  if (!Number.isSafeInteger(target)) {
    throw new RangeError("target must be a safe integer");
  }

  for (let index = 0; index < values.length; index += 1) {
    if (values[index] === target) return index;
  }
  return -1;
}
```

#### Visualization
```text
[-1, 0, 3, 5, 9, 12]
  ^  ^  ^  ^  ^   ^
check each until index 4
```

#### Dry Run
The scan compares `9` with `-1`, `0`, `3`, `5`, and then `9`, returning index `4` after five comparisons.

#### Brute-Force Complexity
**Time: `O(n)`** in the worst case. **Space: `O(1)`** auxiliary space.

### 8. Why Brute Force Is Not Ideal
It ignores sorted order. A failed comparison can rule out many values at once rather than only the current index.

### 9. Optimal Approach
Maintain a closed candidate interval `[low, high]`. Compare the target with its midpoint. Return on equality; otherwise remove the midpoint and the half whose values are guaranteed too small or too large.

Invariant:

```text
if the target exists and has not been returned, some matching index lies in [low, high]
```

### 10. Visualization
```text
[-1, 0, 3, 5, 9, 12], target 9
 low     mid          high    values[2] = 3 < 9
             low mid high    values[4] = 9 -> found
```

Candidate counts shrink approximately as:

```text
6 -> 3 -> found
```

### 11. Step-by-Step Dry Run
| Low | High | Mid | Value | Decision |
|---:|---:|---:|---:|---|
| 0 | 5 | 2 | 3 | target is right; `low = 3` |
| 3 | 5 | 4 | 9 | return `4` |

### 12. TypeScript Solution
```typescript
function binarySearch(values: readonly number[], target: number): number {
  if (!Number.isSafeInteger(target)) {
    throw new RangeError("target must be a safe integer");
  }

  let low = 0;
  let high = values.length - 1;

  while (low <= high) {
    const middle = low + Math.floor((high - low) / 2);
    const value = values[middle];

    if (value === target) return middle;
    if (value < target) {
      low = middle + 1;
    } else {
      high = middle - 1;
    }
  }

  return -1;
}
```

### 13. Code Walkthrough
`low` and `high` include every remaining candidate. The midpoint formula stays within that interval. Both mismatch branches exclude `middle`, guaranteeing progress and termination.

### 14. Complexity Analysis
**Search time: `O(log n)`** because each iteration keeps at most half the candidate indexes. **Space: `O(1)`**. Validating the sorted-array precondition separately would cost `O(n)`.

### 15. Edge Cases
- Empty input returns `-1`.
- One value is found or rejected in one comparison.
- Duplicates may return any matching index.
- Targets below the minimum or above the maximum return `-1`.
- An unsafe target throws `RangeError`.

### 16. Common Mistakes
- Using binary search on unsorted input.
- Writing `low < high` with closed-interval updates and missing one candidate.
- Updating to `low = middle` or `high = middle` and failing to progress.
- Promising the first duplicate from an exact-match implementation.

### 17. Interview Follow-ups
1. Return the first occurrence. 2. Return the insertion position. 3. Write a recursive version. 4. Search descending input.

### 18. Variations
- Lower bound. - Upper bound. - Rotated-array search. - Binary search on a monotonic predicate.

### 19. Pattern Connections
Problems 82–86 convert exact search into boundary search. Later problems apply the same interval invariant to rotated data and numeric answer spaces.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/exact` with `{ values, target }`. Validate sorted safe-integer data at the API boundary, report validation cost separately, and return `{ index, trace, truncated }` with bounded optimized events for `low`, `high`, `middle`, comparison, and discarded interval.

### 21. Mini Challenge
Rewrite the function with a half-open interval `[low, high)` and explain how its loop condition and updates differ.

## Problem 82 — Search Insert Position

### 1. Problem Statement
Given a `readonly` array of safe integers sorted in nondecreasing order and a safe-integer target, return the first index whose value is greater than or equal to the target. If no such value exists, return `values.length`. The result is always in `[0, n]`, and duplicates place the target before all equal values. Sortedness and safe-integer elements are caller preconditions; the function validates only the target in `O(1)` and never mutates the array.

### 2. Example
```text
Input:  values = [1, 3, 5, 6], target = 5
Output: 2

Input:  values = [1, 3, 5, 6], target = 2
Output: 1
```

### 3. Understand the Problem
The answer may be an existing index or the position immediately after the final element. This is the lower-bound boundary: every index before the answer contains a value smaller than the target, while the answer is the first index that could hold the target without breaking order.

#### In Simple Terms
Find where the target belongs in an already sorted list. If equal values exist, return the position before the first equal value; if the target is larger than everything, return the list length.

### 4. Real-World Analogy
When filing a card alphabetically, move past every card that belongs earlier, then insert before the first card that is equal or later.

### 5. Key Observation
For any midpoint, `values[middle] < target` proves that the insertion point is to its right. Otherwise, `middle` might be the first valid position, so it must remain a candidate.

### 6. Pattern
```text
Pattern: Lower Bound with a Half-Open Interval
Why this pattern applies: the answer separates values < target from values >= target
How to recognize it: find the first valid index or an insertion position in sorted data
Similar problems: first occurrence, count occurrences, ceiling
```

### 7. Brute Force
The direct intuition is to scan from left to right. The first value that is not smaller than the target is exactly the insertion boundary. If the scan finishes, the target belongs at index `n`.

#### Pseudocode
```text
validate target
for index from 0 through n - 1:
    if values[index] >= target:
        return index
return n
```

#### Brute-Force TypeScript
```typescript
function searchInsertBruteForce(
  values: readonly number[],
  target: number,
): number {
  if (!Number.isSafeInteger(target)) {
    throw new RangeError("target must be a safe integer");
  }

  for (let index = 0; index < values.length; index += 1) {
    if (values[index] >= target) return index;
  }
  return values.length;
}
```

#### Visualization
```text
values = [1, 3, 5, 6], target = 2
          <  ^
             first value >= 2, so answer = 1
```

#### Dry Run
| Index | Value | Is value >= 2? | Action |
|---:|---:|:---:|---|
| 0 | 1 | No | Continue |
| 1 | 3 | Yes | Return `1` |

#### Brute-Force Complexity
**Time: `O(n)`** because the target may belong after all `n` values, requiring one comparison per element. **Space: `O(1)`** because the scan stores only its index.

### 8. Why Brute Force Is Not Ideal
The scan advances by one index even though sorted order can rule out an entire half after one comparison.

### 9. Optimal Approach
Use the half-open candidate interval `[low, high)`, initialized to `[0, n)`. It can converge to `n`, which naturally represents insertion after the array.

Maintain this invariant:

```text
indexes before low contain values < target
indexes at or after high contain values >= target
the first valid insertion index is in [low, high]
```

If the midpoint value is smaller, discard it and everything left by setting `low = middle + 1`. Otherwise, preserve the midpoint as a possible answer with `high = middle`.

### 10. Visualization
```text
[1, 3, 5, 6], target = 2
 low   mid     high(exclusive)   5 >= 2 -> high = 2
 low mid high(exclusive)         3 >= 2 -> high = 1
 low/high = 1                    boundary found

regions: [values < 2 | values >= 2]
          [    1      | 3, 5, 6 ]
                       ^ index 1
```

### 11. Step-by-Step Dry Run
| Low | High (exclusive) | Middle | Value | Decision |
|---:|---:|---:|---:|---|
| 0 | 4 | 2 | 5 | Could be answer; `high = 2` |
| 0 | 2 | 1 | 3 | Could be answer; `high = 1` |
| 0 | 1 | 0 | 1 | Too small; `low = 1` |
| 1 | 1 | — | — | Return `1` |

### 12. TypeScript Solution
```typescript
function searchInsert(values: readonly number[], target: number): number {
  if (!Number.isSafeInteger(target)) {
    throw new RangeError("target must be a safe integer");
  }

  let low = 0;
  let high = values.length;

  while (low < high) {
    const middle = low + Math.floor((high - low) / 2);

    if (values[middle] < target) {
      low = middle + 1;
    } else {
      high = middle;
    }
  }

  return low;
}
```

### 13. Code Walkthrough
`high` starts at `values.length`, so the answer can be one past the final index. The smaller-than branch excludes `middle`; the other branch keeps it. When `low === high`, the unknown region is empty and `low` is the first index whose value is at least the target.

### 14. Complexity Analysis
**Search time: `O(log n)`**. The candidate width starts at `n` and changes as `n -> n / 2 -> n / 4 -> ... -> 0`, so only logarithmically many comparisons occur. **Space: `O(1)`**. Target validation is `O(1)`. Scanning to validate sortedness would make total time `O(n)`, so sortedness remains a documented caller precondition.

### 15. Edge Cases
- Empty input returns `0`.
- A target below all values returns `0`.
- A target above all values returns `n`.
- Duplicate targets return the first equal index.
- An unsafe target throws `RangeError`.

### 16. Common Mistakes
- Returning any equal index instead of continuing left.
- Initializing half-open `high` to `n - 1`.
- Using `high = middle - 1`, which belongs to a closed-interval design.
- Validating sortedness inside the function while claiming end-to-end `O(log n)`.

### 17. Interview Follow-ups
1. Return the first value strictly greater than the target. 2. Insert without mutating the original array. 3. Adapt the search to descending data. 4. Explain why the result can equal `n`.

### 18. Variations
- Upper-bound insertion after duplicates. - Generic lower bound with a comparator. - Lower bound in a conceptual numeric answer space.

### 19. Pattern Connections
The same lower-bound boundary gives the first occurrence in Problem 83, contributes to the count in Problem 85, and identifies a ceiling in Problem 86.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/search-insert` with `{ values, target }`. The API can validate and establish the sorted safe-integer dataset before timing the search, then return `{ index, trace, truncated }`. Trace only optimized iterations with `low`, exclusive `high`, `middle`, comparison, and the retained interval; do not trace the brute scan as the algorithm execution.

### 21. Mini Challenge
Change the contract so duplicates insert after all equal values. Which single comparison in the optimized loop must change?

## Problem 83 — Find First Occurrence

### 1. Problem Statement
Given a `readonly` array of safe integers sorted in nondecreasing order and a safe-integer target, return the index of the target's first occurrence or `-1` when absent. The caller guarantees sorted safe-integer elements; the function performs only `O(1)` target validation and does not mutate the array.

### 2. Example
```text
Input:  values = [1, 2, 2, 2, 5], target = 2
Output: 1
```

### 3. Understand the Problem
A normal exact binary search may stop at any matching duplicate. This problem asks for the leftmost match, so finding equality is not enough: we must establish that no earlier equal value exists.

#### In Simple Terms
Find the target in a sorted list, but when it appears more than once, return the position of its first copy. Return `-1` if it is missing.

### 4. Real-World Analogy
In an alphabetized attendance list with repeated family names, you need the first person in that surname block, not any person in the middle.

### 5. Key Observation
The first occurrence is the lower-bound index if, and only if, the value at that index equals the target.

### 6. Pattern
```text
Pattern: Lower Bound plus Equality Check
Why this pattern applies: duplicates form one contiguous block in sorted data
How to recognize it: the request says first, leftmost, or earliest equal value
Similar problems: search insert position, count occurrences, first true predicate
```

### 7. Brute Force
The intuition is simple: a left-to-right scan encounters duplicate positions in increasing order, so the first match seen is the required answer.

#### Pseudocode
```text
validate target
for index from 0 through n - 1:
    if values[index] equals target:
        return index
return -1
```

#### Brute-Force TypeScript
```typescript
function findFirstOccurrenceBruteForce(
  values: readonly number[],
  target: number,
): number {
  if (!Number.isSafeInteger(target)) {
    throw new RangeError("target must be a safe integer");
  }

  for (let index = 0; index < values.length; index += 1) {
    if (values[index] === target) return index;
  }
  return -1;
}
```

#### Visualization
```text
[1, 2, 2, 2, 5], target = 2
 ^  ^
 0  first match -> return 1
```

#### Dry Run
| Index | Value | Equals 2? | Action |
|---:|---:|:---:|---|
| 0 | 1 | No | Continue |
| 1 | 2 | Yes | Return `1` |

#### Brute-Force Complexity
**Time: `O(n)`** in the worst case because an absent target or a match near the end requires scanning all `n` values. **Space: `O(1)`** because no collection grows with input size.

### 8. Why Brute Force Is Not Ideal
It checks positions one at a time and does not use the sorted order to skip impossible prefixes or suffixes.

### 9. Optimal Approach
First find the lower bound: the first index whose value is at least the target. Use `[low, high)` so an absent target greater than every value can converge to `n`. Then check whether the boundary is inside the array and actually equals the target.

Invariant:

```text
indexes before low are known to contain values < target
indexes at or after high are known to contain values >= target
the lower-bound position remains in [low, high]
```

This continues left even after equality because equality belongs in the `>= target` region.

### 10. Visualization
```text
[1, 2, 2, 2, 5], target = 2
 low   mid        high(exclusive)  value 2 -> high = 2
 low mid high(exclusive)           value 2 -> high = 1
 low/high = 1

[1 | 2, 2, 2, 5]
     ^ lower bound equals target -> first occurrence
```

### 11. Step-by-Step Dry Run
| Low | High (exclusive) | Middle | Value | Decision |
|---:|---:|---:|---:|---|
| 0 | 5 | 2 | 2 | Keep left boundary; `high = 2` |
| 0 | 2 | 1 | 2 | Keep left boundary; `high = 1` |
| 0 | 1 | 0 | 1 | Too small; `low = 1` |
| 1 | 1 | — | 2 | Equality check passes; return `1` |

### 12. TypeScript Solution
```typescript
function findFirstOccurrence(
  values: readonly number[],
  target: number,
): number {
  if (!Number.isSafeInteger(target)) {
    throw new RangeError("target must be a safe integer");
  }

  let low = 0;
  let high = values.length;

  while (low < high) {
    const middle = low + Math.floor((high - low) / 2);

    if (values[middle] < target) {
      low = middle + 1;
    } else {
      high = middle;
    }
  }

  return low < values.length && values[low] === target ? low : -1;
}
```

### 13. Code Walkthrough
The loop computes the lower-bound position without returning early on equality. `high = middle` keeps an equal midpoint available while searching farther left. The final guard prevents reading beyond the array and distinguishes a true match from the first greater value.

### 14. Complexity Analysis
**Search time: `O(log n)`** because each iteration halves the unresolved interval; the final equality check is `O(1)`. **Space: `O(1)`** because only interval indexes are stored. The array is not scanned for sortedness, since that separate validation would cost `O(n)` and dominate the search.

### 15. Edge Cases
- Empty input returns `-1`.
- All values equal to the target returns `0`.
- A target below or above the range returns `-1`.
- One matching value returns its index.
- An unsafe target throws `RangeError`.

### 16. Common Mistakes
- Returning immediately when an equal midpoint is found.
- Forgetting the final equality check when lower bound points to a greater value.
- Reading `values[n]` without checking the boundary.
- Mixing closed-interval and half-open updates.

### 17. Interview Follow-ups
1. Return the last occurrence. 2. Return the full duplicate range. 3. Count comparisons. 4. Generalize to objects sorted by a key.

### 18. Variations
- Candidate-and-left closed search. - First value satisfying a predicate. - First timestamp at or after a requested time.

### 19. Pattern Connections
This is Search Insert Position plus a final equality check. Paired with an upper bound, it also solves Count Occurrences without scanning the duplicate block.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/first-occurrence` with `{ values, target }`. Establish the sorted-data contract at ingestion or API validation and account for that work separately. Return `{ index, trace, truncated }`, where each optimized trace event records `low`, exclusive `high`, `middle`, whether the midpoint joined the `< target` or `>= target` region, and the final equality check.

### 21. Mini Challenge
Implement the same contract with a closed interval and a remembered `answer` candidate. What must happen after finding equality?

## Problem 84 — Find Last Occurrence

### 1. Problem Statement
Given a `readonly` array of safe integers sorted in nondecreasing order and a safe-integer target, return the index of the target's last occurrence or `-1` when absent. Sortedness and safe-integer elements are caller preconditions; only the scalar target is validated in `O(1)`, and the array is not mutated.

### 2. Example
```text
Input:  values = [1, 2, 2, 2, 5], target = 2
Output: 3
```

### 3. Understand the Problem
Duplicate targets occupy consecutive indexes. We need the rightmost member of that block, so an equal midpoint is a valid candidate but not necessarily the final answer.

#### In Simple Terms
Find the target's final position in a sorted list. Keep looking to the right after a match, and return `-1` when the target never appears.

### 4. Real-World Analogy
In a date-sorted activity log, finding one event from Monday does not identify the last Monday event; continue toward later entries until the date changes.

### 5. Key Observation
When the midpoint equals the target, remember it and discard only the midpoint and everything left. Any better answer must be farther right.

### 6. Pattern
```text
Pattern: Candidate-and-Right Binary Search
Why this pattern applies: equal values form a contiguous block and the right edge is required
How to recognize it: the request says last, rightmost, or latest equal value
Similar problems: upper bound, count occurrences, predecessor search
```

### 7. Brute Force
The baseline intuition is to inspect the entire array and replace the remembered answer whenever the target appears. A full scan is necessary for this brute approach because a later duplicate may exist.

#### Pseudocode
```text
validate target
answer = -1
for index from 0 through n - 1:
    if values[index] equals target:
        answer = index
return answer
```

#### Brute-Force TypeScript
```typescript
function findLastOccurrenceBruteForce(
  values: readonly number[],
  target: number,
): number {
  if (!Number.isSafeInteger(target)) {
    throw new RangeError("target must be a safe integer");
  }

  let answer = -1;
  for (let index = 0; index < values.length; index += 1) {
    if (values[index] === target) answer = index;
  }
  return answer;
}
```

#### Visualization
```text
[1, 2, 2, 2, 5], target = 2
    ^  ^  ^
    1  2  3 <- answer is overwritten through the final match
```

#### Dry Run
| Index | Value | Equals 2? | Remembered answer |
|---:|---:|:---:|---:|
| 0 | 1 | No | -1 |
| 1 | 2 | Yes | 1 |
| 2 | 2 | Yes | 2 |
| 3 | 2 | Yes | 3 |
| 4 | 5 | No | 3 |

#### Brute-Force Complexity
**Time: `O(n)`** because all `n` elements are inspected, even after the first match. **Space: `O(1)`** because only one answer index and the loop index are stored.

### 8. Why Brute Force Is Not Ideal
It scans positions that sorted order could eliminate in groups. The duplicate block's right boundary can be located by repeatedly halving the candidate interval.

### 9. Optimal Approach
Use a closed interval `[low, high]` and initialize `answer = -1`. On equality, save `middle` and set `low = middle + 1` to search for a later match. A smaller midpoint also moves right; a larger midpoint moves left.

Invariant:

```text
answer is -1 or an index known to contain target
if a match later than answer remains possible, it lies in [low, high]
```

The remembered candidate makes it safe to discard a matched midpoint.

### 10. Visualization
```text
[1, 2, 2, 2, 5], target = 2
 low   mid        high   match at 2; answer = 2, move right
          low mid high   match at 3; answer = 3, move right
             low/high    5 is too large; stop with answer 3
```

### 11. Step-by-Step Dry Run
| Low | High | Middle | Value | Answer | Decision |
|---:|---:|---:|---:|---:|---|
| 0 | 4 | 2 | 2 | 2 | Search right; `low = 3` |
| 3 | 4 | 3 | 2 | 3 | Search right; `low = 4` |
| 4 | 4 | 4 | 5 | 3 | Too large; `high = 3` |
| 4 | 3 | — | — | 3 | Return `3` |

### 12. TypeScript Solution
```typescript
function findLastOccurrence(
  values: readonly number[],
  target: number,
): number {
  if (!Number.isSafeInteger(target)) {
    throw new RangeError("target must be a safe integer");
  }

  let low = 0;
  let high = values.length - 1;
  let answer = -1;

  while (low <= high) {
    const middle = low + Math.floor((high - low) / 2);
    const value = values[middle];

    if (value <= target) {
      if (value === target) answer = middle;
      low = middle + 1;
    } else {
      high = middle - 1;
    }
  }

  return answer;
}
```

### 13. Code Walkthrough
`answer` stores the best verified match. For `value <= target`, no index at or left of `middle` can improve a known rightmost match, so the search moves right. For a larger value, sortedness proves the right side is also too large. Crossing `low` and `high` ends the search.

### 14. Complexity Analysis
**Search time: `O(log n)`** because every iteration removes at least half of the closed candidate interval. Continuing after equality does not scan duplicates. **Space: `O(1)`**. Target validation is constant time; the caller supplies already-validated sorted data so no `O(n)` sortedness scan is hidden in this claim.

### 15. Edge Cases
- Empty input returns `-1`.
- All values equal to the target returns `n - 1`.
- A missing target returns `-1`.
- One matching value returns index `0`.
- An unsafe target throws `RangeError`.

### 16. Common Mistakes
- Returning on the first equality.
- Moving left after equality when the rightmost match is required.
- Forgetting to remember the match before discarding `middle`.
- Using a brute scan that stops at the first match.

### 17. Interview Follow-ups
1. Solve with upper bound minus one. 2. Return both first and last indexes. 3. Count occurrences. 4. Find the last value below a threshold.

### 18. Variations
- Last value less than or equal to a target. - Upper bound. - Last true value in a monotonic Boolean sequence.

### 19. Pattern Connections
Candidate-and-right search also finds a floor. An upper-bound boundary yields the same last-occurrence result after subtracting one and checking equality.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/last-occurrence` with `{ values, target }`. Validate the request and sorted-data contract outside the measured search, then return `{ index, trace, truncated }`. The optimized trace should record the closed interval, midpoint, candidate updates, and rightward continuation after equality.

### 21. Mini Challenge
Rewrite the solution as an upper-bound search for the first value greater than the target, then safely convert that boundary into a last-occurrence result.

## Problem 85 — Count Occurrences

### 1. Problem Statement
Given a `readonly` array of safe integers sorted in nondecreasing order and a safe-integer target, return how many times the target occurs. Return `0` when absent. The caller guarantees sorted safe-integer elements, the target is validated in `O(1)`, and the array is never mutated.

### 2. Example
```text
Input:  values = [1, 2, 2, 2, 5], target = 2
Output: 3
```

### 3. Understand the Problem
Because the array is sorted, all copies of the target form one contiguous block. Its length is the boundary after the block minus the boundary before the block.

#### In Simple Terms
Count the target in a sorted list. Instead of visiting every copy, find where its block starts and where it ends, then subtract those positions.

### 4. Real-World Analogy
To count books with one call number on a sorted shelf, locate the first shelf position for that number and the first position belonging to the next number; the distance is the count.

### 5. Key Observation
`lowerBound(target)` is the first index with value `>= target`, and `upperBound(target)` is the first index with value `> target`. Therefore, `upper - lower` counts exactly the equal values, including zero when the target is absent.

### 6. Pattern
```text
Pattern: Two Boundary Searches
Why this pattern applies: sorted duplicates occupy one continuous half-open range
How to recognize it: count equal values without scanning the duplicate block
Similar problems: first and last occurrence, frequency queries, equal range
```

### 7. Brute Force
The direct intuition is to inspect every value and increment a counter for each equality. This works for any array order but does not exploit sortedness.

#### Pseudocode
```text
validate target
count = 0
for each value:
    if value equals target:
        count = count + 1
return count
```

#### Brute-Force TypeScript
```typescript
function countOccurrencesBruteForce(
  values: readonly number[],
  target: number,
): number {
  if (!Number.isSafeInteger(target)) {
    throw new RangeError("target must be a safe integer");
  }

  let count = 0;
  for (const value of values) {
    if (value === target) count += 1;
  }
  return count;
}
```

#### Visualization
```text
[1, 2, 2, 2, 5], target = 2
    +  +  +
count: 1  2  3
```

#### Dry Run
| Value | Equals 2? | Count |
|---:|:---:|---:|
| 1 | No | 0 |
| 2 | Yes | 1 |
| 2 | Yes | 2 |
| 2 | Yes | 3 |
| 5 | No | 3 |

#### Brute-Force Complexity
**Time: `O(n)`** because every one of the `n` values is inspected. **Space: `O(1)`** because the counter uses fixed storage.

### 8. Why Brute Force Is Not Ideal
Even when only a few or no copies exist, the scan reads the whole array. Sortedness lets two logarithmic searches locate both edges directly.

### 9. Optimal Approach
Run a lower-bound search using `< target` to locate the block's left edge. Run an upper-bound search using `<= target` to locate the first value after the block. Both searches use `[low, high)` and return positions in `[0, n]`.

Boundary invariant:

```text
lower bound separates values < target from values >= target
upper bound separates values <= target from values > target
target block is [lower, upper), so count = upper - lower
```

No special absent-target branch is required because both boundaries are equal when no target values lie between them.

### 10. Visualization
```text
values: [1 | 2, 2, 2 | 5]
          ^           ^
        lower=1     upper=4

target indexes: [1, 4)
count = 4 - 1 = 3
```

### 11. Step-by-Step Dry Run
For `values = [1, 2, 2, 2, 5]` and `target = 2`:

| Search | Low | High | Middle | Value | Decision |
|---|---:|---:|---:|---:|---|
| Lower | 0 | 5 | 2 | 2 | `high = 2` |
| Lower | 0 | 2 | 1 | 2 | `high = 1` |
| Lower | 0 | 1 | 0 | 1 | `low = 1`; lower is `1` |
| Upper | 0 | 5 | 2 | 2 | `low = 3` |
| Upper | 3 | 5 | 4 | 5 | `high = 4` |
| Upper | 3 | 4 | 3 | 2 | `low = 4`; upper is `4` |

Result: `4 - 1 = 3`.

### 12. TypeScript Solution
```typescript
function countOccurrences(values: readonly number[], target: number): number {
  if (!Number.isSafeInteger(target)) {
    throw new RangeError("target must be a safe integer");
  }

  function findBoundary(includeEqualsOnLeft: boolean): number {
    let low = 0;
    let high = values.length;

    while (low < high) {
      const middle = low + Math.floor((high - low) / 2);
      const belongsOnLeft = includeEqualsOnLeft
        ? values[middle] <= target
        : values[middle] < target;

      if (belongsOnLeft) {
        low = middle + 1;
      } else {
        high = middle;
      }
    }

    return low;
  }

  const first = findBoundary(false);
  const afterLast = findBoundary(true);
  return afterLast - first;
}
```

### 13. Code Walkthrough
The nested helper is local to this solution, so the snippet is self-contained. With `includeEqualsOnLeft = false`, equality stays on the right and produces lower bound. With it `true`, equality moves left of the boundary and produces upper bound. Subtracting the returned indexes gives the block length.

### 14. Complexity Analysis
Each boundary search halves an interval of width `n`, taking `O(log n)`. Two such searches cost `O(log n) + O(log n) = O(log n)`; subtraction is `O(1)`. **Space: `O(1)`** auxiliary space. Sortedness is a caller precondition because checking it inside this call would add `O(n)` time.

### 15. Edge Cases
- Empty input returns `0`.
- An absent target returns `0` because both boundaries match.
- All `n` values equal the target returns `n`.
- A target outside the value range returns `0`.
- An unsafe target throws `RangeError`.

### 16. Common Mistakes
- Computing `last - first` instead of `afterLast - first`.
- Using the same comparison for both boundaries.
- Scanning from the first occurrence through all duplicates, which can still be `O(n)`.
- Declaring shared helpers that make copied snippets collide with other examples.

### 17. Interview Follow-ups
1. Return `[first, last]` instead of a count. 2. Answer many target-frequency queries. 3. Count values in an inclusive numeric range. 4. Generalize to sorted records by key.

### 18. Variations
- Equal-range query. - Count values less than a target. - Count values between two boundaries. - Frequency lookup over run-length encoded data.

### 19. Pattern Connections
The lower boundary is Problem 83's first occurrence. `upper - 1` is Problem 84's last occurrence when the target exists. The same pair supports range-count queries.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/count-occurrences` with `{ values, target }`. Validate or retrieve a trusted sorted dataset before the timed search, and return `{ count, boundaries, trace, truncated }`. Label optimized trace events by `lower` or `upper` search and record interval changes only; the brute counter is not part of execution tracing.

### 21. Mini Challenge
Given sorted values and safe integers `minimum` and `maximum`, count values in the inclusive range `[minimum, maximum]` using two boundary searches.

## Problem 86 — Find Floor and Ceiling

### 1. Problem Statement
Given a `readonly` array of safe integers sorted in nondecreasing order and a safe-integer target, return `{ floor: number | null, ceiling: number | null }`. The floor is the greatest value `<= target`; the ceiling is the smallest value `>= target`. Return `null` for a missing side, return both as the target on an exact match, and return both `null` for an empty array. The caller guarantees sorted safe-integer elements; only the target is validated in `O(1)`, and the array is not mutated.

### 2. Example
```text
Input:  values = [1, 3, 5, 7], target = 4
Output: { floor: 3, ceiling: 5 }

Input:  values = [1, 3, 5, 7], target = 5
Output: { floor: 5, ceiling: 5 }
```

### 3. Understand the Problem
This problem returns values, not indexes. The two answers surround the target in sorted order. Either side can be absent when the target lies outside the array's range.

#### In Simple Terms
Find the closest list value at or below the target and the closest list value at or above it. Use `null` when one side does not exist.

### 4. Real-World Analogy
If buses depart at sorted times, the floor is the latest departure no later than your chosen time, and the ceiling is the earliest departure no earlier than it.

### 5. Key Observation
Each comparison can update a valid candidate and discard an impossible half: a value below the target can improve the floor only by moving right, while a value above the target can improve the ceiling only by moving left.

### 6. Pattern
```text
Pattern: Dual Boundary Search
Why this pattern applies: floor and ceiling are the two ordered boundaries around a target
How to recognize it: closest value <= target and closest value >= target
Similar problems: predecessor/successor, lower bound, range lookup
```

### 7. Brute Force
The baseline intuition is to inspect every value and keep the greatest valid floor and smallest valid ceiling. Exact equality can return immediately because no closer values are possible.

#### Pseudocode
```text
validate target
floor = null
ceiling = null
for each value:
    if value equals target: return both as target
    if value < target and improves floor: update floor
    if value > target and improves ceiling: update ceiling
return floor and ceiling
```

#### Brute-Force TypeScript
```typescript
function findFloorAndCeilingBruteForce(
  values: readonly number[],
  target: number,
): { floor: number | null; ceiling: number | null } {
  if (!Number.isSafeInteger(target)) {
    throw new RangeError("target must be a safe integer");
  }

  let floor: number | null = null;
  let ceiling: number | null = null;

  for (const value of values) {
    if (value === target) return { floor: target, ceiling: target };
    if (value < target && (floor === null || value > floor)) floor = value;
    if (value > target && (ceiling === null || value < ceiling)) {
      ceiling = value;
    }
  }

  return { floor, ceiling };
}
```

#### Visualization
```text
values = [1, 3, 5, 7], target = 4
             3 < 4 < 5
             ^ floor
                 ^ ceiling
```

#### Dry Run
| Value | Relation to 4 | Floor | Ceiling |
|---:|---|---:|---:|
| 1 | Below | 1 | null |
| 3 | Below | 3 | null |
| 5 | Above | 3 | 5 |
| 7 | Above, not closer | 3 | 5 |

#### Brute-Force Complexity
**Time: `O(n)`** in the worst case because all `n` values may be inspected. **Space: `O(1)`** because only two candidate values are stored.

### 8. Why Brute Force Is Not Ideal
The scan ignores that all values left of a midpoint are no larger and all values right are no smaller. Binary search can update both candidates while removing half the positions.

### 9. Optimal Approach
Use a closed interval and maintain nullable floor and ceiling candidates. If the midpoint equals the target, return `{ floor: target, ceiling: target }`. If it is smaller, it is a valid floor candidate; save it and move right for a larger valid floor. If it is larger, save it as a ceiling candidate and move left for a smaller valid ceiling.

Invariant:

```text
floor is null or the greatest confirmed value <= target seen so far
ceiling is null or the smallest confirmed value >= target seen so far
any candidate that can improve either answer remains in [low, high]
```

### 10. Visualization
```text
[1, 3, 5, 7], target = 4
 low mid    high   value 3 -> floor = 3, move right
       low mid high       value 5 -> ceiling = 5, move left
       high low            interval crossed

result: { floor: 3, ceiling: 5 }
```

### 11. Step-by-Step Dry Run
| Low | High | Middle | Value | Floor | Ceiling | Decision |
|---:|---:|---:|---:|---:|---:|---|
| 0 | 3 | 1 | 3 | 3 | null | Move right; `low = 2` |
| 2 | 3 | 2 | 5 | 3 | 5 | Move left; `high = 1` |
| 2 | 1 | — | — | 3 | 5 | Return both candidates |

### 12. TypeScript Solution
```typescript
function findFloorAndCeiling(
  values: readonly number[],
  target: number,
): { floor: number | null; ceiling: number | null } {
  if (!Number.isSafeInteger(target)) {
    throw new RangeError("target must be a safe integer");
  }

  let low = 0;
  let high = values.length - 1;
  let floor: number | null = null;
  let ceiling: number | null = null;

  while (low <= high) {
    const middle = low + Math.floor((high - low) / 2);
    const value = values[middle];

    if (value === target) return { floor: target, ceiling: target };

    if (value < target) {
      floor = value;
      low = middle + 1;
    } else {
      ceiling = value;
      high = middle - 1;
    }
  }

  return { floor, ceiling };
}
```

### 13. Code Walkthrough
Moving right after a smaller midpoint is safe because that midpoint is the best floor seen on its discarded side. Moving left after a larger midpoint similarly preserves the best ceiling. Exact equality is simultaneously the greatest allowed floor and smallest allowed ceiling.

### 14. Complexity Analysis
**Search time: `O(log n)`** because each midpoint comparison discards at least half of the remaining indexes. **Space: `O(1)`** because the algorithm stores a fixed number of indexes and candidates. It does not scan for sortedness; doing so would change total call time to `O(n)`.

### 15. Edge Cases
- Empty input returns `{ floor: null, ceiling: null }`.
- Exact equality returns the target for both fields.
- Below the minimum gives `floor: null`.
- Above the maximum gives `ceiling: null`.
- Duplicates do not change the returned values.
- An unsafe target throws `RangeError`.

### 16. Common Mistakes
- Returning indexes when the contract asks for values.
- Using `0` instead of `null` for a missing side.
- Moving in the wrong direction after updating a candidate.
- Continuing after exact equality and complicating an already final answer.

### 17. Interview Follow-ups
1. Return floor and ceiling indexes. 2. Require strict predecessor and successor. 3. Support descending input. 4. Answer many queries against one trusted array.

### 18. Variations
- Strict floor `< target`. - Strict ceiling `> target`. - Nearest value with a tie rule. - Predecessor and successor in a binary search tree.

### 19. Pattern Connections
Ceiling is the value at a lower-bound index when one exists. Floor mirrors last-occurrence's candidate-and-right movement, but accepts every value below the target rather than only equality.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/floor-ceiling` with `{ values, target }` and return `{ floor, ceiling, trace, truncated }`. Validate or load trusted sorted values outside the measured algorithm. Trace only optimized midpoint comparisons, candidate replacements, discarded halves, and the exact-match early return.

### 21. Mini Challenge
Modify the contract to return the nearest value to the target, choosing the smaller value when floor and ceiling are equally distant.

## Problem 87 — Square Root using Binary Search

### 1. Problem Statement
Given a nonnegative safe integer `n`, return `floor(sqrt(n))` as an integer: the greatest integer whose square is no greater than `n`. Validate the scalar input in `O(1)`. Do not use `Math.sqrt`, do not create an unsafe square, and do not mutate any input.

### 2. Example
```text
Input:  n = 8
Output: 2

Explanation: 2 * 2 <= 8, but 3 * 3 > 8.
```

### 3. Understand the Problem
The square root need not be an integer. The floor rule asks for the integer immediately below it. Directly computing `candidate * candidate` is unsafe near `Number.MAX_SAFE_INTEGER`, so comparisons use `candidate <= Math.floor(n / candidate)` instead.

#### In Simple Terms
Find the largest whole number that can multiply by itself without going above `n`. For `8`, that number is `2`; do not return a decimal.

### 4. Real-World Analogy
Given `n` square tiles, find the greatest whole side length of a square you can build without needing more tiles than you have.

### 5. Key Observation
For positive candidates, the predicate `candidate <= floor(n / candidate)` is true exactly when `candidate^2 <= n`, without calculating the potentially unsafe square. The predicate changes monotonically from true to false.

### 6. Pattern
```text
Pattern: Binary Search on the Numeric Answer
Why this pattern applies: feasible candidates form a true prefix followed by false candidates
How to recognize it: find the largest integer satisfying a monotonic condition
Similar problems: nth root, minimum feasible capacity, Koko Eating Bananas
```

### 7. Brute Force
The direct intuition is to try positive integers in increasing order. Continue while a candidate can square within `n`, using division for the safety check. The first infeasible candidate is one beyond the answer.

#### Pseudocode
```text
validate n is a nonnegative safe integer
candidate = 1
while candidate <= floor(n / candidate):
    candidate = candidate + 1
return candidate - 1
```

#### Brute-Force TypeScript
```typescript
function integerSquareRootBruteForce(n: number): number {
  if (!Number.isSafeInteger(n) || n < 0) {
    throw new RangeError("n must be a nonnegative safe integer");
  }

  let candidate = 1;
  while (candidate <= Math.floor(n / candidate)) {
    candidate += 1;
  }
  return candidate - 1;
}
```

#### Visualization
```text
n = 8

candidate:       1   2   3
fits by division T   T   F
                       ^ first failure
answer = 3 - 1 = 2
```

#### Dry Run
| Candidate | `floor(8 / candidate)` | Candidate fits? | Action |
|---:|---:|:---:|---|
| 1 | 8 | Yes | Try 2 |
| 2 | 4 | Yes | Try 3 |
| 3 | 2 | No | Return `2` |

#### Brute-Force Complexity
**Time: `O(sqrt(n))`** because candidates `1` through `floor(sqrt(n)) + 1` are tested. **Space: `O(1)`** because only one candidate is stored. Division avoids unsafe multiplication but does not improve the linear number of candidates.

### 8. Why Brute Force Is Not Ideal
The brute loop advances one answer at a time. Since feasibility is monotonic, one comparison can discard half the remaining numeric candidates.

### 9. Optimal Approach
Return immediately for `0` and `1`. Otherwise search the closed candidate interval `[1, min(n, 94_906_265)]`, where `94_906_265` is `floor(sqrt(Number.MAX_SAFE_INTEGER))`. This upper bound guarantees every possible answer for a safe `n` is included.

Maintain this invariant:

```text
answer is the greatest confirmed candidate whose square is <= n
every untested candidate that could improve answer remains in [low, high]
```

Test feasibility with `middle <= Math.floor(n / middle)`. If true, remember `middle` and search right; otherwise search left. No multiplication or `Math.sqrt` is used.

### 10. Visualization
```text
n = 27, candidates [1..27]

[1 ........ 14 ........ 27]  14 > floor(27/14)=1 -> move left
[1 .. 3 .. 7 .. 13]          7 > floor(27/7)=3   -> move left
[1 2 3] 4 5 6                3 <= floor(27/3)=9  -> answer=3, right
[4 5 6]                       5 <= floor(27/5)=5  -> answer=5, right
[6]                           6 > floor(27/6)=4   -> stop

floor(sqrt(27)) = 5
```

### 11. Step-by-Step Dry Run
| Low | High | Middle | `floor(27 / middle)` | Feasible? | Answer | Decision |
|---:|---:|---:|---:|:---:|---:|---|
| 1 | 27 | 14 | 1 | No | 1 | `high = 13` |
| 1 | 13 | 7 | 3 | No | 1 | `high = 6` |
| 1 | 6 | 3 | 9 | Yes | 3 | `low = 4` |
| 4 | 6 | 5 | 5 | Yes | 5 | `low = 6` |
| 6 | 6 | 6 | 4 | No | 5 | `high = 5` |

Return `5`.

### 12. TypeScript Solution
```typescript
function integerSquareRoot(n: number): number {
  if (!Number.isSafeInteger(n) || n < 0) {
    throw new RangeError("n must be a nonnegative safe integer");
  }
  if (n < 2) return n;

  const MAX_SAFE_SQUARE_ROOT = 94_906_265;
  let low = 1;
  let high = Math.min(n, MAX_SAFE_SQUARE_ROOT);
  let answer = 1;

  while (low <= high) {
    const middle = low + Math.floor((high - low) / 2);

    if (middle <= Math.floor(n / middle)) {
      answer = middle;
      low = middle + 1;
    } else {
      high = middle - 1;
    }
  }

  return answer;
}
```

### 13. Code Walkthrough
The early return handles the only answers below `1`. The safe-root cap is a proven inclusive upper bound for every safe integer input. Because `middle` is positive, division is valid; `Math.floor(n / middle)` produces the exact integer quotient needed for comparison without forming `middle * middle`. A feasible midpoint becomes the answer candidate and moves the search right.

### 14. Complexity Analysis
Let `H = min(n, floor(sqrt(Number.MAX_SAFE_INTEGER)))`. The interval width changes as `H -> H / 2 -> H / 4 -> ... -> 1`, so **time is `O(log H)`**, commonly written `O(log n)` over the input range. For JavaScript safe integers, the cap also limits the loop to a small constant number of iterations, but logarithmic answer-space reasoning remains the useful model. **Space is `O(1)`**. Scalar validation is `O(1)`.

### 15. Edge Cases
- `n = 0` returns `0`.
- `n = 1` returns `1`.
- Perfect squares return their exact root.
- Non-perfect squares return the lower integer.
- `Number.MAX_SAFE_INTEGER` is handled without unsafe multiplication.
- Negative, fractional, infinite, `NaN`, or unsafe inputs throw `RangeError`.

### 16. Common Mistakes
- Using `middle * middle`, which can exceed safe integer precision.
- Calling `Math.sqrt`, which bypasses the requested algorithm.
- Returning the first feasible midpoint instead of the greatest feasible one.
- Dividing by zero by including `0` in the general search loop.
- Claiming `O(sqrt(n))` for the optimized binary search.

### 17. Interview Follow-ups
1. Compute an integer cube root safely. 2. Return a decimal approximation within an error tolerance. 3. Find the nth root. 4. Prove the safe upper bound. 5. Implement with `bigint`.

### 18. Variations
- Integer cube root. - Perfect-square test. - Decimal square root by precision iterations. - BigInt square root. - Generic last-true predicate search.

### 19. Pattern Connections
This is binary search over answers rather than array indexes. The same last-feasible invariant appears in capacity, speed, allocation, and threshold problems later in the chapter.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/integer-square-root` with `{ n }`. Validate `n` as a nonnegative safe integer, then return `{ root, trace, truncated }`. Each optimized trace event can include `low`, `high`, `middle`, integer quotient, feasibility, candidate answer, and discarded interval; never calculate a square merely for tracing.

### 21. Mini Challenge
Write `isPerfectSquare(n)` using the returned integer root while preserving safe-integer arithmetic and without multiplying the root by itself.

## Problem 88 — Search in Rotated Sorted Array

### 1. Problem Statement
Given a `readonly number[]` that was strictly increasing before being rotated zero or more times, and a safe-integer `target`, return the target's index or `-1` when it is absent. Values are distinct safe integers. Empty input returns `-1`, and the function must not mutate the array.

The array shape, ordering, distinctness, and safe-integer claims are caller preconditions. The optimized search does not spend `O(n)` time validating them.

### 2. Example
```text
Input:  values = [4, 5, 6, 7, 0, 1, 2], target = 0
Output: 4

Explanation: 0 is stored at index 4.
```

### 3. Understand the Problem
Rotation moves a sorted prefix to the end without changing the order inside either resulting part. The full array may not be sorted, but at least one half around any midpoint is sorted. Distinct values make that sorted-half test unambiguous.

#### In Simple Terms
The sorted list was cut and its pieces were swapped. Find which side of the midpoint is still normally ordered, then decide whether the target fits inside that side.

### 4. Real-World Analogy
Imagine numbered lockers arranged in order, but the row begins partway through the numbering and wraps around. At each checkpoint, one direction still shows a normal consecutive range. If the requested locker number belongs to that range, search there; otherwise search the wrapped side.

### 5. Key Observation
For a closed interval `[low, high]` with distinct values, either `values[low] <= values[middle]`, so the left half is sorted, or the right half is sorted. Once the sorted half is known, inclusive boundary checks tell whether the target can be in it.

### 6. Pattern
```text
Pattern: Modified Binary Search on a Rotated Array
Why this pattern applies: one side of every midpoint is sorted
How to recognize it: a distinct sorted array was rotated and an exact value is requested
Similar problems: find rotated minimum, find rotation count, search a bitonic array
```

### 7. Brute Force
Scan from left to right and return the first index whose value equals the target. Distinctness means there can be at most one match.

#### Pseudocode
```text
for index from 0 through values.length - 1:
    if values[index] equals target:
        return index
return -1
```

#### Brute-Force TypeScript
```typescript
function searchRotatedArrayBruteForce(
  values: readonly number[],
  target: number,
): number {
  for (let index = 0; index < values.length; index += 1) {
    if (values[index] === target) return index;
  }
  return -1;
}
```

#### Visualization
```text
values = [4, 5, 6, 7, 0, 1, 2], target = 0

index:    0  1  2  3  4
value:    4  5  6  7  0
check:    x  x  x  x  match -> return 4
```

#### Dry Run
| Index | Value | Equals 0? | Action |
|---:|---:|:---:|---|
| 0 | 4 | No | Continue |
| 1 | 5 | No | Continue |
| 2 | 6 | No | Continue |
| 3 | 7 | No | Continue |
| 4 | 0 | Yes | Return `4` |

#### Brute-Force Complexity
**Time: `O(n)`** in the worst case because every index may be checked. **Space: `O(1)`** because the scan stores only an index. The input is read but never changed.

### 8. Why Brute Force Is Not Ideal
The scan ignores the order retained by rotation. Identifying a sorted half lets one comparison discard about half of the remaining indexes.

### 9. Optimal Approach
Search the closed interval `[low, high]`. After checking the midpoint for equality, identify a sorted half:

```text
if values[low] <= values[middle]:
    left half is sorted
    keep it only when values[low] <= target < values[middle]
otherwise:
    right half is sorted
    keep it only when values[middle] < target <= values[high]
```

The comparisons include the outer boundary and exclude the already-checked midpoint. If the target is not in the sorted half's value range, discard that half and search the other one.

### 10. Visualization
```text
values = [4, 5, 6, 7, 0, 1, 2], target = 0

low=0             mid=3             high=6
[4, 5, 6,          7,          0, 1, 2]
 [4..7] is sorted; 0 is not in [4..7) -> discard left

                    low=4 mid=5 high=6
                    [0,    1,    2]
                    [0..1] is sorted; 0 is in [0..1) -> keep left

                    low=4=mid=high
                    [0] -> match at index 4
```

### 11. Step-by-Step Dry Run
| Low | High | Middle | Value | Sorted Half | Membership Decision | Update |
|---:|---:|---:|---:|---|---|---|
| 0 | 6 | 3 | 7 | Left `[4..7]` | `0` is not in `[4..7)` | `low = 4` |
| 4 | 6 | 5 | 1 | Left `[0..1]` | `0` is in `[0..1)` | `high = 4` |
| 4 | 4 | 4 | 0 | Match | Target found | Return `4` |

### 12. TypeScript Solution
```typescript
function searchRotatedArray(
  values: readonly number[],
  target: number,
): number {
  let low = 0;
  let high = values.length - 1;

  while (low <= high) {
    const middle = low + Math.floor((high - low) / 2);
    const value = values[middle];

    if (value === target) return middle;

    if (values[low] <= value) {
      if (values[low] <= target && target < value) {
        high = middle - 1;
      } else {
        low = middle + 1;
      }
    } else if (value < target && target <= values[high]) {
      low = middle + 1;
    } else {
      high = middle - 1;
    }
  }

  return -1;
}
```

### 13. Code Walkthrough
The loop first handles equality, so later range tests may exclude `middle`. `values[low] <= value` identifies a sorted left half; otherwise the distinct-value precondition guarantees the right half is sorted. The matching range keeps the target's possible half, while `middle + 1` or `middle - 1` removes the checked midpoint and guarantees progress. An empty array starts with `high = -1`, skips the loop, and returns `-1`.

### 14. Complexity Analysis
**Search time: `O(log n)`** because each iteration keeps at most one half of the current interval: `n -> n / 2 -> n / 4 -> ... -> 1`. After about `log2(n)` halvings, no candidates remain or the target is found. **Space: `O(1)`** because only indexes and one midpoint value are stored.

Checking every element for safe-integer values, distinctness, and rotated sorted order at an API boundary costs `O(n)` separately. That validation is not part of the optimized search and should occur only when the caller is not already trusted.

### 15. Edge Cases
- Empty input returns `-1`.
- One element returns `0` when it matches and `-1` otherwise.
- An unrotated array is valid.
- The target may be at index `0` or `values.length - 1`.
- A rotation by the array length is equivalent to no rotation.
- A missing target returns `-1`.

### 16. Common Mistakes
- Assuming the same half is sorted on every iteration.
- Using `target <= values[middle]` after midpoint equality was already ruled out.
- Omitting equality at `values[low]` or `values[high]` and losing a boundary target.
- Allowing duplicates without changing the algorithm's ambiguity handling.
- Adding an `O(n)` shape-validation scan inside the optimized function and still claiming total `O(log n)` time.

### 17. Interview Follow-ups
1. Support duplicate values. 2. Return an insertion position. 3. Find the first match when duplicates exist. 4. Search a rotated descending array. 5. Validate whether an array has the required shape.

### 18. Variations
- Rotated array with duplicates. - Search in a bitonic array. - Search a rotated matrix row. - Return the target value instead of its index.

### 19. Pattern Connections
This extends exact binary search by using local order rather than global order. Its sorted-half decision is related to the pivot logic used by rotated minimum and rotation-count searches.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/rotated-search` with `{ values, target }` and return `{ index, trace, truncated }`. At the DTO or service boundary, scalar target validation costs `O(1)`, while verifying all array values and the rotated strictly increasing shape costs `O(n)`; keep that cost separate from the trusted `O(log n)` search. Trace only optimized states such as `low`, `high`, `middle`, identified sorted half, membership result, and discarded interval.

### 21. Mini Challenge
Trace the optimized search for `values = [6, 7, 8, 1, 2, 3, 4, 5]` and `target = 7`, writing the sorted half and retained interval at every step.

## Problem 89 — Find Minimum in Rotated Sorted Array

### 1. Problem Statement
Given a non-empty `readonly number[]` that was strictly increasing before being rotated zero or more times, return its minimum value. Values are distinct safe integers. Empty input throws `RangeError`, and the function must not mutate the array.

Array shape, ordering, distinctness, and safe-integer claims are caller preconditions. The optimized function does not perform an `O(n)` validation scan.

### 2. Example
```text
Input:  values = [4, 5, 6, 7, 0, 1, 2]
Output: 0

Explanation: 0 is the smallest value.
```

### 3. Understand the Problem
In an unrotated strictly increasing array, the first value is the minimum. After rotation, the minimum is the point where the values wrap from the largest region back to the smallest region.

#### In Simple Terms
Find the place where the ordered numbers restart. The value at that restart is the minimum.

### 4. Real-World Analogy
Imagine a circular calendar opened at an arbitrary month. The months still move forward, but after December they wrap to January. January is the minimum-like restart point.

### 5. Key Observation
Compare `values[middle]` with `values[high]`. If the midpoint is greater, the minimum must be strictly to its right. Otherwise the midpoint could be the minimum, so keep it and move `high` to `middle`.

### 6. Pattern
```text
Pattern: Binary Search by Closed Convergence
Why this pattern applies: comparison with the right boundary locates the rotation side
How to recognize it: find the smallest value in a distinct rotated sorted array
Similar problems: rotation count, rotated search, first value after a cyclic wrap
```

### 7. Brute Force
Initialize the minimum with the first value, then scan the remaining values and replace it whenever a smaller value appears.

#### Pseudocode
```text
if values is empty:
    throw RangeError
minimum = values[0]
for index from 1 through values.length - 1:
    minimum = smaller of minimum and values[index]
return minimum
```

#### Brute-Force TypeScript
```typescript
function findMinimumRotatedBruteForce(values: readonly number[]): number {
  if (values.length === 0) {
    throw new RangeError("values must not be empty");
  }

  let minimum = values[0];
  for (let index = 1; index < values.length; index += 1) {
    if (values[index] < minimum) minimum = values[index];
  }
  return minimum;
}
```

#### Visualization
```text
values:  4  5  6  7  0  1  2
minimum: 4  4  4  4  0  0  0
                         ^ final minimum
```

#### Dry Run
| Index | Value | Minimum Before | Minimum After |
|---:|---:|---:|---:|
| 1 | 5 | 4 | 4 |
| 2 | 6 | 4 | 4 |
| 3 | 7 | 4 | 4 |
| 4 | 0 | 4 | 0 |
| 5 | 1 | 0 | 0 |
| 6 | 2 | 0 | 0 |

#### Brute-Force Complexity
**Time: `O(n)`** because every value is considered. **Space: `O(1)`** because only the current minimum and an index are stored. The input remains unchanged.

### 8. Why Brute Force Is Not Ideal
The scan uses no information from the two increasing regions. A midpoint-to-right-boundary comparison can identify which half must still contain the wrap point.

### 9. Optimal Approach
Use a closed candidate interval `[low, high]` and converge until one index remains:

```text
while low < high:
    middle = low + floor((high - low) / 2)
    if values[middle] > values[high]:
        low = middle + 1
    else:
        high = middle
return values[low]
```

When the midpoint exceeds the right boundary, both are on opposite sides of the wrap and the minimum is right of `middle`. Otherwise `middle` belongs to the low-valued side and may itself be the minimum, so it must remain a candidate.

### 10. Visualization
```text
values = [4, 5, 6, 7, 0, 1, 2]

low=0             mid=3             high=6
[4, 5, 6,          7,          0, 1, 2]
7 > 2  -> minimum is right of mid -> [0, 1, 2]

                    low=4 mid=5 high=6
                    [0,    1,    2]
1 <= 2 -> mid may be minimum      -> [0, 1]

                    low=4 high=5
                    [0,    1]
0 <= 1 -> keep mid                -> [0]

values[4] = 0
```

### 11. Step-by-Step Dry Run
| Low | High | Middle | `values[middle]` | `values[high]` | Decision | New Interval |
|---:|---:|---:|---:|---:|---|---|
| 0 | 6 | 3 | 7 | 2 | `7 > 2`, move right | `[4, 6]` |
| 4 | 6 | 5 | 1 | 2 | `1 <= 2`, keep middle | `[4, 5]` |
| 4 | 5 | 4 | 0 | 1 | `0 <= 1`, keep middle | `[4, 4]` |

Return `values[4]`, which is `0`.

### 12. TypeScript Solution
```typescript
function findMinimumRotated(values: readonly number[]): number {
  if (values.length === 0) {
    throw new RangeError("values must not be empty");
  }

  let low = 0;
  let high = values.length - 1;

  while (low < high) {
    const middle = low + Math.floor((high - low) / 2);

    if (values[middle] > values[high]) {
      low = middle + 1;
    } else {
      high = middle;
    }
  }

  return values[low];
}
```

### 13. Code Walkthrough
The empty check enforces the return-value contract in `O(1)`. With `low < high`, the lower midpoint is always less than `high`, so comparing two valid indexes makes progress. `low = middle + 1` safely removes a midpoint proven larger than the minimum. `high = middle` keeps a midpoint that may be the answer. Convergence at `low === high` leaves exactly the minimum index, and the function returns its value.

### 14. Complexity Analysis
**Search time: `O(log n)`** because the candidate interval is reduced to at most half each time: `n -> n / 2 -> n / 4 -> ... -> 1`. This requires about `log2(n)` comparisons. **Space: `O(1)`** because the algorithm stores only three indexes.

Validating every value and proving the distinct rotated-order precondition at an external API boundary costs `O(n)` separately. The optimized function assumes its caller has already established that contract.

### 15. Edge Cases
- Empty input throws `RangeError`.
- One element returns that element.
- An unrotated array returns its first value.
- A rotation by one or by `values.length - 1` is valid.
- Negative safe integers work because only ordering matters.
- The minimum may be at either boundary.

### 16. Common Mistakes
- Returning the minimum index when the contract asks for the value.
- Using `while (low <= high)` with convergence updates and failing to terminate.
- Setting `high = middle - 1` when `middle` may be the minimum.
- Comparing with a fixed final element instead of the current `values[high]`.
- Applying this distinct-value logic to duplicates without handling equality ambiguity.

### 17. Interview Follow-ups
1. Return the minimum index. 2. Support duplicate values. 3. Find the maximum value. 4. Count rotations. 5. Prove whether a supplied array is a valid rotation.

### 18. Variations
- Minimum with duplicates. - Maximum in a rotated array. - Rotation count. - Minimum in a rotated descending array.

### 19. Pattern Connections
This is a boundary search for the rotation pivot. Returning the pivot's value solves minimum; returning its index solves the right-rotation count defined in the next problem.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/rotated-minimum` with `{ values }` and return `{ minimum, trace, truncated }`. Reject an empty DTO in `O(1)`. If the endpoint must verify every safe integer and the distinct rotated shape, report that separate `O(n)` boundary-validation cost rather than attributing it to the optimized search. Trace only optimized `low`, `high`, `middle`, boundary comparison, chosen side, and converged index.

### 21. Mini Challenge
Change the optimized function to return `{ value, index }` while preserving `O(log n)` time and `O(1)` space.

## Problem 90 — Find Rotation Count

### 1. Problem Statement
Given a non-empty `readonly number[]` that was strictly increasing before being rotated zero or more times, return the rotation count defined as the **index of the minimum value**. Values are distinct safe integers. An unrotated array returns `0`, empty input throws `RangeError`, and the function must not mutate the array.

This definition measures a right rotation: `[1, 2, 3, 4]` right-rotated once becomes `[4, 1, 2, 3]`, whose minimum is at index `1`. A left rotation by one produces `[2, 3, 4, 1]`, whose minimum index is `3`, so it has right-rotation count `3`, not `1`. Array validity is a caller precondition, not an `O(n)` check inside the optimized function.

### 2. Example
```text
Input:  values = [4, 5, 1, 2, 3]
Output: 2

Explanation: the minimum value 1 is at index 2, so the defined rotation count is 2.
```

### 3. Understand the Problem
The result is not the minimum value. It is the minimum value's index, which also equals the number of right rotations modulo the array length under this contract.

#### In Simple Terms
Find where the smallest number sits and return that position. Be explicit about direction: this chapter calls that index the right-rotation count.

### 4. Real-World Analogy
Numbered cards start in ascending order. Moving the last card to the front once shifts the original first card one position right. The original first card is the minimum, so its new index records how many right shifts occurred modulo the deck size.

### 5. Key Observation
The minimum index can be found with the same convergence rule as the rotated minimum problem. Comparing the midpoint with the current right boundary reveals whether the minimum is strictly right of the midpoint or still at-or-left of it.

### 6. Pattern
```text
Pattern: Binary Search for a Rotation Pivot
Why this pattern applies: the minimum index separates the two increasing regions
How to recognize it: rotation count is explicitly defined as the minimum's index
Similar problems: rotated minimum, rotated exact search, cyclic boundary search
```

### 7. Brute Force
Track the index of the smallest value encountered. Scanning all values directly finds the minimum index, which is the required rotation count.

#### Pseudocode
```text
if values is empty:
    throw RangeError
minimumIndex = 0
for index from 1 through values.length - 1:
    if values[index] < values[minimumIndex]:
        minimumIndex = index
return minimumIndex
```

#### Brute-Force TypeScript
```typescript
function findRotationCountBruteForce(values: readonly number[]): number {
  if (values.length === 0) {
    throw new RangeError("values must not be empty");
  }

  let minimumIndex = 0;
  for (let index = 1; index < values.length; index += 1) {
    if (values[index] < values[minimumIndex]) minimumIndex = index;
  }
  return minimumIndex;
}
```

#### Visualization
```text
values:        4  5  1  2  3
index:         0  1  2  3  4
minimumIndex:  0  0  2  2  2
                     ^ answer
```

#### Dry Run
| Index | Value | Current Minimum Index | Decision |
|---:|---:|---:|---|
| 1 | 5 | 0 | Keep `0` |
| 2 | 1 | 0 | Replace with `2` |
| 3 | 2 | 2 | Keep `2` |
| 4 | 3 | 2 | Keep `2` |

Return `2`.

#### Brute-Force Complexity
**Time: `O(n)`** because every value is inspected. **Space: `O(1)`** because only indexes are stored. The function reads from the array without mutating it.

### 8. Why Brute Force Is Not Ideal
The scan finds the pivot but ignores the ordered regions around it. Binary convergence can discard about half of the possible pivot indexes per comparison.

### 9. Optimal Approach
Maintain the closed candidate interval `[low, high]` and continue while more than one index remains:

```text
compare values[middle] with values[high]
if values[middle] > values[high]:
    pivot is strictly right of middle, so low = middle + 1
otherwise:
    middle may be the pivot, so high = middle
```

When the boundaries meet, return `low`, not `values[low]`. Under the stated definition, this index is the right-rotation count modulo `values.length`. A caller discussing left rotations must convert or relabel the result rather than silently changing the meaning.

### 10. Visualization
```text
values = [4, 5, 1, 2, 3]

low=0      mid=2      high=4
[4, 5,      1,      2, 3]
1 <= 3 -> keep mid and left side -> [4, 5, 1]

low=0 mid=1 high=2
[4,    5,    1]
5 > 1 -> pivot is right of mid   -> [1]
                                        ^ index 2

right-rotation count = minimum index = 2
```

### 11. Step-by-Step Dry Run
| Low | High | Middle | `values[middle]` | `values[high]` | Decision | New Interval |
|---:|---:|---:|---:|---:|---|---|
| 0 | 4 | 2 | 1 | 3 | `1 <= 3`, keep middle | `[0, 2]` |
| 0 | 2 | 1 | 5 | 1 | `5 > 1`, move right | `[2, 2]` |

Return index `2`.

### 12. TypeScript Solution
```typescript
function findRotationCount(values: readonly number[]): number {
  if (values.length === 0) {
    throw new RangeError("values must not be empty");
  }

  let low = 0;
  let high = values.length - 1;

  while (low < high) {
    const middle = low + Math.floor((high - low) / 2);

    if (values[middle] > values[high]) {
      low = middle + 1;
    } else {
      high = middle;
    }
  }

  return low;
}
```

### 13. Code Walkthrough
The empty guard prevents an invalid index result. The lower midpoint and `low < high` ensure each update shrinks the interval. A midpoint larger than the right boundary lies before the wrap, so neither it nor anything left of it can be the minimum. Otherwise the midpoint may be the minimum and remains included through `high = middle`. At convergence, `low` is the minimum index and therefore the defined rotation count.

### 14. Complexity Analysis
**Search time: `O(log n)`** because each comparison leaves at most half the candidate indexes: `n -> n / 2 -> n / 4 -> ... -> 1`. The number of iterations is therefore proportional to `log2(n)`. **Space: `O(1)`** because only `low`, `high`, and `middle` are used.

An API may separately spend `O(n)` checking safe integers, distinctness, and the rotated strictly increasing shape. The optimized algorithm treats those facts as documented caller preconditions.

### 15. Edge Cases
- Empty input throws `RangeError`.
- One element returns `0`.
- An unrotated array returns `0`.
- A full-length rotation is equivalent to zero rotations and returns `0`.
- A right rotation by one returns `1`.
- A left rotation by one returns `values.length - 1` under this right-rotation/minimum-index definition.

### 16. Common Mistakes
- Returning `values[low]` instead of `low`.
- Calling a left-rotation amount the answer without converting it.
- Setting `high = middle - 1` and discarding a possible minimum.
- Using the method on duplicate values without resolving equality ambiguity.
- Scanning to validate shape inside the optimized function while claiming end-to-end `O(log n)` time.

### 17. Interview Follow-ups
1. Return the equivalent left-rotation count. 2. Support duplicates. 3. Rotate the array back without mutation. 4. Determine whether two arrays differ only by rotation. 5. Return both pivot index and minimum value.

### 18. Variations
- Left-rotation count. - Rotation count with duplicates. - Pivot in a descending array. - Circular shift distance between two arrays.

### 19. Pattern Connections
This is the index-returning form of rotated minimum. Once the pivot is known, it can map logical sorted positions to physical indexes or split the array into two ordinary binary-search ranges.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/rotation-count` with `{ values }` and return `{ rightRotationCount, minimumIndex, trace, truncated }`; using both field names makes the direction convention explicit. Reject empty input in `O(1)`. Optional full array validation costs `O(n)` at the boundary and remains separate from the optimized trace of `low`, `high`, `middle`, comparison, retained interval, and final index.

### 21. Mini Challenge
For a non-empty array of length `n`, derive a formula that converts the returned right-rotation count into the equivalent left-rotation count in the range `0` through `n - 1`.

## Problem 91 — Find Peak Element

### 1. Problem Statement
Given a non-empty `readonly number[]` of safe integers whose adjacent values are unequal, return any index whose value is strictly greater than each existing neighbor. Treat neighbors outside the array as negative infinity. Empty input throws `RangeError`, and the function must not mutate the array.

Safe-integer shape and adjacent inequality are caller preconditions. The optimized function does not scan `O(n)` elements to validate them.

### 2. Example
```text
Input:  values = [1, 2, 3, 1]
Output: 2

Explanation: values[2] = 3 is greater than values[1] = 2 and values[3] = 1.
```

### 3. Understand the Problem
The array is not required to be sorted and may contain several peaks. The function may return any valid one. At index `0`, only the right neighbor exists; at the last index, only the left neighbor exists. A one-element array therefore has a peak at index `0`.

#### In Simple Terms
Find any hilltop. A value is a hilltop when it is higher than the value immediately beside it on every side that exists.

### 4. Real-World Analogy
Walk along a mountain profile sampled at distinct neighboring heights. If the path rises to the right, some summit must remain ahead before the profile can leave the array. If it falls to the right, a summit remains at the current position or somewhere to the left.

### 5. Key Observation
Compare `values[middle]` with `values[middle + 1]`. An upward slope means a peak exists in `[middle + 1, high]`; continuing right must eventually reach a descent or the right boundary, either of which supplies a peak. A downward slope means a peak exists in `[low, middle]`, and `middle` itself might be that peak.

### 6. Pattern
```text
Pattern: Binary Search on a Local Slope
Why this pattern applies: slope direction proves one half still contains a peak
How to recognize it: any local maximum is acceptable and adjacent values are unequal
Similar problems: mountain peak, bitonic maximum, local minimum
```

### 7. Brute Force
Check every index. For a missing left or right neighbor, use negative infinity so a boundary value can be a strict peak.

#### Pseudocode
```text
if values is empty:
    throw RangeError
for index from 0 through values.length - 1:
    left = values[index - 1] or negative infinity when absent
    right = values[index + 1] or negative infinity when absent
    if values[index] > left and values[index] > right:
        return index
```

#### Brute-Force TypeScript
```typescript
function findPeakElementBruteForce(values: readonly number[]): number {
  if (values.length === 0) {
    throw new RangeError("values must not be empty");
  }

  for (let index = 0; index < values.length; index += 1) {
    const left = index === 0 ? Number.NEGATIVE_INFINITY : values[index - 1];
    const right =
      index === values.length - 1
        ? Number.NEGATIVE_INFINITY
        : values[index + 1];

    if (values[index] > left && values[index] > right) return index;
  }

  throw new RangeError("adjacent values must be unequal");
}
```

#### Visualization
```text
values = [1, 2, 3, 1]

index 0: -infinity < 1 < 2  -> not a peak
index 1:           1 < 2 < 3  -> not a peak
index 2:           2 < 3 > 1  -> return 2
```

#### Dry Run
| Index | Left | Value | Right | Peak? | Action |
|---:|---:|---:|---:|:---:|---|
| 0 | `-Infinity` | 1 | 2 | No | Continue |
| 1 | 1 | 2 | 3 | No | Continue |
| 2 | 2 | 3 | 1 | Yes | Return `2` |

#### Brute-Force Complexity
**Time: `O(n)`** because the scan may test every index. **Space: `O(1)`** because only an index and two neighbor values are stored. The array is never changed.

### 8. Why Brute Force Is Not Ideal
Checking both neighbors at every position ignores the directional clue supplied by one adjacent pair. A single slope comparison can preserve a half known to contain at least one peak.

### 9. Optimal Approach
Start with the closed candidate interval `[0, values.length - 1]`. While `low < high`, the lower midpoint guarantees `middle + 1 <= high`, so the right-neighbor comparison is valid.

```text
if values[middle] < values[middle + 1]:
    slope rises, so low = middle + 1
otherwise:
    slope falls, so high = middle
```

Why a peak remains on the chosen side: on a rising slope, following rises to the right must either meet a fall, creating a strict peak, or reach the right boundary, which is a peak against negative infinity. The falling case is symmetric toward the left, and `middle` remains included because it may already exceed both neighbors.

### 10. Visualization
```text
values = [1, 2, 1, 3, 5, 6, 4]

low=0       mid=3                 high=6
[1, 2, 1,    3,             5, 6, 4]
3 < 5 -> rising -> a peak remains right -> [5, 6, 4]

                  low=4 mid=5 high=6
                  [5,    6,    4]
6 > 4 -> falling -> keep mid and left   -> [5, 6]

                  low=4 high=5
                  [5,    6]
5 < 6 -> rising -> move right           -> [6]

index 5 is a peak
```

### 11. Step-by-Step Dry Run
| Low | High | Middle | `values[middle]` | `values[middle + 1]` | Slope | Update |
|---:|---:|---:|---:|---:|---|---|
| 0 | 6 | 3 | 3 | 5 | Rising | `low = 4` |
| 4 | 6 | 5 | 6 | 4 | Falling | `high = 5` |
| 4 | 5 | 4 | 5 | 6 | Rising | `low = 5` |

Return index `5`; `values[5] = 6` is greater than both neighbors.

### 12. TypeScript Solution
```typescript
function findPeakElement(values: readonly number[]): number {
  if (values.length === 0) {
    throw new RangeError("values must not be empty");
  }

  let low = 0;
  let high = values.length - 1;

  while (low < high) {
    const middle = low + Math.floor((high - low) / 2);

    if (values[middle] < values[middle + 1]) {
      low = middle + 1;
    } else {
      high = middle;
    }
  }

  return low;
}
```

### 13. Code Walkthrough
The empty guard enforces the non-empty contract. While two or more candidates remain, the lower midpoint always has a valid `middle + 1`. A rise removes `middle` and everything left of it because a peak is guaranteed to the right. A fall keeps `middle` and removes everything strictly right of it. Adjacent inequality makes every comparison strictly rising or falling. When the indexes meet, the invariant guarantees that index is a peak.

### 14. Complexity Analysis
**Search time: `O(log n)`** because every slope comparison keeps at most half of the candidate interval: `n -> n / 2 -> n / 4 -> ... -> 1`. After about `log2(n)` halvings, one peak index remains. **Space: `O(1)`** because only index variables are stored.

Checking every input for safe-integer values and unequal adjacent pairs costs `O(n)` at an untrusted API boundary. That optional validation is separate from the optimized search's `O(log n)` behavior.

### 15. Edge Cases
- Empty input throws `RangeError`.
- One element returns `0` because both outside neighbors are negative infinity.
- A strictly decreasing array returns boundary peak index `0`.
- A strictly increasing array returns boundary peak index `values.length - 1`.
- Multiple peaks are allowed, and any valid peak index may be returned.
- Negative safe integers work, including boundary peaks.

### 16. Common Mistakes
- Requiring the array to be globally sorted.
- Accessing `values[middle + 1]` with a loop that permits `middle === high`.
- Moving `high = middle - 1` on a falling slope and discarding a possible peak at `middle`.
- Treating outside neighbors as zero instead of negative infinity.
- Allowing equal adjacent values without adapting the proof.
- Adding an `O(n)` adjacency check inside the optimized function while reporting total `O(log n)` time.

### 17. Interview Follow-ups
1. Return all peak indexes. 2. Find a peak in a matrix. 3. Find a local minimum. 4. Require the highest peak rather than any peak. 5. Handle equal adjacent values under a revised contract.

### 18. Variations
- Peak in a mountain array. - Maximum in a bitonic array. - Local minimum. - Two-dimensional peak finding. - Circular-array peak.

### 19. Pattern Connections
Unlike ordinary binary search, this algorithm does not compare against a target. It searches a guaranteed local structure. The same slope rule finds a bitonic maximum, while two-dimensional peak finding extends the idea by choosing a direction from row or column maxima.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/peak-element` with `{ values }` and return `{ index, value, trace, truncated }`. Reject empty input in `O(1)`. If requests are untrusted, safe-integer and adjacent-inequality validation costs `O(n)` at the API boundary; do not include it in the optimized search timing. Trace only optimized `low`, `high`, `middle`, adjacent pair, slope direction, retained half, and final peak.

### 21. Mini Challenge
Without changing the asymptotic complexity, return `{ index, value }` for the peak and explain why reading `values[index]` after the search is safe.

## Problem 92 — Capacity to Ship Packages Within D Days

### 1. Problem Statement
Given package `weights: readonly number[]` and a positive integer `days`, return the minimum positive ship capacity that delivers every package in order within at most `days` nonempty shipping days. Each weight must be a positive safe integer. The array must not be mutated.

An empty array returns `0` because no capacity is needed, but `days` must still be a positive safe integer. For nonempty input, shipping may finish before `days`; unused days are allowed. Throw `RangeError` for an invalid scalar, an invalid weight, or a total weight that is not a safe integer.

### 2. Example
```text
Input:  weights = [1,2,3,4,5,6,7,8,9,10], days = 5
Output: 15

One schedule: [1,2,3,4,5] [6,7] [8] [9] [10]
Daily loads:   15          13    8   9    10
```

### 3. Understand the Problem
Packages cannot be reordered, and a package cannot be split. A capacity is valid when a left-to-right scan can form at most `days` nonempty groups whose sums do not exceed that capacity. The goal is not to find a schedule for a given capacity; it is to find the smallest capacity for which such a schedule exists.

#### In Simple Terms
Load packages in their given order without exceeding the ship's daily limit. Find the smallest daily limit that still finishes on time.

### 4. Real-World Analogy
Imagine boxes queued on a conveyor belt. Each morning, load the longest possible prefix that fits, then continue with the next box tomorrow. A larger truck never needs more days than a smaller truck, which creates the ordered yes/no answers needed for binary search.

### 5. Key Observation
The capacity cannot be below the heaviest package, so `max(weights)` is the lower bound. Capacity `sum(weights)` ships everything in one day, so it is the upper bound. After validating that sum as safe, every integer in this interval is a safe candidate.

Feasibility is monotonic:

```text
capacity: 10 11 12 13 14 15 16 ... 55
within 5:  F  F  F  F  F  T  T ...  T
                              ^ find the first true
```

If `days >= weights.length`, the answer is exactly the heaviest package: each package can use its own day, and no capacity below that package's weight can work.

### 6. Pattern
```text
Pattern: Binary Search on the Minimum Feasible Answer
Why this pattern applies: capacity feasibility changes once from false to true
How to recognize it: minimize a limit while a greedy ordered simulation tests the limit
Similar problems: Koko eating bananas, book allocation, split array largest sum
```

### 7. Brute Force
Validate the complete contract, derive the bounds, and test every integer capacity from the heaviest package through the total weight. For one candidate, greedily keep loading the current day until the next package would exceed the capacity, then count a new day. Return false immediately when starting that day would exceed `days`.

#### Pseudocode
```text
validate days
if weights is empty: return 0
validate every weight and safely compute lower=max and upper=sum
for capacity from lower through upper:
    greedily count sequential shipping days
    if the count stays within days: return capacity
```

#### Brute-Force TypeScript
```typescript
function shipWithinDaysBruteForce(
  weights: readonly number[],
  days: number,
): number {
  const { lower, upper } = validateShippingContract(weights, days);
  if (weights.length === 0) return 0;

  let capacity = lower;
  while (capacity <= upper) {
    if (canShipWithinDaysBrute(weights, days, capacity)) return capacity;
    if (capacity === upper) break;
    capacity += 1;
  }

  throw new RangeError("no safe capacity can satisfy the contract");
}

function validateShippingContract(
  weights: readonly number[],
  days: number,
): { lower: number; upper: number } {
  if (!Number.isSafeInteger(days) || days <= 0) {
    throw new RangeError("days must be a positive safe integer");
  }

  let lower = 0;
  let upper = 0;

  for (const weight of weights) {
    if (!Number.isSafeInteger(weight) || weight <= 0) {
      throw new RangeError("weights must contain positive safe integers");
    }
    if (weight > Number.MAX_SAFE_INTEGER - upper) {
      throw new RangeError("total weight must be a safe integer");
    }
    lower = Math.max(lower, weight);
    upper += weight;
  }

  return { lower, upper };
}

function canShipWithinDaysBrute(
  weights: readonly number[],
  days: number,
  capacity: number,
): boolean {
  let usedDays = 1;
  let currentLoad = 0;

  for (const weight of weights) {
    if (weight > capacity - currentLoad) {
      if (usedDays === Number.MAX_SAFE_INTEGER) return false;
      usedDays += 1;
      if (usedDays > days) return false;
      currentLoad = weight;
    } else {
      currentLoad += weight;
    }
  }

  return true;
}
```

#### Visualization
```text
Candidate capacities for weights [1..10], days=5:

10 -> more than 5 days -> reject
11 -> more than 5 days -> reject
12 -> more than 5 days -> reject
13 -> more than 5 days -> reject
14 -> more than 5 days -> reject
15 -> exactly 5 days   -> return 15
```

#### Dry Run
| Capacity | Greedy loads before failure or completion | Used days | Feasible? |
|---:|---|---:|:---:|
| 10 | `[1,2,3,4] [5] [6] [7] [8] [9]...` | More than 5 | No |
| 11 | `[1,2,3,4] [5,6] [7] [8] [9] [10]` | 6 | No |
| 12 | `[1,2,3,4] [5,6] [7] [8] [9] [10]` | 6 | No |
| 13 | `[1,2,3,4] [5,6] [7] [8] [9] [10]` | 6 | No |
| 14 | `[1,2,3,4] [5,6] [7] [8] [9] [10]` | 6 | No |
| 15 | `[1,2,3,4,5] [6,7] [8] [9] [10]` | 5 | Yes |

#### Brute-Force Complexity
Let `n = weights.length` and `R = sum(weights) - max(weights) + 1`. Validation and safe bound construction take `O(n)`. Each feasibility call takes `O(n)`, and up to `R` capacities are tested, so total time is `O(nR)` and extra space is `O(1)`.

### 8. Why Brute Force Is Not Ideal
The candidate interval can be enormous even when there are few packages. Testing every capacity ignores monotonicity: after one capacity works, every larger capacity also works.

### 9. Optimal Approach
Binary-search the closed interval `[max(weights), sum(weights)]` for the first feasible capacity. For each midpoint, greedily scan packages in order. If the midpoint needs more than `days`, discard it and every smaller capacity. Otherwise, keep it as a possible answer and search left.

The greedy scan fills each day as much as possible. Starting a new day earlier cannot reduce the number of days, so this scan computes the minimum number of sequential days for a fixed capacity.

### 10. Visualization
```text
weights = [1,2,3,4,5,6,7,8,9,10], days = 5
search = [10, 55]

[10 .......... 32 .......... 55]  32 works -> keep [10,32]
[10 .... 21 .... 32]               21 works -> keep [10,21]
[10 . 15 . 21]                     15 works -> keep [10,15]
[10 12 15]                         12 fails -> keep [13,15]
[13 14 15]                         14 fails -> keep [15,15]
                                           answer = 15
```

### 11. Step-by-Step Dry Run
| Low | High | Middle | Greedy days | Feasible within 5? | Update |
|---:|---:|---:|---:|:---:|---|
| 10 | 55 | 32 | 2 | Yes | `high = 32` |
| 10 | 32 | 21 | 3 | Yes | `high = 21` |
| 10 | 21 | 15 | 5 | Yes | `high = 15` |
| 10 | 15 | 12 | More than 5 | No | `low = 13` |
| 13 | 15 | 14 | More than 5 | No | `low = 15` |

`low === high === 15`, so return `15`.

### 12. TypeScript Solution
```typescript
function shipWithinDays(weights: readonly number[], days: number): number {
  const { lower, upper } = validateShippingInput(weights, days);
  if (weights.length === 0) return 0;

  let low = lower;
  let high = upper;

  while (low < high) {
    const middle = low + Math.floor((high - low) / 2);

    if (canShipWithinDays(weights, days, middle)) {
      high = middle;
    } else {
      low = middle + 1;
    }
  }

  return low;
}

function validateShippingInput(
  weights: readonly number[],
  days: number,
): { lower: number; upper: number } {
  if (!Number.isSafeInteger(days) || days <= 0) {
    throw new RangeError("days must be a positive safe integer");
  }

  let lower = 0;
  let upper = 0;

  for (const weight of weights) {
    if (!Number.isSafeInteger(weight) || weight <= 0) {
      throw new RangeError("weights must contain positive safe integers");
    }
    if (weight > Number.MAX_SAFE_INTEGER - upper) {
      throw new RangeError("total weight must be a safe integer");
    }
    lower = Math.max(lower, weight);
    upper += weight;
  }

  return { lower, upper };
}

function canShipWithinDays(
  weights: readonly number[],
  days: number,
  capacity: number,
): boolean {
  let usedDays = 1;
  let currentLoad = 0;

  for (const weight of weights) {
    if (weight > capacity - currentLoad) {
      if (usedDays === Number.MAX_SAFE_INTEGER) return false;
      usedDays += 1;
      if (usedDays > days) return false;
      currentLoad = weight;
    } else {
      currentLoad += weight;
    }
  }

  return true;
}
```

### 13. Code Walkthrough
Validation first checks `days`, so empty weights return `0` only under the promised positive-days contract. The same `O(n)` scan validates every weight, finds the heaviest package, and rejects an unsafe sum before addition. The optimized loop uses an overflow-safe midpoint. A feasible midpoint remains in the interval through `high = middle`; an infeasible one is excluded through `low = middle + 1`. The helper preserves order, never mutates `weights`, and stops as soon as another sequential day would exceed `days`.

### 14. Complexity Analysis
Let `R = sum(weights) - max(weights) + 1`. One feasibility call is `O(n)`, not `O(n log R)`. Binary search makes `O(log R)` such calls, so the search costs `O(n log R)`. Contract validation and safe bound construction add `O(n)`, giving `O(n + n log R)`, conventionally `O(n log R)` for a nontrivial range; when `R = 1`, the honest total is `O(n)`. Extra space is `O(1)`.

### 15. Edge Cases
- Empty weights with valid positive `days` return `0`.
- Empty weights with invalid `days` still throw `RangeError`.
- One package returns its weight.
- `days >= weights.length` returns `max(weights)` because each package may occupy a separate day.
- `days = 1` returns `sum(weights)`.
- An unsafe total weight throws before the addition occurs.

### 16. Common Mistakes
- Reordering packages or splitting one package across days.
- Requiring exactly `days` used days instead of allowing completion within `days`.
- Starting the lower bound at `1` instead of the heaviest package.
- Using an unsafe sum or `(low + high) / 2` midpoint.
- Adding loads before checking whether the safe capacity would be exceeded.
- Reporting a feasibility helper as `O(n log R)` instead of `O(n)`.

### 17. Interview Follow-ups
1. Return the package ranges shipped each day. 2. Require exactly `days` nonempty days. 3. Permit package reordering. 4. Add a fixed cost for each day used. 5. Process weights from a stream when the bounds are known.

### 18. Variations
- Split array largest sum. - Painter partition. - Minimum truck capacity. - Ordered workload batching. - Maximum allowed daily processing time.

### 19. Pattern Connections
This is a lower-bound search over answers rather than indexes. Its greedy predicate resembles contiguous partition problems, but the contract permits at most `days` groups. That distinction makes finishing early valid and explains why extra available days can only help.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/ship-capacity` with `{ weights, days }` and return `{ capacity, trace, truncated }`. Validate the scalar, every weight, and the safe total at the request boundary in `O(n)`. Trace only optimized states: `low`, `high`, `middle`, sequential day loads, feasibility, and the retained interval. Cap trace rows independently of the algorithm.

### 21. Mini Challenge
Extend the optimized function to return both the minimum capacity and one ordered schedule that uses at most `days` nonempty days, without mutating `weights`.

## Problem 93 — Koko Eating Bananas

### 1. Problem Statement
Given `piles: readonly number[]` and `hours`, return the minimum positive integer eating speed that finishes all piles within `hours`. Every pile and `hours` must be a positive safe integer, and the array must not be mutated.

An empty array returns `0` after validating `hours`. For nonempty input, require `hours >= piles.length`; each nonempty pile takes at least one whole hour because Koko works on only one pile during an hour. Throw `RangeError` when the contract is invalid or infeasible.

### 2. Example
```text
Input:  piles = [3, 6, 7, 11], hours = 8
Output: 4

Hours at speed 4: ceil(3/4) + ceil(6/4) + ceil(7/4) + ceil(11/4)
                = 1 + 2 + 2 + 3 = 8
```

### 3. Understand the Problem
At speed `k`, a pile of size `p` needs `ceil(p / k)` hours. Any unused part of the final hour for one pile cannot be spent on another pile. The answer is the smallest positive integer `k` whose total required hours do not exceed the available hours.

#### In Simple Terms
Choose how many bananas Koko can eat per hour. Find the slowest positive speed that still finishes every pile by the deadline.

### 4. Real-World Analogy
Think of several jobs that must each occupy their own sequence of whole time slots. Faster processing shortens some jobs, but leftover time in a job's last slot cannot be transferred. Once a speed meets the deadline, every faster speed also meets it.

### 5. Key Observation
Speed `1` is the smallest possible candidate. Speed `max(piles)` finishes every pile in one hour, so it is feasible exactly when `hours >= piles.length`, which the contract requires. Feasibility is monotonic from false to true across `1..max(piles)`.

For positive integers, compute ceiling division safely as:

```text
ceil(pile / speed) = floor((pile - 1) / speed) + 1
```

This avoids the potentially unsafe intermediate sum in `(pile + speed - 1) / speed`.

### 6. Pattern
```text
Pattern: Binary Search on the Minimum Feasible Rate
Why this pattern applies: required hours never increase as speed increases
How to recognize it: choose the smallest integer rate that meets a deadline
Similar problems: minimum shipping capacity, smallest divisor, production rate
```

### 7. Brute Force
Validate the contract, then try speeds from `1` upward. For each speed, safely compute each pile's ceiling hours. Before adding, compare against the remaining hour budget; this both prevents overflow and returns early when the speed is too slow.

#### Pseudocode
```text
validate hours
if piles is empty: return 0
validate every pile and require hours >= number of piles
for speed from 1 through max pile:
    safely total ceil(pile / speed), stopping above hours
    if total is within hours: return speed
```

#### Brute-Force TypeScript
```typescript
function minEatingSpeedBruteForce(
  piles: readonly number[],
  hours: number,
): number {
  const maxPile = validateKokoContract(piles, hours);
  if (piles.length === 0) return 0;

  let speed = 1;
  while (speed <= maxPile) {
    if (canFinishBananasBrute(piles, hours, speed)) return speed;
    if (speed === maxPile) break;
    speed += 1;
  }

  throw new RangeError("no safe speed can satisfy the contract");
}

function validateKokoContract(
  piles: readonly number[],
  hours: number,
): number {
  if (!Number.isSafeInteger(hours) || hours <= 0) {
    throw new RangeError("hours must be a positive safe integer");
  }

  let maxPile = 0;
  for (const pile of piles) {
    if (!Number.isSafeInteger(pile) || pile <= 0) {
      throw new RangeError("piles must contain positive safe integers");
    }
    maxPile = Math.max(maxPile, pile);
  }

  if (piles.length > 0 && hours < piles.length) {
    throw new RangeError("hours must be at least the number of piles");
  }

  return maxPile;
}

function canFinishBananasBrute(
  piles: readonly number[],
  hours: number,
  speed: number,
): boolean {
  let requiredHours = 0;

  for (const pile of piles) {
    const pileHours = Math.floor((pile - 1) / speed) + 1;
    if (pileHours > hours - requiredHours) return false;
    requiredHours += pileHours;
  }

  return true;
}
```

#### Visualization
```text
piles = [3,6,7,11], hours = 8

speed 1 -> 27 hours -> reject
speed 2 -> 15 hours -> reject
speed 3 -> 10 hours -> reject
speed 4 ->  8 hours -> return 4
```

#### Dry Run
| Speed | Per-pile hours | Total or early result | Feasible? |
|---:|---|---:|:---:|
| 1 | `3, 6, ...` | Exceeds 8 early | No |
| 2 | `2, 3, 4, ...` | Exceeds 8 early | No |
| 3 | `1, 2, 3, 4` | 10 | No |
| 4 | `1, 2, 2, 3` | 8 | Yes |

#### Brute-Force Complexity
Let `n = piles.length` and `M = max(piles)`. Validation takes `O(n)`. Each speed check takes `O(n)`, and as many as `M` speeds are tested, so total time is `O(nM)` and extra space is `O(1)`.

### 8. Why Brute Force Is Not Ideal
The largest pile may be close to `Number.MAX_SAFE_INTEGER`. Incrementing through every possible speed is impractical, while each result already tells whether the answer lies above or at-or-below that speed.

### 9. Optimal Approach
Binary-search speeds in the closed interval `[1, max(piles)]`. Safely total the required hours at the midpoint. If the total exceeds `hours`, the speed and every slower speed fail, so move `low` right. If it fits, preserve the midpoint and search for a slower feasible speed on the left.

### 10. Visualization
```text
piles = [3,6,7,11], hours = 8
search = [1,11]

[1 .... 6 .... 11]  speed 6 needs 6 hours -> keep [1,6]
[1 . 3 . 6]         speed 3 needs 10 hours -> keep [4,6]
[4 5 6]             speed 5 needs 8 hours -> keep [4,5]
[4 5]               speed 4 needs 8 hours -> keep [4,4]
                                                   answer = 4
```

### 11. Step-by-Step Dry Run
| Low | High | Middle speed | Per-pile hours | Total | Feasible within 8? | Update |
|---:|---:|---:|---|---:|:---:|---|
| 1 | 11 | 6 | `1,1,2,2` | 6 | Yes | `high = 6` |
| 1 | 6 | 3 | `1,2,3,4` | 10 | No | `low = 4` |
| 4 | 6 | 5 | `1,2,2,3` | 8 | Yes | `high = 5` |
| 4 | 5 | 4 | `1,2,2,3` | 8 | Yes | `high = 4` |

`low === high === 4`, so return `4`.

### 12. TypeScript Solution
```typescript
function minEatingSpeed(piles: readonly number[], hours: number): number {
  const maxPile = validateKokoInput(piles, hours);
  if (piles.length === 0) return 0;

  let low = 1;
  let high = maxPile;

  while (low < high) {
    const middle = low + Math.floor((high - low) / 2);

    if (canFinishBananas(piles, hours, middle)) {
      high = middle;
    } else {
      low = middle + 1;
    }
  }

  return low;
}

function validateKokoInput(
  piles: readonly number[],
  hours: number,
): number {
  if (!Number.isSafeInteger(hours) || hours <= 0) {
    throw new RangeError("hours must be a positive safe integer");
  }

  let maxPile = 0;
  for (const pile of piles) {
    if (!Number.isSafeInteger(pile) || pile <= 0) {
      throw new RangeError("piles must contain positive safe integers");
    }
    maxPile = Math.max(maxPile, pile);
  }

  if (piles.length > 0 && hours < piles.length) {
    throw new RangeError("hours must be at least the number of piles");
  }

  return maxPile;
}

function canFinishBananas(
  piles: readonly number[],
  hours: number,
  speed: number,
): boolean {
  let requiredHours = 0;

  for (const pile of piles) {
    const pileHours = Math.floor((pile - 1) / speed) + 1;
    if (pileHours > hours - requiredHours) return false;
    requiredHours += pileHours;
  }

  return true;
}
```

### 13. Code Walkthrough
The validator checks that `hours` is a positive safe integer, scans all piles for positive safe integers, records the upper bound, and rejects nonempty input with too few hours. The optimized search uses a safe midpoint and retains the first feasible speed. In the helper, `Math.floor((pile - 1) / speed) + 1` performs safe positive ceiling division. Comparing `pileHours` with `hours - requiredHours` before addition keeps the counter safe and enables early failure. Neither function mutates `piles`.

### 14. Complexity Analysis
Let `n = piles.length` and `M = max(piles)`. One feasibility call is `O(n)`. Binary search makes `O(log M)` calls, so its work is `O(n log M)`. Full contract validation adds `O(n)`, and the total remains `O(n log M)` for `M > 1`; when `M = 1`, the honest total is `O(n)`. Extra space is `O(1)`.

### 15. Edge Cases
- Empty piles with valid positive `hours` return `0`.
- Empty piles with invalid `hours` throw `RangeError`.
- Nonempty piles with `hours < piles.length` throw `RangeError` as infeasible.
- `hours === piles.length` returns `max(piles)`.
- One pile returns `ceil(pile / hours)` through the same search.
- Safe-integer piles near the numeric limit do not use unsafe addition for ceiling division.

### 16. Common Mistakes
- Using ordinary division instead of ceiling division.
- Using `(pile + speed - 1)` and overflowing the safe-integer range.
- Letting Koko use the remainder of an hour on another pile.
- Accepting `hours < piles.length` even though no positive speed can help.
- Summing required hours before checking the remaining safe budget.
- Binary-searching from `0`, which permits division by zero.

### 17. Interview Follow-ups
1. Return the hours spent per pile. 2. Allow switching piles within an hour. 3. Minimize cost when faster speeds cost more. 4. Find the minimum common deadline. 5. Support several workers eating different piles in parallel.

### 18. Variations
- Smallest divisor under a threshold. - Minimum production rate. - Train speed before a deadline. - Server throughput sizing. - Minimum machine capacity.

### 19. Pattern Connections
Like shipping capacity, this searches the first true answer in a monotonic numeric domain. The feasibility arithmetic differs: shipping forms sequential groups, while Koko sums independent ceiling-divided durations and cannot combine their fractional remainders.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/koko-speed` with `{ piles, hours }` and return `{ speed, trace, truncated }`. Validate `hours`, every pile, and the feasibility condition `hours >= piles.length` in `O(n)` overall. Trace only optimized `low`, `high`, midpoint speed, safe per-pile hours, early deadline failure, and the retained interval.

### 21. Mini Challenge
Return the minimum speed and a `readonly number[]` containing the hours spent on each pile, while preserving safe arithmetic and `O(n log maxPile)` total time.

## Problem 94 — Allocate Minimum Number of Pages

### 1. Problem Statement
Given book sizes `pages: readonly number[]` and a positive integer `students`, assign every book exactly once, in order, to exactly `students` contiguous nonempty groups. Every page count must be a positive safe integer. Return the minimum possible value of the largest page total assigned to any student, without mutating the array.

The books must be nonempty and `1 <= students <= pages.length`; otherwise throw `RangeError`. Also throw for an invalid scalar, an invalid page count, or a total page count that is not a safe integer.

### 2. Example
```text
Input:  pages = [12, 34, 67, 90], students = 2
Output: 113

Best allocation: [12, 34, 67] [90]
Student totals:   113          90
```

### 3. Understand the Problem
Every student must receive at least one whole book, every book must be assigned, and book order must be preserved. For an allocation, measure the largest student total. Among all valid allocations into exactly `students` groups, return the smallest possible largest total.

#### In Simple Terms
Split the ordered shelf into exactly the requested number of nonempty consecutive parts. Make the busiest student's page total as small as possible.

### 4. Real-World Analogy
Divide an ordered reading list among students by placing separators between books. You cannot reorder books or leave a student empty. Raising the allowed workload makes more separator placements possible, so feasible workload limits form a monotonic range.

### 5. Key Observation
The answer is at least the largest single book and at most the sum of all pages. For a proposed limit, greedily give each student the longest prefix that fits. This produces the fewest contiguous groups possible under that limit.

The feasibility helper accepts a limit when it needs at most `students` groups. This still satisfies the exact-student contract: if it uses fewer groups and `students <= pages.length`, repeatedly split a group containing multiple books. Each split creates nonempty contiguous groups and cannot increase either group's page total. Shipping differs because its contract merely allows finishing within the day limit; here exact groups are recovered by this splitting argument.

### 6. Pattern
```text
Pattern: Binary Search on a Contiguous Partition Limit
Why this pattern applies: a larger maximum workload never makes allocation harder
How to recognize it: minimize the largest sum among ordered nonempty groups
Similar problems: painter partition, split array largest sum, ship capacity
```

### 7. Brute Force
Validate the exact-student contract and safe sum, then try every integer page limit from the largest book through the total. For each candidate, greedily count contiguous groups, returning false as soon as the count exceeds `students`. The first feasible limit is optimal.

#### Pseudocode
```text
validate students and require nonempty books with students <= book count
validate every page count and safely compute lower=max and upper=sum
for limit from lower through upper:
    greedily count contiguous nonempty groups
    if at most students groups are needed: return limit
```

#### Brute-Force TypeScript
```typescript
function allocateMinimumPagesBruteForce(
  pages: readonly number[],
  students: number,
): number {
  const { lower, upper } = validatePageContract(pages, students);

  let limit = lower;
  while (limit <= upper) {
    if (canAllocatePagesBrute(pages, students, limit)) return limit;
    if (limit === upper) break;
    limit += 1;
  }

  throw new RangeError("no safe allocation can satisfy the contract");
}

function validatePageContract(
  pages: readonly number[],
  students: number,
): { lower: number; upper: number } {
  if (!Number.isSafeInteger(students) || students <= 0) {
    throw new RangeError("students must be a positive safe integer");
  }
  if (pages.length === 0 || students > pages.length) {
    throw new RangeError("require 1 <= students <= pages.length");
  }

  let lower = 0;
  let upper = 0;

  for (const pageCount of pages) {
    if (!Number.isSafeInteger(pageCount) || pageCount <= 0) {
      throw new RangeError("pages must contain positive safe integers");
    }
    if (pageCount > Number.MAX_SAFE_INTEGER - upper) {
      throw new RangeError("total pages must be a safe integer");
    }
    lower = Math.max(lower, pageCount);
    upper += pageCount;
  }

  return { lower, upper };
}

function canAllocatePagesBrute(
  pages: readonly number[],
  students: number,
  limit: number,
): boolean {
  let usedStudents = 1;
  let currentPages = 0;

  for (const pageCount of pages) {
    if (pageCount > limit - currentPages) {
      if (usedStudents === Number.MAX_SAFE_INTEGER) return false;
      usedStudents += 1;
      if (usedStudents > students) return false;
      currentPages = pageCount;
    } else {
      currentPages += pageCount;
    }
  }

  return true;
}
```

#### Visualization
```text
pages = [12,34,67,90], students = 2

limit  90 -> needs more than 2 groups -> reject
limit  91 -> needs more than 2 groups -> reject
...
limit 112 -> needs more than 2 groups -> reject
limit 113 -> [12,34,67] [90]         -> return 113
```

#### Dry Run
| Limit | Greedy groups | Groups needed | Feasible for exactly 2? |
|---:|---|---:|:---:|
| 90 | `[12,34] [67] [90]` | 3 | No |
| 100 | `[12,34] [67] [90]` | 3 | No |
| 112 | `[12,34] [67] [90]` | 3 | No |
| 113 | `[12,34,67] [90]` | 2 | Yes |

#### Brute-Force Complexity
Let `n = pages.length` and `R = sum(pages) - max(pages) + 1`. Validation takes `O(n)`. Each feasibility call takes `O(n)`, and up to `R` limits are tested, so total time is `O(nR)` and extra space is `O(1)`.

### 8. Why Brute Force Is Not Ideal
Page totals can create a huge candidate range. Testing every integer limit repeats nearly identical greedy scans even though one failed limit proves that every smaller limit also fails.

### 9. Optimal Approach
Binary-search the first feasible limit in `[max(pages), sum(pages)]`. Greedily count the minimum contiguous groups needed at the midpoint. If more than `students` groups are required, move right. Otherwise, the midpoint can support exactly `students` nonempty groups by splitting, so preserve it and search left.

### 10. Visualization
```text
pages = [12,34,67,90], students = 2
search = [90,203]

[90 ........ 146 ........ 203]  146 uses 2 groups -> keep [90,146]
[90 ... 118 ... 146]            118 uses 2 groups -> keep [90,118]
[90 . 104 . 118]                104 uses 3 groups -> keep [105,118]
[105 111 118]                   111 uses 3 groups -> keep [112,118]
[112 115 118]                   115 uses 2 groups -> keep [112,115]
[112 113 115]                   113 uses 2 groups -> keep [112,113]
[112 113]                       112 uses 3 groups -> keep [113,113]
                                                        answer = 113
```

### 11. Step-by-Step Dry Run
| Low | High | Middle limit | Greedy grouping | Groups | Feasible? | Update |
|---:|---:|---:|---|---:|:---:|---|
| 90 | 203 | 146 | `[12,34,67] [90]` | 2 | Yes | `high = 146` |
| 90 | 146 | 118 | `[12,34,67] [90]` | 2 | Yes | `high = 118` |
| 90 | 118 | 104 | `[12,34] [67] [90]` | 3 | No | `low = 105` |
| 105 | 118 | 111 | `[12,34] [67] [90]` | 3 | No | `low = 112` |
| 112 | 118 | 115 | `[12,34,67] [90]` | 2 | Yes | `high = 115` |
| 112 | 115 | 113 | `[12,34,67] [90]` | 2 | Yes | `high = 113` |
| 112 | 113 | 112 | `[12,34] [67] [90]` | 3 | No | `low = 113` |

`low === high === 113`, so return `113`.

### 12. TypeScript Solution
```typescript
function allocateMinimumPages(
  pages: readonly number[],
  students: number,
): number {
  const { lower, upper } = validatePageInput(pages, students);
  let low = lower;
  let high = upper;

  while (low < high) {
    const middle = low + Math.floor((high - low) / 2);

    if (canAllocatePages(pages, students, middle)) {
      high = middle;
    } else {
      low = middle + 1;
    }
  }

  return low;
}

function validatePageInput(
  pages: readonly number[],
  students: number,
): { lower: number; upper: number } {
  if (!Number.isSafeInteger(students) || students <= 0) {
    throw new RangeError("students must be a positive safe integer");
  }
  if (pages.length === 0 || students > pages.length) {
    throw new RangeError("require 1 <= students <= pages.length");
  }

  let lower = 0;
  let upper = 0;

  for (const pageCount of pages) {
    if (!Number.isSafeInteger(pageCount) || pageCount <= 0) {
      throw new RangeError("pages must contain positive safe integers");
    }
    if (pageCount > Number.MAX_SAFE_INTEGER - upper) {
      throw new RangeError("total pages must be a safe integer");
    }
    lower = Math.max(lower, pageCount);
    upper += pageCount;
  }

  return { lower, upper };
}

function canAllocatePages(
  pages: readonly number[],
  students: number,
  limit: number,
): boolean {
  let usedStudents = 1;
  let currentPages = 0;

  for (const pageCount of pages) {
    if (pageCount > limit - currentPages) {
      if (usedStudents === Number.MAX_SAFE_INTEGER) return false;
      usedStudents += 1;
      if (usedStudents > students) return false;
      currentPages = pageCount;
    } else {
      currentPages += pageCount;
    }
  }

  return true;
}
```

### 13. Code Walkthrough
The validator enforces a positive safe-integer student count, nonempty books, enough books for nonempty groups, positive safe page counts, and a safe total before binary search begins. The optimized helper greedily minimizes the number of ordered groups for a limit and stops when that number exceeds `students`. A result using fewer students is still valid for the exact contract because a multi-book group can be split until exactly `students` groups exist. Safe subtraction checks each proposed addition, and `pages` is never mutated.

### 14. Complexity Analysis
Let `R = sum(pages) - max(pages) + 1`. One greedy feasibility call is `O(n)`. Binary search performs `O(log R)` calls, so the search is `O(n log R)`. The required validation scan adds `O(n)`, making total time `O(n + n log R)`, conventionally `O(n log R)` for a nontrivial range; when `R = 1`, total time is `O(n)`. Extra space is `O(1)`.

### 15. Edge Cases
- Empty books throw `RangeError`; unlike shipping, there is no empty allocation result.
- `students = 0`, a non-integer, or an unsafe integer throws `RangeError`.
- `students > pages.length` throws because every student needs a book.
- `students = 1` returns `sum(pages)`.
- `students = pages.length` returns `max(pages)`.
- An unsafe total page count throws before addition.

### 16. Common Mistakes
- Allowing empty student groups or assigning one book more than once.
- Reordering books instead of preserving contiguous input order.
- Rejecting a candidate because the greedy scan uses fewer than exactly `students` groups.
- Copying shipping's empty-input behavior into this nonempty contract.
- Using an unsafe total or unchecked load addition.
- Reporting the `O(n)` helper itself as `O(n log R)`.

### 17. Interview Follow-ups
1. Return the exact book ranges for each student. 2. Count optimal allocations. 3. Allow noncontiguous assignments. 4. Minimize the difference between largest and smallest loads. 5. Give students different reading speeds.

### 18. Variations
- Painter partition. - Split array largest sum. - Ordered job scheduling. - Contiguous file placement. - Shipping with exactly `D` nonempty days.

### 19. Pattern Connections
This shares shipping's greedy contiguous grouping predicate and answer bounds. The contracts differ: shipping accepts any schedule using at most `days`, including an early finish; page allocation requires exactly `students` nonempty groups. The condition `students <= pages.length` makes every at-most grouping splittable into an exact grouping without increasing the maximum load.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/allocate-pages` with `{ pages, students }` and return `{ minimumMaximumPages, trace, truncated }`. Validate the exact-student contract, every page count, and the safe total in `O(n)`. Trace only optimized `low`, `high`, midpoint limit, greedy contiguous groups, feasibility, and the retained interval; explain exact-group recovery in response metadata rather than tracing a brute-force search.

### 21. Mini Challenge
Extend the optimized solution to return exactly `students` contiguous nonempty index ranges whose maximum page total equals the minimum limit, including the case where the greedy feasibility scan initially uses fewer groups.

## Problem 95 — Median of Two Sorted Arrays

### 1. Problem Statement
Given two readonly arrays of safe integers sorted in nondecreasing order, return the exact mathematical median of all their values without mutating either array. At least one combined value is required; if both arrays are empty, throw `RangeError`. Also throw `RangeError` if an even-length median is a half-integer that JavaScript `number` cannot represent exactly.

Sorted order and safe-integer element shape are caller preconditions. The optimized search does not scan either array to revalidate them. Duplicates and negative values are allowed. The result is a `number`, and an even combined length may produce a half-integer such as `2.5`.

### 2. Example
```text
Input:  first = [1, 3], second = [2]
Output: 2

Combined order: [1, 2, 3]
Middle value:        2
```

For `first = [1, 2]` and `second = [3, 4]`, the two middle values are `2` and `3`, so the median is `2 / 2 + 3 / 2 = 2.5`.

### 3. Understand the Problem
The median depends on the middle position or positions in the combined sorted order, not on which input supplied them. Merging reveals that order directly, but the optimized solution must find the same boundary while examining only logarithmically many partitions.

#### In Simple Terms
Place a divider in each sorted array so the combined left half has the right size and every value on the left is no greater than every value on the right. The values touching those dividers determine the median.

### 4. Real-World Analogy
Imagine two already sorted queues whose members must remain in place. Instead of forming one new queue, choose how many people to take from the front of each queue. When everyone chosen for the left group ranks no higher than anyone left in the right group, the boundary identifies the middle rank.

### 5. Key Observation
If the left half must contain `floor((m + n + 1) / 2)` values, choosing `partitionA` fixes `partitionB`. A partition is valid exactly when `leftA <= rightB` and `leftB <= rightA`. If `leftA > rightB`, the partition in the first array is too far right; otherwise it is too far left.

Binary-searching only the shorter array keeps every derived partition valid and gives the tight bound `O(log min(m, n))`. Edge partitions use `-Infinity` and `Infinity` as comparison sentinels; these are not inserted into either input.

### 6. Pattern
```text
Pattern: Binary Search on Complementary Partitions
Why this pattern applies: choosing one partition determines the other
How to recognize it: two sorted inputs and a combined rank or median
Similar problems: kth element of two sorted arrays, weighted rank boundaries
```

### 7. Brute Force
Merge the two sorted arrays into a new sorted array with two read indexes. Then return the middle value for an odd length or average the two middle values for an even length. This preserves the sorted-input advantage; concatenating and sorting again would unnecessarily cost `O((m + n) log(m + n))`.

#### Pseudocode
```text
if both arrays are empty: throw RangeError
merge by repeatedly taking the smaller current value
append any remaining values from either array
if merged length is odd: return its middle value
form the two-middle-value numerator exactly with bigint
if its half cannot be represented exactly as number: throw RangeError
return the exact average
```

#### Brute-Force TypeScript
```typescript
function medianOfTwoSortedArraysBruteForce(
  first: readonly number[],
  second: readonly number[],
): number {
  if (first.length === 0 && second.length === 0) {
    throw new RangeError("at least one combined value is required");
  }

  const merged: number[] = [];
  let firstIndex = 0;
  let secondIndex = 0;

  while (firstIndex < first.length && secondIndex < second.length) {
    if (first[firstIndex] <= second[secondIndex]) {
      merged.push(first[firstIndex]);
      firstIndex += 1;
    } else {
      merged.push(second[secondIndex]);
      secondIndex += 1;
    }
  }

  while (firstIndex < first.length) {
    merged.push(first[firstIndex]);
    firstIndex += 1;
  }
  while (secondIndex < second.length) {
    merged.push(second[secondIndex]);
    secondIndex += 1;
  }

  const right = Math.floor(merged.length / 2);
  if (merged.length % 2 === 1) return merged[right];

  const left = right - 1;
  const numerator = BigInt(merged[left]) + BigInt(merged[right]);
  if (numerator % 2n !== 0n &&
      (numerator < BigInt(Number.MIN_SAFE_INTEGER) ||
       numerator > BigInt(Number.MAX_SAFE_INTEGER))) {
    throw new RangeError("median cannot be represented exactly as a number");
  }
  return Number(numerator) / 2;
}
```

The `bigint` numerator preserves the exact sum of the two safe integers. An even numerator converts to an exactly representable integer median. An odd numerator represents a half-integer; it is converted only when its magnitude fits the range where JavaScript can represent `.5` exactly. This avoids both unsafe addition and silent rounding.

#### Visualization
```text
first:  [1, 3]
second: [2]

take 1 from first  -> [1]
take 2 from second -> [1, 2]
take 3 from first  -> [1, 2, 3]
                              ^ median
```

#### Dry Run
| First index | Second index | Chosen value | Merged |
|---:|---:|---:|---|
| 0 | 0 | 1 | `[1]` |
| 1 | 0 | 2 | `[1, 2]` |
| 1 | 1 | 3 | `[1, 2, 3]` |

The merged length is odd, so index `floor(3 / 2) = 1` contains the median `2`.

#### Brute-Force Complexity
Let `m = first.length` and `n = second.length`. Each value is copied once, so time is `O(m + n)` and the merged array uses `O(m + n)` extra space.

### 8. Why Brute Force Is Not Ideal
The merged array stores every value even though only the one or two values at the middle boundary are needed. For large already sorted inputs, this gives up both the logarithmic search opportunity and constant auxiliary space.

### 9. Optimal Approach
First ensure the binary search runs on the shorter array by delegating once with the references swapped when necessary. Let that array have length `m` and the other have length `n`. Set `totalLeft = floor((m + n + 1) / 2)`, then binary-search `partitionA` from `0` through `m`; derive `partitionB = totalLeft - partitionA`.

Use edge sentinels when a partition has no value on one side. When both cross-boundary inequalities hold, return `max(leftA, leftB)` for an odd total. For an even total, form `maxLeft + minRight` as a `bigint`, verify that halving it is exactly representable as a `number`, and return that exact average. If `leftA > rightB`, move `high` left; otherwise move `low` right.

### 10. Visualization
```text
first is delegated to the shorter input:

A = [2]       partitionA = 1      [2 | +Infinity]
B = [1, 3]    partitionB = 1      [1 | 3]

combined left size = 2
leftA = 2, rightA = +Infinity
leftB = 1, rightB = 3

2 <= 3 and 1 <= +Infinity -> valid partition
odd total -> max(2, 1) = 2
```

### 11. Step-by-Step Dry Run
For inputs `[1, 3]` and `[2]`, the first call delegates with swapped references so `A = [2]`, `B = [1, 3]`, `m = 1`, `n = 2`, and `totalLeft = 2`.

| Low | High | partitionA | partitionB | leftA | rightA | leftB | rightB | Result |
|---:|---:|---:|---:|---:|---:|---:|---:|---|
| 0 | 1 | 0 | 2 | `-Infinity` | 2 | 3 | `Infinity` | `leftB > rightA`, move `low = 1` |
| 1 | 1 | 1 | 1 | 2 | `Infinity` | 1 | 3 | Valid; odd median is `max(2, 1) = 2` |

For `[1, 2]` and `[3, 4]`, the valid boundary has `maxLeft = 2` and `minRight = 3`, so the even median is `2 / 2 + 3 / 2 = 2.5`.

### 12. TypeScript Solution
```typescript
function medianOfTwoSortedArrays(
  first: readonly number[],
  second: readonly number[],
): number {
  if (first.length > second.length) {
    return medianOfTwoSortedArrays(second, first);
  }
  if (first.length === 0 && second.length === 0) {
    throw new RangeError("at least one combined value is required");
  }

  const firstLength = first.length;
  const secondLength = second.length;
  const totalLength = firstLength + secondLength;
  const totalLeft = Math.floor((totalLength + 1) / 2);
  let low = 0;
  let high = firstLength;

  while (low <= high) {
    const partitionA = low + Math.floor((high - low) / 2);
    const partitionB = totalLeft - partitionA;

    const leftA = partitionA === 0 ? -Infinity : first[partitionA - 1];
    const rightA =
      partitionA === firstLength ? Infinity : first[partitionA];
    const leftB = partitionB === 0 ? -Infinity : second[partitionB - 1];
    const rightB =
      partitionB === secondLength ? Infinity : second[partitionB];

    if (leftA <= rightB && leftB <= rightA) {
      const maxLeft = Math.max(leftA, leftB);
      if (totalLength % 2 === 1) return maxLeft;

      const minRight = Math.min(rightA, rightB);
      const numerator = BigInt(maxLeft) + BigInt(minRight);
      if (numerator % 2n !== 0n &&
          (numerator < BigInt(Number.MIN_SAFE_INTEGER) ||
           numerator > BigInt(Number.MAX_SAFE_INTEGER))) {
        throw new RangeError("median cannot be represented exactly as a number");
      }
      return Number(numerator) / 2;
    }

    if (leftA > rightB) {
      high = partitionA - 1;
    } else {
      low = partitionA + 1;
    }
  }

  throw new Error("inputs must be sorted in nondecreasing order");
}
```

### 13. Code Walkthrough
The initial delegation changes only which references occupy the two parameters; it does not copy or mutate either array and can occur at most once. Because `m <= n`, every `partitionA` in `[0, m]` produces a `partitionB` in `[0, n]`. The loop then checks only the four values adjacent to the two partitions.

The valid-partition inequalities prove that every value in the combined left half is no greater than every value in the combined right half. The extra left value created by `(totalLength + 1) / 2` handles odd totals, while the even case uses the two boundary values. Its `bigint` numerator avoids overflow and lets the function reject an unrepresentable half-integer instead of rounding it. If the loop cannot find a partition, the documented sorted-order precondition was violated, so the final `Error` represents an unreachable state for valid inputs.

### 14. Complexity Analysis
Let `m` and `n` be the input lengths. The merge solution takes `O(m + n)` time and space. The partition solution searches only the shorter input, taking `O(log min(m, n))` time and `O(1)` auxiliary space; the one delegation adds only constant stack depth.

These search bounds assume trusted caller preconditions. Fully checking safe-integer element shape and nondecreasing order would separately cost `O(m + n)`, making end-to-end time linear. The expression `m + n` is safe for practical JavaScript array lengths; this is an array-length fact, not a claim that arbitrary safe-integer addition cannot overflow.

### 15. Edge Cases
- If both arrays are empty, throw `RangeError`.
- If one array is empty, the sentinels expose the middle value or values of the nonempty array.
- Duplicates can touch either side of a valid partition because the checks use `<=`.
- Negative values work without special handling; for example, `[-5, -1]` and `[-3]` have median `-3`.
- Extreme safe integers do not overflow an intermediate sum because the numerator uses `bigint`.
- JavaScript `number` cannot represent every mathematical half-integer near `Number.MAX_SAFE_INTEGER`; for example, `[MAX_SAFE_INTEGER - 1]` and `[MAX_SAFE_INTEGER]` cause `RangeError` rather than a rounded answer.

### 16. Common Mistakes
- Binary-searching the longer array and allowing the derived partition to leave the other array's bounds.
- Using `floor((m + n) / 2)` for the left size without adjusting the odd-length return logic.
- Reversing the movement rule when `leftA > rightB`.
- Using strict `<` comparisons and rejecting valid duplicate boundaries.
- Reading `array[-1]` or `array[array.length]` instead of using sentinels.
- Computing `(maxLeft + minRight) / 2` and risking an unsafe intermediate addition.
- Silently rounding a half-integer median that `number` cannot represent exactly.
- Claiming end-to-end logarithmic time while scanning both arrays to validate sortedness.

### 17. Interview Follow-ups
1. Return the kth smallest combined value. 2. Generalize to more than two sorted arrays. 3. Support descending inputs. 4. Accept iterators instead of random-access arrays. 5. Return an exact rational object instead of throwing for an unrepresentable half-integer.

### 18. Variations
- Kth element of two sorted arrays. - Median of two sorted ranges. - Quantile selection across sorted shards. - Weighted medians. - Streaming median with two heaps.

### 19. Pattern Connections
Like lower-bound search, partition search looks for a boundary rather than an exact target value. Like answer-space search, it relies on a monotonic direction: a partition with `leftA > rightB` is too far right, while the opposite crossing is too far left. Unlike rotated-array and peak searches, the second array supplies a complementary partition whose position is determined by the first.

### 20. NestJS Project Connection
Expose `POST /api/dsa/binary-search/median-two-sorted` with `{ first, second }`. Keep request validation and search timing separate: an untrusted DTO or validation service scans both arrays in `O(m + n)` to enforce safe integers, nondecreasing order, and a nonempty combined input, then passes trusted readonly arrays to the `O(log min(m, n))` search. Trace only optimized partition states: `low`, `high`, `partitionA`, `partitionB`, the four boundary values, the movement decision, and the final median. Represent infinite sentinels with explicit labels in JSON rather than non-JSON numeric values, and map an unrepresentable exact median to a clear validation error.

### 21. Mini Challenge
Adapt the optimized partition search to return the kth smallest value for any safe integer `k` from `1` through `m + n`, preserving `O(log min(m, n))` time and handling an empty input array.

# Topic Summary

## What You Learned

Binary search is a proof-driven way to discard candidates. Exact search uses sorted values; boundary search preserves the side where the first or last valid position can still exist. Rotated arrays retain a sorted side, peaks provide a direction through neighboring values, and answer-space problems replace direct comparisons with monotonic feasibility predicates.

You also learned to search complementary partitions across two sorted arrays. Across every form, correctness depends on a clear interval invariant, justified bounds, updates that remove the current midpoint or shrink toward a boundary, and a halving argument that proves logarithmic iteration count. API complexity must distinguish trusted preconditions from linear validation, and arithmetic must avoid unsafe sums, unsafe ceiling formulas, and unrepresentable promises.

## Patterns You Should Recognize

- **Exact search:** return a matching index from a sorted array or report absence.
- **Lower and upper boundaries:** find insertion points, duplicate edges, counts, floors, and ceilings.
- **Rotated sorted structure:** identify the sorted side or locate the rotation pivot.
- **Neighbor-guided search:** follow an ascent or descent while preserving at least one peak.
- **First feasible answer:** binary-search capacity, speed, or maximum load using a monotonic predicate.
- **Complementary partitions:** choose a cut in the shorter sorted array and derive the cut in the other.
- **Trusted search boundary:** validate untrusted data separately when validation would dominate the advertised search cost.

## Complexity Cheat Sheet

| Problem | Pattern | Optimized time | Auxiliary space |
|---:|---|---|---|
| 81 | Exact binary search | `O(log n)` | `O(1)` |
| 82 | Lower bound / insertion position | `O(log n)` | `O(1)` |
| 83 | First occurrence | `O(log n)` | `O(1)` |
| 84 | Last occurrence | `O(log n)` | `O(1)` |
| 85 | Count from two boundaries | `O(log n)` | `O(1)` |
| 86 | Floor and ceiling boundaries | `O(log n)` | `O(1)` |
| 87 | Integer square-root answer search | `O(log x)` | `O(1)` |
| 88 | Search rotated sorted array | `O(log n)` | `O(1)` |
| 89 | Minimum in rotated sorted array | `O(log n)` | `O(1)` |
| 90 | Rotation count from pivot | `O(log n)` | `O(1)` |
| 91 | Neighbor-guided peak search | `O(log n)` | `O(1)` |
| 92 | Shipping capacity predicate | `O(n log R)` | `O(1)` |
| 93 | Koko eating-speed predicate | `O(n log H)` | `O(1)` |
| 94 | Contiguous allocation predicate | `O(n log R)` | `O(1)` |
| 95 | Two-array partition search | `O(log min(m, n))` | `O(1)` |

The table describes optimized search after its stated preconditions are established. Required validation scans add their own cost, commonly `O(n)` or `O(m + n)`.

## Common Interview Traps

- Mixing closed `[low, high]` and half-open `[low, high)` update rules.
- Returning immediately on a duplicate when the first or last occurrence is required.
- Claiming `O(log n)` total time after performing an `O(n)` sortedness scan.
- Applying distinct-value rotated-array logic when duplicates can hide the sorted side.
- Guessing answer bounds instead of proving an impossible lower bound and feasible upper bound.
- Binary-searching a predicate without proving that its truth values are monotonic.
- Writing ceiling division as `(value + divisor - 1) / divisor` when the addition can overflow.
- Treating an at-most-groups feasibility result as proof of an exact-groups contract without a splitting argument.
- Searching median partitions in the longer array or deriving an out-of-range complementary partition.
- Averaging middle values with an unsafe intermediate sum or promising precision a JavaScript `number` cannot represent.

## Practice Checklist

- [ ] 81. Implement exact binary search with a closed interval.
- [ ] 82. Derive search insert position as a lower bound.
- [ ] 83. Find the first occurrence without scanning left.
- [ ] 84. Find the last occurrence without scanning right.
- [ ] 85. Count duplicates from lower and upper bounds.
- [ ] 86. Return floor and ceiling with explicit missing values.
- [ ] 87. Compute integer square root with safe comparisons.
- [ ] 88. Search a rotated sorted array by identifying the sorted side.
- [ ] 89. Find the minimum value at a rotation pivot.
- [ ] 90. Convert the pivot index into a rotation count.
- [ ] 91. Find a peak through neighboring-value direction.
- [ ] 92. Minimize shipping capacity with a feasibility predicate.
- [ ] 93. Minimize Koko's speed with safe ceiling division.
- [ ] 94. Minimize the largest exact-group page allocation.
- [ ] 95. Find the median with complementary partitions.