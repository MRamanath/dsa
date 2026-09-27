# Two Pointers

## Learning Objectives

By the end of this topic, you should understand:

- how two pointers replace repeated range scans with coordinated movement
- how sorted order makes pointer decisions logically safe
- the difference between opposite-direction, same-direction, and partition pointers
- how to state and maintain a pointer invariant
- when a two-pointer algorithm mutates input and when it builds new output
- how duplicate skipping prevents repeated combinations
- how three-way partitioning extends the pattern beyond two pointers
- why pointer movement often gives linear work even when pointers are nested syntactically
- how to emit meaningful pointer snapshots for a NestJS visualization API

## Prerequisites

- contracts, invariants, and complexity reasoning from `00-dsa-fundamentals.md`
- array traversal, mutation, and rotation basics from `01-arrays.md`
- frequency and complement tradeoffs from `02-hashing.md`
- string code-point contracts from `03-strings.md`
- TypeScript arrays, strings, loops, functions, and tuple types

## Core Theory

### What are two pointers?

**Two pointers** are indexes or references that coordinate while traversing one or more sequences. Their movement discards possibilities, preserves useful state, or places values into final positions.

```text
values: [1, 2, 4, 7, 11]
         ^            ^
        left         right
```

The technique is not merely "using two variables." It depends on a rule that explains why moving one pointer cannot discard a valid answer.

### Why does the pattern exist?

A brute-force solution often checks every pair:

```text
first choice:  n possibilities
second choice: n possibilities
total:         O(n^2)
```

If order or an invariant tells us which choices are impossible, one pointer movement can eliminate an entire group of pairs. When each pointer only moves forward or inward, total movement is usually linear.

### Pointer invariants

An **invariant** is a statement that remains true before and after each iteration. Examples include:

- every index before `write` contains a finalized unique value
- every index before `slow` contains a finalized nonzero value
- no pair outside `[left, right]` can meet the target
- every value before `low` is `0`, and every value after `high` is `2`

Writing the invariant first makes pointer updates easier to justify and test.

### Opposite-direction pointers

Pointers begin at both ends and move inward:

```text
[1, 2, 4, 7, 11]
 ^            ^
left         right
```

Use this form when:

- sorted order lets a sum be increased or decreased predictably
- mirrored positions must be compared or swapped
- a width shrinks while boundary values determine a score

### Same-direction pointers

Both pointers move left to right, but they have different jobs:

```text
[0, 1, 0, 3, 12]
 ^  ^
write scan
```

The scan pointer reads every value. The write pointer marks the next finalized position. This is useful for filtering, compacting, and stable in-place transformations.

### Partition pointers

Some problems maintain several regions rather than one shrinking range:

```text
[ 0 region | unknown region | 2 region ]
             ^           ^
            current     high
```

The Dutch National Flag algorithm uses `low`, `current`, and `high`. It belongs to the same family because monotonic boundaries classify every item once.

### Sorted order is a resource

For a sorted array and current sum `values[left] + values[right]`:

- if the sum is too small, moving `right` left cannot increase it, so advance `left`
- if the sum is too large, moving `left` right cannot decrease it, so retreat `right`

Without sorted order, those deductions are invalid. Sorting first changes both the complexity and, unless original indexes are preserved, the output contract.

### Mutation and ownership

In-place pointer algorithms overwrite or swap array entries. Their contract should say whether callers permit mutation.

```typescript
function moveZeroes(values: number[]): void {
  // The caller's array is intentionally mutated.
}
```

If mutation is not allowed, clone first and count the clone as `O(n)` space.

### Duplicate control

Combination problems such as 3Sum and 4Sum usually require unique value combinations. Sorted order groups duplicates, allowing the algorithm to:

- skip a repeated anchor
- skip repeated left/right values after recording a solution
- retain index distinctness while removing duplicate output tuples

Skipping duplicates too early can miss answers; skipping none can repeat answers.

## Visual Mental Model

### Converging pointers

```text
target = 9

[1, 2, 4, 7, 11]
 L           R         1 + 11 is too large -> R moves
[1, 2, 4, 7, 11]
 L        R            1 + 7 is too small -> L moves
[1, 2, 4, 7, 11]
    L     R             2 + 7 = 9
```

### Read and write pointers

```text
input: [1, 1, 2, 2, 3]

read discovers new value 2
write places it after finalized [1]

[1, 2 | ?, ?, ?]
       ^
      write
```

### Mirrored pointers

```text
"r a c e c a r"
  ^           ^   compare, then move inward
```

### Area boundaries

```text
height
  |        |
  |  |     |
--L--------R--

area = min(left height, right height) x width
```

The shorter boundary limits area, so only moving that boundary can possibly improve the limiting height.

### Three-way partition

```text
[ zeros | ones/unknown | twos ]
          ^          ^
        current     high
```

Each inspection grows a finalized region or shrinks the unknown region.

## Complexity Concepts

Let `n` be the input length.

| Pointer behavior | Typical time | Extra space | Reason |
|---|---:|---:|---|
| One inward pass | `O(n)` | `O(1)` | Each boundary moves at most `n` times. |
| Read/write compaction | `O(n)` | `O(1)` | Every value is read once and written at most once. |
| Sort then two pointers | `O(n log n)` | Depends on sort | Sorting dominates a linear sweep. |
| One anchor plus two pointers | `O(n²)` | Depends on sort/output | `n` anchors each perform a linear sweep. |
| Two anchors plus two pointers | `O(n³)` | Depends on sort/output | `O(n²)` anchor pairs each perform a linear sweep. |

### Why nested movement can still be linear

In a read/write loop, `write` may change inside a loop over `read`, but neither pointer resets. Count total pointer movement, not indentation depth.

```text
read:  0 -> 1 -> 2 -> ... -> n
write: 0 -> 1 -> 2 -> ... -> at most n
```

The total is `O(n + n) = O(n)`.

### Sorting changes the full complexity

A linear two-pointer sweep after sorting is not an `O(n)` algorithm overall:

```text
sort:  O(n log n)
sweep: O(n)
total: O(n log n)
```

Also state whether sorting mutates the caller's array and what auxiliary space the runtime sort uses.

### Output space versus auxiliary space

3Sum and 4Sum can return many tuples. Distinguish:

- auxiliary working space, excluding the result
- output space, required to store returned combinations

An algorithm may use `O(1)` pointer state while still returning substantial output.

### Numeric safety in TypeScript

JavaScript `number` is exact only for integers up to `Number.MAX_SAFE_INTEGER`. Summing large integers can lose precision. Production contracts should restrict inputs to safe integer ranges or use `bigint` consistently.

## Common Patterns

### Opposite ends on sorted data

**Pattern:** Move `left` or `right` according to how the current ordered pair compares with a target.

**Recognize it when:** input is sorted, a pair is requested, and increasing one side or decreasing the other changes the score predictably.

**Similar problems:** sorted Two Sum, pair difference, closest pair sum.

### Read/write compaction

**Pattern:** One pointer scans while another marks the next valid output position.

**Recognize it when:** filtering or deduplicating in place while preserving encounter order.

**Similar problems:** remove duplicates, move zeroes, remove a chosen value.

### Mirrored comparison or swapping

**Pattern:** Process symmetric positions from the outside inward.

**Recognize it when:** reversing or validating a palindrome.

**Similar problems:** reverse string, valid palindrome, reverse array.

### Anchor plus converging pointers

