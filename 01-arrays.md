# Arrays

## Learning Objectives

By the end of this topic, you should understand:

- how arrays organize ordered values and provide direct indexed access
- why reading by index is constant time while front insertion and deletion are linear
- how to traverse, compare, swap, rotate, partition, and compact arrays
- when sorted order enables simpler or faster algorithms
- how to distinguish in-place mutation from creating a new result
- how to derive time and auxiliary-space complexity from array operations
- how to recognize traversal, two-pointer, partition, cyclic-placement, and prefix-state patterns
- how to produce execution steps for array visualizations in a NestJS backend

## Prerequisites

- the contracts, invariants, dry runs, and Big-O reasoning from `00-dsa-fundamentals.md`
- TypeScript arrays, loops, functions, conditionals, `Map`, and `Set`
- comfort with zero-based indexing

## Core Theory

### What is an array?

An **array** is an ordered collection whose elements are reached by numeric position, called an **index**.

```text
Index:     0       1       2       3
        +-------+-------+-------+-------+
Value:  |  10   |  20   |  30   |  40   |
        +-------+-------+-------+-------+
```

The first element is at index `0`; an array of length `n` ends at index `n - 1`.

### Why do arrays exist?

Arrays group related values while preserving order. They make these jobs natural:

- read or replace a value at a known position
- visit values from left to right or right to left
- represent sequences such as prices, scores, events, or pixels
- use neighboring positions as part of an algorithm

A row of numbered lockers is a useful analogy. Knowing locker `17` lets you go directly to it. Finding which locker contains a particular unlabelled item may still require checking many lockers.

### When should I use an array?

Use an array when:

- order matters
- positions have meaning
- most processing is sequential
- direct indexed access is useful
- the collection is naturally list-shaped

Do not choose an array automatically when the main operation is repeated lookup by a business key, frequent removal from the front, or representing arbitrary connections. A `Map`, queue implementation, tree, or graph may fit those operations better.

### What does an array look like internally?

In a traditional fixed-size array, elements occupy contiguous memory locations. If each element has a fixed width, the address of index `i` can be calculated from the start address:

```text
address(i) = start + i x elementWidth
```

That calculation does not depend on array length, which explains `O(1)` indexed access.

JavaScript arrays are dynamic and engine-managed rather than simple fixed-width memory blocks. For DSA reasoning, we still use the standard dynamic-array cost model unless a problem states otherwise.

### Length, valid indexes, and boundaries

For `values.length = n`:

```text
first valid index = 0
last valid index  = n - 1
invalid boundary  = n
```

This is why a full forward traversal normally uses:

```typescript
for (let index = 0; index < values.length; index += 1) {
  // values[index] is valid here.
}
```

Using `index <= values.length` adds one invalid position and is a common off-by-one error.

### Mutation versus copying

An in-place array algorithm changes the caller's array:

```typescript
values.reverse();
```

A non-mutating version first creates a copy:

```typescript
const reversed = [...values].reverse();
```

The copy takes `O(n)` time and `O(n)` output storage because all `n` values are duplicated. State mutation behavior as part of the contract.

### Array invariants

Many array solutions divide the data into meaningful regions:

```text
[ processed and correct | current | not processed yet ]
```

An invariant describes those regions. For example:

> Before processing index `i`, `largest` is the largest value among indexes `0` through `i - 1`.

Naming the invariant makes pointer movement and boundary choices easier to verify.

## Visual Mental Model

### Index as an address label

```text
          index tells us where to go
                    |
                    v
Index:   0    1    2    3    4
       +----+----+----+----+----+
Value: | 12 |  7 | 19 |  4 | 11 |
       +----+----+----+----+----+
                    ^
                 values[2]
```

### Inserting near the front

To create room at index `1`, later values shift right:

```text
Before: [A, B, C, D]
            insert X

Shift:  [A, B, B, C, D]
After:  [A, X, B, C, D]
```

Up to `n` values move, so front or middle insertion is `O(n)`.

### Removing near the front

```text
Before: [A, B, C, D]
Remove A

Shift:  [B, C, D, _]
After:  [B, C, D]
```

This explains why repeated TypeScript `shift()` calls can be expensive.

### Swapping two positions

```text
Before: [4, 9, 2, 7]
         ^        ^
        left    right

After:  [7, 9, 2, 4]
```

A swap changes two positions with a fixed amount of work and constant auxiliary space.

## Complexity Concepts

Let `n` be the array length.

| Operation | Typical time | Why? |
|---|---:|---|
| Read or replace `values[i]` | `O(1)` | The index identifies one position directly. |
| Traverse all values | `O(n)` | Every one of `n` values is visited. |
| Search unsorted values | `O(n)` | The target may be last or absent. |
| Append with `push` | Amortized `O(1)` | Most appends are direct; occasional resizing copies values. |
| Remove with `pop` | `O(1)` | No later values need to shift. |
| Insert or remove at front | `O(n)` | Existing values may be re-indexed. |
| Copy with spread | `O(n)` | Every value is copied. |
| Numeric sort | Usually `O(n log n)` | Sorting performs many comparisons and rearrangements. |

### Why one traversal is linear

```text
n elements
x one constant-time visit per element
= n units of work
= O(n)
```

Two sequential traversals are still linear:

```text
n + n = 2n -> O(n)
```

### Why comparing every pair is quadratic

```text
For each of n values
compare with up to n values

n x n = n^2 -> O(n^2)
```

### Output space versus auxiliary space

Returning an array of `n` results requires `O(n)` output space. An algorithm can still use `O(1)` auxiliary space if its working variables do not grow with `n`. This chapter states both when the distinction matters.

## Common Patterns

### Traversal and accumulation

Carry a small state such as a largest value, count, or running result while visiting each element once.

### Two pointers

Use two indexes for two meaningful regions or directions. Typical uses include reversing, compacting duplicates, moving zeroes, and partitioning signs.

### Hash-based membership

Use a `Set` or `Map` when repeated membership or frequency checks would otherwise rescan the array.

### Cyclic placement

When values are guaranteed to belong to a known index-like range, place or reason about each value at its expected position. This can avoid extra lookup storage, but it often mutates input and requires strict constraints.

### Prefix state

Remember the best information from everything to the left. Maximum difference and stock-style problems often need a running minimum plus the best answer seen so far.

### Reverse to rotate

A rotation can be built from segment reversals. Reversal is useful because each swap fixes two positions using constant auxiliary space.

---

# Problems

## Problem 1 — Find the Largest Element in an Array

### 1. Problem Statement

Given a non-empty array of numbers, return its largest element without sorting the array.

### 2. Example

```text
Input:  [3, 8, 2, 11, 5]
Output: 11
```

### 3. Understand the Problem

We need the largest value, not its index. Duplicates are allowed, and the input order must remain unchanged. The non-empty precondition lets the first element initialize the answer safely.

#### In Simple Terms

Look through the numbers and give back the biggest value. Do not rearrange the original array.

### 4. Real-World Analogy

Imagine checking the highest score on a sheet. Keep the best score seen so far and replace it whenever a higher score appears.

### 5. Key Observation

After inspecting a prefix of the array, only its largest value matters for future comparisons. We do not need to remember every earlier element.

### 6. Pattern

```text
Pattern: Traversal and accumulation
```

Recognize it when a problem asks for one extreme value across an unsorted collection. The accumulator is `largest`.

### 7. Brute Force

Sort a copy in ascending order and return its last value. Sorting works because it places the maximum at the final index, but ordering every element does more work than finding one extreme.

#### Pseudocode

```text
copy the array
sort the copy in ascending order
return the last value
```

#### Brute-Force TypeScript

```typescript
function largestBySorting(values: number[]): number {
  const sorted = [...values].sort((first, second) => first - second);
  return sorted[sorted.length - 1];
}
```

#### Visualization

```text
[3, 8, 2, 11, 5]
        sort
          |
          v
[2, 3, 5, 8, 11]
              ^ largest
```

#### Dry Run

```text
copy   = [3, 8, 2, 11, 5]
sorted = [2, 3, 5, 8, 11]
answer = sorted[4] = 11
```

#### Brute-Force Complexity

Comparison sorting takes `O(n log n)` time. Copying and sorting the copy require `O(n)` additional storage in this contract.

### 8. Why Brute Force Is Not Ideal

Sorting determines the full order even though the problem asks for only one value. It also risks mutating the caller's array unless we copy it.

### 9. Optimal Approach

Initialize `largest` with the first value. Scan from index `1`; whenever the current value is greater, replace `largest`.

Invariant:

> Before processing index `i`, `largest` is the maximum among indexes `0` through `i - 1`.

### 10. Visualization

```text
values = [3, 8, 2, 11, 5]

Start largest = 3
                  largest
                     |
3 -> 8 replaces 3 -> 8
2 -> no change    -> 8
11 -> replaces 8  -> 11
5 -> no change    -> 11
```

### 11. Step-by-Step Dry Run

| Index | Current | Largest before | Action | Largest after |
|---:|---:|---:|---|---:|
| 1 | 8 | 3 | Replace | 8 |
| 2 | 2 | 8 | Keep | 8 |
| 3 | 11 | 8 | Replace | 11 |
| 4 | 5 | 11 | Keep | 11 |

All values have been processed, so the invariant applies to the whole array.

### 12. TypeScript Solution

```typescript
function findLargest(values: number[]): number {
  if (values.length === 0) {
    throw new Error("Expected a non-empty array");
  }

  let largest = values[0];

  for (let index = 1; index < values.length; index += 1) {
    if (values[index] > largest) {
      largest = values[index];
    }
  }

  return largest;
}
```

### 13. Code Walkthrough

The guard enforces the non-empty contract. `values[0]` is a real input value, so it works even when every number is negative. The loop starts at `1` because index `0` already initialized the state. Each successful comparison extends the invariant to include the current value.

### 14. Complexity Analysis

**Time Complexity: `O(n)`**

After constant initialization, the loop checks `n - 1` elements once:

```text
n - 1 comparisons -> linear growth -> O(n)
```

Any correct general solution must inspect every value because an unchecked value could be larger.

**Space Complexity: `O(1)` auxiliary space**

Only `largest` and `index` are stored. Their count does not grow with `n`.

### 15. Edge Cases

- One element: that element is largest.
- All negative: initialize from input, not `0`.
- Duplicate maximum: return the maximum value normally.
- Already sorted or reverse sorted: still one traversal.

### 16. Common Mistakes

- Initializing `largest = 0`, which fails for all-negative arrays.
- Sorting the original array and violating the no-mutation contract.
- Returning the maximum index instead of the value.
- Using `<= values.length` and reading beyond the array.

### 17. Interview Follow-ups

1. How would you return the first index of the largest value?
2. How would you return every index tied for largest?
3. Can the values arrive as a stream?
4. How would invalid values such as `NaN` affect the contract?

### 18. Variations

- Find the smallest value.
- Find both minimum and maximum in one traversal.
- Find the largest object by a selected property.

### 19. Pattern Connections

The same running-best invariant appears in minimum search, maximum difference, stock profit, Kadane's algorithm, and tree traversal aggregates.

### 20. NestJS Project Connection

Expose `POST /api/dsa/arrays/largest` with `{ "values": [3, 8, 2, 11, 5], "includeSteps": true }`. The service can return `result: 11` plus steps shaped as `{ index, current, largestBefore, largestAfter, action }`. Limit trace length because recording all `n` steps adds `O(n)` response space.

### 21. Mini Challenge

Return `{ value, index }`, choosing the smallest index when the maximum occurs more than once. Write the new invariant before changing the code.

## Problem 2 — Find the Second Largest Element

### 1. Problem Statement

Given an array, return its second-largest **distinct** value. Return `null` when fewer than two distinct values exist.

### 2. Example

```text
Input:  [10, 5, 10, 8]
Output: 8
```

