# Hashing

## Learning Objectives

By the end of this topic, you should understand:

- how hash tables connect keys to stored values
- when to use a TypeScript `Map` versus a `Set`
- why hash lookup is average `O(1)`, not guaranteed `O(1)`
- how hashing replaces repeated scans with remembered information
- how to count frequencies, detect duplicates, and find complements
- how prefix sums and frequency maps solve subarray problems
- how canonical keys group structurally equivalent values such as anagrams
- how to state ordering, distinctness, and pair-counting contracts precisely
- how to visualize hash state safely in a NestJS learning API

## Prerequisites

- the contracts, invariants, and Big-O reasoning from `00-dsa-fundamentals.md`
- array traversal and output-space distinctions from `01-arrays.md`
- TypeScript arrays, strings, `Map`, `Set`, loops, and functions

## Core Theory

### What is hashing?

**Hashing** converts a key into a hash value that helps choose where its data should be stored. A **hash table** uses that result to support fast lookup, insertion, and deletion by key.

```text
Key          Hashing process          Storage location
"apple"  ----------------------->    bucket 3
"pear"   ----------------------->    bucket 7
```

In TypeScript, `Map` and `Set` provide the hash-based behavior needed for most DSA problems:

- `Map<Key, Value>` stores a value for each key.
- `Set<Value>` stores membership only.

### Why does hashing exist?

An unsorted array may require scanning many values to answer, "Have I seen `x`?"

```text
Array lookup: inspect 8, then 3, then 11, then 5...
Set lookup:   ask seen.has(5)
```

Remembering processed information trades extra memory for faster repeated questions.

### Real-world mental model

A library catalog does not make you inspect every shelf from the beginning. It turns a known key, such as a title or identifier, into location information. A hash table serves a similar purpose inside a program.

### Map versus Set

| Need | Structure | Example state |
|---|---|---|
| Has this value appeared? | `Set` | `{2, 7, 11}` |
| How many times has it appeared? | `Map` | `2 -> 3` |
| Where did it first appear? | `Map` | `7 -> index 1` |
| Which data belongs to this key? | `Map` | `"aet" -> ["eat", "tea"]` |

Use a `Set` when presence is the whole answer. Use a `Map` when each key needs associated state.

### What does a hash table look like internally?

A simplified table contains buckets:

```text
bucket 0: [key A -> value]
bucket 1: empty
bucket 2: [key B -> value] -> [key C -> value]
bucket 3: [key D -> value]
```

Different keys can choose the same bucket. This is a **collision**. Real implementations resolve collisions and resize storage as data grows.

### Why is lookup average O(1)?

With a well-distributed hash function and controlled load, the table reaches a small bucket directly. That is modeled as average `O(1)` work.

In a pathological collision case, many keys may share a location and lookup can degrade. Therefore this chapter says:

```text
Map/Set lookup and insertion: average O(1)
not universally guaranteed O(1)
```

### When should I use hashing?

Use hashing when a problem repeatedly asks about:

- membership or duplicates
- frequencies
- complements such as `target - value`
- previously seen prefix sums
- grouping by a computed identity
- cached information associated with a key

### When should I not use hashing?

Hashing may be the wrong first tool when:

- sorted order already supports two pointers or binary search
- keys must be processed in sorted order
- memory must remain constant
- the key domain is tiny enough for a fixed frequency array
- range queries, hierarchy, or priority are the main operations

### Keys and equality in TypeScript

Primitive keys compare by value. Object keys compare by identity:

```typescript
const first = { id: 1 };
const second = { id: 1 };

first === second; // false
```

If objects with the same fields should group together, compute a stable primitive key such as an ID or canonical string.

## Visual Mental Model

### A Set remembers membership

```text
Input: [4, 7, 4]

read 4 -> seen = {4}
read 7 -> seen = {4, 7}
read 4 -> already in seen
```

### A Map remembers associated state

```text
Value -> frequency

4 -> 2
7 -> 1
```

### Complement lookup

```text
target = 9
current = 7
needed = 9 - 7 = 2

seen indexes:
2 -> 0

2 exists, so indexes 0 and current form the answer.
```

### Prefix-sum lookup

```text
prefix at j - prefix before i = k

prefix before i = prefix at j - k
```

A map remembers earlier prefix sums so a subarray question becomes a lookup instead of a nested scan.

## Complexity Concepts

Let `n` be the number of input values and `u` the number of distinct keys.

| Operation | Typical time | Why? |
|---|---:|---|
| `map.get(key)` / `map.set(key, value)` | Average `O(1)` | Hashing chooses a storage location. |
| `set.has(value)` / `set.add(value)` | Average `O(1)` | Membership uses hashed storage. |
| Build frequencies | Average `O(n)` | `n` average constant-time updates. |
| Iterate all stored keys | `O(u)` | Every distinct key is visited. |
| Sort stored keys | `O(u log u)` | Ordering requires sorting. |
| Store all distinct keys | `O(u)`, worst `O(n)` | Every input may be distinct. |

### Why hashing often changes O(n²) to average O(n)

Brute force commonly performs a scan for every element:

```text
n outer choices x n lookup scan = O(n^2)
```

Replacing each scan with average `O(1)` lookup gives:

```text
n elements x average O(1) lookup = average O(n)
```

### Hashing does not remove output cost

If a problem returns `p` pairs, writing those pairs costs `O(p)`. A solution may therefore be `O(n + p)` even when lookup is average constant time.

### Key construction has a cost

For strings of length `m`, sorting characters to build a canonical anagram key costs `O(m log m)`. Hash lookup does not erase the work needed to create the key.

## Common Patterns

### Seen set

Store values already processed. Use it for duplicate detection and consecutive-sequence boundaries.

### Frequency map

Store `value -> count`. Use it for repeated/non-repeating values, pair counts, and grouping.

### Complement map

Transform a target equation into a lookup: if `a + b = target`, then `b = target - a`.

### Prefix sum plus map

Store earlier cumulative totals or their counts. Use it for zero-sum and sum-`k` subarrays.

### Canonical key

Transform equivalent items into the same key. Sorted letters or character counts can represent an anagram group.

### Start-of-run detection

For a value set, begin a consecutive run only when `value - 1` is absent. This prevents recounting the same run from every member.

### Coordinated sorted pointers

When inputs are sorted, move the pointer at the smallest current value instead of building extra hash storage. Hashing is useful, but existing order may offer the better tradeoff.

### Pair-sum hashing

Store sums of earlier index pairs so a later pair can search for its required complement. Enforce index boundaries so the two pairs cannot reuse a position.

---

# Problems

## Problem 21 — Two Sum

### 1. Problem Statement

Given an array and a target, return indexes of two distinct elements whose values sum to the target. Assume exactly one valid pair.

### 2. Example

```text
Input: nums = [2, 7, 11, 15], target = 9
Output: [0, 1]
```

### 3. Understand the Problem

Return indexes, not values. One element cannot be used twice, and input order remains unchanged.

#### In Simple Terms

Find two different positions whose values add to the target, and return those two indexes.

### 4. Real-World Analogy

Choose two item prices that exactly spend a gift-card balance. For each price, ask which partner price is still needed.

### 5. Key Observation

For current value `x`, the required earlier value is `target - x`. A map can remember each earlier value's index.

### 6. Pattern

```text
Pattern: Complement lookup
```

Recognize equations with two values and a fixed target, especially on unsorted input.

### 7. Brute Force

Try every pair `i < j` and return the pair whose sum matches.

#### Pseudocode

```text
for i from 0 to n - 1
    for j from i + 1 to n - 1
        if nums[i] + nums[j] equals target
            return [i, j]
```

#### Brute-Force TypeScript

```typescript
function twoSumByCheckingPairs(nums: number[], target: number): [number, number] {
  for (let first = 0; first < nums.length - 1; first += 1) {
    for (let second = first + 1; second < nums.length; second += 1) {
      if (nums[first] + nums[second] === target) return [first, second];
    }
  }
  throw new Error("Expected exactly one valid pair");
}
```

#### Visualization

```text
[2, 7, 11, 15]
 ^  ^             2 + 7 = 9, found
```

#### Dry Run

For `i=0`, compare `2` with `7`. Their sum is `9`, so return `[0,1]`.

#### Brute-Force Complexity

There can be `n(n-1)/2` pairs, which grows as `O(n²)`. Only indexes are stored, so auxiliary space is `O(1)`.

### 8. Why Brute Force Is Not Ideal

Each current value searches earlier or later values repeatedly.

### 9. Optimal Approach

