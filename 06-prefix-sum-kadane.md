# Prefix Sum & Kadane

## Learning Objectives

By the end of this topic, you should understand:

- how prefix state summarizes everything seen before an index
- the difference between an inclusive running sum and a leading-zero prefix array
- how two prefix values answer an immutable range-sum query
- how left and right totals identify equilibrium and pivot indexes
- how a prefix-frequency map counts subarrays even when values are negative
- how Kadane's algorithm chooses between extending and restarting a subarray
- how to adapt maximum-subarray reasoning to circular arrays
- how running minima and local gains solve two stock-profit contracts
- how prefix and suffix products avoid division
- how to define mutation, empty-input, tie-breaking, and numeric-safety contracts

## Prerequisites

- contracts, invariants, and complexity reasoning from `00-dsa-fundamentals.md`
- array traversal and contiguous subarrays from `01-arrays.md`
- maps and frequency counting from `02-hashing.md`
- same-direction pointer movement from `04-two-pointers.md`
- sliding-window limitations with negative values from `05-sliding-window.md`
- TypeScript arrays, maps, loops, functions, and safe integers

## Core Theory

### What is prefix state?

A prefix is the portion of an array from its beginning through some boundary. Prefix algorithms carry a summary of that portion while scanning left to right.

For `values = [3, -1, 4, 2]`, the inclusive running sums are:

```text
index:        0   1   2   3
value:        3  -1   4   2
running sum:  3   2   6   8
```

At index `i`, the running value already summarizes `values[0..i]`. The algorithm does not need to rescan earlier elements.

### Two useful prefix-array shapes

An **inclusive prefix array** has the same length as the input:

```text
inclusive[i] = values[0] + ... + values[i]
```

A **leading-zero prefix array** has length `n + 1`:

```text
prefix[0] = 0
prefix[i + 1] = prefix[i] + values[i]
```

The leading zero makes half-open range sums especially clean:

```text
sum of values[left..right] = prefix[right + 1] - prefix[left]
```

Use names and index diagrams to avoid mixing the two conventions.

### Why subtraction isolates a range

Suppose `prefix[right + 1]` contains everything through `right`. It includes the values before `left`, represented by `prefix[left]`. Subtracting removes exactly that earlier portion.

```text
prefix[right + 1] = before-left + requested-range
prefix[left]      = before-left
difference        = requested-range
```

This preprocessing costs `O(n)`, after which each valid immutable range query costs `O(1)`.

### Left totals, right totals, and balance

At index `i`, let `leftSum` contain values strictly before `i`. If `totalSum` is known, the right side is:

```text
rightSum = totalSum - leftSum - values[i]
```

After checking index `i`, add `values[i]` to `leftSum`. This produces an `O(n)` balance scan with `O(1)` auxiliary space.

### Prefix sums with a frequency map

For a subarray ending at the current index:

```text
subarraySum = currentPrefix - earlierPrefix
```

To make that sum equal `k`:

```text
earlierPrefix = currentPrefix - k
```

A map from prefix sum to occurrence count tells us how many valid starts exist. Seed `0 -> 1` so a subarray beginning at index `0` is counted. Unlike a basic shrinking window, this method remains correct with negative values and zeros.

### Kadane's local decision

Kadane's algorithm tracks the maximum sum of a non-empty subarray ending exactly at the current index.

For each value, that subarray must either:

1. start at the current value, or
2. extend the best subarray ending at the previous index.

```text
current = max(value, current + value)
best = max(best, current)
```

Initializing from the first value, rather than zero, preserves the non-empty contract for all-negative arrays.

### Circular maximum subarrays

A maximum circular subarray is either:

- non-wrapping: the ordinary Kadane maximum, or
- wrapping: the total sum minus one minimum contiguous middle subarray

```text
maximum circular sum = max(maximum normal, total - minimum normal)
```

If every value is negative, removing the minimum subarray would remove the entire array and represent an empty selection. In that case, return the ordinary maximum instead.

### Stock scans as prefix decisions

For one transaction, the best sale at a day depends on the minimum price before that day:

```text
profitToday = priceToday - minimumPriceSoFar
```

For unlimited transactions, every positive day-to-day increase can be collected. These are different contracts and should not share an unexplained implementation.

### Prefix and suffix products

For Product of Array Except Self, each answer is:

```text
product before i * product after i
```

One forward pass writes prefix products into the output. One backward pass multiplies in a rolling suffix product. This handles zeros without division.

### Contract and numeric safety

These problems often accumulate sums, counts, profits, or products. TypeScript uses IEEE-754 numbers, so inputs being safe integers does not guarantee every intermediate result is safe.

A precise API should state:

- whether input values and query indexes are validated
- whether empty input is allowed
- whether a selected subarray must be non-empty
- which index wins when several answers are valid
- whether input is mutated
- what happens when an intermediate result leaves the safe-integer range

The solutions in this chapter do not mutate caller-owned arrays and throw `RangeError` when a required integer contract is violated.

## Visual Mental Model

### Prefix state is a ledger

```text
values:  [ 3, -1,  4, 2]
ledger:  [ 0,  3,  2, 6, 8]
           ^               leading zero

range [1..3] = ledger[4] - ledger[1] = 8 - 3 = 5
```

### A map remembers earlier boundaries

```text
current prefix = 7
target k       = 4
needed earlier = 3

prefix frequency map says 3 occurred twice
=> two target-sum subarrays end here
```

### Kadane extends or restarts

```text
previous current + value   versus   value alone
           extend          versus   restart
```

The larger choice is the best non-empty subarray ending at this index.

### Circular selection removes a middle

```text
[ kept suffix | removed minimum middle | kept prefix ]
  \________________ wraps around _________________/
```

### Product from both directions

```text
answer[i] = [product to the left] * [product to the right]
```

## Complexity Concepts

### Preprocessing versus query cost

A range-sum data structure spends `O(n)` once and answers each immutable query in `O(1)`. For `q` queries, total time is `O(n + q)`, compared with `O(nq)` when every range is rescanned.

### Why prefix-map counting is linear on average

Each prefix sum performs a constant number of `Map` lookups and updates. JavaScript maps provide average `O(1)` access, so the scan is average `O(n)` time and worst-case `O(n)` space.

### Why Kadane is dynamic programming

Kadane compresses a full recurrence into one previous state. It computes one state per index and keeps only `current` and `best`, giving `O(n)` time and `O(1)` auxiliary space.

### Output space versus auxiliary space

Running Sum and Product Except Self must return `n` values, so their output costs `O(n)`. If the only additional state is a few scalars, auxiliary space is `O(1)`. State both measurements rather than hiding the required output.

### Trace generation has a cost

An optimized algorithm may use `O(1)` auxiliary space while a full educational trace stores `O(n)` events. Production endpoints should cap or sample traces and report truncation.

## Common Patterns

### Build once, query many times

```text
build prefix[0..n]
for each query [left, right]:
    answer prefix[right + 1] - prefix[left]
```

### Total minus known parts

```text
rightSum = totalSum - leftSum - currentValue
```

Use this when one scan can maintain the left portion and a precomputed total supplies the rest.

### Prefix complement lookup

```text
currentPrefix += value
count += frequency[currentPrefix - target]
frequency[currentPrefix] += 1
```

Use this for counting arbitrary signed contiguous sums.

### Extend or restart

```text
current = max(value, current + value)
best = max(best, current)
```

Use this when the best contiguous result ending here depends only on the previous ending-here result.

### Forward state plus backward state

```text
forward pass:  record contribution before each index
backward pass: combine contribution after each index
```

Use this when every answer needs information from both sides.

# Problems

## Problem 71 — Running Sum of 1D Array

### 1. Problem Statement
Given an array of safe integers, return an array where position `i` contains the sum of input values from index `0` through `i`. Do not mutate the input. Throw `RangeError` if an input or accumulated sum is not a safe integer.

### 2. Example
```text
Input:  [1, 2, 3, 4]
Output: [1, 3, 6, 10]
```

### 3. Understand the Problem
The output has the same length as the input. Every output position represents an inclusive prefix; it is not the sum of a separate fixed-size window.

#### In Simple Terms
Read the numbers from left to right while keeping a total. After each number is added, place the new total into the answer. For `[1, 2, 3, 4]`, the totals are `1`, then `3`, then `6`, then `10`.

### 4. Real-World Analogy
A bank statement's running balance shows the total after each transaction instead of making you add every earlier transaction again.

### 5. Key Observation
The running sum at index `i` equals the previous running sum plus `values[i]`. Earlier values never need to be revisited.

### 6. Pattern
```text
Pattern: Inclusive Prefix Sum
Why this pattern applies: each answer summarizes values from index 0 through i
How to recognize it: the output asks for a cumulative result at every position
Similar problems: cumulative scores, prefix products, range-sum preprocessing
```

### 7. Brute Force
For every output index, start from zero and add all values from index `0` through that index.

#### Pseudocode
```text
answer = []
for end from 0 through n - 1:
    total = 0
    for index from 0 through end:
        validate and add values[index]
    append total to answer
return answer
```

#### Brute-Force TypeScript
```typescript
function runningSumBruteForce(values: number[]): number[] {
  const result: number[] = [];

  for (let end = 0; end < values.length; end += 1) {
    let total = 0;
    for (let index = 0; index <= end; index += 1) {
      if (!Number.isSafeInteger(values[index])) {
        throw new RangeError("values must contain only safe integers");
      }
      total += values[index];
      if (!Number.isSafeInteger(total)) {
        throw new RangeError("running sum exceeds the safe-integer range");
      }
    }
    result.push(total);
  }

  return result;
}
```

