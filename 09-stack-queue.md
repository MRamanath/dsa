# Stack & Queue

## Learning Objectives

By the end of this topic, you should understand:

- how LIFO stacks and FIFO queues define which item may leave next
- how `push`, `pop`, `peek`, `enqueue`, `dequeue`, `front`, `size`, and `isEmpty` behave
- how an explicit underflow contract avoids accidental exceptions
- how arrays and linked nodes represent the same abstract operations with different costs
- why an array queue should use a head index instead of shifting every dequeue
- how occasional compaction gives a head-index queue amortized `O(1)` operations
- how lazy transfer implements a queue with two stacks
- how queue rotation implements a stack with two queues
- when monotonic stacks and deques store indexes instead of values
- how ownership, stale references, duplicates, and bounded traces affect production implementations
- how to separate required output space from auxiliary working space

## Prerequisites

- arrays, zero-based indexes, and amortized growth from `01-arrays.md`
- references and generic linked nodes from `08-linked-list.md`
- TypeScript generics, classes, access modifiers, loops, and `null`/`undefined`
- Big-O notation, worst-case analysis, and output-versus-auxiliary space

## Core Theory

### LIFO and FIFO contracts

A **stack** is last in, first out (LIFO). `push` adds the new top; `pop` removes the top; `peek` reads it without removal.

```text
push A, push B, push C

top -> C   pop order: C, B, A
       B
       A
```

A **queue** is first in, first out (FIFO). `enqueue` adds at the back; `dequeue` removes the front; `front` reads it without removal.

```text
enqueue A, enqueue B, enqueue C

front -> A -> B -> C <- back
dequeue order: A, B, C
```

Operation names are part of the contract. This chapter uses stack methods `push`, `pop`, `peek`, `size`, and `isEmpty`; queues use `enqueue`, `dequeue`, `front`, `size`, and `isEmpty`.

### Underflow and `undefined`

Underflow means removing or inspecting an empty structure. The public classes here return `undefined` from `pop`, `peek`, `dequeue`, or `front` instead of throwing. `size()` returns `0`, and `isEmpty()` returns `true`.

The classes are generic over `T`, so TypeScript cannot prevent a caller from choosing a type that includes `undefined`. If stored `undefined` is allowed, an empty result and a real stored value are indistinguishable. The implementations remain type-correct, but callers should avoid storing `undefined` or use a tagged result contract such as `{ found, value }` when that distinction matters.

### Array and linked representations

An array stack keeps its top at the array's end. Native `push` and `pop` operate there in amortized `O(1)` time. Putting the top at index `0` requires `unshift` and `shift`, which move the remaining elements.

A linked stack keeps its top at the head. A fresh node can become the head in `O(1)`, and popping detaches that head in `O(1)`. Keeping the top at the tail of a singly linked list makes removal expensive because finding the predecessor requires traversal.

A linked queue tracks both head and tail. Enqueue attaches a fresh node after the tail; dequeue advances the head. When the last node leaves, both references must describe emptiness, so `tail` becomes `null` too.

### Head-index array queues

Using `Array.shift()` for every dequeue moves all remaining elements and costs `O(n)`. A better array queue leaves the backing array in place and advances a head index:

```text
items = [A, B, C, D]
head = 2
logical queue = [C, D]
```

Consumed slots can retain stale references. To release them occasionally without copying on every operation, compact only after substantial waste, such as when `head >= 1024` and `head * 2 >= items.length`. Compaction copies the live suffix and resets `head` to zero. When the queue becomes empty, reset immediately to an empty array and zero head.

One compaction can cost `O(n)`, but many cheap dequeues precede it. Charging the copy to the slots that became stale gives amortized `O(1)` dequeue. Enqueue is also amortized `O(1)` because dynamic arrays occasionally resize.

### Linked boundaries and sentinels

Head and tail references encode boundaries directly. A dummy or sentinel node can simplify algorithms that repeatedly splice before the real head, but these basic stack and queue operations need no permanent dummy: explicit empty and last-item branches are clearer. If a temporary sentinel is introduced in a variation, it is auxiliary structure and must never be exposed as user data.

### Two structures can simulate one

A queue can be built from two stacks. New values enter an input stack. Dequeue or front uses an output stack; only when it is empty are all input values transferred to it. Transfer reverses order, making the oldest value the output top. Each value crosses between stacks at most once, so operations are amortized `O(1)`.

A stack can be built from two queues. One standard variant makes `push` expensive: enqueue the new value into a secondary queue, drain the primary queue behind it, then swap queue roles. The newest value is always at the primary front, so `pop` and `peek` are amortized `O(1)`. This tradeoff is useful for pop-heavy workloads.

### Mutation and ownership

Each class owns its backing array, head index, or linked nodes. Methods may mutate that private state, but callers receive values, not the internal array or node references. Linked pushes and enqueues allocate fresh nodes. Linked removals detach the removed node so the live structure no longer owns its outgoing link. Encapsulation protects invariants such as exact size and synchronized head/tail boundaries.

## Visual Mental Model

Think of a stack as a pile of trays: only the top tray is accessible. Think of a queue as a checkout line: arrivals join the back and service happens at the front.

```text
STACK                      QUEUE

   top                     front                 back
    |                        |                      |
    v                        v                      v
  +---+                    +---+  +---+  +---+  +---+
  | C |                    | A |->| B |->| C |->| D |
  +---+                    +---+  +---+  +---+  +---+
  | B |
  +---+
  | A |
  +---+
```

For a two-stack queue, transfer is a reversal that restores FIFO order:

```text
input top -> C, B, A    transfer    output top -> A, B, C
enqueue side                               dequeue side
```

For a two-queue stack with costly push, rotate the older values behind the newcomer:

```text
primary front: A, B
push C: secondary = C, then drain A, B
swap: primary front = C, A, B
```

## Complexity Concepts

### Worst-case and amortized cost

Worst-case cost describes one operation. Amortized cost spreads occasional expensive work over a sequence. An array append may resize, a head-index queue may compact, and a two-stack queue may transfer many items, yet each still offers amortized `O(1)` operations under the stated strategy.

### Per-operation analysis

Data-structure APIs should report each operation separately. A design may have `O(n)` `push` and `O(1)` `pop`, or the reverse. Calling the whole class simply `O(n)` hides the workload tradeoff.

### Output space and auxiliary space

These implementation methods return at most one stored value, so their required output space is `O(1)`. The structure itself owns `O(n)` storage for `n` live values; that persistent representation is not temporary algorithm output. Temporary transfer stacks, secondary queues, or compaction copies can use `O(n)` auxiliary space during an operation. A bounded trace is additional output or instrumentation space and should be reported separately.

### Stale array slots

Advancing a head index is logically a removal, but consumed cells can still reference objects physically. Resetting on empty and compacting after enough waste limits retention. Clearing every cell eagerly is possible, but writing `undefined` into a generic `T[]` complicates typing and still does not reclaim array capacity.

### Bounded traces

An educational NestJS endpoint should trace only the optimized path and cap events, for example at 200. Return `{ trace, truncated }` so callers know whether the visualization is complete. Without a cap, trace storage can change an `O(1)`-auxiliary operation sequence into unbounded response memory. Trace records should expose values, sizes, and phase labels, never private arrays or node objects.

## Common Patterns

### End-backed stack

```text
push: append at length
pop: remove at length - 1
peek: read at length - 1
```

### Head-index queue with occasional compaction

```text
enqueue: append at backing-array end
dequeue: read items[head], then increment head
if empty: reset
else if waste is large and at least half: copy live suffix
```

### Lazy two-stack transfer

```text
enqueue into input
before dequeue/front:
    if output is empty:
        move every input top to output
use output top
```

### Queue rotation for a stack

```text
enqueue new value into secondary
drain primary into secondary
swap primary and secondary
```

### Monotonic stacks and deques preview

Later problems use stacks or deques that maintain increasing or decreasing order. They often store **indexes**, not values, because an index preserves position, supports distance calculations, and can still retrieve the value. Duplicate strictness controls whether equal values are popped: `>` and `>=` encode different ownership of ties and can change nearest-boundary results.

A monotonic deque supports candidate removal at both ends, commonly for sliding-window maximum. It is not the same as a general FIFO queue even though both can remove from the front.

### Ordering is not a complete LRU cache

JavaScript `Map` preserves insertion order, but ordering alone is not a full least-recently-used cache. A correct LRU contract must refresh recency on `get` and updates, enforce capacity, evict exactly the least-recently-used entry, define zero capacity, and consider ownership of mutable keys and values. A `Map` can help implement those rules; its ordering does not supply them automatically.

# Problems

## Problem 116 — Implement Stack using Array

### 1. Problem Statement
Implement a generic array-backed stack with `push`, `pop`, `peek`, `size`, and `isEmpty`. `pop` and `peek` return `undefined` on an empty stack. Do not expose the backing array.

### 2. Example
```text
Operations: push(10), push(20), peek(), pop(), size(), pop(), pop()
Outputs:    -,        -,        20,     20,    1,      10,    undefined
```

### 3. Understand the Problem
The most recently pushed value is the only value removed by `pop`. The stack owns its array, and reads must not let callers mutate that array directly.

#### In Simple Terms
Add and remove values from one end so the newest value always comes out first.

### 4. Real-World Analogy
A pile of plates allows adding, viewing, or removing only the plate at the top.

### 5. Key Observation
The end of a JavaScript array already supports append and removal without shifting all existing elements.

### 6. Pattern
```text
Pattern: End-Backed Array Stack
Why this pattern applies: the array end is the LIFO boundary
How to recognize it: newest-in must be first-out
Similar problems: undo history, expression parsing, DFS
```

### 7. Brute Force
Keep the stack top at index `0`. `unshift` adds a new top and `shift` removes it. The behavior is valid, but every mutation reindexes the remaining elements.

#### Pseudocode
```text
push value: insert value at index 0
pop: remove index 0, or return undefined
peek: read index 0
size: return array length
```

#### Brute-Force TypeScript
```typescript
class ArrayStackBruteForce<T> {
  private readonly items: T[] = [];

  push(value: T): void {
    this.items.unshift(value);
  }

  pop(): T | undefined {
    return this.items.shift();
  }

  peek(): T | undefined {
    return this.items[0];
  }

  size(): number {
    return this.items.length;
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }
}
```

#### Visualization
```text
push C into [B, A]
[B, A] -> [C, B, A]  (B and A move right)
```

#### Dry Run
Starting empty, `push(10)` makes `[10]`. `push(20)` inserts at the front and makes `[20, 10]`. `pop()` shifts out `20`, leaving `[10]`.

#### Brute-Force Complexity
`push` and `pop` are `O(n)`; `peek`, `size`, and `isEmpty` are `O(1)`. The stack owns `O(n)` storage and each returned value uses `O(1)` output space.

### 8. Why Brute Force Is Not Ideal
LIFO operations do not require preserving index `0` as the top. Reindexing every older value wastes work.

### 9. Optimal Approach
Store the bottom at index `0` and the top at `length - 1`. Use array `push` and `pop`; read the final element for `peek`. Keep the array private.

### 10. Visualization
```text
push 10: [10]
push 20: [10, 20] <- top
pop:     [10]      returns 20
```

### 11. Step-by-Step Dry Run
| Operation | Private array | Return |
|---|---|---|
| `push(10)` | `[10]` | - |
| `push(20)` | `[10, 20]` | - |
| `peek()` | `[10, 20]` | `20` |
| `pop()` | `[10]` | `20` |
| `size()` | `[10]` | `1` |

### 12. TypeScript Solution
```typescript
class ArrayStack<T> {
  private readonly items: T[] = [];

  push(value: T): void {
    this.items.push(value);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  size(): number {
    return this.items.length;
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }
}
```

### 13. Code Walkthrough
`push` appends at the top boundary. `pop` delegates the empty case to the array contract, and `peek` reads without mutation. No method returns `items`, so callers cannot corrupt stack order or size.

### 14. Complexity Analysis
`push` is amortized `O(1)` and worst-case `O(n)` during array growth. `pop`, `peek`, `size`, and `isEmpty` are `O(1)`. The structure owns `O(n)` space; each operation uses `O(1)` auxiliary and output space.

### 15. Edge Cases
- Empty `pop` and `peek` return `undefined`.
- Repeated empty operations leave size at zero.
- Duplicate values remain separate entries.
- A generic `T` may include `undefined`, making stored `undefined` ambiguous with underflow; callers should avoid that value or choose a tagged API.
- Object values are stored by reference.

### 16. Common Mistakes
- Using `shift` for the optimized implementation.
- Returning the private array from a getter.
- Decreasing a separate size counter on empty `pop`.
- Claiming array `push` is worst-case `O(1)` instead of amortized `O(1)`.

### 17. Interview Follow-ups
1. Add a fixed capacity. 2. Return a tagged underflow result. 3. Implement iteration without exposing storage. 4. Design a persistent immutable stack.

### 18. Variations
- Minimum-tracking stack. - Bounded stack. - Stack with bulk pop. - Immutable copy-on-write stack.

### 19. Pattern Connections
An end-backed stack supports iterative DFS, balanced-delimiter checks, postfix evaluation, and monotonic-stack scans.

### 20. NestJS Project Connection
Expose `POST /api/dsa/stack/array` with an operation sequence. Run only the optimized class and return outputs plus a bounded trace of `{ operation, size, top }` and `truncated`; never return the private backing array.

### 21. Mini Challenge
Add `clear(): void` while preserving encapsulation and `O(1)` observable behavior.

## Problem 117 — Implement Stack using Linked List

### 1. Problem Statement
Implement the same generic stack API with a singly linked list. Every push allocates a fresh node. `pop` and `peek` return `undefined` on empty; a popped node must be detached.

### 2. Example
```text
Operations: push("A"), push("B"), pop(), peek(), size(), pop()
Outputs:    -,         -,         "B",   "A",    1,      "A"
```

### 3. Understand the Problem
The public behavior matches an array stack, but order is encoded by node links. The class owns nodes and exposes only stored values.

#### In Simple Terms
Keep the newest value in a node at the front of a chain so it can be removed immediately.

### 4. Real-World Analogy
Place each new note on top of a linked pile; each note points to the one directly below it.

### 5. Key Observation
The head of a singly linked list has no predecessor, so insertion and removal there are constant-time operations.

### 6. Pattern
```text
Pattern: Head-Backed Linked Stack
Why this pattern applies: head insertion and deletion are O(1)
How to recognize it: LIFO order on singly linked nodes
Similar problems: call stacks, DFS worklists, persistent lists
```

### 7. Brute Force
Treat the tail as the top while storing only `head`. Push walks to append a node. Pop walks to the tail's predecessor, unlinks the tail, and returns its value.

#### Pseudocode
```text
push: walk from head to tail and append a fresh node
pop: walk to the predecessor of tail, detach tail
peek: walk to tail and read it
track count for exact size
```

#### Brute-Force TypeScript
```typescript
class StackNode<T> {
  constructor(
    public value: T,
    public next: StackNode<T> | null = null,
  ) {}
}

class LinkedStackBruteForce<T> {
  private head: StackNode<T> | null = null;
  private count = 0;

  push(value: T): void {
    const node = new StackNode(value);
    if (this.head === null) {
      this.head = node;
    } else {
      let tail = this.head;
      while (tail.next !== null) tail = tail.next;
      tail.next = node;
    }
    this.count += 1;
  }

  pop(): T | undefined {
    if (this.head === null) return undefined;
    if (this.head.next === null) {
      const value = this.head.value;
      this.head = null;
      this.count = 0;
      return value;
    }

    let predecessor = this.head;
    while (predecessor.next!.next !== null) {
      predecessor = predecessor.next!;
    }
    const removed = predecessor.next!;
    predecessor.next = null;
    this.count -= 1;
    return removed.value;
  }

  peek(): T | undefined {
    if (this.head === null) return undefined;
    let tail = this.head;
    while (tail.next !== null) tail = tail.next;
    return tail.value;
  }

  size(): number {
    return this.count;
  }

  isEmpty(): boolean {
    return this.count === 0;
  }
}
```

#### Visualization
```text
head -> [A] -> [B] -> [C] <- top
pop must walk A, B before detaching C
```

#### Dry Run
After pushes `A`, `B`, `C`, the tail is `C`. `pop()` walks from `A` to predecessor `B`, sets `B.next = null`, and returns `C`.