**Pattern:** Fix one or more sorted anchors, then solve a reduced pair problem with two pointers.

**Recognize it when:** unique triplets or quadruplets must meet a target.

**Similar problems:** 3Sum, 4Sum, 3Sum Closest.

### Boundary optimization

**Pattern:** Evaluate a score from two boundaries and move the boundary that limits improvement.

**Recognize it when:** width shrinks but a weaker boundary can potentially be replaced by a stronger one.

**Similar problems:** Container With Most Water and Trapping Rain Water.

### Multi-region partition

**Pattern:** Maintain finalized regions around an unknown interval.

**Recognize it when:** a small fixed set of categories must be arranged in one pass.

**Similar problems:** Sort Colors and three-way quicksort partitioning.

---

# Problems

## Problem 51 — Two Sum in a Sorted Array

### 1. Problem Statement
Given an ascending array of safe integers and a target, return the zero-based indexes of two distinct values whose sum equals the target, or `null` if no pair exists.

### 2. Example
```text
Input: values = [1, 2, 4, 7, 11], target = 9
Output: [1, 3]
```

### 3. Understand the Problem
The input is already sorted and is not mutated. Any one valid pair is acceptable; indexes must refer to the original sorted array.

#### In Simple Terms

Find two different positions in the sorted array whose values add to the target, and return their indexes.

### 4. Real-World Analogy
Choose two priced items from an ordered shelf whose prices exactly spend a gift card balance.

### 5. Key Observation
If the smallest and largest remaining values sum too low, only a larger left value can help. If they sum too high, only a smaller right value can help.

### 6. Pattern
```text
Pattern: Opposite-direction pointers on sorted data
Why this pattern applies: sorted order makes each pointer decision safe.
How to recognize it: find a pair in sorted input with a target sum.
Similar problems: pair closest to target, pair with a given difference.
```

### 7. Brute Force
Try every index pair and return the first sum equal to the target.

#### Pseudocode
```text
for left from 0 to n - 2
    for right from left + 1 to n - 1
        if values[left] + values[right] equals target
            return [left, right]
return null
```

#### Brute-Force TypeScript

```typescript
function twoSumSortedByCheckingPairs(values: number[], target: number): [number, number] | null {
  for (let left = 0; left < values.length - 1; left += 1) {
    for (let right = left + 1; right < values.length; right += 1) {
      if (values[left] + values[right] === target) return [left, right];
    }
  }
  return null;
}
```

#### Visualization
```text
[1,2] [1,4] [1,7] ... [2,7] -> 9
```

#### Dry Run
For `[1,2,4,7,11]`, nested loops test pairs beginning with `1`, then find `2 + 7 = 9`.

#### Brute-Force Complexity
**Time: `O(n²)`** because up to `n(n-1)/2` pairs are tested. **Space: `O(1)`** besides the returned pair.

### 8. Why Brute Force Is Not Ideal
It ignores sorted order and tests many pairs that one comparison could eliminate together.

### 9. Optimal Approach
Start at both ends. Return on equality; advance `left` when the sum is too small, otherwise retreat `right`.

### 10. Visualization
```text
[1, 2, 4, 7, 11] target 9
 L            R    12 > 9 -> R--
 L         R        8 < 9 -> L++
    L      R        9 = 9 -> [1,3]
```

### 11. Step-by-Step Dry Run
| Left | Right | Sum | Action |
|---:|---:|---:|---|
| 0 | 4 | 12 | move right |
| 0 | 3 | 8 | move left |
| 1 | 3 | 9 | return `[1,3]` |

### 12. TypeScript Solution
```typescript
function twoSumSorted(values: number[], target: number): [number, number] | null {
  let left = 0;
  let right = values.length - 1;
  while (left < right) {
    const sum = values[left] + values[right];
    if (sum === target) return [left, right];
    if (sum < target) left += 1;
    else right -= 1;
  }
  return null;
}
```

### 13. Code Walkthrough
`left < right` enforces distinct indexes. Each comparison removes all pairs involving one boundary that cannot reach the target.

### 14. Complexity Analysis
**Time: `O(n)`** because each pointer moves inward at most `n` positions. **Space: `O(1)`** besides the constant-size result.

### 15. Edge Cases
- Fewer than two values returns `null`.
- Duplicates can form a pair at distinct indexes.
- Negative values work because ordering, not positivity, drives movement.
- Inputs must stay within safe integer sum ranges.

### 16. Common Mistakes
- Applying the movement rule to unsorted input.
- Allowing `left === right` to reuse one element.
- Returning values when indexes were requested.
- Using one-based indexes without stating it.

### 17. Interview Follow-ups
1. Return one-based indexes. 2. Return all unique value pairs. 3. Find the closest sum. 4. Preserve indexes if input is unsorted.

### 18. Variations
- Pair with target difference. - Count target-sum pairs. - Two Sum across two sorted arrays.

### 19. Pattern Connections
The same ordered elimination powers 3Sum, 4Sum, and binary-search-style decision making.

### 20. NestJS Project Connection
Expose `POST /api/dsa/two-pointers/two-sum-sorted` with `{ values, target }`. Return indexes plus trace steps `{ left, right, sum, action }` after validating sorted safe-integer input.

### 21. Mini Challenge
Return all unique value pairs summing to the target without duplicate pairs.

## Problem 52 — Remove Duplicates Using Two Pointers

### 1. Problem Statement
Given an ascending number array, remove duplicates in place and return the number `k` of unique values; the first `k` positions must contain those values in order.

### 2. Example
```text
Input: [1, 1, 2, 2, 3]
Output: k = 3, first k values = [1, 2, 3]
```

### 3. Understand the Problem
Only the prefix `values.slice(0, k)` is meaningful afterward. Values beyond `k` may remain unchanged, and allocating another `O(n)` collection is not allowed in the optimal solution.

#### In Simple Terms

Keep one copy of each value at the front of the sorted array and return how many unique values remain.

### 4. Real-World Analogy
Scan sorted tickets and move one copy of each new ticket number into the front display slots.

### 5. Key Observation
Sorted duplicates are adjacent. A value differs from the last finalized unique value exactly when it should be written next.

### 6. Pattern
```text
Pattern: Same-direction read/write pointers
Why this pattern applies: read discovers new values while write owns the compact prefix.
How to recognize it: stable in-place filtering or deduplication is requested.
Similar problems: remove element, move zeroes, compact sorted records.
```

### 7. Brute Force
Build a separate array, appending a value when it differs from the previous appended value, then copy it back.

#### Pseudocode
```text
unique = empty array
for each value
    if unique is empty or unique last differs
        append value
copy unique into input prefix
return unique length
```

#### Brute-Force TypeScript

```typescript
function removeSortedDuplicatesWithBuffer(values: number[]): number {
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
[1,1,2,2,3] -> new [1,2,3] -> copy to input
```

#### Dry Run
The auxiliary array grows from `[1]` to `[1,2]` to `[1,2,3]`, then overwrites the first three input positions.

#### Brute-Force Complexity
**Time: `O(n)`** for scanning and copying. **Space: `O(n)`** for the auxiliary unique array.

### 8. Why Brute Force Is Not Ideal
It meets the value requirement but violates the constant-extra-space goal.

### 9. Optimal Approach
Treat index `0` as the first finalized unique value. Scan from index `1`; when a new value appears, write it at `write` and advance `write`.

### 10. Visualization
```text
[1, 1, 2, 2, 3]
 ^     ^
last  read=2 -> write 2 at index 1

[1, 2 | 2, 2, 3]
       ^
      write
```