### 3. Understand the Problem

"Distinct" means another `10` is not second largest. The result is the greatest value strictly below the maximum. The input is not mutated.

#### In Simple Terms

Find the second different value in descending order. Repeated copies of the largest value do not count as second largest.

### 4. Real-World Analogy

On a leaderboard, two players may share first place. The second distinct score is the next lower score, not another copy of first place.

### 5. Key Observation

Only two values from the processed prefix matter: its largest distinct value and its second-largest distinct value.

### 6. Pattern

```text
Pattern: Traversal with two running extremes
```

Recognize it when the problem asks for the top few distinct values and sorting the entire input is unnecessary.

### 7. Brute Force

Remove duplicates with a `Set`, sort the distinct values descending, and read index `1`.

#### Pseudocode

```text
distinct = unique values
sort distinct descending
if distinct has fewer than 2 values, return null
return distinct[1]
```

#### Brute-Force TypeScript

```typescript
function secondLargestBySorting(values: number[]): number | null {
  const sorted = [...values].sort((first, second) => second - first);
  for (let index = 1; index < sorted.length; index += 1) {
    if (sorted[index] !== sorted[0]) return sorted[index];
  }
  return null;
}
```

#### Visualization

```text
[10, 5, 10, 8] -> {10, 5, 8} -> [10, 8, 5]
                                      ^   ^
                                  largest second
```

#### Dry Run

```text
unique = [10, 5, 8]
sorted = [10, 8, 5]
answer = 8
```

#### Brute-Force Complexity

Building distinct values costs `O(n)` average time; sorting up to `n` values costs `O(n log n)`. The set and sorted array use `O(n)` space.

### 8. Why Brute Force Is Not Ideal

Sorting computes every rank although only two ranks are needed. It also allocates storage proportional to the input.

### 9. Optimal Approach

Track `largest` and `secondLargest`. For each value:

- if it exceeds `largest`, move the old largest into second place
- otherwise, if it lies strictly between the two, update second place
- ignore another copy of `largest`

Both variables begin at `-Infinity`; return `null` if second place never changes.

### 10. Visualization

```text
value  largest  secondLargest
10       10        -Infinity
 5       10         5
10       10         5          duplicate maximum: ignore
 8       10         8
```

### 11. Step-by-Step Dry Run

| Value | Decision | Largest | Second largest |
|---:|---|---:|---:|
| 10 | New largest | 10 | `-Infinity` |
| 5 | Between largest and second | 10 | 5 |
| 10 | Equal to largest, ignore | 10 | 5 |
| 8 | Better second | 10 | 8 |

### 12. TypeScript Solution

```typescript
function findSecondLargest(values: number[]): number | null {
  let largest = -Infinity;
  let secondLargest = -Infinity;

  for (const value of values) {
    if (value > largest) {
      secondLargest = largest;
      largest = value;
    } else if (value < largest && value > secondLargest) {
      secondLargest = value;
    }
  }

  return secondLargest === -Infinity ? null : secondLargest;
}
```

### 13. Code Walkthrough

When a new maximum arrives, the old maximum is exactly the best candidate for second place. The `else if` requires `value < largest`, so duplicate maxima cannot replace the second distinct value. The final sentinel check reports that no second distinct value was found.

### 14. Complexity Analysis

**Time Complexity: `O(n)`** because one loop performs constant comparisons for each of `n` values. **Space Complexity: `O(1)` auxiliary space** because only two running values are stored.

### 15. Edge Cases

- `[]`, `[4]`, or `[4, 4]` returns `null`.
- All negative values work because the sentinels are below every finite input.
- Duplicate largest and second-largest values do not change distinct ranking.

### 16. Common Mistakes

- Treating a duplicate maximum as second largest.
- Updating `largest` before saving its old value.
- Initializing to `0`, which fails for negative inputs.
- Forgetting to define behavior when no second distinct value exists.

### 17. Interview Follow-ups

1. What if "second largest" includes duplicates?
2. How would you return the index too?
3. How would you find the third largest distinct value?
4. Can this process a stream?

### 18. Variations

- Return the top `k` distinct values.
- Find the second largest object by a property.
- Return both largest and second largest.

### 19. Pattern Connections

Running extremes lead to tournament methods, stock-profit state, and top-`k` structures such as heaps.

### 20. NestJS Project Connection

Use `POST /api/dsa/arrays/second-largest`. Trace fields can be `{ value, largestBefore, secondBefore, largestAfter, secondAfter }`; serialize missing sentinels as `null`, not JSON-incompatible infinity.

### 21. Mini Challenge

Return the second-largest distinct value and the first index where it occurs without adding another traversal.

## Problem 3 — Find the Smallest and Second Smallest Element

### 1. Problem Statement

Return the smallest and second-smallest distinct values in an array, or `null` if two distinct values do not exist.

### 2. Example

```text
Input:  [4, 1, 3, 1, 2]
Output: { smallest: 1, secondSmallest: 2 }
```

### 3. Understand the Problem

The two values must be distinct. Their original positions do not matter, and the input must remain unchanged.

#### In Simple Terms

Return the smallest value and the next different value above it. If two distinct values do not exist, report that no answer exists.

### 4. Real-World Analogy

From several delivery quotes, choose the cheapest distinct price and the next-cheapest distinct alternative.

### 5. Key Observation

The processed prefix can be summarized by its two smallest distinct values. Everything larger can be forgotten.

### 6. Pattern

```text
Pattern: Traversal with two running extremes
```

The pattern applies because the required output has fixed size and each new value can be compared with the two current candidates.

### 7. Brute Force

Deduplicate, sort ascending, and return the first two values.

#### Pseudocode

```text
distinct = unique values
sort distinct ascending
if fewer than 2 distinct values, return null
return distinct[0] and distinct[1]
```

#### Brute-Force TypeScript

```typescript
function twoSmallestBySorting(values: number[]): [number, number] | null {
  const sorted = [...values].sort((first, second) => first - second);
  for (let index = 1; index < sorted.length; index += 1) {
    if (sorted[index] !== sorted[0]) return [sorted[0], sorted[index]];
  }
  return null;
}
```

#### Visualization

```text
[4, 1, 3, 1, 2] -> [4, 1, 3, 2] -> [1, 2, 3, 4]
                                         ^  ^
                                      first second
```

#### Dry Run

```text
distinct values = [4, 1, 3, 2]
sorted values   = [1, 2, 3, 4]
answer          = {1, 2}
```

#### Brute-Force Complexity

Deduplication is average `O(n)` and sorting is `O(n log n)`. Storing distinct values costs `O(n)` space.

### 8. Why Brute Force Is Not Ideal

Full ordering and a collection of every distinct value are unnecessary when the output contains only two values.

### 9. Optimal Approach

Track `smallest` and `secondSmallest`, initialized to `Infinity`. A new minimum pushes the old minimum to second place. A value strictly between them updates only second place.

### 10. Visualization

```text
value  smallest  secondSmallest
 4        4         Infinity
 1        1             4
 3        1             3
 1        1             3       duplicate: ignore
 2        1             2
```

### 11. Step-by-Step Dry Run

| Value | Action | Smallest | Second smallest |
|---:|---|---:|---:|
| 4 | New smallest | 4 | `Infinity` |
| 1 | Push 4 to second | 1 | 4 |
| 3 | Better second | 1 | 3 |
| 1 | Duplicate minimum | 1 | 3 |
| 2 | Better second | 1 | 2 |

### 12. TypeScript Solution

```typescript
type TwoSmallest = {
  smallest: number;
  secondSmallest: number;
};

function findTwoSmallest(values: number[]): TwoSmallest | null {
  let smallest = Infinity;
  let secondSmallest = Infinity;

  for (const value of values) {
    if (value < smallest) {
      secondSmallest = smallest;
      smallest = value;
    } else if (value > smallest && value < secondSmallest) {
      secondSmallest = value;
    }
  }

  return secondSmallest === Infinity
    ? null
    : { smallest, secondSmallest };
}
```

### 13. Code Walkthrough

The first branch preserves the displaced minimum. The second branch excludes duplicate minima and accepts only an improvement to second place. The return type makes the insufficient-data case explicit.

### 14. Complexity Analysis

**Time Complexity: `O(n)`** because all `n` values are visited once. **Space Complexity: `O(1)` auxiliary space** because the two candidates and returned fixed-size object do not grow with `n`.

### 15. Edge Cases

- Empty, one-value, and all-equal arrays return `null`.
- Negative and decimal values compare normally.
- The values may appear in any order.

### 16. Common Mistakes

- Allowing the same value to occupy both positions.
- Losing the old minimum when a new minimum arrives.
- Returning an ambiguous sentinel instead of `null`.
- Sorting the caller's array.

### 17. Interview Follow-ups

1. Can you minimize the number of comparisons?
2. How would the contract change if duplicates count?
3. How would you find the `k` smallest distinct values?
4. Can the method process streaming values?

### 18. Variations

- Find two largest values.
- Return indexes with stable tie behavior.
- Track the smallest object by a selector function.

### 19. Pattern Connections

This is the mirror image of second-largest tracking and prepares for fixed-size top-`k` state and heap problems.

### 20. NestJS Project Connection

Expose `POST /api/dsa/arrays/two-smallest`. Validate finite numbers and return state transitions. Convert internal `Infinity` to `null` in trace DTOs so JSON remains valid.

### 21. Mini Challenge

Extend the function to return the three smallest distinct values while still using one traversal.

## Problem 4 — Check if an Array Is Sorted

### 1. Problem Statement

Return `true` when an array is sorted in non-decreasing order; otherwise return `false`.

### 2. Example

```text
Input:  [1, 2, 2, 5]
Output: true

Input:  [1, 4, 3]
Output: false
```

### 3. Understand the Problem

Non-decreasing means each value is greater than or equal to the previous value. Equal neighbors are valid. Empty and one-element arrays are sorted because no pair violates the rule.

#### In Simple Terms

Check whether every value is at least as large as the value immediately before it.

### 4. Real-World Analogy

Check books arranged by height: inspect each neighboring pair and stop at the first book shorter than the one before it.

### 5. Key Observation

An array is non-decreasing exactly when no adjacent pair has `values[i] < values[i - 1]`.

### 6. Pattern

```text
Pattern: Adjacent-pair traversal with early exit
```

Recognize it when a global ordering property can be disproved by one local neighboring violation.

### 7. Brute Force

Sort a copy and compare every position with the original.

#### Pseudocode

```text
copy and sort values
for every index
    if original[index] differs from sorted[index], return false
return true
```

#### Brute-Force TypeScript

```typescript
function isSortedByComparingAllPairs(values: number[]): boolean {
  for (let first = 0; first < values.length; first += 1) {
    for (let second = first + 1; second < values.length; second += 1) {
      if (values[first] > values[second]) return false;
    }
  }
  return true;
}
```

#### Visualization

```text
original: [1, 4, 3]
sorted:   [1, 3, 4]
              ^ mismatch
```

#### Dry Run

The copied array becomes `[1, 3, 4]`. Index `1` differs from the original value `4`, so return `false`.

#### Brute-Force Complexity

Sorting costs `O(n log n)` time; copying and the comparison each cost `O(n)`. The copy uses `O(n)` space.

### 8. Why Brute Force Is Not Ideal

Sorting repairs the order and performs much more work than merely detecting a violation.

### 9. Optimal Approach

Scan from index `1`. Compare each value with its immediate predecessor. Return `false` on the first decrease; if none exists, return `true`.

### 10. Visualization

```text
[1, 2, 2, 5]
 |--|  1 <= 2
    |--|  2 <= 2
       |--|  2 <= 5

No violation -> sorted
```

### 11. Step-by-Step Dry Run

For `[1, 4, 3, 7]`:

| Index | Previous | Current | Valid? | Decision |
|---:|---:|---:|:---:|---|
| 1 | 1 | 4 | Yes | Continue |
| 2 | 4 | 3 | No | Return `false` |

