# Heaps and Priority Queues

## Learning Objectives

By the end of this topic, you should understand:

- how a binary heap combines a complete-tree shape with a local priority invariant
- how a dense array represents parent and child relationships without node objects
- how a comparator turns one heap implementation into a min-heap or max-heap
- why insertion uses sift-up and root removal uses sift-down
- how bottom-up heap construction differs from repeated insertion
- why heap sort is in-place but not stable
- how bounded heaps solve kth-element and top-k problems
- how a heap performs multiway merging by storing only one candidate per source
- how two heaps maintain the median of an online stream
- why greedy rope merging always combines the two smallest available lengths
- how ownership, tie-breaking, safe arithmetic, and bounded traces affect production APIs

## Prerequisites

- arrays, loops, classes, generics, and object references
- binary-tree terminology from `11-binary-trees.md`
- sorting and selection concepts from earlier chapters
- strict TypeScript, readonly input, and explicit mutation contracts
- Big-O analysis, including amortized array growth and output space
- safe-integer arithmetic for exact numeric algorithms

## Core Theory

### Heap versus priority queue

A **priority queue** is an abstract data type: insert an item, inspect the highest-priority item, and remove the highest-priority item. A **binary heap** is one efficient array-backed implementation.

A heap is not a fully sorted collection. It guarantees only that each parent has at least as much priority as its children. The root is therefore optimal, while arbitrary siblings and cousins need not be ordered.

### Generic comparator contract

This chapter uses one comparator convention:

```typescript
type Comparator<T> = (left: T, right: T) => number;

const minNumberPriority: Comparator<number> = (left, right) => left - right;
const maxNumberPriority: Comparator<number> = (left, right) => right - left;
```

`compare(a, b) < 0` means `a` has higher priority and should be closer to the root. Zero means equal priority; heaps permit duplicates, and equal-priority removal order is unspecified unless an algorithm supplies an explicit secondary key.

The comparator must be deterministic and define a consistent total preorder. Algorithms inspect only its sign and cannot repair a stateful, nontransitive, or contradictory comparator.

### Complete-tree array layout

A binary heap occupies a dense array with no structural gaps. For a node at zero-based index `i`:

```text
parent(i) = floor((i - 1) / 2), for i > 0
left(i)   = 2i + 1
right(i)  = 2i + 2
```

The complete-tree shape guarantees height `O(log n)`. No child pointers or explicit tree nodes are needed.

```text
array: [2, 5, 4, 9, 7, 8]

          2
        /   \
       5     4
      / \   /
     9   7 8
```

### Min-heap and max-heap invariants

In a min-heap, no child has higher priority than its parent under ascending comparison, so the smallest value is at index `0`. In a max-heap, descending comparison gives larger values higher priority, so the largest value is at index `0`.

Neither invariant sorts the whole array. For example, `[2, 9, 4]` is a valid min-heap even though `9 > 4` appears before `4`.

### Sift-up

Insertion appends one value at the next open slot, preserving complete shape. While the new value has higher priority than its parent, swap them. Each swap moves one level upward, so insertion takes `O(log n)` worst-case time.

```text
append 1 to [2, 5, 4, 9, 7]
[2, 5, 4, 9, 7, 1]
swap with 4 -> [2, 5, 1, 9, 7, 4]
swap with 2 -> [1, 5, 2, 9, 7, 4]
```

### Sift-down

Removing the root saves its value, moves the final array value into index `0`, shrinks the array, and repeatedly swaps the replacement with its higher-priority child. Comparing both children before swapping is essential.

Sift-down also repairs a local violation during bottom-up heap construction and heap sort.

### Bottom-up heap construction

Every index from `floor(n / 2)` through `n - 1` is a leaf and already satisfies the heap invariant. Bottom-up heapify starts at the last parent, `floor(n / 2) - 1`, and sifts each parent downward.

Although one sift-down can take `O(log n)`, most nodes lie near the leaves and move only a few levels. Summed over all nodes, bottom-up heapify takes `O(n)` time, not `O(n log n)`.

### Ownership and mutation

The heap classes in Problems 176 and 177 own their internal arrays. Constructors copy caller input, `toArray()` returns a copy, and stored object values retain their identities. `peek()` and `pop()` return stored values or `undefined` when empty.

Problem 178 heapifies a caller-owned mutable array in place and returns the same array reference. Problem 179 sorts a mutable numeric array in place and returns that same reference. Selection, frequency, merge, median-stream input, and rope-cost functions do not mutate caller-owned arrays unless their problem statement explicitly says otherwise.

### Stability and deterministic ties

A basic heap is not stable. Equal-priority items may leave in any order because swaps depend only on priority. Algorithms whose output order is observable add explicit tie-breakers:

- Top K Frequent Elements: higher frequency first, then smaller numeric value.
- Merge K Sorted Arrays: smaller value first, then smaller source-array index, then smaller element index.

The tie-breaker belongs in the heap entry comparator, not in assumptions about incidental array layout.

### Bounded heaps for selection

To find the kth largest value, retain a min-heap of at most `k` values. Its root is the smallest retained candidate; a larger incoming value replaces it. The root after all input is the kth largest.

The kth smallest problem is symmetric: retain a max-heap of size `k`, whose root is the largest retained candidate. These algorithms take `O(n log k)` time and `O(k)` auxiliary space.

### Multiway merge

For `k` sorted arrays, insert only each nonempty array's first value. Removing the smallest entry reveals exactly one next candidate from the same source. The heap therefore stores at most `k` entries while producing all `N` output values in `O(N log k)` time.

### Streaming median invariant

Maintain two heaps:

- a max-heap `lower` for the lower half
- a min-heap `upper` for the upper half

Every value in `lower` is at most every value in `upper`, and their sizes differ by at most one. Keep `lower` the same size as `upper` or one larger. The median is `lower.peek()` for odd count and the overflow-aware numeric average of both roots for even count. Ordinary half-integers such as `1.5` are exact, but an extreme mathematical half-integer may round when no JavaScript `number` represents it.

### Safe numeric arithmetic

Problems involving exact counts or accumulated costs state safe-integer contracts. Median averages avoid overflowing intermediate addition, while their final `number` result remains subject to IEEE-754 representability. JavaScript can silently round integer arithmetic outside `Number.MIN_SAFE_INTEGER` through `Number.MAX_SAFE_INTEGER`.

For an overflow-safe average of two safe integers `a <= b`, use:

```text
a + (b - a) / 2
```

The difference can itself exceed the safe range when signs differ, so this chapter's median implementation handles same-sign and opposite-sign pairs separately and validates the result. Rope merging validates every addition and accumulated total before returning it.

## Visual Mental Model

For min-heap array `[1, 3, 2, 8, 6, 7, 5]`:

```text
index:  0  1  2  3  4  5  6
value: [1, 3, 2, 8, 6, 7, 5]

             1
          /     \
         3       2
        / \     / \
       8   6   7   5
```

Removing `1` moves `5` to the root. Between children `3` and `2`, value `2` has higher min-priority, so sift-down swaps with `2`, not merely with the left child.

## Complexity Concepts

### Core heap operations

| Operation | Time | Auxiliary space beyond owned array |
|---|---:|---:|
| `peek` / `size` | `O(1)` | `O(1)` |
| `push` | `O(log n)` worst case | `O(1)` |
| `pop` | `O(log n)` worst case | `O(1)` |
| Bottom-up heapify | `O(n)` | `O(1)` in place |
| Copy a heap snapshot | `O(n)` | `O(n)` returned copy |

Dynamic-array append is amortized `O(1)` before sift-up. A heap's complete shape keeps every root-to-leaf path logarithmic.

### Sorting and selection

Heap sort performs `O(n log n)` comparisons in the best, average, and worst cases and uses `O(1)` auxiliary space, but it is unstable. A bounded size-`k` heap changes full sorting from `O(n log n)` to `O(n log k)` when only one rank or `k` winners are needed.

### Output-sensitive algorithms

Merging sorted arrays must emit `N` values, so `O(N)` output space is unavoidable. Top-k output requires `O(k)` result space. Auxiliary heap storage should be reported separately from returned output.

## Common Patterns

### Insert and sift up

```text
append value
while value outranks parent:
    swap value with parent
```

### Remove root and sift down

```text
save root
move final value to root and shrink
while a child outranks current:
    choose highest-priority child
    swap downward
return saved root
```

### Keep only k candidates

```text
for each value:
    push while heap has fewer than k
    otherwise replace root only when value beats the boundary
```

### Pop then advance source

```text
seed one entry per source
while heap not empty:
    pop best entry and emit it
    push that source's next entry
```

## Common Mistakes

- Treating a heap array as globally sorted.
- Swapping with the left child without comparing the right child.
- Calling `Array.shift()` for root removal and making every pop linear.
- Forgetting that duplicates are valid heap elements.
- Claiming bottom-up heapify is `O(n log n)`.
- Forgetting heap sort's ascending form needs a max-heap.
- Keeping `n - k + 1` candidates when a size-`k` heap is simpler.
- Omitting deterministic tie-breakers from observable top-k or merge output.
- Mutating caller arrays in read-only problems.
- Averaging two large integers with `(a + b) / 2` without overflow analysis.
- Accumulating rope costs without safe-integer checks.
- Treating heap operations as stable for comparator-equal objects.

## Production Notes

- Use fixed server-side comparators; never accept executable comparator code from clients.
- Bound input length, heap capacity, output size, and trace events.
- Validate numeric finiteness and safe-integer requirements before mutation when atomic failure is promised.
- Avoid logging arbitrary heap values or full heap snapshots in each trace event.
- Use stable IDs and explicit tie-breakers when equal priorities require deterministic responses.
- Prefer request-scoped heap instances unless shared mutation and concurrency are deliberately synchronized.
- Use a proven queue or heap package in production when decrease-key, handles, concurrency, or persistence is required.

## Problem 176 — Implement Min Heap

### 1. Problem Statement
Implement a generic `MinHeap<T>` backed by an owned dense array. The constructor accepts readonly initial values and a `Comparator<T>`, copies the values, and builds the heap bottom-up in `O(n)` time. A negative comparator result means the left value has higher priority, so an ordinary ascending comparator creates a min-heap. Provide a `size` getter, `peek(): T | undefined`, `push(value): void`, `pop(): T | undefined`, and `toArray(): T[]`.

The comparator's sign alone is meaningful and must define a consistent total preorder. Duplicates are allowed, equal-priority removal order is unspecified, and stored object identities must be preserved. The heap owns its internal array: later caller mutations cannot affect it, and callers cannot mutate it through `toArray()`.

### 2. Example
```text
compare = (left, right) => left - right
initial = [7, 2, 5, 2]

heap.peek() -> 2
heap.size   -> 4
heap.pop()  -> 2
heap.push(1)
heap.peek() -> 1

The two equal 2 values may be removed in either order.
```

### 3. Understand the Problem
The array must satisfy one local rule: no child has higher priority than its parent. With the chapter's comparator convention, `compare(child, parent) < 0` identifies a violation. The root is therefore a minimum under an ascending comparator even though the rest of the array is not globally sorted.

#### In Simple Terms
Keep the smallest-priority value at the front by bubbling new values up and replacement roots down.

### 4. Real-World Analogy
Imagine an emergency desk where the most urgent case is always at the front. A new case moves upward only while it outranks its parent position; after the front case leaves, the final case moves to the front and descends until more urgent children are above it.

### 5. Key Observation
A complete binary tree has logarithmic height and fits densely in an array. Appending preserves its shape, and removing the last element is constant time. Therefore `push` and `pop` need to repair only one root-to-leaf path; root removal never needs `shift()`.

### 6. Pattern
```text
Pattern: Array-Backed Heap with Sift Operations
Why this pattern applies: complete shape gives index arithmetic and one-path repair
How to recognize it: repeatedly insert, inspect, or remove the highest-priority value
Similar problems: max heap, bounded top-k heap, Dijkstra frontier, event scheduler
```

### 7. Brute Force
Store an owned but unsorted array. `push` appends in `O(1)` amortized time. `peek` and `pop` scan for the first highest-priority value; `pop` removes that index with `splice`, which is linear because later elements may shift. This baseline satisfies the API and makes no stability promise.

#### Pseudocode
```text
push(value): append value
peek(): scan for an item that outranks the current candidate
pop():
        scan for the highest-priority index
        remove and return that item
empty peek or pop: return undefined
```

#### Brute-Force TypeScript
```typescript
type Comparator<T> = (left: T, right: T) => number;

class BruteMinHeap<T> {
    private readonly values: T[];

    constructor(
        initialValues: readonly T[],
        private readonly compare: Comparator<T>,
    ) {
        this.values = [...initialValues];
    }

    get size(): number {
        return this.values.length;
    }

    peek(): T | undefined {
        const index = this.bestIndex();
        return index === -1 ? undefined : this.values[index];
    }

    push(value: T): void {
        this.values.push(value);
    }

    pop(): T | undefined {
        const index = this.bestIndex();
        if (index === -1) return undefined;
        return this.values.splice(index, 1)[0];
    }

    toArray(): T[] {
        return [...this.values];
    }

    private bestIndex(): number {
        if (this.values.length === 0) return -1;

        let best = 0;
        for (let index = 1; index < this.values.length; index += 1) {
            if (this.compare(this.values[index]!, this.values[best]!) < 0) {
                best = index;
            }
        }
        return best;
    }
}
```

#### Visualization
```text
unsorted storage: [7, 2a, 5, 2b]
scan candidates:   7 -> 2a -> 2a -> 2a
pop removes 2a:   [7, 5, 2b]

Choosing 2a rather than 2b is incidental, not a stability guarantee.
```

#### Dry Run
Starting with `[7, 2, 5, 2]`, `bestIndex` begins at `7`, changes to the first `2`, and remains there because equal priority does not produce a negative result. `pop` removes that element and preserves the identities of all remaining objects. On an empty array, `bestIndex` returns `-1`, so neither `peek` nor `pop` calls the comparator.

#### Brute-Force Complexity
Construction takes `O(n)` time and `O(n)` owned storage for the copy. `push` is amortized `O(1)`, while `peek` and `pop` are `O(n)`; `pop` also may shift `O(n)` elements. `size` is `O(1)`, and `toArray` takes `O(n)` time and output space.

### 8. Why Brute Force Is Not Ideal
The baseline repeatedly scans values whose relative priority could have been organized once. A heap spends logarithmic work during updates so the highest-priority value is always available at index `0` in constant time.