#### Visualization
```text
result[0]: 1                       = 1
result[1]: 1 + 2                   = 3
result[2]: 1 + 2 + 3               = 6
result[3]: 1 + 2 + 3 + 4           = 10
```

#### Dry Run
For `[1, 2, 3, 4]`, the nested loops perform `1 + 2 + 3 + 4 = 10` additions and return `[1, 3, 6, 10]`.

#### Brute-Force Complexity
**Time: `O(n²)`** because prefixes are repeatedly rescanned. **Space: `O(n)`** for the required output and `O(1)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
Adjacent prefixes overlap almost completely. Recomputing every prefix discards the total that was already calculated for the previous output position.

### 9. Optimal Approach
Keep one `runningTotal`. For each value, validate it, add it once, validate the new total, and append that total to a new result array.

The invariant after processing index `i` is:

```text
runningTotal = values[0] + values[1] + ... + values[i]
result[i] = runningTotal
```

### 10. Visualization
```text
value       runningTotal       result
  1       0 + 1 = 1           [1]
  2       1 + 2 = 3           [1, 3]
  3       3 + 3 = 6           [1, 3, 6]
  4       6 + 4 = 10          [1, 3, 6, 10]
```

### 11. Step-by-Step Dry Run
| Index | Value | Previous total | New total | Result |
|---:|---:|---:|---:|---|
| 0 | 1 | 0 | 1 | `[1]` |
| 1 | 2 | 1 | 3 | `[1, 3]` |
| 2 | 3 | 3 | 6 | `[1, 3, 6]` |
| 3 | 4 | 6 | 10 | `[1, 3, 6, 10]` |

### 12. TypeScript Solution
```typescript
function runningSum(values: number[]): number[] {
  const result: number[] = [];
  let runningTotal = 0;

  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
    runningTotal += value;
    if (!Number.isSafeInteger(runningTotal)) {
      throw new RangeError("running sum exceeds the safe-integer range");
    }
    result.push(runningTotal);
  }

  return result;
}
```

### 13. Code Walkthrough
`runningTotal` stores the inclusive prefix through the current value. `result.push` writes a new array, so caller-owned input is unchanged. Both input values and accumulated totals are checked because safe inputs can still overflow when added.

### 14. Complexity Analysis
**Time: `O(n)`** because each value is processed once. **Space: `O(n)`** for the required result and `O(1)` auxiliary space.

### 15. Edge Cases
- Empty input returns `[]`.
- One value returns a one-element copy.
- Negative values can make later running sums smaller.
- Zero leaves the running total unchanged.
- Unsafe input or an unsafe accumulated total throws `RangeError`.

### 16. Common Mistakes
- Recomputing each prefix from the beginning.
- Starting the output with an extra zero when the contract requires length `n`.
- Mutating the input without documenting it.
- Validating individual values but not accumulated sums.

### 17. Interview Follow-ups
1. Produce the result in place when mutation is allowed. 2. Stream running sums from an iterable. 3. Support exact values beyond the safe-integer range with `bigint`. 4. Reset the running total at specified boundaries.

### 18. Variations
- Running product. - Cumulative XOR. - Two-dimensional prefix sum. - Difference arrays for range updates.

### 19. Pattern Connections
This is the simplest prefix-state scan. Problem 72 adds a leading zero so subtraction can isolate arbitrary ranges, while Problem 75 combines prefix sums with hashing.

### 20. NestJS Project Connection
Expose `POST /api/dsa/prefix-sum/running` with `{ values }`. Validate an array of safe integers and return `{ result, trace, truncated }`, where each bounded optimized trace event contains `index`, `value`, `previousTotal`, and `runningTotal`.

### 21. Mini Challenge
Return running sums as `bigint[]` from a `bigint[]` input, and explain which parts of the numeric-safety contract change.

## Problem 72 — Range Sum Query

### 1. Problem Statement
Given an immutable array of safe integers and inclusive index pairs `[left, right]`, return the sum for every requested range. Indexes must be integers satisfying `0 <= left <= right < values.length`. Do not mutate either input, and throw `RangeError` for invalid indexes or unsafe arithmetic.

### 2. Example
```text
Input:  values = [-2, 0, 3, -5, 2, -1]
        queries = [[0, 2], [2, 5], [0, 5]]
Output: [1, -1, -3]
```

### 3. Understand the Problem
Each query asks for an inclusive contiguous range. The array does not change between queries, so preprocessing may be reused. Results must correspond to queries in their original order.

#### In Simple Terms
You receive one list of numbers and several requests such as “add everything from index 2 through index 5.” Return one answer per request. Because every request uses the same unchanged list, prepare useful totals once instead of adding each range from scratch.

### 4. Real-World Analogy
A utility dashboard stores cumulative daily usage, then subtracts two readings to report consumption between any two dates.

### 5. Key Observation
A leading-zero prefix array stores the sum before every input boundary. For inclusive `[left, right]`, subtracting `prefix[left]` from `prefix[right + 1]` removes everything before the requested range.

### 6. Pattern
```text
Pattern: Prefix Sum Preprocessing
Why this pattern applies: many queries reuse one immutable array
How to recognize it: repeated contiguous range sums with no updates
Similar problems: immutable 2D range sums, cumulative counts, range averages
```

### 7. Brute Force
Validate the array once. For each query, validate its boundaries and loop from `left` through `right` to calculate that answer.

#### Pseudocode
```text
validate every value
answers = []
for each [left, right]:
    validate left and right
    total = 0
    for index from left through right:
        total += values[index]
        validate total
    append total
return answers
```

#### Brute-Force TypeScript
```typescript
type RangeQuery = readonly [left: number, right: number];

function rangeSumsBruteForce(
  values: number[],
  queries: RangeQuery[],
): number[] {
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
  }

  const answers: number[] = [];
  for (const [left, right] of queries) {
    if (!Number.isInteger(left) || !Number.isInteger(right) ||
        left < 0 || left > right || right >= values.length) {
      throw new RangeError("each query must be a valid inclusive range");
    }

    let total = 0;
    for (let index = left; index <= right; index += 1) {
      total += values[index];
      if (!Number.isSafeInteger(total)) {
        throw new RangeError("range sum exceeds the safe-integer range");
      }
    }
    answers.push(total);
  }

  return answers;
}
```

#### Visualization
```text
query [2, 5] on [-2, 0, 3, -5, 2, -1]
                    3 + -5 + 2 + -1 = -1
                    ^---------------^
```

#### Dry Run
The three example queries scan `3 + 4 + 6 = 13` values and produce `[1, -1, -3]`.

#### Brute-Force Complexity
For total queried length `L`, **time is `O(n + L)`**, which is `O(nq)` in the worst case for `q` queries. **Space is `O(q)`** for answers and `O(1)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
Large or overlapping ranges repeatedly add the same values. With many queries, the repeated work dominates the one-time cost of reading the array.

### 9. Optimal Approach
Build `prefix` with length `n + 1`, where `prefix[i + 1]` is the sum through `values[i]`. Then answer every valid inclusive query with:

```text
prefix[right + 1] - prefix[left]
```

Validate prefix additions and query-result subtraction because either may leave the safe-integer range.

### 10. Visualization
```text
values: [-2,  0,  3, -5,  2, -1]
prefix: [ 0, -2, -2,  1, -4, -2, -3]
index:     0   1   2   3   4   5   6

query [2, 5] = prefix[6] - prefix[2]
             = -3 - (-2)
             = -1
```

### 11. Step-by-Step Dry Run
| Query | Prefix expression | Answer |
|---|---|---:|
| `[0, 2]` | `prefix[3] - prefix[0] = 1 - 0` | 1 |
| `[2, 5]` | `prefix[6] - prefix[2] = -3 - (-2)` | -1 |
| `[0, 5]` | `prefix[6] - prefix[0] = -3 - 0` | -3 |

### 12. TypeScript Solution
```typescript
type RangeQuery = readonly [left: number, right: number];

function rangeSums(values: number[], queries: RangeQuery[]): number[] {
  const prefix = Array<number>(values.length + 1).fill(0);

  for (let index = 0; index < values.length; index += 1) {
    if (!Number.isSafeInteger(values[index])) {
      throw new RangeError("values must contain only safe integers");
    }
    prefix[index + 1] = prefix[index] + values[index];
    if (!Number.isSafeInteger(prefix[index + 1])) {
      throw new RangeError("prefix sum exceeds the safe-integer range");
    }
  }

  return queries.map(([left, right]) => {
    if (!Number.isInteger(left) || !Number.isInteger(right) ||
        left < 0 || left > right || right >= values.length) {
      throw new RangeError("each query must be a valid inclusive range");
    }
    const sum = prefix[right + 1] - prefix[left];
    if (!Number.isSafeInteger(sum)) {
      throw new RangeError("range sum exceeds the safe-integer range");
    }
    return sum;
  });
}
```

### 13. Code Walkthrough
`prefix[0]` represents the empty prefix. The build loop stores one additional cumulative boundary per value. Each query maps its inclusive indexes to half-open prefix boundaries and performs one subtraction.

### 14. Complexity Analysis
For `n` values and `q` queries, **time is `O(n + q)`**. **Space is `O(n + q)`** including the prefix array and returned answers; auxiliary space excluding output is `O(n)`.

### 15. Edge Cases
- No queries returns `[]` after validating the values.
- Empty values permits no non-empty valid query.
- A one-index range returns that value.
- Negative values require no special algorithm change.
- Reversed, fractional, negative, or out-of-bounds indexes throw.

### 16. Common Mistakes
- Using `prefix[right] - prefix[left]` for an inclusive range.
- Forgetting the leading zero.
- Rebuilding the prefix array for every query.
- Applying this immutable structure when updates must be supported.