#### Brute-Force Complexity
`push`, `pop`, and `peek` are `O(n)` in the worst case; `size` and `isEmpty` are `O(1)`. Nodes use `O(n)` owned space and operations use `O(1)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
Singly linked nodes support cheap head changes, yet the baseline repeatedly traverses to use the opposite boundary.

### 9. Optimal Approach
Make `head` the stack top. Push a fresh node whose `next` is the old head. Pop saves the head value, advances `head`, detaches the removed node, and decrements the count.

### 10. Visualization
```text
before push C: top -> [B] -> [A] -> null
after push C:  top -> [C] -> [B] -> [A] -> null
pop C:         top -> [B] -> [A] -> null, [C] -> null
```

### 11. Step-by-Step Dry Run
| Operation | Head chain | Return | Size |
|---|---|---|---:|
| `push("A")` | `A -> null` | - | 1 |
| `push("B")` | `B -> A -> null` | - | 2 |
| `pop()` | `A -> null` | `"B"` | 1 |
| `peek()` | `A -> null` | `"A"` | 1 |

### 12. TypeScript Solution
```typescript
class StackNode<T> {
  constructor(
    public value: T,
    public next: StackNode<T> | null = null,
  ) {}
}

class LinkedStack<T> {
  private top: StackNode<T> | null = null;
  private count = 0;

  push(value: T): void {
    this.top = new StackNode(value, this.top);
    this.count += 1;
  }

  pop(): T | undefined {
    if (this.top === null) return undefined;
    const removed = this.top;
    this.top = removed.next;
    removed.next = null;
    this.count -= 1;
    return removed.value;
  }

  peek(): T | undefined {
    return this.top?.value;
  }

  size(): number {
    return this.count;
  }

  isEmpty(): boolean {
    return this.count === 0;
  }
}
```

### 13. Code Walkthrough
`top` always references the newest node. Each push creates a distinct node. Pop saves the node before moving `top`, then clears the removed node's outgoing link without exposing that node.

### 14. Complexity Analysis
All five operations are `O(1)`. The stack owns `O(n)` node space for `n` values. Each call uses `O(1)` auxiliary and output space; allocating a pushed node increases persistent structure space by `O(1)`.

### 15. Edge Cases
- Empty `pop` and `peek` return `undefined`.
- Popping the only node sets `top` to `null` and size to zero.
- Duplicate values occupy distinct fresh nodes.
- Stored `undefined` is ambiguous with underflow under the public return type.
- External object values remain shared references even though wrapper nodes are owned.

### 16. Common Mistakes
- Putting the top at the tail of a singly linked list.
- Moving `top` before saving the removed node.
- Forgetting to detach `removed.next`.
- Returning a node and leaking private structure ownership.

### 17. Interview Follow-ups
1. Remove the size counter and compute size. 2. Add a minimum node field. 3. Make the stack immutable. 4. Compare garbage-collection behavior with arrays.

### 18. Variations
- Doubly linked stack. - Fixed node pool. - Tagged underflow result. - Persistent linked stack sharing suffixes.

### 19. Pattern Connections
Head insertion is also used by adjacency-list construction and list reversal. Detachment mirrors ownership-safe linked-list deletion.

### 20. NestJS Project Connection
Expose `POST /api/dsa/stack/linked` with operations and JSON-safe values. Run only the optimized head-backed implementation and return `{ trace, truncated }` with bounded trace events such as `{ operation, size, topValue }`; do not serialize nodes or links.

### 21. Mini Challenge
Add `drain(): T[]` that pops all values in LIFO order and leaves a valid empty stack.

## Problem 118 — Implement Queue using Array

### 1. Problem Statement
Implement a generic array-backed queue with `enqueue`, `dequeue`, `front`, `size`, and `isEmpty`. Empty reads return `undefined`. The optimized implementation must use a head index with occasional compaction, not `Array.shift()`.

### 2. Example
```text
Operations: enqueue(10), enqueue(20), front(), dequeue(), size(), dequeue(), dequeue()
Outputs:    -,           -,           10,      10,        1,      20,        undefined
```

### 3. Understand the Problem
FIFO order requires removing the oldest live value. Logical removal and physical array deletion do not have to happen at the same moment.

#### In Simple Terms
Append new values at the back and move a front marker forward as old values leave.

### 4. Real-World Analogy
A numbered waiting room calls the smallest live ticket while new arrivals receive tickets at the end.

### 5. Key Observation
Advancing an integer is constant work; shifting every remaining array element is not. Rare compaction can reclaim consumed storage.

### 6. Pattern
```text
Pattern: Head-Index Array Queue
Why this pattern applies: FIFO removal happens at the logical front
How to recognize it: frequent front removals from an array-backed sequence
Similar problems: BFS worklists, stream buffers, task schedulers
```

### 7. Brute Force
Append with `push` and remove the front with `shift`. This directly resembles queue behavior but reindexes every surviving element on each dequeue.

#### Pseudocode
```text
enqueue value: append value
dequeue: shift first value or return undefined
front: read index 0
size: return array length
```

#### Brute-Force TypeScript
```typescript
class ArrayQueueBruteForce<T> {
  private readonly items: T[] = [];

  enqueue(value: T): void {
    this.items.push(value);
  }

  dequeue(): T | undefined {
    return this.items.shift();
  }

  front(): T | undefined {
    return this.items[0];
  }

  size(): number {
    return this.items.length;
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }
}
```

#### Visualization
```text
[A, B, C, D] dequeue A
[B, C, D]     B, C, and D each move left
```

#### Dry Run
Enqueue `10` and `20` to get `[10, 20]`. Dequeue shifts `10`; JavaScript moves `20` to index `0`, and front then reads `20`.

#### Brute-Force Complexity
`enqueue` is amortized `O(1)`; `dequeue` is `O(n)`; `front`, `size`, and `isEmpty` are `O(1)`. The queue owns `O(n)` storage and uses `O(1)` operation output space.

### 8. Why Brute Force Is Not Ideal
FIFO order needs a logical front, not a physically packed array after every removal. Repeated `shift` can make draining `n` values cost `O(n²)`.

### 9. Optimal Approach
Append normally and store `head`, the index of the oldest live value. Dequeue reads then increments it. Reset immediately when empty. Otherwise compact only when at least 1024 slots are consumed and at least half the backing array is stale.

### 10. Visualization
```text
items = [A, B, C, D], head = 0
dequeue A
items = [A, B, C, D], head = 1, logical queue = [B, C, D]

after enough waste:
items = live suffix, head = 0
```

### 11. Step-by-Step Dry Run
| Operation | Backing array | Head | Logical size | Return |
|---|---|---:|---:|---|
| `enqueue(10)` | `[10]` | 0 | 1 | - |
| `enqueue(20)` | `[10, 20]` | 0 | 2 | - |
| `dequeue()` | `[10, 20]` | 1 | 1 | `10` |
| `dequeue()` | `[]` | 0 | 0 | `20` |

### 12. TypeScript Solution
```typescript
class ArrayQueue<T> {
  private items: T[] = [];
  private head = 0;

  enqueue(value: T): void {
    this.items.push(value);
  }

  dequeue(): T | undefined {
    if (this.isEmpty()) return undefined;

    const value = this.items[this.head];
    this.head += 1;

    if (this.head === this.items.length) {
      this.items = [];
      this.head = 0;
    } else if (this.head >= 1024 && this.head * 2 >= this.items.length) {
      this.items = this.items.slice(this.head);
      this.head = 0;
    }
    return value;
  }

  front(): T | undefined {
    return this.isEmpty() ? undefined : this.items[this.head];
  }

  size(): number {
    return this.items.length - this.head;
  }

  isEmpty(): boolean {
    return this.size() === 0;
  }
}
```

### 13. Code Walkthrough
`size` is always `items.length - head`. Empty reset releases all consumed references. The threshold avoids frequent small copies; slicing only the live suffix removes stale references and restores a zero-based logical front.

### 14. Complexity Analysis
`enqueue` and `dequeue` are amortized `O(1)`; a resize or compaction is worst-case `O(n)`. `front`, `size`, and `isEmpty` are `O(1)`. Owned storage is `O(n + s)` before compaction for `n` live and `s` stale slots, bounded periodically; compaction briefly uses `O(n)` auxiliary copy space. Outputs are `O(1)`.

### 15. Edge Cases
- Empty `dequeue` and `front` return `undefined` without changing `head`.
- Removing the last item resets both array and head.
- Exact size excludes consumed slots.
- Compaction preserves FIFO order at threshold boundaries.
- Stored `undefined` remains ambiguous with underflow.

### 16. Common Mistakes
- Using `shift` in the optimized path.
- Reporting `items.length` as logical size.
- Never resetting or compacting, retaining stale object references indefinitely.
- Compacting after every dequeue and losing amortized efficiency.

### 17. Interview Follow-ups
1. Make compaction thresholds configurable. 2. Use a circular buffer. 3. Add fixed capacity and backpressure. 4. Return tagged underflow results.

### 18. Variations
- Ring buffer. - Bounded queue. - Priority queue. - Deque with front and back indexes.

### 19. Pattern Connections
Head-index queues power BFS and event processing. Circular buffers extend the idea by reusing physical slots instead of slicing.

### 20. NestJS Project Connection
Expose `POST /api/dsa/queue/array` with operations. Run only the optimized head-index implementation and return `{ trace, truncated }` with bounded trace events like `{ operation, size, front, compacted }`; omit stale cells and the private backing array from the response.

### 21. Mini Challenge
Add a `capacity()` diagnostic and explain why it should not become part of the abstract queue contract.

## Problem 119 — Implement Queue using Linked List

### 1. Problem Statement
Implement a generic linked queue with `enqueue`, `dequeue`, `front`, `size`, and `isEmpty`. Track head and tail, allocate a fresh node per enqueue, detach removed nodes, and set tail to `null` when the last node leaves.

### 2. Example
```text
Operations: enqueue("A"), enqueue("B"), dequeue(), front(), dequeue(), isEmpty()
Outputs:    -,            -,            "A",       "B",    "B",       true
```

### 3. Understand the Problem
The head is the oldest node and the tail is the newest. Empty state requires both references to be `null` and count to be zero.

#### In Simple Terms
Add fresh nodes at the tail and remove nodes from the head so arrival order is preserved.

### 4. Real-World Analogy
A line has an entrance at the back and a service desk at the front; remembering both ends avoids walking through everyone.

### 5. Key Observation
Tail tracking turns append from traversal into one link update. Last-item removal is the boundary where tail must also be cleared.

### 6. Pattern
```text
Pattern: Head-and-Tail Linked Queue
Why this pattern applies: FIFO uses opposite boundaries for add and remove
How to recognize it: constant-time queue operations without contiguous storage
Similar problems: message channels, BFS, producer-consumer buffers
```

### 7. Brute Force
Store only `head`. Enqueue walks to the current tail before attaching a fresh node. Dequeue removes and detaches the head.

#### Pseudocode
```text
enqueue: if empty set head, else walk to tail and append
dequeue: save head, advance head, detach removed node
front: read head value
track count
```

#### Brute-Force TypeScript
```typescript
class QueueNode<T> {
  constructor(
    public value: T,
    public next: QueueNode<T> | null = null,
  ) {}
}

class LinkedQueueBruteForce<T> {
  private head: QueueNode<T> | null = null;
  private count = 0;

  enqueue(value: T): void {
    const node = new QueueNode(value);
    if (this.head === null) {
      this.head = node;
    } else {
      let tail = this.head;
      while (tail.next !== null) tail = tail.next;
      tail.next = node;
    }
    this.count += 1;
  }

  dequeue(): T | undefined {
    if (this.head === null) return undefined;
    const removed = this.head;
    this.head = removed.next;
    removed.next = null;
    this.count -= 1;
    return removed.value;
  }

  front(): T | undefined {
    return this.head?.value;
  }

  size(): number {
    return this.count;
  }

  isEmpty(): boolean {
    return this.count === 0;
  }
}
```

#### Visualization
```text
head -> [A] -> [B] -> [C]
enqueue D walks A, B, C before attaching [D]
```

#### Dry Run
Enqueue `A`, then `B`; the second enqueue traverses from `A` before linking `B`. Dequeue saves `A`, moves head to `B`, detaches `A`, and returns its value.

#### Brute-Force Complexity
`enqueue` is `O(n)`; `dequeue`, `front`, `size`, and `isEmpty` are `O(1)`. The queue owns `O(n)` node space and each operation uses `O(1)` auxiliary and output space.

### 8. Why Brute Force Is Not Ideal
Every enqueue rediscovers a tail that the previous enqueue already reached. The traversal grows with queue length.

### 9. Optimal Approach
Track `head` and `tail`. On the first enqueue, both reference the fresh node. Later enqueues link from tail and advance it. Dequeue advances head, clears tail if the queue became empty, and detaches the removed node.

### 10. Visualization
```text
head                 tail
 |                     |
 v                     v
[A] -> [B] -> [C] -> null

dequeue A: head -> [B] -> [C] <- tail, [A] -> null
```

### 11. Step-by-Step Dry Run
| Operation | Head | Tail | Size | Return |
|---|---|---|---:|---|
| `enqueue("A")` | A | A | 1 | - |
| `enqueue("B")` | A | B | 2 | - |
| `dequeue()` | B | B | 1 | `"A"` |
| `dequeue()` | `null` | `null` | 0 | `"B"` |

### 12. TypeScript Solution
```typescript
class QueueNode<T> {
  constructor(
    public value: T,
    public next: QueueNode<T> | null = null,
  ) {}
}

class LinkedQueue<T> {
  private head: QueueNode<T> | null = null;
  private tail: QueueNode<T> | null = null;
  private count = 0;

  enqueue(value: T): void {
    const node = new QueueNode(value);
    if (this.tail === null) {
      this.head = node;
      this.tail = node;
    } else {
      this.tail.next = node;
      this.tail = node;
    }
    this.count += 1;
  }

  dequeue(): T | undefined {
    if (this.head === null) return undefined;
    const removed = this.head;
    this.head = removed.next;
    if (this.head === null) this.tail = null;
    removed.next = null;
    this.count -= 1;
    return removed.value;
  }

  front(): T | undefined {
    return this.head?.value;
  }

  size(): number {
    return this.count;
  }

  isEmpty(): boolean {
    return this.count === 0;
  }
}
```

### 13. Code Walkthrough
The invariant is `head === null` exactly when `tail === null` and `count === 0`. Enqueue changes the old tail's link at most once. Dequeue handles the last-node transition before detaching the removed node.

### 14. Complexity Analysis
All five operations are `O(1)`. The queue owns `O(n)` node space. Each operation uses `O(1)` auxiliary and output space; enqueue adds one persistent node.

### 15. Edge Cases
- Empty dequeue and front return `undefined`.
- First enqueue initializes both boundaries.
- Last dequeue clears both boundaries and detaches the node.
- Duplicate values live in separate nodes.
- Stored `undefined` is ambiguous with underflow.

### 16. Common Mistakes
- Updating head but leaving a stale tail after the last dequeue.
- Forgetting to advance tail after linking a node.
- Traversing from head even though tail is available.
- Exposing nodes and allowing callers to rewrite links.

### 17. Interview Follow-ups
1. Add a bounded capacity. 2. Implement a deque with doubly linked nodes. 3. Remove the count field. 4. Support async consumers.

### 18. Variations
- Circular linked queue. - Sentinel-node queue. - Lock-free queue concept. - Queue with tagged underflow.

### 19. Pattern Connections
Head/tail ownership also appears in linked deques and stream buffers. A dummy node can reduce boundary branches in more mutation-heavy queue algorithms, but it must stay internal.

### 20. NestJS Project Connection
Expose `POST /api/dsa/queue/linked`. Run only the optimized head/tail implementation and return `{ trace, truncated }` with bounded trace events such as `{ operation, size, frontValue, backValue }`; serialize values, not node identities or links.

### 21. Mini Challenge
Add `back(): T | undefined` without changing any existing operation's asymptotic complexity.

## Problem 120 — Implement Queue using Two Stacks

### 1. Problem Statement
Implement a generic FIFO queue using stack operations only. Provide `enqueue`, `dequeue`, `front`, `size`, and `isEmpty`; empty reads return `undefined`. The optimized version must use lazy transfer between input and output stacks.

### 2. Example
```text
Operations: enqueue(1), enqueue(2), front(), enqueue(3), dequeue(), dequeue(), dequeue()
Outputs:    -,          -,          1,       -,          1,         2,         3
```

### 3. Understand the Problem
A stack reverses order. Reversing a group twice lets the oldest queued value become the next stack top without using array front removal.

#### In Simple Terms
Put new values in one stack and reverse them into another stack only when the dequeue side runs out.

### 4. Real-World Analogy
New mail lands on one pile. When the outbox is empty, flip the entire pile so the oldest letter is on top for delivery.

### 5. Key Observation
An output stack already in use must not be mixed with newer input. Delaying transfer until output is empty preserves global FIFO order.

### 6. Pattern
```text
Pattern: Lazy Reversal with Two Stacks
Why this pattern applies: one reversal exposes the oldest element
How to recognize it: FIFO behavior is required from LIFO primitives
Similar problems: batched streams, functional queues, inbox/outbox buffers
```

### 7. Brute Force
Keep the queue front at the top of one stack. For every enqueue, move all current values to a buffer, push the new value, then restore the older values above it. Dequeue can then pop immediately.

#### Pseudocode
```text
enqueue value:
    move every primary top to buffer
    push value to primary
    move every buffer top back to primary