### 11. Step-by-Step Dry Run
| Read index | Value | Last unique | Action | `write` |
|---:|---:|---:|---|---:|
| 1 | 1 | 1 | skip | 1 |
| 2 | 2 | 1 | write at 1 | 2 |
| 3 | 2 | 2 | skip | 2 |
| 4 | 3 | 2 | write at 2 | 3 |

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
Before every iteration, indexes below `write` are the finalized unique prefix. Comparing with `values[write - 1]` preserves that invariant.

### 14. Complexity Analysis
**Time: `O(n)`** because `read` visits each value once. **Extra space: `O(1)`** because the result reuses the input array.

### 15. Edge Cases
- Empty returns `0`.
- One value returns `1`.
- All duplicates produce `k = 1`.
- All unique values produce `k = n`.

### 16. Common Mistakes
- Forgetting that input must be sorted.
- Returning the array instead of `k` under this contract.
- Reading unspecified values beyond the first `k` positions.
- Starting `write` at zero for nonempty input.

### 17. Interview Follow-ups
1. Allow each value at most twice. 2. Remove a chosen value. 3. Return a new array without mutation. 4. Deduplicate unsorted input.

### 18. Variations
- Keep at most `m` copies. - Compact records by key. - Deduplicate two merged sorted streams.

### 19. Pattern Connections
Read/write compaction also powers stable filtering and in-place merging techniques.

### 20. NestJS Project Connection
Use `POST /api/dsa/two-pointers/remove-duplicates` with `{ values }`. Clone request data before demonstrating mutation and return `{ length, prefix, steps }`, where each step records `read`, `write`, and action.

### 21. Mini Challenge
Modify the invariant so each distinct value may appear at most twice.

## Problem 53 — Move Zeroes Using Two Pointers

### 1. Problem Statement
Move every zero to the end of a number array in place while preserving the relative order of nonzero values.

### 2. Example
```text
Input: [0, 1, 0, 3, 12]
Output: [1, 3, 12, 0, 0]
```

### 3. Understand the Problem
The operation is stable for nonzero values, mutates the array, and returns no separate result.

#### In Simple Terms

Move all zeroes to the end without changing the order of the nonzero values.

### 4. Real-World Analogy
Move filled folders to the front without changing their order, then leave all empty placeholders at the back.

### 5. Key Observation
Write every nonzero value into the next front position, then fill the remaining suffix with zeroes.

### 6. Pattern
```text
Pattern: Same-direction read/write pointers
Why this pattern applies: one pointer reads all values while another builds the stable nonzero prefix.
How to recognize it: stable in-place partition around a removable value.
Similar problems: remove element, compact nulls, remove sorted duplicates.
```

### 7. Brute Force
Collect all nonzero values in a new array, append the required number of zeroes, and copy the result back.

#### Pseudocode
```text
result = all nonzero values
while result length is below input length
    append 0
copy result into input
```

#### Brute-Force TypeScript

```typescript
function moveZeroesWithBuffer(values: number[]): void {
  const result = values.filter((value) => value !== 0);
  while (result.length < values.length) result.push(0);
  for (let index = 0; index < values.length; index += 1) values[index] = result[index];
}
```

#### Visualization
```text
[0,1,0,3,12] -> [1,3,12] + [0,0] -> copy
```

#### Dry Run
The auxiliary result receives `1`, `3`, and `12`, then two zeroes.

#### Brute-Force Complexity
**Time: `O(n)`** for collection, filling, and copying. **Space: `O(n)`** for the result array.

### 8. Why Brute Force Is Not Ideal
It allocates storage proportional to input even though the caller permits in-place mutation.

### 9. Optimal Approach
Use `write` for the next nonzero position. After one stable pass, overwrite positions from `write` onward with zero.

### 10. Visualization
```text
read:  0 1 2 3 4
value: 0 1 0 3 12
write:   1   3 12 -> finalized prefix
fill:  [1,3,12,0,0]
```

### 11. Step-by-Step Dry Run
| Read | Value | Action | Write after | Prefix |
|---:|---:|---|---:|---|
| 0 | 0 | skip | 0 | `[]` |
| 1 | 1 | write | 1 | `[1]` |
| 2 | 0 | skip | 1 | `[1]` |
| 3 | 3 | write | 2 | `[1,3]` |
| 4 | 12 | write | 3 | `[1,3,12]` |

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
The first pass preserves nonzero encounter order in `[0, write)`. The fill loop finalizes exactly the remaining number of positions.

### 14. Complexity Analysis
**Time: `O(n)`** because each position is read once and written at most once during suffix filling, with at most `2n` total work. **Extra space: `O(1)`**.

### 15. Edge Cases
- Empty and all-zero arrays remain valid.
- An array with no zeroes is unchanged.
- Negative values are nonzero and retain order.
- `-0` compares equal to `0` in JavaScript and moves to the suffix as zero.

### 16. Common Mistakes
- Sorting, which loses nonzero order.
- Splicing while iterating and skipping positions.
- Swapping every nonzero unnecessarily without stating stability.
- Forgetting to fill the suffix.

### 17. Interview Follow-ups
1. Minimize writes. 2. Return the zero count. 3. Move zeroes to the front. 4. Generalize to a chosen sentinel.

### 18. Variations
- Move nulls. - Stable binary partition. - Remove a target value and return new length.

### 19. Pattern Connections
The finalized-prefix invariant is the same core idea as sorted deduplication.

### 20. NestJS Project Connection
Expose `POST /api/dsa/two-pointers/move-zeroes`. Clone the DTO array, mutate the clone, and return snapshots `{ read, write, values, phase }` with an input-size cap.

### 21. Mini Challenge
Perform the same stable transformation while avoiding writes when a nonzero is already in its final position.

## Problem 54 — Reverse a String Using Two Pointers

### 1. Problem Statement
Return a new string with its Unicode code points in reverse order by swapping a mutable code-point array from both ends.

### 2. Example
```text
Input: "hello"
Output: "olleh"
```

### 3. Understand the Problem
JavaScript strings are immutable. This contract reverses code points, not grapheme clusters, so some combined visible symbols may separate.

#### In Simple Terms

Return the string's code points in reverse order, leaving the original immutable string unchanged.

### 4. Real-World Analogy
Swap cards at opposite ends of a row, then step inward until all cards have exchanged mirrored positions.

### 5. Key Observation
One swap finalizes both outer positions, so no position needs to be revisited.

### 6. Pattern
```text
Pattern: Mirrored opposite-direction pointers
Why this pattern applies: reversal maps each position to its symmetric partner.
How to recognize it: symmetric swapping or comparison from sequence ends.
Similar problems: reverse array, palindrome check, reverse a range.
```

### 7. Brute Force
Traverse from the end and repeatedly append each symbol to an immutable result string.

#### Pseudocode
```text
result = empty string
for index from last down to first
    result = result + character[index]
return result
```

#### Brute-Force TypeScript

```typescript
function reverseStringByPrepending(text: string): string {
  let reversed = "";
  for (const character of text) reversed = character + reversed;
  return reversed;
}
```

#### Visualization
```text
"h" -> "o" -> "ol" -> "oll" -> "olle" -> "olleh"
```

#### Dry Run
Reading `hello` backward appends `o`, `l`, `l`, `e`, then `h`.

#### Brute-Force Complexity
Traversal is `O(n)`, but repeated immutable concatenation can copy a growing result for `O(n²)` time. Output space is `O(n)`.