Index `3` need not be inspected because one violation proves the answer.

### 12. TypeScript Solution

```typescript
function isSorted(values: number[]): boolean {
  for (let index = 1; index < values.length; index += 1) {
    if (values[index] < values[index - 1]) {
      return false;
    }
  }

  return true;
}
```

### 13. Code Walkthrough

Starting at `1` guarantees a valid predecessor. The strict `<` comparison permits equal neighbors. Early return is safe because one decrease permanently disproves non-decreasing order.

### 14. Complexity Analysis

**Time Complexity: `O(n)` worst case.** A sorted array requires `n - 1` comparisons. An early violation can make the best case `O(1)`. **Space Complexity: `O(1)`** because no input-sized storage is created.

### 15. Edge Cases

- `[]` and `[7]` return `true`.
- Duplicate neighbors are valid.
- Negative values use ordinary numeric order.
- A violation at the last pair requires a full scan.

### 16. Common Mistakes

- Using `<=` for the violation and rejecting duplicates.
- Starting at index `0` and reading index `-1`.
- Sorting the input, which changes the data being checked.
- Checking only the first and last values.

### 17. Interview Follow-ups

1. How would you check non-increasing order?
2. How would you determine whether either direction is sorted?
3. Can you return the first violating index?
4. What changes for objects and a comparator?

### 18. Variations

- Strictly increasing order.
- Sorted and rotated arrays.
- Lexicographic ordering of strings.

### 19. Pattern Connections

Adjacent-pair checks appear in inversion detection, monotonic arrays, runs, local peaks, and interval validation.

### 20. NestJS Project Connection

Use `POST /api/dsa/arrays/is-sorted`. A trace can return `{ leftIndex, rightIndex, leftValue, rightValue, valid }` and stop at the first violation.

### 21. Mini Challenge

Return `{ sorted, violationIndex }`, where `violationIndex` is the current index in the first decreasing pair or `null`.

## Problem 5 — Reverse an Array

### 1. Problem Statement

Reverse an array in place so its first value becomes last, its second becomes second-to-last, and so on.

### 2. Example

```text
Input:  [1, 2, 3, 4, 5]
After:  [5, 4, 3, 2, 1]
```

### 3. Understand the Problem

The function mutates the given array and returns it for convenience. Values are rearranged, not changed. Empty and one-element arrays remain the same.

#### In Simple Terms

Return the same values in the opposite order, so the last value becomes first and the first becomes last.

### 4. Real-World Analogy

Reverse a row of cards by swapping the outermost cards, then moving inward until no unswapped pair remains.

### 5. Key Observation

Index `left` belongs at `n - 1 - left`. Each outer-pair swap places two values in their final positions.

### 6. Pattern

```text
Pattern: Two pointers moving inward
```

Recognize it when positions at opposite ends correspond and the operation is symmetric.

### 7. Brute Force

Create a result array by reading the input from right to left.

#### Pseudocode

```text
result = empty array
for index from n - 1 down to 0
    append values[index] to result
return result
```

#### Brute-Force TypeScript

```typescript
function reverseWithExtraArray<T>(values: T[]): T[] {
  const reversed: T[] = [];
  for (let index = values.length - 1; index >= 0; index -= 1) {
    reversed.push(values[index]);
  }
  return reversed;
}
```

#### Visualization

```text
input:  [1, 2, 3, 4]
read:    4  3  2  1
result: [4, 3, 2, 1]
```

#### Dry Run

Append `4`, then `3`, then `2`, then `1`. The output is reversed, but four new positions are allocated.

#### Brute-Force Complexity

Reading `n` values takes `O(n)` time. The result stores `n` values, so output space is `O(n)`.

### 8. Why Brute Force Is Not Ideal

When mutation is allowed, a second array is unnecessary. The original positions can be exchanged directly.

### 9. Optimal Approach

Set `left = 0` and `right = n - 1`. Swap those positions, move both pointers inward, and stop when they meet or cross.

Invariant:

> Values outside `[left, right]` are already in their final reversed positions.

### 10. Visualization

```text
[1, 2, 3, 4, 5]
 ^           ^       swap
[5, 2, 3, 4, 1]
    ^     ^          swap
[5, 4, 3, 2, 1]
       ^             stop
```

### 11. Step-by-Step Dry Run

| Step | Left | Right | Swap | Array after |
|---:|---:|---:|---|---|
| 1 | 0 | 4 | 1 and 5 | `[5, 2, 3, 4, 1]` |
| 2 | 1 | 3 | 2 and 4 | `[5, 4, 3, 2, 1]` |

Now `left = right = 2`; the middle value is already correct.

### 12. TypeScript Solution

```typescript
function reverseArray<T>(values: T[]): T[] {
  let left = 0;
  let right = values.length - 1;

  while (left < right) {
    [values[left], values[right]] = [values[right], values[left]];
    left += 1;
    right -= 1;
  }

  return values;
}
```

### 13. Code Walkthrough

The pointers start at opposite boundaries. Destructuring swaps without losing either value. Moving both pointers guarantees progress, and `left < right` avoids swapping the middle element with itself.

### 14. Complexity Analysis

**Time Complexity: `O(n)`.** There are `floor(n / 2)` swaps. Dropping the constant factor gives linear time. **Space Complexity: `O(1)` auxiliary space** because swaps use a fixed amount of temporary storage.

### 15. Edge Cases

- Empty and one-element arrays perform no swaps.
- Even lengths end with crossed pointers.
- Odd lengths leave the middle value untouched.
- Objects are moved by reference; their contents are not cloned.

### 16. Common Mistakes

- Using `left <= right` unnecessarily.
- Moving only one pointer and causing an infinite loop.
- Overwriting one side before saving the other.
- Claiming the function is non-mutating.

### 17. Interview Follow-ups

1. How would you return a reversed copy?
2. How would you reverse only indexes `start` through `end`?
3. Can you implement it recursively?
4. What is the recursive stack cost?

### 18. Variations

- Reverse a string's characters.
- Reverse words but not characters within each word.
- Reverse nodes in a linked list.

### 19. Pattern Connections

Inward pointers power palindrome checks, segment reversal, array rotation, and pair problems on sorted arrays.

### 20. NestJS Project Connection

Expose `POST /api/dsa/arrays/reverse`. Copy the validated DTO array before calling the in-place service algorithm so request data remains available for logging and comparison. Trace each `{ left, right, before, after }` swap.

### 21. Mini Challenge

Implement `reverseRange(values, start, end)` with validation and `O(1)` auxiliary space.

## Problem 6 — Remove Duplicates from a Sorted Array

### 1. Problem Statement

Given a sorted array, remove duplicates in place and return the number of unique values. Store those values in the first returned-length positions.

### 2. Example

```text
Input:  [1, 1, 2, 2, 3]
Output length: 3
Valid prefix:  [1, 2, 3]
```

### 3. Understand the Problem

Only the prefix of returned length matters afterward. The array is sorted in non-decreasing order, mutation is allowed, and relative order is preserved.

#### In Simple Terms

Keep one copy of each value in this already sorted array and place those unique values at the front.

### 4. Real-World Analogy

From an alphabetized guest list, keep the first copy of each adjacent name and write accepted names at the front.

### 5. Key Observation

Sorted duplicates are adjacent. A value is new exactly when it differs from the last unique value written.

### 6. Pattern

```text
Pattern: Two-pointer in-place compaction
```

One pointer reads candidates; another marks the next write position.

### 7. Brute Force

Insert all values into a `Set`, then copy the set back.

#### Pseudocode

```text
unique = set of values
write each unique value into the array
return unique size
```

#### Brute-Force TypeScript

```typescript
function removeSortedDuplicatesWithExtraArray(values: number[]): number {
  const unique: number[] = [];
  for (const value of values) {
    if (unique.length === 0 || unique[unique.length - 1] !== value) unique.push(value);
  }
  for (let index = 0; index < unique.length; index += 1) values[index] = unique[index];
  return unique.length;
}
```

#### Visualization

```text
[1, 1, 2, 2, 3] -> Set {1, 2, 3} -> [1, 2, 3, _, _]
```

#### Dry Run

The set rejects second copies of `1` and `2`; three values are copied back.

#### Brute-Force Complexity

Two linear passes give `O(n)` average time. The set may store `n` values, so auxiliary space is `O(n)`.

### 8. Why Brute Force Is Not Ideal

It ignores the useful sorted order and allocates input-sized storage.

### 9. Optimal Approach

Return `0` for empty input. Set `write = 1`; scan with `read = 1`. When `values[read] !== values[write - 1]`, copy it to `values[write]` and advance `write`.

### 10. Visualization

```text
[1, 1, 2, 2, 3]
    R              duplicate
       R           write 2 at W=1 -> [1,2,2,2,3]
          R        duplicate
             R     write 3 at W=2 -> [1,2,3,2,3]
```

### 11. Step-by-Step Dry Run

| Read value | Last unique | Action | Unique prefix |
|---:|---:|---|---|
| 1 | 1 | Skip | `[1]` |
| 2 | 1 | Write | `[1, 2]` |
| 2 | 2 | Skip | `[1, 2]` |
| 3 | 2 | Write | `[1, 2, 3]` |

### 12. TypeScript Solution

```typescript
function removeSortedDuplicates(values: number[]): number {
  if (values.length === 0) return 0;

  let write = 1;
  for (let read = 1; read < values.length; read += 1) {
    if (values[read] !== values[write - 1]) {
      values[write] = values[read];
      write += 1;
    }
  }
  return write;
}
```

### 13. Code Walkthrough

`write` is both the next destination and the unique count. The prefix before `write` always contains each processed distinct value once.

### 14. Complexity Analysis

**Time: `O(n)`** because `read` visits each value once. **Auxiliary space: `O(1)`** because two indexes are stored; the input provides output storage.

### 15. Edge Cases

- Empty returns `0`.
- One value returns `1`.
- All equal returns `1`.
- All distinct returns the original length.

### 16. Common Mistakes

- Comparing with `values[read - 1]` after writes obscure the unique boundary.
- Returning the array instead of the required length.
- Treating positions after the valid prefix as meaningful.
- Applying this method to unsorted input.

### 17. Interview Follow-ups

1. Allow each value at most twice.
2. Return a new array without mutation.
3. Remove a chosen value in place.

### 18. Variations

- Compact null values.
- Deduplicate sorted objects by key.
- Keep at most `k` copies.

### 19. Pattern Connections

The read/write boundary also powers moving zeroes, filtering in place, and merging sorted data.

### 20. NestJS Project Connection

Use `POST /api/dsa/arrays/remove-sorted-duplicates`; return `{ length, values: values.slice(0, length) }` and trace read/write positions.

### 21. Mini Challenge

Modify the invariant and code so every distinct value may appear at most twice.

## Problem 7 — Remove Duplicates from an Unsorted Array

### 1. Problem Statement

Return a new array containing the first occurrence of each value from an unsorted array, preserving encounter order.

### 2. Example

```text
Input:  [4, 2, 4, 1, 2]
Output: [4, 2, 1]
```

### 3. Understand the Problem

The input remains unchanged. "First occurrence" makes output order explicit; values need not be sorted.

#### In Simple Terms

Return one copy of each value, keeping the order in which values first appeared.

### 4. Real-World Analogy

A door attendant keeps the first arrival of each ticket number and rejects later repeats.

### 5. Key Observation

Unlike sorted input, duplicates may be far apart. Remembering all accepted values avoids rescanning the output.

### 6. Pattern

```text
Pattern: Hash-based membership during traversal
```

Recognize it when repeated "have I seen this?" questions occur on unsorted data.

### 7. Brute Force

For each value, linearly search the result before appending it.

#### Pseudocode

```text
result = []
for each value
    if result does not contain value
        append value
```