dequeue: pop primary
front: read primary top
```

#### Brute-Force TypeScript
```typescript
class TwoStackQueueBruteForce<T> {
  private readonly primary: T[] = [];

  enqueue(value: T): void {
    const buffer: T[] = [];
    while (this.primary.length > 0) buffer.push(this.primary.pop()!);
    this.primary.push(value);
    while (buffer.length > 0) this.primary.push(buffer.pop()!);
  }

  dequeue(): T | undefined {
    return this.primary.pop();
  }

  front(): T | undefined {
    return this.primary[this.primary.length - 1];
  }

  size(): number {
    return this.primary.length;
  }

  isEmpty(): boolean {
    return this.primary.length === 0;
  }
}
```

#### Visualization
```text
primary top [A, B] means A is front
enqueue C:
move A, B out -> push C -> restore B, A
primary top [A, B, C]
```

#### Dry Run
After enqueueing `1` and `2`, primary's top is `1`. Enqueue `3` moves `1` and `2` away, pushes `3`, then restores `2` and `1`; dequeue pops `1`.

#### Brute-Force Complexity
`enqueue` is `O(n)`; `dequeue`, `front`, `size`, and `isEmpty` are `O(1)`. Persistent storage is `O(n)`, and enqueue temporarily uses `O(n)` auxiliary buffer space.

### 8. Why Brute Force Is Not Ideal
Every enqueue reverses all existing values twice even if no dequeue follows. The same elements move repeatedly.

### 9. Optimal Approach
Push new values onto `input`. For `dequeue` or `front`, transfer all input values to `output` only when output is empty. Existing output values are older than every input value and must leave first.

### 10. Visualization
```text
enqueue 1, 2, 3: input top [3, 2, 1], output []
front triggers transfer:
input [], output top [1, 2, 3]
dequeue -> 1
enqueue 4: input [4], output top [2, 3]
```

### 11. Step-by-Step Dry Run
| Operation | Input top-first | Output top-first | Return |
|---|---|---|---|
| `enqueue(1)` | `[1]` | `[]` | - |
| `enqueue(2)` | `[2, 1]` | `[]` | - |
| `front()` | `[]` | `[1, 2]` | `1` |
| `enqueue(3)` | `[3]` | `[1, 2]` | - |
| `dequeue()` | `[3]` | `[2]` | `1` |

### 12. TypeScript Solution
```typescript
class TwoStackQueue<T> {
  private readonly input: T[] = [];
  private readonly output: T[] = [];

  enqueue(value: T): void {
    this.input.push(value);
  }

  dequeue(): T | undefined {
    this.moveInputIfNeeded();
    return this.output.pop();
  }

  front(): T | undefined {
    this.moveInputIfNeeded();
    return this.output[this.output.length - 1];
  }

  size(): number {
    return this.input.length + this.output.length;
  }

  isEmpty(): boolean {
    return this.size() === 0;
  }

  private moveInputIfNeeded(): void {
    if (this.output.length > 0) return;
    while (this.input.length > 0) {
      this.output.push(this.input.pop()!);
    }
  }
}
```

### 13. Code Walkthrough
Enqueue never disturbs older output. The private transfer runs only at an empty output boundary. Each value is pushed to input once, moved to output once, and popped once.

### 14. Complexity Analysis
`enqueue` is amortized `O(1)`. `dequeue` and `front` are amortized `O(1)` but worst-case `O(n)` when they trigger transfer. `size` and `isEmpty` are `O(1)`. The queue owns `O(n)` total stack space; each operation has `O(1)` output, while a transfer uses existing owned arrays rather than a third `O(n)` buffer.

### 15. Edge Cases
- Empty dequeue and front transfer nothing and return `undefined`.
- New input never jumps ahead of older output.
- Repeated front calls do not repeat transfer.
- Size sums both stacks during split state.
- Stored `undefined` is ambiguous with underflow but still transfers correctly.

### 16. Common Mistakes
- Transferring on every enqueue.
- Moving input while output still contains older values.
- Reporting only one stack's length as size.
- Calling a transfer-triggering operation worst-case `O(1)` instead of amortized `O(1)`.

### 17. Interview Follow-ups
1. Prove the amortized bound with aggregate analysis. 2. Return tagged results. 3. Make the queue immutable. 4. Compare with a circular buffer.

### 18. Variations
- Costly-enqueue queue. - Recursive one-stack queue. - Functional two-list queue. - Bounded two-stack queue.

### 19. Pattern Connections
Lazy transfer is batched reversal. It resembles amortized resizing and delayed cleanup: one expensive phase pays for many subsequent cheap operations.

### 20. NestJS Project Connection
Expose `POST /api/dsa/queue/two-stacks`. Run only the optimized lazy-transfer implementation and return `{ trace, truncated }` with bounded trace events such as `{ operation, phase, inputSize, outputSize, front }`, including transfer steps.

### 21. Mini Challenge
Write an aggregate proof that any sequence of `m` queue operations performs at most `2m` internal value moves before final pops.

## Problem 121 — Implement Stack using Two Queues

### 1. Problem Statement
Implement a generic LIFO stack using queue operations. Provide `push`, `pop`, `peek`, `size`, and `isEmpty`; empty reads return `undefined`. Optimize for pop-heavy use by making push `O(n)` and keeping the newest value at the primary queue's front.

### 2. Example
```text
Operations: push(1), push(2), peek(), push(3), pop(), pop(), pop(), pop()
Outputs:    -,       -,       2,      -,       3,     2,     1,     undefined
```

### 3. Understand the Problem
Queues expose their oldest value, while a stack needs its newest value. Rotating all older queued values behind each newcomer makes that newcomer immediately accessible.

#### In Simple Terms
On every push, put the new value in an empty queue first, move all older values behind it, and swap queue roles.

### 4. Real-World Analogy
To make the newest document the next one served, open a new line with it first, then move the entire old line behind it.

### 5. Key Observation
There is no asymptotically best two-queue stack for every operation. Paying during push is a deliberate contract that benefits frequent pop and peek calls.

### 6. Pattern
```text
Pattern: Costly Push by Queue Rotation
Why this pattern applies: primary front must equal stack top
How to recognize it: LIFO behavior is required from FIFO primitives
Similar problems: queue rotation, scheduling order inversion, deque simulation
```

### 7. Brute Force
Keep pushes cheap by appending to one array queue. For pop, repeatedly `shift` older values into a buffer until only the newest remains, remove it, then swap arrays. This is simple and valid but repeated front deletion causes extra copying.

#### Pseudocode
```text
push value: append to active queue
pop:
    while active has more than one value:
        shift front into buffer
    shift and save newest value
    replace active with buffer
peek: read active's last value
```

#### Brute-Force TypeScript
```typescript
class TwoQueueStackBruteForce<T> {
  private active: T[] = [];

  push(value: T): void {
    this.active.push(value);
  }

  pop(): T | undefined {
    if (this.active.length === 0) return undefined;
    const buffer: T[] = [];
    while (this.active.length > 1) buffer.push(this.active.shift()!);
    const value = this.active.shift();
    this.active = buffer;
    return value;
  }

  peek(): T | undefined {
    return this.active[this.active.length - 1];
  }

  size(): number {
    return this.active.length;
  }

  isEmpty(): boolean {
    return this.active.length === 0;
  }
}
```

#### Visualization
```text
active front [A, B, C] back, where C is stack top
pop C: shift A, then B into buffer; shift C; swap
```

#### Dry Run
Push `1`, `2`, `3` to get `[1, 2, 3]`. Pop shifts `1` and `2` into a buffer, removes `3`, and makes `[1, 2]` active.

#### Brute-Force Complexity
`push`, `peek`, `size`, and `isEmpty` are `O(1)`. Because each JavaScript `shift` is linear and pop performs repeated shifts, `pop` is `O(n²)` in this literal array implementation. Persistent and temporary space are `O(n)`; output is `O(1)`.

### 8. Why Brute Force Is Not Ideal
It optimizes the wrong boundary for a pop-heavy contract and compounds queue rotation with repeated array reindexing.

### 9. Optimal Approach
Use two internal head-index queues whose dequeue is amortized `O(1)`. Push enqueues the new value into the empty secondary queue, drains primary behind it, swaps references, and clears the now-empty secondary queue. Pop and peek then use primary's front.

### 10. Visualization
```text
primary front [B, A], secondary []
push C:
secondary [C]
drain primary -> secondary [C, B, A]
swap -> primary front [C, B, A]
```

### 11. Step-by-Step Dry Run
| Operation | Primary front-first | Return |
|---|---|---|
| `push(1)` | `[1]` | - |
| `push(2)` | `[2, 1]` | - |
| `peek()` | `[2, 1]` | `2` |
| `push(3)` | `[3, 2, 1]` | - |
| `pop()` | `[2, 1]` | `3` |

### 12. TypeScript Solution
```typescript
class ArrayQueue<T> {
  private items: T[] = [];
  private head = 0;

  enqueue(value: T): void {
    this.items.push(value);
  }

  dequeue(): T {
    if (this.isEmpty()) throw new Error("internal queue underflow");
    const value = this.items[this.head];
    this.head += 1;
    if (this.head === this.items.length) this.clear();
    return value;
  }

  front(): T {
    if (this.isEmpty()) throw new Error("internal queue underflow");
    return this.items[this.head];
  }

  size(): number {
    return this.items.length - this.head;
  }

  isEmpty(): boolean {
    return this.size() === 0;
  }

  clear(): void {
    this.items = [];
    this.head = 0;
  }
}

class TwoQueueStack<T> {
  private primary = new ArrayQueue<T>();
  private secondary = new ArrayQueue<T>();

  push(value: T): void {
    this.secondary.enqueue(value);
    while (!this.primary.isEmpty()) {
      this.secondary.enqueue(this.primary.dequeue());
    }
    [this.primary, this.secondary] = [this.secondary, this.primary];
    this.secondary.clear();
  }

  pop(): T | undefined {
    return this.primary.isEmpty() ? undefined : this.primary.dequeue();
  }

  peek(): T | undefined {
    return this.primary.isEmpty() ? undefined : this.primary.front();
  }

  size(): number {
    return this.primary.size();
  }

  isEmpty(): boolean {
    return this.primary.isEmpty();
  }
}
```

### 13. Code Walkthrough
The internal queue throws only on invariant violations; public stack underflow still returns `undefined`. `push` calls dequeue only after `isEmpty` is false, so values including `undefined` rotate without being confused with internal underflow. After swapping, secondary is empty and reset for the next push.

### 14. Complexity Analysis
`push` is `O(n)` because it moves the `n` existing values. `pop` and `peek` are amortized `O(1)` under the head-index queue, and `size`/`isEmpty` are `O(1)`. The two queues own `O(n)` total live storage; push uses `O(n)` auxiliary queue capacity during rotation, and outputs are `O(1)`.

### 15. Edge Cases
- Empty public pop and peek return `undefined`.
- First push swaps a one-item secondary into primary.
- Last pop resets the internal primary queue.
- Duplicate and `undefined` values preserve position, though public `undefined` remains ambiguous.
- Repeated pushes leave secondary empty after every swap.

### 16. Common Mistakes
- Draining the newcomer behind old values instead of placing it first.
- Forgetting to swap queue roles.
- Using `shift` inside the optimized internal queue.
- Calling this variant universally optimal instead of identifying its pop-heavy tradeoff.

### 17. Interview Follow-ups
1. Implement the costly-pop variant with head-index queues. 2. Use one queue by rotating after push. 3. Prove the primary-front invariant. 4. Compare operation mixes.

### 18. Variations
- One-queue stack. - Costly-pop two-queue stack. - Tagged underflow. - Bounded internal queues.

### 19. Pattern Connections
Queue rotation changes which boundary is cheap. The same tradeoff analysis appears in eager versus lazy transfer and in choosing index versus value storage for monotonic structures.

### 20. NestJS Project Connection
Expose `POST /api/dsa/stack/two-queues`. Run only the optimized costly-push head-index implementation and return `{ trace, truncated }` with bounded trace events like `{ operation, phase, primarySize, secondarySize, top }`; cap per-value drain events.

### 21. Mini Challenge
Implement the one-queue costly-push variant and compare its invariants and auxiliary queue count with this solution.

## Problem 122 — Valid Parentheses

### 1. Problem Statement
Given a string containing only the six ASCII bracket characters `()[]{}`, return `true` exactly when every opening bracket is closed by the same bracket type in the correct nesting order. The empty string is valid. Any other Unicode code point must throw `RangeError` before matching begins, so validation is atomic rather than interleaved with an early algorithm result.

### 2. Example
```text
Input:  "{[()]}"   Output: true
Input:  "([)]"     Output: false
Input:  ""         Output: true
Input:  "()a"      Throws: RangeError before bracket matching
```

### 3. Understand the Problem
A closing bracket must match the most recent unmatched opening bracket, not merely an opening bracket somewhere earlier. First scan the whole string with `for...of` and reject any non-bracket code point; because valid input is ASCII, later string indexing is safe.

#### In Simple Terms
Finish checking that every character is allowed, then use last-in, first-out order so each closer meets the opener waiting for it.

### 4. Real-World Analogy
Nested brackets are like nested boxes: the last box opened must be the first box closed. Closing an outer box while an inner one remains open breaks the nesting.

### 5. Key Observation
The next legal closer is determined by the latest unmatched opener. A stack stores exactly that pending order, while a count or unordered collection loses bracket type and nesting information.

### 6. Pattern
```text
Pattern: Stack-Based Delimiter Matching
Why this pattern applies: closers resolve unmatched openers in reverse order
How to recognize it: nested pairs must be matched by both type and position
Similar problems: expression parsing, HTML tag validation, compiler syntax checks
```

### 7. Brute Force
After a complete validation pass, repeatedly remove every adjacent `()`, `[]`, and `{}` pair. Properly nested input eventually becomes empty because each innermost pair becomes adjacent; malformed input reaches a nonempty fixed point. This is clearer than a regex loop because regex adds no structural insight, can hide replacement-order assumptions, and still creates new strings.

#### Pseudocode
```text
validate every code point is one of ( ) [ ] { }; otherwise throw RangeError
remaining = input
repeat:
    next = remaining with (), [], and {} removed
    if next equals remaining: return next is empty
    remaining = next