### 8. Why Brute Force Is Not Ideal
It can repeatedly copy partial output and does not demonstrate symmetric pointer movement.

### 9. Optimal Approach
Spread into a code-point array, swap left/right positions while `left < right`, then join once.

### 10. Visualization
```text
[h, e, l, l, o]
 L           R  swap
[o, e, l, l, h]
    L     R     swap
[o, l, l, e, h]
       ^        done
```

### 11. Step-by-Step Dry Run
Swap indexes `0` and `4`, producing `oellh`; swap indexes `1` and `3`, producing `olleh`; pointers meet at index `2`.

### 12. TypeScript Solution
```typescript
function reverseWithTwoPointers(text: string): string {
  const characters = [...text];
  let left = 0;
  let right = characters.length - 1;
  while (left < right) {
    [characters[left], characters[right]] = [characters[right], characters[left]];
    left += 1;
    right -= 1;
  }
  return characters.join("");
}
```

### 13. Code Walkthrough
The spread makes code points mutable. Every swap expands finalized reversed regions at both ends; `join` constructs the required immutable output.

### 14. Complexity Analysis
**Time: `O(n)`** for spreading, at most `n/2` swaps, and joining. **Space: `O(n)`** for the code-point array and returned string; pointer state is `O(1)`.

### 15. Edge Cases
- Empty and one-code-point strings return unchanged.
- A single-code-point emoji stays intact.
- Combining marks and joined emoji require grapheme segmentation for visual reversal.

### 16. Common Mistakes
- Claiming a JavaScript string was reversed in place.
- Using UTF-16 indexes while promising code-point behavior.
- Forgetting output allocation in space complexity.
- Swapping while `left <= right` unnecessarily.

### 17. Interview Follow-ups
1. Reverse a mutable character array in place. 2. Reverse grapheme clusters. 3. Reverse only letters. 4. Reverse each word independently.

### 18. Variations
- Reverse vowels. - Reverse a subrange. - Reverse word order.

### 19. Pattern Connections
Mirrored movement also supports palindrome checks and symmetric array transformations.

### 20. NestJS Project Connection
Use `POST /api/dsa/two-pointers/reverse-string` with `{ text }`. Return `{ result, steps }`, where each step contains pointer indexes and the post-swap character array; cap trace length.

### 21. Mini Challenge
Use `Intl.Segmenter` to reverse user-perceived grapheme clusters instead of code points.

## Problem 55 — Valid Palindrome

### 1. Problem Statement
Return whether a string is a palindrome after ignoring non-ASCII-alphanumeric characters and letter case.

### 2. Example
```text
Input: "A man, a plan, a canal: Panama"
Output: true
```

### 3. Understand the Problem
This chapter's contract accepts ASCII letters and digits as comparable symbols. Punctuation and whitespace are skipped; letters compare case-insensitively.

#### In Simple Terms

Ignore punctuation and spaces, ignore ASCII letter case, and check whether the remaining letters and digits read the same both ways.

### 4. Real-World Analogy
Two readers start at opposite ends of a decorated banner, ignore separators, and compare the next meaningful symbol each sees.

### 5. Key Observation
There is no need to construct the fully normalized string: each pointer can skip ignored code units before comparing.

### 6. Pattern
```text
Pattern: Mirrored pointers with filtering
Why this pattern applies: only symmetric meaningful symbols determine validity.
How to recognize it: compare a sequence under an ignore/normalization policy.
Similar problems: reverse only letters, almost-palindrome, palindrome range.
```

### 7. Brute Force
Build a normalized lowercase string containing only ASCII letters and digits, reverse it, and compare the two strings.

#### Pseudocode
```text
normalized = filter and lowercase text
reversed = reverse normalized
return normalized equals reversed
```

#### Brute-Force TypeScript

```typescript
function isValidPalindromeByCleaning(text: string): boolean {
  const normalized = [...text]
    .filter((character) => /[A-Za-z0-9]/.test(character))
    .map((character) => character.toLowerCase())
    .join("");
  return [...normalized].reverse().join("") === normalized;
}
```

#### Visualization
```text
"A man..." -> "amanaplanacanalpanama" -> reverse -> equal
```

#### Dry Run
Filtering removes spaces, comma, and colon; lowercasing makes both endpoint `A/a` symbols equal.

#### Brute-Force Complexity
**Time: `O(n)`** with array accumulation and one join/reverse. **Space: `O(n)`** for normalized and reversed representations.

### 8. Why Brute Force Is Not Ideal
It stores complete normalized copies even though comparisons can happen directly from the ends.

### 9. Optimal Approach
Move each pointer past ignored code units. Compare lowercase meaningful symbols; fail on mismatch and otherwise move inward.

### 10. Visualization
```text
"A man, a plan, a canal: Panama"
  L                              R
  a == a -> move inward, skipping spaces/punctuation
```

### 11. Step-by-Step Dry Run
The pairs normalize to `a/a`, `m/m`, `a/a`, `n/n`, and continue symmetrically. Every punctuation position is skipped, and the pointers cross without a mismatch.

### 12. TypeScript Solution
```typescript
function isAsciiAlphaNumeric(character: string): boolean {
  return /[A-Za-z0-9]/.test(character);
}

function isValidPalindrome(text: string): boolean {
  let left = 0;
  let right = text.length - 1;
  while (left < right) {
    while (left < right && !isAsciiAlphaNumeric(text[left])) left += 1;
    while (left < right && !isAsciiAlphaNumeric(text[right])) right -= 1;
    if (text[left].toLowerCase() !== text[right].toLowerCase()) return false;
    left += 1;
    right -= 1;
  }
  return true;
}
```

### 13. Code Walkthrough
The inner loops preserve the invariant that compared endpoints are meaningful under the ASCII contract. Each code unit is skipped or compared at most once.

### 14. Complexity Analysis
**Time: `O(n)`** because both pointers move only inward across at most `n` code units. **Extra space: `O(1)`**; lowercase endpoint strings have constant bounded size under the ASCII contract.

### 15. Edge Cases
- Empty or punctuation-only text returns true.
- Digits participate in comparison.
- Case variants compare equal.
- Non-ASCII letters are ignored by this explicit contract.

### 16. Common Mistakes
- Leaving the normalization policy unspecified.
- Using `\w`, which also accepts underscore under JavaScript regex rules.
- Building normalized copies while claiming constant space.
- Comparing punctuation before skipping it.

### 17. Interview Follow-ups
1. Support Unicode letters and numbers. 2. Allow deleting one symbol. 3. Return the first mismatch. 4. Check a specified range.

### 18. Variations
- Valid Palindrome II. - Palindrome sentence with locale rules. - Alphanumeric mirrored token check.

### 19. Pattern Connections
Filtered mirrored pointers combine string normalization with the same inward invariant used for reversal.

### 20. NestJS Project Connection
Expose `POST /api/dsa/two-pointers/valid-palindrome` with `{ text }`. Return comparisons and skip actions as bounded trace events without logging sensitive full text by default.

### 21. Mini Challenge
Return whether the text can become valid after deleting at most one comparable character.

## Problem 56 — Container With Most Water

### 1. Problem Statement
Given nonnegative integer heights at unit-spaced positions, return the maximum water area formed by any two vertical lines.

### 2. Example
```text
Input: [1, 8, 6, 2, 5, 4, 8, 3, 7]
Output: 49
```