### 9. Optimal Approach
Copy the initial values and sift down every parent from `floor(n / 2) - 1` to `0`; this bottom-up construction is `O(n)`. For `push`, append and sift up while the child outranks its parent. For `pop`, save the root, remove the final value with `pop()`, place it at the root when elements remain, and sift it down against the higher-priority of both children.

### 10. Visualization
```text
push 1 into [2, 5, 4, 9, 7]:
[2, 5, 4, 9, 7, 1]
[2, 5, 1, 9, 7, 4]
[1, 5, 2, 9, 7, 4]

pop from [1, 5, 2, 9, 7, 4]:
[4, 5, 2, 9, 7]
[2, 5, 4, 9, 7]
```

### 11. Step-by-Step Dry Run
| Operation | Array after operation | Returned value |
|---|---|---:|
| construct from `[7, 2, 5, 2]` | one valid arrangement is `[2, 2, 5, 7]` | — |
| `peek()` | `[2, 2, 5, 7]` | `2` |
| `pop()` | `[2, 7, 5]` | `2` |
| `push(1)` | `[1, 2, 5, 7]` | — |

Other valid arrangements among comparator-equal values are permitted.

### 12. TypeScript Solution
```typescript
type Comparator<T> = (left: T, right: T) => number;

class MinHeap<T> {
    private readonly values: T[];

    constructor(
        initialValues: readonly T[],
        private readonly compare: Comparator<T>,
    ) {
        this.values = [...initialValues];
        for (let index = Math.floor(this.values.length / 2) - 1; index >= 0; index -= 1) {
            this.siftDown(index);
        }
    }

    get size(): number {
        return this.values.length;
    }

    peek(): T | undefined {
        return this.values[0];
    }

    push(value: T): void {
        this.values.push(value);
        this.siftUp(this.values.length - 1);
    }

    pop(): T | undefined {
        if (this.values.length === 0) return undefined;

        const root = this.values[0]!;
        const last = this.values.pop()!;
        if (this.values.length > 0) {
            this.values[0] = last;
            this.siftDown(0);
        }
        return root;
    }

    toArray(): T[] {
        return [...this.values];
    }

    private siftUp(startIndex: number): void {
        let index = startIndex;
        while (index > 0) {
            const parent = Math.floor((index - 1) / 2);
            if (this.compare(this.values[index]!, this.values[parent]!) >= 0) break;
            [this.values[index], this.values[parent]] = [this.values[parent]!, this.values[index]!];
            index = parent;
        }
    }

    private siftDown(startIndex: number): void {
        let index = startIndex;
        while (true) {
            const left = index * 2 + 1;
            if (left >= this.values.length) return;

            const right = left + 1;
            let bestChild = left;
            if (
                right < this.values.length &&
                this.compare(this.values[right]!, this.values[left]!) < 0
            ) {
                bestChild = right;
            }

            if (this.compare(this.values[bestChild]!, this.values[index]!) >= 0) return;
            [this.values[index], this.values[bestChild]] = [
                this.values[bestChild]!,
                this.values[index]!,
            ];
            index = bestChild;
        }
    }
}
```

### 13. Code Walkthrough
The constructor copies before mutating, then repairs only parent indexes. `siftUp` compares a value with its parent. `siftDown` first chooses the higher-priority child and only then decides whether to swap, preventing the common left-child-only bug. `pop()` uses the array's last element, never `shift()`. Values move only as references, so object identities are preserved.

### 14. Complexity Analysis
Bottom-up construction takes `O(n)` time and the class owns `O(n)` storage for its required copy. `push` and `pop` take `O(log n)` worst-case time and `O(1)` auxiliary space. `peek` and `size` take `O(1)`. `toArray` takes `O(n)` time and returns `O(n)` independent output storage.

### 15. Edge Cases
- Empty construction creates a size-zero heap without comparator calls.
- `peek()` and `pop()` return exactly `undefined` when empty.
- Popping a one-element heap returns that value and leaves size zero.
- Duplicates and comparator-equal distinct objects are retained.
- Equal-priority removal order is unspecified.
- Caller mutation of the initial array does not affect the heap.
- Mutation of a `toArray()` result does not affect the heap.
- Comparator result magnitude is irrelevant; only its sign is inspected.

### 16. Common Mistakes
- Passing a descending comparator while expecting an ordinary numeric min-heap.
- Keeping the caller's array reference instead of copying it.
- Building the constructor by repeated `push` and claiming `O(n)` time.
- Using `shift()` to remove the root.
- Comparing only the left child during sift-down.
- Dropping duplicates or claiming equal values leave stably.
- Returning the internal array from `toArray()`.

### 17. Interview Follow-ups
1. Add `replaceRoot`. 2. Add a bounded-capacity policy. 3. Support explicit stable sequence-number ties. 4. Explain why bottom-up heapify is linear. 5. Add index handles for decrease-key.

### 18. Variations
- Min-priority queue. - D-ary heap. - Indexed heap. - Stable heap with explicit sequence keys. - Immutable persistent heap.

### 19. Pattern Connections
The same sift operations power bounded top-k selection, multiway merging, shortest-path frontiers, and event scheduling. Problem 178 exposes constructor heapify as a standalone in-place algorithm, while heap sort repeatedly applies a max-oriented sift-down to a shrinking prefix.

### 20. NestJS Project Connection
Expose a bounded operation endpoint such as `POST /api/dsa/heaps/min`. Select a fixed server-side comparator and copy DTO arrays before constructing or mutating algorithm state. Return approved serialized values rather than internal storage. If demonstrations emit comparison or swap traces, cap the event count and mark truncation; do not log arbitrary payload objects or a full snapshot after every step.

### 21. Mini Challenge
Add `replaceRoot(value): T | undefined`, which returns `undefined` and inserts on an empty heap, or replaces and returns the old root otherwise. Preserve object identities, use only comparator signs, and achieve `O(log n)` worst-case time without calling `pop()` followed by `push()`.

## Problem 177 — Implement Max Heap

### 1. Problem Statement
Implement a generic `MaxHeap<T>` backed by an owned dense array. The caller supplies an ordinary ascending value comparator: `compare(a, b) < 0` means `a` precedes `b`, and `compare(a, b) > 0` means `a` follows and is larger than `b`. The class internally treats a positive result as higher priority, so the largest value is at the root.

The constructor must copy readonly initial values and build the heap bottom-up in `O(n)` time. Provide a `size` getter, `peek(): T | undefined`, `push(value): void`, `pop(): T | undefined`, and `toArray(): T[]`. The comparator's sign alone matters and must define a consistent total preorder. Duplicates are allowed, equal-priority order is unspecified, and stored object identities are preserved.

### 2. Example
```text
ascendingCompare = (left, right) => left - right
initial = [3, 8, 5, 8]

heap.peek() -> 8
heap.size   -> 4
heap.pop()  -> 8
heap.push(10)
heap.peek() -> 10

Either comparator-equal 8 may be removed first.
```

### 3. Understand the Problem
This problem deliberately differs from the chapter foundation's generic priority comparator. There, a negative result directly means higher priority, and a max-heap could receive a reversed comparator. Here, callers always provide ordinary ascending value order, and `MaxHeap` itself interprets `compare(a, b) > 0` as “`a` has higher max priority.”

#### In Simple Terms
Callers describe normal ascending order; the class keeps the largest value at the front.

### 4. Real-World Analogy
Think of a leaderboard whose comparison rule says which score is numerically lower or higher. The board, not the caller, decides that the higher score belongs at the top and repairs positions after every arrival or removal.

### 5. Key Observation
The heap mechanics do not change between min and max heaps; only the higher-priority relation changes. Defining that relation once as `compare(left, right) > 0` prevents accidental double reversal and lets all sift logic read consistently.

### 6. Pattern
```text
Pattern: Max-Oriented Array Heap
Why this pattern applies: one reversed priority relation reuses standard sift mechanics
How to recognize it: repeatedly retrieve the greatest value under an ascending order
Similar problems: heap sort, kth smallest, maximum event score, bounded lower ranks
```

### 7. Brute Force
Keep a copied unsorted array. Appending is amortized constant time. To inspect or remove the maximum, scan for an element whose ascending comparison against the current candidate is positive. Remove by index with `splice`; do not claim stability when equal values happen to remain in insertion order in a particular run.

#### Pseudocode
```text
push(value): append value
peek(): scan for the greatest value under ascending comparison
pop():
        scan for the greatest index
        remove and return that item
empty peek or pop: return undefined
```

#### Brute-Force TypeScript
```typescript
type Comparator<T> = (left: T, right: T) => number;

class BruteMaxHeap<T> {
    private readonly values: T[];

    constructor(
        initialValues: readonly T[],
        private readonly compare: Comparator<T>,
    ) {
        this.values = [...initialValues];
    }

    get size(): number {
        return this.values.length;
    }

    peek(): T | undefined {
        const index = this.greatestIndex();
        return index === -1 ? undefined : this.values[index];
    }

    push(value: T): void {
        this.values.push(value);
    }

    pop(): T | undefined {
        const index = this.greatestIndex();
        if (index === -1) return undefined;
        return this.values.splice(index, 1)[0];
    }

    toArray(): T[] {
        return [...this.values];
    }

    private greatestIndex(): number {
        if (this.values.length === 0) return -1;

        let greatest = 0;
        for (let index = 1; index < this.values.length; index += 1) {
            if (this.compare(this.values[index]!, this.values[greatest]!) > 0) {
                greatest = index;
            }
        }
        return greatest;
    }
}
```

#### Visualization
```text
unsorted storage: [3, 8a, 5, 8b]
scan candidates:   3 -> 8a -> 8a -> 8a
pop removes 8a:   [3, 5, 8b]

The scan's first-equal choice is not a promised stable policy.
```

#### Dry Run
The candidate starts at `3`. Comparing `8` with `3` is positive, so the candidate becomes the first `8`; `5` cannot replace it, and the second `8` compares equal. `pop` removes the selected stored reference. Empty operations return `undefined` without invoking the comparator.

#### Brute-Force Complexity
Copy construction takes `O(n)` time and owned storage. `push` is amortized `O(1)`. `peek` and `pop` take `O(n)` time, including possible element shifts during `splice`. `size` is `O(1)`, and `toArray` takes `O(n)` time and output space.

### 8. Why Brute Force Is Not Ideal
Every maximum query repeats a full scan. Organizing the same owned values as a max-heap keeps the greatest value at index `0`, reducing inspection to constant time and updates to one logarithmic path.

### 9. Optimal Approach
Copy the input and bottom-up sift every parent using `higherPriority(a, b)`, defined as `compare(a, b) > 0`. Append and sift up for insertion. For removal, save the root, move the final element to index `0`, shrink with `pop()`, and sift against the higher-priority of both children.

### 10. Visualization
```text
push 10 into [8, 7, 5, 3]:
[8, 7, 5, 3, 10]
[8, 10, 5, 3, 7]
[10, 8, 5, 3, 7]

pop from [10, 8, 5, 3, 7]:
[7, 8, 5, 3]
[8, 7, 5, 3]
```

### 11. Step-by-Step Dry Run
| Operation | One valid internal array | Returned value |
|---|---|---:|
| construct from `[3, 8, 5, 8]` | `[8, 8, 5, 3]` | — |
| `peek()` | `[8, 8, 5, 3]` | `8` |
| `pop()` | `[8, 3, 5]` | `8` |
| `push(10)` | `[10, 8, 5, 3]` | — |

Comparator-equal references may occupy different valid positions.

### 12. TypeScript Solution
```typescript
type Comparator<T> = (left: T, right: T) => number;

class MaxHeap<T> {
    private readonly values: T[];

    constructor(
        initialValues: readonly T[],
        private readonly compare: Comparator<T>,
    ) {
        this.values = [...initialValues];
        for (let index = Math.floor(this.values.length / 2) - 1; index >= 0; index -= 1) {
            this.siftDown(index);
        }
    }

    get size(): number {
        return this.values.length;
    }

    peek(): T | undefined {
        return this.values[0];
    }

    push(value: T): void {
        this.values.push(value);
        this.siftUp(this.values.length - 1);
    }

    pop(): T | undefined {
        if (this.values.length === 0) return undefined;

        const root = this.values[0]!;
        const last = this.values.pop()!;
        if (this.values.length > 0) {
            this.values[0] = last;
            this.siftDown(0);
        }
        return root;
    }

    toArray(): T[] {
        return [...this.values];
    }

    private higherPriority(left: T, right: T): boolean {
        return this.compare(left, right) > 0;
    }

    private siftUp(startIndex: number): void {
        let index = startIndex;
        while (index > 0) {
            const parent = Math.floor((index - 1) / 2);
            if (!this.higherPriority(this.values[index]!, this.values[parent]!)) break;
            [this.values[index], this.values[parent]] = [this.values[parent]!, this.values[index]!];
            index = parent;
        }
    }

    private siftDown(startIndex: number): void {
        let index = startIndex;
        while (true) {
            const left = index * 2 + 1;
            if (left >= this.values.length) return;

            const right = left + 1;
            let bestChild = left;
            if (
                right < this.values.length &&
                this.higherPriority(this.values[right]!, this.values[left]!)
            ) {
                bestChild = right;
            }

            if (!this.higherPriority(this.values[bestChild]!, this.values[index]!)) return;
            [this.values[index], this.values[bestChild]] = [
                this.values[bestChild]!,
                this.values[index]!,
            ];
            index = bestChild;
        }
    }
}
```

### 13. Code Walkthrough
`higherPriority` is the semantic boundary: positive ascending comparison means the left value belongs closer to the max root. Both sift methods use that helper, so there is no second comparator reversal. The constructor and `toArray` establish array ownership, while swaps and returns preserve each stored object's identity.

### 14. Complexity Analysis
The constructor performs `O(n)` bottom-up heap construction after an `O(n)` copy. `push` and `pop` take `O(log n)` worst-case time and `O(1)` auxiliary space. `peek` and `size` are `O(1)`. `toArray` takes `O(n)` time and returns an `O(n)` copy.

### 15. Edge Cases
- Empty `peek()` and `pop()` return exactly `undefined`.
- A single value can be pushed, inspected, and popped normally.
- Negative values work under a valid ascending comparator.
- Duplicates remain separate heap elements.
- Comparator-equal objects retain identity but have unspecified removal order.
- Initial-array and snapshot-array mutations cannot change internal storage.
- Comparator results need not be exactly `-1`, `0`, and `1`.

### 16. Common Mistakes
- Supplying a descending comparator and reversing max priority twice.
- Reusing the foundation convention without documenting this problem's API difference.
- Retaining or exposing the caller's mutable array.
- Calling `shift()` for root removal.
- Choosing the left child before checking whether the right child is greater.
- Claiming stable behavior for comparator-equal values.
- Describing repeated insertion construction as `O(n)`.