#### Brute-Force TypeScript

```typescript
function removeUnsortedDuplicatesByScanning(values: number[]): number[] {
  const unique: number[] = [];
  for (const value of values) {
    if (!unique.includes(value)) unique.push(value);
  }
  return unique;
}
```

#### Visualization

```text
4 -> []       append
2 -> [4]      scan, append
4 -> [4,2]    scan, skip
```

#### Dry Run

The result grows `[4]`, `[4,2]`, remains unchanged for `4`, then becomes `[4,2,1]` and skips final `2`.

#### Brute-Force Complexity

`includes` may scan up to `n` results for each of `n` values: `n x n = O(n^2)`. Output space is `O(n)`.

### 8. Why Brute Force Is Not Ideal

It repeatedly searches values already processed.

### 9. Optimal Approach

Maintain a `Set` named `seen`. If a value is absent, add it to both `seen` and `result`; otherwise skip it.

### 10. Visualization

```text
value  seen before  action   result
4      {}           add      [4]
2      {4}          add      [4,2]
4      {4,2}        skip     [4,2]
1      {4,2}        add      [4,2,1]
2      {4,2,1}      skip     [4,2,1]
```

### 11. Step-by-Step Dry Run

At every step, `seen` equals the set of values in `result`. Therefore membership determines whether appending would create a duplicate.

### 12. TypeScript Solution

```typescript
function removeUnsortedDuplicates(values: number[]): number[] {
  const seen = new Set<number>();
  const result: number[] = [];

  for (const value of values) {
    if (!seen.has(value)) {
      seen.add(value);
      result.push(value);
    }
  }
  return result;
}
```

### 13. Code Walkthrough

The set provides average constant-time membership. Appending only when absent preserves the first encounter order.

### 14. Complexity Analysis

**Time: `O(n)` average** because `n` set lookups/additions are average `O(1)`. **Space: `O(n)`** for up to `n` set entries and `n` output values.

### 15. Edge Cases

- Empty returns `[]`.
- All equal returns one value.
- All distinct returns a copy in the same order.
- Negative and zero values work normally.

### 16. Common Mistakes

- Sorting and losing encounter order.
- Using an object and coercing keys unexpectedly.
- Claiming guaranteed rather than average `O(1)` hash operations.
- Mutating input despite the contract.

### 17. Interview Follow-ups

1. Can order be discarded?
2. Can you mutate the input?
3. What if values are objects with an `id`?
4. What if values lie in a small fixed range?

### 18. Variations

- Keep the last occurrence.
- Deduplicate case-insensitive strings.
- Return duplicate values only.

### 19. Pattern Connections

Seen sets appear in duplicate detection, longest-consecutive sequence, graph traversal, and cycle prevention.

### 20. NestJS Project Connection

Expose `POST /api/dsa/arrays/remove-duplicates`; trace `{ index, value, alreadySeen, resultLength }` while limiting large traces.

### 21. Mini Challenge

Deduplicate objects by `id` while preserving the first complete object for each ID.

## Problem 8 — Move All Zeroes to the End

### 1. Problem Statement

Move every zero to the end in place while preserving the relative order of non-zero values.

### 2. Example

```text
Input:  [0, 1, 0, 3, 12]
After:  [1, 3, 12, 0, 0]
```

### 3. Understand the Problem

This is a **stable** transformation: `1`, `3`, and `12` remain in that order. The array length is unchanged and mutation is allowed.

#### In Simple Terms

Move every zero behind the nonzero values, while keeping the nonzero values in their original order.

### 4. Real-World Analogy

Move empty boxes to the back of a conveyor without changing the order of filled boxes.

### 5. Key Observation

Non-zero values can be compacted into the next available front position. All remaining positions are then filled with zeroes.

### 6. Pattern

```text
Pattern: Stable read/write compaction
```

The read pointer inspects every item; `write` marks the next non-zero destination.

### 7. Brute Force

Build a new array of non-zero values, then append the counted zeroes.

#### Pseudocode

```text
result = all non-zero values
append zero until result length equals input length
copy result back
```

#### Brute-Force TypeScript

```typescript
function moveZeroesWithExtraArray(values: number[]): void {
  const nonzero = values.filter((value) => value !== 0);
  const zeroCount = values.length - nonzero.length;
  values.splice(0, values.length, ...nonzero, ...Array(zeroCount).fill(0));
}
```

#### Visualization

```text
[0,1,0,3,12] -> [1,3,12] + [0,0] -> [1,3,12,0,0]
```

#### Dry Run

Collect `1`, `3`, `12`; two omitted positions become zeroes.

#### Brute-Force Complexity

Sequential passes total `O(n)` time. The result array uses `O(n)` auxiliary space.

### 8. Why Brute Force Is Not Ideal

The existing array can hold the compacted result, so an input-sized temporary array is unnecessary.

### 9. Optimal Approach

Write each non-zero value at `write` and advance it. After reading everything, fill indexes from `write` through the end with zero.

### 10. Visualization

```text
read values: 0, 1, 0, 3, 12
written:        [1, 3, 12, _, _]
fill tail:      [1, 3, 12, 0, 0]
```

### 11. Step-by-Step Dry Run

| Read | Value | Action | Write | Array prefix |
|---:|---:|---|---:|---|
| 0 | 0 | Skip | 0 | `[]` |
| 1 | 1 | Write | 1 | `[1]` |
| 2 | 0 | Skip | 1 | `[1]` |
| 3 | 3 | Write | 2 | `[1,3]` |
| 4 | 12 | Write | 3 | `[1,3,12]` |

### 12. TypeScript Solution

```typescript
function moveZeroes(values: number[]): void {
  let write = 0;
  for (const value of values) {
    if (value !== 0) {
      values[write] = value;
      write += 1;
    }
  }
  while (write < values.length) {
    values[write] = 0;
    write += 1;
  }
}
```

### 13. Code Walkthrough

The first loop preserves non-zero encounter order. After it, indexes before `write` are correct; the second loop replaces stale tail values with zeroes.

### 14. Complexity Analysis

**Time: `O(n)`** because at most `n + n` simple visits occur, and `2n` is linear. **Auxiliary space: `O(1)`** because only `write` is stored.

### 15. Edge Cases

- Empty or all-zero arrays remain unchanged.
- No-zero arrays are rewritten identically.
- Negative values are non-zero and retain order.

### 16. Common Mistakes

- Swapping with the end and destroying stable order.
- Forgetting to zero the stale tail.
- Using `filter` while claiming constant auxiliary space.
- Splicing during traversal and skipping elements.

### 17. Interview Follow-ups

1. Minimize writes using swaps.
2. Return a new array instead.
3. Move a chosen sentinel value.
4. Move zeroes to the front stably.

### 18. Variations

- Compact nulls.
- Stable partition by a predicate.
- Remove a value and return the valid length.

### 19. Pattern Connections

This is sorted deduplication's read/write pattern with a different acceptance rule.

### 20. NestJS Project Connection

Use `POST /api/dsa/arrays/move-zeroes`; return the transformed copy and phases such as `compact` and `fill` in trace steps.

### 21. Mini Challenge

Rewrite the solution to avoid assigning a non-zero value when `read === write`, then count its writes.

## Problem 9 — Move All Negative Numbers to One Side

### 1. Problem Statement

Rearrange an array in place so all negative values appear before all non-negative values. Relative order does not need to be preserved.

### 2. Example

```text
Input:  [1, -2, 3, -4, 0]
One valid output: [-4, -2, 3, 1, 0]
```

### 3. Understand the Problem

Any arrangement satisfying the partition is valid. Zero is non-negative. Mutation is allowed, enabling constant auxiliary space.

#### In Simple Terms

Rearrange the array so every negative value is on the left side and every nonnegative value is on the right.

### 4. Real-World Analogy

Separate red and blue cards into two sides of a table; exact order within a color does not matter.

### 5. Key Observation

When order is irrelevant, misplaced values at opposite ends can be swapped, fixing both sides at once.

### 6. Pattern

```text
Pattern: Two-pointer partition
```

Recognize it when values must be divided by a predicate and stability is not required.

### 7. Brute Force

Collect negatives and non-negatives separately, then concatenate them.

#### Pseudocode

```text
negatives = []
others = []
classify each value
return negatives followed by others
```

#### Brute-Force TypeScript

```typescript
function moveNegativesWithExtraArrays(values: number[]): void {
  const negatives = values.filter((value) => value < 0);
  const nonnegatives = values.filter((value) => value >= 0);
  values.splice(0, values.length, ...negatives, ...nonnegatives);
}
```

#### Visualization

```text
[1,-2,3,-4,0] -> [-2,-4] + [1,3,0]
```

#### Dry Run

Classification creates `[-2,-4]` and `[1,3,0]`; joining produces a valid partition.

#### Brute-Force Complexity

Classification and concatenation are `O(n)` time; temporary arrays hold `n` values, so space is `O(n)`.

### 8. Why Brute Force Is Not Ideal

It allocates new storage even though order preservation is not requested.

### 9. Optimal Approach

Advance `left` past correct negatives and `right` past correct non-negatives. When both stop on misplaced values, swap them and continue.

### 10. Visualization

```text
[1, -2, 3, -4, 0]
 ^          ^          swap 1 and -4
[-4,-2, 3,  1, 0]
        ^   ^          pointers meet/cross after scans
```

### 11. Step-by-Step Dry Run

`left=0` stops on `1`; `right` skips `0`, then stops on `-4`. Swap. Next, `left` skips `-4,-2`, stops on `3`; `right` skips `1,3` and crosses `left`. Partition complete.

### 12. TypeScript Solution

```typescript
function partitionNegatives(values: number[]): void {
  let left = 0;
  let right = values.length - 1;

  while (left <= right) {
    while (left <= right && values[left] < 0) left += 1;
    while (left <= right && values[right] >= 0) right -= 1;
    if (left < right) {
      [values[left], values[right]] = [values[right], values[left]];
      left += 1;
      right -= 1;
    }
  }
}
```

### 13. Code Walkthrough

Inner loops skip already-correct regions. Boundary checks prevent out-of-range reads. A swap occurs only when `left` is non-negative and `right` is negative.

### 14. Complexity Analysis

**Time: `O(n)`** because each pointer crosses each position at most once; nested syntax does not cause repeated full scans. **Auxiliary space: `O(1)`**.

### 15. Edge Cases

- Empty, all-negative, or all-non-negative arrays need no swaps.
- Zero belongs on the non-negative side.
- Duplicates do not affect partition correctness.

### 16. Common Mistakes

- Promising stable order with this swapping method.
- Classifying zero as negative.
- Omitting boundary checks in inner loops.
- Assuming nested loops automatically mean `O(n^2)`.

### 17. Interview Follow-ups

1. Preserve relative order.
2. Partition around a pivot.
3. Separate negative, zero, and positive values.
4. Return the partition boundary.

### 18. Variations

- Partition odd and even values.
- Quicksort partitioning.
- Stable predicate partition with extra space.

### 19. Pattern Connections

Partitioning underlies quicksort, sort-colors, filtering, and many two-pointer problems.

### 20. NestJS Project Connection

Expose `POST /api/dsa/arrays/partition-negatives`; execute on a DTO copy and trace pointer scans separately from swaps.

### 21. Mini Challenge

Return the first index of the non-negative region after partitioning.

## Problem 10 — Find the Missing Number from 1..n

### 1. Problem Statement

An array contains `n - 1` distinct integers from `1` through `n`. Return the one missing number.

### 2. Example

```text
Input:  values = [1, 2, 4, 5], n = 5
Output: 3
```

### 3. Understand the Problem

Exactly one number is missing; values are distinct and within the stated range. These constraints are required for the arithmetic method.

#### In Simple Terms

The array should contain every number from `1` through `n` except one. Return the number that is absent.

### 4. Real-World Analogy