### 3. Understand the Problem
For indexes `left < right`, area is `(right - left) * min(height[left], height[right])`. Lines cannot tilt, and input values must yield safe-integer areas.

#### In Simple Terms

Choose two lines that hold the most water between them. The shorter line limits the water height.

### 4. Real-World Analogy
Choose two fence posts to hold water; the shorter post sets the water level while their distance sets the width.

### 5. Key Observation
Moving the taller boundary inward reduces width while the shorter boundary still limits height, so it cannot improve area. Move a shortest boundary.

### 6. Pattern
```text
Pattern: Boundary optimization with opposite pointers
Why this pattern applies: the shorter boundary limits the score.
How to recognize it: a score combines shrinking width with two endpoint values.
Similar problems: trapping rain water, widest valid pair, maximum distance constraints.
```

### 7. Brute Force
Calculate area for every pair of lines and retain the maximum.

#### Pseudocode
```text
best = 0
for left from 0 to n - 2
    for right from left + 1 to n - 1
        best = max(best, width * shorter height)
return best
```

#### Brute-Force TypeScript

```typescript
function maxContainerAreaByCheckingPairs(heights: number[]): number {
  let best = 0;
  for (let left = 0; left < heights.length - 1; left += 1) {
    for (let right = left + 1; right < heights.length; right += 1) {
      const area = (right - left) * Math.min(heights[left], heights[right]);
      best = Math.max(best, area);
    }
  }
  return best;
}
```

#### Visualization
```text
pair (0,1), (0,2), ... then (1,2), (1,3), ...
```

#### Dry Run
For the example, pair indexes `1` and `8` has width `7`, limiting height `7`, and area `49`.

#### Brute-Force Complexity
**Time: `O(n²)`** because every pair is evaluated. **Space: `O(1)`**.

### 8. Why Brute Force Is Not Ideal
It recomputes areas for pairs that can be ruled out once a limiting boundary is known.

### 9. Optimal Approach
Start with maximum width. Record the current area, then move the shorter line inward because only replacing it with a taller line can offset lost width.

### 10. Visualization
```text
[1,8,6,2,5,4,8,3,7]
 L               R   area = 8; left shorter -> L++
   L             R   area = 49; right shorter -> R--
```

### 11. Step-by-Step Dry Run
| Left | Right | Heights | Area | Best | Move |
|---:|---:|---|---:|---:|---|
| 0 | 8 | 1, 7 | 8 | 8 | left |
| 1 | 8 | 8, 7 | 49 | 49 | right |
| 1 | 7 | 8, 3 | 18 | 49 | right |
| 1 | 6 | 8, 8 | 40 | 49 | either |

### 12. TypeScript Solution
```typescript
function maxContainerArea(heights: number[]): number {
  let left = 0;
  let right = heights.length - 1;
  let best = 0;
  while (left < right) {
    const limitingHeight = Math.min(heights[left], heights[right]);
    best = Math.max(best, (right - left) * limitingHeight);
    if (heights[left] <= heights[right]) left += 1;
    else right -= 1;
  }
  return best;
}
```

### 13. Code Walkthrough
The first pair has maximum width. Each step evaluates that width once and discards a boundary that cannot help any narrower pair unless its limiting height changes.

### 14. Complexity Analysis
**Time: `O(n)`** because one pointer moves inward each iteration. **Space: `O(1)`** because only indexes and scalar values are stored.

### 15. Edge Cases
- Fewer than two lines returns `0`.
- Zero heights contribute zero area.
- Equal boundaries may move either side safely.
- Negative heights are rejected by the contract.

### 16. Common Mistakes
- Moving the taller line.
- Using maximum height instead of minimum.
- Forgetting width is the index difference.
- Confusing this with trapped water between many bars.

### 17. Interview Follow-ups
1. Return the chosen indexes. 2. Handle non-unit x-coordinates. 3. Explain why moving the shorter side is safe. 4. Find all maximum pairs.

### 18. Variations
- Maximum area with coordinates. - Maximum width ramp. - Closest score under a width constraint.

### 19. Pattern Connections
Unlike target-sum elimination, this proof moves the endpoint that limits an objective function.

### 20. NestJS Project Connection
Expose `POST /api/dsa/two-pointers/max-container` with `{ heights }`. Return the best area and steps `{ left, right, area, best, moved }`; validate nonnegative safe integers and cap trace size.

### 21. Mini Challenge
Return the lexicographically smallest index pair among all pairs with maximum area.

## Problem 57 — 3Sum

### 1. Problem Statement
Return every unique ascending triplet of values whose sum is zero, with no duplicate triplets.

### 2. Example
```text
Input: [-1, 0, 1, 2, -1, -4]
Output: [[-1, -1, 2], [-1, 0, 1]]
```

### 3. Understand the Problem
Triplet elements come from three distinct indexes. Output uniqueness is by values, input is not mutated, and values/sums stay within safe integer range.

#### In Simple Terms

Return every different value triplet that uses three different indexes and adds to zero. Do not repeat the same triplet values.

### 4. Real-World Analogy
Fix one weight, then choose two weights from an ordered rack that exactly balance it back to zero.

### 5. Key Observation
After sorting and fixing one anchor, the remaining task is Two Sum in a sorted suffix. Adjacent duplicates can be skipped safely.

### 6. Pattern
```text
Pattern: Sorted anchor plus converging pointers
Why this pattern applies: fixing one value reduces 3Sum to sorted Two Sum.
How to recognize it: unique triples must meet a target and sorting is allowed.
Similar problems: 3Sum Closest, triangle enumeration, 4Sum.
```

### 7. Brute Force
Enumerate every triple of distinct indexes, test its sum, sort the triplet values, and use a set key to remove duplicate outputs.

#### Pseudocode
```text
for i before j before k
    if values sum to zero
        canonicalize triplet and add by key
return unique triplets
```

#### Brute-Force TypeScript

```typescript
function threeSumByCheckingTriples(values: number[]): number[][] {
  const seenTriplets = new Set<string>();
  const triplets: number[][] = [];
  for (let first = 0; first < values.length - 2; first += 1) {
    for (let second = first + 1; second < values.length - 1; second += 1) {
      for (let third = second + 1; third < values.length; third += 1) {
        if (values[first] + values[second] + values[third] !== 0) continue;
        const triplet = [values[first], values[second], values[third]].sort((a, b) => a - b);
        const key = JSON.stringify(triplet);
        if (!seenTriplets.has(key)) {
          seenTriplets.add(key);
          triplets.push(triplet);
        }
      }
    }
  }
  return triplets;
}
```

#### Visualization
```text
choose i: O(n)
 choose j: O(n)
  choose k: O(n)
```

#### Dry Run
The example discovers `[-1,0,1]` and later another index combination with the same values; canonical keys suppress the duplicate.

#### Brute-Force Complexity
**Time: `O(n³)`** for all index triples. **Space: `O(r)`** for `r` unique output triplets and their deduplication keys.

### 8. Why Brute Force Is Not Ideal
It tests cubic combinations and needs separate deduplication state instead of exploiting sorted adjacency.

### 9. Optimal Approach
Sort a copy. For each nonduplicate anchor, sweep the suffix with two pointers. On a zero sum, record the triplet and skip duplicate endpoint values.

### 10. Visualization
```text
sorted: [-4,-1,-1,0,1,2]
anchor -1 at index 1
          L       R   -1 + -1 + 2 = 0 -> record
             L  R     -1 + 0 + 1 = 0 -> record
```