### 17. Interview Follow-ups
1. Accept the foundation's direct priority comparator instead. 2. Implement `replaceRoot`. 3. Add explicit stable ties. 4. Build a size-`k` max heap. 5. Explain max-heap use in ascending heap sort.

### 18. Variations
- Max-priority queue. - Tournament queue. - D-ary max heap. - Indexed max heap. - Double-ended priority queue.

### 19. Pattern Connections
Max heaps retain the lower `k` values for kth-smallest selection and supply the extraction structure for ascending heap sort. The mechanics mirror a min heap, but keeping comparator meaning explicit avoids subtle inversions when code moves between generic priority APIs and ordinary value-order APIs.

### 20. NestJS Project Connection
For an endpoint such as `POST /api/dsa/heaps/max`, choose an approved ascending comparator on the server and copy DTO arrays before constructing or mutating the heap. Bound input and response sizes. Cap comparison or swap traces and expose a truncation flag rather than recording every full array state; never treat incidental equal-priority order as an API guarantee.

### 21. Mini Challenge
Implement `drain(): T[]` that removes and returns all stored values in nonincreasing comparator order. Preserve value identities, leave the heap empty, document that equal-order output is unspecified, and derive the `O(n log n)` time and `O(n)` returned-output bounds.

## Problem 178 — Heapify an Array

### 1. Problem Statement
Implement `heapifyInPlace<T>(values: T[], compare: Comparator<T>): T[]`. A negative comparator result means the left value has higher priority and should be closer to the root. Rearrange the caller-owned array into a valid binary heap, mutate that exact array, and return the exact same reference. Use bottom-up heap construction in `O(n)` time and `O(1)` auxiliary space.

The comparator's sign alone matters and it must define a consistent total preorder. Duplicates are accepted, equal-priority positions are unspecified, and object identities are preserved. If the comparator throws, the operation provides no atomicity guarantee: `values` may already be partially rearranged.

### 2. Example
```text
values = [9, 4, 7, 1, 3]
compare = (left, right) => left - right

returned = heapifyInPlace(values, compare)
one valid result: [1, 3, 7, 4, 9]

returned === values -> true
root 1 has no lower-valued child
every other parent obeys the same rule
```

The exact valid heap layout need not be unique.

### 3. Understand the Problem
Heapify does not sort the array. It establishes the local parent-child priority invariant over the dense array layout. Under a negative-is-higher-priority comparator, each child must satisfy `compare(child, parent) >= 0` after completion.

#### In Simple Terms
Rearrange the same array so no child deserves to be above its parent.

### 4. Real-World Analogy
Imagine seats already arranged as a complete tournament bracket. Heapifying does not rank every participant from first to last; it swaps participants within the existing seats until every parent seat holds someone at least as highly prioritized as its children.

### 5. Key Observation
Leaves already satisfy the heap rule because they have no children. Starting at the last parent and moving backward means both child subtrees are heaps before their parent is sifted down. This dependency order is what makes one pass sufficient.

### 6. Pattern
```text
Pattern: Bottom-Up Sift-Down
Why this pattern applies: leaves are solved subheaps and parents depend on children
How to recognize it: transform a complete-tree array into a heap in place
Similar problems: heap constructors, heap sort build phase, priority-queue bulk load
```

### 7. Brute Force
Copy the original references into a snapshot, clear the same caller array, and insert each snapshot value back through ordinary sift-up. The final array is still the original object, but the snapshot uses `O(n)` auxiliary space and repeated insertion takes `O(n log n)` worst-case time.

#### Pseudocode
```text
snapshot = shallow copy of values
clear values without replacing its reference
for each value in snapshot:
        append value to values
        sift that new value upward
return values
```

#### Brute-Force TypeScript
```typescript
type Comparator<T> = (left: T, right: T) => number;

function heapifyByInsertionInPlace<T>(
    values: T[],
    compare: Comparator<T>,
): T[] {
    const snapshot = [...values];
    values.length = 0;

    for (const value of snapshot) {
        values.push(value);
        let index = values.length - 1;

        while (index > 0) {
            const parent = Math.floor((index - 1) / 2);
            if (compare(values[index]!, values[parent]!) >= 0) break;
            [values[index], values[parent]] = [values[parent]!, values[index]!];
            index = parent;
        }
    }

    return values;
}
```

#### Visualization
```text
snapshot: [9, 4, 7, 1, 3]
same array after clear: []

insert 9 -> [9]
insert 4 -> [4, 9]
insert 7 -> [4, 9, 7]
insert 1 -> [1, 4, 7, 9]
insert 3 -> [1, 3, 7, 9, 4]
```

#### Dry Run
The snapshot holds the original value references. Clearing sets only the original array's length to zero; it does not assign a new array. Each insertion restores one reference and repairs the new leaf-to-root path. If comparison throws midway, the same array can contain only a partially rebuilt prefix, which is permitted by the non-atomic contract.

#### Brute-Force Complexity
Copying and clearing take `O(n)` time, and `n` sift-up insertions take `O(n log n)` worst-case time. The shallow snapshot uses `O(n)` auxiliary space. The returned reference is the input array itself, and no value objects are cloned.

### 8. Why Brute Force Is Not Ideal
Repeated insertion walks many values through upper levels that can instead be repaired in place. Bottom-up heapify skips every leaf and gives most internal nodes only a very short possible descent, reducing total work to linear time.

### 9. Optimal Approach
Begin at `floor(n / 2) - 1`, the last index that can have a child. For each parent moving down to index `0`, repeatedly choose the higher-priority child by comparing both children. Swap only when that child outranks the current value. All operations occur inside the caller's array.

### 10. Visualization
```text
start: [9, 4, 7, 1, 3]

last parent index = 1
index 1: children 1 and 3 -> [9, 1, 7, 4, 3]
index 0: children 1 and 7 -> [1, 9, 7, 4, 3]
continue index 1: children 4 and 3, choose 3
result: [1, 3, 7, 4, 9]
```

### 11. Step-by-Step Dry Run
| Parent being repaired | Higher-priority child | Array after repair |
|---:|---:|---|
| index `1`, value `4` | value `1` | `[9, 1, 7, 4, 3]` |
| index `0`, value `9` | value `1` | `[1, 9, 7, 4, 3]` |
| continuing at index `1` | value `3` | `[1, 3, 7, 4, 9]` |

Every parent now has priority at least as high as each existing child.

### 12. TypeScript Solution
```typescript
type Comparator<T> = (left: T, right: T) => number;

function heapifyInPlace<T>(
    values: T[],
    compare: Comparator<T>,
): T[] {
    const siftDown = (startIndex: number): void => {
        let index = startIndex;

        while (true) {
            const left = index * 2 + 1;
            if (left >= values.length) return;

            const right = left + 1;
            let bestChild = left;
            if (right < values.length && compare(values[right]!, values[left]!) < 0) {
                bestChild = right;
            }

            if (compare(values[bestChild]!, values[index]!) >= 0) return;
            [values[index], values[bestChild]] = [values[bestChild]!, values[index]!];
            index = bestChild;
        }
    };

    for (let index = Math.floor(values.length / 2) - 1; index >= 0; index -= 1) {
        siftDown(index);
    }

    return values;
}
```

### 13. Code Walkthrough
The nested helper closes over the one mutable input array and allocates no proportional workspace. It compares right against left before selecting a child, then compares that best child against the current parent. The outer loop processes only internal nodes in reverse level order. Returning `values` preserves exact array identity.

### 14. Complexity Analysis
Bottom-up heapify takes `O(n)` time: nodes at height `k` are exponentially fewer, so the sum of possible downward moves is linear. It uses `O(1)` auxiliary space because only indexes and swap temporaries are stored. The input array is also the output, and value objects are neither allocated nor cloned.

### 15. Edge Cases
- Empty and one-element arrays return immediately with the same reference.
- Two elements may require one swap.
- Already heapified arrays remain valid, though comparisons may still occur.
- Duplicate and comparator-equal objects are all retained.
- Equal-priority arrangement is unspecified.
- A descending priority comparator legitimately creates max-oriented behavior.
- If the comparator throws, the array may be partially rearranged.

### 16. Common Mistakes
- Returning a newly allocated heap array instead of the exact input reference.
- Calling the optimized algorithm `O(n log n)` by multiplying one worst-case sift by every node.
- Starting from a leaf rather than the last parent.
- Comparing only one child during sift-down.
- Treating comparator equality as permission to remove a duplicate.
- Promising rollback when a comparator throws.

### 17. Interview Follow-ups
1. Prove the linear-time bound. 2. Heapify only a prefix. 3. Validate the heap invariant. 4. Record a bounded swap trace. 5. Compare bottom-up construction with repeated insertion on nearly sorted input.

### 18. Variations
- Max-oriented heapify. - D-ary heapify. - Heapify a subrange. - Floyd's hole-based optimization. - Immutable heap construction.

### 19. Pattern Connections
Bottom-up dependency order also appears when computing subtree summaries from leaves toward roots. Heap constructors use this function after copying caller input, and heap sort uses the same sift-down logic while progressively shrinking the active heap prefix.

### 20. NestJS Project Connection
For `POST /api/dsa/heaps/heapify`, validate and bound the DTO, choose a fixed server-side comparator, then copy the DTO array before calling the mutating algorithm so request transport data is not altered. Return the copied result. Cap comparison and swap traces, and document that comparator failure is not internally atomic even if boundary validation makes such failure unlikely.

### 21. Mini Challenge
Implement `isHeap<T>(values: readonly T[], compare: Comparator<T>): boolean` without mutation. Check both children of every parent, accept duplicates, inspect only comparator signs, and achieve `O(n)` time with `O(1)` auxiliary space.

## Problem 179 — Heap Sort

### 1. Problem Statement
Implement `heapSortInPlace<T>(values: T[], compare: Comparator<T>): T[]`. The comparator defines the desired ascending order: `compare(left, right) < 0` means `left` must precede `right` in the final array. Mutate the caller's exact array and return that same reference.

Use an internal max heap, treating `compare(a, b) > 0` as “`a` belongs closer to the root.” The optimized algorithm must take `O(n log n)` time in best, average, and worst cases with `O(1)` auxiliary space. Duplicates are allowed, object identities are preserved, and the sort is unstable. The comparator must define a consistent total preorder; malformed results such as `NaN` are outside the contract.

### 2. Example
```text
values = [4, 1, 3, 2, 3]
compare = (left, right) => left - right

returned = heapSortInPlace(values, compare)
values   = [1, 2, 3, 3, 4]

returned === values -> true
```

Two comparator-equal object references may reverse relative order.

### 3. Understand the Problem
Desired ascending order and heap priority point in opposite directions during sorting. A max heap places the greatest remaining value at index `0`; swapping that root with the final position of the active prefix puts one value into its permanent sorted location. The active heap then shrinks by one.

#### In Simple Terms
Keep moving the largest unsorted value to the right end until the whole array is ordered.

### 4. Real-World Analogy
Picture arranging score cards from low to high. First organize them so the highest score is easy to reach, place that card into the last open slot, then repeat with the remaining open slots. Cards with tied scores can cross during swaps, so their arrival order is not preserved.

### 5. Key Observation
After bottom-up max-heap construction, the root is the greatest value in the active prefix. One root-end swap fixes the next suffix position. Sift-down restores the max heap using only the shorter prefix, so no secondary array is required.

### 6. Pattern
```text
Pattern: Build Max Heap, Shrink Active Prefix
Why this pattern applies: each root extraction fixes one greatest suffix value
How to recognize it: in-place comparison sorting with guaranteed O(n log n) time
Similar problems: priority-queue draining, partial heap selection, introsort fallback
```

### 7. Brute Force
Use in-place selection sort. For each destination index, scan the unsorted suffix for the earliest strictly smaller value and swap it into place. This mutates and returns the same array with constant auxiliary space, but arbitrary swaps make it unstable and the repeated scans are quadratic.

#### Pseudocode
```text
for each destination from left to right:
        smallest = destination
        scan the remaining suffix
        if a value precedes the current smallest: remember its index
        swap destination with smallest
return the same values reference
```

#### Brute-Force TypeScript
```typescript
type Comparator<T> = (left: T, right: T) => number;

function selectionSortInPlace<T>(
    values: T[],
    compare: Comparator<T>,
): T[] {
    for (let destination = 0; destination < values.length - 1; destination += 1) {
        let smallest = destination;

        for (let index = destination + 1; index < values.length; index += 1) {
            if (compare(values[index]!, values[smallest]!) < 0) {
                smallest = index;
            }
        }

        if (smallest !== destination) {
            [values[destination], values[smallest]] = [
                values[smallest]!,
                values[destination]!,
            ];
        }
    }

    return values;
}
```

#### Visualization
```text
[4, 1, 3a, 2, 3b]
 ^ find 1 and swap -> [1, 4, 3a, 2, 3b]
        ^ find 2       -> [1, 2, 3a, 4, 3b]
             ^ keep 3a   -> [1, 2, 3a, 4, 3b]
                    ^ swap   -> [1, 2, 3a, 3b, 4]

A different input can make equal elements cross; selection sort is not stable.
```

#### Dry Run
On `[4, 1, 3, 2, 3]`, the first scan selects `1`, the second selects `2`, and later scans place both `3` values before `4`. Every swap exchanges stored references rather than cloning values. The apparent order of equal values in this example is incidental and not guaranteed.

#### Brute-Force Complexity
The nested scans perform `O(n^2)` comparisons in best, average, and worst cases. Swaps use `O(1)` auxiliary space, and the input array is the output. Object values retain identity, but equal values may change relative order.

### 8. Why Brute Force Is Not Ideal
Selection sort rescans nearly the entire remaining suffix for every output position. A heap keeps the greatest remaining value at a known index and repairs that structure in logarithmic time after each extraction, improving the guaranteed bound to `O(n log n)`.

### 9. Optimal Approach
First build a max heap over the full array using bottom-up sift-down. Then iterate `end` from the last index toward `1`: swap the maximum at index `0` with `values[end]`, exclude that settled suffix position, and sift the new root down only within indexes below `end`. Choose the greater of both children under the supplied ascending comparator.

### 10. Visualization
```text
input:          [4, 1, 3, 2]
build max heap: [4, 2, 3, 1]

place 4: [1, 2, 3 | 4]
repair:  [3, 2, 1 | 4]
place 3: [1, 2 | 3, 4]
repair:  [2, 1 | 3, 4]
place 2: [1 | 2, 3, 4]
```