Scan left to right. Before storing the current value, check whether its complement is already mapped. Checking first prevents using the current element twice.

Invariant: the map contains values and indexes from positions strictly before the current index.

### 10. Visualization

```text
index 0: value 2, need 7, absent -> store 2 -> 0
index 1: value 7, need 2, found at 0 -> [0, 1]
```

### 11. Step-by-Step Dry Run

| Index | Value | Needed | Map before | Decision |
|---:|---:|---:|---|---|
| 0 | 2 | 7 | `{}` | Store `2 -> 0` |
| 1 | 7 | 2 | `{2 -> 0}` | Return `[0,1]` |

### 12. TypeScript Solution

```typescript
function twoSum(nums: number[], target: number): [number, number] {
  const indexByValue = new Map<number, number>();

  for (let index = 0; index < nums.length; index += 1) {
    const needed = target - nums[index];
    const matchIndex = indexByValue.get(needed);

    if (matchIndex !== undefined) {
      return [matchIndex, index];
    }

    indexByValue.set(nums[index], index);
  }

  throw new Error("Expected exactly one valid pair");
}
```

### 13. Code Walkthrough

`needed` rewrites the sum as a lookup. Testing `!== undefined` is safer than truthiness because index `0` is valid. Storing after checking enforces distinct indexes.

### 14. Complexity Analysis

**Time: average `O(n)`** because `n` iterations each perform average `O(1)` map operations. **Space: `O(n)`** because the map may store every value before the answer.

### 15. Edge Cases

- Duplicate values can form a pair, such as `[3,3]` for target `6`.
- Negative values and zero work normally.
- The stated contract guarantees a solution.
- Floating-point precision can affect decimal sums.

### 16. Common Mistakes

- Returning values instead of indexes.
- Checking map results by truthiness and losing index `0`.
- Storing first and using the same index twice.
- Sorting without preserving original indexes.

### 17. Interview Follow-ups

1. Return all distinct pairs.
2. What if the input is sorted?
3. Count pairs instead of returning one.
4. Handle no-solution input without throwing.

### 18. Variations

- Two Sum in a sorted array.
- Three Sum and Four Sum.
- Pair with a given difference.

### 19. Pattern Connections

Complement lookup appears in pair counting, subarray equations, and balancing two collections.

### 20. NestJS Project Connection

Expose `POST /api/dsa/hashing/two-sum` with `nums`, `target`, and optional tracing. Return steps such as `{ index, value, needed, matchIndex, storedKeys }`, limiting key snapshots for large requests.

### 21. Mini Challenge

Return `null` when no pair exists and preserve the earliest valid pair when several answers are possible.

## Problem 22 — Contains Duplicate

### 1. Problem Statement

Return `true` if any value appears at least twice; otherwise return `false`.

### 2. Example

```text
Input: [1, 2, 3, 1]
Output: true
```

### 3. Understand the Problem

Only existence matters, not which duplicate or how often it appears. The function may stop at the first repeat.

#### In Simple Terms

Return `true` as soon as you find the same value in two different positions. Otherwise return `false`.

### 4. Real-World Analogy

While checking ticket numbers, stop as soon as a number already checked in appears again.

### 5. Key Observation

A duplicate is exactly a value already present in the processed-prefix set.

### 6. Pattern

```text
Pattern: Seen set with early exit
```

Use it when repeated membership is the only needed state.

### 7. Brute Force

Compare each value with every later value.

#### Pseudocode

```text
for each i
    for each j after i
        if values are equal, return true
return false
```

#### Brute-Force TypeScript

```typescript
function containsDuplicateByCheckingPairs(values: number[]): boolean {
  for (let first = 0; first < values.length - 1; first += 1) {
    for (let second = first + 1; second < values.length; second += 1) {
      if (values[first] === values[second]) return true;
    }
  }
  return false;
}
```

#### Visualization

```text
1 compares with 2, 3, then 1 -> duplicate
```

#### Dry Run

For `[1,2,3,1]`, the first and last positions match, so return `true`.

#### Brute-Force Complexity

An all-distinct array checks `n(n-1)/2` pairs: `O(n²)` time and `O(1)` auxiliary space.

### 8. Why Brute Force Is Not Ideal

Earlier values are searched repeatedly. A set remembers them once.

### 9. Optimal Approach

For each value, return `true` if `seen.has(value)`; otherwise add it. Return `false` after the full traversal.

### 10. Visualization

```text
1 -> seen {1}
2 -> seen {1,2}
3 -> seen {1,2,3}
1 -> already present -> true
```

### 11. Step-by-Step Dry Run

| Value | Seen before? | Action |
|---:|:---:|---|
| 1 | No | Add |
| 2 | No | Add |
| 3 | No | Add |
| 1 | Yes | Return `true` |

### 12. TypeScript Solution

```typescript
function containsDuplicate(values: number[]): boolean {
  const seen = new Set<number>();

  for (const value of values) {
    if (seen.has(value)) return true;
    seen.add(value);
  }

  return false;
}
```

### 13. Code Walkthrough

Membership is checked before insertion. Early return avoids processing values after the answer is known.

### 14. Complexity Analysis

**Time: average `O(n)` worst traversal** because each of `n` values uses average `O(1)` set operations. Best case is `O(1)` when the first two values match. **Space: `O(n)`** in the all-distinct case.

### 15. Edge Cases

- Empty and one-value arrays return `false`.
- Adjacent or distant duplicates both work.
- `NaN` is considered equal to `NaN` by `Set` membership.

### 16. Common Mistakes

- Comparing only adjacent values on unsorted input.
- Building the entire set and missing early exit.
- Claiming worst-case guaranteed constant hash lookup.
- Returning the duplicate instead of a boolean.

### 17. Interview Follow-ups

1. Return the first repeated value.
2. Detect duplicates within distance `k`.
3. Solve by sorting with lower extra space.

### 18. Variations

- Contains nearby duplicate.
- Find all duplicate values.
- Duplicate objects by selected key.

### 19. Pattern Connections

Seen sets also support cycle detection, graph visitation, and unique-window tracking.

### 20. NestJS Project Connection

Use `POST /api/dsa/hashing/contains-duplicate`; return the duplicate index as optional educational metadata even though the algorithm result is boolean.

### 21. Mini Challenge

Return the indexes of the first pair of equal values encountered during left-to-right traversal.

## Problem 23 — Count Frequencies of Array Elements

### 1. Problem Statement

Return a map from each array value to the number of times it appears.

### 2. Example

```text
Input: [2, 3, 2, 5, 3, 2]
Output: 2 -> 3, 3 -> 2, 5 -> 1
```

### 3. Understand the Problem

Every distinct value becomes one key. Encounter order is preserved by TypeScript `Map`, but frequency correctness does not depend on order.

#### In Simple Terms

For every different value, report how many times that value appears in the array.

### 4. Real-World Analogy

Tally votes by adding one mark beside a candidate's name for each ballot.

### 5. Key Observation

The new count is the previous count, or zero if absent, plus one.

### 6. Pattern

```text
Pattern: Frequency map
```

Recognize requests involving counts, repetitions, uniqueness, or grouping.

### 7. Brute Force

For each distinct candidate, scan the whole array to count matches, while separately avoiding repeated candidates.

#### Pseudocode

```text
for each value not counted before
    scan all values and count matches
    store count
```

#### Brute-Force TypeScript

```typescript
function countFrequenciesByRescanning(values: number[]): Map<number, number> {
  const frequencies = new Map<number, number>();
  for (const candidate of values) {
    if (frequencies.has(candidate)) continue;
    let count = 0;
    for (const value of values) {
      if (value === candidate) count += 1;
    }
    frequencies.set(candidate, count);
  }
  return frequencies;
}
```

#### Visualization

```text
candidate 2 -> scan six values -> count 3
candidate 3 -> scan six values -> count 2
candidate 5 -> scan six values -> count 1
```

#### Dry Run

Three distinct candidates each trigger a full scan of `[2,3,2,5,3,2]`.

#### Brute-Force Complexity

With `n` distinct values, `n` candidates each scan `n` elements: `O(n²)` time. Stored frequencies require `O(n)` space.

### 8. Why Brute Force Is Not Ideal

The same input is rescanned. Counts can be updated during one traversal.

### 9. Optimal Approach

Read the previous count with `get`, default it to zero, add one, and write it back.

Invariant: after processing a prefix, the map exactly counts that prefix.

### 10. Visualization

```text
2 -> {2:1}
3 -> {2:1, 3:1}
2 -> {2:2, 3:1}
5 -> {2:2, 3:1, 5:1}
3 -> {2:2, 3:2, 5:1}
2 -> {2:3, 3:2, 5:1}
```