```

#### Brute-Force TypeScript
```typescript
function isValidParenthesesBruteForce(input: string): boolean {
  const allowed = new Set(["(", ")", "[", "]", "{", "}"]);
  for (const character of input) {
    if (!allowed.has(character)) {
      throw new RangeError("input must contain only ASCII bracket characters");
    }
  }

  let remaining = input;
  while (true) {
    const next = remaining.replaceAll("()", "").replaceAll("[]", "").replaceAll("{}", "");
    if (next === remaining) return next.length === 0;
    remaining = next;
  }
}
```

#### Visualization
```text
{[()]} -> {[]} -> {} -> "" -> valid
([)]   -> ([)]            -> unchanged and nonempty -> invalid
```

#### Dry Run
For `{[()]}`, validation first accepts all six code points. Replacement passes produce `{[]}`, then `{}`, then the empty string, so the function returns `true`. For `()a`, validation sees `a` and throws before any pair is removed.

#### Brute-Force Complexity
Validation is `O(n)`. Up to `O(n)` replacement rounds can each scan and allocate `O(n)` characters, giving `O(n²)` time and `O(n)` peak string space. The returned boolean uses `O(1)` output space.

### 8. Why Brute Force Is Not Ideal
Immutable replacement repeatedly rescans and copies characters that a single stack pass can settle once. It also expresses nesting indirectly through convergence instead of maintaining the unmatched-openers invariant.

### 9. Optimal Approach
Validate the complete input first. Then scan by index and push the expected closing character for each opener. For a closer, pop and compare; a mismatch or missing expectation returns `false` immediately. Return `true` only when the stack is empty after the scan.

### 10. Visualization
```text
Input:  { [ ( ) ] }
Action: +} +] +) -) -] -}
Stack:  [}] [},]] [},],)] [},]] [}] []

Input:  ( [ ) ]
Stack:  [)] [),]]
Read ): expected ] -> mismatch -> false
```

### 11. Step-by-Step Dry Run
| Index | Character | Expected-closing stack | Result |
|---:|:---:|---|---|
| 0 | `{` | `['}']` | continue |
| 1 | `[` | `['}', ']']` | continue |
| 2 | `(` | `['}', ']', ')']` | continue |
| 3 | `)` | `['}', ']']` | match |
| 4 | `]` | `['}']` | match |
| 5 | `}` | `[]` | match; final stack empty |

### 12. TypeScript Solution
```typescript
function isValidParentheses(input: string): boolean {
  const allowed = new Set(["(", ")", "[", "]", "{", "}"]);
  for (const character of input) {
    if (!allowed.has(character)) {
      throw new RangeError("input must contain only ASCII bracket characters");
    }
  }

  const expectedClosers: string[] = [];
  const closerFor: Record<string, string> = {
    "(": ")",
    "[": "]",
    "{": "}",
  };

  for (let index = 0; index < input.length; index += 1) {
    const character = input[index];
    const expectedCloser = closerFor[character];
    if (expectedCloser !== undefined) {
      expectedClosers.push(expectedCloser);
    } else if (expectedClosers.pop() !== character) {
      return false;
    }
  }

  return expectedClosers.length === 0;
}
```

### 13. Code Walkthrough
The first loop establishes the six-character input contract for the entire string, even when a malformed prefix could otherwise return early. The second loop uses string indexes safely because every accepted code point is one ASCII code unit. Storing expected closers makes each close operation a direct comparison, and a final nonempty stack identifies unclosed openers.

### 14. Complexity Analysis
The validation pass and matching pass each take `O(n)`, so total time is `O(n)`. The stack owns at most `n` expected closers, giving `O(n)` auxiliary space and `O(1)` output space. Early mismatch shortens only the matching pass; full validation has already completed.

### 15. Edge Cases
- Empty input returns `true`.
- A single opener or closer returns `false`.
- Adjacent pairs such as `()[]{}` return `true`.
- Correct counts with wrong nesting, such as `([)]`, return `false`.
- Unclosed openers, such as `((`, leave a nonempty stack.
- Whitespace, letters, emoji, and every other non-bracket code point throw `RangeError` before matching.

### 16. Common Mistakes
- Checking only bracket counts and missing nesting order.
- Validating while matching, which can return `false` before discovering a later invalid code point.
- Returning `true` without checking for leftover openers.
- Using a regex as though it can directly model arbitrary nesting.
- Iterating Unicode by index during validation and accidentally reasoning about code units as code points.

### 17. Interview Follow-ups
1. Return the first mismatch index. 2. Report the expected and actual closer. 3. Support configurable delimiter pairs. 4. Ignore brackets inside quoted strings.

### 18. Variations
- Longest valid parentheses substring. - Minimum removals for validity. - Wildcard brackets. - Streaming validation across chunks.

### 19. Pattern Connections
This is the same stack discipline used by parsers, recursive call stacks, and depth-first traversal: the most recently opened context must finish first. It also illustrates why validation and computation may need separate passes when errors have precedence over normal results.

### 20. NestJS Project Connection
Expose `POST /api/dsa/parentheses/validate` with body `{ input: string }`. Map the optimized algorithm's `RangeError` to HTTP `400`; otherwise return `{ valid, trace, truncated }`. Trace only the optimized pass after validation with bounded events such as `{ index, character, action, expected, depth }`, cap the event count, and never run the brute-force implementation in the endpoint.

### 21. Mini Challenge
Extend the optimized solution to return `{ valid, mismatchIndex, expected, actual }` while preserving the rule that any invalid code point takes precedence over bracket-matching results.

## Problem 123 — Min Stack

### 1. Problem Statement
Design a stack for numeric safe integers with `push`, `pop`, `top`, `getMin`, `size`, and `isEmpty`. `push` must validate with `Number.isSafeInteger` before mutation and throw `RangeError` otherwise. On an empty stack, `pop`, `top`, and `getMin` return `undefined`; all stored values are numbers, so that sentinel is unambiguous.

### 2. Example
```text
Operations: push(-2), push(0), push(-3), getMin(), pop(), getMin()
Outputs:    -,        -,       -,        -3,       -3,    -2
```

### 3. Understand the Problem
A normal stack can read its top in constant time, but finding the minimum by scanning costs linear time. The optimized representation must retain the minimum for every stack state, including states created by duplicate minimum values.

#### In Simple Terms
Store each pushed value together with the smallest value seen at or below it, so removing the top also restores the previous minimum.

### 4. Real-World Analogy
Imagine writing the current lowest temperature beside every new diary entry. Tearing off the latest page reveals both the previous entry and the minimum that was current then.

### 5. Key Observation
After pushing `value`, the new minimum is `min(value, previousMinimum)`. Recording that snapshot in the same entry makes pop automatically reveal the correct prior minimum, and duplicate minima require no special branch.

### 6. Pattern
```text
Pattern: Stack with Aggregated State
Why this pattern applies: each prefix needs a minimum available in constant time
How to recognize it: a LIFO structure must answer a reversible aggregate query
Similar problems: max stack, encoded browser history, prefix aggregates with rollback
```

### 7. Brute Force
Store safe integers in an array with the top at the end. Validate before push so a rejected value cannot alter state. Implement `getMin` by scanning all current values and retaining the smallest.

#### Pseudocode
```text
push value:
    if value is not a safe integer: throw RangeError
    append value
pop/top: use the array end; return undefined when empty
getMin:
    if empty: return undefined
    scan values and return the smallest
```

#### Brute-Force TypeScript
```typescript
class MinStackBruteForce {
  private values: number[] = [];

  push(value: number): void {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("value must be a safe integer");
    }
    this.values.push(value);
  }

  pop(): number | undefined {
    return this.values.pop();
  }

  top(): number | undefined {
    return this.values[this.values.length - 1];
  }

  getMin(): number | undefined {
    if (this.values.length === 0) return undefined;
    let minimum = this.values[0];
    for (let index = 1; index < this.values.length; index += 1) {
      minimum = Math.min(minimum, this.values[index]);
    }
    return minimum;
  }

  size(): number {
    return this.values.length;
  }

  isEmpty(): boolean {
    return this.values.length === 0;
  }
}
```

#### Visualization
```text
values: [-2, 0, -3] <- top
getMin scans: -2 -> -2 -> -3
pop removes -3; next scan returns -2
```

#### Dry Run
Starting empty, pushes of `-2`, `0`, and `-3` create `[-2, 0, -3]`. `getMin` scans and returns `-3`; `pop` returns `-3`; the next scan sees `[-2, 0]` and returns `-2`.

#### Brute-Force Complexity
`push`, `pop`, `top`, `size`, and `isEmpty` are amortized `O(1)` with an array end. `getMin` is `O(n)`. The class owns `O(n)` storage, `getMin` uses `O(1)` auxiliary space, and each operation returns `O(1)` output.

### 8. Why Brute Force Is Not Ideal
Repeated minimum queries rescan values whose prefix minima could have been recorded once during push. A workload with `n` pushes followed by `n` minimum queries can therefore take `O(n²)` time.

### 9. Optimal Approach
Keep one array of entries `{ value, minimum }`. Before mutation, reject any value that is not a safe integer. On push, store the smaller of the new value and the previous top entry's minimum; then pop, top, and getMin can all read the array end in constant time.

### 10. Visualization
```text
push(-2)  [{ value: -2, minimum: -2 }]
push(0)   [{ value: -2, minimum: -2 }, { value: 0,  minimum: -2 }]
push(-3)  [{ value: -2, minimum: -2 }, { value: 0,  minimum: -2 },
           { value: -3, minimum: -3 }]
pop()     [{ value: -2, minimum: -2 }, { value: 0,  minimum: -2 }]
```

### 11. Step-by-Step Dry Run
| Operation | Entry stack `(value, minimum)` | Return |
|---|---|---:|
| `push(-2)` | `[(-2, -2)]` | - |
| `push(0)` | `[(-2, -2), (0, -2)]` | - |
| `push(-3)` | `[(-2, -2), (0, -2), (-3, -3)]` | - |
| `getMin()` | unchanged | `-3` |
| `pop()` | `[(-2, -2), (0, -2)]` | `-3` |
| `getMin()` | unchanged | `-2` |

Duplicate minimum trace:

| Operation | Entry stack `(value, minimum)` | Minimum |
|---|---|---:|
| `push(2)` | `[(2, 2)]` | `2` |
| `push(2)` | `[(2, 2), (2, 2)]` | `2` |
| `pop()` | `[(2, 2)]` | `2` |

### 12. TypeScript Solution
```typescript
type MinStackEntry = {
  value: number;
  minimum: number;
};

class MinStack {
  private entries: MinStackEntry[] = [];

  push(value: number): void {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("value must be a safe integer");
    }
    const previousMinimum = this.entries[this.entries.length - 1]?.minimum;
    const minimum = previousMinimum === undefined ? value : Math.min(value, previousMinimum);
    this.entries.push({ value, minimum });
  }

  pop(): number | undefined {
    return this.entries.pop()?.value;
  }

  top(): number | undefined {
    return this.entries[this.entries.length - 1]?.value;
  }

  getMin(): number | undefined {
    return this.entries[this.entries.length - 1]?.minimum;
  }

  size(): number {
    return this.entries.length;
  }

  isEmpty(): boolean {
    return this.entries.length === 0;
  }
}
```

### 13. Code Walkthrough
`push` performs its safe-integer check before reading or changing stack state, so failure is atomic. Each entry owns the minimum for the prefix ending at that entry. Array-end access implements the stack boundary, optional chaining supplies the empty `undefined` contract, and identical minimum snapshots preserve duplicate minima naturally.

### 14. Complexity Analysis
`push`, `pop`, `top`, `getMin`, `size`, and `isEmpty` are amortized `O(1)`; array growth makes the mutating bounds amortized rather than strict worst-case. The entries array owns `O(n)` space, each entry adds `O(1)` metadata, auxiliary working space per operation is `O(1)`, and output space is `O(1)`.

### 15. Edge Cases
- Empty `pop`, `top`, and `getMin` return `undefined`; `size` is `0` and `isEmpty` is `true`.
- One value is both the top and minimum.
- Duplicate minima remain correct after one duplicate is popped.
- Negative values and both safe-integer boundaries are accepted.
- `NaN`, infinities, fractions, and integers outside the safe range throw `RangeError` before mutation.
- Because only numbers are stored, `undefined` cannot be a stored value and underflow is unambiguous.

### 16. Common Mistakes
- Tracking one mutable minimum without retaining the previous minimum for pop.
- Updating only for strictly smaller values in a two-stack design and losing duplicate minima.
- Validating after push and leaving invalid state when validation fails.
- Using `shift` or `unshift` instead of the array end.
- Claiming strict worst-case `O(1)` push for a dynamically growing array.

### 17. Interview Follow-ups
1. Implement the equivalent two-stack design. 2. Add `getMax` in `O(1)`. 3. Generalize to a caller-supplied comparator. 4. Discuss a tagged underflow result if stored values could include `undefined`.

### 18. Variations
- Max stack. - Min-max stack. - Persistent immutable min stack. - Queue with constant-time minimum. - Stack with constant-time aggregate rollback.

### 19. Pattern Connections
Per-entry minima are prefix aggregates aligned with LIFO rollback. The same idea stores maximums, greatest common divisors, or parser state snapshots when removing the latest item must restore the previous aggregate immediately.

### 20. NestJS Project Connection
Expose `POST /api/dsa/min-stack/execute` with body `{ operations }`. Validate the request shape, then run only the optimized entry-stack implementation. Map invalid push values to HTTP `400` and return `{ outputs, trace, truncated }` with bounded post-operation events such as `{ operation, value, size, top, minimum }`; never expose the private entries array.

### 21. Mini Challenge
Implement a min-max stack whose entries preserve both aggregates, then prove that duplicate minima and maxima remain correct through arbitrary pushes and pops.

## Problem 124 — Next Greater Element

### 1. Problem Statement
Given a readonly array of safe integers, return an array where each position contains the value of the first element strictly greater than the current value to its right, or `null` when no such element exists. Equal values are not greater. Validate every input value with `Number.isSafeInteger` before running the algorithm, and never mutate the input.

### 2. Example
```text
Input:  [2, 1, 2, 4, 3]
Output: [4, 2, 4, null, null]
```

### 3. Understand the Problem
For each index, search only to its right and stop at the first value that is strictly larger. The result stores values, not indexes: for index `0`, the qualifying position is index `3`, but the returned entry is `4`.

#### In Simple Terms
For every number, report the first bigger number that appears later; use `null` if one never appears.

### 4. Real-World Analogy
Imagine people standing in a line and looking right for the first taller person. The answer records that person's height, not how far away the person stands.

### 5. Key Observation
While scanning from right to left, a value less than or equal to the current value can never help the current position or any earlier position for which the current value is closer and at least as useful. Removing `<= current` values leaves the nearest viable greater candidate on top.

### 6. Pattern
```text
Pattern: Monotonic Decreasing Candidate Stack
Why this pattern applies: dominated right-side values can be discarded permanently
How to recognize it: each position asks for the first later value satisfying a strict comparison
Similar problems: next smaller element, daily temperatures, stock span
```

### 7. Brute Force
Validate the complete input first. For each index, scan later positions from left to right and append the first value greater than the current value. Append `null` if the scan reaches the end.

#### Pseudocode
```text
validate every value is a safe integer; otherwise throw RangeError
for each index:
    answer = null
    for each later index from left to right:
        if later value > current value:
            answer = later value
            stop the inner scan
    append answer to results
return results
```

#### Brute-Force TypeScript
```typescript
function nextGreaterElementBruteForce(values: readonly number[]): Array<number | null> {
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
  }

  return values.map((value, index) => {
    for (let nextIndex = index + 1; nextIndex < values.length; nextIndex += 1) {
      if (values[nextIndex] > value) return values[nextIndex];
    }
    return null;
  });
}
```

#### Visualization
```text
Index 0, value 2: inspect 1, 2, 4 -> first greater value is 4
Index 1, value 1: inspect 2       -> first greater value is 2
Index 2, value 2: inspect 4       -> first greater value is 4
Index 3, value 4: inspect 3       -> none
Index 4, value 3: no values right -> none
```

#### Dry Run
For `[2, 1, 2, 4, 3]`, the scan for the first `2` passes over `1` and the equal `2` before finding `4`. The scan for `1` stops immediately at `2`. The second `2` also finds `4`, while `4` and `3` have no strictly greater value to their right.

#### Brute-Force Complexity
Validation takes `O(n)`. The nested scans take `O(n²)` time in the worst case, such as a nonincreasing array. The algorithm uses `O(1)` auxiliary space beyond the returned `O(n)` array.

### 8. Why Brute Force Is Not Ideal
Neighboring searches repeatedly inspect the same suffix values. A nonincreasing input forces nearly every position to scan all remaining positions even though those failed comparisons can be summarized.

### 9. Optimal Approach
After full validation, scan from right to left with a stack of candidate values. Pop while the top is less than or equal to the current value because neither a smaller nor an equal value is strictly greater. The remaining top is the first greater value, or the empty stack means `null`; then push the current value for earlier positions.

### 10. Visualization
```text
Input: [2, 1, 2, 4, 3]