### 11. Step-by-Step Dry Run
| Active end | Root moved to index | Array after repair |
|---:|---:|---|
| `4` | `4` | `[3, 3, 1, 2, 4]` |
| `3` | `3` | `[3, 2, 1, 3, 4]` |
| `2` | `2` | `[2, 1, 3, 3, 4]` |
| `1` | `1` | `[1, 2, 3, 3, 4]` |

The suffix to the right of the active end is already in final order and is never touched again.

### 12. TypeScript Solution
```typescript
type Comparator<T> = (left: T, right: T) => number;

function heapSortInPlace<T>(
    values: T[],
    compare: Comparator<T>,
): T[] {
    const siftDown = (startIndex: number, endExclusive: number): void => {
        let index = startIndex;

        while (true) {
            const left = index * 2 + 1;
            if (left >= endExclusive) return;

            const right = left + 1;
            let greatestChild = left;
            if (
                right < endExclusive &&
                compare(values[right]!, values[left]!) > 0
            ) {
                greatestChild = right;
            }

            if (compare(values[greatestChild]!, values[index]!) <= 0) return;
            [values[index], values[greatestChild]] = [
                values[greatestChild]!,
                values[index]!,
            ];
            index = greatestChild;
        }
    };

    for (let index = Math.floor(values.length / 2) - 1; index >= 0; index -= 1) {
        siftDown(index, values.length);
    }

    for (let end = values.length - 1; end > 0; end -= 1) {
        [values[0], values[end]] = [values[end]!, values[0]!];
        siftDown(0, end);
    }

    return values;
}
```

### 13. Code Walkthrough
`siftDown` receives `endExclusive`, so settled suffix values are outside the heap. It chooses a comparator-positive child because the internal structure is a max heap, then swaps only when that child is greater than the current parent. The build loop heapifies once; the extraction loop fixes one final position per iteration. Every operation is an in-array reference swap.

### 14. Complexity Analysis
Bottom-up max-heap construction takes `O(n)`. The algorithm then performs `n - 1` extractions, each with at most `O(log n)` sift-down work, so total time is `O(n log n)` in best, average, and worst cases. It uses `O(1)` auxiliary space and returns the input reference. Heap sort is not stable.

### 15. Edge Cases
- Empty and one-element arrays return the exact same reference unchanged.
- Already sorted and reverse-sorted inputs still take `O(n log n)` time.
- Duplicate and comparator-equal objects are retained but may be reordered.
- A comparator for descending desired output legitimately reverses the final order.
- Large comparator magnitudes are valid because only signs are inspected.
- `NaN` comparator results, nontransitive order, and stateful comparison are outside the contract.
- A throwing comparator may leave the array partially rearranged.

### 16. Common Mistakes
- Building a min heap and producing descending output accidentally.
- Sorting a copy and violating the exact same-reference contract.
- Including settled suffix values in later sift-down operations.
- Choosing the left child without comparing the right child.
- Claiming heap sort or selection sort is stable.
- Reporting bottom-up heap construction as `O(n log n)`.
- Treating `NaN` as a valid negative, zero, or positive comparator result.

### 17. Interview Follow-ups
1. Produce descending output. 2. Sort only a subrange. 3. Stop after the largest `k` values are placed. 4. Compare cache behavior with quicksort. 5. Explain why stability cannot be inferred from tie handling alone.

### 18. Variations
- Partial heap sort. - Top-k suffix extraction. - D-ary heap sort. - Introsort heap fallback. - External-memory priority sorting.

### 19. Pattern Connections
Heap sort combines Problem 178's bottom-up construction with repeated max-root removal. Its shrinking-prefix boundary resembles partition-based sorting, but its complete-tree invariant provides a deterministic worst-case bound without recursion or proportional auxiliary storage.

### 20. NestJS Project Connection
For `POST /api/dsa/heaps/sort`, validate a bounded DTO and select a fixed server-side ascending comparator. Copy the DTO array before calling `heapSortInPlace` so transport input is not mutated, then serialize the sorted copy. Cap comparison and swap traces, expose truncation explicitly, and reject non-finite numeric values such as `NaN` when the chosen numeric comparator cannot order them.

### 21. Mini Challenge
Implement `heapSortPrefixInPlace<T>(values, length, compare)` so only indexes from `0` through `length - 1` are sorted and the remaining suffix is untouched. Return the same array, validate the length before mutation, and retain `O(length log length)` time with `O(1)` auxiliary space.

## Problem 180 — Kth Largest Element

### 1. Problem Statement
Implement `kthLargest(values: readonly number[], k: number): number`. Treat duplicates as separate sorted ranks and do not mutate `values`.

Validate `k` first: it must be a positive safe integer no greater than `values.length`; otherwise throw `RangeError`. Then validate every input value before performing the algorithm or returning: each must be a safe integer, or the function throws `RangeError`. Consequently, an empty input has no valid `k`. The optimized solution must use a bounded min heap of size `k` in `O(n log k)` time and `O(k)` auxiliary space.

### 2. Example
```text
values = [7, 4, 7, 1, 9]
k = 3

descending ranks = [9, 7, 7, 4, 1]
result = 7
```

The two `7` values occupy the second and third ranks.

### 3. Understand the Problem
The kth largest value is the value at index `k - 1` after conceptual descending sorting. Only that boundary rank is needed; the other values do not need to be fully ordered. Validation order is observable, so an invalid `k` is rejected before any value is inspected.

#### In Simple Terms
Keep the `k` biggest numbers seen so far; the smallest of those winners is the answer.

### 4. Real-World Analogy
Imagine a podium with `k` qualifying places. The lowest score currently on the podium is the cutoff. A new score replaces that cutoff only when it is larger; ties remain separate competitors and can occupy separate places.

### 5. Key Observation
A min heap containing the largest `k` values exposes the smallest retained value at its root. That root is the kth largest overall. Once the heap is full, an incoming value can affect the answer only when it strictly beats the root boundary.

### 6. Pattern
```text
Pattern: Bounded Min Heap
Why this pattern applies: only the largest k candidates matter
How to recognize it: one descending rank is requested without full ordering
Similar problems: kth smallest, top-k frequency, streaming top-k values
```

### 7. Brute Force
Validate the complete contract, copy the input, sort the copy in descending numeric order, and return index `k - 1`. Copying preserves the readonly input.

#### Pseudocode
```text
validate k
validate every value
copy values and sort descending
return copy[k - 1]
```

#### Brute-Force TypeScript
```typescript
function kthLargestBySorting(
    values: readonly number[],
    k: number,
): number {
    if (!Number.isSafeInteger(k) || k <= 0 || k > values.length) {
        throw new RangeError("k must be a positive safe integer within values");
    }

    for (const value of values) {
        if (!Number.isSafeInteger(value)) {
            throw new RangeError("every value must be a safe integer");
        }
    }

    const sorted = [...values].sort((left, right) => right - left);
    return sorted[k - 1]!;
}
```

#### Visualization
```text
input:      [7, 4, 7, 1, 9]
copy:       [7, 4, 7, 1, 9]
descending: [9, 7, 7, 4, 1]
index k-1:          ^ 7
```

#### Dry Run
For `k = 3`, sorting places `9` first and both copies of `7` next. Index `2` therefore contains `7`. The original array remains unchanged because only the copy is sorted.

#### Brute-Force Complexity
Copying and validation take `O(n)` time. Sorting takes `O(n log n)` time, and the copied array uses `O(n)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
Full sorting establishes the relative order of all `n` values even though only one rank is requested. A size-`k` heap retains only candidates that can still determine the answer.

### 9. Optimal Approach
After all validation succeeds, create an empty min heap. Push values until it contains `k` entries. For every later value, replace the root only when the incoming value is greater than that smallest retained boundary. The root after the scan is the kth largest value.

### 10. Visualization
```text
values = [7, 4, 7, 1, 9], k = 3

retain 7, 4, 7 -> min-heap root 4
incoming 1      -> does not beat 4; ignore
incoming 9      -> beats 4; replace root
retained values -> {7, 7, 9}; root 7
```

### 11. Step-by-Step Dry Run
| Incoming | Action | Retained boundary |
|---:|---|---:|
| `7` | Push | `7` |
| `4` | Push | `4` |
| `7` | Push | `4` |
| `1` | Ignore | `4` |
| `9` | Replace root | `7` |

The final root is `7`, the third largest value.

### 12. TypeScript Solution
```typescript
function kthLargest(
    values: readonly number[],
    k: number,
): number {
    if (!Number.isSafeInteger(k) || k <= 0 || k > values.length) {
        throw new RangeError("k must be a positive safe integer within values");
    }

    for (const value of values) {
        if (!Number.isSafeInteger(value)) {
            throw new RangeError("every value must be a safe integer");
        }
    }

    class MinHeap {
        private readonly heap: number[] = [];

        get size(): number {
            return this.heap.length;
        }

        peek(): number {
            return this.heap[0]!;
        }

        push(value: number): void {
            this.heap.push(value);
            let index = this.heap.length - 1;

            while (index > 0) {
                const parent = Math.floor((index - 1) / 2);
                if (this.heap[parent]! <= this.heap[index]!) break;
                [this.heap[parent], this.heap[index]] = [
                    this.heap[index]!,
                    this.heap[parent]!,
                ];
                index = parent;
            }
        }

        replaceRoot(value: number): void {
            this.heap[0] = value;
            let index = 0;

            while (true) {
                const left = index * 2 + 1;
                if (left >= this.heap.length) return;

                const right = left + 1;
                let smallest = left;
                if (right < this.heap.length && this.heap[right]! < this.heap[left]!) {
                    smallest = right;
                }

                if (this.heap[index]! <= this.heap[smallest]!) return;
                [this.heap[index], this.heap[smallest]] = [
                    this.heap[smallest]!,
                    this.heap[index]!,
                ];
                index = smallest;
            }
        }
    }

    const candidates = new MinHeap();
    for (const value of values) {
        if (candidates.size < k) {
            candidates.push(value);
        } else if (value > candidates.peek()) {
            candidates.replaceRoot(value);
        }
    }

    return candidates.peek();
}
```

### 13. Code Walkthrough
The function validates `k` before traversing `values`, then completes value validation before allocating or changing heap state. The local `MinHeap` keeps its least value at index `0`. `replaceRoot` is called only when the incoming value strictly exceeds that boundary; an equal value cannot improve the retained multiset.

### 14. Complexity Analysis
Validation takes `O(n)`. Each value performs at most one `O(log k)` heap operation, for `O(n log k)` total time and `O(k)` auxiliary space. The input is never mutated. For `k = 1`, the heap tracks the maximum; for `k = n`, it retains every value and exposes the minimum.

### 15. Edge Cases
- `k = 1` returns the maximum value.
- `k = values.length` returns the minimum value.
- Duplicates occupy separate ranks rather than being deduplicated.
- Negative safe integers and safe-integer boundary values are accepted.
- Empty input, zero, negative, fractional, unsafe, or oversized `k` throws `RangeError`.
- Any non-safe-integer value throws `RangeError` after `k` has passed validation.
- Object identity is irrelevant because the contract accepts and returns only numbers.

### 16. Common Mistakes
- Validating values before `k` and violating the specified error order.
- Mutating the readonly input by sorting it directly.
- Keeping a max heap, which exposes the wrong boundary for largest-`k` retention.
- Replacing the root when an incoming value is smaller or equal.
- Removing duplicates before ranking them.
- Returning before every value has been validated.

### 17. Interview Follow-ups
1. Return the largest `k` values in deterministic order. 2. Process an unbounded stream. 3. Support updates by identifier. 4. Compare the heap with quickselect. 5. Explain how duplicate ranks affect the answer.

### 18. Variations
- Kth distinct largest. - Streaming kth largest. - Top-k sorted output. - Sliding-window kth largest. - Comparator-based object selection.

### 19. Pattern Connections
This is the bounded-candidate pattern introduced in the chapter theory and reused by top-k frequency selection. Quickselect can achieve expected `O(n)` time, but this problem deliberately uses a heap because it belongs to the heaps and priority queues chapter and the bounded structure extends naturally to streams.

### 20. NestJS Project Connection
For `POST /api/dsa/heaps/kth-largest`, validate `k` first, then validate and cap the DTO array length and every numeric value. Reject requests above server limits rather than allocating an oversized heap. The service can pass the readonly DTO data directly because the function does not mutate it; map contract failures to a stable client error response.

### 21. Mini Challenge
Implement `largestK(values, k): number[]` with the same validation contract. Use a bounded min heap, then return exactly `k` values in descending order without mutating the input.

## Problem 181 — Kth Smallest Element

### 1. Problem Statement
Implement `kthSmallest(values: readonly number[], k: number): number`. Treat duplicates as separate sorted ranks and do not mutate `values`.

Validate `k` first: it must be a positive safe integer no greater than `values.length`; otherwise throw `RangeError`. Then validate every input value before performing the algorithm or returning: each must be a safe integer, or the function throws `RangeError`. Consequently, an empty input has no valid `k`. The optimized solution must use a bounded max heap of size `k` in `O(n log k)` time and `O(k)` auxiliary space.

### 2. Example
```text
values = [5, 2, 8, 2, 6]
k = 3

ascending ranks = [2, 2, 5, 6, 8]
result = 5
```

The two `2` values occupy the first and second ranks.

### 3. Understand the Problem
The kth smallest value is the value at index `k - 1` after conceptual ascending sorting. Only that boundary rank is needed. Validation order is part of the contract: reject an invalid `k` before inspecting values, then validate all values before algorithm work or return.

#### In Simple Terms
Keep the `k` smallest numbers seen so far; the largest of those winners is the answer.

### 4. Real-World Analogy
Imagine `k` seats reserved for the lowest bids. The highest bid currently seated is the cutoff. A lower incoming bid replaces that cutoff, while a higher or equal bid cannot improve the selected group.

### 5. Key Observation
A max heap containing the smallest `k` values exposes the largest retained value at its root. That root is the kth smallest overall. Once full, the heap changes only when an incoming value strictly beats the root by being smaller.

### 6. Pattern
```text
Pattern: Bounded Max Heap
Why this pattern applies: only the smallest k candidates matter
How to recognize it: one ascending rank is requested without full ordering
Similar problems: kth largest, bottom-k sampling, streaming thresholds
```

### 7. Brute Force
Validate the complete contract, copy the input, sort the copy in ascending numeric order, and return index `k - 1`. The original readonly array is never changed.

#### Pseudocode
```text
validate k
validate every value
copy values and sort ascending
return copy[k - 1]
```

#### Brute-Force TypeScript
```typescript
function kthSmallestBySorting(
    values: readonly number[],
    k: number,
): number {
    if (!Number.isSafeInteger(k) || k <= 0 || k > values.length) {
        throw new RangeError("k must be a positive safe integer within values");
    }

    for (const value of values) {
        if (!Number.isSafeInteger(value)) {
            throw new RangeError("every value must be a safe integer");
        }
    }

    const sorted = [...values].sort((left, right) => left - right);
    return sorted[k - 1]!;
}
```

#### Visualization
```text
input:     [5, 2, 8, 2, 6]
copy:      [5, 2, 8, 2, 6]
ascending: [2, 2, 5, 6, 8]
index k-1:         ^ 5
```

#### Dry Run
For `k = 3`, sorting places the two copies of `2` in the first two ranks and `5` in the third. Index `2` therefore contains `5`; the caller's array still has its original order.

#### Brute-Force Complexity
Copying and validation take `O(n)` time. Sorting takes `O(n log n)` time, and the copied array uses `O(n)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
Sorting computes every rank even though only one is requested. A bounded heap discards values as soon as they cannot belong to the smallest `k`, reducing work when `k` is much smaller than `n`.