### 11. Step-by-Step Dry Run

| Value | Previous count | New count |
|---:|---:|---:|
| 2 | 0 | 1 |
| 3 | 0 | 1 |
| 2 | 1 | 2 |
| 5 | 0 | 1 |
| 3 | 1 | 2 |
| 2 | 2 | 3 |

### 12. TypeScript Solution

```typescript
function countFrequencies(values: number[]): Map<number, number> {
  const frequencies = new Map<number, number>();

  for (const value of values) {
    frequencies.set(value, (frequencies.get(value) ?? 0) + 1);
  }

  return frequencies;
}
```

### 13. Code Walkthrough

The nullish fallback handles absent keys without confusing a stored zero with absence. Each iteration extends the counting invariant by one value.

### 14. Complexity Analysis

**Time: average `O(n)`** for `n` average constant-time map updates. **Space: `O(u)`**, where `u` is the number of distinct values; worst case `u = n`.

### 15. Edge Cases

- Empty input returns an empty map.
- All equal creates one key with count `n`.
- All distinct creates `n` keys with count `1`.
- Negative numbers and zero are valid keys.

### 16. Common Mistakes

- Writing `get(value) + 1` when the first lookup is `undefined`.
- Using a plain object and unintentionally stringifying keys.
- Recounting with `filter` for each value.
- Reporting space as `O(1)` because each individual count is small.

### 17. Interview Follow-ups

1. Return the most frequent value.
2. Sort values by frequency.
3. Use a fixed array when values lie in a small range.
4. Count object keys.

### 18. Variations

- Character frequencies.
- Word frequencies.
- Top `k` frequent elements.

### 19. Pattern Connections

Frequency maps power anagrams, pair counts, majority voting checks, and histogram algorithms.

### 20. NestJS Project Connection

Expose `POST /api/dsa/hashing/frequencies`. Convert the `Map` to JSON-friendly entries such as `[{ value: 2, count: 3 }]`; trace only the updated key and count per step.

### 21. Mini Challenge

Return both the frequency map and the first value that reaches a frequency of three.

## Problem 24 — Find the First Repeating Element

### 1. Problem Statement
Return the value whose **first occurrence has the smallest index** among all values appearing more than once. Return `null` if none repeats.

### 2. Example
```text
Input: [10, 5, 3, 4, 3, 5, 6]
Output: 5
```

### 3. Understand the Problem
Although the second `3` appears before the second `5`, `5` is the answer because its first occurrence is at index `1`, earlier than `3` at index `2`.

#### In Simple Terms

Among all values that appear more than once, return the one whose first copy occurs earliest in the array.

### 4. Real-World Analogy
Among customers who visit more than once, find the one whose original visit happened earliest.

### 5. Key Observation
Frequencies tell which values repeat. A second left-to-right pass finds the earliest index belonging to one of them.

### 6. Pattern
```text
Pattern: Frequency map followed by ordered selection
```
Use it when frequency qualifies candidates but original order chooses the answer.

### 7. Brute Force
For each index, scan later positions for an equal value; return the first index with a match.

#### Pseudocode
```text
for i from left to right
    for j after i
        if values match, return values[i]
return null
```

#### Brute-Force TypeScript

```typescript
function firstRepeatingBySuffixSearch(values: number[]): number | null {
  for (let first = 0; first < values.length - 1; first += 1) {
    for (let second = first + 1; second < values.length; second += 1) {
      if (values[first] === values[second]) return values[first];
    }
  }
  return null;
}
```

#### Visualization
```text
10 -> no later 10
5  -> later 5 at index 5 -> return 5
```

#### Dry Run
The scan for index `0` fails. The scan for index `1` finds another `5`, making it the earliest repeating first occurrence.

#### Brute-Force Complexity
The no-duplicate case checks `O(n²)` pairs and uses `O(1)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
Suffixes are rescanned for every candidate.

### 9. Optimal Approach
Count every value, then scan input from the beginning and return the first value with count greater than one.

### 10. Visualization
```text
frequencies: 10:1, 5:2, 3:2, 4:1, 6:1
scan: 10 -> count 1; 5 -> count 2 -> answer
```

### 11. Step-by-Step Dry Run
The counting pass builds the map above. The selection pass rejects `10` and immediately accepts `5`.

### 12. TypeScript Solution
```typescript
function firstRepeating(values: number[]): number | null {
  const frequencies = countFrequencies(values);
  for (const value of values) {
    if ((frequencies.get(value) ?? 0) > 1) return value;
  }
  return null;
}
```

### 13. Code Walkthrough
The shared frequency helper establishes repeat status. Traversing the original array, not map keys, guarantees the smallest first-occurrence index.

### 14. Complexity Analysis
**Time: average `O(n)`** for two sequential traversals with hash lookups. **Space: `O(u)`**, worst `O(n)`, for frequencies.

### 15. Edge Cases
- Empty or all-distinct returns `null`.
- All equal returns the first value.
- Negative and zero keys work.

### 16. Common Mistakes
- Returning the first duplicate encountered during one forward scan (`3` in the example).
- Iterating map keys and assuming arbitrary desired ordering in other languages.
- Returning an index when the contract asks for a value.

### 17. Interview Follow-ups
1. Return the first index instead.
2. Return the earliest second occurrence.
3. Solve by scanning right to left with a set.

### 18. Variations
- First repeated character.
- First value occurring exactly twice.
- First duplicate within distance `k`.

### 19. Pattern Connections
Frequency qualification plus ordered selection also finds first unique values and stable frequency-based filters.

### 20. NestJS Project Connection
Use `POST /api/dsa/hashing/first-repeating`; show separate `count` and `select` trace phases so the ordering rule is visible.

### 21. Mini Challenge
Return `{ value, firstIndex, secondIndex }` for the answer without changing its selection rule.

## Problem 25 — Find the First Non-Repeating Element

### 1. Problem Statement
Return the earliest value that appears exactly once, or `null` when no such value exists.

### 2. Example
```text
Input: [9, 4, 9, 6, 7, 4]
Output: 6
```

### 3. Understand the Problem
"First" refers to input index. The answer must have total frequency one, not merely be new at the moment it is read.

#### In Simple Terms

Read from left to right and return the first value that appears exactly once in the whole array.

### 4. Real-World Analogy
After all votes arrive, find the earliest submitted choice that no other ballot repeated.

### 5. Key Observation
Uniqueness cannot be confirmed until total frequencies are known; original order then selects the first unique value.

### 6. Pattern
```text
Pattern: Frequency map followed by stable lookup
```
The same two-pass shape as Problem 24 uses a different frequency predicate.

### 7. Brute Force
For each value, scan the full array and count its occurrences; return the first count of one.

#### Pseudocode
```text
for each value from left to right
    count it across the whole array
    if count equals 1, return value