current  stack after pops  answer  stack after push
3        []                null    [3]
4        []                null    [4]
2        [4]               4       [4, 2]
1        [4, 2]            2       [4, 2, 1]
2        [4]               4       [4, 2]
```

### 11. Step-by-Step Dry Run
| Index | Current | Values popped | Candidate top | Result |
|---:|---:|---|---:|---:|
| 4 | 3 | none | none | `null` |
| 3 | 4 | `3` | none | `null` |
| 2 | 2 | none | `4` | `4` |
| 1 | 1 | none | `2` | `2` |
| 0 | 2 | `1`, equal `2` | `4` | `4` |

### 12. TypeScript Solution
```typescript
function nextGreaterElement(values: readonly number[]): Array<number | null> {
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
  }

  const result: Array<number | null> = new Array(values.length).fill(null);
  const candidates: number[] = [];

  for (let index = values.length - 1; index >= 0; index -= 1) {
    const value = values[index];
    while (candidates.length > 0 && candidates[candidates.length - 1] <= value) {
      candidates.pop();
    }
    result[index] = candidates[candidates.length - 1] ?? null;
    candidates.push(value);
  }

  return result;
}
```

### 13. Code Walkthrough
The validation pass finishes before either working array is created. During the right-to-left pass, `candidates` contains useful values from the suffix in nearest-candidate order. The `<=` pop condition enforces strict greater-than semantics, and the code writes the top value rather than an index. Only local result and stack arrays change; the readonly input is never mutated.

### 14. Complexity Analysis
Validation is `O(n)`. In the algorithm, each value is pushed once and popped at most once, so all `while` iterations total `O(n)` rather than `O(n²)`. Total time is `O(n)`, auxiliary stack space is `O(n)`, and the returned array uses `O(n)` output space.

### 15. Edge Cases
- Empty input returns `[]`.
- A single value returns `[null]`.
- Strictly decreasing values all return `null`.
- Strictly increasing values return the next adjacent value except at the end.
- Equal values are popped and never count as greater.
- Negative values and both safe-integer boundaries are accepted.
- Any fraction, infinity, `NaN`, or unsafe integer throws `RangeError` before computation.

### 16. Common Mistakes
- Using `>` instead of `<=` in the pop condition and allowing equal values to survive.
- Returning the candidate's index when the contract requires its value.
- Scanning left to right without retaining enough unresolved positions.
- Validating during computation and potentially producing partial internal work first.
- Claiming `O(n²)` because the code contains a nested `while`; each value is popped only once.

### 17. Interview Follow-ups
1. Return indexes instead of values. 2. Return distances to the next greater value. 3. Use greater-than-or-equal semantics. 4. Process a circular array.

### 18. Variations
- Next greater element to the left. - Next greater frequency. - Circular next greater element. - Next greater node in a linked list.

### 19. Pattern Connections
This is a monotonic-stack dominance argument: once the current value removes a candidate, that candidate cannot be the first strictly greater value for any earlier position. Storing values is sufficient here because the contract asks for values and no distance calculation needs an index.

### 20. NestJS Project Connection
Expose `POST /api/dsa/next-greater` with body `{ values: number[] }`. Validate the request shape, run only the optimized function, map `RangeError` to HTTP `400`, and return `{ result, trace, truncated }`. Bound trace events such as `{ index, value, popped, candidate }` and never execute the quadratic version in the endpoint.

### 21. Mini Challenge
Change the function to return the distance to the first strictly greater value, using `0` when none exists. Determine why the stack must then store indexes instead of only values.

## Problem 125 — Next Smaller Element

### 1. Problem Statement
Given a readonly array of safe integers, return an array where each position contains the value of the first element strictly smaller than the current value to its right, or `null` when no such element exists. Equal values are not smaller. Validate every input value with `Number.isSafeInteger` before running the algorithm, and never mutate the input.

### 2. Example
```text
Input:  [4, 8, 5, 2, 25]
Output: [2, 5, 2, null, null]
```

### 3. Understand the Problem
Each position needs the nearest later value that is strictly lower, not the smallest value anywhere in the suffix. The answer stores that value rather than its index.

#### In Simple Terms
For every number, report the first smaller number that appears later; use `null` if one never appears.

### 4. Real-World Analogy
Think of each building looking right for the first shorter building. A very short building farther away does not matter if another shorter building appears sooner.

### 5. Key Observation
Scanning from right to left, any candidate greater than or equal to the current value is dominated: it is not smaller than the current value, and the current value is closer for earlier positions. Popping `>= current` leaves the nearest viable smaller value on top.

### 6. Pattern
```text
Pattern: Monotonic Increasing Candidate Stack
Why this pattern applies: dominated right-side values can be discarded permanently
How to recognize it: each position asks for the first later value satisfying a strict smaller comparison
Similar problems: next greater element, largest rectangle, sum of subarray minimums
```

### 7. Brute Force
Validate the whole input first. For every index, inspect later positions in order and append the first value strictly smaller than the current one. Append `null` when no later value qualifies.

#### Pseudocode
```text
validate every value is a safe integer; otherwise throw RangeError
for each index:
    answer = null
    for each later index from left to right:
        if later value < current value:
            answer = later value
            stop the inner scan
    append answer to results
return results
```

#### Brute-Force TypeScript
```typescript
function nextSmallerElementBruteForce(values: readonly number[]): Array<number | null> {
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
  }

  return values.map((value, index) => {
    for (let nextIndex = index + 1; nextIndex < values.length; nextIndex += 1) {
      if (values[nextIndex] < value) return values[nextIndex];
    }
    return null;
  });
}
```

#### Visualization
```text
Index 0, value 4: inspect 8, 5, 2 -> first smaller value is 2
Index 1, value 8: inspect 5       -> first smaller value is 5
Index 2, value 5: inspect 2       -> first smaller value is 2
Index 3, value 2: inspect 25      -> none
Index 4, value 25: no values right -> none
```

#### Dry Run
For `[4, 8, 5, 2, 25]`, `4` skips `8` and `5` before finding `2`. The values `8` and `5` immediately find `5` and `2`, respectively. Neither `2` nor the final `25` has a smaller value to its right.

#### Brute-Force Complexity
Validation takes `O(n)`. Nested suffix scans take `O(n²)` worst-case time, such as for a nondecreasing array. Auxiliary space is `O(1)` beyond the returned `O(n)` array.

### 8. Why Brute Force Is Not Ideal
Many positions repeatedly scan the same suffix and rediscover that the same values are not smaller. On a nondecreasing array, almost every possible pair is compared.

### 9. Optimal Approach
After complete validation, scan right to left with a stack of candidate values. Pop while the top is greater than or equal to the current value; equality must be removed because the result must be strictly smaller. Record the remaining top or `null`, then push the current value.

### 10. Visualization
```text
Input: [4, 8, 5, 2, 25]

current  stack after pops  answer  stack after push
25       []                null    [25]
2        []                null    [2]
5        [2]               2       [2, 5]
8        [2, 5]            5       [2, 5, 8]
4        [2]               2       [2, 4]
```

### 11. Step-by-Step Dry Run
| Index | Current | Values popped | Candidate top | Result |
|---:|---:|---|---:|---:|
| 4 | 25 | none | none | `null` |
| 3 | 2 | `25` | none | `null` |
| 2 | 5 | none | `2` | `2` |
| 1 | 8 | none | `5` | `5` |
| 0 | 4 | `8`, `5` | `2` | `2` |

### 12. TypeScript Solution
```typescript
function nextSmallerElement(values: readonly number[]): Array<number | null> {
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
  }

  const result: Array<number | null> = new Array(values.length).fill(null);
  const candidates: number[] = [];

  for (let index = values.length - 1; index >= 0; index -= 1) {
    const value = values[index];
    while (candidates.length > 0 && candidates[candidates.length - 1] >= value) {
      candidates.pop();
    }
    result[index] = candidates[candidates.length - 1] ?? null;
    candidates.push(value);
  }

  return result;
}
```

### 13. Code Walkthrough
Validation completes before allocating algorithm state. The right-to-left pass removes candidates that cannot satisfy strict smaller-than semantics. The `>=` condition is the mirror of Problem 124's `<=` condition, and the stack top is written as a value, not an index. Local arrays change while the readonly input remains untouched.

### 14. Complexity Analysis
Validation is `O(n)`. Each value enters the candidate stack once and can leave it only once, so the total number of pushes and pops is linear. Total time is `O(n)`, auxiliary stack space is `O(n)`, and output space is `O(n)`.

### 15. Edge Cases
- Empty input returns `[]`.
- A single value returns `[null]`.
- Strictly increasing values all return `null`.
- Strictly decreasing values return the adjacent next value except at the end.
- Equal values are popped and never count as smaller.
- Negative safe integers work without special handling.
- Invalid numeric values throw `RangeError` before computation.

### 16. Common Mistakes
- Keeping equal candidates by popping only `>` rather than `>=`.
- Returning the minimum suffix value instead of the first smaller value.
- Returning indexes despite the value-based contract.
- Mutating the caller's array with `reverse`.
- Treating the nested `while` as quadratic without counting each candidate's single pop.

### 17. Interview Follow-ups
1. Return indexes or distances. 2. Find the next smaller value in a circular array. 3. Change the comparison to smaller-than-or-equal. 4. Compute previous smaller values.

### 18. Variations
- Previous smaller element. - Next smaller element in a linked list. - Largest rectangle in a histogram. - Remove digits to form the smallest number.

### 19. Pattern Connections
Next smaller and next greater are symmetric monotonic-stack problems, but their pop comparisons are not interchangeable. Strictly smaller requires popping `>=`; strictly greater requires popping `<=`. That equality choice determines whether duplicates can become answers.

### 20. NestJS Project Connection
Expose `POST /api/dsa/next-smaller` with body `{ values: number[] }`. Run only the optimized function after DTO shape validation, translate `RangeError` to HTTP `400`, and return `{ result, trace, truncated }`. Bound optimized trace events such as `{ index, value, popped, candidate }` to avoid response growth tied to internal operations.

### 21. Mini Challenge
Return both the next smaller value and its distance for every position while retaining `O(n)` time. Explain which stack data must change.

## Problem 126 — Daily Temperatures

### 1. Problem Statement
Given a readonly array of safe-integer daily temperatures, return an array where each position contains the number of days until a future strictly warmer temperature. Return `0` when no warmer day exists. Equal temperatures are not warmer. Validate every temperature with `Number.isSafeInteger` before running the algorithm, and never mutate the input.

### 2. Example
```text
Input:  [73, 74, 75, 71, 69, 72, 76, 73]
Output: [1, 1, 4, 2, 1, 1, 0, 0]
```

### 3. Understand the Problem
The result is a distance in days, so the algorithm must know both temperatures and their indexes. For each day, the target is the first future day with a strictly higher temperature, not the warmest future day.

#### In Simple Terms
For each day, count how many days pass before it gets warmer; write `0` if it never does.

### 4. Real-World Analogy
Each day's forecast waits in a queue of unresolved questions. When a warmer reading arrives, it answers every colder waiting day that it can reach, and the index difference gives each wait length.

### 5. Key Observation
A left-to-right stack can hold unresolved indexes whose temperatures are decreasing or equal from bottom to top. A current temperature resolves top indexes while it is strictly warmer; equality must remain unresolved because an equal day is not an answer.

### 6. Pattern
```text
Pattern: Monotonic Stack of Unresolved Indexes
Why this pattern applies: one future value can resolve several earlier positions
How to recognize it: answers are distances to the first later strict threshold crossing
Similar problems: next greater element, stock span, next warmer timestamp
```

### 7. Brute Force
Validate every temperature first. For each day, scan future days in chronological order. At the first strictly warmer temperature, append the index difference; append `0` if no future day qualifies.

#### Pseudocode
```text
validate every temperature is a safe integer; otherwise throw RangeError
for each day:
    wait = 0
    for each future day in order:
        if future temperature > current temperature:
            wait = future index - current index
            stop the inner scan
    append wait to results
return results
```

#### Brute-Force TypeScript
```typescript
function dailyTemperaturesBruteForce(temperatures: readonly number[]): number[] {
  for (const temperature of temperatures) {
    if (!Number.isSafeInteger(temperature)) {
      throw new RangeError("temperatures must contain only safe integers");
    }
  }

  return temperatures.map((temperature, index) => {
    for (let futureIndex = index + 1; futureIndex < temperatures.length; futureIndex += 1) {
      if (temperatures[futureIndex] > temperature) return futureIndex - index;
    }
    return 0;
  });
}
```

#### Visualization
```text
Day 0, 73 -> day 1, 74: wait 1
Day 1, 74 -> day 2, 75: wait 1
Day 2, 75 -> day 6, 76: wait 4
Day 3, 71 -> day 5, 72: wait 2
Day 4, 69 -> day 5, 72: wait 1
Day 5, 72 -> day 6, 76: wait 1
Days 6 and 7: no warmer future day
```

#### Dry Run
For day `2` at `75`, the scan rejects `71`, `69`, and `72` before finding `76` at day `6`, so the wait is `6 - 2 = 4`. Day `6` at `76` finds no warmer future temperature and returns `0`.

#### Brute-Force Complexity
Validation is `O(n)`. The future-day scans take `O(n²)` worst-case time for a nonincreasing or all-equal input. Auxiliary space is `O(1)` beyond the returned `O(n)` array.

### 8. Why Brute Force Is Not Ideal
Long cool or equal stretches make many days rescan the same future readings. An all-equal input performs nearly every pairwise comparison even though no day can resolve another.

### 9. Optimal Approach
After validation, scan left to right and store unresolved indexes. While the current temperature is strictly greater than the temperature at the top index, pop that index and set its answer to `currentIndex - previousIndex`. Keep equal temperatures on the stack, then push the current index.

### 10. Visualization
```text
Temperatures: [73, 74, 75, 71, 69, 72, 76, 73]