### 11. Step-by-Step Dry Run
Anchor `-4` finds no zero sum. Anchor `-1` starts with `left=-1`, `right=2`, records `[-1,-1,2]`, moves past duplicates, then records `[-1,0,1]`. The next anchor `-1` is skipped.

### 12. TypeScript Solution
```typescript
function threeSum(values: number[]): number[][] {
  const sorted = [...values].sort((first, second) => first - second);
  const triplets: number[][] = [];
  for (let anchor = 0; anchor < sorted.length - 2; anchor += 1) {
    if (anchor > 0 && sorted[anchor] === sorted[anchor - 1]) continue;
    if (sorted[anchor] > 0) break;
    let left = anchor + 1;
    let right = sorted.length - 1;
    while (left < right) {
      const sum = sorted[anchor] + sorted[left] + sorted[right];
      if (sum < 0) left += 1;
      else if (sum > 0) right -= 1;
      else {
        triplets.push([sorted[anchor], sorted[left], sorted[right]]);
        const leftValue = sorted[left];
        const rightValue = sorted[right];
        while (left < right && sorted[left] === leftValue) left += 1;
        while (left < right && sorted[right] === rightValue) right -= 1;
      }
    }
  }
  return triplets;
}
```

### 13. Code Walkthrough
Cloning protects caller input. Anchor skipping removes repeated first values; endpoint skipping after a match removes repeated second/third values without missing a new combination.

### 14. Complexity Analysis
**Time: `O(n²)`** after `O(n log n)` sorting because each of `O(n)` anchors performs one linear sweep. **Auxiliary space: `O(n)`** for the sorted copy, excluding runtime sort workspace and `O(r)` output.

### 15. Edge Cases
- Fewer than three values returns empty.
- `[0,0,0,0]` returns one `[0,0,0]`.
- All positive or all negative values return empty.
- Duplicate-heavy input must not repeat triplets.

### 16. Common Mistakes
- Forgetting numeric comparator in JavaScript sort.
- Mutating input without stating it.
- Skipping duplicates before recording a valid tuple incorrectly.
- Using the same index more than once.

### 17. Interview Follow-ups
1. Target a sum other than zero. 2. Find the closest triplet sum. 3. Count index triplets. 4. Return original indexes.

### 18. Variations
- 3Sum Closest. - Smaller-than-target triplet count. - Triangle triplet count.

### 19. Pattern Connections
This is sorted Two Sum inside an anchor loop; 4Sum adds one more anchor, and k-Sum generalizes recursively.

### 20. NestJS Project Connection
Use `POST /api/dsa/two-pointers/three-sum` with `{ values }`. Return triplets and bounded events for sorting, anchor selection, pointer comparison, duplicate skip, and match.

### 21. Mini Challenge
Generalize the function to return unique triplets for an arbitrary target.

## Problem 58 — 4Sum

### 1. Problem Statement
Given safe integers and a target, return every unique ascending quadruplet of values from four distinct indexes whose sum equals the target.

### 2. Example
```text
Input: values = [1, 0, -1, 0, -2, 2], target = 0
Output: [[-2,-1,1,2], [-2,0,0,2], [-1,0,0,1]]
```

### 3. Understand the Problem
Uniqueness is by quadruplet values, not indexes. The input is not mutated, and all intermediate sums must remain safe integers.

#### In Simple Terms

Return every different value quadruplet that uses four different indexes and adds to the target.

### 4. Real-World Analogy
Choose two fixed items, then slide inward through an ordered catalog to find two more prices completing a budget.

### 5. Key Observation
After sorting and fixing two anchors, each remaining problem is a target-sum pair search in a sorted suffix.

### 6. Pattern
```text
Pattern: Two sorted anchors plus converging pointers
Why this pattern applies: two fixed values reduce 4Sum to sorted Two Sum.
How to recognize it: unique quadruplets must meet a target.
Similar problems: 3Sum, k-Sum, four-array sum counting.
```

### 7. Brute Force
Enumerate four increasing indexes, test each sum, canonicalize matching values, and deduplicate with a set.

#### Pseudocode
```text
for a before b before c before d
    if four-value sum equals target
        add canonical quadruplet by key
```

#### Brute-Force TypeScript

```typescript
function fourSumByCheckingQuadruples(values: number[], target: number): number[][] {
  const seenQuadruplets = new Set<string>();
  const quadruplets: number[][] = [];
  for (let first = 0; first < values.length - 3; first += 1) {
    for (let second = first + 1; second < values.length - 2; second += 1) {
      for (let third = second + 1; third < values.length - 1; third += 1) {
        for (let fourth = third + 1; fourth < values.length; fourth += 1) {
          const sum = values[first] + values[second] + values[third] + values[fourth];
          if (sum !== target) continue;
          const quadruplet = [values[first], values[second], values[third], values[fourth]]
            .sort((a, b) => a - b);
          const key = JSON.stringify(quadruplet);
          if (!seenQuadruplets.has(key)) {
            seenQuadruplets.add(key);
            quadruplets.push(quadruplet);
          }
        }
      }
    }
  }
  return quadruplets;
}
```

#### Visualization
```text
four nested choices -> O(n^4) index combinations
```

#### Dry Run
The example can discover `[-2,0,0,2]` through multiple index choices; a canonical set retains it once.

#### Brute-Force Complexity
**Time: `O(n⁴)`** for four nested choices. **Space: `O(r)`** for unique results and deduplication keys.

### 8. Why Brute Force Is Not Ideal
It ignores ordered elimination and spends a full loop choosing the final two values.

### 9. Optimal Approach
Sort a copy, choose nonduplicate first and second anchors, then run two pointers over the remaining suffix while skipping duplicate endpoints after matches.

### 10. Visualization
```text
[-2,-1,0,0,1,2], target 0
 A  B L       R   sum -1 -> L++
 A  B   L     R   sum -1 -> L++
 A  B     L   R   sum 0 -> [-2,-1,1,2]
```

### 11. Step-by-Step Dry Run
With anchors `-2,-1`, pointers advance until `1+2` completes zero. With anchors `-2,0`, the pair `0,2` records `[-2,0,0,2]`; duplicate second anchors are skipped.

### 12. TypeScript Solution
```typescript
function fourSum(values: number[], target: number): number[][] {
  const sorted = [...values].sort((first, second) => first - second);
  const quadruplets: number[][] = [];
  for (let first = 0; first < sorted.length - 3; first += 1) {
    if (first > 0 && sorted[first] === sorted[first - 1]) continue;
    for (let second = first + 1; second < sorted.length - 2; second += 1) {
      if (second > first + 1 && sorted[second] === sorted[second - 1]) continue;
      let left = second + 1;
      let right = sorted.length - 1;
      while (left < right) {
        const sum = sorted[first] + sorted[second] + sorted[left] + sorted[right];
        if (sum < target) left += 1;
        else if (sum > target) right -= 1;
        else {
          quadruplets.push([sorted[first], sorted[second], sorted[left], sorted[right]]);
          const leftValue = sorted[left];
          const rightValue = sorted[right];
          while (left < right && sorted[left] === leftValue) left += 1;
          while (left < right && sorted[right] === rightValue) right -= 1;
        }
      }
    }
  }
  return quadruplets;
}
```

### 13. Code Walkthrough
Each anchor loop skips only duplicates at its own depth. The two-pointer sweep then emits each remaining value pair once for that unique anchor pair.