Compare the expected total of numbered tickets with the total of tickets actually returned; the difference is the missing ticket.

### 5. Key Observation

The sum from `1` through `n` is `n(n + 1) / 2`. Removing one value lowers the actual sum by exactly that value.

### 6. Pattern

```text
Pattern: Expected aggregate minus observed aggregate
```

Recognize it when a complete known range differs by exactly one item.

### 7. Brute Force

For every candidate from `1` to `n`, scan the array to check whether it exists.

#### Pseudocode

```text
for candidate from 1 through n
    scan values for candidate
    if absent, return candidate
```

#### Brute-Force TypeScript

```typescript
function findMissingBySearching(values: number[], n: number): number | null {
  for (let candidate = 1; candidate <= n; candidate += 1) {
    if (!values.includes(candidate)) return candidate;
  }
  return null;
}
```

#### Visualization

```text
candidate 1 -> found
candidate 2 -> found
candidate 3 -> full scan -> absent
```

#### Dry Run

For `[1,2,4,5]`, candidates `1` and `2` are found; candidate `3` is absent and returned.

#### Brute-Force Complexity

Up to `n` candidates each trigger an `O(n)` scan: `O(n^2)` time. Auxiliary space is `O(1)`.

### 8. Why Brute Force Is Not Ideal

It rereads the same array for every candidate.

### 9. Optimal Approach

Compute the expected sum, subtract every observed value, and return the remainder.

Invariant:

> After processing a prefix, `missing` equals the expected total minus the sum of that processed prefix.

### 10. Visualization

```text
Expected: 1 + 2 + 3 + 4 + 5 = 15
Observed: 1 + 2 + 4 + 5     = 12
Difference:                    3
```

### 11. Step-by-Step Dry Run

| Action | Remaining difference |
|---|---:|
| Start with `5 x 6 / 2` | 15 |
| Subtract 1 | 14 |
| Subtract 2 | 12 |
| Subtract 4 | 8 |
| Subtract 5 | 3 |

### 12. TypeScript Solution

```typescript
function findMissingNumber(values: number[], n: number): number {
  let missing = (n * (n + 1)) / 2;
  for (const value of values) missing -= value;
  return missing;
}
```

### 13. Code Walkthrough

The formula creates the complete-range total in constant time. Each present value is removed once, leaving only the absent value.

### 14. Complexity Analysis

**Time: `O(n)`** because `n - 1` observed values are subtracted once. **Auxiliary space: `O(1)`** because only the running difference is stored.

### 15. Edge Cases

- `[]` with `n=1` returns `1`.
- The missing value may be `1` or `n`.
- Invalid duplicates or out-of-range values break the contract.
- Very large `n` can exceed JavaScript's safe integer range.

### 16. Common Mistakes

- Using array length as `n` even though length is `n - 1`.
- Forgetting arithmetic overflow or safe-integer limits.
- Applying the formula when duplicates are allowed.
- Sorting unnecessarily.

### 17. Interview Follow-ups

1. Solve with XOR to avoid large sums.
2. Find two missing numbers.
3. Validate the input contract.
4. Handle values from `0..n`.

### 18. Variations

- Missing number from `0..n`.
- Missing and repeated number.
- Multiple missing values using marking or sets.

### 19. Pattern Connections

Expected-versus-observed reasoning appears in XOR cancellation, frequency balances, and checksum validation.

### 20. NestJS Project Connection

Use `POST /api/dsa/arrays/missing-number` with `values` and `n`. Validate length, range, uniqueness, and safe-integer arithmetic before executing; trace each subtraction.

### 21. Mini Challenge

Implement the XOR version and explain why equal values cancel without using extra space.

## Problem 11 — Find the Number That Appears Once

### 1. Problem Statement
Every integer appears exactly twice except one. Return the value that appears once.

### 2. Example
```text
Input: [4, 1, 2, 1, 2]
Output: 4
```

### 3. Understand the Problem
The pair-and-single constraint is guaranteed. Order is irrelevant and input is unchanged.

#### In Simple Terms

Every integer appears exactly twice except one. Return the value that has no matching copy.

### 4. Real-World Analogy
Matched socks cancel in pairs; the remaining sock identifies the unmatched type.

### 5. Key Observation
XOR has `x ^ x = 0`, `x ^ 0 = x`, and order does not matter. Every pair cancels.

### 6. Pattern
```text
Pattern: XOR cancellation
```
Recognize it when identical integers occur an even number of times except one.

### 7. Brute Force
For each value, count its occurrences by scanning the whole array.

#### Pseudocode
```text
for each value
    count matching values
    if count is 1, return value
```

#### Brute-Force TypeScript

```typescript
function findSingleByCounting(values: number[]): number | null {
  for (const candidate of values) {
    let count = 0;
    for (const value of values) {
      if (value === candidate) count += 1;
    }
    if (count === 1) return candidate;
  }
  return null;
}
```

#### Visualization
```text
4 -> scan all -> count 1 -> return 4
```

#### Dry Run
The first candidate `4` differs from all later values, so its count is one.

#### Brute-Force Complexity
Up to `n` candidates perform `n` comparisons: `O(n^2)` time and `O(1)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
It recounts the same pairs. A map improves time but uses `O(n)` space; XOR uses the pair constraint directly.

### 9. Optimal Approach
Start `single = 0` and XOR every value. Pairs become zero, leaving the singleton.

### 10. Visualization
```text
0 ^ 4 = 4
4 ^ 1 = 5
5 ^ 2 = 7
7 ^ 1 = 6
6 ^ 2 = 4
```

### 11. Step-by-Step Dry Run
The running state above ends at `4`. Regrouping explains why: `(1 ^ 1) ^ (2 ^ 2) ^ 4 = 0 ^ 0 ^ 4`.

### 12. TypeScript Solution
```typescript
function findSingleNumber(values: number[]): number {
  let single = 0;
  for (const value of values) single ^= value;
  return single;
}
```

### 13. Code Walkthrough
The accumulator begins at XOR's identity, zero. Each pair eventually reverses its own bits, regardless of position.

### 14. Complexity Analysis
**Time: `O(n)`** for one XOR per value. **Auxiliary space: `O(1)`** for one accumulator. TypeScript bitwise operators convert numbers to signed 32-bit integers.

### 15. Edge Cases
- One value returns itself.
- Negative 32-bit integers work.
- Values outside signed 32-bit range are unsafe for this TypeScript implementation.
- A broken pair constraint makes the result meaningless.

### 16. Common Mistakes
- Using XOR for non-integers or large safe integers.
- Applying it when values occur three times.
- Sorting despite the linear constant-space solution.

### 17. Interview Follow-ups
1. Two values appear once.
2. Every other value appears three times.
3. Solve with a frequency map.

### 18. Variations
- Missing number with XOR.
- Odd-occurrence value.
- Two singleton values using a distinguishing bit.

### 19. Pattern Connections
XOR cancellation connects to bit masks, parity, missing values, and subset-state techniques.

### 20. NestJS Project Connection
Expose `POST /api/dsa/arrays/single-number`; validate signed 32-bit integers and trace `{ value, accumulatorBefore, accumulatorAfter }`.

### 21. Mini Challenge
Find two values that each appear once while every other value appears twice.

## Problem 12 — Find the Maximum Consecutive 1s

### 1. Problem Statement
Return the longest contiguous run of `1`s in a binary array.

### 2. Example
```text
Input: [1, 1, 0, 1, 1, 1]
Output: 3
```

### 3. Understand the Problem
Only adjacent `1`s belong to the same run. Values are guaranteed to be `0` or `1`.

#### In Simple Terms

Find the longest uninterrupted block of `1`s. A `0` ends the current block.

### 4. Real-World Analogy
Find the longest streak of days on which a habit was completed.

### 5. Key Observation
A `1` extends the current streak; a `0` ends it. Preserve the best streak before continuing.

### 6. Pattern
```text
Pattern: Running count and running best
```
Recognize consecutive-run questions where one value resets local state.

### 7. Brute Force
Start at every index and count forward while values remain `1`.

#### Pseudocode
```text
for each start
    count consecutive ones from start
    update best
```

#### Brute-Force TypeScript

```typescript
function maxConsecutiveOnesFromEveryStart(values: number[]): number {
  let best = 0;
  for (let start = 0; start < values.length; start += 1) {
    let length = 0;
    while (start + length < values.length && values[start + length] === 1) {
      length += 1;
    }
    best = Math.max(best, length);
  }
  return best;
}
```

#### Visualization
```text
start 0 -> length 2
start 1 -> length 1
start 3 -> length 3
```

#### Dry Run
The longest attempted run starts at index `3` and reaches the array end with length `3`.

#### Brute-Force Complexity
An all-ones input rescans suffixes: `n + (n-1) + ... + 1 = O(n^2)` time; space is `O(1)`.

### 8. Why Brute Force Is Not Ideal
Overlapping runs are counted repeatedly.

### 9. Optimal Approach
Maintain `current` and `best`. Increment current on `1`; reset it on `0`; update best after each `1`.

### 10. Visualization
```text
value:   1 1 0 1 1 1
current: 1 2 0 1 2 3
best:    1 2 2 2 2 3
```

### 11. Step-by-Step Dry Run
The state rows above show every transition. The final best is `3`, even though earlier state resets.

### 12. TypeScript Solution
```typescript
function maxConsecutiveOnes(values: number[]): number {
  let current = 0;
  let best = 0;
  for (const value of values) {
    current = value === 1 ? current + 1 : 0;
    best = Math.max(best, current);
  }
  return best;
}
```

### 13. Code Walkthrough
`current` describes the run ending at the current position; `best` remembers a longer run even after reset.

### 14. Complexity Analysis
**Time: `O(n)`** because each element causes constant work once. **Auxiliary space: `O(1)`** for two counters.

### 15. Edge Cases
- Empty or all-zero returns `0`.
- All-one returns the array length.
- Runs at either boundary are handled.

### 16. Common Mistakes
- Forgetting to update the best final run.
- Counting total ones rather than consecutive ones.
- Not resetting on zero.

### 17. Interview Follow-ups
1. Flip at most one zero.
2. Flip at most `k` zeroes.
3. Return the best run's indexes.

### 18. Variations
- Longest run of any repeated value.
- Longest increasing contiguous run.
- Longest substring after replacements.

### 19. Pattern Connections
Running local and global best state leads toward sliding windows and Kadane's algorithm.

### 20. NestJS Project Connection
Use `POST /api/dsa/arrays/max-consecutive-ones`; trace `{ index, value, current, best }`.

### 21. Mini Challenge
Return `{ length, start, end }` for the first longest run.

## Problem 13 — Find the Union of Two Arrays

### 1. Problem Statement
Given two sorted arrays, return their sorted union with each distinct value once.

### 2. Example
```text
Input: [1, 2, 2, 4], [2, 3, 4]
Output: [1, 2, 3, 4]
```

### 3. Understand the Problem
Union means values present in either array. Inputs are sorted and unchanged; output is sorted and distinct.

#### In Simple Terms

Combine both sorted arrays, keep each value only once, and return the values in sorted order.

### 4. Real-World Analogy
Merge two alphabetized invite lists while writing each name once.

### 5. Key Observation
The smallest unprocessed value must be at one of the two current pointers.

### 6. Pattern
```text
Pattern: Merge two sorted arrays
```
Recognize it when two ordered inputs must produce one ordered output.

### 7. Brute Force
Add all values to a set, convert to an array, and sort.

#### Pseudocode
```text
set = values from both arrays
result = sorted set values
```

#### Brute-Force TypeScript

```typescript
function sortedUnionWithSet(left: number[], right: number[]): number[] {
  return [...new Set([...left, ...right])].sort((first, second) => first - second);
}
```

#### Visualization
```text
[1,2,2,4] + [2,3,4] -> {1,2,3,4} -> [1,2,3,4]
```

#### Dry Run
The set removes duplicate `2` and `4`; sorting restores numeric order.

#### Brute-Force Complexity
For total size `n + m`, insertion is average `O(n+m)` and sorting is `O((n+m) log(n+m))`; space is `O(n+m)`.

### 8. Why Brute Force Is Not Ideal
It discards and then reconstructs order that the inputs already provide.

### 9. Optimal Approach
Compare pointer values. Append the smaller, or append once and advance both when equal. A helper avoids duplicating the last output value.

### 10. Visualization
```text
[1,2,2,4]
 ^