Read 73: stack [0]
Read 74: resolve 0 with 1; stack [1]
Read 75: resolve 1 with 1; stack [2]
Read 71: stack [2, 3]
Read 69: stack [2, 3, 4]
Read 72: resolve 4 with 1, resolve 3 with 2; stack [2, 5]
Read 76: resolve 5 with 1, resolve 2 with 4; stack [6]
Read 73: stack [6, 7]; both remain 0
```

### 11. Step-by-Step Dry Run
| Day | Temperature | Resolved indexes and waits | Unresolved stack |
|---:|---:|---|---|
| 0 | 73 | none | `[0]` |
| 1 | 74 | `0 -> 1` | `[1]` |
| 2 | 75 | `1 -> 1` | `[2]` |
| 3 | 71 | none | `[2, 3]` |
| 4 | 69 | none | `[2, 3, 4]` |
| 5 | 72 | `4 -> 1`, `3 -> 2` | `[2, 5]` |
| 6 | 76 | `5 -> 1`, `2 -> 4` | `[6]` |
| 7 | 73 | none | `[6, 7]` |

### 12. TypeScript Solution
```typescript
function dailyTemperatures(temperatures: readonly number[]): number[] {
  for (const temperature of temperatures) {
    if (!Number.isSafeInteger(temperature)) {
      throw new RangeError("temperatures must contain only safe integers");
    }
  }

  const waits: number[] = new Array(temperatures.length).fill(0);
  const unresolvedIndexes: number[] = [];

  for (let index = 0; index < temperatures.length; index += 1) {
    while (
      unresolvedIndexes.length > 0 &&
      temperatures[index] > temperatures[unresolvedIndexes[unresolvedIndexes.length - 1]]
    ) {
      const previousIndex = unresolvedIndexes.pop();
      if (previousIndex !== undefined) waits[previousIndex] = index - previousIndex;
    }
    unresolvedIndexes.push(index);
  }

  return waits;
}
```

### 13. Code Walkthrough
The first pass rejects invalid temperatures before answer computation. The stack stores indexes because answers are distances. Its temperatures remain decreasing or equal: only a strictly warmer current day triggers pops. Each popped index receives its first possible warmer day because days are processed chronologically. The input stays unchanged; only local output and stack arrays mutate.

### 14. Complexity Analysis
Validation takes `O(n)`. Every index is pushed once and popped at most once, so all stack operations total `O(n)` despite the nested `while`. Total time is `O(n)`, auxiliary stack space is `O(n)`, and the returned waits array uses `O(n)` output space.

### 15. Edge Cases
- Empty input returns `[]`.
- One temperature returns `[0]`.
- A strictly increasing input returns `1` for every day except the last.
- A nonincreasing input returns all zeroes.
- Equal temperatures remain unresolved until a strictly warmer day or the end.
- Any safe integer temperature is accepted; no real-world temperature range is assumed.
- Invalid numeric values throw `RangeError` before the algorithm begins.

### 16. Common Mistakes
- Using `>=` for the current-day comparison and treating equal temperatures as warmer.
- Storing temperatures without indexes and losing the distance calculation.
- Replacing unresolved indexes rather than resolving all colder indexes on top.
- Returning a future temperature instead of the number of days.
- Misclassifying the amortized `O(n)` loop as `O(n²)`.

### 17. Interview Follow-ups
1. Return the next warmer temperature as well as the wait. 2. Support a minimum warming threshold. 3. Process forecast updates as a stream. 4. Return hours for timestamped readings.

### 18. Variations
- Next greater value. - Next day at least a fixed amount warmer. - Circular temperature history. - Previous warmer day.

### 19. Pattern Connections
This is next greater element with indexes retained because the output asks for distance. The strict `>` comparison keeps equal temperatures unresolved, while each index's one push and one possible pop provide the amortized linear proof.

### 20. NestJS Project Connection
Expose `POST /api/dsa/daily-temperatures` with body `{ temperatures: number[] }`. Validate the DTO shape, call only the optimized function, map `RangeError` to HTTP `400`, and return `{ waits, trace, truncated }`. Bound trace events such as `{ day, temperature, resolvedDays, unresolvedDepth }` rather than returning the full stack every time.

### 21. Mini Challenge
Modify the function so day `i` waits for a future temperature at least `threshold` degrees warmer. State which values of `threshold` preserve the same monotonic-stack strategy.

## Problem 127 — Stock Span Problem

### 1. Problem Statement
Given a readonly array of safe-integer stock prices, return an array where each position contains the number of consecutive days ending on that day whose prices are less than or equal to today's price, including today. Equal prices extend the span. Validate every price with `Number.isSafeInteger` before running the algorithm, and never mutate the input.

### 2. Example
```text
Input:  [100, 80, 60, 70, 60, 75, 85]
Output: [1, 1, 1, 2, 1, 4, 6]
```

### 3. Understand the Problem
For each day, walk backward only through the contiguous block ending today while prior prices are less than or equal to today's price. A larger price stops the span, and today's day always contributes one.

#### In Simple Terms
Count today plus the uninterrupted run of earlier days whose prices were no higher than today's price.

### 4. Real-World Analogy
Today's price looks backward through consecutive lower or equal hurdles. It absorbs each hurdle's covered run until a strictly higher wall blocks it.

### 5. Key Observation
If a prior price is less than or equal to today's price, today's span includes that day and every consecutive day already summarized by its span. Storing `{ price, span }` groups those days so a whole block can be absorbed in one pop.

### 6. Pattern
```text
Pattern: Monotonic Stack with Aggregated Spans
Why this pattern applies: dominated consecutive blocks can be merged into one summary
How to recognize it: each new item looks backward until a strict blocker
Similar problems: previous greater element, daily temperatures, online histogram processing
```

### 7. Brute Force
Validate all prices first. For each day, start its span at one and scan left while each previous price is less than or equal to today's price. Stop at the first strictly greater price.

#### Pseudocode
```text
validate every price is a safe integer; otherwise throw RangeError
for each day:
    span = 1
    previous = day - 1
    while previous exists and previous price <= today's price:
        span = span + 1
        previous = previous - 1
    append span to results
return results
```

#### Brute-Force TypeScript
```typescript
function stockSpanBruteForce(prices: readonly number[]): number[] {
  for (const price of prices) {
    if (!Number.isSafeInteger(price)) {
      throw new RangeError("prices must contain only safe integers");
    }
  }

  return prices.map((price, index) => {
    let span = 1;
    for (let previousIndex = index - 1; previousIndex >= 0; previousIndex -= 1) {
      if (prices[previousIndex] > price) break;
      span += 1;
    }
    return span;
  });
}
```

#### Visualization
```text
100 -> [100]                         span 1
 80 -> [80]                          span 1; 100 blocks
 60 -> [60]                          span 1; 80 blocks
 70 -> [60, 70]                      span 2; 80 blocks
 60 -> [60]                          span 1; 70 blocks
 75 -> [60, 70, 60, 75]              span 4; 80 blocks
 85 -> [80, 60, 70, 60, 75, 85]      span 6; 100 blocks
```

#### Dry Run
At price `75`, scan left across `60`, `70`, and `60`; price `80` is strictly greater and stops the scan, so the span is `4`. At price `85`, all six days back through `80` qualify before `100` blocks the span, producing `6`.

#### Brute-Force Complexity
Validation is `O(n)`. A nondecreasing input makes each day scan its entire prefix, so worst-case time is `O(n²)`. Auxiliary space is `O(1)` beyond the returned `O(n)` array.

### 8. Why Brute Force Is Not Ideal
Long rising or equal-price runs are recounted for every new day. The work already done to establish an earlier day's span is discarded instead of reused as a block summary.

### 9. Optimal Approach
After validation, process prices from left to right with a stack of `{ price, span }` entries. Start each span at one. While the top price is less than or equal to the current price, pop it and add its entire span; equality is included because equal prices extend the run. Push the aggregated entry and append its span. This same state transition supports online processing because each new price needs only the retained stack, not future prices or the original array.

### 10. Visualization
```text
price  popped blocks       span  stack after push (price:span)
100    none                1     [100:1]
80     none                1     [100:1, 80:1]
60     none                1     [100:1, 80:1, 60:1]
70     60:1                2     [100:1, 80:1, 70:2]
60     none                1     [100:1, 80:1, 70:2, 60:1]
75     60:1, 70:2          4     [100:1, 80:1, 75:4]
85     75:4, 80:1          6     [100:1, 85:6]
```

### 11. Step-by-Step Dry Run
| Day | Price | Popped aggregate spans | Today's span |
|---:|---:|---|---:|
| 0 | 100 | none | 1 |
| 1 | 80 | none | 1 |
| 2 | 60 | none | 1 |
| 3 | 70 | `1` from price 60 | 2 |
| 4 | 60 | none | 1 |
| 5 | 75 | `1` from 60, `2` from 70 | 4 |
| 6 | 85 | `4` from 75, `1` from 80 | 6 |

### 12. TypeScript Solution
```typescript
function stockSpan(prices: readonly number[]): number[] {
  for (const price of prices) {
    if (!Number.isSafeInteger(price)) {
      throw new RangeError("prices must contain only safe integers");
    }
  }

  const spans: number[] = [];
  const candidates: Array<{ price: number; span: number }> = [];

  for (const price of prices) {
    let span = 1;
    while (candidates.length > 0 && candidates[candidates.length - 1].price <= price) {
      const previous = candidates.pop();
      if (previous !== undefined) {
        const nextSpan = span + previous.span;
        if (!Number.isSafeInteger(nextSpan)) {
          throw new RangeError("computed span exceeds the safe integer range");
        }
        span = nextSpan;
      }
    }
    candidates.push({ price, span });
    spans.push(span);
  }

  return spans;
}
```

### 13. Code Walkthrough
The validation pass completes before stack processing. Each candidate entry summarizes one price plus a contiguous block it already absorbed. The `<=` pop condition makes equal prices extend the current span. The safe-integer guard protects every aggregate addition, although JavaScript's practical maximum array length is far below `Number.MAX_SAFE_INTEGER`. The readonly prices array is never changed.

### 14. Complexity Analysis
Validation is `O(n)`. Each price produces one push and each entry can be popped only once, so all inner-loop work is `O(n)` amortized and total time remains `O(n)`. The stack uses `O(n)` auxiliary space, and the returned spans use `O(n)` output space.

### 15. Edge Cases
- Empty input returns `[]`.
- One price has span `1`.
- Strictly decreasing prices all have span `1`.
- Strictly increasing prices have spans `1, 2, ..., n`.
- Equal prices extend the span because the comparison is `<=`.
- Negative prices are accepted because the contract requires safe integers, not positive prices.
- Invalid prices throw `RangeError` before processing; an unsafe aggregate also throws.

### 16. Common Mistakes
- Popping only strictly lower prices and failing to include equal prices.
- Counting all earlier qualifying prices rather than only the consecutive suffix.
- Storing only prices and then recounting the blocks they represent.
- Forgetting that today's day contributes one.
- Calling the nested `while` quadratic even though each entry is popped once.

### 17. Interview Follow-ups
1. Implement a `StockSpanner` class with `next(price): number` for online calls. 2. Return the previous strictly greater index. 3. Use strictly lower rather than lower-or-equal prices. 4. Persist and restore the online stack state.

### 18. Variations
- Previous greater element. - Online stock span. - Span with timestamps. - Maximum-width ramp. - Consecutive threshold windows.

### 19. Pattern Connections
The aggregated span is a compressed run length. Popping an entry skips exactly the consecutive days represented by that entry, while the surviving top is the nearest strictly greater blocker. This is the same one-push, one-pop amortization used by other monotonic stacks, enhanced with summary metadata.

### 20. NestJS Project Connection
Expose `POST /api/dsa/stock-span` with body `{ prices: number[] }` for batch calculation and a stateful service method for an authenticated online stream. Run only the optimized transition, map `RangeError` to HTTP `400`, isolate each stream's stack, and return bounded events such as `{ price, absorbedEntries, span, depth }`; do not expose internal candidate entries.

### 21. Mini Challenge
Implement a `StockSpanner` class whose `next(price)` validates before changing state and returns the new span in amortized `O(1)` time. Show that a rejected price leaves the online stack unchanged.

## Problem 128 — Largest Rectangle in Histogram

### 1. Problem Statement
Given a readonly array of nonnegative safe-integer bar heights, return the largest rectangular area that can be formed from one or more contiguous histogram bars. A rectangle's height is the minimum bar height in its range and its width is the number of bars. Return `0` for an empty array. Throw `RangeError` if any height is invalid or if any candidate `width * height` would exceed `Number.MAX_SAFE_INTEGER`. Never mutate the input.

### 2. Example
```text
Input:  [2, 1, 5, 6, 2, 3]
Output: 10
```

### 3. Understand the Problem
Every contiguous range defines a rectangle whose height is limited by its shortest bar. The task is to choose the range that maximizes minimum height times range width while preserving exact safe-integer arithmetic.

#### In Simple Terms
Choose neighboring bars, use the shortest chosen bar as the rectangle's height, and find the largest safe area.

### 4. Real-World Analogy
Imagine placing a rigid rectangular sign in front of consecutive fence posts. The shortest selected post limits the sign's height, while the number of selected posts determines its width.

### 5. Key Observation
When a shorter bar arrives, every taller bar on top of an increasing stack has found its first strictly shorter boundary on the right. After popping one, the new stack top is its first strictly shorter boundary on the left, so its maximal width is known immediately.

### 6. Pattern
```text
Pattern: Monotonic Increasing Stack with a Sentinel Iteration
Why this pattern applies: each popped height receives its maximal left and right boundaries
How to recognize it: contiguous-range area is controlled by the minimum value
Similar problems: maximal rectangle in a binary matrix, sum of subarray minimums, next smaller element
```

### 7. Brute Force
Validate every height first. For each left boundary, extend the right boundary one bar at a time, maintain the minimum height seen in that range, verify multiplication is safe, and update the maximum area.

#### Pseudocode
```text
validate every height is a nonnegative safe integer; otherwise throw RangeError
maximum area = 0
for each left boundary:
    minimum height = infinity
    for each right boundary from left through the end:
        minimum height = min(minimum height, height at right)
        width = right - left + 1
        if minimum height is nonzero and width > floor(MAX_SAFE_INTEGER / minimum height):
            throw RangeError
        maximum area = max(maximum area, width * minimum height)
return maximum area
```

#### Brute-Force TypeScript
```typescript
function largestRectangleAreaBruteForce(heights: readonly number[]): number {
  for (const height of heights) {
    if (!Number.isSafeInteger(height) || height < 0) {
      throw new RangeError("heights must contain only nonnegative safe integers");
    }
  }

  let maximumArea = 0;
  for (let left = 0; left < heights.length; left += 1) {
    let minimumHeight = Number.POSITIVE_INFINITY;
    for (let right = left; right < heights.length; right += 1) {
      minimumHeight = Math.min(minimumHeight, heights[right]);
      const width = right - left + 1;
      if (
        minimumHeight !== 0 &&
        width > Math.floor(Number.MAX_SAFE_INTEGER / minimumHeight)
      ) {
        throw new RangeError("rectangle area exceeds the safe integer range");
      }
      maximumArea = Math.max(maximumArea, width * minimumHeight);
    }
  }

  return maximumArea;
}
```

#### Visualization
```text
Heights: [2, 1, 5, 6, 2, 3]

Range [5]       -> width 1, minimum 5, area 5
Range [5, 6]    -> width 2, minimum 5, area 10
Range [5, 6, 2] -> width 3, minimum 2, area 6