### 17. Interview Follow-ups
1. Support point updates with a Fenwick tree. 2. Extend to rectangular matrix queries. 3. Return range averages. 4. Build a reusable query object once and call it many times.

### 18. Variations
- Range XOR query. - Prefix frequency query. - Difference arrays. - Two-dimensional summed-area tables.

### 19. Pattern Connections
Problem 71 stores inclusive prefix results directly. This problem adds an empty-prefix boundary so one subtraction can remove an unwanted earlier prefix.

### 20. NestJS Project Connection
Expose `POST /api/dsa/prefix-sum/range-sums` with `{ values, queries }`. Return `{ answers, trace, truncated }`; optimized trace events should show bounded prefix-build steps followed by each query's two prefix indexes and subtraction.

### 21. Mini Challenge
Design a `RangeSumQuery` class that preprocesses in its constructor and answers one query method call in `O(1)`.

## Problem 73 — Find equilibrium index

### 1. Problem Statement
Given an array of safe integers, return every index where the sum of values strictly to its left equals the sum strictly to its right. Return indexes in ascending order, do not mutate the input, and throw `RangeError` for unsafe arithmetic.

### 2. Example
```text
Input:  [-7, 1, 5, 2, -4, 3, 0]
Output: [3, 6]
```

### 3. Understand the Problem
The value at a candidate index belongs to neither side. An empty side has sum zero, so the first or last index can be an equilibrium index. This chapter uses “equilibrium index” to return all balance points; Problem 74 uses the common “pivot index” contract and returns only the leftmost one.

#### In Simple Terms
Imagine balancing the array on one position. Add everything before that position and everything after it, but ignore the value under the balance point. Return every position where those two totals match.

### 4. Real-World Analogy
A seesaw balances at every support position where the total weight on its left equals the total weight on its right, excluding the support itself.

### 5. Key Observation
Once the total array sum and the current left sum are known, the right sum at index `i` is `totalSum - leftSum - values[i]`.

### 6. Pattern
```text
Pattern: Total Sum Minus Known Parts
Why this pattern applies: total = left side + current value + right side
How to recognize it: compare aggregate values on both sides of each index
Similar problems: pivot index, equal prefix/suffix partition, balance point
```

### 7. Brute Force
For every candidate index, independently sum all values before it and all values after it. Record the index when the totals match.

#### Pseudocode
```text
validate every value
answers = []
for pivot from 0 through n - 1:
    leftSum = sum values before pivot
    rightSum = sum values after pivot
    if leftSum equals rightSum:
        append pivot
return answers
```

#### Brute-Force TypeScript
```typescript
function equilibriumIndexesBruteForce(values: number[]): number[] {
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
  }

  const indexes: number[] = [];
  for (let pivot = 0; pivot < values.length; pivot += 1) {
    let leftSum = 0;
    let rightSum = 0;

    for (let index = 0; index < pivot; index += 1) {
      leftSum += values[index];
      if (!Number.isSafeInteger(leftSum)) {
        throw new RangeError("side sum exceeds the safe-integer range");
      }
    }
    for (let index = pivot + 1; index < values.length; index += 1) {
      rightSum += values[index];
      if (!Number.isSafeInteger(rightSum)) {
        throw new RangeError("side sum exceeds the safe-integer range");
      }
    }
    if (leftSum === rightSum) indexes.push(pivot);
  }

  return indexes;
}
```

#### Visualization
```text
pivot 3: [-7, 1, 5] | 2 | [-4, 3, 0]
left sum  = -1
right sum = -1
```

#### Dry Run
On the example, rescanning both sides identifies indexes `3` and `6`. At index `6`, the left sum is `0` and the empty right side also sums to `0`.

#### Brute-Force Complexity
**Time: `O(n²)`** because almost the entire array is summed for every candidate. **Space: `O(r)`** for `r` returned indexes and `O(1)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
The left side at one candidate differs from the next by one value, yet brute force throws away that work. The right side can also be derived from a known total.

### 9. Optimal Approach
First calculate `totalSum`. Scan again with `leftSum = 0`. At each index, derive `rightSum`, record a match, and only then add the current value to `leftSum`.

Checking before the update preserves the strict-side invariant:

```text
leftSum = sum of values at indexes less than current index
```

### 10. Visualization
```text
values:   [-7, 1, 5, 2, -4, 3, 0]   total = 0

index 3:
leftSum  = -1
rightSum = 0 - (-1) - 2 = -1   -> record 3

index 6:
leftSum  = 0
rightSum = 0 - 0 - 0 = 0       -> record 6
```

### 11. Step-by-Step Dry Run
| Index | Value | Left before | Right | Match |
|---:|---:|---:|---:|---|
| 0 | -7 | 0 | 7 | no |
| 1 | 1 | -7 | 6 | no |
| 2 | 5 | -6 | 1 | no |
| 3 | 2 | -1 | -1 | yes |
| 4 | -4 | 1 | 3 | no |
| 5 | 3 | -3 | 0 | no |
| 6 | 0 | 0 | 0 | yes |

### 12. TypeScript Solution
```typescript
function findEquilibriumIndexes(values: number[]): number[] {
  let totalSum = 0;
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
    totalSum += value;
    if (!Number.isSafeInteger(totalSum)) {
      throw new RangeError("total sum exceeds the safe-integer range");
    }
  }

  const indexes: number[] = [];
  let leftSum = 0;
  for (let index = 0; index < values.length; index += 1) {
    const rightSum = totalSum - leftSum - values[index];
    if (!Number.isSafeInteger(rightSum)) {
      throw new RangeError("side sum exceeds the safe-integer range");
    }
    if (leftSum === rightSum) indexes.push(index);

    leftSum += values[index];
    if (!Number.isSafeInteger(leftSum)) {
      throw new RangeError("side sum exceeds the safe-integer range");
    }
  }

  return indexes;
}
```

### 13. Code Walkthrough
The first pass validates values while calculating the whole-array total. During the second pass, `leftSum` excludes the current value until after comparison. Appending matches preserves ascending index order.

### 14. Complexity Analysis
**Time: `O(n)`** for two linear passes. **Space: `O(r)`** for `r` returned indexes and `O(1)` auxiliary space.

### 15. Edge Cases
- Empty input returns `[]`.
- A one-element array returns `[0]` because both sides are empty.
- The first or last index may qualify.
- Negative values and multiple equilibrium indexes are supported.
- Unsafe values or arithmetic throw `RangeError`.

### 16. Common Mistakes
- Including the current value in one side.
- Updating `leftSum` before checking the current index.
- Assuming only one equilibrium index can exist.
- Treating an empty side as invalid rather than sum zero.

### 17. Interview Follow-ups
1. Return only the leftmost index. 2. Return only the count. 3. Accept `bigint` values. 4. Find a balance boundary between elements instead of an index.

### 18. Variations
- Pivot index. - Equal left/right product. - Three-way equal-sum partition. - Weighted balance point.

### 19. Pattern Connections
Like Problem 72, this problem avoids rebuilding range totals. It needs only the whole total and a moving left prefix, so no prefix array is required unless queries must be revisited.

### 20. NestJS Project Connection
Expose `POST /api/dsa/prefix-sum/equilibrium-indexes` with `{ values }`. Return all indexes plus bounded optimized events containing `index`, `leftSum`, `value`, `rightSum`, and `isEquilibrium`.

### 21. Mini Challenge
Change the contract to return balance boundaries `b` where `sum(values[0..b))` equals `sum(values[b..n))`, and handle boundaries `0` and `n`.

## Problem 74 — Pivot Index

### 1. Problem Statement
Given an array of safe integers, return the leftmost index where the sum strictly to its left equals the sum strictly to its right. Return `-1` if no pivot exists. Do not mutate the input, and throw `RangeError` for unsafe arithmetic.

### 2. Example
```text
Input:  [1, 7, 3, 6, 5, 6]
Output: 3
```

### 3. Understand the Problem
The current value is excluded from both sides. Empty sides count as sum zero. If several indexes qualify, return the smallest index rather than every match.

#### In Simple Terms
Move from left to right looking for the first position that balances the numbers on either side. Ignore the number at that position. Return its index, or `-1` when no position balances.

### 4. Real-World Analogy
You test possible support points from left to right and stop at the first place where the load on both sides is equal.

### 5. Key Observation
The right sum does not need a separate scan. At each index it is `totalSum - leftSum - currentValue`. A left-to-right scan naturally enforces the leftmost tie rule.

### 6. Pattern
```text
Pattern: Total Sum with Running Prefix
Why this pattern applies: the right side is the total minus the known left and current parts
How to recognize it: find the first index balancing aggregates on both sides
Similar problems: equilibrium indexes, center of mass, equal-sum partition
```

### 7. Brute Force
At each index, independently sum the values to the left and right. Return immediately on the first equality.

#### Pseudocode
```text
validate every value
for pivot from 0 through n - 1:
    leftSum = sum before pivot
    rightSum = sum after pivot
    if leftSum equals rightSum:
        return pivot