[2,3,4]
 ^
take 1, then equal 2, skip duplicate 2, take 3, equal 4
```

### 11. Step-by-Step Dry Run
| Left | Right | Action | Result |
|---:|---:|---|---|
| 1 | 2 | Take left | `[1]` |
| 2 | 2 | Take once, advance both | `[1,2]` |
| 2 | 3 | Duplicate output, advance left | `[1,2]` |
| 4 | 3 | Take right | `[1,2,3]` |
| 4 | 4 | Take once | `[1,2,3,4]` |

### 12. TypeScript Solution
```typescript
function sortedUnion(left: number[], right: number[]): number[] {
  const result: number[] = [];
  let i = 0;
  let j = 0;
  const append = (value: number) => {
    if (result.length === 0 || result[result.length - 1] !== value) result.push(value);
  };
  while (i < left.length && j < right.length) {
    if (left[i] < right[j]) append(left[i++]);
    else if (left[i] > right[j]) append(right[j++]);
    else { append(left[i]); i += 1; j += 1; }
  }
  while (i < left.length) append(left[i++]);
  while (j < right.length) append(right[j++]);
  return result;
}
```

### 13. Code Walkthrough
Pointers only move forward. `append` enforces output distinctness, including duplicates within one input. Tail loops drain the unfinished array.

### 14. Complexity Analysis
**Time: `O(n+m)`** because each pointer advances at most its input length. **Output space: `O(n+m)`** in the all-distinct case; auxiliary pointer state is `O(1)` excluding output.

### 15. Edge Cases
- Either or both inputs empty.
- All values overlap.
- Duplicates inside either array.
- Negative values remain numerically sorted.

### 16. Common Mistakes
- Returning duplicate values.
- Forgetting remaining tails.
- Sorting again and losing linear time.
- Advancing only one pointer when values are equal.

### 17. Interview Follow-ups
1. Inputs are unsorted.
2. Preserve multiplicity instead of distinctness.
3. Compute union of `k` sorted arrays.

### 18. Variations
- Multiset union.
- Set difference.
- Lazy generator for large streams.

### 19. Pattern Connections
This merge pattern powers merge sort, interval merging, and external data merges.

### 20. NestJS Project Connection
Expose `POST /api/dsa/arrays/union` with `left` and `right`; validate sortedness and trace pointer decisions plus appended values.

### 21. Mini Challenge
Return the sorted set difference `left - right` in `O(n+m)` time.

## Problem 14 — Find the Intersection of Two Arrays

### 1. Problem Statement
Given two sorted arrays, return sorted distinct values present in both.

### 2. Example
```text
Input: [1, 2, 2, 4], [2, 2, 3, 4]
Output: [2, 4]
```

### 3. Understand the Problem
Intersection requires membership in both arrays. Duplicate matches appear once in output.

#### In Simple Terms

Return the distinct values that appear in both sorted arrays, in sorted order.

### 4. Real-World Analogy
Walk two alphabetized rosters and record only names appearing on both.

### 5. Key Observation
If one current value is smaller, it cannot match the other's current or later values, so advance its pointer.

### 6. Pattern
```text
Pattern: Coordinated pointers on sorted inputs
```
Sorted order safely eliminates impossible candidates.

### 7. Brute Force
Compare every value in the first array with every value in the second, adding matches to a set.

#### Pseudocode
```text
for each left value
    for each right value
        if equal, add to result set
```

#### Brute-Force TypeScript

```typescript
function sortedIntersectionByComparingAll(left: number[], right: number[]): number[] {
  const matches = new Set<number>();
  for (const leftValue of left) {
    for (const rightValue of right) {
      if (leftValue === rightValue) matches.add(leftValue);
    }
  }
  return [...matches].sort((first, second) => first - second);
}
```

#### Visualization
```text
1 checks all; 2 finds 2; 4 finds 4
```

#### Dry Run
Matches create set `{2,4}`, converted to `[2,4]`.

#### Brute-Force Complexity
`n x m` comparisons give `O(nm)` time; output set space is `O(min(n,m))`.

### 8. Why Brute Force Is Not Ideal
It ignores sorted order and compares pairs that cannot match.

### 9. Optimal Approach
When values differ, advance the pointer at the smaller value. When equal, append once and advance both.

### 10. Visualization
```text
left:  [1,2,2,4]
        ^
right: [2,2,3,4]
        ^
1 < 2 -> advance left; 2 = 2 -> record; continue
```

### 11. Step-by-Step Dry Run
| Left | Right | Action | Result |
|---:|---:|---|---|
| 1 | 2 | Advance left | `[]` |
| 2 | 2 | Record, advance both | `[2]` |
| 2 | 2 | Duplicate result, advance both | `[2]` |
| 4 | 3 | Advance right | `[2]` |
| 4 | 4 | Record | `[2,4]` |

### 12. TypeScript Solution
```typescript
function sortedIntersection(left: number[], right: number[]): number[] {
  const result: number[] = [];
  let i = 0;
  let j = 0;
  while (i < left.length && j < right.length) {
    if (left[i] < right[j]) i += 1;
    else if (left[i] > right[j]) j += 1;
    else {
      if (result.at(-1) !== left[i]) result.push(left[i]);
      i += 1;
      j += 1;
    }
  }
  return result;
}
```

### 13. Code Walkthrough
Only equality produces output. Every branch advances at least one pointer, guaranteeing progress and termination.

### 14. Complexity Analysis
**Time: `O(n+m)`** because pointers never move backward. **Output space: `O(min(n,m))`**; auxiliary pointer storage is `O(1)` excluding output.

### 15. Edge Cases
- Empty input gives empty intersection.
- No overlap returns `[]`.
- Complete overlap returns distinct shared values.
- Duplicates are emitted once.

### 16. Common Mistakes
- Advancing neither pointer after equality.
- Emitting duplicate matches.
- Forgetting the sorted precondition.
- Calling output storage auxiliary storage without clarifying.

### 17. Interview Follow-ups
1. Inputs are unsorted.
2. Keep duplicate multiplicity.
3. One array is much smaller.
4. Inputs are streams.

### 18. Variations
- Intersection of three sorted arrays.
- Multiset intersection.
- Set difference.

### 19. Pattern Connections
Sorted elimination appears in two-sum, merge operations, and database merge joins.

### 20. NestJS Project Connection
Use `POST /api/dsa/arrays/intersection`; trace comparisons and which pointer advances, with sorted-input validation.

### 21. Mini Challenge
Return duplicates according to the minimum frequency in the two arrays.

## Problem 15 — Left Rotate an Array by One Position

### 1. Problem Statement
Rotate an array left by one position in place: move the first value to the end and shift every other value left.

### 2. Example
```text
Input: [1, 2, 3, 4]
After: [2, 3, 4, 1]
```

### 3. Understand the Problem
Rotation wraps a value around; it does not discard it. Empty and one-value arrays remain unchanged.

#### In Simple Terms

Move the first value to the end and shift every other value one place to the left.

### 4. Real-World Analogy
The first person in a circular turn order moves to the back after taking a turn.

### 5. Key Observation
Save the first value before overwriting it, shift indexes `1..n-1` left, then place the saved value last.

### 6. Pattern
```text
Pattern: Save, shift, restore
```
Recognize one-position rotations and removals where an overwritten boundary value must be preserved.

### 7. Brute Force
Create a new array by copying indexes `1..n-1`, then append index `0`.

#### Pseudocode
```text
result = values from index 1 onward
append first value
```

#### Brute-Force TypeScript

```typescript
function rotateLeftOnceWithCopy<T>(values: T[]): void {
  if (values.length < 2) return;
  const rotated = [...values.slice(1), values[0]];
  for (let index = 0; index < values.length; index += 1) values[index] = rotated[index];
}
```

#### Visualization
```text
[1 | 2,3,4] -> [2,3,4 | 1]
```

#### Dry Run
Copy `2,3,4`, then append saved `1`.

#### Brute-Force Complexity
Copying `n` values takes `O(n)` time and `O(n)` output space.

### 8. Why Brute Force Is Not Ideal
Mutation is allowed, so only the displaced first value needs temporary storage.

### 9. Optimal Approach
Save `values[0]`; assign `values[i-1] = values[i]` for every later index; restore the saved value at `n-1`.

### 10. Visualization
```text
save 1
[1,2,3,4] -> [2,2,3,4] -> [2,3,3,4] -> [2,3,4,4]
restore -> [2,3,4,1]
```

### 11. Step-by-Step Dry Run
| Source index | Write index | Value | State |
|---:|---:|---:|---|
| 1 | 0 | 2 | `[2,2,3,4]` |
| 2 | 1 | 3 | `[2,3,3,4]` |
| 3 | 2 | 4 | `[2,3,4,4]` |
| restore | 3 | 1 | `[2,3,4,1]` |

### 12. TypeScript Solution
```typescript
function rotateLeftOnce<T>(values: T[]): void {
  if (values.length < 2) return;
  const first = values[0];
  for (let index = 1; index < values.length; index += 1) {
    values[index - 1] = values[index];
  }
  values[values.length - 1] = first;
}
```

### 13. Code Walkthrough
The guard avoids invalid access. Saving occurs before shifts overwrite index `0`; left-to-right copying is safe because each source is read before being overwritten.

### 14. Complexity Analysis
**Time: `O(n)`** because `n-1` shifts occur. **Auxiliary space: `O(1)`** because one saved value and one index are stored.

### 15. Edge Cases
- Empty and one-value arrays do nothing.
- Two values swap.
- Generic values move by reference without cloning.

### 16. Common Mistakes
- Failing to save the first value.
- Shifting in the wrong direction and propagating one value.
- Treating rotation as reversal.
- Allocating a full copy while claiming constant space.

### 17. Interview Follow-ups
1. Rotate left by `k`.
2. Rotate right by one.
3. Return a non-mutating rotated copy.

### 18. Variations
- Circular queue advancement.
- String rotation.
- Rotate a subarray.

### 19. Pattern Connections
One-step rotation motivates reversal-based `k` rotation and modular index mapping.

### 20. NestJS Project Connection
Expose `POST /api/dsa/arrays/rotate-left-one`; execute on a request copy and trace `{ sourceIndex, destinationIndex, value, state }` for small inputs.

### 21. Mini Challenge
Implement an in-place right rotation by one and explain why its shift direction must reverse.

## Problem 16 — Left Rotate an Array by k Positions

### 1. Problem Statement
Rotate an array left by `k` positions in place.

### 2. Example
```text
Input: [1,2,3,4,5], k = 2
After: [3,4,5,1,2]
```

### 3. Understand the Problem
Rotation wraps displaced values around. Assume `k >= 0`; normalize `k` because rotating by the length changes nothing.

#### In Simple Terms

Move the first `k` positions around to the end, preserving the order of all values.

### 4. Real-World Analogy
Advance a circular schedule by `k` turns; skipped people rejoin at the end in the same order.

### 5. Key Observation
Left rotation changes `[A | B]` into `[B | A]`. Three reversals perform that rearrangement in place.

### 6. Pattern
```text
Pattern: Segment reversal
```
Recognize rotations requiring linear time and constant auxiliary space.

### 7. Brute Force
Repeat the one-position left rotation `k` times.

#### Pseudocode
```text
repeat k times
    save first
    shift all remaining values left
    restore first at end