The best rectangle spans heights 5 and 6.
```

#### Dry Run
For left boundary `2`, the running minimum is `5` at right boundary `2`, remains `5` at right boundary `3`, and drops to `2` at right boundary `4`. The corresponding areas are `5`, `10`, and `6`; the middle range produces the global maximum `10`.

#### Brute-Force Complexity
Validation is `O(n)`. There are `O(n²)` contiguous ranges and each extension updates the running minimum in `O(1)`, so total time is `O(n²)`. Auxiliary space is `O(1)`, and the numeric return value uses `O(1)` output space.

### 8. Why Brute Force Is Not Ideal
Overlapping ranges repeatedly rediscover the same limiting bars. A long nondecreasing histogram makes each left boundary scan most of the remaining array even though shorter-boundary information can be retained in a stack.

### 9. Optimal Approach
Validate all heights, then scan indexes `0` through `n` inclusive. At index `n`, use a sentinel `currentHeight` of `0` but do not push the sentinel index. While the stack top's height is strictly greater than the current height, pop it, set `leftBoundary` to the new top or `-1`, and use width `index - leftBoundary - 1`. Check safety before multiplying. Push only real indexes. The strict `>` condition leaves equal heights stacked; this is correct because final or later shorter-bar pops give the earliest equal bar the widest range.

### 10. Visualization
```text
index  height  action                         stack       maximum
0      2       push 0                         [0]         0
1      1       pop 0: width 1, area 2         [1]         2
2      5       push 2                         [1, 2]      2
3      6       push 3                         [1, 2, 3]   2
4      2       pop 3: area 6; pop 2: area 10  [1, 4]      10
5      3       push 5                         [1, 4, 5]   10
6      0       final pops; do not push 6      [1]         10
```

### 11. Step-by-Step Dry Run
| Index | Current height | Popped height and width | Maximum area |
|---:|---:|---|---:|
| 0 | 2 | none | 0 |
| 1 | 1 | `2 x 1` | 2 |
| 2 | 5 | none | 2 |
| 3 | 6 | none | 2 |
| 4 | 2 | `6 x 1`, `5 x 2` | 10 |
| 5 | 3 | none | 10 |
| 6 sentinel | 0 | `3 x 1`, `2 x 4`, `1 x 6` | 10 |

The zero-height sentinel iteration flushes every positive height. A real zero also flushes prior positive bars and contributes only area `0`.

### 12. TypeScript Solution
```typescript
function largestRectangleArea(heights: readonly number[]): number {
  for (const height of heights) {
    if (!Number.isSafeInteger(height) || height < 0) {
      throw new RangeError("heights must contain only nonnegative safe integers");
    }
  }

  let maximumArea = 0;
  const increasingIndexes: number[] = [];

  for (let index = 0; index <= heights.length; index += 1) {
    const currentHeight = index === heights.length ? 0 : heights[index];

    while (
      increasingIndexes.length > 0 &&
      heights[increasingIndexes[increasingIndexes.length - 1]] > currentHeight
    ) {
      const heightIndex = increasingIndexes.pop();
      if (heightIndex === undefined) break;

      const height = heights[heightIndex];
      const leftBoundary =
        increasingIndexes.length === 0
          ? -1
          : increasingIndexes[increasingIndexes.length - 1];
      const width = index - leftBoundary - 1;
      if (height !== 0 && width > Math.floor(Number.MAX_SAFE_INTEGER / height)) {
        throw new RangeError("rectangle area exceeds the safe integer range");
      }
      maximumArea = Math.max(maximumArea, width * height);
    }

    if (index < heights.length) increasingIndexes.push(index);
  }

  return maximumArea;
}
```

### 13. Code Walkthrough
The validation pass rejects negative, fractional, infinite, `NaN`, and unsafe heights before stack state exists. Real indexes enter the stack in nondecreasing-height order. A strictly shorter current height closes each taller bar's maximal range; after the pop, the new top supplies the left boundary. The division guard runs before every candidate multiplication, including final sentinel pops, so no unsafe product is evaluated. Equal heights stay because the condition is strict `>`: later equal indexes may produce narrower candidates, then the earliest equal index eventually spans the full valid width. The sentinel index is never pushed, and the readonly input is never changed.

### 14. Complexity Analysis
Validation takes `O(n)`. Every real index is pushed once and popped at most once, so the sentinel scan takes `O(n)` total time despite its inner `while`. The stack uses `O(n)` auxiliary space in the worst case, and the scalar result uses `O(1)` output space.

### 15. Edge Cases
- Empty input returns `0`.
- A single zero returns `0`.
- Zero-height bars split positive rectangles into independent regions.
- Strictly increasing heights are finalized by the sentinel iteration.
- Strictly decreasing heights are popped as each shorter bar arrives.
- Equal heights remain stacked; the earliest equal index eventually receives the widest range.
- Any invalid height throws `RangeError` before processing.
- A candidate whose area would be unsafe throws before multiplication.

### 16. Common Mistakes
- Multiplying first and checking `Number.isSafeInteger` afterward, when precision may already be lost.
- Using the popped index as the width instead of deriving both boundaries.
- Using `>=` without preserving the earliest equal-height boundary.
- Forgetting to flush bars that survive to the end.
- Pushing sentinel index `n` and later reading `heights[n]` as a real bar.
- Mutating the input by appending a zero sentinel.

### 17. Interview Follow-ups
1. Return the winning left and right boundaries as well as its area. 2. Resolve ties by the leftmost rectangle. 3. Compute the maximal rectangle in a binary matrix. 4. Process bars online when the final boundary is not yet known.

### 18. Variations
- Maximal rectangle in a binary matrix. - Sum of subarray minimums. - Largest square under a histogram. - Circular histogram. - Variable bar widths.

### 19. Pattern Connections
This is a next-smaller-boundary problem evaluated lazily. The current index supplies a popped bar's first strictly shorter right boundary, and the new stack top supplies its first strictly shorter left boundary. Keeping equal heights with strict `>` delays them safely: later equal bars close first, while the earliest equal bar obtains the widest rectangle at that height.

### 20. NestJS Project Connection
Expose `POST /api/dsa/histogram/largest-rectangle` with body `{ heights: number[] }`. Validate the DTO array shape, run only the optimized function, map `RangeError` to HTTP `400`, and return `{ area, trace, truncated }`. Bound trace events such as `{ index, currentHeight, poppedHeight, leftBoundary, width, area }`; record only safe computed areas, cap event count, and never expose or mutate the internal stack or request array.

### 21. Mini Challenge
Return `{ area, left, right, height }` for a maximum rectangle while retaining `O(n)` time. Define and implement a deterministic tie rule without performing unsafe multiplication.

## Problem 129 — Sliding Window Maximum

### 1. Problem Statement
Given a readonly array of safe integers and a positive integer window size `k`, return the maximum value in every contiguous window of length `k`. Require `1 <= k <= values.length`, so empty input is invalid for every `k`. Throw `RangeError` for an invalid `k` or any invalid value, and never mutate the input.

### 2. Example
```text
Input:  values = [1, 3, -1, -3, 5, 3, 6, 7], k = 3
Output: [3, 3, 5, 5, 6, 7]
```

### 3. Understand the Problem
The windows overlap heavily: after one result, the leftmost value expires and one new value enters. The goal is to retain exactly the values that could still become a maximum in the current or a future window.

#### In Simple Terms
Slide a fixed-size frame across the array and report the largest value visible in the frame at each position.

### 4. Real-World Analogy
A monitoring dashboard shows the highest reading from the latest `k` samples. Old readings expire from the left, and a newer reading removes older readings that are no larger because they can never lead again.

### 5. Key Observation
An earlier value less than or equal to the current value cannot be a future maximum while the current value remains in range: the current value is at least as large and expires later. A decreasing deque therefore needs to retain only useful indexes, with its front always holding the current maximum.

### 6. Pattern
```text
Pattern: Monotonic Deque of Indexes
Why this pattern applies: candidates expire by index and are dominated by newer values
How to recognize it: every fixed-size window needs an extremum in linear total time
Similar problems: sliding window minimum, constrained subarray maximum, shortest subarray with bounds
```

### 7. Brute Force
Validate `k` and every value before creating output. For each possible window start, scan its `k` values, retain the greatest value, and append it to the result.

#### Pseudocode
```text
validate k is an integer and 1 <= k <= values.length; otherwise throw RangeError
validate every value is a safe integer; otherwise throw RangeError
for each window start from 0 through values.length - k:
    maximum = value at window start
    for each remaining index in this window:
        maximum = max(maximum, value at index)
    append maximum to results
return results
```

#### Brute-Force TypeScript
```typescript
function slidingWindowMaximumBruteForce(
  values: readonly number[],
  k: number,
): number[] {
  if (!Number.isInteger(k) || k < 1 || k > values.length) {
    throw new RangeError("k must be an integer between 1 and values.length");
  }
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
  }

  const maximums: number[] = [];
  for (let left = 0; left + k <= values.length; left += 1) {
    let maximum = values[left];
    for (let index = left + 1; index < left + k; index += 1) {
      maximum = Math.max(maximum, values[index]);
    }
    maximums.push(maximum);
  }
  return maximums;
}
```

#### Visualization
```text
[1, 3, -1]             -> 3
   [3, -1, -3]         -> 3
       [-1, -3, 5]     -> 5
           [-3, 5, 3]  -> 5
               [5, 3, 6] -> 6
                  [3, 6, 7] -> 7
```

#### Dry Run
The first window scans `1`, `3`, and `-1` and returns `3`. Moving one place right scans `3`, `-1`, and `-3`, again returning `3`. The process repeats for all `n - k + 1` windows to produce `[3, 3, 5, 5, 6, 7]`.

#### Brute-Force Complexity
Validation is `O(n)`. There are `n - k + 1` windows and each scans `k` values, giving `O((n - k + 1)k)` time. Auxiliary space is `O(1)` beyond the returned `O(n - k + 1)` output array.

### 8. Why Brute Force Is Not Ideal
Adjacent windows rescan `k - 1` shared values. When `k` is proportional to `n`, this repeated comparison work is quadratic even though each index has a simple expiration point.

### 9. Optimal Approach
After complete validation, scan `right` from left to right with a decreasing deque of indexes stored in a `number[]` plus a `head` offset. First advance `head` while the front index is at most `right - k`, because it has expired. Then pop from the back while the stored value is less than or equal to the current value; discarding earlier equals keeps the newer equal value because it expires later. Append `right`. Once `right >= k - 1`, append the value at the front index. Never call `shift()`.

### 10. Visualization
```text
right  value  deque values after update  emitted maximum
0      1      [1]                        none
1      3      [3]                        none
2     -1      [3, -1]                    3
3     -3      [3, -1, -3]                3
4      5      [5]                        5
5      3      [5, 3]                     5
6      6      [6]                        6
7      7      [7]                        7
```

### 11. Step-by-Step Dry Run
| Right | Current value | Expired front | Removed from back | Front maximum |
|---:|---:|---|---|---:|
| 0 | 1 | none | none | not emitted |
| 1 | 3 | none | index 0 | not emitted |
| 2 | -1 | none | none | 3 |
| 3 | -3 | none | none | 3 |
| 4 | 5 | index 1 | indexes 3 and 2 | 5 |
| 5 | 3 | none | none | 5 |
| 6 | 6 | none | indexes 5 and 4 | 6 |
| 7 | 7 | none | index 6 | 7 |

At `right = 4`, index `1` expires because `1 <= 4 - 3`. The new value `5` then dominates `-3` and `-1`, leaving itself as the maximum.

### 12. TypeScript Solution
```typescript
function slidingWindowMaximum(values: readonly number[], k: number): number[] {
  if (!Number.isInteger(k) || k < 1 || k > values.length) {
    throw new RangeError("k must be an integer between 1 and values.length");
  }
  for (const value of values) {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("values must contain only safe integers");
    }
  }

  const maximums: number[] = [];
  const candidateIndexes: number[] = [];
  let head = 0;

  for (let right = 0; right < values.length; right += 1) {
    while (head < candidateIndexes.length && candidateIndexes[head] <= right - k) {
      head += 1;
    }
    while (
      head < candidateIndexes.length &&
      values[candidateIndexes[candidateIndexes.length - 1]] <= values[right]
    ) {
      candidateIndexes.pop();
    }

    candidateIndexes.push(right);
    if (right >= k - 1) maximums.push(values[candidateIndexes[head]]);
  }

  return maximums;
}
```

### 13. Code Walkthrough
The scalar `k` check happens first and rejects empty input for every possible `k`; the value pass then rejects any unsafe element before output or deque allocation. The deque contains only unexpired indexes and their values decrease strictly from front to back. Expiration uses `<= right - k`. Back removal uses `<=`, so a newer duplicate replaces an earlier duplicate and remains available longer. The `head` offset makes front removal `O(1)` without `shift()`; no compaction is needed during this single pass. Only local arrays change, and the readonly input remains untouched.

### 14. Complexity Analysis
Validation takes `O(n)`. Each index is appended once, removed from the back at most once, and passed by `head` at most once, so total algorithm time is `O(n)`. The index deque uses `O(n)` auxiliary space in the worst case because expired slots remain in its backing array, and the returned array uses `O(n - k + 1)` output space.

### 15. Edge Cases
- Empty input throws `RangeError` for every `k` because no valid window exists.
- `k = 1` returns a fresh array containing every input value.
- `k = values.length` returns one maximum.
- Duplicate maxima discard earlier equals so the newest equal index survives.
- Strictly decreasing values can retain up to `k` live candidates.
- Negative safe integers are valid.
- Fractional, infinite, `NaN`, or unsafe values throw `RangeError`.
- Noninteger, nonpositive, or oversized `k` throws `RangeError` before value validation.

### 16. Common Mistakes
- Storing values instead of indexes and losing expiration information.
- Calling `shift()` and making repeated front removal linear.
- Expiring indexes with `< right - k` instead of `<= right - k`.
- Removing only smaller values and retaining unnecessary earlier duplicates.
- Emitting output before the first complete window reaches index `k - 1`.
- Accepting empty input with a window size that cannot exist.

### 17. Interview Follow-ups
1. Return sliding-window minima. 2. Return both minimum and maximum. 3. Support timestamped windows rather than fixed counts. 4. Process an unbounded stream with bounded live deque storage.

### 18. Variations
- Sliding window minimum. - First negative value per window. - Longest subarray within a value limit. - Moving median. - Maximum of every time interval.

### 19. Pattern Connections
The deque combines two filters: indexes expire from the front by age, while dominated values leave from the back by usefulness. Removing earlier equals is a tie policy that keeps the candidate with the later expiration. The one-entry, one-back-removal, one-front-expiration lifecycle gives the linear bound.

### 20. NestJS Project Connection
Expose `POST /api/dsa/sliding-window-maximum` with body `{ values: number[], k: number }`. Validate DTO shapes, call only the optimized function, map `RangeError` to HTTP `400`, and return `{ maximums, trace, truncated }`. Cap optimized trace events such as `{ right, value, expiredIndexes, dominatedIndexes, maximum }`; report indexes or bounded counts without exposing the internal deque, and never mutate the request array.

### 21. Mini Challenge
Implement a streaming class with `next(value): number | null` for a fixed validated `k`. Validate each value before changing state, return `null` until the first full window, and bound retained live candidates to `O(k)`.

## Problem 130 — LRU Cache

### 1. Problem Statement
Design a generic `LRUCache<K, V>` with `constructor(capacity)`, `get(key): V | undefined`, `put(key, value): void`, a readonly `size`, and `has(key): boolean`. Capacity must be a nonnegative safe integer or construction throws `RangeError`. Reading an existing key promotes it to most recently used. Updating an existing key changes its value and promotes it. Inserting beyond capacity evicts the least recently used key; capacity `0` stores nothing. Key equality follows JavaScript `Map` SameValueZero semantics.

Stored `undefined` values are allowed, so `get` alone cannot distinguish an existing key whose value is `undefined` from a miss. Call `has(key)` when that distinction matters; an alternative API could disallow `undefined` or return a tagged result.

### 2. Example
```text
capacity = 2
put(1, "A")        order: [1]
put(2, "B")        order: [1, 2]
get(1) -> "A"      order: [2, 1]
put(3, "C")        evicts 2; order: [1, 3]
get(2) -> undefined
put(1, "Z")        order: [3, 1]
```

The order shown is least recently used to most recently used.

### 3. Understand the Problem
The cache must combine direct key lookup with a changing recency order. A successful `get` is a mutation of that order even though it does not change the stored value. Both insertion and update make a key most recent, and only a new insertion can increase size and trigger eviction.

#### In Simple Terms
Keep the freshest keys near one end, remove the stalest key from the other end, and still find any key immediately.

### 4. Real-World Analogy
A small desk holds only the files used most recently. Opening a file moves it to the top of the pile. When a new file arrives at a full desk, the file untouched for the longest time is archived.

### 5. Key Observation
A hash map gives average `O(1)` lookup but does not by itself provide a portable, explicit recency structure. A doubly linked list gives `O(1)` removal from any known node and `O(1)` insertion at either boundary. Mapping each key to its list node combines both capabilities, provided every mutation keeps the map and list synchronized.

### 6. Pattern
```text
Pattern: Hash Map + Doubly Linked List
Why this pattern applies: lookup and arbitrary-node promotion must both be constant time
How to recognize it: keyed items need bounded storage and order changes on access
Similar problems: LFU cache, session eviction, recent-items lists, memoization with bounds
```

### 7. Brute Force
Store records in an array ordered from least recent to most recent. Use a SameValueZero comparison while linearly finding a key. On a hit, splice out its record and append it. On a new insertion, append the record and remove index `0` if the capacity is exceeded. Return immediately for capacity `0`.

#### Pseudocode
```text
constructor capacity:
    if capacity is not a nonnegative safe integer: throw RangeError

get key:
    linearly find key using SameValueZero
    if absent: return undefined
    remove its record and append it as most recent
    return its value

put key, value:
    if capacity is zero: return
    linearly find key using SameValueZero
    if present: update its value, remove it, and append it
    otherwise append a new record
    if size exceeds capacity: remove the first record
```

#### Brute-Force TypeScript
```typescript
type LRURecord<K, V> = {
  key: K;
  value: V;
};

class LRUCacheBruteForce<K, V> {
  private readonly records: Array<LRURecord<K, V>> = [];

  constructor(private readonly capacity: number) {
    if (!Number.isSafeInteger(capacity) || capacity < 0) {
      throw new RangeError("capacity must be a nonnegative safe integer");
    }
  }

  get size(): number {
    return this.records.length;
  }

  has(key: K): boolean {
    return this.findIndex(key) !== -1;
  }

  get(key: K): V | undefined {
    const index = this.findIndex(key);
    if (index === -1) return undefined;

    const record = this.records[index];
    this.records.splice(index, 1);
    this.records.push(record);
    return record.value;
  }

  put(key: K, value: V): void {
    if (this.capacity === 0) return;

    const index = this.findIndex(key);
    if (index !== -1) {
      const record = this.records[index];
      record.value = value;
      this.records.splice(index, 1);
      this.records.push(record);
      return;
    }

    this.records.push({ key, value });
    if (this.records.length > this.capacity) {
      this.records.splice(0, 1);
    }
  }

  private findIndex(key: K): number {
    return this.records.findIndex((record) => this.sameValueZero(record.key, key));
  }