### 14. Complexity Analysis
**Time: `O(n³)`** after `O(n log n)` sorting: `O(n²)` anchor pairs each drive an `O(n)` sweep. **Auxiliary space: `O(n)`** for the sorted copy, excluding sort workspace and `O(r)` output.

### 15. Edge Cases
- Fewer than four values returns empty.
- Four equal values may form one result when enough copies exist.
- Duplicate-heavy inputs still return unique quadruplets.
- Large integers require safe-range validation or a consistent `bigint` implementation.

### 16. Common Mistakes
- Skipping the second anchor relative to the wrong boundary.
- Deduplicating values globally and losing required copies.
- Mutating caller input through in-place sort.
- Ignoring overflow/precision assumptions.

### 17. Interview Follow-ups
1. Add min/max pruning. 2. Generalize to k-Sum. 3. Return original index quadruplets. 4. Count quadruplets across four separate arrays.

### 18. Variations
- 4Sum Count. - Closest quadruplet sum. - Unique k-Sum combinations.

### 19. Pattern Connections
4Sum is one more reduction layer than 3Sum: anchors reduce dimensionality until ordered Two Sum remains.

### 20. NestJS Project Connection
Expose `POST /api/dsa/two-pointers/four-sum` with `{ values, target }`. Return results plus paged or capped trace events for anchors, pointers, duplicate skips, sums, and matches.

### 21. Mini Challenge
Add safe lower/upper-bound pruning for each anchor without excluding any valid quadruplet.

## Problem 59 — Sort Colors

### 1. Problem Statement
Sort an array containing only `0`, `1`, and `2` in place using one pass and constant extra space.

### 2. Example
```text
Input: [2, 0, 2, 1, 1, 0]
Output: [0, 0, 1, 1, 2, 2]
```

### 3. Understand the Problem
The values model three colors and must be ordered `0`, then `1`, then `2`. The array is intentionally mutated; other values violate the contract.

#### In Simple Terms

Rearrange an array containing only `0`, `1`, and `2` so all zeroes come first, then ones, then twos.

### 4. Real-World Analogy
Sort red, white, and blue cards by growing a finished pile for the first color at the left and the last color at the right.

### 5. Key Observation
Maintain four regions: finalized zeroes, finalized ones, unknown values, and finalized twos. Every step shrinks the unknown region.

### 6. Pattern
```text
Pattern: Dutch National Flag three-way partition
Why this pattern applies: the input has exactly three ordered categories.
How to recognize it: partition a small fixed domain in one pass and in place.
Similar problems: three-way quicksort, partition around a pivot, segregate categories.
```

### 7. Brute Force
Use a comparison sort to order the numeric values.

#### Pseudocode
```text
sort values in ascending numeric order
```

#### Brute-Force TypeScript

```typescript
function sortColorsWithGeneralSort(values: number[]): void {
  for (const value of values) {
    if (value !== 0 && value !== 1 && value !== 2) {
      throw new RangeError("values must contain only 0, 1, and 2");
    }
  }
  values.sort((first, second) => first - second);
}
```

#### Visualization
```text
[2,0,2,1,1,0] --comparison sort--> [0,0,1,1,2,2]
```

#### Dry Run
The sorting routine repeatedly compares and rearranges values until the full order is established.

#### Brute-Force Complexity
**Time: typically `O(n log n)`** for a comparison sort. **Extra space:** depends on the runtime sorting implementation and is not guaranteed `O(1)`.

### 8. Why Brute Force Is Not Ideal
It ignores the three-value domain and does more comparisons than a direct partition requires.

### 9. Optimal Approach
Use `low` for the next zero position, `current` for the next unknown value, and `high` for the next two position. Swap according to the current value.

### 10. Visualization
```text
[ zeros | ones | unknown | twos ]
          low/current     high

current 0 -> swap with low; advance both
current 1 -> advance current
current 2 -> swap with high; retreat high; recheck current
```

### 11. Step-by-Step Dry Run
| Array | low | current | high | Action |
|---|---:|---:|---:|---|
| `[2,0,2,1,1,0]` | 0 | 0 | 5 | swap 2 with high |
| `[0,0,2,1,1,2]` | 0 | 0 | 4 | place 0 left |
| `[0,0,2,1,1,2]` | 1 | 1 | 4 | place 0 left |
| `[0,0,2,1,1,2]` | 2 | 2 | 4 | swap 2 with high |
| `[0,0,1,1,2,2]` | 2 | 2 | 3 | advance across ones |

### 12. TypeScript Solution
```typescript
function sortColors(values: number[]): void {
  let low = 0;
  let current = 0;
  let high = values.length - 1;
  while (current <= high) {
    if (values[current] === 0) {
      [values[low], values[current]] = [values[current], values[low]];
      low += 1;
      current += 1;
    } else if (values[current] === 1) {
      current += 1;
    } else if (values[current] === 2) {
      [values[current], values[high]] = [values[high], values[current]];
      high -= 1;
    } else {
      throw new RangeError("values must contain only 0, 1, and 2");
    }
  }
}
```

### 13. Code Walkthrough
Indexes below `low` are zeroes, `[low,current)` are ones, and indexes above `high` are twos. A value swapped from `high` remains unknown, so `current` must not advance in that branch.

### 14. Complexity Analysis
**Time: `O(n)`** because each iteration advances `current` or retreats `high`, shrinking the unknown interval. **Extra space: `O(1)`** because sorting uses only three indexes and swaps.

### 15. Edge Cases
- Empty and single-value arrays work.
- Already grouped values remain valid.
- All values may belong to one category.
- Any value outside `0`, `1`, and `2` throws.

### 16. Common Mistakes
- Advancing `current` after swapping with `high`.
- Using three output arrays while claiming in-place space.
- Omitting input-domain validation in a public API.
- Confusing color labels with arbitrary sortable numbers.

### 17. Interview Follow-ups
1. Sort four categories. 2. Partition around a pivot. 3. Make the result stable. 4. Count swaps or writes.

### 18. Variations
- Counting-sort colors. - Three-way quicksort partition. - Binary segregation.

### 19. Pattern Connections
This generalizes two-pointer partitioning by maintaining three monotonic boundaries and four regions.

### 20. NestJS Project Connection
Expose `POST /api/dsa/two-pointers/sort-colors` with `{ values }`. Validate the domain, mutate a clone, and return snapshots `{ low, current, high, values, action }`.

### 21. Mini Challenge
Design a one-pass in-place partition for values less than, equal to, and greater than an arbitrary pivot.

## Problem 60 — Trapping Rain Water

### 1. Problem Statement
Given nonnegative integer bar heights of unit width, return the total amount of rain water trapped after raining.

### 2. Example
```text
Input: [0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1]
Output: 6
```

### 3. Understand the Problem
Water above index `i` is limited by the shorter of the highest bar on its left and right, minus its own height. The total must remain a safe integer.

#### In Simple Terms

For each bar, find how much water can sit above it between taller bars, then add those amounts together.

### 4. Real-World Analogy
Each low bar is a floor section between walls; water depth depends on the shorter surrounding wall.

### 5. Key Observation
When the left boundary is no taller than the right boundary, the left side's water is fully determined by `leftMax`; a right boundary at least that tall exists. The symmetric rule holds on the right.

### 6. Pattern
```text
Pattern: Opposite pointers with running boundary maxima
Why this pattern applies: the lower current boundary determines which side is safe to finalize.
How to recognize it: each position depends on maxima from both directions.
Similar problems: Container With Most Water, prefix/suffix maxima, bounded elevation fill.
```