```

#### Brute-Force TypeScript

```typescript
function rotateLeftRepeatedly<T>(values: T[], k: number): void {
  if (values.length === 0) return;
  const shifts = k % values.length;
  for (let count = 0; count < shifts; count += 1) {
    const first = values[0];
    for (let index = 1; index < values.length; index += 1) {
      values[index - 1] = values[index];
    }
    values[values.length - 1] = first;
  }
}
```

#### Visualization
```text
[1,2,3,4,5] -> [2,3,4,5,1] -> [3,4,5,1,2]
```

#### Dry Run
Each of two rounds shifts four values and restores one.

#### Brute-Force Complexity
Each rotation is `O(n)` and repeats `k` times: `O(nk)` time, `O(1)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
Values are shifted repeatedly, especially when `k` is large.

### 9. Optimal Approach
Normalize `k %= n`; reverse `[0,k-1]`, reverse `[k,n-1]`, then reverse the whole array.

### 10. Visualization
```text
[1,2 | 3,4,5]
[2,1 | 3,4,5]
[2,1 | 5,4,3]
[3,4,5,1,2]
```

### 11. Step-by-Step Dry Run
For `k=2`, the three states above show the first segment reversed, second segment reversed, and final whole-array reversal.

### 12. TypeScript Solution
```typescript
function reverseRange<T>(values: T[], left: number, right: number): void {
  while (left < right) {
    [values[left], values[right]] = [values[right], values[left]];
    left += 1;
    right -= 1;
  }
}

function rotateLeft<T>(values: T[], k: number): void {
  if (values.length === 0) return;
  const shift = k % values.length;
  reverseRange(values, 0, shift - 1);
  reverseRange(values, shift, values.length - 1);
  reverseRange(values, 0, values.length - 1);
}
```

### 13. Code Walkthrough
Normalization removes complete cycles. Empty ranges safely perform no swaps. Reversals restore internal order while exchanging the two segments.

### 14. Complexity Analysis
**Time: `O(n)`** because the three reversals together perform a constant multiple of `n` swaps. **Auxiliary space: `O(1)`**.

### 15. Edge Cases
- Empty and one-value arrays.
- `k=0`, `k=n`, or `k>n`.
- This contract excludes negative `k`.

### 16. Common Mistakes
- Taking modulo zero for empty input.
- Forgetting to normalize `k`.
- Reversing segments in the wrong boundaries.
- Calling three sequential linear operations `O(n^3)`.

### 17. Interview Follow-ups
1. Support negative rotations.
2. Return a rotated copy.
3. Use the juggling algorithm.

### 18. Variations
- Rotate only a subarray.
- Rotate strings.
- Circular buffer indexing.

### 19. Pattern Connections
Segment reversal also appears in reversing words and block-swap algorithms.

### 20. NestJS Project Connection
Use `POST /api/dsa/arrays/rotate-left` with `values` and `k`; trace three named reversal phases rather than every unchanged state.

### 21. Mini Challenge
Normalize negative `k` so `-1` means one right rotation.

## Problem 17 — Right Rotate an Array by k Positions

### 1. Problem Statement
Rotate an array right by `k` positions in place.

### 2. Example
```text
Input: [1,2,3,4,5], k = 2
After: [4,5,1,2,3]
```

### 3. Understand the Problem
The last `k` values wrap to the front. Assume non-negative `k` and preserve all values.

#### In Simple Terms

Move the last `k` positions around to the front, preserving the order of all values.

### 4. Real-World Analogy
Move the final `k` people in a circular line to the front without changing either group's internal order.

### 5. Key Observation
For `[A | B]`, right rotation needs `[B | A]`. Reversing the whole array and then each resulting segment restores their order.

### 6. Pattern
```text
Pattern: Segment reversal
```
The direction changes boundaries, not the fundamental rotation idea.

### 7. Brute Force
Repeat: save the last value, shift everything right, place the saved value first.

#### Pseudocode
```text
repeat k times
    save last
    shift values right
    restore saved at index 0
```

#### Brute-Force TypeScript

```typescript
function rotateRightRepeatedly<T>(values: T[], k: number): void {
  if (values.length === 0) return;
  const shifts = k % values.length;
  for (let count = 0; count < shifts; count += 1) {
    const last = values[values.length - 1];
    for (let index = values.length - 1; index > 0; index -= 1) {
      values[index] = values[index - 1];
    }
    values[0] = last;
  }
}
```

#### Visualization
```text
[1,2,3,4,5] -> [5,1,2,3,4] -> [4,5,1,2,3]
```

#### Dry Run
Two rounds each shift four positions.

#### Brute-Force Complexity
`k` rounds of `O(n)` shifting give `O(nk)` time and `O(1)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
Repeated shifts redo nearly the same movement.

### 9. Optimal Approach
Normalize the shift. Reverse the whole array, then reverse the first `shift` values and the remaining suffix.

### 10. Visualization
```text
[1,2,3 | 4,5]
whole:  [5,4,3,2,1]
first:  [4,5,3,2,1]
rest:   [4,5,1,2,3]
```

### 11. Step-by-Step Dry Run
For shift `2`, reverse indexes `0..4`, then `0..1`, then `2..4`, producing the target.

### 12. TypeScript Solution
```typescript
function rotateRight<T>(values: T[], k: number): void {
  if (values.length === 0) return;
  const shift = k % values.length;
  reverseRange(values, 0, values.length - 1);
  reverseRange(values, 0, shift - 1);
  reverseRange(values, shift, values.length - 1);
}
```

### 13. Code Walkthrough
This reuses Problem 16's helper. The first reversal exchanges segment positions but reverses their internals; the next two repair those internals.

### 14. Complexity Analysis
**Time: `O(n)`** for three sequential reversal passes. **Auxiliary space: `O(1)`** because all swaps occur in place.

### 15. Edge Cases
- Empty input avoids modulo zero.
- Shift zero makes reversals cancel to the original.
- `k` larger than length is normalized.

### 16. Common Mistakes
- Using left-rotation boundaries.
- Shifting left during a right rotation.
- Forgetting the empty guard.
- Mutating despite a non-mutating caller expectation.

### 17. Interview Follow-ups
1. Derive right rotation from left rotation.
2. Support signed `k`.
3. Rotate a linked list.

### 18. Variations
- Circular deque operations.
- Rotate matrix layers.
- Lazy rotated view using modular indexes.

### 19. Pattern Connections
The same normalization and modular movement appear in circular arrays and ring buffers.

### 20. NestJS Project Connection
Expose `POST /api/dsa/arrays/rotate-right`; include normalized shift and reversal phase boundaries in the visualization response.

### 21. Mini Challenge
Return `valueAtRotatedIndex(i)` without physically rotating the array.

## Problem 18 — Find the Maximum Difference arr[j] - arr[i] Where j > i

### 1. Problem Statement
Return the largest difference `values[j] - values[i]` with `j > i`. Require at least two values.

### 2. Example
```text
Input: [7, 1, 5, 3, 6, 4]
Output: 5  (6 - 1)
```

### 3. Understand the Problem
Order matters: the subtracted value must occur earlier. A negative answer is valid when the array strictly decreases.

#### In Simple Terms

Choose an earlier value to subtract from a later value and return the largest possible result.

### 4. Real-World Analogy
Find the best later selling price after an earlier purchase, even if every possible trade loses money.

### 5. Key Observation
For each later value, the best earlier partner is the smallest value seen before it.

### 6. Pattern
```text
Pattern: Prefix minimum plus global best
```
Recognize ordered pair optimization where one side must precede the other.

### 7. Brute Force
Compute the difference for every valid pair `i < j`.

#### Pseudocode
```text
best = -Infinity
for i
    for j after i
        best = max(best, values[j] - values[i])