return -1
```

#### Brute-Force TypeScript
```typescript
function pivotIndexBruteForce(values: number[]): number {
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
  }

  for (let pivot = 0; pivot < values.length; pivot += 1) {
    let leftSum = 0;
    let rightSum = 0;

    for (let index = 0; index < pivot; index += 1) {
      leftSum += values[index];
      if (!Number.isSafeInteger(leftSum)) {
        throw new RangeError("side sum exceeds the safe-integer range");
      }
    }
    for (let index = pivot + 1; index < values.length; index += 1) {
      rightSum += values[index];
      if (!Number.isSafeInteger(rightSum)) {
        throw new RangeError("side sum exceeds the safe-integer range");
      }
    }
    if (leftSum === rightSum) return pivot;
  }

  return -1;
}
```

#### Visualization
```text
index 0: [] | 1 | [7, 3, 6, 5, 6]   0 != 27
index 1: [1] | 7 | [3, 6, 5, 6]     1 != 20
...
index 3: [1, 7, 3] | 6 | [5, 6]    11 == 11 -> return 3
```

#### Dry Run
For `[1, 7, 3, 6, 5, 6]`, indexes `0`, `1`, and `2` fail. Index `3` has side sums `11` and `11`, so the search stops.

#### Brute-Force Complexity
**Time: `O(n²)`** in the worst case when no early match exists. **Space: `O(1)`** auxiliary space.

### 8. Why Brute Force Is Not Ideal
It recalculates nearly identical side sums at every candidate. The total and a running left sum contain all information needed for the comparison.

### 9. Optimal Approach
Calculate the total in one pass. In a second left-to-right pass, derive the current right sum and return as soon as it equals `leftSum`. If it does not match, add the current value to `leftSum` and continue.

### 10. Visualization
```text
total = 28

i  value  left  right = total - left - value
0    1      0     27
1    7      1     20
2    3      8     17
3    6     11     11  -> return 3
```

### 11. Step-by-Step Dry Run
| Index | Value | Left before | Right | Decision |
|---:|---:|---:|---:|---|
| 0 | 1 | 0 | 27 | continue |
| 1 | 7 | 1 | 20 | continue |
| 2 | 3 | 8 | 17 | continue |
| 3 | 6 | 11 | 11 | return `3` |

### 12. TypeScript Solution
```typescript
function findPivotIndex(values: number[]): number {
  let totalSum = 0;
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
    totalSum += value;
    if (!Number.isSafeInteger(totalSum)) {
      throw new RangeError("total sum exceeds the safe-integer range");
    }
  }

  let leftSum = 0;
  for (let index = 0; index < values.length; index += 1) {
    const rightSum = totalSum - leftSum - values[index];
    if (!Number.isSafeInteger(rightSum)) {
      throw new RangeError("side sum exceeds the safe-integer range");
    }
    if (leftSum === rightSum) return index;

    leftSum += values[index];
    if (!Number.isSafeInteger(leftSum)) {
      throw new RangeError("side sum exceeds the safe-integer range");
    }
  }

  return -1;
}
```

### 13. Code Walkthrough
The total pass also validates every value. The search checks indexes in ascending order, so the first returned match satisfies the tie rule. The current value joins `leftSum` only after its pivot test.

### 14. Complexity Analysis
**Time: `O(n)`** for two passes. **Space: `O(1)`** auxiliary space.

### 15. Edge Cases
- Empty input returns `-1`.
- A one-element array returns `0`.
- A pivot may occur at either endpoint.
- Multiple pivots return only the leftmost.
- Negative values are fully supported.

### 16. Common Mistakes
- Returning every pivot when the contract asks for the leftmost.
- Adding the current value to `leftSum` before testing it.
- Rejecting endpoint pivots because one side is empty.
- Using `reduce` without checking intermediate numeric safety.

### 17. Interview Follow-ups
1. Return all pivots. 2. Return the rightmost pivot. 3. Process `bigint` input. 4. Support changing values between pivot queries.

### 18. Variations
- Equilibrium indexes. - Weighted pivot. - Equal-sum split boundary. - Minimum left/right difference.

### 19. Pattern Connections
This uses the same balance equation as Problem 73 but changes the output contract. Returning immediately removes output storage and makes the leftmost rule explicit.

### 20. NestJS Project Connection
Expose `POST /api/dsa/prefix-sum/pivot-index` with `{ values }`. Return `{ pivotIndex, trace, truncated }`; each bounded optimized event should include the candidate index, left sum, derived right sum, and whether the endpoint returns.

### 21. Mini Challenge
Return the index minimizing `abs(leftSum - rightSum)`, breaking ties toward the leftmost index.

## Problem 75 — Subarray Sum Equals K

### 1. Problem Statement
Given an array of safe integers and a safe-integer target `k`, return the number of non-empty contiguous subarrays whose sum equals `k`. Values may be positive, negative, or zero. Do not mutate the input, and throw `RangeError` if arithmetic or the result count becomes unsafe.

### 2. Example
```text
Input:  values = [1, 1, 1], k = 2
Output: 2
```

### 3. Understand the Problem
Count index ranges, not distinct value sequences. Overlapping ranges count separately. The presence of negative values means a sum can rise or fall as either boundary moves.

#### In Simple Terms
Look at every unbroken slice of the array and count how many add exactly to `k`. Two slices still count separately when they overlap or contain the same numbers at different positions.

### 4. Real-World Analogy
Given daily balance changes, count every continuous date interval whose total change equals a requested amount.

### 5. Key Observation
If the current prefix sum is `P`, a subarray ending here has sum `k` for every earlier boundary whose prefix was `P - k`. A frequency map counts all such boundaries, including repeated prefix sums.

### 6. Pattern
```text
Pattern: Prefix Sum + Frequency Map
Why this pattern applies: target subarrays are differences between two prefix sums
How to recognize it: count contiguous sums with negative values or zeros allowed
Similar problems: longest target-sum subarray, zero-sum subarrays, divisible-sum counts
```

### 7. Brute Force
Choose every start index, grow the subarray one value at a time, and count whenever its running sum equals `k`.

#### Pseudocode
```text
validate target and values
count = 0
for start from 0 through n - 1:
    sum = 0
    for end from start through n - 1:
        sum += values[end]
        if sum equals target:
            count += 1
return count
```

#### Brute-Force TypeScript
```typescript
function countSubarraysWithSumBruteForce(
  values: number[],
  target: number,
): number {
  if (!Number.isSafeInteger(target)) {
    throw new RangeError("target must be a safe integer");
  }
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
  }

  let count = 0;
  for (let start = 0; start < values.length; start += 1) {
    let sum = 0;
    for (let end = start; end < values.length; end += 1) {
      sum += values[end];
      if (!Number.isSafeInteger(sum)) {
        throw new RangeError("subarray sum exceeds the safe-integer range");
      }
      if (sum === target) {
        count += 1;
        if (!Number.isSafeInteger(count)) {
          throw new RangeError("result count exceeds the safe-integer range");
        }
      }
    }
  }
  return count;
}
```

#### Visualization
```text
[1, 1],1  -> sum 2, count 1
 1,[1, 1] -> sum 2, count 2
```

#### Dry Run
For `[1, 1, 1]`, ranges `[0, 1]` and `[1, 2]` sum to `2`. All one-element and three-element ranges do not.

#### Brute-Force Complexity
**Time: `O(n²)`** because there are `O(n²)` start/end pairs. **Space: `O(1)`** auxiliary space.

### 8. Why Brute Force Is Not Ideal
Many candidate ranges share the same prefixes. Repeating their additions ignores the algebraic relation between current and earlier prefix sums.

### 9. Optimal Approach
Store how often each prefix sum has occurred. Seed the map with prefix `0` occurring once, representing the boundary before index `0`.

For each value:

1. update `currentPrefix`
2. calculate `neededPrefix = currentPrefix - target`
3. add the frequency of `neededPrefix` to the answer
4. record the current prefix for future subarrays

The lookup must happen before recording the current prefix so only non-empty subarrays are counted, especially when `target` is zero.

### 10. Visualization
```text
values = [1, 1, 1], target = 2
initial frequencies: {0: 1}