### 9. Optimal Approach
After validation, push values into a max heap until it reaches size `k`. For each remaining value, replace the root only when the incoming value is smaller than that largest retained boundary. The root at the end is the kth smallest value.

### 10. Visualization
```text
values = [5, 2, 8, 2, 6], k = 3

retain 5, 2, 8 -> max-heap root 8
incoming 2      -> beats 8; replace root
incoming 6      -> does not beat 5; ignore
retained values -> {2, 2, 5}; root 5
```

### 11. Step-by-Step Dry Run
| Incoming | Action | Retained boundary |
|---:|---|---:|
| `5` | Push | `5` |
| `2` | Push | `5` |
| `8` | Push | `8` |
| `2` | Replace root | `5` |
| `6` | Ignore | `5` |

The final root is `5`, the third smallest value.

### 12. TypeScript Solution
```typescript
function kthSmallest(
    values: readonly number[],
    k: number,
): number {
    if (!Number.isSafeInteger(k) || k <= 0 || k > values.length) {
        throw new RangeError("k must be a positive safe integer within values");
    }

    for (const value of values) {
        if (!Number.isSafeInteger(value)) {
            throw new RangeError("every value must be a safe integer");
        }
    }

    class MaxHeap {
        private readonly heap: number[] = [];

        get size(): number {
            return this.heap.length;
        }

        peek(): number {
            return this.heap[0]!;
        }

        push(value: number): void {
            this.heap.push(value);
            let index = this.heap.length - 1;

            while (index > 0) {
                const parent = Math.floor((index - 1) / 2);
                if (this.heap[parent]! >= this.heap[index]!) break;
                [this.heap[parent], this.heap[index]] = [
                    this.heap[index]!,
                    this.heap[parent]!,
                ];
                index = parent;
            }
        }

        replaceRoot(value: number): void {
            this.heap[0] = value;
            let index = 0;

            while (true) {
                const left = index * 2 + 1;
                if (left >= this.heap.length) return;

                const right = left + 1;
                let largest = left;
                if (right < this.heap.length && this.heap[right]! > this.heap[left]!) {
                    largest = right;
                }

                if (this.heap[index]! >= this.heap[largest]!) return;
                [this.heap[index], this.heap[largest]] = [
                    this.heap[largest]!,
                    this.heap[index]!,
                ];
                index = largest;
            }
        }
    }

    const candidates = new MaxHeap();
    for (const value of values) {
        if (candidates.size < k) {
            candidates.push(value);
        } else if (value < candidates.peek()) {
            candidates.replaceRoot(value);
        }
    }

    return candidates.peek();
}
```

### 13. Code Walkthrough
The contract checks finish before heap construction. The local `MaxHeap` bubbles larger values toward its root and repairs downward after root replacement. The scan replaces that root only when an incoming number is strictly smaller, so the heap always contains the smallest `k` values seen so far.

### 14. Complexity Analysis
Validation takes `O(n)`. The bounded heap performs at most one `O(log k)` operation per value, giving `O(n log k)` total time and `O(k)` auxiliary space. For `k = 1`, it tracks the minimum; for `k = n`, it retains all values and exposes the maximum.

### 15. Edge Cases
- `k = 1` returns the minimum value.
- `k = values.length` returns the maximum value.
- Duplicate values consume separate ascending ranks.
- Negative safe integers and safe-integer boundary values are accepted.
- Empty input, zero, negative, fractional, unsafe, or oversized `k` throws `RangeError`.
- Any non-safe-integer value throws `RangeError` only after valid `k` validation.
- Object identity is irrelevant because this numeric contract has no object inputs or outputs.

### 16. Common Mistakes
- Inspecting values before validating `k`.
- Sorting the readonly input rather than a copy in the brute-force version.
- Using a min heap and exposing the smallest retained value instead of the boundary.
- Replacing the root when the incoming value is larger or equal.
- Deduplicating values before assigning ranks.
- Returning as soon as the answer seems known without validating later input values.

### 17. Interview Follow-ups
1. Return the smallest `k` values in order. 2. Accept a stream rather than an array. 3. Find the kth distinct smallest value. 4. Compare with quickselect. 5. Support a changing value window.

### 18. Variations
- Kth distinct smallest. - Streaming kth smallest. - Bottom-k sorted output. - Sliding-window kth smallest. - Comparator-based records.

### 19. Pattern Connections
Kth smallest is the orientation-reversed form of Problem 180: the retained set changes from largest values to smallest values, and the heap root remains the worst retained boundary. Quickselect is a valid expected-linear alternative, but the bounded max heap is used here because this is the heap chapter and the same invariant supports streaming input.

### 20. NestJS Project Connection
For `POST /api/dsa/heaps/kth-smallest`, validate `k` before the values, cap both array length and accepted `k`, and reject non-safe integers at the DTO or service boundary. Passing validated readonly data into the service is safe because neither implementation mutates the request array. Convert `RangeError` into a consistent client-facing validation response.

### 21. Mini Challenge
Implement `smallestK(values, k): number[]` with the same validation and immutability contract. Use a bounded max heap and return exactly `k` values in ascending order.

## Problem 182 — Top K Frequent Elements

### 1. Problem Statement
Implement `topKFrequent(values: readonly number[], k: number): number[]`. Validate `k` first; it must be a positive safe integer or the function throws `RangeError`. Then validate every value before producing a result; each must be a safe integer or the function throws `RangeError`. Do not mutate `values`.

For valid `k`, empty input returns `[]`. Otherwise return exactly `min(k, distinctCount)` distinct values ordered by frequency descending, then by numeric value ascending when frequencies tie. The optimized solution must combine a frequency `Map` with a bounded heap containing at most `k` entries.

### 2. Example
```text
values = [4, 1, 4, 2, 2, 3]
k = 3

frequencies: 4 -> 2, 2 -> 2, 1 -> 1, 3 -> 1
result: [2, 4, 1]
```

Values `2` and `4` tie at frequency two, so smaller value `2` appears first. Between the frequency-one values, `1` wins the final retained position.

### 3. Understand the Problem
There are two orderings to encode. Better output entries have higher frequency, with smaller numeric value breaking ties. A bounded heap should expose the opposite boundary: its root is the worst retained entry, meaning lower frequency or, at equal frequency, larger numeric value.

#### In Simple Terms
Count each number, keep only the best `k` counts, then arrange those winners in the required order.

### 4. Real-World Analogy
Imagine a leaderboard with at most `k` rows. More votes rank higher, and a smaller contestant number wins a vote tie. The last-place retained row must be easy to remove whenever a better contestant arrives.

### 5. Key Observation
The frequency table reduces `n` occurrences to `d` distinct entries. A heap of at most `k` entries can discard losers while scanning those `d` entries. Making the root the worst retained entry turns replacement into one direct comparison.

### 6. Pattern
```text
Pattern: Count, Retain with Worst-First Heap, Final Sort
Why this pattern applies: rank d aggregates but return only the best k
How to recognize it: top-k output with deterministic secondary ordering
Similar problems: top-k words, heavy hitters, bounded leaderboards
```

### 7. Brute Force
Build a `Map` from value to frequency, convert all `d` entries into records, sort every record by the final output order, take the first `min(k, d)`, and project the values.

#### Pseudocode
```text
validate k
validate every value
count frequencies in a map
sort all entries by frequency descending, then value ascending
take the first min(k, distinct count) values
```

#### Brute-Force TypeScript
```typescript
function topKFrequentBySorting(
    values: readonly number[],
    k: number,
): number[] {
    if (!Number.isSafeInteger(k) || k <= 0) {
        throw new RangeError("k must be a positive safe integer");
    }

    for (const value of values) {
        if (!Number.isSafeInteger(value)) {
            throw new RangeError("every value must be a safe integer");
        }
    }

    const frequencies = new Map<number, number>();
    for (const value of values) {
        frequencies.set(value, (frequencies.get(value) ?? 0) + 1);
    }

    return [...frequencies.entries()]
        .sort(([leftValue, leftFrequency], [rightValue, rightFrequency]) =>
            rightFrequency - leftFrequency || leftValue - rightValue,
        )
        .slice(0, Math.min(k, frequencies.size))
        .map(([value]) => value);
}
```

#### Visualization
```text
all entries:      [(4,2), (1,1), (2,2), (3,1)]
fully sorted:     [(2,2), (4,2), (1,1), (3,1)]
take first k = 3: [  2,     4,     1  ]
```

#### Dry Run
Counting produces four distinct entries. Full sorting first groups frequency `2` ahead of frequency `1`, then orders each tied group by value. Slicing three records yields `[2, 4, 1]`. Neither counting nor sorting the newly allocated entry array changes `values`.

#### Brute-Force Complexity
Counting takes `O(n)` time and sorting all `d` distinct entries takes `O(d log d)`, for `O(n + d log d)` total time. The map and entry array use `O(d)` auxiliary space; the returned array has at most `min(k, d)` values.

### 8. Why Brute Force Is Not Ideal
When `k` is much smaller than `d`, fully sorting every distinct entry performs unnecessary comparisons. A bounded heap keeps only entries that can still appear in the result.

### 9. Optimal Approach
Build the frequency map after validation. Scan its entries while maintaining a heap of at most `k` records whose root is the worst retained record: lower frequency is worse, and for equal frequency a larger value is worse. Push until full; afterward, replace the root only when the incoming record is better. Finally sort the retained records by frequency descending and value ascending so the returned order is deterministic.

### 10. Visualization
```text
k = 3; entries arrive as (4,2), (1,1), (2,2), (3,1)

push first three -> retained {(4,2), (1,1), (2,2)}
worst root       -> (1,1)
incoming (3,1)   -> same frequency but larger value, so worse; ignore
final sort       -> [(2,2), (4,2), (1,1)]
result           -> [2, 4, 1]
```

### 11. Step-by-Step Dry Run
| Entry | Heap action | Worst retained entry |
|---|---|---|
| `(4, 2)` | Push | `(4, 2)` |
| `(1, 1)` | Push | `(1, 1)` |
| `(2, 2)` | Push | `(1, 1)` |
| `(3, 1)` | Ignore | `(1, 1)` |

The final sort applies output order rather than relying on internal heap layout.

### 12. TypeScript Solution
```typescript
function topKFrequent(
    values: readonly number[],
    k: number,
): number[] {
    if (!Number.isSafeInteger(k) || k <= 0) {
        throw new RangeError("k must be a positive safe integer");
    }

    for (const value of values) {
        if (!Number.isSafeInteger(value)) {
            throw new RangeError("every value must be a safe integer");
        }
    }

    interface Entry {
        value: number;
        frequency: number;
    }

    const compareWorstFirst = (left: Entry, right: Entry): number => {
        if (left.frequency !== right.frequency) {
            return left.frequency - right.frequency;
        }
        return right.value - left.value;
    };

    const isBetter = (candidate: Entry, boundary: Entry): boolean =>
        candidate.frequency > boundary.frequency ||
        (candidate.frequency === boundary.frequency &&
            candidate.value < boundary.value);

    class WorstFirstHeap {
        private readonly heap: Entry[] = [];

        get size(): number {
            return this.heap.length;
        }

        peek(): Entry {
            return this.heap[0]!;
        }

        push(entry: Entry): void {
            this.heap.push(entry);
            let index = this.heap.length - 1;

            while (index > 0) {
                const parent = Math.floor((index - 1) / 2);
                if (compareWorstFirst(this.heap[index]!, this.heap[parent]!) >= 0) {
                    break;
                }
                [this.heap[index], this.heap[parent]] = [
                    this.heap[parent]!,
                    this.heap[index]!,
                ];
                index = parent;
            }
        }

        replaceRoot(entry: Entry): void {
            this.heap[0] = entry;
            let index = 0;

            while (true) {
                const left = index * 2 + 1;
                if (left >= this.heap.length) return;

                const right = left + 1;
                let worst = left;
                if (
                    right < this.heap.length &&
                    compareWorstFirst(this.heap[right]!, this.heap[left]!) < 0
                ) {
                    worst = right;
                }

                if (compareWorstFirst(this.heap[worst]!, this.heap[index]!) >= 0) {
                    return;
                }
                [this.heap[index], this.heap[worst]] = [
                    this.heap[worst]!,
                    this.heap[index]!,
                ];
                index = worst;
            }
        }

        toArray(): Entry[] {
            return [...this.heap];
        }
    }

    const frequencies = new Map<number, number>();
    for (const value of values) {
        frequencies.set(value, (frequencies.get(value) ?? 0) + 1);
    }

    const candidates = new WorstFirstHeap();
    for (const [value, frequency] of frequencies) {
        const entry: Entry = { value, frequency };
        if (candidates.size < k) {
            candidates.push(entry);
        } else if (isBetter(entry, candidates.peek())) {
            candidates.replaceRoot(entry);
        }
    }

    return candidates
        .toArray()
        .sort((left, right) =>
            right.frequency - left.frequency || left.value - right.value,
        )
        .map((entry) => entry.value);
}
```

### 13. Code Walkthrough
`compareWorstFirst` assigns heap priority to lower frequencies and then larger values, so a negative result means the left entry is worse and belongs closer to the root. `isBetter` encodes the exact inverse output preference used for replacement. The heap never exceeds `k`; `toArray()` returns a copy that is sorted into the required final order.