### 7. Brute Force
For every index, scan left and right to find the highest surrounding bars, then add nonnegative trapped depth.

#### Pseudocode
```text
water = 0
for each index
    leftMax = maximum from start through index
    rightMax = maximum from index through end
    water += min(leftMax, rightMax) - height[index]
return water
```

#### Brute-Force TypeScript

```typescript
function trapRainWaterByScanningBothSides(heights: number[]): number {
  let water = 0;
  for (let index = 0; index < heights.length; index += 1) {
    let leftMax = heights[index];
    let rightMax = heights[index];
    for (let left = 0; left <= index; left += 1) leftMax = Math.max(leftMax, heights[left]);
    for (let right = index; right < heights.length; right += 1) {
      rightMax = Math.max(rightMax, heights[right]);
    }
    water += Math.min(leftMax, rightMax) - heights[index];
  }
  return water;
}
```

#### Visualization
```text
for each bar i:
scan <--- left max | i | right max ---> scan
```

#### Dry Run
At a valley of height `0` between known maxima `2` and `3`, the depth is `min(2,3)-0 = 2`.

#### Brute-Force Complexity
**Time: `O(n²)`** because two range scans are repeated for each index. **Space: `O(1)`** besides scalar maxima.

### 8. Why Brute Force Is Not Ideal
Neighboring positions repeatedly rescan almost identical left and right ranges.

### 9. Optimal Approach
Track maxima while moving inward. Finalize the side with the lower current boundary, update that side's maximum, and add the gap below it.

### 10. Visualization
```text
heights: [0,1,0,2,1,0,1,3,2,1,2,1]
          L                     R

lower side moves inward
leftMax/rightMax retain the best walls already crossed
accumulated gap at each finalized bar becomes permanent
```

### 11. Step-by-Step Dry Run
| Side finalized | Height | Side max after | Water added | Total |
|---|---:|---:|---:|---:|
| left index 0 | 0 | 0 | 0 | 0 |
| left index 1 | 1 | 1 | 0 | 0 |
| left index 2 | 0 | 1 | 1 | 1 |
| right index 11 | 1 | 1 | 0 | 1 |
| right index 10 | 2 | 2 | 0 | 1 |

Continuing finalizes all indexes and accumulates `6` units.

### 12. TypeScript Solution
```typescript
function trapRainWater(heights: number[]): number {
  let left = 0;
  let right = heights.length - 1;
  let leftMax = 0;
  let rightMax = 0;
  let water = 0;
  while (left <= right) {
    if (heights[left] <= heights[right]) {
      leftMax = Math.max(leftMax, heights[left]);
      water += leftMax - heights[left];
      left += 1;
    } else {
      rightMax = Math.max(rightMax, heights[right]);
      water += rightMax - heights[right];
      right -= 1;
    }
  }
  return water;
}
```

### 13. Code Walkthrough
The lower current boundary proves a boundary at least that high exists across the interval, so its side can be finalized. Running maxima convert each finalized bar directly into trapped depth.

### 14. Complexity Analysis
**Time: `O(n)`** because each iteration finalizes one index and pointers never reverse. **Space: `O(1)`** because only pointers, maxima, and the total are stored.

### 15. Edge Cases
- Fewer than three bars traps zero.
- Monotonic and flat arrays trap zero.
- Interior zeroes may trap water.
- Negative heights are invalid under the elevation contract.

### 16. Common Mistakes
- Adding `min(height[left], height[right]) - current height` without running maxima.
- Moving the taller side without the correct invariant.
- Confusing total trapped water with maximum container area.
- Allowing negative depth contributions.

### 17. Interview Follow-ups
1. Solve with prefix/suffix arrays. 2. Solve with a monotonic stack. 3. Return depth at each index. 4. Handle variable bar widths.

### 18. Variations
- Trapping Rain Water II on a grid. - Pour Water simulation. - Basin identification.

### 19. Pattern Connections
Container With Most Water chooses two boundaries for one area; this problem aggregates local areas using remembered boundary maxima.

### 20. NestJS Project Connection
Use `POST /api/dsa/two-pointers/trap-rain-water` with `{ heights }`. Return total water and capped events `{ left, right, leftMax, rightMax, added, total, side }` after nonnegative safe-integer validation.

### 21. Mini Challenge
Return both the total and an array containing trapped depth above every index in `O(n)` time.

---

# Topic Summary

## What You Learned

- Two-pointer correctness depends on an invariant, not merely on two index variables.
- Sorted order lets one movement eliminate many impossible pairs.
- Same-direction read/write pointers compact values stably in place.
- Mirrored pointers reverse or compare symmetric sequence positions.
- A limiting boundary determines which side an optimization can safely discard or finalize.
- Sorted anchors reduce 3Sum and 4Sum to repeated Two Sum sweeps.
- Duplicate skipping must happen at the correct anchor or endpoint depth.
- Three-way partitioning grows finalized regions around a shrinking unknown interval.
- Mutation, sort copies, numeric safety, and output storage belong in the contract and complexity analysis.

## Patterns You Should Recognize

- sorted pair target -> opposite-direction pointers
- stable in-place filtering -> read/write pointers
- reversal or palindrome -> mirrored pointers
- triplets/quadruplets -> sorted anchors plus Two Sum
- endpoint-limited score -> move or finalize the limiting boundary
- three fixed categories -> Dutch National Flag partition
- per-position two-sided bounds -> running left/right maxima

## Complexity Cheat Sheet

| Operation or pattern | Time | Extra space |
|---|---:|---:|
| Opposite-end sorted sweep | `O(n)` | `O(1)` |
| Read/write compaction | `O(n)` | `O(1)` |
| Mirrored code-point string reversal | `O(n)` | `O(n)` representation/output |
| Sort then one sweep | `O(n log n)` | Depends on clone/sort |
| 3Sum with sorted two pointers | `O(n²)` | `O(n)` clone plus output |
| 4Sum with sorted two pointers | `O(n³)` | `O(n)` clone plus output |
| Dutch National Flag partition | `O(n)` | `O(1)` |
| Two-sided running maxima | `O(n)` | `O(1)` |

## Common Interview Traps

- Applying sorted-pointer movement to unsorted data.
- Counting nested pointer movement as quadratic when neither pointer resets.
- Sorting without including `O(n log n)` in total complexity.
- Mutating caller input through `sort`, swaps, or compaction without stating it.
- Returning values when indexes were requested, or using the wrong index base.
- Skipping tuple duplicates at the wrong loop depth.
- Advancing the current pointer after swapping in an unknown high-side value.
- Moving the taller container boundary instead of the limiting boundary.
- Treating Trapping Rain Water as the same objective as Container With Most Water.
- Ignoring JavaScript safe-integer limits for sums, areas, and totals.

## Practice Checklist

- [ ] Problem 51 — Two Sum in a Sorted Array
- [ ] Problem 52 — Remove Duplicates Using Two Pointers
- [ ] Problem 53 — Move Zeroes Using Two Pointers
- [ ] Problem 54 — Reverse a String Using Two Pointers
- [ ] Problem 55 — Valid Palindrome
- [ ] Problem 56 — Container With Most Water
- [ ] Problem 57 — 3Sum
- [ ] Problem 58 — 4Sum
- [ ] Problem 59 — Sort Colors
- [ ] Problem 60 — Trapping Rain Water