prefix 1 needs -1 -> +0; record 1
prefix 2 needs  0 -> +1; record 2
prefix 3 needs  1 -> +1; record 3
count = 2
```

### 11. Step-by-Step Dry Run
| Index | Value | Prefix | Needed | Prior frequency | Count |
|---:|---:|---:|---:|---:|---:|
| start | — | 0 | — | store `0 -> 1` | 0 |
| 0 | 1 | 1 | -1 | 0 | 0 |
| 1 | 1 | 2 | 0 | 1 | 1 |
| 2 | 1 | 3 | 1 | 1 | 2 |

### 12. TypeScript Solution
```typescript
function countSubarraysWithSum(values: number[], target: number): number {
  if (!Number.isSafeInteger(target)) {
    throw new RangeError("target must be a safe integer");
  }

  const prefixFrequencies = new Map<number, number>([[0, 1]]);
  let currentPrefix = 0;
  let count = 0;

  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
    currentPrefix += value;
    if (!Number.isSafeInteger(currentPrefix)) {
      throw new RangeError("prefix sum exceeds the safe-integer range");
    }

    const neededPrefix = currentPrefix - target;
    if (!Number.isSafeInteger(neededPrefix)) {
      throw new RangeError("prefix difference exceeds the safe-integer range");
    }
    count += prefixFrequencies.get(neededPrefix) ?? 0;
    if (!Number.isSafeInteger(count)) {
      throw new RangeError("result count exceeds the safe-integer range");
    }

    const nextFrequency = (prefixFrequencies.get(currentPrefix) ?? 0) + 1;
    if (!Number.isSafeInteger(nextFrequency)) {
      throw new RangeError("prefix frequency exceeds the safe-integer range");
    }
    prefixFrequencies.set(currentPrefix, nextFrequency);
  }

  return count;
}
```

### 13. Code Walkthrough
The map stores prefix boundaries, not array values. Every previous occurrence of `currentPrefix - target` defines a different valid start boundary. Recording the current prefix last prevents pairing a boundary with itself.

### 14. Complexity Analysis
**Average time: `O(n)`** with average constant-time map operations. **Space: `O(n)`** in the worst case for distinct prefix sums.

### 15. Edge Cases
- Empty input returns `0` after validating `target`.
- `target = 0` can match repeated prefix sums.
- Negative values are supported.
- Overlapping and duplicate-valued ranges count separately.
- Unsafe prefixes, differences, frequencies, or counts throw.

### 16. Common Mistakes
- Using a positive-only sliding window when negatives are allowed.
- Storing only whether a prefix exists instead of its frequency.
- Forgetting the initial `0 -> 1` entry.
- Recording the current prefix before counting when `target` is zero.

### 17. Interview Follow-ups
1. Return the longest matching range. 2. Return all matching index pairs with output-sensitive complexity. 3. Count sums divisible by `k`. 4. Process a stream incrementally.

### 18. Variations
- Zero-sum subarray count. - Binary Subarrays With Sum. - Subarray Sums Divisible by K. - Longest target-sum subarray.

### 19. Pattern Connections
This combines Problem 71's cumulative state with hashing from the Hashing chapter. It also explains why Problem 70's shrinking window cannot simply be reused for arbitrary signed values.

### 20. NestJS Project Connection
Expose `POST /api/dsa/prefix-sum/subarray-sum-count` with `{ values, target }`. Return `{ count, trace, truncated }`; each bounded optimized event should show `index`, `currentPrefix`, `neededPrefix`, `matchesAdded`, and the updated frequency.

### 21. Mini Challenge
Return the longest target-sum subarray as `[start, end]`, breaking equal-length ties by the smaller start index.

## Problem 76 — Maximum Subarray

### 1. Problem Statement
Given a non-empty array of safe integers, return the largest sum among all non-empty contiguous subarrays. Do not mutate the input. Throw `RangeError` for empty input, unsafe values, or unsafe intermediate sums.

### 2. Example
```text
Input:  [-2, 1, -3, 4, -1, 2, 1, -5, 4]
Output: 6
Explanation: [4, -1, 2, 1] has the largest sum.
```

### 3. Understand the Problem
The chosen values must be adjacent and at least one value must be selected. Only the maximum sum is returned, so index tie-breaking is outside this contract.

#### In Simple Terms
Choose one unbroken section of the array whose numbers add to the greatest possible total. You cannot skip a number inside the section, and you cannot choose nothing.

### 4. Real-World Analogy
Find the most profitable uninterrupted period in a sequence of daily gains and losses. Once the period starts, every day through its end is included.

### 5. Key Observation
At each index, the best non-empty subarray ending there either starts fresh with the current value or extends the best subarray ending at the previous index.

### 6. Pattern
```text
Pattern: Kadane's Algorithm / Compressed Dynamic Programming
Why this pattern applies: the best range ending here has only extend-or-restart choices
How to recognize it: optimize a non-empty contiguous sum over all boundaries
Similar problems: minimum subarray, circular maximum, maximum product subarray
```

### 7. Brute Force
Choose every start index, extend an end index to the right, and maintain that candidate's running sum. Update the best sum after every extension.

#### Pseudocode
```text
reject empty input and validate values
best = first value
for start from 0 through n - 1:
    sum = 0
    for end from start through n - 1:
        sum += values[end]
        best = max(best, sum)
return best
```

#### Brute-Force TypeScript
```typescript
function maximumSubarraySumBruteForce(values: number[]): number {
  if (values.length === 0) {
    throw new RangeError("values must be non-empty");
  }
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
  }

  let bestSum = values[0];
  for (let start = 0; start < values.length; start += 1) {
    let sum = 0;
    for (let end = start; end < values.length; end += 1) {
      sum += values[end];
      if (!Number.isSafeInteger(sum)) {
        throw new RangeError("subarray sum exceeds the safe-integer range");
      }
      bestSum = Math.max(bestSum, sum);
    }
  }
  return bestSum;
}
```

#### Visualization
```text
start at index 3:
[4]             sum 4, best 4
[4, -1]         sum 3, best 4
[4, -1, 2]      sum 5, best 5
[4, -1, 2, 1]   sum 6, best 6
```

#### Dry Run
The baseline grows every range. Among the example's 45 ranges, `[4, -1, 2, 1]` reaches the maximum sum `6`.

#### Brute-Force Complexity
**Time: `O(n²)`** for all start/end pairs. **Space: `O(1)`** auxiliary space.

### 8. Why Brute Force Is Not Ideal
It evaluates every possible boundary pair even though only one fact from the previous index matters: the best sum of a subarray ending there.

### 9. Optimal Approach
Initialize `currentSum` and `bestSum` from the first value. For each later value, calculate the extension safely, then choose the larger of the value alone and that extension. Update the global best.

Invariant after index `i`:

```text
currentSum = maximum sum of a non-empty subarray ending exactly at i
bestSum    = maximum sum of a non-empty subarray within values[0..i]
```

### 10. Visualization
```text
value -2: current -2, best -2
value  1: max(1, -2+1) = 1, best 1       restart
value -3: max(-3, 1-3) = -2, best 1      extend
value  4: max(4, -2+4) = 4, best 4       restart
value -1: current 3, best 4               extend
value  2: current 5, best 5               extend
value  1: current 6, best 6               extend
```

### 11. Step-by-Step Dry Run
| Index | Value | Extension | Current | Best |
|---:|---:|---:|---:|---:|
| 0 | -2 | — | -2 | -2 |
| 1 | 1 | -1 | 1 | 1 |
| 2 | -3 | -2 | -2 | 1 |
| 3 | 4 | 2 | 4 | 4 |
| 4 | -1 | 3 | 3 | 4 |
| 5 | 2 | 5 | 5 | 5 |
| 6 | 1 | 6 | 6 | 6 |
| 7 | -5 | 1 | 1 | 6 |
| 8 | 4 | 5 | 5 | 6 |

### 12. TypeScript Solution
```typescript
function maximumSubarraySum(values: number[]): number {
  if (values.length === 0) {
    throw new RangeError("values must be non-empty");
  }
  if (!Number.isSafeInteger(values[0])) {
    throw new RangeError("values must contain only safe integers");
  }

  let currentSum = values[0];
  let bestSum = values[0];

  for (let index = 1; index < values.length; index += 1) {
    const value = values[index];
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
    const extendedSum = currentSum + value;
    if (!Number.isSafeInteger(extendedSum)) {
      throw new RangeError("subarray sum exceeds the safe-integer range");
    }
    currentSum = Math.max(value, extendedSum);
    bestSum = Math.max(bestSum, currentSum);
  }

  return bestSum;
}
```

### 13. Code Walkthrough
First-value initialization ensures a real element is always selected. `currentSum` forgets a harmful prefix by restarting at `value`; `bestSum` remembers the best ending position encountered anywhere.

### 14. Complexity Analysis
**Time: `O(n)`** because each value is processed once. **Space: `O(1)`** auxiliary space.

### 15. Edge Cases
- Empty input throws because no non-empty answer exists.
- A one-element array returns its value.
- An all-negative array returns its largest element.
- Zeros can form the best sum when every other value is negative.
- Unsafe extension arithmetic throws.

### 16. Common Mistakes
- Initializing the best sum to zero and accidentally allowing an empty subarray.
- Resetting every negative `currentSum` without preserving the all-negative case.
- Confusing a contiguous subarray with an arbitrary subsequence.
- Updating only the global best and losing the ending-at-current state.

### 17. Interview Follow-ups
1. Return start and end indexes. 2. Apply a deterministic tie rule. 3. Find the minimum subarray. 4. Allow one deletion inside the selected range.

### 18. Variations
- Maximum circular subarray. - Maximum product subarray. - Maximum sum with one deletion. - Two-dimensional maximum-sum rectangle.

### 19. Pattern Connections
Kadane's algorithm is a prefix decision rather than a prefix-sum lookup: it keeps the one earlier state that can improve a range ending at the current index.

### 20. NestJS Project Connection
Expose `POST /api/dsa/kadane/maximum-subarray` with `{ values }`. Return `{ maximumSum, trace, truncated }`; each bounded optimized event should show `index`, `value`, `extendedSum`, `decision`, `currentSum`, and `bestSum`.

### 21. Mini Challenge
Return `{ sum, start, end }`, choosing the shortest subarray when sums tie and then the smallest start index.

## Problem 77 — Maximum Circular Subarray

### 1. Problem Statement
Given a non-empty circular array of safe integers, return the largest sum of a non-empty contiguous subarray that uses each index at most once. A range may wrap from the final index to index `0`. Do not mutate the input, and throw `RangeError` for unsafe arithmetic.

### 2. Example
```text
Input:  [5, -3, 5]
Output: 10
Explanation: choose [5] at index 2, wrap, then [5] at index 0.
```

### 3. Understand the Problem
The array's ends are adjacent. A selected range may be ordinary or wrapping, but it cannot contain more than `n` values and cannot be empty.

#### In Simple Terms
Pretend the array is arranged in a circle. Choose one continuous arc with the largest total. You may cross from the last number back to the first, but you may not go around twice or choose no numbers.

### 4. Real-World Analogy
Choose the best continuous operating period on a weekly schedule where Sunday is followed by Monday. The period may cross the week boundary once.

### 5. Key Observation
A wrapping maximum keeps a suffix and a prefix, which is equivalent to taking the total array sum and removing one contiguous middle subarray with minimum sum.

### 6. Pattern
```text
Pattern: Dual Kadane (Maximum and Minimum Subarray)
Why this pattern applies: a wrapping range is the complement of one middle range
How to recognize it: contiguous selection on a circular array with at most one wrap
Similar problems: circular minimum subarray, maximum circular average, ring scheduling
```

### 7. Brute Force
Treat every index as a possible circular start. Extend lengths from `1` through `n`, use modulo to wrap, and keep the largest sum.

#### Pseudocode
```text
validate non-empty values
best = first value
for start from 0 through n - 1:
    sum = 0
    for length from 1 through n:
        index = (start + length - 1) modulo n
        sum += values[index]
        best = max(best, sum)