### 14. Complexity Analysis
Let `d` be the number of distinct values. Validation and counting take `O(n)`. Processing `d` map entries with a size-`k` bounded heap takes `O(d log k)`, and deterministically sorting the retained entries takes `O(k log k)`, for `O(n + d log k + k log k)` total time. The frequency map uses `O(d)` space and the heap actually uses `O(min(k, d))`, conventionally reported together as `O(d + k)` auxiliary space. Because the heap cannot hold more than `d` entries, the map dominates auxiliary storage.

### 15. Edge Cases
- Valid `k` with empty `values` returns `[]`.
- `k` greater than the distinct count returns every distinct value in output order.
- `k = 1` returns the highest-frequency value, choosing the smallest value on a tie.
- All values distinct are returned in ascending numeric order up to the requested count.
- Invalid `k` throws before any input value is inspected.
- Any non-safe-integer input throws before counting or returning a result.
- The input array remains unchanged.

### 16. Common Mistakes
- Making the heap root the best entry, which makes bounded replacement awkward or wrong.
- Treating a larger tied value as better instead of worse.
- Returning heap-array order without the final deterministic sort.
- Returning fewer than `min(k, d)` values or padding when `k > d`.
- Mutating the input while grouping or sorting.
- Counting before all values have passed validation.
- Claiming frequency increments can exceed exact integer representation for a JavaScript array.

### 17. Interview Follow-ups
1. Return frequencies with each value. 2. Handle string values with lexical ties. 3. Process a stream approximately. 4. Support incremental updates. 5. Compare with bucket sorting when `n` is bounded.

### 18. Variations
- Top-k frequent words. - Least-k frequent values. - Stable first-seen tie-breaking. - Approximate heavy hitters. - Sliding-window frequencies.

### 19. Pattern Connections
This problem combines hashing from the frequency-map chapter with bounded heaps from Problems 180 and 181. Unlike one-rank selection, observable output order requires a final sort of retained entries. Frequency counts remain exact: a real JavaScript array's length is bounded below `2^32`, so no per-value count can approach `Number.MAX_SAFE_INTEGER`.

### 20. NestJS Project Connection
For `POST /api/dsa/heaps/top-k-frequent`, validate positive safe-integer `k` first, then validate and cap the array length, `k`, and every value. Reject requests exceeding service limits even though `k > distinctCount` is valid within the algorithm contract. The service need not copy the DTO array because the function does not mutate it; return the already deterministic result directly.

### 21. Mini Challenge
Implement `bottomKFrequent(values, k): number[]` with the same validation and immutability rules. Return lower frequencies first and use smaller numeric value as the tie-breaker, while keeping only `k` entries during selection.

## Problem 183 — Merge K Sorted Arrays

### 1. Problem Statement
Implement `mergeKSortedArrays(arrays: readonly (readonly number[])[]): number[]`. Before producing output or beginning the merge, validate every source completely: each value must be a safe integer, and every source must be nondecreasing. Throw `RangeError` if either condition fails. Do not mutate the outer array or any source array.

Return all values in nondecreasing order, retaining duplicates. An empty outer array or an outer array containing only empty sources returns `[]`. The optimized solution must use a min heap with at most one candidate from each source. Candidate priority is deterministic: value ascending, then source index ascending, then element index ascending.

### 2. Example
```text
arrays = [
    [-3, 4, 4],
    [],
    [-3, 2, 9],
    [0, 4],
]

result = [-3, -3, 0, 2, 4, 4, 4, 9]
```

The first `-3` comes from source `0`, then the equal value from source `2`. All three copies of `4` remain in the output.

### 3. Understand the Problem
Each source is already sorted, but the next globally smallest value may belong to any nonexhausted source. Validation is deliberately a separate first phase: even a malformed value near the end of a source must throw before the function allocates output entries or performs heap work.

#### In Simple Terms
Check every list first, then repeatedly take the smallest front value and advance only the list it came from.

### 4. Real-World Analogy
Imagine several numbered checkout lines whose customers are already ordered by appointment time. A coordinator compares only the first waiting customer in each line, serves the earliest one, and then admits the next customer from that same line to the comparison.

### 5. Key Observation
After one value is removed from a sorted source, only its immediate successor can become that source's next global candidate. There is no reason to store every unmerged value in the heap, so at most one entry per source is sufficient.

### 6. Pattern
```text
Pattern: Multiway Merge with a Min Heap
Why this pattern applies: each sorted source exposes one next candidate
How to recognize it: merge many ordered sequences without flattening first
Similar problems: merge k linked lists, external merge sort, event-stream merge
```

### 7. Brute Force
Validate every value and every adjacent pair first. Then copy all values into a new array and numerically sort that array with relational comparisons. The copy preserves all duplicates and leaves every caller-owned array unchanged.

#### Pseudocode
```text
for each source in source order:
    for each value in element order:
        require a safe integer
        require previous value <= current value
copy every validated value into a result array
sort the result numerically ascending
return the result
```

#### Brute-Force TypeScript
```typescript
function mergeKSortedArraysBySorting(
    arrays: readonly (readonly number[])[],
): number[] {
    for (const source of arrays) {
        for (let elementIndex = 0; elementIndex < source.length; elementIndex++) {
            const value = source[elementIndex]!;
            if (!Number.isSafeInteger(value)) {
                throw new RangeError("every value must be a safe integer");
            }
            if (elementIndex > 0 && source[elementIndex - 1]! > value) {
                throw new RangeError("every source must be nondecreasing");
            }
        }
    }

    const merged: number[] = [];
    for (const source of arrays) {
        for (const value of source) {
            merged.push(value);
        }
    }

    return merged.sort((left, right) => {
        if (left < right) return -1;
        if (left > right) return 1;
        return 0;
    });
}
```

#### Visualization
```text
validated sources: [-3,4,4]  []  [-3,2,9]  [0,4]
flattened copy:    [-3,4,4,-3,2,9,0,4]
numeric sort:      [-3,-3,0,2,4,4,4,9]
```

#### Dry Run
The validation pass confirms that all eight values are safe integers and that no source decreases. Flattening copies values in source order. Sorting the new array produces the required numeric order while the four source arrays retain their original contents.

#### Brute-Force Complexity
Let `N` be the total number of values. Validation and flattening take `O(N)` time, and sorting takes `O(N log N)`, so total time is `O(N log N)`. The copied result uses `O(N)` space, which is also the returned output space.

### 8. Why Brute Force Is Not Ideal
Flattening discards the useful ordering already present in each source. A full sort compares values that were known to be ordered before the algorithm began, while a multiway merge compares only current source fronts.

### 9. Optimal Approach
First validate all sources completely. Seed a min heap with element `0` from each nonempty source. Pop the best candidate, append its value, and push the next element from that same source if one exists. Compare candidates by value, source index, and element index using relational checks rather than subtraction, because subtracting extreme safe integers can produce an unsafe integer.

### 10. Visualization
```text
sources: 0:[-3,4,4]  1:[]  2:[-3,2,9]  3:[0,4]
seed:    (-3,s0,e0), (-3,s2,e0), (0,s3,e0)

pop (-3,s0,e0) -> emit -3 -> push (4,s0,e1)
pop (-3,s2,e0) -> emit -3 -> push (2,s2,e1)
pop (0,s3,e0)  -> emit 0  -> push (4,s3,e1)
```

Equal values are selected by source index and then element index, even though only their indistinguishable numeric values appear in the returned array.

### 11. Step-by-Step Dry Run
| Pop | Appended | Candidate pushed | Heap size after push |
|---|---:|---|---:|
| `(-3, s0, e0)` | `-3` | `(4, s0, e1)` | `3` |
| `(-3, s2, e0)` | `-3` | `(2, s2, e1)` | `3` |
| `(0, s3, e0)` | `0` | `(4, s3, e1)` | `3` |
| `(2, s2, e1)` | `2` | `(9, s2, e2)` | `3` |
| `(4, s0, e1)` | `4` | `(4, s0, e2)` | `3` |
| `(4, s0, e2)` | `4` | none | `2` |
| `(4, s3, e1)` | `4` | none | `1` |
| `(9, s2, e2)` | `9` | none | `0` |

### 12. TypeScript Solution
```typescript
function mergeKSortedArrays(
    arrays: readonly (readonly number[])[],
): number[] {
    interface Candidate {
        value: number;
        sourceIndex: number;
        elementIndex: number;
    }

    const compareCandidate = (left: Candidate, right: Candidate): number => {
        if (left.value < right.value) return -1;
        if (left.value > right.value) return 1;
        if (left.sourceIndex < right.sourceIndex) return -1;
        if (left.sourceIndex > right.sourceIndex) return 1;
        if (left.elementIndex < right.elementIndex) return -1;
        if (left.elementIndex > right.elementIndex) return 1;
        return 0;
    };

    class CandidateMinHeap {
        private readonly heap: Candidate[] = [];

        get size(): number {
            return this.heap.length;
        }

        push(candidate: Candidate): void {
            this.heap.push(candidate);
            let index = this.heap.length - 1;

            while (index > 0) {
                const parent = Math.floor((index - 1) / 2);
                if (compareCandidate(this.heap[index]!, this.heap[parent]!) >= 0) {
                    return;
                }
                [this.heap[index], this.heap[parent]] = [
                    this.heap[parent]!,
                    this.heap[index]!,
                ];
                index = parent;
            }
        }

        pop(): Candidate | undefined {
            if (this.heap.length === 0) return undefined;

            const root = this.heap[0]!;
            const last = this.heap.pop()!;
            if (this.heap.length === 0) return root;

            this.heap[0] = last;
            let index = 0;
            while (true) {
                const left = index * 2 + 1;
                if (left >= this.heap.length) return root;

                const right = left + 1;
                let smallest = left;
                if (
                    right < this.heap.length &&
                    compareCandidate(this.heap[right]!, this.heap[left]!) < 0
                ) {
                    smallest = right;
                }

                if (compareCandidate(this.heap[smallest]!, this.heap[index]!) >= 0) {
                    return root;
                }
                [this.heap[index], this.heap[smallest]] = [
                    this.heap[smallest]!,
                    this.heap[index]!,
                ];
                index = smallest;
            }
        }
    }

    for (const source of arrays) {
        for (let elementIndex = 0; elementIndex < source.length; elementIndex++) {
            const value = source[elementIndex]!;
            if (!Number.isSafeInteger(value)) {
                throw new RangeError("every value must be a safe integer");
            }
            if (elementIndex > 0 && source[elementIndex - 1]! > value) {
                throw new RangeError("every source must be nondecreasing");
            }
        }
    }

    const candidates = new CandidateMinHeap();
    for (let sourceIndex = 0; sourceIndex < arrays.length; sourceIndex++) {
        const source = arrays[sourceIndex]!;
        if (source.length > 0) {
            candidates.push({
                value: source[0]!,
                sourceIndex,
                elementIndex: 0,
            });
        }
    }

    const merged: number[] = [];
    while (candidates.size > 0) {
        const candidate = candidates.pop()!;
        merged.push(candidate.value);

        const source = arrays[candidate.sourceIndex]!;
        const nextElementIndex = candidate.elementIndex + 1;
        if (nextElementIndex < source.length) {
            candidates.push({
                value: source[nextElementIndex]!,
                sourceIndex: candidate.sourceIndex,
                elementIndex: nextElementIndex,
            });
        }
    }

    return merged;
}
```

### 13. Code Walkthrough
The validation loops finish before the heap or output array is created. Each heap entry carries numeric value and provenance. `compareCandidate` uses explicit relational branches for all three keys, avoiding unsafe subtraction at the safe-integer boundaries. After a pop, only the next element from that candidate's source is eligible to enter the heap.

### 14. Complexity Analysis
Let `N` be the total number of values and `k` the number of sources. Validation takes `O(N)`. The heap holds at most `min(k, N)` candidates, so merging takes `O(N log min(k, N))` time and `O(min(k, N))` auxiliary heap space, conventionally stated as `O(N log k)` time and `O(k)` auxiliary space. The returned output uses `O(N)` additional result space. When `k = 0`, the function returns in `O(1)`; when at most one source is nonempty, heap operations are constant-time and the full operation is `O(N)`.

### 15. Edge Cases
- An empty outer array returns `[]`.
- Any number of empty sources, including all-empty input, is valid.
- One nonempty source is copied without mutation in linear time.
- Duplicate values within and across sources are retained.
- `Number.MIN_SAFE_INTEGER` and `Number.MAX_SAFE_INTEGER` are valid values.
- A non-safe integer or a decreasing adjacent pair throws before heap seeding or output production.
- Equal candidates have deterministic provenance order even though equal numbers are indistinguishable in the result.

### 16. Common Mistakes
- Beginning to emit values before every source has been validated.
- Sorting a source in place to hide invalid nondecreasing input.
- Dropping duplicates while merging.
- Putting every value in the heap instead of one candidate per source.
- Comparing safe integers with subtraction and assuming the difference is also safe.
- Omitting source and element indexes from deterministic tie-breaking.
- Claiming `O(N log k)` without addressing the empty and single-candidate cases.

### 17. Interview Follow-ups
1. Merge iterators that reveal values lazily. 2. Return provenance with each value. 3. Limit output to the first `m` values. 4. Merge descending sources. 5. Read sources from external storage with bounded buffers.

### 18. Variations
- Merge k sorted linked lists. - Merge timestamped event streams. - Deduplicated multiway merge. - Stable record merge. - External merge sort.

### 19. Pattern Connections
This is the heap form of the two-pointer merge pattern: each additional source contributes one pointer and one current candidate. The explicit provenance tie-break extends the chapter's deterministic-ordering rule. Although equal numeric outputs reveal no source order, deterministic entries make traces, tests, and future record-returning variants reproducible.

### 20. NestJS Project Connection
For `POST /api/dsa/heaps/merge-sorted`, validate and cap both source count and total element count before calling the service. Keep DTO arrays readonly and let the algorithm's full prevalidation enforce safe integers and nondecreasing order without mutating request data. If tracing candidate provenance, bound trace events and avoid logging complete source payloads.

### 21. Mini Challenge
Implement `mergeKSortedArraysWithSource` so each returned entry contains `value`, `sourceIndex`, and `elementIndex`. Preserve the same validation, immutability, and deterministic comparator contract.

## Problem 184 — Find Median from Data Stream

### 1. Problem Statement
Implement a class `MedianFinder` with `add(value: number): void`, `get size(): number`, and `findMedian(): number | null`. Each added value must be a safe integer. An invalid `add` throws `RangeError` before mutating owned state. Duplicates are accepted, values are never removed, and an empty stream has median `null`.

The optimized class must own a lower max heap and an upper min heap. The lower heap has either the same size as the upper heap or one additional value, and every lower value is at most every upper value. Median lookup must return the middle safe integer for odd size and the overflow-aware numeric average of both roots for even size, including `.5` results when representable.