return null
```

#### Brute-Force TypeScript

```typescript
function firstNonRepeatingByCounting(values: number[]): number | null {
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
9 -> count 2
4 -> count 2
9 -> count 2
6 -> count 1 -> answer
```

#### Dry Run
The first three positions fail the frequency test; `6` at index `3` passes.

#### Brute-Force Complexity
Up to `n` full scans produce `O(n²)` time and `O(1)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
The same occurrence information is recalculated for each index.

### 9. Optimal Approach
Build frequencies once, then return the first input value whose mapped count is one.

### 10. Visualization
```text
map: 9:2, 4:2, 6:1, 7:1
order: 9(x), 4(x), 9(x), 6(check) -> answer
```

### 11. Step-by-Step Dry Run
After counting, scan indexes `0..3`: counts are `2,2,2,1`, so stop at `6`.

### 12. TypeScript Solution
```typescript
function firstNonRepeating(values: number[]): number | null {
  const frequencies = countFrequencies(values);
  for (const value of values) {
    if (frequencies.get(value) === 1) return value;
  }
  return null;
}
```

### 13. Code Walkthrough
The frequency map provides a global fact; the second pass preserves the original ordering requirement.

### 14. Complexity Analysis
**Time: average `O(n)`** because two linear passes perform average constant-time map work. **Space: `O(u)`**, at most `O(n)`.

### 15. Edge Cases
- Empty returns `null`.
- One value returns itself.
- All repeated returns `null`.
- Several unique values return the leftmost one.

### 16. Common Mistakes
- Returning the first value before seeing future duplicates.
- Selecting the smallest numeric unique value instead of earliest.
- Using a set, which cannot distinguish count one from count many by itself.

### 17. Interview Follow-ups
1. Process a character stream.
2. Return every non-repeating value.
3. Find the first value with frequency `k`.

### 18. Variations
- First unique character.
- First unique value in a queue stream.
- Stable unique filtering.

### 19. Pattern Connections
Global frequency plus stable order appears in unique-character, voting, and log-analysis tasks.

### 20. NestJS Project Connection
Expose `POST /api/dsa/hashing/first-non-repeating`; return count updates and a distinct selection phase, converting absent results to JSON `null`.

### 21. Mini Challenge
Return the first two non-repeating values while preserving input order.

## Problem 26 — Find Duplicates in an Array
### 1. Problem Statement
Return each value occurring more than once exactly once, in the order its second occurrence is encountered.
### 2. Example
```text
Input: [4,3,2,7,3,2,3]
Output: [3,2]
```
### 3. Understand the Problem
The third `3` must not create another output. The contract uses duplicate-discovery order.

#### In Simple Terms

Return a value when you meet its second copy. Do not return that value again if a third or later copy appears.

### 4. Real-World Analogy
Alert on a reused ticket number once, then suppress later alerts for that number.
### 5. Key Observation
One set tracks values seen once; another tracks duplicates already emitted.
### 6. Pattern
```text
Pattern: Seen set plus emitted set
```
### 7. Brute Force
At each position, count how many copies of that value have appeared from the start through the current position. Emit it exactly when that prefix count becomes two.
#### Pseudocode
```text
for each index
    count equal values from index 0 through the current index
    append the current value if the count equals 2
```

#### Brute-Force TypeScript

```typescript
function findDuplicatesByPrefixCounting(values: number[]): number[] {
  const duplicates: number[] = [];
  for (let index = 0; index < values.length; index += 1) {
    let prefixCount = 0;
    for (let previous = 0; previous <= index; previous += 1) {
      if (values[previous] === values[index]) prefixCount += 1;
    }
    if (prefixCount === 2) duplicates.push(values[index]);
  }
  return duplicates;
}
```

#### Visualization
```text
4:1, 3:3 -> append, 2:2 -> append, 7:1
```
#### Dry Run
The second `3` appears first, so it is emitted before the second `2`; the third `3` has prefix count three and is skipped.
#### Brute-Force Complexity
Up to `n` prefix scans of length `n` give `O(n²)` time and `O(n)` output space.
### 8. Why Brute Force Is Not Ideal
It recalculates occurrence information already observed.
### 9. Optimal Approach
Add unseen values to `seen`. When seen but not emitted, append and mark emitted.
### 10. Visualization
```text
4 new; 3 new; 2 new; 7 new; 3 emit; 2 emit; 3 skip
```
### 11. Step-by-Step Dry Run
| Value | Seen? | Emitted? | Action |
|---:|:---:|:---:|---|
| 3 (second) | Yes | No | Append 3 |
| 2 (second) | Yes | No | Append 2 |
| 3 (third) | Yes | Yes | Skip |
### 12. TypeScript Solution
```typescript
function findDuplicates(values: number[]): number[] {
  const seen = new Set<number>();
  const emitted = new Set<number>();
  const result: number[] = [];
  for (const value of values) {
    if (!seen.has(value)) seen.add(value);
    else if (!emitted.has(value)) {
      emitted.add(value);
      result.push(value);
    }
  }
  return result;
}
```
### 13. Code Walkthrough
The two sets distinguish first occurrence, first duplicate, and later duplicates.
### 14. Complexity Analysis
**Time: average `O(n)`** for set operations. **Space: `O(u)`**, worst `O(n)`.
### 15. Edge Cases
- Empty/all-distinct returns `[]`.
- All equal returns one value.
- Discovery order can differ from first-occurrence order.
### 16. Common Mistakes
- Emitting every occurrence after the first.
- Sorting and changing output order.
- Using one set without suppressing repeated output.
### 17. Interview Follow-ups
1. Return counts. 2. Use first-occurrence order. 3. Solve in-place for values `1..n`.
### 18. Variations
- Duplicate IDs. - Values occurring at least `k` times. - Duplicate characters.
### 19. Pattern Connections
Two-stage membership appears in event deduplication and alert suppression.
### 20. NestJS Project Connection
Expose `POST /api/dsa/hashing/duplicates`; trace `{ value, status: "new" | "emit" | "skip" }`.
### 21. Mini Challenge
Return duplicates in first-occurrence order while retaining average `O(n)` time.

## Problem 27 — Find Common Elements in Three Sorted Arrays
### 1. Problem Statement
Return distinct values present in all three sorted arrays, in sorted order.
### 2. Example
```text
Input: [1,5,10,20,40,80], [6,7,20,80,100], [3,4,15,20,30,70,80]
Output: [20,80]
```
### 3. Understand the Problem
Inputs are non-decreasing; duplicate common values appear once.

#### In Simple Terms

Return each distinct value that can be found in all three sorted arrays.

### 4. Real-World Analogy
Walk three alphabetized rosters and record names found on all three.
### 5. Key Observation
When current values differ, every pointer holding the minimum must advance to possibly catch up.
### 6. Pattern
```text
Pattern: Coordinated pointers on sorted inputs
```
### 7. Brute Force
Build sets for arrays B and C; test distinct values from A.
#### Pseudocode
```text
setB, setC
for each distinct a
    append if both sets contain a
```
#### Brute-Force TypeScript

```typescript
function commonInThreeWithSets(a: number[], b: number[], c: number[]): number[] {
  const inB = new Set(b);
  const inC = new Set(c);
  const result: number[] = [];
  for (const value of a) {
    if (inB.has(value) && inC.has(value) && result.at(-1) !== value) result.push(value);
  }
  return result;
}
```

#### Visualization
```text
20 -> in B and C -> append
80 -> in B and C -> append
```
#### Dry Run
Membership checks produce `[20,80]`.
#### Brute-Force Complexity
Average `O(n+m+p)` time and `O(m+p)` auxiliary space.
### 8. Why Brute Force Is Not Ideal
Sorted order permits the same linear time with constant pointer storage.
### 9. Optimal Approach
Record and advance all when equal; otherwise advance pointers whose current value is minimum.
### 10. Visualization
```text
A ...20...80
B ...20...80
C ...20...80
all equal -> record, advance all
```
### 11. Step-by-Step Dry Run
Pointers discard smaller values until meeting at `20`, then later at `80`; exhaustion stops the search.
### 12. TypeScript Solution
```typescript
function commonInThree(a: number[], b: number[], c: number[]): number[] {
  const result: number[] = [];
  let i = 0, j = 0, k = 0;
  while (i < a.length && j < b.length && k < c.length) {
    if (a[i] === b[j] && b[j] === c[k]) {
      if (result.at(-1) !== a[i]) result.push(a[i]);
      i += 1; j += 1; k += 1;
    } else {
      const min = Math.min(a[i], b[j], c[k]);
      if (a[i] === min) i += 1;
      if (b[j] === min) j += 1;
      if (c[k] === min) k += 1;
    }
  }
  return result;
}
```
### 13. Code Walkthrough
Advancing all minimum ties guarantees progress; the last-output check removes duplicate matches.
### 14. Complexity Analysis
**Time: `O(n+m+p)`** because pointers only move forward. **Auxiliary space: `O(1)` excluding output**.
### 15. Edge Cases
- Any empty array returns `[]`.
- No overlap returns `[]`.
- Repeated common values emit once.
### 16. Common Mistakes
- Advancing only one pointer after equality.
- Forgetting distinct output.
- Using pointer elimination on unsorted input.
### 17. Interview Follow-ups
1. Unsorted inputs. 2. Preserve multiplicity. 3. Intersect `k` arrays.
### 18. Variations
- Two-array intersection. - Sorted streams. - Multi-way merge join.
### 19. Pattern Connections
Coordinated pointers generalize merge operations; heaps handle many sorted inputs.
### 20. NestJS Project Connection
Use `POST /api/dsa/hashing/common-three-sorted`; validate sorting and trace current triples.
### 21. Mini Challenge
Emit each common value according to its minimum frequency across the arrays.

## Problem 28 — Longest Consecutive Sequence
### 1. Problem Statement
Return the longest run length of consecutive integer values in an unsorted array.
### 2. Example
```text
Input: [100,4,200,1,3,2]
Output: 4
```
### 3. Understand the Problem
Consecutive refers to values `1,2,3,4`, not adjacent positions; duplicates do not extend runs.

#### In Simple Terms

Find the longest chain of integer values that increase by one, even when those values are scattered through the array.

### 4. Real-World Analogy
Find the longest uninterrupted run among scattered calendar dates.
### 5. Key Observation
Expand only from `x` when `x-1` is absent, so each run is counted once.
### 6. Pattern
```text
Pattern: Set membership plus run-start detection
```
### 7. Brute Force
From every value, repeatedly scan the array for its successor.
#### Pseudocode
```text
for each value
    while array contains next integer
        extend length
```
#### Brute-Force TypeScript

```typescript
function longestConsecutiveBySearching(values: number[]): number {
  let best = 0;
  for (const start of values) {
    let current = start;
    let length = 1;
    while (values.includes(current + 1)) {
      current += 1;
      length += 1;
    }
    best = Math.max(best, length);
  }
  return best;
}
```

#### Visualization
```text
start 1 -> find 2,3,4
start 2 -> repeats 3,4 work
```
#### Dry Run
Overlapping starts recount the same sequence.
#### Brute-Force Complexity
Repeated linear membership can reach `O(n²)` time; auxiliary space is `O(1)`.
### 8. Why Brute Force Is Not Ideal
Both membership scans and non-boundary starts repeat work.
### 9. Optimal Approach
Create a set and expand only values lacking a predecessor.
### 10. Visualization
```text
1 lacks 0 -> 1,2,3,4 length 4
2 has 1 -> skip as start
100 lacks 99 -> length 1
```
### 11. Step-by-Step Dry Run
Starts `100`, `200`, and `1` produce lengths `1`, `1`, and `4`; best is `4`.
### 12. TypeScript Solution
```typescript
function longestConsecutive(values: number[]): number {
  const set = new Set(values);
  let best = 0;
  for (const value of set) {
    if (set.has(value - 1)) continue;
    let current = value;
    let length = 1;
    while (set.has(current + 1)) { current += 1; length += 1; }
    best = Math.max(best, length);
  }
  return best;
}
```
### 13. Code Walkthrough
Set iteration removes duplicates; predecessor absence proves a true run start.
### 14. Complexity Analysis
**Time: average `O(n)`** because each distinct value participates in one expansion. **Space: `O(n)`**.
### 15. Edge Cases
- Empty returns `0`. - Duplicates are ignored. - Negative runs work. - One value returns `1`.
### 16. Common Mistakes
- Starting from every value.
- Confusing array adjacency with numeric consecutiveness.
- Calling the nested loops quadratic despite boundary-only expansion.
### 17. Interview Follow-ups
1. Return the run. 2. Tie-break by smallest start. 3. Solve by sorting.
### 18. Variations
- Consecutive ranges. - Arithmetic sequences. - Consecutive dates.
### 19. Pattern Connections
Boundary-only expansion resembles connected-component and interval processing.
### 20. NestJS Project Connection
Expose `POST /api/dsa/hashing/longest-consecutive`; trace run starts and completed lengths.
### 21. Mini Challenge
Return `{ start, end, length }`, choosing the smallest start on ties.

## Problem 29 — Subarray with Sum Zero
### 1. Problem Statement
Return `true` if a non-empty contiguous subarray sums to zero.
### 2. Example
```text
Input: [4,2,-3,1,6]
Output: true
```
### 3. Understand the Problem
The selection must be contiguous; negatives make a positive-only sliding window invalid.

#### In Simple Terms

Return `true` if any continuous block of values adds up to exactly zero.

### 4. Real-World Analogy
Find a continuous accounting period where gains and losses cancel.
### 5. Key Observation
Equal prefix sums mean the values between them sum to zero.
### 6. Pattern
```text
Pattern: Prefix sum plus seen set
```
### 7. Brute Force
Start at every index and extend a running sum.
#### Pseudocode
```text
for each start
    sum = 0
    for each end from start
        add value; return true if sum is 0
```
#### Brute-Force TypeScript

```typescript
function hasZeroSumSubarrayByEnumeration(values: number[]): boolean {
  for (let start = 0; start < values.length; start += 1) {
    let sum = 0;
    for (let end = start; end < values.length; end += 1) {
      sum += values[end];
      if (sum === 0) return true;
    }
  }
  return false;
}
```

#### Visualization
```text
start 1: 2 -> -1 -> 0, found
```
#### Dry Run
Indexes `1..3` contain `2,-3,1`, whose sum is zero.
#### Brute-Force Complexity
Testing `O(n²)` subarrays with incremental sums takes `O(n²)` time and `O(1)` space.
### 8. Why Brute Force Is Not Ideal
Overlapping sums are recomputed from many starts.
### 9. Optimal Approach
Seed seen prefixes with `0`; return true when a newly computed prefix has appeared before.
### 10. Visualization
```text
seen {0}; prefixes 4,6,3,4 -> repeated 4 -> zero-sum segment
```
### 11. Step-by-Step Dry Run
| Index | Prefix | Seen? |
|---:|---:|:---:|
| 0 | 4 | No |
| 1 | 6 | No |
| 2 | 3 | No |
| 3 | 4 | Yes |
### 12. TypeScript Solution
```typescript
function hasZeroSumSubarray(values: number[]): boolean {
  const seen = new Set<number>([0]);
  let prefix = 0;
  for (const value of values) {
    prefix += value;
    if (seen.has(prefix)) return true;
    seen.add(prefix);
  }
  return false;
}
```
### 13. Code Walkthrough
Seed zero represents the prefix before index `0`; checking before adding prevents self-matching.
### 14. Complexity Analysis
**Time: average `O(n)`**; **space: `O(n)`** for distinct prefixes.
### 15. Edge Cases
- Empty returns false. - A zero value returns true. - A zero-sum prefix is found via seeded zero.
### 16. Common Mistakes
- Forgetting seed zero.
- Adding before checking.
- Treating non-contiguous values as a subarray.
### 17. Interview Follow-ups
1. Return indexes. 2. Count zero-sum subarrays. 3. Find the longest.
### 18. Variations
- Sum `k`. - Equal zeroes and ones. - Sum divisible by `k`.
### 19. Pattern Connections
Repeated prefix states represent equal cumulative totals at two boundaries.
### 20. NestJS Project Connection
Use `POST /api/dsa/hashing/zero-sum-subarray`; trace `{ index, prefix, repeated }`.
### 21. Mini Challenge
Store prefix indexes and return the first detected zero-sum interval.

## Problem 30 — Count Subarrays with Sum K
### 1. Problem Statement
Return the number of non-empty contiguous subarrays whose sum equals `k`.
### 2. Example
```text
Input: [1,1,1], k = 2
Output: 2
```
### 3. Understand the Problem
Overlapping subarrays count separately; negative and zero values are allowed.

#### In Simple Terms

Count every continuous block whose values add to `k`. Two blocks count separately even if they overlap.

### 4. Real-World Analogy
Count continuous periods whose net account change equals a target.
### 5. Key Observation
For current prefix `P`, every earlier prefix `P-k` creates one valid ending subarray.
### 6. Pattern
```text
Pattern: Prefix sum frequency map
```
### 7. Brute Force
Extend a running sum from every start and count target matches.
#### Pseudocode
```text
for each start
    sum = 0
    for each end from start
        add value; increment if sum equals k
```
#### Brute-Force TypeScript

```typescript
function countSubarraysWithSumByEnumeration(values: number[], k: number): number {
  let total = 0;
  for (let start = 0; start < values.length; start += 1) {
    let sum = 0;
    for (let end = start; end < values.length; end += 1) {
      sum += values[end];
      if (sum === k) total += 1;
    }
  }
  return total;
}
```

#### Visualization
```text
indexes 0..1 -> 2; indexes 1..2 -> 2
```
#### Dry Run
Exactly two of the six subarrays sum to `2`.
#### Brute-Force Complexity
`O(n²)` subarrays and constant incremental work give `O(n²)` time, `O(1)` space.
### 8. Why Brute Force Is Not Ideal
Overlapping prefix totals are recomputed.
### 9. Optimal Approach
Seed `0 -> 1`; query count of `prefix-k`, add it to the answer, then record current prefix.
### 10. Visualization
```text
P=1 need -1 +0
P=2 need 0  +1
P=3 need 1  +1 -> total 2
```
### 11. Step-by-Step Dry Run
| Prefix | Needed | Matches | Total |
|---:|---:|---:|---:|
| 1 | -1 | 0 | 0 |
| 2 | 0 | 1 | 1 |
| 3 | 1 | 1 | 2 |
### 12. TypeScript Solution
```typescript
function countSubarraysWithSum(values: number[], k: number): number {
  const counts = new Map<number, number>([[0, 1]]);
  let prefix = 0;
  let total = 0;
  for (const value of values) {
    prefix += value;
    total += counts.get(prefix - k) ?? 0;
    counts.set(prefix, (counts.get(prefix) ?? 0) + 1);
  }
  return total;
}
```
### 13. Code Walkthrough
Frequencies count several possible starts. Query-before-update avoids counting an empty subarray when `k=0`.
### 14. Complexity Analysis
**Time: average `O(n)`**; **space: `O(n)`** for prefix frequencies.
### 15. Edge Cases
- Empty returns `0`. - `k=0` uses repeated prefixes. - Negatives work. - Result can be `O(n²)`.
### 16. Common Mistakes
- Storing membership instead of frequency.
- Forgetting `0 -> 1`.
- Updating before querying.
- Using a positive-only sliding window with negatives.
### 17. Interview Follow-ups
1. Return intervals. 2. Find longest sum-`k` subarray. 3. Count sums divisible by `k`.
### 18. Variations
- Binary subarray sum. - Equal zeroes/ones. - Two-dimensional submatrix sums.
### 19. Pattern Connections
Prefix frequency maps generalize complement lookup from values to cumulative states.
### 20. NestJS Project Connection
Expose `POST /api/dsa/hashing/count-subarrays-sum-k`; trace `{ prefix, needed, matchesAdded, total }`.
### 21. Mini Challenge
Specialize the method to count zero-sum subarrays and explain repeated-prefix contributions.

## Problem 31 — Longest Subarray with Sum K
### 1. Problem Statement
Return the maximum length of a contiguous subarray whose sum equals `k`.
### 2. Example
```text
Input: [1,-1,5,-2,3], k = 3
Output: 4  (indexes 0..3)
```
### 3. Understand the Problem
Negative values are allowed. Return `0` when no non-empty matching subarray exists.

#### In Simple Terms

Among all continuous blocks that add to `k`, return the largest number of elements in one block.

### 4. Real-World Analogy
Find the longest continuous reporting period whose net change equals a target.
### 5. Key Observation
At prefix `P` and index `i`, an earlier prefix `P-k` at index `j` gives length `i-j`. Store the earliest index for maximum length.
### 6. Pattern
```text
Pattern: Prefix sum to earliest index
```
### 7. Brute Force
Extend a running sum from every start and track the longest match.
#### Pseudocode
```text
for each start
    sum = 0
    for each end from start
        add value
        if sum equals k, update best length
```
#### Brute-Force TypeScript

```typescript
function longestSubarrayWithSumByEnumeration(values: number[], k: number): number {
  let best = 0;
  for (let start = 0; start < values.length; start += 1) {
    let sum = 0;
    for (let end = start; end < values.length; end += 1) {
      sum += values[end];
      if (sum === k) best = Math.max(best, end - start + 1);
    }
  }
  return best;
}
```

#### Visualization
```text
start 0: sums 1,0,5,3 -> length 4 match
```
#### Dry Run
Indexes `0..3` sum to `3`; no longer range matches.
#### Brute-Force Complexity
`O(n²)` subarrays are tested with incremental sums; auxiliary space is `O(1)`.
### 8. Why Brute Force Is Not Ideal
Overlapping cumulative totals are recomputed.
### 9. Optimal Approach
Seed prefix `0` at index `-1`. Query `prefix-k`; store a prefix index only on its first occurrence.
### 10. Visualization
```text
prefix 1 at 0; prefix 0 at 1; prefix 5 at 2
prefix 3 at 3, need 0 found at -1 -> length 4
```
### 11. Step-by-Step Dry Run
| Index | Prefix | Needed | Earliest | Best |
|---:|---:|---:|---:|---:|
| 0 | 1 | -2 | none | 0 |
| 1 | 0 | -3 | none | 0 |
| 2 | 5 | 2 | none | 0 |
| 3 | 3 | 0 | -1 | 4 |
| 4 | 6 | 3 | 3 | 4 |
### 12. TypeScript Solution
```typescript
function longestSubarrayWithSum(values: number[], k: number): number {
  const firstIndex = new Map<number, number>([[0, -1]]);
  let prefix = 0;
  let best = 0;
  for (let index = 0; index < values.length; index += 1) {
    prefix += values[index];
    const startBoundary = firstIndex.get(prefix - k);
    if (startBoundary !== undefined) best = Math.max(best, index - startBoundary);
    if (!firstIndex.has(prefix)) firstIndex.set(prefix, index);
  }
  return best;
}
```
### 13. Code Walkthrough
The `-1` seed supports prefixes beginning at zero. Keeping the earliest occurrence maximizes every later distance.
### 14. Complexity Analysis
**Time: average `O(n)`** for one traversal and map operations. **Space: `O(n)`** for distinct prefixes.
### 15. Edge Cases
- Empty/no match returns `0`. - Whole array may match. - `k=0` uses repeated prefixes. - Negatives work.
### 16. Common Mistakes
- Overwriting the earliest prefix index.
- Forgetting `0 -> -1`.
- Storing frequencies when indexes are needed.
- Using a positive-only window with negatives.
### 17. Interview Follow-ups
1. Return indexes. 2. Count sum-`k` subarrays. 3. Positive-only constant-space version.
### 18. Variations
- Longest zero-sum subarray. - Equal zeroes/ones. - Longest divisible-by-`k` sum.
### 19. Pattern Connections
The stored map value changes with the question: frequency for counts, earliest index for maximum length.
### 20. NestJS Project Connection
Expose `POST /api/dsa/hashing/longest-subarray-sum-k`; trace `{ index, prefix, needed, earliestIndex, best }`.
### 21. Mini Challenge
Return the earliest longest interval `{ start, end, length }` under ties.

## Problem 32 — Count Pairs with a Given Sum
### 1. Problem Statement
Count unordered index pairs `(i,j)` with `i < j` and `values[i] + values[j] = target`.
### 2. Example
```text
Input: [1,5,7,-1,5], target = 6
Output: 3
```
### 3. Understand the Problem
Equal values at different indexes form different pairs. The three pairs are `(1,5)` twice and `(7,-1)` once.

#### In Simple Terms

Count pairs of different indexes whose values add to the target. Repeated values can create several index pairs.

### 4. Real-World Analogy
Count every distinct pair of receipts whose totals combine to a fixed reimbursement amount.
### 5. Key Observation
When reading `x`, every earlier occurrence of `target-x` creates one new pair ending here.
### 6. Pattern
```text
Pattern: Complement frequency map
```
### 7. Brute Force
Check every index pair once.
#### Pseudocode
```text
for i
    for j after i
        increment if values sum to target
```
#### Brute-Force TypeScript

```typescript
function countPairsWithSumByEnumeration(values: number[], target: number): number {
  let total = 0;
  for (let first = 0; first < values.length - 1; first += 1) {
    for (let second = first + 1; second < values.length; second += 1) {
      if (values[first] + values[second] === target) total += 1;
    }
  }
  return total;
}
```

#### Visualization
```text
(1,5 at index1), (1,5 at index4), (7,-1)
```
#### Dry Run
Testing all ten pairs finds exactly three matches.
#### Brute-Force Complexity
`n(n-1)/2` pairs give `O(n²)` time and `O(1)` auxiliary space.
### 8. Why Brute Force Is Not Ideal
Each current value rescans earlier candidates.
### 9. Optimal Approach
Add the earlier complement frequency to total, then increment the current value frequency.
### 10. Visualization
```text
1 need 5: +0; store 1
5 need 1: +1; store 5
7 need -1: +0
-1 need 7: +1
5 need 1: +1 -> total 3
```
### 11. Step-by-Step Dry Run
| Value | Needed | Earlier frequency | Total |
|---:|---:|---:|---:|
| 1 | 5 | 0 | 0 |
| 5 | 1 | 1 | 1 |
| 7 | -1 | 0 | 1 |
| -1 | 7 | 1 | 2 |
| 5 | 1 | 1 | 3 |
### 12. TypeScript Solution
```typescript
function countPairsWithSum(values: number[], target: number): number {
  const seenCounts = new Map<number, number>();
  let total = 0;
  for (const value of values) {
    total += seenCounts.get(target - value) ?? 0;
    seenCounts.set(value, (seenCounts.get(value) ?? 0) + 1);
  }
  return total;
}
```
### 13. Code Walkthrough
Query-before-update enforces `i < j` and prevents self-use. Frequencies preserve multiplicity.
### 14. Complexity Analysis
**Time: average `O(n)`**; **space: `O(u)`**, worst `O(n)`. Writing the numeric count is `O(1)` even if its value is quadratic.
### 15. Edge Cases
- Fewer than two values returns `0`. - Duplicates count by index. - Negative values work. - Large counts may exceed safe integers.
### 16. Common Mistakes
- Using a set and losing multiplicity.
- Dividing by two despite counting only earlier complements.
- Updating before querying and counting self-pairs.
### 17. Interview Follow-ups
1. Return unique value pairs. 2. Return index pairs. 3. Sorted two-pointer count.
### 18. Variations
- Two Sum. - Pair product target. - Pairs across two arrays.
### 19. Pattern Connections
Frequency-based complements extend Two Sum from existence to combinatorial counting.
### 20. NestJS Project Connection
Use `POST /api/dsa/hashing/count-pairs-sum`; trace `{ value, needed, matchesAdded, total }`.
### 21. Mini Challenge
Return distinct value pairs instead of counting index pairs, and define duplicate handling.

## Problem 33 — Count Pairs with a Given Difference
### 1. Problem Statement
For non-negative `k`, count unordered index pairs `(i,j)`, `i < j`, where `|values[i]-values[j]| = k`.
### 2. Example
```text
Input: [1,5,3,4,2], k = 2
Output: 3  ({1,3}, {5,3}, {4,2})
```
### 3. Understand the Problem
This contract counts index pairs. For `k=0`, equal values pair with one another.

#### In Simple Terms

Count pairs of different indexes whose values are exactly `k` apart, ignoring which value is larger.

### 4. Real-World Analogy
Count pairs of measurements separated by exactly a permitted tolerance.
### 5. Key Observation
For current `x` and `k>0`, earlier partners can be `x-k` or `x+k`. For `k=0`, query `x` only once.
### 6. Pattern
```text
Pattern: Two-complement frequency lookup
```
### 7. Brute Force
Check absolute difference for every pair.
#### Pseudocode
```text
for each i
    for each j after i
        increment if absolute difference equals k
```
#### Brute-Force TypeScript

```typescript
function countPairsWithDifferenceByEnumeration(values: number[], k: number): number {
  if (k < 0) throw new Error("Expected a non-negative difference");
  let total = 0;
  for (let first = 0; first < values.length - 1; first += 1) {
    for (let second = first + 1; second < values.length; second += 1) {
      if (Math.abs(values[first] - values[second]) === k) total += 1;
    }
  }
  return total;
}
```

#### Visualization
```text
|1-3|=2; |5-3|=2; |4-2|=2
```
#### Dry Run
Every unordered pair is checked once; three satisfy the rule.
#### Brute-Force Complexity
`O(n²)` pair comparisons and `O(1)` auxiliary space.
### 8. Why Brute Force Is Not Ideal
Earlier values are rescanned for each current value.
### 9. Optimal Approach
Count matching earlier frequencies, then store current. Handle `k=0` separately to avoid adding the same key twice.
### 10. Visualization
```text
k=2, current 3: earlier 1 and 5 each contribute one
```
### 11. Step-by-Step Dry Run
For `[1,5,3,4,2]`, `3` adds two matches, `4` adds none then `2` finds earlier `4`, totaling three.
### 12. TypeScript Solution
```typescript
function countPairsWithDifference(values: number[], k: number): number {
  if (k < 0) throw new Error("Expected a non-negative difference");
  const seenCounts = new Map<number, number>();
  let total = 0;
  for (const value of values) {
    if (k === 0) total += seenCounts.get(value) ?? 0;
    else {
      total += seenCounts.get(value - k) ?? 0;
      total += seenCounts.get(value + k) ?? 0;
    }
    seenCounts.set(value, (seenCounts.get(value) ?? 0) + 1);
  }
  return total;
}
```
### 13. Code Walkthrough
Only earlier indexes are queried, so each unordered pair is counted once. The zero branch prevents double-counting the identical complement.
### 14. Complexity Analysis
**Time: average `O(n)`** with constant map work per value. **Space: `O(u)`**, worst `O(n)`.
### 15. Edge Cases
- Negative `k` violates the contract. - `k=0` counts combinations of equal values. - Duplicates count by index. - Empty returns zero.
### 16. Common Mistakes
- Counting both pair directions.
- Double-querying the same key for `k=0`.
- Confusing signed difference with absolute difference.
- Using a set and losing multiplicity.
### 17. Interview Follow-ups
1. Count distinct value pairs. 2. Return indexes. 3. Solve sorted input with pointers.
### 18. Variations
- Given sum. - Difference at most `k`. - Pairs across separate arrays.
### 19. Pattern Connections
Multiple possible complements appear whenever a symmetric relation is rewritten around the current value.
### 20. NestJS Project Connection
Expose `POST /api/dsa/hashing/count-pairs-difference`; validate `k >= 0` and return the contract in response metadata.
### 21. Mini Challenge
Change the function to count distinct value pairs and test `k=0` carefully.

## Problem 34 — Find Four Elements with a Given Sum
### 1. Problem Statement
Return indexes `[a,b,c,d]` with `a < b < c < d` whose values sum to `target`, or `null` if none exist.
### 2. Example
```text
Input: [10,2,3,4,5,9,7,8], target = 23
One output: [0,1,2,7]  (10+2+3+8)
```
### 3. Understand the Problem
Indexes must be distinct. Any one valid quadruplet is acceptable; input is not mutated.

#### In Simple Terms

Find four different positions whose values add to the target, and return their indexes in increasing order.

### 4. Real-World Analogy
Combine two earlier two-item bundles with a later two-item bundle to reach a budget.
### 5. Key Observation
Rewrite four-sum as equality between two pair sums while ensuring the stored pair ends before the queried pair starts.
### 6. Pattern
```text
Pattern: Pair-sum hashing with index boundaries
```
### 7. Brute Force
Enumerate every ordered quadruplet of increasing indexes.
#### Pseudocode
```text
for a
  for b after a
    for c after b
      for d after c
        test sum
```
#### Brute-Force TypeScript

```typescript
function fourSumIndexesByEnumeration(
  values: number[],
  target: number,
): [number, number, number, number] | null {
  for (let first = 0; first < values.length - 3; first += 1) {
    for (let second = first + 1; second < values.length - 2; second += 1) {
      for (let third = second + 1; third < values.length - 1; third += 1) {
        for (let fourth = third + 1; fourth < values.length; fourth += 1) {
          if (values[first] + values[second] + values[third] + values[fourth] === target) {
            return [first, second, third, fourth];
          }
        }
      }
    }
  }
  return null;
}
```

#### Visualization
```text
choose 4 of n positions and test each combination
```
#### Dry Run
Eventually indexes `[0,1,2,7]` are tested and sum to `23`.
#### Brute-Force Complexity
There are `O(n⁴)` quadruplets; auxiliary space is `O(1)`.
### 8. Why Brute Force Is Not Ideal
The sums of the same index pairs are recomputed for many quadruplets.
### 9. Optimal Approach
Before choosing current `c`, add all pairs ending at `c-1` to a sum map. Then query every `d>c` for `target-values[c]-values[d]`. Stored pairs are guaranteed disjoint and earlier.
### 10. Visualization
```text
stored pair (a,b) where b < c < d
sum(a,b) = target - value[c] - value[d]
```
### 11. Step-by-Step Dry Run
When `c=2`, store pair `(0,1)` sum `12`. At `d=7`, current pair sum is `11`; needed earlier sum is `12`, so return `[0,1,2,7]`.
### 12. TypeScript Solution
```typescript
function fourSumIndexes(values: number[], target: number): [number, number, number, number] | null {
  const pairBySum = new Map<number, [number, number]>();
  for (let c = 1; c < values.length - 1; c += 1) {
    const b = c - 1;
    for (let a = 0; a < b; a += 1) {
      pairBySum.set(values[a] + values[b], [a, b]);
    }
    for (let d = c + 1; d < values.length; d += 1) {
      const pair = pairBySum.get(target - values[c] - values[d]);
      if (pair) return [pair[0], pair[1], c, d];
    }
  }
  return null;
}
```
### 13. Code Walkthrough
Adding pairs ending at `c-1` before queries makes every stored `b < c`, preventing reused indexes without extra checks.
### 14. Complexity Analysis
**Time: average `O(n²)`** because pair insertions and queries total quadratic work. **Space: `O(n²)`** possible distinct pair sums, though this implementation stores one pair per sum.
### 15. Edge Cases
- Fewer than four values returns `null`. - Duplicates at distinct indexes are valid. - Several answers may exist. - Decimal precision can affect sums.
### 16. Common Mistakes
- Reusing an index across two pairs.
- Storing pairs without enforcing boundary order.
- Returning values when indexes were requested.
- Claiming `O(n)` because map lookup is constant.
### 17. Interview Follow-ups
1. Return unique value quadruplets. 2. Sort and use two pointers. 3. Generalize to `k`-sum.
### 18. Variations
- Three Sum. - Four Sum count across four arrays. - Pair sums closest to target.
### 19. Pattern Connections
Meet-in-the-middle converts larger combinations into joins between smaller computed states.
### 20. NestJS Project Connection
Use `POST /api/dsa/hashing/four-sum`; trace pair-index boundaries and successful complement queries, with strict input-size limits.
### 21. Mini Challenge
Return all unique value quadruplets and explain why one stored pair per sum is no longer sufficient.

## Problem 35 — Group Anagrams
### 1. Problem Statement
Group strings that are anagrams: each group contains words made from the same characters with the same frequencies.
### 2. Example
```text
Input: ["eat","tea","tan","ate","nat","bat"]
Output: [["eat","tea","ate"],["tan","nat"],["bat"]]
```
### 3. Understand the Problem
Character order inside a word does not matter; character multiplicity does. Preserve encounter order within groups and group creation order.

#### In Simple Terms

Put words in the same group when they use exactly the same characters the same number of times, only in a different order.

### 4. Real-World Analogy
Sort letter tiles in each word; words whose sorted tile rows match belong in the same box.
### 5. Key Observation
Sorting a word's characters creates the same canonical key for every anagram in its group.
### 6. Pattern
```text
Pattern: Canonical representation as a hash key
```
### 7. Brute Force
For every ungrouped word, compare its character frequencies with every later word.
#### Pseudocode
```text
for each ungrouped word
    create group
    compare with every later ungrouped word
```
#### Brute-Force TypeScript

```typescript
function areAnagramsByCounting(first: string, second: string): boolean {
  const firstCharacters = [...first];
  const secondCharacters = [...second];
  if (firstCharacters.length !== secondCharacters.length) return false;
  const counts = new Map<string, number>();
  for (const character of firstCharacters) {
    counts.set(character, (counts.get(character) ?? 0) + 1);
  }
  for (const character of secondCharacters) {
    const remaining = counts.get(character);
    if (remaining === undefined) return false;
    if (remaining === 1) counts.delete(character);
    else counts.set(character, remaining - 1);
  }
  return counts.size === 0;
}

function groupAnagramsByPairwiseComparison(words: string[]): string[][] {
  const assigned = Array<boolean>(words.length).fill(false);
  const groups: string[][] = [];
  for (let first = 0; first < words.length; first += 1) {
    if (assigned[first]) continue;
    assigned[first] = true;
    const group = [words[first]];
    for (let second = first + 1; second < words.length; second += 1) {
      if (!assigned[second] && areAnagramsByCounting(words[first], words[second])) {
        assigned[second] = true;
        group.push(words[second]);
      }
    }
    groups.push(group);
  }
  return groups;
}
```

#### Visualization
```text
eat compares tea -> match; tan -> no; ate -> match...
```
#### Dry Run
Repeated pairwise comparisons build three groups.
#### Brute-Force Complexity
For `n` words of maximum length `m`, pairwise frequency comparisons cost `O(n²m)` time; grouping/output tracking uses `O(nm)` space.
### 8. Why Brute Force Is Not Ideal
Words are compared against many others instead of computing one reusable identity.
### 9. Optimal Approach
Sort each word into a key and append the original word to that key's map group.
### 10. Visualization
```text
eat -> aet -> [eat]
tea -> aet -> [eat,tea]
tan -> ant -> [tan]
ate -> aet -> [eat,tea,ate]
```
### 11. Step-by-Step Dry Run
| Word | Key | Group after |
|---|---|---|
| eat | aet | `[eat]` |
| tea | aet | `[eat,tea]` |
| tan | ant | `[tan]` |
| ate | aet | `[eat,tea,ate]` |
| nat | ant | `[tan,nat]` |
| bat | abt | `[bat]` |
### 12. TypeScript Solution
```typescript
function groupAnagrams(words: string[]): string[][] {
  const groups = new Map<string, string[]>();
  for (const word of words) {
    const key = [...word].sort().join("");
    const group = groups.get(key) ?? [];
    group.push(word);
    groups.set(key, group);
  }
  return [...groups.values()];
}
```
### 13. Code Walkthrough
Character sorting supplies structural equality through a primitive string key. Map insertion order preserves group creation order.
### 14. Complexity Analysis
For `n` words with maximum length `m`, **time is `O(n m log m)`** because each key sorts up to `m` characters. **Space is `O(nm)`** for keys and grouped output.
### 15. Edge Cases
- Empty input returns `[]`. - Empty strings share key `""`. - Duplicate words remain duplicate entries. - Case and Unicode policy must be defined.
### 16. Common Mistakes
- Using a set of letters and losing frequencies.
- Assuming `"ab"` and `"aab"` are equivalent.
- Ignoring case normalization requirements.
- Calling key creation `O(1)`.
### 17. Interview Follow-ups
1. Use 26-letter counts for `O(m)` key construction. 2. Case-insensitive grouping. 3. Stream groups incrementally.
### 18. Variations
- Group shifted strings. - Group by normalized object shape. - Find all anagram windows.
### 19. Pattern Connections
Canonical keys support memoization, structural deduplication, and equivalence-class grouping.
### 20. NestJS Project Connection
Expose `POST /api/dsa/hashing/group-anagrams`; return steps `{ word, key, groupSize }` and document normalization rules in the DTO contract.
### 21. Mini Challenge
Replace sorting with a lowercase-English 26-count key and derive the new complexity.

---

# Topic Summary

## What You Learned
- `Set` stores membership; `Map` stores state associated with keys.
- Hash operations are average `O(1)`, while storing up to `n` keys costs `O(n)` space.
- Complement lookup replaces repeated pair searches.
- Frequency maps preserve multiplicity for counting problems.
- Prefix maps turn contiguous-range equations into earlier-state lookups.
- The stored value matters: frequency counts occurrences; earliest index maximizes length.
- Sorted inputs may favor pointers over hashing.
- Canonical keys group items that are equivalent despite different surface order.

## Patterns You Should Recognize
- seen sets for membership and duplicates
- frequency maps for counts and stable qualification
- complement maps for sum and difference relations
- prefix-sum sets for existence
- prefix-sum frequencies for counts
- prefix-sum earliest indexes for maximum lengths
- start-of-run detection for consecutive sequences
- canonical keys for grouping
- pair-sum hashing for meet-in-the-middle problems

## Complexity Cheat Sheet
| Operation or pattern | Time | Space |
|---|---:|---:|
| `Map`/`Set` lookup or update | Average `O(1)` | Per stored key |
| Build membership/frequencies | Average `O(n)` | `O(u)` |
| Complement lookup traversal | Average `O(n)` | `O(n)` worst case |
| Prefix-map traversal | Average `O(n)` | `O(n)` |
| Three sorted-array pointers | `O(n+m+p)` | `O(1)` auxiliary |
| Pair-sum hashing | Average `O(n²)` | `O(n²)` |
| Sorted-character anagram keys | `O(n m log m)` | `O(nm)` |

## Common Interview Traps
- Calling hash operations guaranteed `O(1)` instead of average `O(1)`.
- Using truthiness for a mapped index when index `0` is valid.
- Using a `Set` when frequencies are required.
- Updating current state before querying and counting an item against itself.
- Forgetting the seeded prefix `0` state.
- Overwriting an earliest prefix index in a longest-length problem.
- Confusing first duplicate discovered with repeating value having earliest first occurrence.
- Ignoring key-construction and output costs.
- Reusing indexes in pair-sum combinations.

## Practice Checklist
- [ ] Problem 21 — Two Sum
- [ ] Problem 22 — Contains Duplicate
- [ ] Problem 23 — Count frequencies of array elements
- [ ] Problem 24 — Find the first repeating element
- [ ] Problem 25 — Find the first non-repeating element
- [ ] Problem 26 — Find duplicates in an array
- [ ] Problem 27 — Find common elements in three sorted arrays
- [ ] Problem 28 — Longest consecutive sequence
- [ ] Problem 29 — Subarray with sum zero
- [ ] Problem 30 — Count subarrays with sum K
- [ ] Problem 31 — Longest subarray with sum K
- [ ] Problem 32 — Count pairs with a given sum
- [ ] Problem 33 — Count pairs with a given difference
- [ ] Problem 34 — Find four elements with a given sum
- [ ] Problem 35 — Group Anagrams