return best
```

#### Brute-Force TypeScript
```typescript
function maximumCircularSubarraySumBruteForce(values: number[]): number {
  if (values.length === 0) {
    throw new RangeError("values must be non-empty");
  }
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
  }

  let bestSum = values[0];
  for (let start = 0; start < values.length; start += 1) {
    let sum = 0;
    for (let length = 1; length <= values.length; length += 1) {
      const index = (start + length - 1) % values.length;
      sum += values[index];
      if (!Number.isSafeInteger(sum)) {
        throw new RangeError("circular sum exceeds the safe-integer range");
      }
      bestSum = Math.max(bestSum, sum);
    }
  }
  return bestSum;
}
```

#### Visualization
```text
start 2, length 2 on [5, -3, 5]
                     ^        take index 2: 5
                      \__wrap take index 0: 5
sum = 10
```

#### Dry Run
For `[5, -3, 5]`, ordinary ranges reach at most `7`, while the length-two range starting at index `2` wraps and reaches `10`.

#### Brute-Force Complexity
**Time: `O(n²)`** for `n` starts and `n` lengths. **Space: `O(1)`** auxiliary space.

### 8. Why Brute Force Is Not Ideal
It evaluates every circular boundary pair. The best non-wrapping range and the worst removable middle range can both be found in one scan.

### 9. Optimal Approach
Run maximum and minimum Kadane recurrences together while accumulating `totalSum`:

```text
maximumNormal = best ordinary subarray
minimumNormal = worst ordinary subarray
maximumWrap   = totalSum - minimumNormal
```

Return `maximumNormal` when it is negative, because then every value is negative and subtracting the whole-array minimum would create an illegal empty range. Otherwise return the larger of normal and wrapping sums.

### 10. Visualization
```text
[5, -3, 5]
total          = 7
maximum normal = 7       [5, -3, 5]
minimum normal = -3      [-3]
maximum wrap   = 7 - (-3) = 10

keep [5] suffix + [5] prefix
```

### 11. Step-by-Step Dry Run
| Index | Value | Max ending | Max best | Min ending | Min best | Total |
|---:|---:|---:|---:|---:|---:|---:|
| 0 | 5 | 5 | 5 | 5 | 5 | 5 |
| 1 | -3 | 2 | 5 | -3 | -3 | 2 |
| 2 | 5 | 7 | 7 | 2 | -3 | 7 |

`maximumWrap = 7 - (-3) = 10`, which beats `maximumNormal = 7`.

### 12. TypeScript Solution
```typescript
function maximumCircularSubarraySum(values: number[]): number {
  if (values.length === 0) {
    throw new RangeError("values must be non-empty");
  }
  if (!Number.isSafeInteger(values[0])) {
    throw new RangeError("values must contain only safe integers");
  }

  let totalSum = values[0];
  let maximumEnding = values[0];
  let maximumNormal = values[0];
  let minimumEnding = values[0];
  let minimumNormal = values[0];

  for (let index = 1; index < values.length; index += 1) {
    const value = values[index];
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }

    const maximumExtended = maximumEnding + value;
    const minimumExtended = minimumEnding + value;
    totalSum += value;
    if (!Number.isSafeInteger(maximumExtended) ||
        !Number.isSafeInteger(minimumExtended) ||
        !Number.isSafeInteger(totalSum)) {
      throw new RangeError("subarray sum exceeds the safe-integer range");
    }

    maximumEnding = Math.max(value, maximumExtended);
    maximumNormal = Math.max(maximumNormal, maximumEnding);
    minimumEnding = Math.min(value, minimumExtended);
    minimumNormal = Math.min(minimumNormal, minimumEnding);
  }

  if (maximumNormal < 0) return maximumNormal;

  const maximumWrap = totalSum - minimumNormal;
  if (!Number.isSafeInteger(maximumWrap)) {
    throw new RangeError("circular sum exceeds the safe-integer range");
  }
  return Math.max(maximumNormal, maximumWrap);
}
```

### 13. Code Walkthrough
The scan maintains ordinary maximum and minimum ending-at-current states plus their global results. The negative maximum guard identifies the strictly all-negative case. Otherwise, removing the minimum range leaves a valid non-empty wrapping candidate or ties safely with the normal result.

### 14. Complexity Analysis
**Time: `O(n)`** for one scan. **Space: `O(1)`** auxiliary space.

### 15. Edge Cases
- Empty input throws.
- One value returns that value.
- All-negative input returns the largest single value.
- Zeros prevent the all-negative empty-wrap problem.
- The ordinary non-wrapping range may still be best.

### 16. Common Mistakes
- Returning `totalSum - minimumNormal` for an all-negative array.
- Allowing a range longer than `n` in the baseline.
- Duplicating the array and permitting an index to be selected twice.
- Forgetting that the best answer may not wrap.

### 17. Interview Follow-ups
1. Return wrapped index boundaries. 2. Apply a tie rule between wrapping and normal ranges. 3. Find the minimum circular subarray. 4. Limit the chosen length.

### 18. Variations
- Circular maximum with length bounds. - Maximum circular average. - Circular House Robber. - Circular gas-station feasibility.

### 19. Pattern Connections
This extends Problem 76 by running both maximum and minimum forms of Kadane. The complement equation resembles prefix subtraction: remove one summarized region to obtain the retained region.

### 20. NestJS Project Connection
Expose `POST /api/dsa/kadane/maximum-circular-subarray` with `{ values }`. Return the result and bounded optimized events containing maximum/minimum ending states, global states, total, and the final normal-versus-wrap decision.

### 21. Mini Challenge
Return whether the winning subarray wraps, along with its start and end indexes under a documented tie rule.

## Problem 78 — Best Time to Buy and Sell Stock

### 1. Problem Statement
Given daily stock prices as nonnegative safe integers, return the maximum profit from at most one purchase followed by at most one later sale. Return `0` when no profitable transaction exists. Do not mutate the input.

### 2. Example
```text
Input:  [7, 1, 5, 3, 6, 4]
Output: 5
Explanation: buy at 1 and later sell at 6.
```

### 3. Understand the Problem
The purchase must occur before the sale, and only one share is involved. Choosing no transaction is allowed, so the answer cannot be negative.

#### In Simple Terms
Look through prices in date order. You may buy once and sell once on a later day. Return the most money you could gain, or `0` if every possible sale would lose money.

### 4. Real-World Analogy
You may buy one concert ticket on an earlier day and resell it once on a later day. You want the largest positive difference between the later and earlier prices.

### 5. Key Observation
For a sale on the current day, the best purchase is the lowest price seen on an earlier day. Only that running minimum and the best profit need to be retained.

### 6. Pattern
```text
Pattern: Running Minimum / Best Prefix
Why this pattern applies: each sale needs the cheapest valid earlier purchase
How to recognize it: maximize later value minus earlier value under index order
Similar problems: maximum difference, minimum prefix, one-transaction profit
```

### 7. Brute Force
Try every earlier buy day with every later sell day. Calculate each profit and retain the largest value, starting from zero.

#### Pseudocode
```text
validate all prices
best = 0
for buy from 0 through n - 2:
    for sell from buy + 1 through n - 1:
        best = max(best, prices[sell] - prices[buy])
return best
```

#### Brute-Force TypeScript
```typescript
function maximumStockProfitBruteForce(prices: number[]): number {
  for (const price of prices) {
    if (!Number.isSafeInteger(price) || price < 0) {
      throw new RangeError("prices must be nonnegative safe integers");
    }
  }

  let bestProfit = 0;
  for (let buy = 0; buy < prices.length; buy += 1) {
    for (let sell = buy + 1; sell < prices.length; sell += 1) {
      bestProfit = Math.max(bestProfit, prices[sell] - prices[buy]);
    }
  }
  return bestProfit;
}
```

#### Visualization
```text
buy at 1:
sell at 5 -> profit 4
sell at 3 -> profit 2
sell at 6 -> profit 5  best
sell at 4 -> profit 3
```

#### Dry Run
For `[7, 1, 5, 3, 6, 4]`, checking every ordered pair finds the largest difference at buy index `1` and sell index `4`.

#### Brute-Force Complexity
**Time: `O(n²)`** for all buy/sell pairs. **Space: `O(1)`** auxiliary space.

### 8. Why Brute Force Is Not Ideal
Every sale day compares itself with all earlier days even though only the smallest earlier price can produce its greatest profit.

### 9. Optimal Approach
Scan left to right. Before treating a price as a sale, compare it with `minimumPrice`, which summarizes all earlier buy choices. Update `bestProfit`, then include the current price in the running minimum for future days.

The sale-before-minimum-update order makes the “later sale” contract explicit, although allowing same-day buy and sale would only add zero profit.

### 10. Visualization
```text
price  minimum before  candidate  best  minimum after
  7       none            —        0        7
  1        7             -6        0        1
  5        1              4        4        1
  3        1              2        4        1
  6        1              5        5        1
  4        1              3        5        1