### 2. Example
```text
finder.findMedian()                         -> null
finder.add(1); finder.findMedian()          -> 1
finder.add(2); finder.findMedian()          -> 1.5
finder.add(2); finder.findMedian()          -> 2

edges.add(Number.MIN_SAFE_INTEGER)
edges.add(Number.MAX_SAFE_INTEGER)
edges.findMedian()                          -> 0

before = finder.size
finder.add(Number.NaN)                      -> throws RangeError
finder.size === before                      -> true
```

The rejected addition is atomic: neither heap nor the brute-force sorted array changes before the exception.

### 3. Understand the Problem
The stream grows one value at a time, and callers may ask for the median after any addition. Re-sorting all prior values for each query wastes previous ordering work. The API also owns its state, so caller variables cannot mutate values after insertion.

#### In Simple Terms
Keep the smaller half and larger half balanced so the one or two middle values are always available immediately.

### 4. Real-World Analogy
Imagine a scale with lower scores on the left and upper scores on the right. The left side may hold one extra score. The values nearest the center of the scale determine the median, so the outer scores never need to be inspected for a query.

### 5. Key Observation
A max heap exposes the largest value in the lower half, while a min heap exposes the smallest value in the upper half. Those roots are exactly the middle pair. Routing by the lower root and then moving one root when sizes drift preserves both ordering and balance.

### 6. Pattern
```text
Pattern: Two Heaps for an Online Median
Why this pattern applies: only the middle boundary of each half is queried
How to recognize it: incremental values with repeated median requests
Similar problems: running percentiles, load balancing, streaming statistics
```

### 7. Brute Force
Own one sorted array. Validate an incoming value before touching the array, scan for its insertion position, and insert it there. Median lookup reads the center value or center pair without changing the stored sequence.

#### Pseudocode
```text
add(value):
    require value to be a safe integer
    scan to the first stored value greater than value
    insert value at that position

findMedian():
    return null when empty
    return the center value when size is odd
    return the overflow-aware average of the center pair when size is even
```

#### Brute-Force TypeScript
```typescript
function averageSafeIntegers(lower: number, upper: number): number {
    const average =
        lower < 0 && upper > 0
            ? (lower + upper) / 2
            : lower + (upper - lower) / 2;

    if (!Number.isFinite(average) || average < lower || average > upper) {
        throw new RangeError("median must be a finite value within its roots");
    }
    return average;
}

class MedianFinder {
    private readonly values: number[] = [];

    get size(): number {
        return this.values.length;
    }

    add(value: number): void {
        if (!Number.isSafeInteger(value)) {
            throw new RangeError("value must be a safe integer");
        }

        let insertionIndex = 0;
        while (
            insertionIndex < this.values.length &&
            this.values[insertionIndex]! <= value
        ) {
            insertionIndex++;
        }
        this.values.splice(insertionIndex, 0, value);
    }

    findMedian(): number | null {
        if (this.values.length === 0) return null;

        const upperMiddle = Math.floor(this.values.length / 2);
        if (this.values.length % 2 === 1) {
            return this.values[upperMiddle]!;
        }

        return averageSafeIntegers(
            this.values[upperMiddle - 1]!,
            this.values[upperMiddle]!,
        );
    }
}
```

#### Visualization
```text
add 5: [5]          median 5
add 2: [2,5]        median 3.5
add 8: [2,5,8]      median 5
add 5: [2,5,5,8]    median 5
```

#### Dry Run
Each valid addition scans the sorted owned array until the first larger value. Duplicates are inserted after existing equals, although duplicate order has no numeric effect. The invalid `NaN` example throws before the scan or `splice`, leaving both content and size unchanged.

#### Brute-Force Complexity
For `n` stored values, `add` takes `O(n)` time for the linear search and array shift. `findMedian` and `size` take `O(1)` time. The class owns `O(n)` state and uses `O(1)` temporary auxiliary space per operation.

### 8. Why Brute Force Is Not Ideal
Maintaining a fully sorted array shifts up to `n` values on every insertion. Median queries need only the boundary between two halves, so preserving complete order performs unnecessary work.

### 9. Optimal Approach
Store the lower half in a max heap and the upper half in a min heap. After validating, route a value to `lower` when it is no greater than the lower root; otherwise route it to `upper`. If `lower` has more than one extra value, move its root to `upper`. If `upper` becomes larger, move its root to `lower`.

For even size, let `lowerRoot <= upperRoot`. When they have opposite nonzero signs, `(lowerRoot + upperRoot) / 2` is safe because the sum lies within the safe-integer range. Otherwise, `lowerRoot + (upperRoot - lowerRoot) / 2` is safe because the ordered same-sign difference is a safe integer. Zero belongs to the second case. A final finite and range check guards the arithmetic contract.

### 10. Visualization
```text
stream: 5, 2, 8, 5

add 5 -> lower(max): [5]    upper(min): []
add 2 -> lower gets 2, then move 5
         lower: [2]         upper: [5]
add 8 -> upper gets 8, then move 5
         lower: [5,2]       upper: [8]
add 5 -> lower gets 5, then move one 5
         lower: [5,2]       upper: [5,8]
median roots: 5 and 5 -> 5
```

### 11. Step-by-Step Dry Run
| Added | Lower root / size | Upper root / size | Median |
|---:|---|---|---:|
| `5` | `5 / 1` | `- / 0` | `5` |
| `2` | `2 / 1` | `5 / 1` | `3.5` |
| `8` | `5 / 2` | `8 / 1` | `5` |
| `5` | `5 / 2` | `5 / 2` | `5` |

After every row, lower has equal size or one extra element, and its root is no greater than the upper root when both heaps are nonempty.

### 12. TypeScript Solution
```typescript
function averageSafeIntegers(lower: number, upper: number): number {
    const average =
        lower < 0 && upper > 0
            ? (lower + upper) / 2
            : lower + (upper - lower) / 2;

    if (!Number.isFinite(average) || average < lower || average > upper) {
        throw new RangeError("median must be a finite value within its roots");
    }
    return average;
}

class BinaryHeap {
    private readonly heap: number[] = [];

    constructor(
        private readonly comesBefore: (left: number, right: number) => boolean,
    ) {}

    get size(): number {
        return this.heap.length;
    }

    peek(): number {
        return this.heap[0]!;
    }

    push(value: number): void {
        this.heap.push(value);
        let index = this.heap.length - 1;

        while (index > 0) {
            const parent = Math.floor((index - 1) / 2);
            if (!this.comesBefore(this.heap[index]!, this.heap[parent]!)) {
                return;
            }
            [this.heap[index], this.heap[parent]] = [
                this.heap[parent]!,
                this.heap[index]!,
            ];
            index = parent;
        }
    }

    pop(): number | undefined {
        if (this.heap.length === 0) return undefined;

        const root = this.heap[0]!;
        const last = this.heap.pop()!;
        if (this.heap.length === 0) return root;

        this.heap[0] = last;
        let index = 0;
        while (true) {
            const left = index * 2 + 1;
            if (left >= this.heap.length) return root;

            const right = left + 1;
            let preferred = left;
            if (
                right < this.heap.length &&
                this.comesBefore(this.heap[right]!, this.heap[left]!)
            ) {
                preferred = right;
            }

            if (!this.comesBefore(this.heap[preferred]!, this.heap[index]!)) {
                return root;
            }
            [this.heap[index], this.heap[preferred]] = [
                this.heap[preferred]!,
                this.heap[index]!,
            ];
            index = preferred;
        }
    }
}

class MedianFinder {
    private readonly lower = new BinaryHeap((left, right) => left > right);
    private readonly upper = new BinaryHeap((left, right) => left < right);

    get size(): number {
        return this.lower.size + this.upper.size;
    }

    add(value: number): void {
        if (!Number.isSafeInteger(value)) {
            throw new RangeError("value must be a safe integer");
        }

        if (this.lower.size === 0 || value <= this.lower.peek()) {
            this.lower.push(value);
        } else {
            this.upper.push(value);
        }

        if (this.lower.size > this.upper.size + 1) {
            this.upper.push(this.lower.pop()!);
        } else if (this.upper.size > this.lower.size) {
            this.lower.push(this.upper.pop()!);
        }
    }

    findMedian(): number | null {
        if (this.size === 0) return null;
        if (this.lower.size > this.upper.size) return this.lower.peek();
        return averageSafeIntegers(this.lower.peek(), this.upper.peek());
    }
}
```

### 13. Code Walkthrough
`BinaryHeap` owns its dense numeric array and receives a fixed priority predicate, creating a max heap for `lower` and a min heap for `upper`. `add` rejects invalid input before either heap can change. Routing preserves cross-heap order; moving only a root restores the size invariant. `findMedian` reads roots without mutation and delegates even-size arithmetic to the checked helper.

### 14. Complexity Analysis
With `n` stored values, each `add` performs one heap insertion and at most one pop-plus-insertion rebalance, taking `O(log n)` time. `size` and `findMedian` take `O(1)` time. The two heaps together own `O(n)` state, with `O(1)` temporary auxiliary space per operation.

### 15. Edge Cases
- A new finder has size `0` and median `null`.
- One value is both the lower root and the median.
- Duplicates are accepted and counted separately.
- Negative values, zero, and positive values can be mixed.
- `Number.MIN_SAFE_INTEGER` followed by `Number.MAX_SAFE_INTEGER` has median `0`.
- Values `1` and `2` have exact median `1.5`.
- `NaN`, infinities, fractions, and unsafe integers throw before state mutation.
- Repeated median queries do not mutate either heap.

### 16. Common Mistakes
- Validating after pushing and violating atomic failure.
- Using two min heaps or two max heaps.
- Allowing the upper heap to contain more values than the lower heap.
- Rebalancing sizes without preserving lower-to-upper ordering.
- Averaging roots with `(lower + upper) / 2` for all signs.
- Using `lower + (upper - lower) / 2` when opposite signs can make the difference unsafe.
- Re-sorting heap snapshots during every median query.

### 17. Interview Follow-ups
1. Support removal for a sliding-window median. 2. Merge two median finders. 3. Return a rational median with no floating-point representability loss. 4. Persist and restore stream state. 5. Generalize to arbitrary quantiles.

### 18. Variations
- Sliding-window median. - Weighted median. - Running percentile. - Median of timestamped windows. - Distributed approximate median.

### 19. Pattern Connections
The two roots form a dynamic partition boundary, connecting heaps to binary-search partition problems. The class also demonstrates owned mutable state: readonly request input is not relevant after individual numbers are copied into internal heaps. JavaScript `number` represents ordinary half-integer medians such as `1.5` exactly; callers requiring an exact rational for every extreme safe-integer pair need a different return contract.

### 20. NestJS Project Connection
A median stream must have explicit ownership. Use a request-scoped `MedianFinder` only when one HTTP request supplies and queries the complete stream. For WebSocket, session, or long-lived ingestion streams, bind one finder to a clearly identified lifecycle, cap its size, clean it up on disconnect or expiry, and synchronize concurrent updates. Do not place unrelated tenants in one singleton finder; a singleton registry needs authenticated stream keys, isolation, and bounded retention.

### 21. Mini Challenge
Implement `findMedianFraction(): { numerator: bigint; denominator: 1n | 2n } | null` so every median of safe-integer inputs is represented exactly, including adjacent values near the safe-integer boundaries.

## Problem 185 — Connect Ropes with Minimum Cost

### 1. Problem Statement
Implement `minimumRopeConnectionCost(lengths: readonly number[]): number`. Every rope length must be a nonnegative safe integer. Validate every length before allocating working storage or returning, so even a one-element input containing a negative, fractional, non-finite, or unsafe value throws `RangeError`. Do not mutate `lengths`.

Connecting ropes of lengths `a` and `b` creates one rope of length `a + b` and costs `a + b`. Continue until at most one rope remains and return the minimum possible total cost. Empty and one-element valid inputs return `0`; duplicate and zero lengths are allowed. Every merged length addition and every accumulated-total addition must be checked with an explicit `safeAdd` helper and must remain a safe integer. Throw `RangeError` at the first evaluated overflow.

### 2. Example
```text
lengths = [4, 3, 2, 6]

connect 2 and 3 -> merged 5,  total 5
connect 4 and 5 -> merged 9,  total 14
connect 6 and 9 -> merged 15, total 29

result = 29

[0, 0, 1] -> 1
[Number.MAX_SAFE_INTEGER, 1] -> throws RangeError when evaluating the merge
```

### 3. Understand the Problem
Each connection changes both the immediate cost and the length of a rope that may be charged again later. A large rope merged early can therefore contribute to many later costs. The contract also makes arithmetic order observable: validate the complete input first, then check the selected merge, then check its addition to the accumulated total.

#### In Simple Terms
Always join the two shortest available ropes so small costs are repeated and large costs are delayed.

### 4. Real-World Analogy
Imagine combining sorted data files where merging two files costs the sum of their sizes. A file created early may be read and written again in later merges, so combining the smallest files first avoids repeatedly processing a large file.

### 5. Key Observation
Suppose an optimal merge plan does not combine two currently shortest lengths `x <= y` first. At the earliest merge involving either one, replacing the selected no-smaller leaves with `x` and `y` cannot increase that merge or any ancestor merge containing their combined weight. This exchange moves the two smallest ropes together without increasing total cost. Repeating the exchange argument yields a greedy plan that always combines the two minima.

### 6. Pattern
```text
Pattern: Repeatedly Combine the Two Minimums
Why this pattern applies: each merged weight is charged now and in every ancestor merge
How to recognize it: pairwise combination cost equals the sum of combined weights
Similar problems: Huffman coding, optimal merge pattern, minimum file-merge cost
```

### 7. Brute Force
After validating all lengths, copy them into working storage. On each iteration, scan the current array once to identify the two minimum indexes without sorting the whole array. Check their merged length with `safeAdd`, then check the new accumulated cost with `safeAdd`. Remove the two selected entries in descending index order so removing one cannot shift the other selected index, and append the merged rope.

#### Pseudocode
```text
validate every length before allocating working storage or returning
if fewer than two ropes: return 0
remaining = copy of lengths
total = 0
while at least two ropes remain:
    scan once for indexes of the two minimum lengths
    merged = safeAdd(the two lengths)
    total = safeAdd(total, merged)
    remove the higher selected index, then the lower selected index
    append merged
return total
```