  private sameValueZero(left: K, right: K): boolean {
    return left === right || (left !== left && right !== right);
  }
}
```

#### Visualization
```text
least recent                              most recent
     |                                         |
     v                                         v
  [{ 1, A }, { 2, B }] -- get(1) --> [{ 2, B }, { 1, A }]
  put(3, C)             -- evict --> [{ 1, A }, { 3, C }]
```

#### Dry Run
With capacity `2`, appending keys `1` and `2` gives `[1, 2]`. `get(1)` finds index `0`, removes that record, appends it, and returns `"A"`, producing `[2, 1]`. `put(3, "C")` appends `3`, then removes the first record because size became `3`, producing `[1, 3]`. Updating key `1` to `"Z"` moves it to the end, producing `[3, 1]`.

#### Brute-Force Complexity
Let `n` be the current cache size, with `n <= capacity`. `get`, `put`, and `has` take `O(n)` time because finding a key is linear; array splicing can also shift `O(n)` records. `size` is `O(1)`. The cache owns `O(capacity)` space, each operation uses `O(1)` auxiliary space, and each operation returns `O(1)` output.

### 8. Why Brute Force Is Not Ideal
Every access may scan the whole cache, and promotion or eviction shifts array elements. A long request sequence against a large capacity can therefore spend linear time per operation even though both the key and the record to move have stable identities.

### 9. Optimal Approach
Map every key to a private doubly linked `CacheNode<K, V>`. Track nullable `leastRecent` and `mostRecent` boundaries rather than sentinel nodes, avoiding fake generic key/value fields. `detach` removes a known node in `O(1)`; `appendMostRecent` attaches one at the tail; promotion performs those operations unless the node is already most recent. On a new insertion, add the map entry and node, then evict and delete the least-recent node if size exceeds capacity. A zero-capacity `put` returns before allocating.

### 10. Visualization
```text
Map                           Recency list
1 -> node(1, A)               LRU                 MRU
2 -> node(2, B)                |                   |
                               v                   v
                            [2, B] <----------> [1, A]

get(1): already MRU
put(3, C): append 3, detach 2, delete Map key 2

Map                           Recency list
1 -> node(1, A)             [1, A] <----------> [3, C]
3 -> node(3, C)               LRU                 MRU
```

### 11. Step-by-Step Dry Run
| Operation | Return | LRU to MRU | Size | Effect |
|---|---|---|---:|---|
| `put(1, "A")` | - | `[1]` | 1 | append new node |
| `put(2, "B")` | - | `[1, 2]` | 2 | append new node |
| `get(1)` | `"A"` | `[2, 1]` | 2 | detach 1, append 1 |
| `put(3, "C")` | - | `[1, 3]` | 2 | append 3, evict 2 |
| `get(2)` | `undefined` | `[1, 3]` | 2 | miss; no mutation |
| `put(1, "Z")` | - | `[3, 1]` | 2 | update and promote 1 |

For `put(3, "C")`, the map and list briefly contain three entries. Eviction detaches the current `leastRecent` node for key `2`, then deletes key `2` from the map, restoring `size = 2` and preserving one map entry per live node.

### 12. TypeScript Solution
```typescript
class CacheNode<K, V> {
  previous: CacheNode<K, V> | null = null;
  next: CacheNode<K, V> | null = null;

  constructor(
    readonly key: K,
    public value: V,
  ) {}
}

class LRUCache<K, V> {
  private readonly nodes = new Map<K, CacheNode<K, V>>();
  private leastRecent: CacheNode<K, V> | null = null;
  private mostRecent: CacheNode<K, V> | null = null;

  constructor(private readonly capacity: number) {
    if (!Number.isSafeInteger(capacity) || capacity < 0) {
      throw new RangeError("capacity must be a nonnegative safe integer");
    }
  }

  get size(): number {
    return this.nodes.size;
  }

  has(key: K): boolean {
    return this.nodes.has(key);
  }

  get(key: K): V | undefined {
    const node = this.nodes.get(key);
    if (node === undefined) return undefined;

    this.promote(node);
    return node.value;
  }

  put(key: K, value: V): void {
    if (this.capacity === 0) return;

    const existing = this.nodes.get(key);
    if (existing !== undefined) {
      existing.value = value;
      this.promote(existing);
      return;
    }

    const node = new CacheNode(key, value);
    this.nodes.set(key, node);
    this.appendMostRecent(node);

    if (this.nodes.size > this.capacity) {
      const nodeToEvict = this.leastRecent;
      if (nodeToEvict !== null) {
        this.detach(nodeToEvict);
        this.nodes.delete(nodeToEvict.key);
      }
    }
  }

  private promote(node: CacheNode<K, V>): void {
    if (node === this.mostRecent) return;
    this.detach(node);
    this.appendMostRecent(node);
  }

  private detach(node: CacheNode<K, V>): void {
    if (node.previous === null) {
      this.leastRecent = node.next;
    } else {
      node.previous.next = node.next;
    }

    if (node.next === null) {
      this.mostRecent = node.previous;
    } else {
      node.next.previous = node.previous;
    }

    node.previous = null;
    node.next = null;
  }

  private appendMostRecent(node: CacheNode<K, V>): void {
    node.previous = this.mostRecent;
    node.next = null;

    if (this.mostRecent === null) {
      this.leastRecent = node;
    } else {
      this.mostRecent.next = node;
    }
    this.mostRecent = node;
  }
}
```

### 13. Code Walkthrough
The constructor rejects negative, fractional, infinite, `NaN`, and unsafe capacities before any state exists. `nodes` uses SameValueZero key equality, including one `NaN` key and equivalence between `-0` and `0`. A hit obtains the exact node and promotes it without searching the list. An update changes the node value before the same promotion. A new key creates one node, registers it in the map, and appends it as most recent; overflow removes the least-recent boundary from both structures. `detach` repairs both neighbors and both boundaries, while clearing the removed node's links. Nodes remain private, so callers cannot violate the invariant that map membership and list membership match.

Because `V` may include `undefined`, `get` returning `undefined` is intentionally ambiguous. `has` resolves that ambiguity without changing recency; callers can test `cache.has(key)` separately when a stored `undefined` must be distinguished from absence.

### 14. Complexity Analysis
With JavaScript `Map` average-case bounds, `get`, `put`, and `has` take average `O(1)` time; `size` is `O(1)`. Linked-list detach, append, promotion, and eviction are strict `O(1)` once a node is known. The cache owns `O(capacity)` map entries and nodes, while each operation uses `O(1)` auxiliary space and returns `O(1)` output. Hash-table behavior is not a strict worst-case `O(1)` guarantee.

### 15. Edge Cases
- Capacity `0` accepts operations but never allocates or stores an entry; `size` remains `0`.
- A negative, fractional, infinite, `NaN`, or unsafe capacity throws `RangeError` during construction.
- A miss returns `undefined` and does not change recency.
- A stored `undefined` also returns `undefined`; `has` distinguishes it from a miss.
- Updating an existing key does not change size and promotes that key.
- Reading an existing key promotes it even when its value is `undefined`.
- Repeatedly accessing the most-recent key leaves boundaries unchanged.
- `NaN` keys match `NaN`, and `-0` and `0` identify the same key under SameValueZero.
- Object keys compare by identity, as they do in `Map`.

### 16. Common Mistakes
- Using only a map and forgetting that every successful read must update recency.
- Using a singly linked list and scanning for a predecessor during promotion.
- Updating a value without promoting its key.
- Evicting on an existing-key update even though size did not grow.
- Removing a node from the list but not its map entry, or the reverse.
- Treating `get(key) === undefined` as proof of absence when `undefined` values are allowed.
- Using `Object.is` for baseline keys, which incorrectly distinguishes `-0` from `0`.
- Accepting unsafe capacities or validating after state mutation.

### 17. Interview Follow-ups
1. Add a `delete(key)` operation. 2. Add time-based expiration without breaking recency. 3. Make eviction observable through a callback. 4. Compare LRU with LFU. 5. Discuss concurrency or asynchronous access around compound `has` and `get` calls.

### 18. Variations
- JavaScript-specific LRU with `Map` insertion order and delete-then-set promotion. - Timestamped map entries with a linear eviction scan and a checked monotonic safe-integer clock. - LFU cache with frequency buckets. - TTL cache. - Segmented LRU. - Weighted capacity instead of entry count.

JavaScript `Map` insertion order can implement a shorter LRU by deleting and re-setting a key, then evicting `map.keys().next().value`. That variation is practical but depends on language-specific iteration semantics. The explicit linked-list invariant is the canonical design and transfers to runtimes whose hash maps do not encode recency.

### 19. Pattern Connections
The map is an index from stable keys to mutable positions, while the doubly linked list is an ordered ownership structure. This is the same composition used when one representation answers identity queries and another answers ordering queries. The central invariant is bidirectional: every map entry references exactly one live list node, and every live list node is reachable from exactly one map entry.

### 20. NestJS Project Connection
Expose `POST /api/dsa/lru-cache/execute` with body `{ capacity, operations }`, where operations are tagged `get`, `put`, or `has` requests. Validate the full DTO before constructing or mutating the optimized cache, map `RangeError` to HTTP `400`, and run only `LRUCache`. Return `{ outputs, trace, truncated }` with at most 200 post-operation events such as `{ operation, key, hit, returnedValue, size, leastRecentKey, mostRecentKey, evictedKey }`. Instrument the class through an internal callback or a service wrapper rather than exposing nodes or the map; preserve JSON-safe key/value constraints at the API boundary, represent returned `undefined` explicitly in response DTOs, and never run the brute-force implementation in the endpoint.

Trace for the example:

| Operation | Output | LRU key | MRU key | Evicted | Size |
|---|---|---|---|---|---:|
| `put(1, "A")` | - | `1` | `1` | none | 1 |
| `put(2, "B")` | - | `1` | `2` | none | 2 |
| `get(1)` | `"A"` | `2` | `1` | none | 2 |
| `put(3, "C")` | - | `1` | `3` | `2` | 2 |
| `get(2)` | miss | `1` | `3` | none | 2 |
| `put(1, "Z")` | - | `3` | `1` | none | 2 |

### 21. Mini Challenge
Add `delete(key): boolean` in average `O(1)` time. Return whether a key existed, preserve the remaining recency order, and handle deleting the only node, either boundary, `NaN`, and the `-0`/`0` key equivalence.

# Topic Summary

## What You Learned
- Stacks restrict removal to the most recent item, while queues remove the oldest item.
- Array ends support amortized `O(1)` stack operations; linked heads support strict `O(1)` stack operations.
- A head index avoids the repeated `O(n)` shifting of an array queue, with occasional compaction preserving amortized bounds and limiting stale references.
- Head and tail references make linked queue boundaries constant time when the empty and last-item transitions stay synchronized.
- Two stacks can simulate a queue through lazy reversal, and two queues can simulate a stack through deliberate rotation.
- Underflow contracts must account for whether `undefined` is a legal stored value.
- Monotonic stacks resolve next-element, temperature, span, and histogram questions by discarding candidates that can no longer matter.
- Monotonic deques combine front expiration with back dominance for fixed-window extrema.
- Amortized proofs count how often each element can enter, transfer, and leave a structure.
- Strict versus non-strict comparisons determine duplicate behavior in monotonic structures.
- Indexes preserve position, age, width, and expiration information that values alone cannot encode.
- An LRU cache composes a hash map for key lookup with a doubly linked list for explicit recency.
- Encapsulation keeps arrays, linked nodes, maps, and boundary pointers from escaping and invalidating invariants.
- Persistent owned storage, temporary auxiliary space, returned output, and bounded trace space are separate complexity categories.

## Patterns You Should Recognize
- **End-backed stack:** use an array's end when push, pop, and peek should be amortized `O(1)`.
- **Head-backed linked stack:** replace the head to obtain strict `O(1)` stack operations.
- **Head-index queue:** advance an offset and compact occasionally instead of shifting every dequeue.
- **Head/tail linked queue:** retain both boundaries so enqueue and dequeue remain `O(1)`.
- **Lazy transfer:** move items between two stacks only when the dequeue side is empty.
- **Queue rotation:** place a new stack item before all older items by draining one queue into another.
- **Matching stack:** push expected closing symbols and compare them as closers arrive.
- **Per-entry aggregate:** store the current minimum beside each stack value for reversible `O(1)` queries.
- **Monotonic stack of indexes:** discard dominated candidates while preserving position-based answers.
- **Monotonic deque of indexes:** expire old candidates at the front and remove dominated candidates at the back.
- **Map plus doubly linked list:** combine keyed lookup with constant-time promotion and boundary eviction.

## Complexity Cheat Sheet
| Problem | Optimized time | Owned / auxiliary space |
|---:|---|---|
| 116. Implement Stack using Array | `push` amortized `O(1)`; `pop`, `peek`, `size`, `isEmpty` `O(1)` | `O(n)` owned; `O(1)` auxiliary |
| 117. Implement Stack using Linked List | all operations `O(1)` | `O(n)` owned; `O(1)` auxiliary |
| 118. Implement Queue using Array | `enqueue` and `dequeue` amortized `O(1)`; accessors `O(1)` | `O(n)` owned; occasional `O(n)` compaction copy |
| 119. Implement Queue using Linked List | all operations `O(1)` | `O(n)` owned; `O(1)` auxiliary |
| 120. Implement Queue using Two Stacks | `enqueue` `O(1)`; `dequeue` and `front` amortized `O(1)` | `O(n)` owned; `O(1)` per operation beyond transfer storage |
| 121. Implement Stack using Two Queues | `push` `O(n)`; `pop`, `peek`, `size`, `isEmpty` amortized `O(1)` | `O(n)` owned and temporary rotation storage |
| 122. Valid Parentheses | `O(n)` | `O(n)` auxiliary |
| 123. Min Stack | all operations amortized `O(1)` | `O(n)` owned; `O(1)` auxiliary |
| 124. Next Greater Element | `O(n)` | `O(n)` auxiliary plus `O(n)` output |
| 125. Next Smaller Element | `O(n)` | `O(n)` auxiliary plus `O(n)` output |
| 126. Daily Temperatures | `O(n)` | `O(n)` auxiliary plus `O(n)` output |
| 127. Stock Span Problem | `O(n)` | `O(n)` auxiliary plus `O(n)` output |
| 128. Largest Rectangle in Histogram | `O(n)` | `O(n)` auxiliary; `O(1)` output |
| 129. Sliding Window Maximum | `O(n)` | `O(n)` auxiliary in this head-offset implementation plus `O(n - k + 1)` output |
| 130. LRU Cache | `get`, `put`, `has` average `O(1)`; `size` `O(1)` | `O(capacity)` owned; `O(1)` auxiliary per operation |

## Common Interview Traps
- Calling `shift()` or `unshift()` inside performance-sensitive array stack or queue operations.
- Forgetting to reset both linked boundaries when the last node is removed.
- Claiming strict worst-case `O(1)` for dynamic-array growth, hash-map access, or amortized transfer.
- Treating `undefined` as an unambiguous miss when the generic value type can include it.
- Validating after mutation and leaving partial state when an exception occurs.
- Storing values when expiration, distance, width, or answer placement requires indexes.
- Choosing the wrong strictness for duplicate values in a monotonic stack or deque.
- Forgetting to expire deque indexes before reading a window result.
- Counting repeated scans instead of proving that each candidate is pushed and popped only once.
- Mixing required output space, persistent data-structure storage, and temporary auxiliary space.
- Returning private arrays or node references and allowing callers to break invariants.
- Updating an LRU value without promoting it, or promoting a node in only one of the map and list representations.
- Using equality semantics inconsistent with the required contract, especially for `NaN`, `-0`, and object identity.
- Letting educational traces grow without a cap or exposing private structural state through them.

## Practice Checklist
- [ ] 116. Implement Stack using Array
- [ ] 117. Implement Stack using Linked List
- [ ] 118. Implement Queue using Array
- [ ] 119. Implement Queue using Linked List
- [ ] 120. Implement Queue using Two Stacks
- [ ] 121. Implement Stack using Two Queues
- [ ] 122. Valid Parentheses
- [ ] 123. Min Stack
- [ ] 124. Next Greater Element
- [ ] 125. Next Smaller Element
- [ ] 126. Daily Temperatures
- [ ] 127. Stock Span Problem
- [ ] 128. Largest Rectangle in Histogram
- [ ] 129. Sliding Window Maximum
- [ ] 130. LRU Cache