```

#### Brute-Force TypeScript

```typescript
function maximumForwardDifferenceByPairs(values: number[]): number {
  if (values.length < 2) throw new Error("Expected at least two values");
  let best = Number.NEGATIVE_INFINITY;
  for (let first = 0; first < values.length - 1; first += 1) {
    for (let second = first + 1; second < values.length; second += 1) {
      best = Math.max(best, values[second] - values[first]);
    }
  }
  return best;
}
```

#### Visualization
```text
7 pairs with 1,5,3,6,4
1 pairs with 5,3,6,4
... every forward pair
```

#### Dry Run
Among all pairs, indexes `1` and `4` produce `6 - 1 = 5`.

#### Brute-Force Complexity
There are `n(n-1)/2` valid pairs, so time is `O(n^2)`; auxiliary space is `O(1)`.

### 8. Why Brute Force Is Not Ideal
Each later value is compared with many earlier values even though only their minimum matters.

### 9. Optimal Approach
Initialize `minimum` to index `0` and `best` to the first valid difference. For each later value, evaluate its difference before updating the minimum.

### 10. Visualization
```text
value:    7   1   5   3   6   4
min:      7   1   1   1   1   1
best:       -6   4   4   5   5
```

### 11. Step-by-Step Dry Run
| Current | Earlier minimum | Difference | Best after |
|---:|---:|---:|---:|
| 1 | 7 | -6 | -6 |
| 5 | 1 | 4 | 4 |
| 3 | 1 | 2 | 4 |
| 6 | 1 | 5 | 5 |
| 4 | 1 | 3 | 5 |

### 12. TypeScript Solution
```typescript
function maximumForwardDifference(values: number[]): number {
  if (values.length < 2) throw new Error("Expected at least two values");
  let minimum = values[0];
  let best = values[1] - values[0];
  for (let index = 1; index < values.length; index += 1) {
    best = Math.max(best, values[index] - minimum);
    minimum = Math.min(minimum, values[index]);
  }
  return best;
}
```

### 13. Code Walkthrough
Computing the difference before updating `minimum` guarantees the earlier index condition. Initializing with a real pair preserves negative answers.

### 14. Complexity Analysis
**Time: `O(n)`** because each later value is processed once. **Auxiliary space: `O(1)`** for minimum and best.

### 15. Edge Cases
- Fewer than two values violates the contract.
- Strictly decreasing input returns the least negative loss.
- Duplicate values can produce zero.
- Negative numbers work normally.

### 16. Common Mistakes
- Returning zero for decreasing input without permission to skip the pair.
- Using the global minimum even if it occurs after `j`.
- Updating minimum before evaluating the pair.
- Sorting and destroying index order.

### 17. Interview Follow-ups
1. Return the two indexes.
2. Permit no transaction, making zero possible.
3. Maximum absolute difference without order.

### 18. Variations
- Best stock trade.
- Maximum difference within a fixed distance.
- Multiple buy/sell transactions.

### 19. Pattern Connections
Prefix extrema support stock profit, max subarray ideas, and one-pass ordered-pair optimization.

### 20. NestJS Project Connection
Use `POST /api/dsa/arrays/maximum-forward-difference`; trace `{ index, current, minimumBefore, difference, bestAfter }`.

### 21. Mini Challenge
Return `{ difference, buyIndex, sellIndex }`, keeping the earliest valid pair on ties.

## Problem 19 — Rearrange Positive and Negative Numbers

### 1. Problem Statement
Return a new array alternating non-negative and negative values, starting with non-negative. Preserve relative order within each sign; append leftovers when one group ends.

### 2. Example
```text
Input:  [3, 1, -2, -5, 2, -4]
Output: [3, -2, 1, -5, 2, -4]
```

### 3. Understand the Problem
Zero belongs to the non-negative group. Stability and the starting sign are explicit. A new array is allowed.

#### In Simple Terms

Build a new array by taking a nonnegative value, then a negative value, while keeping each group's original order. Append leftovers when one group runs out.

### 4. Real-World Analogy
Interleave two waiting lines, taking one person from each while preserving each line's arrival order.

### 5. Key Observation
Stable alternation is simple after separating values into two ordered queues.

### 6. Pattern
```text
Pattern: Stable partition then interleave
```
Recognize it when categories must alternate without losing internal order.

### 7. Brute Force
Repeatedly scan the remaining array for the next required sign and remove it.

#### Pseudocode
```text
need non-negative, then negative
search remaining values for needed sign
remove and append it; repeat
```

#### Brute-Force TypeScript

```typescript
function alternateSignsByRepeatedSearch(values: number[]): number[] {
  const remaining = [...values];
  const result: number[] = [];
  let needNonnegative = true;
  while (remaining.length > 0) {
    const index = remaining.findIndex((value) => needNonnegative ? value >= 0 : value < 0);
    if (index === -1) {
      result.push(...remaining);
      break;
    }
    result.push(remaining[index]);
    remaining.splice(index, 1);
    needNonnegative = !needNonnegative;
  }
  return result;
}
```

#### Visualization
```text
find 3, rescan for -2, rescan for 1, rescan for -5...
```

#### Dry Run
Repeated searches produce `[3,-2,1,-5,2,-4]` but revisit and shift values.

#### Brute-Force Complexity
Up to `n` searches/removals each cost `O(n)`, producing `O(n^2)` time and `O(n)` output space.

### 8. Why Brute Force Is Not Ideal
It repeatedly searches and removal may shift the array.

### 9. Optimal Approach
Traverse once into stable positive and negative arrays, then append one from each while available and drain leftovers.

### 10. Visualization
```text
non-negative: [3,1,2]
negative:     [-2,-5,-4]
zip:          [3,-2,1,-5,2,-4]
```

### 11. Step-by-Step Dry Run
| Positive pointer | Negative pointer | Appended | Result |
|---:|---:|---|---|
| 0 | 0 | 3, -2 | `[3,-2]` |
| 1 | 1 | 1, -5 | `[3,-2,1,-5]` |
| 2 | 2 | 2, -4 | `[3,-2,1,-5,2,-4]` |

### 12. TypeScript Solution
```typescript
function alternateSigns(values: number[]): number[] {
  const nonNegative = values.filter((value) => value >= 0);
  const negative = values.filter((value) => value < 0);
  const result: number[] = [];
  let positiveIndex = 0;
  let negativeIndex = 0;
  while (positiveIndex < nonNegative.length && negativeIndex < negative.length) {
    result.push(nonNegative[positiveIndex++], negative[negativeIndex++]);
  }
  result.push(...nonNegative.slice(positiveIndex), ...negative.slice(negativeIndex));
  return result;
}
```

### 13. Code Walkthrough
Filtering preserves encounter order. The paired loop enforces alternation; the final append implements the leftover contract.

### 14. Complexity Analysis
**Time: `O(n)`**: two filter passes plus interleaving are sequential linear work. **Space: `O(n)`** for category arrays and output. Slices also allocate leftover arrays; an indexed drain can reduce constants but not Big-O.

### 15. Edge Cases
- Empty returns `[]`.
- One sign only returns that sign's stable order.
- Zero is non-negative.
- Unequal groups leave a same-sign suffix.

### 16. Common Mistakes
- Failing to specify starting sign or zero treatment.
- Claiming strict alternation is possible with unequal counts.
- Losing stable order through swaps.
- Calling multiple sequential passes quadratic.

### 17. Interview Follow-ups
1. Rearrange in place without stability.
2. Start with the majority sign.
3. Require strict alternation or report failure.

### 18. Variations
- Alternate odd and even.
- Interleave two arrays.
- Stable three-category arrangement.

### 19. Pattern Connections
Stable partition plus merge appears in scheduling, UI feeds, and grouped stream interleaving.

### 20. NestJS Project Connection
Expose `POST /api/dsa/arrays/alternate-signs`; return partition and interleave phases so learners see the space tradeoff.

### 21. Mini Challenge
Start with whichever sign has more values while preserving stability and maximizing alternation.

## Problem 20 — Find the Leaders in an Array

### 1. Problem Statement
Return values that are strictly greater than every value to their right, preserving original order. The last value is always a leader.

### 2. Example
```text
Input:  [16, 17, 4, 3, 5, 2]
Output: [17, 5, 2]
```

### 3. Understand the Problem
"Strictly greater" means an equal value to the right prevents leadership. Output follows left-to-right input order.

#### In Simple Terms

Return each value that is strictly larger than every value to its right. Keep the leaders in their original order.

### 4. Real-World Analogy
Looking from the right edge of a skyline, a building is visible only if it is taller than every building closer to you.

### 5. Key Observation
Scanning right to left, only the maximum value already seen to the right matters.

### 6. Pattern
```text
Pattern: Suffix maximum during reverse traversal
```
Recognize right-side comparison questions where a suffix can be summarized by one extreme.

### 7. Brute Force
For each index, scan every value to its right and reject it if any is greater or equal.

#### Pseudocode
```text
for each index i
    if value[i] is greater than all values after i
        append it
```

#### Brute-Force TypeScript

```typescript
function findLeadersByScanningSuffixes(values: number[]): number[] {
  const leaders: number[] = [];
  for (let index = 0; index < values.length; index += 1) {
    let isLeader = true;
    for (let right = index + 1; right < values.length; right += 1) {
      if (values[right] >= values[index]) {
        isLeader = false;
        break;
      }
    }
    if (isLeader) leaders.push(values[index]);
  }
  return leaders;
}
```

#### Visualization
```text
16 sees 17 -> reject
17 sees 4,3,5,2 -> leader
4 sees 5 -> reject
```

#### Dry Run
Checking all suffixes identifies `17`, `5`, and last value `2`.

#### Brute-Force Complexity
Suffix lengths total `(n-1)+(n-2)+...+1 = O(n^2)` time. Output uses up to `O(n)` space.

### 8. Why Brute Force Is Not Ideal
Suffix values are rescanned. Their only needed summary is the suffix maximum.

### 9. Optimal Approach
Start from the right with `maximumRight = -Infinity`. Record a value when it is greater, then update the maximum. Reverse collected leaders at the end.

### 10. Visualization
```text
scan: 2 <- 5 <- 3 <- 4 <- 17 <- 16
      L    L    -    -     L     -
collected right-to-left: [2,5,17]
reverse:                 [17,5,2]
```

### 11. Step-by-Step Dry Run
| Value | Maximum right before | Leader? | Collected |
|---:|---:|:---:|---|
| 2 | `-Infinity` | Yes | `[2]` |
| 5 | 2 | Yes | `[2,5]` |
| 3 | 5 | No | `[2,5]` |
| 4 | 5 | No | `[2,5]` |
| 17 | 5 | Yes | `[2,5,17]` |
| 16 | 17 | No | `[2,5,17]` |

### 12. TypeScript Solution
```typescript
function findLeaders(values: number[]): number[] {
  const leaders: number[] = [];
  let maximumRight = -Infinity;
  for (let index = values.length - 1; index >= 0; index -= 1) {
    if (values[index] > maximumRight) {
      leaders.push(values[index]);
      maximumRight = values[index];
    }
  }
  leaders.reverse();
  return leaders;
}
```

### 13. Code Walkthrough
Before each iteration, `maximumRight` summarizes all positions strictly to the right. Leaders are discovered backward, so one final reversal restores input order.

### 14. Complexity Analysis
**Time: `O(n)`**: one reverse scan plus reversing at most `n` leaders gives `n + n = O(n)`. **Output space: `O(n)`** worst case for a decreasing array; auxiliary state excluding output is `O(1)`.

### 15. Edge Cases
- Empty returns `[]`.
- One value is a leader.
- Strictly decreasing means every value leads.
- Equal values: only the rightmost equal value can lead under the strict contract.

### 16. Common Mistakes
- Using `>=` and changing strict semantics.
- Forgetting to reverse discovered leaders.
- Treating the current value as part of its own right suffix.
- Calling output storage constant space.

### 17. Interview Follow-ups
1. Use greater-than-or-equal leader semantics.
2. Return indexes.
3. Produce leaders online from right-to-left input.

### 18. Variations
- Replace each value with greatest value to its right.
- Suffix minima.
- Stock-span-style visibility.

### 19. Pattern Connections
Suffix extrema appear in rainwater, product-except-self, next-greater, and right-side replacement problems.

### 20. NestJS Project Connection
Use `POST /api/dsa/arrays/leaders`; trace `{ index, value, maximumRightBefore, isLeader }` and return leaders in original order.

### 21. Mini Challenge
Return leader indexes without a final sort and explain the required output reversal.

---

# Topic Summary

## What You Learned

- Arrays preserve order and provide direct `O(1)` indexed access.
- Full traversal is `O(n)`; front shifts and repeated nested comparisons can be expensive.
- Running extrema summarize processed prefixes or suffixes.
- Read/write pointers compact arrays; inward pointers reverse and partition them.
- Sorted order enables linear merge, union, intersection, and adjacent duplicate removal.
- Sets trade `O(n)` memory for average constant-time membership on unsorted data.
- Rotation can use segment reversal for `O(n)` time and `O(1)` auxiliary space.
- Mutation, stability, distinctness, and tie behavior must be part of the contract.

## Patterns You Should Recognize

- traversal and accumulation for extrema, streaks, and leaders
- read/write pointers for compaction and stable movement
- inward pointers for reversal and unstable partitioning
- hash-based membership for unsorted duplicates
- expected-versus-observed arithmetic or XOR cancellation
- merge pointers for two sorted arrays
- prefix minimum and suffix maximum state
- segment reversal for rotation

## Complexity Cheat Sheet

| Operation | Time | Space |
|---|---:|---:|
| Read or replace by index | `O(1)` | `O(1)` |
| Traverse all values | `O(n)` | `O(1)` auxiliary |
| Linear search | `O(n)` | `O(1)` |
| Compare every pair | `O(n^2)` | `O(1)` auxiliary |
| Copy an array | `O(n)` | `O(n)` |
| Sort numeric values | Usually `O(n log n)` | Engine-dependent |
| Set-based deduplication | Average `O(n)` | `O(n)` |
| Merge two sorted arrays | `O(n+m)` | Output-dependent |
| In-place reverse or rotation | `O(n)` | `O(1)` auxiliary |

## Common Interview Traps

- Using index `n` as the last valid index instead of `n - 1`.
- Mutating input without saying so.
- Ignoring whether duplicates are distinct candidates.
- Losing stable order when the contract requires it.
- Sorting and destroying an original-order constraint.
- Forgetting output space or hidden copies from spread, filter, and slice.
- Calling monotonic nested pointer loops quadratic.
- Failing to normalize rotation counts.
- Initializing extrema to zero and breaking all-negative inputs.

## Practice Checklist

- [ ] Problem 1 — Find the largest element in an array
- [ ] Problem 2 — Find the second largest element
- [ ] Problem 3 — Find the smallest and second smallest element
- [ ] Problem 4 — Check if an array is sorted
- [ ] Problem 5 — Reverse an array
- [ ] Problem 6 — Remove duplicates from a sorted array
- [ ] Problem 7 — Remove duplicates from an unsorted array
- [ ] Problem 8 — Move all zeroes to the end
- [ ] Problem 9 — Move all negative numbers to one side
- [ ] Problem 10 — Find the missing number from 1..n
- [ ] Problem 11 — Find the number that appears once
- [ ] Problem 12 — Find the maximum consecutive 1s
- [ ] Problem 13 — Find the union of two arrays
- [ ] Problem 14 — Find the intersection of two arrays
- [ ] Problem 15 — Left rotate an array by one position
- [ ] Problem 16 — Left rotate an array by k positions
- [ ] Problem 17 — Right rotate an array by k positions
- [ ] Problem 18 — Find the Maximum Difference arr[j] - arr[i] Where j > i
- [ ] Problem 19 — Rearrange positive and negative numbers
- [ ] Problem 20 — Find the leaders in an array