```

### 11. Step-by-Step Dry Run
| Day | Price | Minimum earlier | Candidate profit | Best |
|---:|---:|---:|---:|---:|
| 0 | 7 | — | — | 0 |
| 1 | 1 | 7 | -6 | 0 |
| 2 | 5 | 1 | 4 | 4 |
| 3 | 3 | 1 | 2 | 4 |
| 4 | 6 | 1 | 5 | 5 |
| 5 | 4 | 1 | 3 | 5 |

### 12. TypeScript Solution
```typescript
function maximumStockProfit(prices: number[]): number {
  let minimumPrice = Number.POSITIVE_INFINITY;
  let bestProfit = 0;

  for (const price of prices) {
    if (!Number.isSafeInteger(price) || price < 0) {
      throw new RangeError("prices must be nonnegative safe integers");
    }

    if (minimumPrice !== Number.POSITIVE_INFINITY) {
      bestProfit = Math.max(bestProfit, price - minimumPrice);
    }
    minimumPrice = Math.min(minimumPrice, price);
  }

  return bestProfit;
}
```

### 13. Code Walkthrough
`minimumPrice` compresses all valid earlier buy days into the only one relevant to the current sale. `bestProfit` starts at zero to represent skipping the transaction. Every price is validated even when no transaction is possible.

### 14. Complexity Analysis
**Time: `O(n)`** for one scan. **Space: `O(1)`** auxiliary space.

### 15. Edge Cases
- Empty or one-day input returns `0`.
- Strictly decreasing prices return `0`.
- Equal prices produce zero profit.
- The lowest price after the best sale cannot be used retroactively.
- Negative, fractional, or unsafe prices throw `RangeError`.

### 16. Common Mistakes
- Subtracting the global minimum from the global maximum without respecting day order.
- Updating the minimum only after profitable days.
- Returning a negative loss instead of zero.
- Accidentally allowing multiple transactions.

### 17. Interview Follow-ups
1. Return buy and sell indexes. 2. Allow unlimited transactions. 3. Allow at most two transactions. 4. Add a transaction fee or cooldown.

### 18. Variations
- Maximum ordered difference. - Stock II. - Stock with fee. - Stock with cooldown.

### 19. Pattern Connections
Like Kadane, this keeps the best prefix state needed by the current position. The state is a minimum purchase price rather than a maximum subarray ending sum.

### 20. NestJS Project Connection
Expose `POST /api/dsa/prefix-state/stock-profit` with `{ prices }`. Return `{ maximumProfit, trace, truncated }`; optimized events should include `day`, `price`, `minimumPriceBefore`, `candidateProfit`, and `bestProfit`.

### 21. Mini Challenge
Return `{ profit, buyDay, sellDay }`, choosing the earliest buy day and then earliest sell day when maximum profits tie.

## Problem 79 — Best Time to Buy and Sell Stock II

### 1. Problem Statement
Given daily stock prices as nonnegative safe integers, return the maximum total profit from any number of transactions. You may hold at most one share, must sell before buying again, and may choose no transaction. Do not mutate the input, and throw `RangeError` if total profit becomes unsafe.

### 2. Example
```text
Input:  [7, 1, 5, 3, 6, 4]
Output: 7
Explanation: profit 4 from 1 -> 5 and profit 3 from 3 -> 6.
```

### 3. Understand the Problem
Unlike Problem 78, several non-overlapping buy/sell pairs are allowed. Only one share may be held, so a new purchase cannot occur while a previous position remains open.

#### In Simple Terms
You may buy and sell repeatedly across different days, but you can own only one share at a time. Add together the gains from the best sequence of trades. Skip any trade that would lose money.

### 4. Real-World Analogy
You can repeatedly buy an item before a local price rise and sell it afterward, but you must finish one resale before purchasing the next item.

### 5. Key Observation
Every multi-day rise can be split into adjacent positive rises without changing total profit:

```text
(later - earlier) = sum of each positive step along that rising segment
```

Therefore collecting every positive day-to-day difference is globally optimal.

### 6. Pattern
```text
Pattern: Greedy Positive Differences
Why this pattern applies: each upward step can be captured without blocking another gain
How to recognize it: unlimited sequential transactions with no fee or cooldown
Similar problems: accumulate positive deltas, monotone-run profit, Stock with constraints
```

### 7. Brute Force
Recursively consider each day. If not holding, either skip or buy. If holding, either keep holding or sell. Explore every valid decision sequence and return the best completed profit.

#### Pseudocode
```text
search(day, holding):
    if day equals n:
        return 0 if not holding, otherwise impossible
    skip = search(day + 1, holding)
    if holding:
        trade = price[day] + search(day + 1, false)
    else:
        trade = -price[day] + search(day + 1, true)
    return max(skip, trade)
```

#### Brute-Force TypeScript
```typescript
function maximumUnlimitedStockProfitBruteForce(prices: number[]): number {
  for (const price of prices) {
    if (!Number.isSafeInteger(price) || price < 0) {
      throw new RangeError("prices must be nonnegative safe integers");
    }
  }

  const search = (day: number, holding: boolean): number => {
    if (day === prices.length) {
      return holding ? Number.NEGATIVE_INFINITY : 0;
    }

    const skipProfit = search(day + 1, holding);
    const futureProfit = search(day + 1, !holding);
    const tradeProfit = holding
      ? prices[day] + futureProfit
      : -prices[day] + futureProfit;

    if (Number.isFinite(tradeProfit) && !Number.isSafeInteger(tradeProfit)) {
      throw new RangeError("profit exceeds the safe-integer range");
    }
    return Math.max(skipProfit, tradeProfit);
  };

  return search(0, false);
}
```

#### Visualization
```text
day 0, not holding
├─ skip
│  ├─ skip ...
│  └─ buy ...
└─ buy
   ├─ hold ...
   └─ sell ...
```

#### Dry Run
The decision tree for `[7, 1, 5, 3, 6, 4]` explores both actions on every day. Its best completed path buys at `1`, sells at `5`, buys at `3`, and sells at `6`, totaling `7`.

#### Brute-Force Complexity
**Time: `O(2^n)`** because the un-memoized recursion branches on each day. **Space: `O(n)`** for the recursion stack.

### 8. Why Brute Force Is Not Ideal
The decision tree repeatedly solves the same `(day, holding)` states. More importantly, this fee-free unlimited contract has a direct greedy property that makes state search unnecessary.

### 9. Optimal Approach
For each day after the first, calculate `prices[day] - prices[day - 1]`. Add it when positive and ignore it otherwise.

On a rise `1 -> 3 -> 6`, collecting `(3 - 1) + (6 - 3)` equals the single trade `(6 - 1)`. Splitting gains therefore loses nothing and automatically handles every rising run.

### 10. Visualization
```text
prices: 7 -> 1 -> 5 -> 3 -> 6 -> 4
deltas:    -6    +4    -2    +3    -2
take:             4           3
total: 7
```

### 11. Step-by-Step Dry Run
| Day transition | Difference | Action | Total profit |
|---|---:|---|---:|
| `7 -> 1` | -6 | skip | 0 |
| `1 -> 5` | 4 | collect | 4 |
| `5 -> 3` | -2 | skip | 4 |
| `3 -> 6` | 3 | collect | 7 |
| `6 -> 4` | -2 | skip | 7 |

### 12. TypeScript Solution
```typescript
function maximumUnlimitedStockProfit(prices: number[]): number {
  for (const price of prices) {
    if (!Number.isSafeInteger(price) || price < 0) {
      throw new RangeError("prices must be nonnegative safe integers");
    }
  }

  let totalProfit = 0;
  for (let day = 1; day < prices.length; day += 1) {
    const dailyGain = prices[day] - prices[day - 1];
    if (dailyGain > 0) {
      totalProfit += dailyGain;
      if (!Number.isSafeInteger(totalProfit)) {
        throw new RangeError("profit exceeds the safe-integer range");
      }
    }
  }
  return totalProfit;
}
```

### 13. Code Walkthrough
The validation pass enforces the price contract even for short input. The second pass collects only positive adjacent gains. Those gains can be viewed as separate transactions or merged across an uninterrupted rise; both interpretations produce the same total.

### 14. Complexity Analysis
**Time: `O(n)`** for validation and one gain scan. **Space: `O(1)`** auxiliary space.

### 15. Edge Cases
- Empty or one-day input returns `0`.
- Decreasing or constant prices return `0`.
- One long rise collects the same profit as one buy and one sale.
- Alternating rises and falls can produce several transactions.
- Unsafe accumulated profit throws `RangeError`.

### 16. Common Mistakes
- Reusing the one-transaction algorithm from Problem 78.
- Adding negative differences.
- Applying this greedy rule when fees, cooldowns, or transaction limits exist.
- Interpreting every positive delta as simultaneous holdings.

### 17. Interview Follow-ups
1. Return the transaction intervals. 2. Add a fixed transaction fee. 3. Add a cooldown day. 4. Limit the number of transactions to `k`.

### 18. Variations
- Stock with transaction fee. - Stock with cooldown. - At most two transactions. - At most `k` transactions.

### 19. Pattern Connections
This accumulates locally independent positive contributions, whereas Kadane may need to carry a temporary negative value to reach a better contiguous total. Contract details determine whether local greed is valid.

### 20. NestJS Project Connection
Expose `POST /api/dsa/greedy/stock-profit-unlimited` with `{ prices }`. Return `{ maximumProfit, trace, truncated }`; bounded optimized events should show each adjacent pair, its difference, whether it was collected, and the running profit.

### 21. Mini Challenge
Return a minimal list of `[buyDay, sellDay]` transactions that earns the same maximum profit by merging each uninterrupted rising run.

## Problem 80 — Product of Array Except Self

### 1. Problem Statement
Given an array of safe integers, return an array where `result[i]` is the product of every input value except `values[i]`. Do not use division and do not mutate the input. Return `[]` for empty input and `[1]` for one value, using `1` as the empty product. Throw `RangeError` for unsafe multiplication.

### 2. Example
```text
Input:  [1, 2, 3, 4]
Output: [24, 12, 8, 6]
```

### 3. Understand the Problem
Every output position excludes exactly the input at that same index, not every occurrence of the same value. Zeros and negative values are allowed, and division is forbidden.

#### In Simple Terms
For each position, multiply all the other numbers and place that product in the answer. Skip only the number at the current position. Build the answer without calculating one giant product and dividing.

### 4. Real-World Analogy
For each team member, calculate the combined contribution multiplier of everyone else. You need the group before that member and the group after that member.

### 5. Key Observation
The product excluding index `i` equals the product strictly before `i` multiplied by the product strictly after `i`. Those two directional contributions can be built without division.

### 6. Pattern
```text
Pattern: Prefix Products + Rolling Suffix Product
Why this pattern applies: each answer combines information from both sides of an index
How to recognize it: compute all-except-current values without division
Similar problems: prefix/suffix maxima, trapped water, left/right aggregate products
```

### 7. Brute Force
For each output index, scan the complete input and multiply every value whose index differs from the excluded index.

#### Pseudocode
```text
validate all values
result = []
for excluded from 0 through n - 1:
    product = 1
    for index from 0 through n - 1:
        if index differs from excluded:
            product *= values[index]
            validate product
    append product