#### Brute-Force TypeScript
```typescript
function safeAdd(left: number, right: number): number {
    const sum = left + right;
    if (!Number.isSafeInteger(sum)) {
        throw new RangeError("rope arithmetic must remain a safe integer");
    }
    return sum;
}

function minimumRopeConnectionCostByScanning(
    lengths: readonly number[],
): number {
    for (const length of lengths) {
        if (!Number.isSafeInteger(length) || length < 0) {
            throw new RangeError("every rope length must be a nonnegative safe integer");
        }
    }

    if (lengths.length <= 1) return 0;
    const remaining = [...lengths];
    let total = 0;

    while (remaining.length > 1) {
        let smallestIndex = 0;
        let secondSmallestIndex = 1;
        if (remaining[secondSmallestIndex]! < remaining[smallestIndex]!) {
            [smallestIndex, secondSmallestIndex] = [
                secondSmallestIndex,
                smallestIndex,
            ];
        }

        for (let index = 2; index < remaining.length; index += 1) {
            if (remaining[index]! < remaining[smallestIndex]!) {
                secondSmallestIndex = smallestIndex;
                smallestIndex = index;
            } else if (remaining[index]! < remaining[secondSmallestIndex]!) {
                secondSmallestIndex = index;
            }
        }

        const merged = safeAdd(
            remaining[smallestIndex]!,
            remaining[secondSmallestIndex]!,
        );
        total = safeAdd(total, merged);

        const higherIndex = Math.max(smallestIndex, secondSmallestIndex);
        const lowerIndex = Math.min(smallestIndex, secondSmallestIndex);
        remaining.splice(higherIndex, 1);
        remaining.splice(lowerIndex, 1);
        remaining.push(merged);
    }

    return total;
}
```

#### Visualization
```text
remaining [4, 3, 2, 6]
scan minima: indexes 2 and 1 -> lengths 2 and 3
remove index 2 first: [4, 3, 6]
remove index 1 next:  [4, 6]
append merged 5:      [4, 6, 5]

Repeat scans until one rope remains.
```

#### Dry Run
For `[4, 3, 2, 6]`, the first scan selects `2` and `3`, producing checked merge `5` and checked total `5`. The next scan selects `4` and `5`, producing merge `9` and total `14`. The final scan selects `6` and `9`, producing merge `15` and total `29`. The original array remains unchanged because every removal occurs in `remaining`.

#### Brute-Force Complexity
With `m` ropes remaining, finding the two minima takes `O(m)` time and the two `splice` calls shift `O(m)` elements. Summing over `m = n` down to `2` gives `O(n^2)` time. The working copy uses `O(n)` auxiliary space. Validation takes `O(n)` time and uses no proportional storage.

### 8. Why Brute Force Is Not Ideal
The baseline rediscovers the smallest ropes with a full scan after every merge, and indexed removal repeatedly shifts array elements. A min heap maintains the same minimum boundary incrementally and removes its root in logarithmic time.

### 9. Optimal Approach
Validate every input first. For at least two ropes, copy the lengths into an owned min heap and build it bottom-up in `O(n)` time. Repeatedly pop the two smallest lengths, compute their checked merge, compute the checked accumulated total, and push the merged length. Stop when one heap entry remains. This directly implements the greedy choice proved by the exchange intuition.

### 10. Visualization
```text
input multiset: {4, 3, 2, 6}
bottom-up min heap has root 2

pop 2, pop 3, push 5  -> {4, 5, 6}, total 5
pop 4, pop 5, push 9  -> {6, 9},    total 14
pop 6, pop 9, push 15 -> {15},      total 29
```

### 11. Step-by-Step Dry Run
| Step | Two minimums | Checked merge | Checked total | Remaining multiset |
|---:|---|---:|---:|---|
| 1 | `2`, `3` | `5` | `5` | `{4, 5, 6}` |
| 2 | `4`, `5` | `9` | `14` | `{6, 9}` |
| 3 | `6`, `9` | `15` | `29` | `{15}` |

For `[0, 0, 1]`, the checked merges are `0` and `1`, and the checked totals are `0` and `1`. For `[Number.MAX_SAFE_INTEGER, 1]`, the first merge addition is unsafe, so the function throws before evaluating the accumulated-total addition.

### 12. TypeScript Solution
```typescript
function safeAdd(left: number, right: number): number {
    const sum = left + right;
    if (!Number.isSafeInteger(sum)) {
        throw new RangeError("rope arithmetic must remain a safe integer");
    }
    return sum;
}

class MinNumberHeap {
    private readonly heap: number[];

    constructor(values: readonly number[]) {
        this.heap = [...values];
        for (let index = Math.floor(this.heap.length / 2) - 1; index >= 0; index -= 1) {
            this.siftDown(index);
        }
    }

    get size(): number {
        return this.heap.length;
    }

    push(value: number): void {
        this.heap.push(value);
        let index = this.heap.length - 1;

        while (index > 0) {
            const parent = Math.floor((index - 1) / 2);
            if (this.heap[parent]! <= this.heap[index]!) return;
            [this.heap[parent], this.heap[index]] = [
                this.heap[index]!,
                this.heap[parent]!,
            ];
            index = parent;
        }
    }

    pop(): number | undefined {
        if (this.heap.length === 0) return undefined;

        const root = this.heap[0]!;
        const last = this.heap.pop()!;
        if (this.heap.length === 0) return root;

        this.heap[0] = last;
        this.siftDown(0);
        return root;
    }

    private siftDown(startIndex: number): void {
        let index = startIndex;

        while (true) {
            const left = index * 2 + 1;
            if (left >= this.heap.length) return;

            const right = left + 1;
            let smallest = left;
            if (right < this.heap.length && this.heap[right]! < this.heap[left]!) {
                smallest = right;
            }

            if (this.heap[index]! <= this.heap[smallest]!) return;
            [this.heap[index], this.heap[smallest]] = [
                this.heap[smallest]!,
                this.heap[index]!,
            ];
            index = smallest;
        }
    }
}

function minimumRopeConnectionCost(lengths: readonly number[]): number {
    for (const length of lengths) {
        if (!Number.isSafeInteger(length) || length < 0) {
            throw new RangeError("every rope length must be a nonnegative safe integer");
        }
    }

    if (lengths.length <= 1) return 0;
    const ropes = new MinNumberHeap(lengths);
    let total = 0;

    while (ropes.size > 1) {
        const merged = safeAdd(ropes.pop()!, ropes.pop()!);
        total = safeAdd(total, merged);
        ropes.push(merged);
    }

    return total;
}
```

### 13. Code Walkthrough
The validation loop completes before the early return or heap allocation. `safeAdd` computes the ordinary JavaScript sum and immediately verifies `Number.isSafeInteger`, so each selected merge is checked before that merge is added to `total`. `MinNumberHeap` owns a copy and heapifies it bottom-up. Each loop removes exactly the two roots, updates checked arithmetic in contract order, and reinserts one merged rope.

### 14. Complexity Analysis
Validation and bottom-up heap construction each take `O(n)` time. There are `n - 1` merges, each performing two `O(log n)` pops and one `O(log n)` push, for `O(n log n)` total time. The owned heap copy uses `O(n)` auxiliary space. Empty and one-element valid inputs take `O(n)` validation time, which is `O(1)` for their bounded sizes, and return without allocating the working heap.

### 15. Edge Cases
- Empty input returns `0` after vacuous validation.
- One valid length, including `0` or `Number.MAX_SAFE_INTEGER`, returns `0`.
- One invalid length throws instead of returning `0`.
- Duplicate and zero lengths are retained as separate ropes.
- `[0, 0, 1]` returns `1`.
- `[4, 3, 2, 6]` returns `29`.
- A merged length beyond the safe-integer range throws before total accumulation.
- A safe merged length whose addition makes the total unsafe throws at that total addition.
- The readonly input remains unchanged on success and failure.

### 16. Common Mistakes
- Returning `0` for a one-element input before validating its length.
- Sorting or splicing the caller's readonly array.
- Repeatedly sorting the whole working array instead of finding two minima or using a heap.
- Removing the lower selected index first and shifting the other selected index.
- Combining arbitrary ropes rather than the two current minima.
- Checking only the final total while allowing an unsafe merged rope.
- Computing additions without an explicit `Number.isSafeInteger` check.
- Claiming repeated heap insertion is bottom-up `O(n)` construction.

### 17. Interview Follow-ups
1. Return the chosen merge sequence. 2. Represent lengths and costs with `bigint`. 3. Generalize to combining `d` ropes at a time. 4. Prove the greedy rule through weighted leaf depths. 5. Process an external source with a bounded in-memory frontier.

### 18. Variations
- Huffman coding. - Optimal file merge. - Merge stones with different cost rules. - D-ary combination. - Exact `bigint` rope costs.

### 19. Pattern Connections
This is the optimal merge pattern underlying Huffman coding: every original length contributes once for each merge ancestor above it, so delaying larger weights minimizes their repeated contribution. The min heap supplies the two greedy choices efficiently, while the checked arithmetic contract extends the chapter's safe-integer discipline beyond median averaging.

### 20. NestJS Project Connection
For `POST /api/dsa/heaps/connect-ropes`, cap the DTO array length and validate every value as a nonnegative safe integer before service work. Map input and arithmetic `RangeError` failures to stable client errors without exposing internal stack details. The service can pass readonly DTO data directly because the function owns its heap copy and never mutates the request array. Cap any merge trace independently of input size, mark truncation explicitly, and avoid logging full customer payloads or unbounded intermediate heaps.

### 21. Mini Challenge
Implement `minimumRopeConnectionPlan(lengths)` so it returns both the checked minimum total and a deterministic list of merged pairs. Preserve full prevalidation, input immutability, duplicate ropes as distinct entries, safe arithmetic, and `O(n log n)` time by assigning each original rope a stable input index for tie-breaking.

# Topic Summary

## What You Learned
A binary heap stores a complete tree in a dense array and guarantees local parent-child priority rather than global sorted order. Sift-up repairs insertion, sift-down repairs root replacement, and bottom-up heap construction runs in `O(n)`. A comparator can orient the same mechanics toward minimum or maximum priority, while explicit secondary keys make observable ties deterministic.

Heaps reduce repeated best-candidate selection from linear scans to logarithmic updates. Bounded heaps retain only the `k` candidates relevant to rank and top-k queries, a one-candidate-per-source heap performs multiway merge, two complementary heaps expose a streaming median, and a min heap implements the optimal rope-merge rule. Heap sort offers guaranteed `O(n log n)` in-place sorting but is not stable.

Production contracts make ownership and arithmetic explicit. Some algorithms mutate and return the caller's exact array, while others copy readonly input. Safe-integer validation must occur in the documented order. Streaming median avoids unsafe intermediate arithmetic, but an extreme mathematical half-integer can still round when no JavaScript `number` represents it; exact universal medians require a rational contract. Rope merging instead requires every merged length and accumulated total to remain an exact safe integer and throws on the first evaluated overflow.

## Patterns You Should Recognize
- **Sift-up after append:** repair one leaf-to-root path after insertion.
- **Sift-down after root replacement:** choose the higher-priority child before swapping.
- **Bottom-up heapify:** repair parents from the last internal node to the root in `O(n)`.
- **Shrinking active heap:** move each max root into the sorted suffix for heap sort.
- **Bounded candidate heap:** retain only the best `k` values or aggregate entries.
- **Worst-first top-k boundary:** expose the weakest retained entry for direct replacement.
- **One candidate per source:** pop globally smallest, then advance only that source.
- **Two balanced heaps:** expose the lower and upper middle boundaries of a stream.
- **Greedy two-minimum merge:** combine the two smallest current weights repeatedly.
- **Validate before work:** finish contract checks before mutation, allocation, output, or early return when required.

## Complexity Cheat Sheet
| Problem | Main approach | Time | Auxiliary space | Required output / owned state |
|---:|---|---|---|---|
| 176 | Generic min heap with bottom-up build | build `O(n)`; push/pop `O(log n)` | `O(1)` per update | `O(n)` owned copy |
| 177 | Max heap under ascending comparator | build `O(n)`; push/pop `O(log n)` | `O(1)` per update | `O(n)` owned copy |
| 178 | In-place bottom-up heapify | `O(n)` | `O(1)` | same input array |
| 179 | In-place heap sort | `O(n log n)` | `O(1)` | same input array |
| 180 | Size-`k` min heap for kth largest | `O(n log k)` | `O(k)` | one number |
| 181 | Size-`k` max heap for kth smallest | `O(n log k)` | `O(k)` | one number |
| 182 | Frequency map plus bounded heap | `O(n + d log k + k log k)` | `O(d + k)` | up to `min(k, d)` values |
| 183 | One-candidate-per-source min heap | `O(N log min(k, N))` | `O(min(k, N))` | `O(N)` merged array |
| 184 | Lower max heap plus upper min heap | add `O(log n)`; median `O(1)` | `O(1)` per update | `O(n)` owned state |
| 185 | Bottom-up min heap and greedy merges | `O(n log n)` | `O(n)` | one checked safe-integer cost |

## Common Interview Traps
- Treating the heap array as globally sorted or assuming equal-priority stability.
- Removing a root with `shift()` and turning heap operations linear.
- Comparing only the left child during sift-down.
- Calling bottom-up heap construction `O(n log n)`.
- Reversing a comparator twice when switching between generic priority and max-heap APIs.
- Mutating readonly input or failing to return the exact input reference for in-place contracts.
- Deduplicating values when duplicate ranks or occurrences must remain separate.
- Omitting deterministic tie-breakers from observable top-k or multiway-merge behavior.
- Returning internal heap storage instead of an owned snapshot.
- Validating after mutation, allocation, output, or an early return when atomic prevalidation is required.
- Averaging median roots with one formula without considering safe intermediate arithmetic and final representability.
- Claiming every mathematical half-integer median is exactly representable as a JavaScript `number`.
- Merging ropes in input order instead of repeatedly selecting both minima.
- Removing brute-force minima in ascending index order and shifting the second selected index.
- Checking only the final rope cost instead of every merged length and accumulated-total addition.

## Practice Checklist
- [ ] 176. Implement an owned generic min heap with bottom-up construction.
- [ ] 177. Implement a max heap while keeping comparator orientation explicit.
- [ ] 178. Heapify the caller's exact array in linear time.
- [ ] 179. Heap-sort in place with a shrinking active prefix.
- [ ] 180. Find the kth largest value with a bounded min heap.
- [ ] 181. Find the kth smallest value with a bounded max heap.
- [ ] 182. Return deterministic top-k frequencies with a worst-first heap.
- [ ] 183. Merge sorted arrays with one deterministic candidate per source.
- [ ] 184. Maintain a streaming median with two balanced heaps and careful averaging.
- [ ] 185. Minimize rope connection cost with greedy minima and checked arithmetic.