return result
```

#### Brute-Force TypeScript
```typescript
function productExceptSelfBruteForce(values: number[]): number[] {
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
  }

  const result: number[] = [];
  for (let excluded = 0; excluded < values.length; excluded += 1) {
    let product = 1;
    for (let index = 0; index < values.length; index += 1) {
      if (index === excluded) continue;
      product *= values[index];
      if (!Number.isSafeInteger(product)) {
        throw new RangeError("product exceeds the safe-integer range");
      }
    }
    result.push(product);
  }
  return result;
}
```

#### Visualization
```text
exclude index 0:     2 * 3 * 4 = 24
exclude index 1: 1     * 3 * 4 = 12
exclude index 2: 1 * 2     * 4 = 8
exclude index 3: 1 * 2 * 3     = 6
```

#### Dry Run
For `[1, 2, 3, 4]`, the baseline performs three multiplications for each of four output positions and returns `[24, 12, 8, 6]`.

#### Brute-Force Complexity
**Time: `O(n²)`** because every output scans the array. **Space: `O(n)`** for the required result and `O(1)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
Products on the same side of nearby indexes are repeatedly recalculated. Division would appear faster but violates the contract and requires special zero handling.

### 9. Optimal Approach
Use the output array to store directional state:

1. scan left to right, writing the product strictly before each index
2. scan right to left with one rolling suffix product
3. multiply each stored prefix by the suffix strictly after that index

Validate input first. Only calculate partial products that a later output needs, avoiding an unused full-array product.

### 10. Visualization
```text
values:          [1,  2, 3, 4]
prefix before:   [1,  1, 2, 6]
suffix after:    [24, 12, 4, 1]
combined result: [24, 12, 8, 6]
```

### 11. Step-by-Step Dry Run
Forward pass:

| Index | Stored prefix before index | Next prefix |
|---:|---:|---:|
| 0 | 1 | 1 |
| 1 | 1 | 2 |
| 2 | 2 | 6 |
| 3 | 6 | not needed |

Backward pass:

| Index | Stored prefix | Suffix after | Output | Next suffix |
|---:|---:|---:|---:|---:|
| 3 | 6 | 1 | 6 | 4 |
| 2 | 2 | 4 | 8 | 12 |
| 1 | 1 | 12 | 12 | 24 |
| 0 | 1 | 24 | 24 | not needed |

### 12. TypeScript Solution
```typescript
function productExceptSelf(values: number[]): number[] {
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
  }

  const result = Array<number>(values.length).fill(1);
  let prefixProduct = 1;

  for (let index = 0; index < values.length; index += 1) {
    result[index] = prefixProduct;
    if (index + 1 < values.length) {
      prefixProduct *= values[index];
      if (!Number.isSafeInteger(prefixProduct)) {
        throw new RangeError("prefix product exceeds the safe-integer range");
      }
    }
  }

  let suffixProduct = 1;
  for (let index = values.length - 1; index >= 0; index -= 1) {
    result[index] *= suffixProduct;
    if (!Number.isSafeInteger(result[index])) {
      throw new RangeError("result product exceeds the safe-integer range");
    }

    if (index > 0) {
      suffixProduct *= values[index];
      if (!Number.isSafeInteger(suffixProduct)) {
        throw new RangeError("suffix product exceeds the safe-integer range");
      }
    }
  }

  return result;
}
```

### 13. Code Walkthrough
The forward pass writes each index's left-side product before including that value. The backward pass combines the right-side product before extending it leftward. The returned array doubles as prefix storage, so no second `O(n)` helper array is needed.

### 14. Complexity Analysis
**Time: `O(n)`** for validation and two directional passes. **Space: `O(n)`** for the required result and `O(1)` auxiliary space.

### 15. Edge Cases
- Empty input returns `[]`.
- One value returns `[1]`, the product of no other values.
- One zero makes only its own output potentially nonzero.
- Two or more zeros make every output zero.
- Negative values affect output signs normally.
- Unsafe partial or final products throw `RangeError`.

### 16. Common Mistakes
- Using division despite the contract.
- Dividing a total product without handling zeros.
- Including `values[i]` in its own prefix or suffix contribution.
- Claiming `O(1)` total space without separating the required output.
- Multiplying an unused full-array prefix and throwing unnecessarily.

### 17. Interview Follow-ups
1. Use `bigint` for exact large products. 2. Generalize to another associative operation with an identity. 3. Return results modulo a positive integer. 4. Explain whether true in-place output is possible under an immutable-input contract.

### 18. Variations
- Prefix and suffix maximum. - Product except self modulo `m`. - Matrix row/column products. - All-except-current sums.

### 19. Pattern Connections
This is a multiplicative counterpart to prefix sums. It also resembles the two-direction state used by other array problems, but compresses the backward state to one rolling scalar.

### 20. NestJS Project Connection
Expose `POST /api/dsa/prefix-state/product-except-self` with `{ values }`. Return `{ result, trace, truncated }`; bounded optimized events should distinguish forward prefix writes from backward suffix combinations and include overflow failures without exposing partial results as successful output.

### 21. Mini Challenge
Implement the same contract for `bigint[]` and verify cases with zero, multiple zeros, negative values, empty input, and one value.

# Topic Summary

## What You Learned

- Prefix state replaces repeated rescanning with one evolving summary.
- Leading-zero prefix arrays make immutable inclusive range queries constant time.
- A total plus a running left sum can derive the right side without extra storage.
- Prefix-frequency maps count arbitrary signed subarray sums by matching earlier boundaries.
- Kadane's algorithm compresses a contiguous dynamic-programming recurrence into two scalars.
- Circular maximum subarrays combine ordinary maximum and minimum Kadane scans.
- Stock-profit contracts change the correct state: one transaction needs a running minimum, while unlimited fee-free trades allow greedy positive differences.
- Prefix and suffix products compute all-except-current answers without division.
- Empty selections, ties, mutation, indexes, and numeric safety belong in the contract.
- Educational traces add storage and should be bounded separately from algorithm state.

## Patterns You Should Recognize

| Signal in the problem | Likely pattern |
|---|---|
| Cumulative answer at every index | Inclusive prefix state |
| Many range sums on immutable data | Leading-zero prefix array |
| Compare totals on both sides of an index | Total minus running prefix |
| Count target sums with negatives allowed | Prefix sum plus frequency map |
| Maximum non-empty contiguous sum | Kadane's extend-or-restart recurrence |
| Circular maximum contiguous sum | Maximum Kadane plus minimum Kadane |
| Maximize later price minus earlier price once | Running minimum |
| Unlimited trades without fee or cooldown | Sum positive adjacent differences |
| Every output excludes its own element | Prefix contribution plus suffix contribution |

## Complexity Cheat Sheet

| Problem or operation | Time | Auxiliary space |
|---|---:|---:|
| Running sum | `O(n)` | `O(1)` excluding output |
| Build range-sum prefix array | `O(n)` | `O(n)` |
| One immutable range query | `O(1)` | `O(1)` |
| Equilibrium or pivot scan | `O(n)` | `O(1)` excluding output |
| Count subarrays equal to `k` | average `O(n)` | `O(n)` |
| Maximum subarray | `O(n)` | `O(1)` |
| Maximum circular subarray | `O(n)` | `O(1)` |
| One stock transaction | `O(n)` | `O(1)` |
| Unlimited fee-free stock transactions | `O(n)` | `O(1)` |
| Product except self | `O(n)` | `O(1)` excluding output |

## Common Interview Traps

- Mixing inclusive prefix arrays with leading-zero prefix arrays.
- Using the wrong prefix indexes for an inclusive range.
- Forgetting the seeded zero prefix in target-sum counting.
- Replacing a frequency map with a set when repeated prefixes represent multiple starts.
- Applying a shrinking window to arbitrary signed sums.
- Initializing Kadane to zero when the selected subarray must be non-empty.
- Letting the circular formula select an empty range for all-negative input.
- Ignoring the buy-before-sell order in one-transaction stock profit.
- Applying the unlimited-transactions greedy rule to fees, cooldowns, or transaction limits.
- Using division for Product Except Self and mishandling zeros.
- Reporting required output or trace storage as free auxiliary space.
- Checking input values but not accumulated sums, counts, profits, or products.

## Practice Checklist

- [ ] 71. Build a non-mutating running sum and state its output-space cost.
- [ ] 72. Derive inclusive range sums from a leading-zero prefix array.
- [ ] 73. Return every equilibrium index, including valid endpoints.
- [ ] 74. Return the leftmost pivot with constant auxiliary space.
- [ ] 75. Count target-sum subarrays using prefix frequencies.
- [ ] 76. Apply Kadane correctly to an all-negative array.
- [ ] 77. Explain and guard the all-negative circular case.
- [ ] 78. Enforce buy-before-sell order with a running minimum.
- [ ] 79. Prove why positive adjacent gains are optimal for unlimited trades.
- [ ] 80. Compute Product Except Self without division or helper arrays.