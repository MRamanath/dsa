# Linked List

## Learning Objectives

By the end of this topic, you should understand:

- how nodes use references to form a sequence without contiguous indexes
- how `head`, `tail`, and `null` describe list boundaries
- how to traverse a singly linked list safely under an acyclic contract
- how insertion and deletion rewire links while preserving unaffected node identities
- why a node's identity is different from the value stored inside it
- when a function owns new nodes and when it mutates caller-owned links
- how acyclic and cyclic list contracts change termination guarantees
- how fast/slow pointers, dummy nodes, sorted merge, and length checks solve common list problems
- how to separate required output space from auxiliary working space
- why unlinking a node makes it eligible for garbage collection rather than destroying it immediately
- how a NestJS endpoint can return useful but bounded execution traces

## Prerequisites

- references, mutation, contracts, and Big-O notation from `00-dsa-fundamentals.md`
- array traversal and zero-based indexes from `01-arrays.md`
- two-pointer reasoning from `04-two-pointers.md`
- TypeScript generics, classes, loops, functions, and `null`

## Core Theory

### Node and reference model

A singly linked list is made of separate node objects. Each node stores a value and either a reference to the next node or `null`.

```text
class ListNode<T> {
  constructor(public value: T, public next: ListNode<T> | null = null) {}
}
```

This generic model lets the same structure hold numbers, strings, or domain objects. The list is not one large container like an array. Its order comes from following `next` references.

```text
node A           node B           node C
+-------+        +-------+        +-------+
|  10   |------->|  20   |------->|  30   |-------> null
+-------+        +-------+        +-------+
```

Every TypeScript fence in this chapter is independently readable and self-contained. Each fence defines `ListNode<T>` and every other type or class it uses, so no example relies on declarations from another fence.

### Head, tail, and null termination

`head` is the reference used to enter the list. For a non-empty list, `tail` is the final node. The tail's `next` is `null`, which marks termination.

```text
head                                      tail
 |                                         |
 v                                         v
[10 | next] -> [20 | next] -> [30 | null]
```

An empty list has `head = null` and no tail node. A function may track a tail reference for efficiency, but returning only the head is enough to reach every node.

### Traversal

Traversal starts with a cursor at `head`, processes the current node, and advances to `current.next` until the cursor becomes `null`.

```text
current = head
while current is not null:
    use current.value
    current = current.next
```

Singly linked lists support forward movement. Reaching the node at index `i` takes `O(i)` time because there is no direct indexing operation.

### Insertion and deletion rewiring

Insertion creates a fresh node and changes references around one boundary:

```text
before: [A] --------> [B]
after:  [A] -> [new] -> [B]
```

The safe middle-insertion order is:

```text
new.next = predecessor.next
predecessor.next = new
```

Deletion bypasses a target:

```text
before: [A] -> [target] -> [B]
after:  [A] ------------> [B]
```

Saving the target before bypassing it allows `removed.next = null` afterward. Clearing that one outgoing link must happen only after the predecessor points to the successor, or the remaining suffix can be lost.

### Node identity versus values

Two nodes can contain equal values and still be different objects:

```text
first.value  = 7
second.value = 7
first !== second
```

Contracts that preserve node identity require relinking existing objects, not rebuilding equivalent values into replacement nodes. Problems 98 and 99 preserve every unaffected node object.

### Mutation and ownership

Ownership answers who may change a node's `next` reference:

- A construction function owns the fresh nodes it creates and does not mutate its input array.
- A traversal or search function borrows the list read-only and must not mutate any node.
- An insertion or deletion function in this chapter explicitly mutates caller-owned links and returns the possibly changed head.

Callers must use the returned head because inserting or deleting at index `0` changes the entry reference.

### Acyclic versus cyclic contracts

An acyclic list eventually reaches `null`. A cyclic list points back to an earlier node and may never terminate under ordinary traversal.

```text
acyclic: [1] -> [2] -> [3] -> null

cyclic:  [1] -> [2] -> [3]
                 ^       |
                 +-------+
```

Problems 97–100 require an acyclic list as a caller precondition. Their loops and recursive calls do not detect cycles. Cycle detection is a separate operation and cost.

### Common structural tools

**Fast/slow pointers:** Move one reference one step and another two steps. If they meet, a cycle exists; if the fast pointer reaches `null`, the list is acyclic. The same speed difference can find a midpoint.

**Dummy node:** A temporary node before the real head gives head and middle operations a common predecessor. It often removes a special case, but the dummy is not part of the returned data.

**Sorted merge:** Compare the front nodes of two sorted lists, attach the smaller node, and advance only that list. Existing nodes can be reused in `O(n + m)` time.

**List length:** Counting length costs `O(n)`. A known length can validate an index, align two lists, or locate a position, but computing it in a separate pass is still linear work.

## Visual Mental Model

Think of each node as a train car that knows only which car follows it.

```text
head
 |
 v
+-------+    +-------+    +-------+
| red   |--->| blue  |--->| green |---> null
+-------+    +-------+    +-------+
```

The values are labels inside the cars. The arrows are the actual structure. Changing a value repaints a car; changing `next` reconnects the train.

### Moving a traversal cursor

```text
step 0: current
          |
          v
         [A] -> [B] -> [C] -> null

step 1:        current
                 |
                 v
         [A] -> [B] -> [C] -> null
```

Moving `current` does not move or copy nodes. It only changes which node the local reference points to.

### Rewiring without losing the suffix

```text
predecessor.next
       |
       v
[A] -> [B] -> [C]

save [B], connect a fresh node to [B], then connect [A] to fresh

[A] -> [X] -> [B] -> [C]
```

## Complexity Concepts

### Access and traversal

Reading the head is `O(1)`. Reaching an arbitrary index or finding a value is `O(n)` in the worst case because links must be followed one at a time.

### Rewiring versus locating

Changing one or two known links is `O(1)`. Locating the predecessor is usually `O(n)`, so insertion or deletion by index is `O(n)` overall unless the needed node reference is already available.

### Output space versus auxiliary space

Required result data and temporary working memory are reported separately:

- Building `n` nodes has `O(n)` output space even when only `head` and `tail` are temporary references.
- Returning `n` values has `O(n)` output space.
- An iterative traversal uses `O(1)` auxiliary space.
- Recursive traversal uses `O(n)` call-stack space in addition to its output.
- Storing node references in an array uses `O(n)` auxiliary space without copying the nodes themselves.

### Garbage collection

JavaScript uses garbage collection. Setting `removed.next = null` does not manually free the node. It only removes one reference. A node becomes eligible for collection when no reachable reference points to it. External code may still hold the removed node, so algorithms should describe unlinking rather than immediate destruction.

### Cycles and termination

For an acyclic list of `n` nodes, traversal performs at most `n` node visits. In a cyclic list, `current !== null` may remain true forever. Complexity claims in this chapter therefore depend on the explicit acyclic precondition.

### Bounded NestJS traces

An educational endpoint may record optimized steps such as `{ index, value, nextValue }`. The service should cap events, for example at 200 steps, and return `{ trace, truncated }`. A bounded trace prevents visualization data from turning an `O(1)`-auxiliary algorithm into unbounded server memory use. Validation, serialization, and trace generation should be reported separately from the core algorithm.

## Common Patterns

### Linear traversal

```text
current = head
index = 0
while current exists:
    inspect current
    current = current.next
    index += 1
```

### Predecessor rewiring

```text
walk to node at index - 1
validate that the target position exists
change predecessor.next
```

### Dummy-head technique

```text
dummy.next = head
perform one predecessor-based operation
return dummy.next
```

### Fast and slow pointers

```text
slow moves one link
fast moves two links
compare their positions or use where fast stops
```

### Merge sorted lists

```text
compare current nodes
attach the smaller node after a tail
advance that input
attach the remaining suffix
```

# Problems

## Problem 96 — Create a Singly Linked List

### 1. Problem Statement
Implement `fromArray<T>(values: readonly T[]): ListNode<T> | null`. Create a fresh node for every input value, preserve the input order, and return the head. Do not mutate the input array. Empty input returns `null`.

### 2. Example
```text
Input:  [10, 20, 30]
Output: head -> [10] -> [20] -> [30] -> null

Input:  []
Output: null
```

### 3. Understand the Problem
The output is a chain of new node objects. `head` identifies the first node, and the final node is the `tail`, whose `next` remains `null`.

#### In Simple Terms
Given an array, make one new linked-list node per value and connect them in the same order. Return the first node, or `null` when there are no values.

### 4. Real-World Analogy
Turn a written passenger list into a line of people where each person points to the next person. The first person is the head and the last person is the tail.

### 5. Key Observation
The newest node belongs after the current tail. Remembering that tail avoids walking from the head to rediscover the end after every insertion.

### 6. Pattern
```text
Pattern: Tail-Based List Construction
Why this pattern applies: every fresh node is appended at the end
How to recognize it: convert ordered input into a new forward-only chain
Similar problems: stream-to-list conversion, list cloning, sorted merge output
```

### 7. Brute Force
Create the head from the first value. For each later value, start at the head, traverse to the current tail, and attach a fresh node. This is correct but repeatedly scans links already visited.

#### Pseudocode
```text
if values is empty: return null
head = fresh node for values[0]
for each remaining value:
    current = head
    while current.next exists: advance current
    current.next = fresh node for value
return head
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function fromArrayBruteForce<T>(values: readonly T[]): ListNode<T> | null {
  if (values.length === 0) return null;

  const head = new ListNode(values[0]);
  for (let index = 1; index < values.length; index += 1) {
    let tail = head;
    while (tail.next !== null) tail = tail.next;
    tail.next = new ListNode(values[index]);
  }
  return head;
}
```

#### Visualization
```text
append 20: scan [10], then [10] -> [20]
append 30: scan [10] -> [20], then attach [30]
```

#### Dry Run
For `[10, 20, 30]`, create `[10]`, walk zero links to append `[20]`, then walk one link to append `[30]`. The returned head reaches all three fresh nodes.

#### Brute-Force Complexity
**Time: `O(n²)`** because appending all nodes performs `0 + 1 + ... + (n - 2)` link traversals. **Space: `O(n)` output** for fresh nodes and **`O(1)` auxiliary** references.

### 8. Why Brute Force Is Not Ideal
Each append forgets where the previous scan ended. The same prefix is traversed repeatedly even though the tail was known one moment earlier.

### 9. Optimal Approach
Keep both `head` and `tail`. The first fresh node initializes both references. For every later value, set `tail.next` to a fresh node and then move `tail` to that node. The input remains read-only.

### 10. Visualization
```text
value 10: head, tail
             |
             v
            [10] -> null

value 20: head       tail
             |          |
             v          v
            [10] ----> [20] -> null

value 30: head                tail
             |                   |
             v                   v
            [10] -> [20] ----> [30] -> null
```

### 11. Step-by-Step Dry Run
| Input index | Fresh node | Head value | Tail value | Chain |
|---:|---:|---:|---:|---|
| 0 | 10 | 10 | 10 | `10 -> null` |
| 1 | 20 | 10 | 20 | `10 -> 20 -> null` |
| 2 | 30 | 10 | 30 | `10 -> 20 -> 30 -> null` |

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function fromArray<T>(values: readonly T[]): ListNode<T> | null {
  let head: ListNode<T> | null = null;
  let tail: ListNode<T> | null = null;

  for (const value of values) {
    const node = new ListNode(value);
    if (head === null) {
      head = node;
      tail = node;
    } else {
      tail!.next = node;
      tail = node;
    }
  }
  return head;
}
```

### 13. Code Walkthrough
`head` stays on the first node while `tail` moves after each append. Every `new ListNode(value)` guarantees fresh identity. The non-empty branch can use `tail` because `head` and `tail` are initialized together.

### 14. Complexity Analysis
**Time: `O(n)`** because each of the `n` input values is read once and each node is linked once. **Space: `O(n)` output** for the required nodes and **`O(1)` auxiliary** space for `head`, `tail`, and `node` references.

### 15. Edge Cases
- Empty input returns `null`.
- One value makes the same node both head and tail.
- Duplicate values produce distinct nodes.
- Object values are stored by reference, while the list nodes themselves are always fresh.
- A readonly input is never reordered or changed.

### 16. Common Mistakes
- Returning the tail instead of the head.
- Forgetting to advance `tail` after linking a node.
- Reusing one node object for every value.
- Calling repeated traversal `O(n)` instead of accounting for all appends.

### 17. Interview Follow-ups
1. Return both head and tail. 2. Build from an iterable. 3. Create a doubly linked list. 4. Clone an existing acyclic list.

### 18. Variations
- Prepend each input while reading backward. - Build a circular list. - Maintain a list object with length and tail. - Construct nodes lazily from a stream.

### 19. Pattern Connections
Tail tracking appears in queues and sorted-list merge. Fresh-node ownership also prepares for deep versus shallow cloning discussions.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/create` with `{ values }`. Return serialized `result`, plus a bounded optimized trace of `{ inputIndex, value, headValue, tailValue }` and `truncated`; never serialize raw cyclic object references.

### 21. Mini Challenge
Return `{ head, tail, length }` while keeping `O(n)` time and `O(1)` auxiliary space.

## Problem 97 — Traverse a Linked List

### 1. Problem Statement
Given the head of an acyclic singly linked list, return its values in order. Do not mutate any node. A `null` head returns `[]`; acyclicity is a caller precondition.

### 2. Example
```text
Input:  head -> [4] -> [8] -> [15] -> null
Output: [4, 8, 15]
```

### 3. Understand the Problem
The only entry is `head`, so each `next` reference must be followed in sequence. This problem does not detect cycles; with cyclic input, either implementation may fail to terminate normally.

#### In Simple Terms
Start at the first node, copy each value into an array, and move to the next node until `null`. Return the collected values without changing the list.

### 4. Real-World Analogy
Read stations on a one-way train route by starting at the first station and following each sign to the next one.

### 5. Key Observation
Every node is reachable by repeatedly following `next`. No index lookup or backward movement is needed.

### 6. Pattern
```text
Pattern: Linear Linked-List Traversal
Why this pattern applies: each node exposes the only reference to its successor
How to recognize it: process every value of an acyclic list in order
Similar problems: list search, list length, serialization
```

### 7. Brute Force
Use recursion. Each call processes one node and asks the next call to process the suffix. The call stack remembers where every unfinished call must return.

#### Pseudocode
```text
result = []
visit(node):
    if node is null: return
    append node.value
    visit(node.next)
visit(head)
return result
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function traverseRecursive<T>(head: ListNode<T> | null): T[] {
  const values: T[] = [];

  function visit(node: ListNode<T> | null): void {
    if (node === null) return;
    values.push(node.value);
    visit(node.next);
  }

  visit(head);
  return values;
}
```

#### Visualization
```text
visit(4)
  visit(8)
    visit(15)
      visit(null)
```

#### Dry Run
The calls append `4`, `8`, and `15`. The `null` base case returns, then the three waiting calls unwind without changing a link.

#### Brute-Force Complexity
**Time: `O(n)`** because each node is visited once. **Space: `O(n)` output** plus **`O(n)` call-stack auxiliary space**. Recursion does not detect a cycle.

### 8. Why Brute Force Is Not Ideal
The recursive shape matches the list, but a long list can exhaust the JavaScript call stack. Those stack frames are unnecessary when traversal needs only the next node.

### 9. Optimal Approach
Use one iterative cursor. While it is non-null, append its value and replace the cursor with `current.next`. The cursor is a local reference, so moving it does not mutate the list.

### 10. Visualization
```text
current
   v
  [4] -> [8] -> [15] -> null     values = []

         current
            v
  [4] -> [8] -> [15] -> null     values = [4]

                current
                   v
  [4] -> [8] -> [15] -> null     values = [4, 8]
```

### 11. Step-by-Step Dry Run
| Step | Current value | Values after append | Next current |
|---:|---:|---|---|
| 0 | 4 | `[4]` | node `8` |
| 1 | 8 | `[4, 8]` | node `15` |
| 2 | 15 | `[4, 8, 15]` | `null` |

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function traverse<T>(head: ListNode<T> | null): T[] {
  const values: T[] = [];
  let current = head;

  while (current !== null) {
    values.push(current.value);
    current = current.next;
  }
  return values;
}
```

### 13. Code Walkthrough
`current` initially aliases the head node. `values.push` copies the stored value into required output, and `current = current.next` advances only the local cursor. The loop ends at the tail's `null` link.

### 14. Complexity Analysis
**Time: `O(n)`** because the cursor crosses each of `n` nodes once. **Space: `O(n)` output** for returned values and **`O(1)` auxiliary** space for the cursor. The claim assumes acyclic input.

### 15. Edge Cases
- `null` returns `[]`.
- One node returns a one-element array.
- Duplicate values are all included.
- Stored objects are returned as the same value references.
- Cyclic input violates the caller precondition.

### 16. Common Mistakes
- Forgetting to advance `current` and creating an infinite loop.
- Changing `head.next` instead of moving a local cursor.
- Claiming the recursive version detects cycles.
- Omitting required output space from the space explanation.

### 17. Interview Follow-ups
1. Return the list length. 2. Traverse in reverse order. 3. Detect a cycle first. 4. Expose values as a generator.

### 18. Variations
- Recursive reverse traversal. - Mapping values into a new list. - Reducing values to one result. - Traversing a doubly linked list backward.

### 19. Pattern Connections
Search adds a stopping condition to the same scan. Fast/slow pointer algorithms run two traversals at different speeds, while cycle-safe serialization tracks visited identities.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/traverse` with an acyclic array representation that the service converts to nodes. Return `{ values, trace, truncated }`; each bounded optimized event can include `index`, `value`, and `nextValue`.

### 21. Mini Challenge
Write an iterative generator that yields each value without first allocating an output array.

## Problem 98 — Insert a Node

### 1. Problem Statement
Implement insertion into an acyclic singly linked list at zero-based `index`, where `0 <= index <= length`. Create one fresh node holding generic `value`, mutate links, preserve all existing node identities, and return the possibly new head. An invalid index throws `RangeError` without partial mutation.

### 2. Example
```text
Input:  10 -> 30 -> null, value = 20, index = 1
Output: 10 -> 20 -> 30 -> null

Input:  null, value = 5, index = 0
Output: 5 -> null
```

### 3. Understand the Problem
Index `0` inserts before the old head; index `length` appends after the old tail. Negative, non-integer, or beyond-end indexes are invalid, and validation must finish before any existing link changes.

#### In Simple Terms
Put one new node at the requested position. Keep every old node object, reconnect only the needed links, and throw without changing the list when the position is invalid.

### 4. Real-World Analogy
Add a new train car at a numbered position by uncoupling one connection and reconnecting both sides through the fresh car.

### 5. Key Observation
Insertion needs the predecessor at `index - 1`. Once that predecessor is known and the index is proven valid, two assignments insert the fresh node without rebuilding the list.

### 6. Pattern
```text
Pattern: Validate Then Rewire at a Predecessor
Why this pattern applies: insertion changes the link immediately before the position
How to recognize it: mutate a list by zero-based position while preserving nodes
Similar problems: deletion by index, splice, insert into sorted list
```

### 7. Brute Force
Store every existing node reference in an array. Validate the complete index before allocating or rewiring. Then create one node and handle head, end, or middle insertion using the stored identities.

#### Pseudocode
```text
if index is not a nonnegative integer: throw
store all node references in nodes
if index > nodes.length: throw
create fresh node
if index is 0: fresh.next = head; return fresh
predecessor = nodes[index - 1]
fresh.next = index equals length ? null : nodes[index]
predecessor.next = fresh
return head
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function insertAtBruteForce<T>(
  head: ListNode<T> | null,
  value: T,
  index: number,
): ListNode<T> | null {
  if (!Number.isInteger(index) || index < 0) {
    throw new RangeError("index must be a nonnegative integer");
  }

  const nodes: ListNode<T>[] = [];
  for (let current = head; current !== null; current = current.next) {
    nodes.push(current);
  }
  if (index > nodes.length) throw new RangeError("index is out of range");

  const node = new ListNode(value);
  if (index === 0) {
    node.next = head;
    return node;
  }

  const predecessor = nodes[index - 1];
  node.next = index === nodes.length ? null : nodes[index];
  predecessor.next = node;
  return head;
}
```

#### Visualization
```text
nodes = [ref(10), ref(30)]
index 1 is valid
new.next = ref(30)
ref(10).next = new
```

#### Dry Run
For `10 -> 30` at index `1`, collect both node references, validate `1 <= 2`, create `20`, point it to the original `30`, then point the original `10` to `20`.

#### Brute-Force Complexity
**Time: `O(n)`** for collecting references. **Space: `O(n)` auxiliary** for the reference array and **`O(1)` output** for the single fresh node.

### 8. Why Brute Force Is Not Ideal
Only the predecessor is needed, but the array retains every node reference. It also traverses the suffix even when insertion is near the head.

### 9. Optimal Approach
First validate the numeric shape of `index`. Index `0` is always valid, so create and return a new head. Otherwise traverse to the predecessor without mutating. If the predecessor cannot be reached, throw. A reached predecessor proves positions through `length` work, including appending after the tail. Only then create and link the new node.

### 10. Visualization
```text
insert 20 at index 1

predecessor
     |
     v
   [10] --------> [30] -> null
     |               ^
     +-> [20] -------+

result: [10] -> [20] -> [30] -> null
```

### 11. Step-by-Step Dry Run
| State | Position | Current/predecessor | Action |
|---|---:|---|---|
| validate | 1 | head `10` | index shape is valid |
| locate | 0 | node `10` | predecessor reached |
| prepare | 1 | node `10` | fresh `20.next = 30` |
| mutate | 1 | node `10` | `10.next = fresh 20` |

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function insertAt<T>(
  head: ListNode<T> | null,
  value: T,
  index: number,
): ListNode<T> | null {
  if (!Number.isInteger(index) || index < 0) {
    throw new RangeError("index must be a nonnegative integer");
  }
  if (index === 0) return new ListNode(value, head);

  let predecessor = head;
  for (let position = 0; position < index - 1; position += 1) {
    if (predecessor === null) throw new RangeError("index is out of range");
    predecessor = predecessor.next;
  }
  if (predecessor === null) throw new RangeError("index is out of range");

  predecessor.next = new ListNode(value, predecessor.next);
  return head;
}
```

### 13. Code Walkthrough
The loop only moves references; it performs no writes. If `index` exceeds `length`, `predecessor` becomes `null` and the function throws before creating or linking a node. At the valid end position, the predecessor is the tail and its old `next` is `null`.

### 14. Complexity Analysis
**Time: `O(n)`** in the worst case because reaching the tail predecessor may scan the list. **Space: `O(1)` auxiliary** plus **`O(1)` output** for one fresh node. Existing nodes keep their identities.

### 15. Edge Cases
- Insert into `null` only at index `0`.
- Index `0` returns a new head.
- Index `length` appends correctly.
- Negative, fractional, and beyond-length indexes throw before mutation.
- Duplicate values do not affect position or identity.

### 16. Common Mistakes
- Mutating a link before proving the full index is valid.
- Rejecting `index === length` and breaking append.
- Losing the suffix by overwriting `predecessor.next` before saving it.
- Rebuilding old nodes and violating identity preservation.

### 17. Interview Follow-ups
1. Insert after a given node reference. 2. Insert into a sorted list. 3. Use a dummy head. 4. Maintain head, tail, and length in a list class.

### 18. Variations
- Insert before the first matching value. - Insert into a circular list. - Insert into a doubly linked list. - Perform an immutable insertion by cloning.

### 19. Pattern Connections
Deletion uses the same predecessor search but bypasses instead of adding a node. Dummy heads generalize the index-zero case, and sorted merge repeatedly performs tail insertion.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/insert` with `{ values, value, index }`. Build an acyclic list, run the optimized insertion, and return serialized values plus a bounded trace of predecessor positions, validation outcome, and the final rewire; errors map to HTTP 400.

### 21. Mini Challenge
Return both the possibly new head and the inserted node reference without increasing asymptotic complexity.

## Problem 99 — Delete a Node

### 1. Problem Statement
Delete the node at zero-based `index` from an acyclic singly linked list, where `0 <= index < length`. Mutate links, preserve every other node identity, and return the possibly new head. Invalid input throws `RangeError` before mutation; any index on `null` is invalid.

### 2. Example
```text
Input:  10 -> 20 -> 30 -> null, index = 1
Output: 10 -> 30 -> null

Input:  10 -> null, index = 0
Output: null
```

### 3. Understand the Problem
Deleting index `0` changes the head. For another index, the predecessor must bypass the target. The removed node may be detached with `removed.next = null` after the remaining list has been reconnected.

#### In Simple Terms
Remove the node at the requested position without replacing the other nodes. If that position does not exist, throw and leave every link exactly as it was.

### 4. Real-World Analogy
Remove one train car by first coupling the previous car to the following car, then disconnecting the removed car's remaining coupling.

### 5. Key Observation
The predecessor controls whether the target stays in the chain. Save the target, bypass it, and only then clear the removed node's `next` reference.

### 6. Pattern
```text
Pattern: Validate Then Bypass a Target
Why this pattern applies: deletion changes the predecessor's outgoing link
How to recognize it: remove by position while preserving all other node objects
Similar problems: remove nth node, filter in place, list splice
```

### 7. Brute Force
Collect every node reference, validate the entire index, and then mutate. At index `0`, save the old head's successor before detaching it. Otherwise connect the predecessor to the target's successor, then detach the target.

#### Pseudocode
```text
if index is not a nonnegative integer: throw
store all node references in nodes
if index >= nodes.length: throw
removed = nodes[index]
if index is 0: newHead = removed.next
else: nodes[index - 1].next = removed.next; newHead = head
removed.next = null
return newHead
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function deleteAtBruteForce<T>(
  head: ListNode<T> | null,
  index: number,
): ListNode<T> | null {
  if (!Number.isInteger(index) || index < 0) {
    throw new RangeError("index must be a nonnegative integer");
  }

  const nodes: ListNode<T>[] = [];
  for (let current = head; current !== null; current = current.next) {
    nodes.push(current);
  }
  if (index >= nodes.length) throw new RangeError("index is out of range");

  const removed = nodes[index];
  const newHead = index === 0 ? removed.next : head;
  if (index > 0) nodes[index - 1].next = removed.next;
  removed.next = null;
  return newHead;
}
```

#### Visualization
```text
nodes = [ref(10), ref(20), ref(30)]
ref(10).next = ref(30)
ref(20).next = null
```

#### Dry Run
For index `1`, collect all three references and validate. Save original node `20`, connect original `10` to original `30`, then detach `20`. No unaffected node is replaced.

#### Brute-Force Complexity
**Time: `O(n)`** to collect references. **Space: `O(n)` auxiliary** for the array. The output reuses the remaining nodes and allocates no replacement list.

### 8. Why Brute Force Is Not Ideal
Deletion needs only the predecessor and target, yet the reference array retains every node and scans the entire list even when deleting near the head.

### 9. Optimal Approach
Validate the numeric index. For index `0`, first reject `null`; save `head.next`, detach the old head, and return the saved successor. Otherwise traverse without mutation to index `index - 1`. Confirm both the predecessor and `predecessor.next` exist, then bypass and detach the target.

### 10. Visualization
```text
delete index 1

predecessor   removed       successor
     |           |              |
     v           v              v
   [10] ------> [20] --------> [30] -> null
     |                           ^
     +---------------------------+

then: [20] -> null
```

### 11. Step-by-Step Dry Run
| State | Predecessor | Removed | Successor | Action |
|---|---:|---:|---:|---|
| locate | 10 | 20 | 30 | target exists; index valid |
| bypass | 10 | 20 | 30 | `10.next = 30` |
| detach | 10 | 20 | 30 | `20.next = null` |

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function deleteAt<T>(
  head: ListNode<T> | null,
  index: number,
): ListNode<T> | null {
  if (!Number.isInteger(index) || index < 0) {
    throw new RangeError("index must be a nonnegative integer");
  }
  if (index === 0) {
    if (head === null) throw new RangeError("index is out of range");
    const newHead = head.next;
    head.next = null;
    return newHead;
  }

  let predecessor = head;
  for (let position = 0; position < index - 1; position += 1) {
    if (predecessor === null) throw new RangeError("index is out of range");
    predecessor = predecessor.next;
  }
  if (predecessor === null || predecessor.next === null) {
    throw new RangeError("index is out of range");
  }

  const removed = predecessor.next;
  predecessor.next = removed.next;
  removed.next = null;
  return head;
}
```

### 13. Code Walkthrough
All traversal and existence checks happen before a write. `removed` preserves access to the target while `predecessor.next = removed.next` keeps the remaining suffix connected. Detaching afterward cannot disconnect the returned list.

### 14. Complexity Analysis
**Time: `O(n)`** in the worst case because deletion near the tail requires walking to its predecessor. **Space: `O(1)` auxiliary** for local references. No output nodes are allocated, and unaffected identities remain intact.

### 15. Edge Cases
- Any index on `null` throws.
- Deleting the only node returns `null`.
- Index `0` returns a new head and detaches the old one.
- Deleting the tail makes its predecessor the new tail.
- Negative, fractional, and `index === length` inputs throw before mutation.

### 16. Common Mistakes
- Clearing `removed.next` before saving the successor.
- Returning the old head after deleting index `0`.
- Treating `index === length` as valid deletion.
- Partially rewiring before discovering an invalid index.

### 17. Interview Follow-ups
1. Delete by node reference. 2. Delete the first matching value. 3. Remove every matching value with a dummy head. 4. Return the removed node too.

### 18. Variations
- Immutable deletion by cloning. - Delete from a doubly linked list. - Remove the nth node from the end. - Delete duplicates from a sorted list.

### 19. Pattern Connections
Insertion and deletion share predecessor traversal. Removing from the end motivates fast/slow pointers, while repeated conditional deletion motivates a dummy node.

### 20. NestJS Project Connection
Expose `DELETE /api/dsa/linked-list/node` with `{ values, index }`. Return serialized remaining values and a bounded optimized trace of predecessor visits, target validation, bypass, and detach; map `RangeError` to a stable HTTP 400 response.

### 21. Mini Challenge
Modify the function to return `{ head, removed }`, where `removed` is detached and all failure behavior stays atomic.

## Problem 100 — Search in a Linked List

### 1. Problem Statement
Given the head of an acyclic singly linked list and generic `target`, return the first zero-based index whose value satisfies `Object.is(value, target)`, or `-1` if absent. Do not mutate nodes. A `null` head returns `-1`.

### 2. Example
```text
Input:  5 -> NaN -> 5 -> null, target = NaN
Output: 1

Input:  null, target = 7
Output: -1
```

### 3. Understand the Problem
Search follows links from the head and stops at the first match. `Object.is` is the equality contract, so `NaN` matches `NaN`, while `0` and `-0` are different.

#### In Simple Terms
Walk through the list from the beginning. Return the position of the first value that matches the target under `Object.is`, or `-1` when no node matches.

### 4. Real-World Analogy
Look for the first requested stop on a one-way route. Count stops from zero and stop searching as soon as the label matches exactly.

### 5. Key Observation
Without an index or ordering guarantee, a missing target may require checking every node. Keeping the current index alongside the cursor avoids any extra pass.

### 6. Pattern
```text
Pattern: Linear Search over Linked Nodes
Why this pattern applies: values are reachable only by following next references
How to recognize it: find the first matching value in an unsorted list
Similar problems: array linear search, find first node, predicate search
```

### 7. Brute Force
Copy all node values into an array, then use `findIndex` with `Object.is`. Copying is unnecessary but cleanly separates traversal from searching.

#### Pseudocode
```text
values = []
for each node from head to null: append node.value
return first array index where Object.is(value, target)
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function searchBruteForce<T>(
  head: ListNode<T> | null,
  target: T,
): number {
  const values: T[] = [];
  for (let current = head; current !== null; current = current.next) {
    values.push(current.value);
  }
  return values.findIndex((value) => Object.is(value, target));
}
```

#### Visualization
```text
list:   [5] -> [NaN] -> [5] -> null
array:  [5, NaN, 5]
findIndex(Object.is(_, NaN)) -> 1
```

#### Dry Run
The traversal copies three values. `findIndex` rejects `5`, matches `NaN` with `Object.is`, and returns `1`.

#### Brute-Force Complexity
**Time: `O(n)`** overall, although it may perform one traversal to copy and another scan to search. **Space: `O(n)` auxiliary** for copied values.

### 8. Why Brute Force Is Not Ideal
The value can be compared while its node is already being visited. Copying every value delays early return and uses memory proportional to list length.

### 9. Optimal Approach
Maintain `current` and `index`. Compare `current.value` with `target` using `Object.is`; return immediately on the first match. Otherwise advance both state variables. Reaching `null` proves absence under the acyclic precondition.

### 10. Visualization
```text
target = NaN

index 0: [5] ----> [NaN] -> [5] -> null   no match
index 1:             ^                     Object.is(NaN, NaN)
                     return 1
```

### 11. Step-by-Step Dry Run
| Index | Current value | `Object.is(value, NaN)` | Decision |
|---:|---:|---|---|
| 0 | 5 | false | advance |
| 1 | `NaN` | true | return `1` |

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function search<T>(head: ListNode<T> | null, target: T): number {
  let current = head;
  let index = 0;

  while (current !== null) {
    if (Object.is(current.value, target)) return index;
    current = current.next;
    index += 1;
  }
  return -1;
}
```

### 13. Code Walkthrough
The cursor and index describe the same position. `Object.is` implements the exact equality contract, including `NaN` and signed zero behavior. No assignment targets a node field, so the list is unchanged.

### 14. Complexity Analysis
**Time: `O(n)`** in the worst case because an absent target or last-position match visits all nodes. **Space: `O(1)` auxiliary** for the cursor and index. No output collection is required.

### 15. Edge Cases
- `null` returns `-1`.
- A head match returns `0` immediately.
- Duplicate matches return the first index.
- `NaN` matches `NaN` because equality uses `Object.is`.
- Object targets match only the same object reference; cyclic input violates the precondition.

### 16. Common Mistakes
- Using `===` and failing the required `NaN` behavior.
- Returning a node when the contract requires an index.
- Continuing after a match and returning the last duplicate.
- Forgetting to increment the index or advance the cursor.

### 17. Interview Follow-ups
1. Return the matching node reference. 2. Accept a predicate instead of a target. 3. Return every matching index. 4. Search a sorted list with early stopping.

### 18. Variations
- Find the last match. - Search by object key. - Search a circular list safely. - Maintain a secondary index for repeated queries.

### 19. Pattern Connections
Traversal and search share the same loop. Predicate-based search generalizes equality, while cycle detection adds fast/slow pointers when null termination is not guaranteed.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/search` with `{ values, target }`. Return `{ index, trace, truncated }`, where bounded optimized events record `index`, `value`, and `matched`; preserve an explicit wire representation for special numeric values such as `NaN` if the API supports them.

### 21. Mini Challenge
Implement `findNode<T>(head, predicate)` that returns the first matching node reference or `null` without mutating the list.

## Problem 101 — Reverse a Linked List

### 1. Problem Statement
Given the head of an acyclic singly linked list, reverse its direction by mutating the existing `next` links in place and return the new head. Preserve every node identity and allocate no replacement nodes. A `null` head returns `null`; external references to nodes observe the changed links.

### 2. Example
```text
Input:  1 -> 2 -> 3 -> null
Output: 3 -> 2 -> 1 -> null

Input:  null
Output: null
```

### 3. Understand the Problem
The values stay in their original node objects; only link direction changes. The old tail becomes the new head, and the old head becomes the new tail with `next = null`.

#### In Simple Terms
Turn every arrow around without creating new nodes. Return the node that used to be last, or `null` for an empty list.

### 4. Real-World Analogy
Imagine a one-way chain of hikers. Each hiker turns to point to the person who was previously behind them, so the last hiker becomes the leader.

### 5. Key Observation
Overwriting `current.next` destroys the only link to the unprocessed suffix unless that link is saved first. The safe order is save next, reverse the link, then advance.

### 6. Pattern
```text
Pattern: In-Place Pointer Reversal
Why this pattern applies: every next edge must point to the previous node
How to recognize it: reverse a singly linked structure while preserving identities
Similar problems: reverse a sublist, reverse in groups, palindrome restoration
```

### 7. Brute Force
Collect all node references in an array. Then walk the array backward conceptually: make each later node point to the node before it, terminate the original head, and return the original tail. This mutates the same nodes but stores `O(n)` references.

#### Pseudocode
```text
nodes = every node reference from head to null
if nodes is empty: return null
for index from last down to 1:
  nodes[index].next = nodes[index - 1]
nodes[0].next = null
return nodes[last]
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function reverseBruteForce<T>(
  head: ListNode<T> | null,
): ListNode<T> | null {
  const nodes: ListNode<T>[] = [];
  for (let current = head; current !== null; current = current.next) {
    nodes.push(current);
  }
  if (nodes.length === 0) return null;

  for (let index = nodes.length - 1; index > 0; index -= 1) {
    nodes[index]!.next = nodes[index - 1]!;
  }
  nodes[0]!.next = null;
  return nodes[nodes.length - 1]!;
}
```

#### Visualization
```text
nodes = [ref(1), ref(2), ref(3)]
ref(3).next = ref(2)
ref(2).next = ref(1)
ref(1).next = null
```

#### Dry Run
For `1 -> 2 -> 3`, collect the three original references. Rewire original `3` to original `2`, original `2` to original `1`, terminate original `1`, and return original `3`.

#### Brute-Force Complexity
**Time: `O(n)`** to collect and rewire all nodes. **Space: `O(n)` auxiliary** for the reference array. No output nodes are allocated.

### 8. Why Brute Force Is Not Ideal
The array stores every node even though reversal only needs the previous, current, and saved-next references. It also separates traversal from rewiring without improving asymptotic time.

### 9. Optimal Approach
Maintain `previous` for the already reversed prefix and `current` for the first unprocessed node. Before changing `current.next`, save it as `nextSaved`. Point `current.next` to `previous`, then advance both references. When `current` reaches `null`, `previous` is the new head.

### 10. Visualization
```text
before one iteration:

previous    current      nextSaved
   |           |             |
   v           v             v
 [1] -> null  [2] --------> [3] -> null

after reversing current.next:

 [1] <------- [2]           [3] -> null
   ^           ^             ^
 previous    current      nextSaved

then advance: previous = 2, current = 3
```

### 11. Step-by-Step Dry Run
| Iteration | `previous` | `current` | `nextSaved` | Reversed link |
|---:|---|---|---|---|
| start | `null` | `1` | - | - |
| 1 | `null` | `1` | `2` | `1 -> null` |
| 2 | `1` | `2` | `3` | `2 -> 1` |
| 3 | `2` | `3` | `null` | `3 -> 2` |
| return | `3` | `null` | - | new head is `3` |

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function reverse<T>(head: ListNode<T> | null): ListNode<T> | null {
  let previous: ListNode<T> | null = null;
  let current = head;

  while (current !== null) {
    const nextSaved: ListNode<T> | null = current.next;
    current.next = previous;
    previous = current;
    current = nextSaved;
  }
  return previous;
}
```

### 13. Code Walkthrough
`nextSaved` preserves access to the untouched suffix before the write. After `current.next = previous`, the reversed prefix grows by one node. Advancing `previous` and `current` restores the loop state, and termination leaves `previous` at the original tail.

### 14. Complexity Analysis
**Time: `O(n)`** because each node is visited once. **Space: `O(1)` auxiliary** for three references. The operation allocates no nodes and preserves all node identities.

### 15. Edge Cases
- `null` returns `null`.
- A one-node list returns the same node with `next` still `null`.
- Two nodes swap head and tail roles.
- Duplicate values do not matter because links and identities are reversed.
- External node references remain valid but observe mutated `next` links.

### 16. Common Mistakes
- Overwriting `current.next` before saving the original successor.
- Returning the original head instead of `previous`.
- Forgetting to make the old head the terminated tail.
- Allocating replacement nodes and losing identity preservation.

### 17. Interview Follow-ups
1. Reverse recursively. 2. Reverse only positions `left` through `right`. 3. Reverse nodes in groups of `k`. 4. Restore the list after a palindrome check.

### 18. Variations
- Reverse a doubly linked list. - Reverse alternate groups. - Return both new head and new tail. - Produce an immutable reversed copy.

### 19. Pattern Connections
The same save-before-overwrite discipline appears in sublist reversal and pair swapping. Palindrome checks often reverse the second half, compare halves, and reverse again to restore external observations.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/reverse` with `{ values }`. Build one acyclic list, retain stable node IDs for demonstration, run the optimized three-reference loop, and return reversed values plus a bounded trace of `previousId`, `currentId`, `nextSavedId`, and each link mutation.

### 21. Mini Challenge
Return `{ head, tail }` after in-place reversal, where `tail` is the original head, without adding another traversal.

## Problem 102 — Find Middle of Linked List

### 1. Problem Statement
Given the head of an acyclic singly linked list, return the middle node reference without mutation. For even length, return the **second** middle: `1 -> 2 -> 3 -> 4` returns the node holding `3`. A `null` head returns `null`; node identity, not merely its value, is the result.

### 2. Example
```text
Input:  1 -> 2 -> 3 -> 4 -> 5 -> null
Output: reference to node 3

Input:  1 -> 2 -> 3 -> 4 -> null
Output: reference to node 3 (the second middle)
```

### 3. Understand the Problem
A list of length `n` returns the node at zero-based index `floor(n / 2)`. Returning a node reference matters when values repeat: the correct object is determined by position, not by value equality.

#### In Simple Terms
Find the halfway node without changing any arrows. When two nodes share the center, choose the one on the right.

### 4. Real-World Analogy
Two runners start together on a one-way route. One moves twice as fast; when the fast runner reaches the end, the slower runner is at the midpoint.

### 5. Key Observation
After the fast pointer advances two links for every one link of the slow pointer, the slow pointer has advanced `floor(n / 2)` times when the fast pointer can no longer take a full two-step move.

### 6. Pattern
```text
Pattern: Slow and Fast Pointers
Why this pattern applies: a 2:1 speed ratio locates the halfway position
How to recognize it: derive a relative position without knowing list length
Similar problems: cycle detection, palindrome check, nth node from end
```

### 7. Brute Force
Collect every node reference in an array. If the list is empty, return `null`; otherwise return the reference at `floor(nodes.length / 2)`, which naturally selects the second middle for even lengths.

#### Pseudocode
```text
nodes = every node reference from head to null
if nodes is empty: return null
return nodes[floor(nodes.length / 2)]
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function findMiddleBruteForce<T>(
  head: ListNode<T> | null,
): ListNode<T> | null {
  const nodes: ListNode<T>[] = [];
  for (let current = head; current !== null; current = current.next) {
    nodes.push(current);
  }
  return nodes[Math.floor(nodes.length / 2)] ?? null;
}
```

#### Visualization
```text
list:  [1] -> [2] -> [3] -> [4] -> null
nodes: [ref(1), ref(2), ref(3), ref(4)]
index: floor(4 / 2) = 2 -> ref(3)
```

#### Dry Run
For four nodes, the array length is `4`. `floor(4 / 2)` is `2`, so the function returns the original third node reference, the required second middle.

#### Brute-Force Complexity
**Time: `O(n)`** to collect the references. **Space: `O(n)` auxiliary** for the array. The list is not mutated.

### 8. Why Brute Force Is Not Ideal
The array retains every node solely to access one position. A relative-speed traversal can identify the same node in one pass with constant auxiliary space.

### 9. Optimal Approach
Initialize `slow` and `fast` at `head`. While `fast` and `fast.next` both exist, move `slow` one step and `fast` two steps. The loop condition controls the even-length rule: after an even number of nodes, `slow` lands on the right-hand middle.

### 10. Visualization
```text
even list: [1] -> [2] -> [3] -> [4] -> null

start:     S,F
            |
           [1] -> [2] -> [3] -> [4] -> null

step 1:           S      F
                   |      |
           [1] -> [2] -> [3] -> [4] -> null

step 2:                  S             F
                          |             |
           [1] -> [2] -> [3] -> [4] -> null

fast is null; return the node at S, original node 3
```

### 11. Step-by-Step Dry Run
| Loop state | `slow` | `fast` | Condition permits move? | Next state |
|---|---:|---:|---|---|
| start | 1 | 1 | yes | `slow = 2`, `fast = 3` |
| after 1 | 2 | 3 | yes | `slow = 3`, `fast = null` |
| after 2 | 3 | `null` | no | return node `3` |

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function findMiddle<T>(head: ListNode<T> | null): ListNode<T> | null {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;
  }
  return slow;
}
```

### 13. Code Walkthrough
Both pointers begin at the same node. Each successful loop advances `slow` once and `fast` twice. Because the loop continues whenever two fast-pointer links are available, even-length input performs exactly `n / 2` slow moves and returns the second middle. No node field is assigned.

### 14. Complexity Analysis
**Time: `O(n)`** because `fast` traverses the list once at double speed. **Space: `O(1)` auxiliary** for two references. The returned object is an existing node, and the list remains unchanged.

### 15. Edge Cases
- `null` returns `null`.
- One node returns that same node reference.
- Two nodes return the second node.
- Even lengths return the second middle.
- Duplicate values still return the positionally correct node identity.

### 16. Common Mistakes
- Using `while (fast.next && fast.next.next)` and returning the first middle.
- Returning `slow.value` instead of the node reference.
- Advancing `slow` twice or `fast` once.
- Mutating links even though the operation is read-only.

### 17. Interview Follow-ups
1. Return the first middle instead. 2. Return both middle nodes for even length. 3. Delete the middle node. 4. Find the node one-third of the way through.

### 18. Variations
- Find the midpoint of a sublist. - Split a list into balanced halves. - Find the middle of a circular list. - Return the middle index with the node.

### 19. Pattern Connections
Merge sort splits a linked list at its middle. Palindrome checks use the midpoint before reversing one half, and cycle detection uses the same two speeds for a different stopping condition.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/middle` with `{ values }`. Build the acyclic list with stable node IDs, run the optimized slow/fast loop, and return the middle node's ID and value plus a bounded trace of `slowId`, `fastId`, and each two-speed advance.

### 21. Mini Challenge
Return both the middle node reference and its zero-based index while preserving `O(n)` time, `O(1)` auxiliary space, and the second-middle rule.

## Problem 103 — Nth Node from End

### 1. Problem Statement
Given the head of an acyclic singly linked list and `n`, return the node reference that is `n`th from the end, where `1` means the tail. The function must not mutate the list. If `n` is not a positive integer or exceeds the list length, throw `RangeError`; therefore a `null` list throws for every positive `n`.

### 2. Example
```text
Input:  10 -> 20 -> 30 -> 40 -> null, n = 2
Output: reference to node 30

Input:  10 -> 20 -> 30 -> null, n = 3
Output: reference to node 10
```

### 3. Understand the Problem
For length `L`, the answer is at zero-based index `L - n`. Validate the scalar shape of `n` before traversing. The result is the original node object, not a copied value.

#### In Simple Terms
Count backward from the tail without changing the list: `1` picks the tail, and the full length picks the head. Invalid distances throw.

### 4. Real-World Analogy
To identify the runner `n` places behind the finish, send a lead runner `n` checkpoints ahead. Move both runners together; when the leader crosses the finish, the follower is at the answer.

### 5. Key Observation
If two pointers remain exactly `n` nodes apart, the trailing pointer reaches the desired node at the moment the leading pointer reaches `null`. Failure to create that gap proves `n` exceeds the length.

### 6. Pattern
```text
Pattern: Fixed-Gap Two Pointers
Why this pattern applies: the answer is a fixed distance behind the list end
How to recognize it: locate a node relative to the tail in one pass
Similar problems: remove nth from end, kth-to-last, fixed-window traversal
```

### 7. Brute Force
First reject a noninteger or nonpositive `n`, before touching the list. Then collect all node references, reject `n > nodes.length`, and return the reference at index `nodes.length - n`.

#### Pseudocode
```text
if n is not a positive integer: throw RangeError
nodes = every node reference from head to null
if n > nodes.length: throw RangeError
return nodes[nodes.length - n]
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function nthFromEndBruteForce<T>(
  head: ListNode<T> | null,
  n: number,
): ListNode<T> {
  if (!Number.isInteger(n) || n <= 0) {
    throw new RangeError("n must be a positive integer");
  }

  const nodes: ListNode<T>[] = [];
  for (let current = head; current !== null; current = current.next) {
    nodes.push(current);
  }
  if (n > nodes.length) throw new RangeError("n exceeds list length");
  return nodes[nodes.length - n]!;
}
```

#### Visualization
```text
nodes = [ref(10), ref(20), ref(30), ref(40)]
n = 2
index = 4 - 2 = 2
answer = ref(30)
```

#### Dry Run
For four nodes and `n = 2`, collect all four original references. Validation succeeds because `2 <= 4`; index `4 - 2` is `2`, so return the original node holding `30`.

#### Brute-Force Complexity
**Time: `O(n)`** to collect all references. **Space: `O(n)` auxiliary** for the array. Validation of the scalar occurs before traversal.

### 8. Why Brute Force Is Not Ideal
The array stores every node to calculate one index. Maintaining a fixed pointer gap discovers the same node in one pass with constant auxiliary space.

### 9. Optimal Approach
Validate that `n` is a positive integer. Advance `lead` exactly `n` steps, throwing if it becomes `null` before a required step. Then initialize `follow` at `head` and move both pointers one step until `lead` reaches `null`. At that point `follow` is nonnull and points to the answer.

### 10. Visualization
```text
list: [10] -> [20] -> [30] -> [40] -> null, n = 2

after advancing lead 2 steps:
 follow           lead
   |                |
   v                v
 [10] -> [20] -> [30] -> [40] -> null

move together until lead is null:
                 follow           lead
                    |                |
                    v                v
 [10] -> [20] -> [30] -> [40] -> null

return original node 30
```

### 11. Step-by-Step Dry Run
| Phase | Step | `lead` | `follow` | Decision |
|---|---:|---:|---:|---|
| create gap | 0 | 10 | - | advance lead |
| create gap | 1 | 20 | - | advance lead |
| gap ready | 2 | 30 | 10 | move together |
| together | 1 | 40 | 20 | move together |
| together | 2 | `null` | 30 | return node `30` |

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function nthFromEnd<T>(head: ListNode<T> | null, n: number): ListNode<T> {
  if (!Number.isInteger(n) || n <= 0) {
    throw new RangeError("n must be a positive integer");
  }

  let lead = head;
  for (let step = 0; step < n; step += 1) {
    if (lead === null) throw new RangeError("n exceeds list length");
    lead = lead.next;
  }

  let follow = head;
  while (lead !== null) {
    lead = lead.next;
    follow = follow!.next;
  }
  if (follow === null) throw new RangeError("n exceeds list length");
  return follow;
}
```

### 13. Code Walkthrough
The scalar check runs before traversal. The first loop either establishes an exact `n`-node gap or throws before a nonexistent step. The second loop preserves that gap; when `lead` is `null`, `follow` is the requested original node. For `n === length`, `lead` is already `null` and `follow` remains at the head.

### 14. Complexity Analysis
**Time: `O(n)`** because each pointer advances at most the list length. **Space: `O(1)` auxiliary** for two references and a counter. The list is not mutated and the returned node keeps its identity.

### 15. Edge Cases
- Noninteger, zero, and negative `n` throw before traversal.
- A `null` list throws for every positive `n`.
- `n = 1` returns the tail.
- `n === length` returns the head.
- `n > length` throws `RangeError`; duplicate values do not affect identity.

### 16. Common Mistakes
- Treating `n = 0` as the tail.
- Advancing the lead pointer `n - 1` steps and creating an off-by-one error.
- Returning a value instead of the node reference.
- Returning `null` rather than throwing when `n` exceeds the length.

### 17. Interview Follow-ups
1. Remove the nth node from the end. 2. Return the nth value without exposing nodes. 3. Support repeated queries with cached length. 4. Find a node `n` positions before a known reference.

### 18. Variations
- Return a default instead of throwing. - Use zero-based distance from the tail. - Find the nth node in a doubly linked list. - Return both the node and its predecessor.

### 19. Pattern Connections
Removing the nth node from the end adds a predecessor or dummy node to the same fixed-gap pattern. Sliding windows also maintain a fixed distance between boundaries while traversing once.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/nth-from-end` with `{ values, n }`. Validate `n` at the DTO boundary and again in the domain function, run the optimized fixed-gap traversal, and return the selected stable node ID and value plus a bounded trace of gap steps and paired `leadId`/`followId` moves; map `RangeError` to HTTP 400.

### 21. Mini Challenge
Return `{ node, predecessor }` for the nth node from the end in one pass and `O(1)` auxiliary space, using `null` as the predecessor when the answer is the head.

## Problem 104 — Detect Cycle

### 1. Problem Statement
Given the head of a singly linked list that may be cyclic, return `true` when following `next` references eventually revisits the same node object; otherwise return `false`. Do not mutate any node. Node identity, not stored value, determines whether a node repeats. Here, `n` means the number of distinct nodes reachable from `head`.

### 2. Example
```text
Input:  values [10, 20, 30, 40], with tail.next connected to index 1
Shape:  10 -> 20 -> 30 -> 40
              ^           |
              +-----------+
Output: true

Input:  values [10, 20, 30], with tail.next connected to no index (null)
Shape:  10 -> 20 -> 30 -> null
Output: false
```

### 3. Understand the Problem
Ordinary traversal is unsafe because a cyclic list never reaches `null`. The algorithm needs a second stopping condition that recognizes a repeated node identity. Equal values are irrelevant: two different nodes holding `7` are not a cycle, while revisiting one node object is.

#### In Simple Terms
Follow the arrows and answer whether they ever bring you back to a node object you have already visited. Do not change an arrow.

### 4. Real-World Analogy
Imagine walking through rooms whose doors each point to one next room. Repeated room names prove nothing, but returning to the exact same physical room proves the route loops.

### 5. Key Observation
In a cycle, a one-step pointer and a two-step pointer must eventually occupy the same node because the faster pointer gains one position per round within a finite loop. In an acyclic list, the faster pointer reaches `null` instead.

### 6. Pattern
```text
Pattern: Floyd's Fast/Slow Pointers
Why this pattern applies: different speeds expose repetition without storing visited nodes
How to recognize it: traversal may loop and only a boolean cycle answer is required
Similar problems: cycle entry, happy number, repeated state in deterministic transitions
```

### 7. Brute Force
Keep a `Set` of visited node references. Before recording each current node, test whether that exact object is already present. A repeat returns `true`; reaching `null` returns `false`. The set stores identities even when values are duplicated.

#### Pseudocode
```text
visited = empty identity set
current = head
while current is not null:
    if current identity is in visited: return true
    add current identity to visited
    current = current.next
return false
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function hasCycleBruteForce<T>(head: ListNode<T> | null): boolean {
  const visited = new Set<ListNode<T>>();
  let current = head;

  while (current !== null) {
    if (visited.has(current)) return true;
    visited.add(current);
    current = current.next;
  }

  return false;
}
```

#### Visualization
```text
10 -> 20 -> 30 -> 40
      ^           |
      +-----------+

visit 10: visited = {ref(10)}
visit 20: visited = {ref(10), ref(20)}
visit 30: visited = {ref(10), ref(20), ref(30)}
visit 40: visited = {ref(10), ref(20), ref(30), ref(40)}
visit ref(20) again: already present -> true
```

#### Dry Run
For tail-to-index `1`, traversal records the four distinct node references. The next reference after node `40` is the original node at index `1`, so `visited.has(current)` is true and the function returns `true` without another trip around the cycle.

#### Brute-Force Complexity
**Time: `O(n)`** because each distinct reachable node is processed once before a repeat or `null`. **Space: `O(n)` auxiliary** for the identity set. The list is not mutated.

### 8. Why Brute Force Is Not Ideal
The set gives direct evidence of repetition but stores every distinct reachable node. Floyd's pointers preserve termination and the same `O(n)` time while using only two references.

### 9. Optimal Approach
Initialize `slow` and `fast` at `head`. While `fast` and `fast.next` are nonnull, advance `slow` once and `fast` twice, then compare their node identities. Return `true` on equality. If the loop ends, the fast pointer found an acyclic boundary, so return `false`. Comparing after movement correctly detects a self-cycle.

### 10. Visualization
```text
tail.next -> index 1

index:   0       1       2       3
        [10] -> [20] -> [30] -> [40]
                 ^               |
                 +---------------+

after round 1: slow = index 1, fast = index 2
after round 2: slow = index 2, fast = index 1
after round 3: slow = index 3, fast = index 3 -> same identity
```

### 11. Step-by-Step Dry Run
| Round | `slow` after move | `fast` after move | Identity comparison |
|---:|---:|---:|---|
| 0 | index 0 | index 0 | initialize; do not decide yet |
| 1 | index 1 | index 2 | different nodes |
| 2 | index 2 | index 1 | different nodes |
| 3 | index 3 | index 3 | same node, return `true` |

For an acyclic list, a round eventually cannot begin because `fast === null` or `fast.next === null`; return `false` without dereferencing either boundary.

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function hasCycle<T>(head: ListNode<T> | null): boolean {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;
    if (slow === fast) return true;
  }

  return false;
}
```

### 13. Code Walkthrough
The loop guard makes the two-step advance safe. Movement happens before the identity comparison, so two pointers initially sharing `head` do not create a false positive. A self-cycle advances both pointers back to the same head node in the first round. No value comparison or link assignment occurs.

### 14. Complexity Analysis
**Time: `O(n)`** where `n` is the number of distinct nodes reachable from `head`: Floyd either reaches `null` or meets within a linear number of advances. **Space: `O(1)` auxiliary** for two references. The input is unchanged.

### 15. Edge Cases
- `head === null` returns `false`.
- One node whose `next` is `null` returns `false`.
- A self-cycle whose node points to itself returns `true`.
- A tail connected to index `0` or any later index returns `true`.
- Duplicate values in distinct nodes do not imply a cycle.

### 16. Common Mistakes
- Comparing `slow.value === fast.value` instead of node identities.
- Traversing only until `current === null`, which never terminates on a cycle.
- Comparing pointers before their first move and reporting every nonempty list as cyclic.
- Advancing `fast.next.next` without checking both `fast` and `fast.next`.

### 17. Interview Follow-ups
1. Return the cycle entry node. 2. Return the cycle length. 3. Return the noncyclic prefix length. 4. Prove why different pointer speeds must meet.

### 18. Variations
- Detect repetition in a deterministic state machine. - Detect a cycle in an array of next indexes. - Detect a cycle with Brent's algorithm. - Report whether two heads enter the same cycle.

### 19. Pattern Connections
Floyd's algorithm turns repeated-state detection into a two-pointer problem. The same reasoning appears in happy-number computation, function iteration, and cycle discovery in implicit graphs with one outgoing edge per state.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/detect-cycle` with `{ values, cycleToIndex }`, where `cycleToIndex` is `null` or a valid zero-based index that receives the tail connection. Run only the optimized algorithm for tracing. Return `{ hasCycle, trace, truncated }`, with each bounded event containing stable `slowIndex`, `fastIndex`, and `sameIdentity`; cap events independently of `null` termination so cyclic input cannot create an unbounded response.

### 21. Mini Challenge
Extend Floyd's solution to return `{ hasCycle, cycleLength }` in `O(n)` time and `O(1)` auxiliary space, where an acyclic list has length `0`.

## Problem 105 — Find Starting Point of Cycle

### 1. Problem Statement
Given the head of a singly linked list that may be cyclic, return the original node reference where the cycle begins, or `null` if the list is acyclic. Do not mutate the list. The cycle entry is the first node on the path from `head` that would later be revisited. Here, `n` is the number of distinct nodes reachable from `head`.

### 2. Example
```text
Input:  values [10, 20, 30, 40, 50], with tail.next connected to index 2
Shape:  10 -> 20 -> 30 -> 40 -> 50
                    ^           |
                    +-----------+
Output: reference to the original node at index 2, whose value is 30

Input:  values [10, 20, 30], with tail.next connected to no index (null)
Output: null
```

### 3. Understand the Problem
The answer is a node object, not its value or index. Duplicate values may exist before or inside the cycle, so only reference identity can identify the entry. Traversal must terminate for both cyclic and acyclic input, and no `next` link may change.

#### In Simple Terms
If the arrows loop, return the exact node where the path first enters that loop. If they do not loop, return `null`.

### 4. Real-World Analogy
A road leaves town and eventually joins a circular track. The answer is the physical junction where the road first enters the track, not another track location with the same sign.

### 5. Key Observation
After Floyd's pointers meet, place one pointer at `head` and leave one at the meeting node. Moving both one step at a time makes them meet at the cycle entry. The distance from `head` to entry matches the remaining wrapped distance from the meeting point to entry.

### 6. Pattern
```text
Pattern: Floyd's Cycle Entry in Two Phases
Why this pattern applies: phase 1 proves a cycle and phase 2 locates its entry
How to recognize it: return the first repeated node without retaining visited identities
Similar problems: linked-list cycle entry, prefix and period discovery, repeated function state
```

### 7. Brute Force
Store visited node identities while walking from `head`. The first current node already in the set is exactly the cycle entry: it is the earliest repeated node encountered along the deterministic path. Return `null` if traversal reaches `null`.

#### Pseudocode
```text
visited = empty identity set
current = head
while current is not null:
    if current identity is in visited: return current
    add current identity to visited
    current = current.next
return null
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function findCycleStartBruteForce<T>(
  head: ListNode<T> | null,
): ListNode<T> | null {
  const visited = new Set<ListNode<T>>();
  let current = head;

  while (current !== null) {
    if (visited.has(current)) return current;
    visited.add(current);
    current = current.next;
  }

  return null;
}
```

#### Visualization
```text
tail.next -> index 2

10 -> 20 -> 30 -> 40 -> 50
            ^           |
            +-----------+

visited in order: ref(10), ref(20), ref(30), ref(40), ref(50)
next current:      ref(30), already visited -> return this reference
```

#### Dry Run
The set receives the five distinct references in path order. Following node `50` returns to the original node at index `2`. That is the first repeated identity, so the baseline returns the node holding `30`, not merely the number `30`.

#### Brute-Force Complexity
**Time: `O(n)`** for at most one visit per distinct reachable node plus the repeat check. **Space: `O(n)` auxiliary** for node identities. No node is mutated.

### 8. Why Brute Force Is Not Ideal
The identity set makes the entry obvious but retains every reachable reference. Floyd's meeting geometry locates the same entry with constant auxiliary space.

### 9. Optimal Approach
Phase 1 runs slow and fast pointers until they share an identity or the fast pointer reaches an acyclic boundary. If no meeting occurs, return `null`. In phase 2, start one pointer at `head` and another at the meeting node; advance each by one link until their identities match, then return that node.

Let `mu` be the distance from `head` to entry, `lambda` the cycle length, and `x` the distance from entry to the phase-1 meeting. The slow pointer has traveled `mu + x`, while the fast pointer has gained a whole-number multiple of `lambda`, so `mu + x` is congruent to `0` modulo `lambda`. Therefore `mu` steps from the meeting point wrap through `lambda - x` to the entry, exactly as `mu` steps from `head` reach it.

### 10. Visualization
```text
tail.next -> index 2

head                entry
 |                    |
 v                    v
[10] -> [20] -> [30] -> [40] -> [50]
                 ^               |
                 +---------------+

phase 1: slow and fast meet inside the cycle
phase 2: fromHead starts at 10; fromMeeting starts at the meeting node
         both move one step per round and first share identity at 30
```

### 11. Step-by-Step Dry Run
For the five-node example, phase 1 meets at index `3`.

| Phase | Round | First pointer | Second pointer | Decision |
|---|---:|---:|---:|---|
| meeting | 0 | slow index 0 | fast index 0 | initialize |
| meeting | 1 | slow index 1 | fast index 2 | continue |
| meeting | 2 | slow index 2 | fast index 4 | continue |
| meeting | 3 | slow index 3 | fast index 3 | meeting found |
| entry | 0 | head index 0 | meeting index 3 | move both once |
| entry | 1 | head index 1 | cycle index 4 | move both once |
| entry | 2 | head index 2 | cycle index 2 | return original entry |

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function findCycleStart<T>(head: ListNode<T> | null): ListNode<T> | null {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;

    if (slow === fast) {
      let fromHead = head;
      let fromMeeting = slow;

      while (fromHead !== fromMeeting) {
        fromHead = fromHead!.next;
        fromMeeting = fromMeeting!.next;
      }

      return fromHead;
    }
  }

  return null;
}
```

### 13. Code Walkthrough
The guarded phase-1 loop either finds equal node references after movement or proves the path acyclic. A meeting implies `head`, `slow`, and every phase-2 successor needed before entry are nonnull, so the nonnull assertions express the proven invariant. Phase 2 compares identities before each move, which immediately handles a self-cycle or a cycle whose entry is `head`.

### 14. Complexity Analysis
**Time: `O(n)`** where `n` is the number of distinct reachable nodes: both Floyd phases perform linear total movement. **Space: `O(1)` auxiliary** for pointer references. The returned object belongs to the input list, and no links are changed.

### 15. Edge Cases
- `head === null` returns `null`.
- A one-node acyclic list returns `null`.
- A self-cycle returns `head`.
- A tail connected to index `0` returns `head`.
- Duplicate values do not change which node identity is the entry.

### 16. Common Mistakes
- Returning the phase-1 meeting node as the entry; they are not generally the same.
- Comparing node values instead of references.
- Moving one phase-2 pointer faster than the other.
- Restarting `fast` at `head` while it still moves two steps.
- Using ordinary null-terminated traversal after a cycle is known.

### 17. Interview Follow-ups
1. Return the entry index. 2. Return the cycle length. 3. Return both prefix length and cycle length. 4. Determine whether two cyclic lists share the same loop.

### 18. Variations
- Find the first repeated state of a function. - Locate a duplicate number through an implicit index graph. - Use Brent's algorithm before locating the entry. - Return stable metadata instead of exposing a node.

### 19. Pattern Connections
The prefix length `mu` and cycle length `lambda` describe an eventually periodic sequence. This connects linked-list cycle entry to modular arithmetic, repeated-state machines, and the duplicate-number problem modeled as index-to-index links.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/cycle-start` with `{ values, cycleToIndex }`. Construct stable node IDs, run only the optimized two-phase algorithm for tracing, and return `{ entry: { id, value } | null, trace, truncated }`. Bound phase-1 and phase-2 events, record pointer IDs and phase labels, and serialize the result rather than the cyclic node graph itself.

### 21. Mini Challenge
Return `{ entry, prefixLength, cycleLength }` without a set and without mutating the list, still in `O(n)` time and `O(1)` auxiliary space.

## Problem 106 — Remove Cycle

### 1. Problem Statement
Given the head of a singly linked list that may be cyclic, remove its cycle if one exists and return the original head reference. Mutate at most one `next` link. Preserve every node and the traversal order from `head` until the point where the cyclic path would repeat; the result must be acyclic. A `null` or already acyclic list is unchanged. Here, `n` is the number of distinct nodes reachable from `head`.

### 2. Example
```text
Input:  values [10, 20, 30, 40, 50], with tail.next connected to index 2
Before: 10 -> 20 -> 30 -> 40 -> 50
                    ^           |
                    +-----------+
After:  10 -> 20 -> 30 -> 40 -> 50 -> null
Output: the original reference to node 10

Input:  values [7], with tail.next connected to index 0 (self-cycle)
Before: 7 --+
        ^    |
        +----+
After:  7 -> null
```

### 3. Understand the Problem
The link to clear is the cycle predecessor's `next`: the link whose successor is the cycle entry. Clearing any earlier link would lose reachable nodes, while clearing any other cycle link would rotate or alter the required head-first order. External references to these node objects observe the mutation because the algorithm edits the original structure.

#### In Simple Terms
Find the one arrow that closes the loop and replace only that arrow with `null`. Keep the same head and all the same nodes in the same first-visit order.

### 4. Real-World Analogy
A one-way tour route accidentally sends its final stop back to an earlier stop. Close only that return gate so a traveler still visits every stop once in the original order and then exits.

### 5. Key Observation
Once the cycle entry is known, its unique predecessor inside the cycle is the node whose `next` is that exact entry object. Walking around the cycle from the entry must find that predecessor in at most one cycle length, including when the entry points to itself.

### 6. Pattern
```text
Pattern: Detect, Locate Entry, Then Rewire
Why this pattern applies: safe mutation requires the exact link that closes the cycle
How to recognize it: preserve first-visit order while converting a cyclic list to acyclic
Similar problems: break a circular list, repair corrupted next links, isolate a loop boundary
```

### 7. Brute Force
Walk with `previous` and `current` while storing visited identities. On the first repeated `current`, `previous` is the final distinct node in first-visit order and its `next` closes the cycle. Set only `previous!.next` to `null` and return the original head. If `current` reaches `null`, return the unchanged head.

#### Pseudocode
```text
visited = empty identity set
previous = null
current = head
while current is not null:
    if current identity is in visited:
        previous.next = null
        return head
    add current identity to visited
    previous = current
    current = current.next
return head
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function removeCycleBruteForce<T>(
  head: ListNode<T> | null,
): ListNode<T> | null {
  const visited = new Set<ListNode<T>>();
  let previous: ListNode<T> | null = null;
  let current = head;

  while (current !== null) {
    if (visited.has(current)) {
      previous!.next = null;
      return head;
    }

    visited.add(current);
    previous = current;
    current = current.next;
  }

  return head;
}
```

#### Visualization
```text
tail.next -> index 2

first visits: 10 -> 20 -> 30 -> 40 -> 50
                                      previous
                                          |
                                          v
repeat detected: current = original node 30
rewire:          node 50.next = null

result: 10 -> 20 -> 30 -> 40 -> 50 -> null
```

#### Dry Run
The set records nodes `10` through `50`. After `previous` becomes node `50`, `current` follows the closing link to the already visited node `30`. Clearing `previous.next` removes exactly that closing link. In a self-cycle, the first iteration makes the only node `previous`; the second detects the same node and clears its self-reference.

#### Brute-Force Complexity
**Time: `O(n)`** for the `n` distinct reachable nodes and one repeat check. **Space: `O(n)` auxiliary** for the identity set. A cyclic input changes exactly one link; acyclic and `null` inputs change none.

### 8. Why Brute Force Is Not Ideal
The set safely identifies the closing predecessor but retains every visited reference. Floyd's algorithm can locate the entry, after which one bounded trip around the cycle finds the predecessor with constant auxiliary space.

### 9. Optimal Approach
Run Floyd's slow/fast phase to find a meeting node. If the fast pointer reaches an acyclic boundary, return `head` unchanged. Otherwise locate the entry by moving one pointer from `head` and one from the meeting node one step each until they share identity. Starting at the entry, follow `next` until a node has `next === entry`; set that predecessor's `next` to `null` and return the original head.

### 10. Visualization
```text
tail.next -> index 2

before:
[10] -> [20] -> [30] -> [40] -> [50]
                 ^               |
                 +---------------+
                  entry       predecessor

phase 1: find a meeting inside the cycle
phase 2: locate entry ref(30)
phase 3: walk 30 -> 40 -> 50; 50.next === entry
rewire:  50.next = null

after:
[10] -> [20] -> [30] -> [40] -> [50] -> null
```

### 11. Step-by-Step Dry Run
| Phase | Step | Relevant references | Action |
|---|---:|---|---|
| meeting | 1 | slow `20`, fast `30` | continue |
| meeting | 2 | slow `30`, fast `50` | continue |
| meeting | 3 | slow `40`, fast `40` | save meeting |
| entry | 0 | fromHead `10`, fromMeeting `40` | move both |
| entry | 1 | fromHead `20`, fromMeeting `50` | move both |
| entry | 2 | both `30` | entry found |
| predecessor | 0 | node `30`, next `40` | continue around cycle |
| predecessor | 1 | node `40`, next `50` | continue around cycle |
| predecessor | 2 | node `50`, next is entry `30` | set `next = null` |

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function removeCycle<T>(head: ListNode<T> | null): ListNode<T> | null {
  let slow = head;
  let fast = head;
  let meeting: ListNode<T> | null = null;

  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;
    if (slow === fast) {
      meeting = slow;
      break;
    }
  }

  if (meeting === null) return head;

  let fromHead = head;
  let fromMeeting: ListNode<T> | null = meeting;
  while (fromHead !== fromMeeting) {
    fromHead = fromHead!.next;
    fromMeeting = fromMeeting!.next;
  }

  const entry = fromHead!;
  let predecessor = entry;
  while (predecessor.next !== entry) {
    predecessor = predecessor.next!;
  }

  predecessor.next = null;
  return head;
}
```

### 13. Code Walkthrough
The first loop records a meeting only after both pointers move; no meeting means no mutation. The second loop uses Floyd's distance invariant to obtain the original entry node. The predecessor loop compares references and is guaranteed to stay inside the known cycle. For a self-cycle, `predecessor` starts at `entry` and already satisfies `predecessor.next === entry`, so its link becomes `null` immediately.

### 14. Complexity Analysis
**Time: `O(n)`** where `n` is the number of distinct nodes reachable from `head`: meeting detection, entry location, and predecessor location are each linear in bounded portions of the structure. **Space: `O(1)` auxiliary**. At most one `next` link is mutated, and no node is created or removed.

### 15. Edge Cases
- `head === null` returns `null` without mutation.
- An acyclic list returns the same head and changes no link.
- A self-cycle uses the entry itself as predecessor and changes its `next` to `null`.
- When the cycle entry is `head`, the predecessor is the tail-like node that links back to `head`.
- A cycle beginning after a noncyclic prefix preserves the prefix and every cycle node in first-visit order.

### 16. Common Mistakes
- Clearing the meeting node's `next`; the meeting is not necessarily the cycle predecessor.
- Losing the original head by returning a traversal pointer.
- Comparing values when locating the entry or predecessor.
- Traversing toward `null` to find a tail before breaking the cycle.
- Rebuilding nodes, which violates identity preservation and hides mutation from external references.

### 17. Interview Follow-ups
1. Return whether a link was changed. 2. Return the former entry and predecessor. 3. Restore the removed link later. 4. Remove a cycle from a list with stable node IDs and an audit record.

### 18. Variations
- Copy first-visit values into a new acyclic list without mutating input. - Break a circular list at a requested node. - Reject cyclic input instead of repairing it. - Return the removed edge as two node references.

### 19. Pattern Connections
This problem composes detection, entry location, and a precise structural update. It connects Floyd's repeated-state reasoning with linked-list deletion: most of the cost is locating the correct boundary, while the actual rewiring is `O(1)`.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/remove-cycle` with `{ values, cycleToIndex }`. Build request-scoped nodes, run only the optimized algorithm for tracing, and return the acyclic values plus `{ removedLink: { fromId, toId } | null, trace, truncated }`. Bound events across meeting, entry, and predecessor phases; never serialize the cyclic graph before removal. In reusable domain code, document that callers holding external node references observe the one-link mutation.

### 21. Mini Challenge
Modify the optimized function to return `{ head, removedEdge }`, where `removedEdge` is the original `{ from, to }` node-reference pair or `null`, while preserving `O(n)` time and `O(1)` auxiliary space.

## Problem 107 — Merge Two Sorted Linked Lists

### 1. Problem Statement
Given the heads of two acyclic singly linked lists whose numeric safe-integer values are sorted in nondecreasing order, merge them and return the merged head. The two node sets must be disjoint: no node object may be reachable from both heads. Mutate and reuse every original node; do not copy values into replacement data nodes. When current values are equal, take the node from the first list before the node from the second list. A `null` input represents an empty list.

Sortedness, safe-integer values, acyclicity, and disjoint identity are caller preconditions. Validating them is a separate `O(m + n)` operation that generally needs `O(m + n)` identity storage; that cost is not part of the merge algorithm. Here, `m` and `n` are the input lengths and `N = m + n`.

### 2. Example
```text
First:  1(a) -> 3(b) -> 3(c) -> 7(d) -> null
Second: 1(e) -> 3(f) -> 5(g) -> 8(h) -> null

Output: 1(a) -> 1(e) -> 3(b) -> 3(c) -> 3(f) -> 5(g) -> 7(d) -> 8(h) -> null

The letters show identity. Every output node is an original node, and ties favor
the first list while both current heads have the same value.
```

### 3. Understand the Problem
This is a structural merge, not an array or value-copying exercise. The output must contain exactly the original node objects, so rewiring is visible through any external references to those nodes. The stable tie rule is equivalent to choosing `first` whenever `first.value <= second.value`. If either head is `null`, the other original head is already the answer.

#### In Simple Terms
Look only at the two front nodes, move the smaller one to the result, and use the first list's node when the values tie. Reconnect the original nodes rather than making replacement nodes.

### 4. Real-World Analogy
Two checkout lines are already ordered by ticket number. A clerk repeatedly admits the smaller front ticket; if both front tickets match, the first line goes first. People move into one line, but nobody is duplicated or replaced.

### 5. Key Observation
Because each remaining chain is sorted, the smallest remaining value must be at one of its two heads. After choosing that head, the same statement remains true for the shortened lists. No search behind either head is needed.

### 6. Pattern
```text
Pattern: Stable Two-Way Merge with Pointer Rewiring
Why this pattern applies: both inputs are sorted, so only their current heads compete
How to recognize it: combine sorted linked structures while preserving node identity
Similar problems: merge-sort merge step, merge k sorted lists, merge sorted event streams
```

### 7. Brute Force
Collect references to all nodes with their source list and within-list position. Sort those records by value, then by source so first-list nodes win cross-list ties, then by original position to preserve order inside each list. Finally, rewire the sorted references and terminate the last node with `null`. This baseline uses every original node and satisfies the same mutation and stability contract as the optimized solution.

JavaScript's `Array.prototype.sort` is specified as stable, but the comparator still includes source and position so the required order is explicit rather than dependent on an implicit tie.

#### Pseudocode
```text
records = every first-list node tagged (source 0, increasing order)
append every second-list node tagged (source 1, increasing order)
sort records by value, then source, then order
for each adjacent pair: records[i].node.next = records[i + 1].node
if records is nonempty: last node.next = null
return first sorted node or null
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function mergeSortedListsBruteForce(
  first: ListNode<number> | null,
  second: ListNode<number> | null,
): ListNode<number> | null {
  const records: Array<{
    node: ListNode<number>;
    source: 0 | 1;
    order: number;
  }> = [];

  let current = first;
  let order = 0;
  while (current !== null) {
    records.push({ node: current, source: 0, order });
    current = current.next;
    order++;
  }

  current = second;
  order = 0;
  while (current !== null) {
    records.push({ node: current, source: 1, order });
    current = current.next;
    order++;
  }

  records.sort((left, right) => {
    if (left.node.value < right.node.value) return -1;
    if (left.node.value > right.node.value) return 1;
    if (left.source !== right.source) return left.source - right.source;
    return left.order - right.order;
  });

  for (let index = 0; index + 1 < records.length; index++) {
    records[index].node.next = records[index + 1].node;
  }

  if (records.length === 0) return null;
  records[records.length - 1].node.next = null;
  return records[0].node;
}
```

#### Visualization
```text
collected records:
(1,a,L1,0) (3,b,L1,1) (3,c,L1,2) (7,d,L1,3)
(1,e,L2,0) (3,f,L2,1) (5,g,L2,2) (8,h,L2,3)

sorted references:
a -> e -> b -> c -> f -> g -> d -> h

rewired values:
1 -> 1 -> 3 -> 3 -> 3 -> 5 -> 7 -> 8 -> null
```

#### Dry Run
The two value-`1` records compare equal by value, so source `0` places node `a` before node `e`. For value `3`, source `0` places nodes `b` and `c` before source-`1` node `f`, while their position metadata keeps `b` before `c`. Rewiring those sorted references produces the required stable chain and explicitly clears the final node's old link.

#### Brute-Force Complexity
**Time: `O(N log N)`** for sorting `N = m + n` records; collection and rewiring are `O(N)`. **Space: `O(N)` auxiliary** for the records and sort storage. No data node is allocated, and every original node is reused.

### 8. Why Brute Force Is Not Ideal
Sorting ignores the strongest input guarantee: both lists are already sorted. Comparing only the two current heads determines the next node in constant time, reducing the merge to linear time and constant auxiliary space.

### 9. Optimal Approach
Create one dummy sentinel and keep a `tail` pointer at the end of the merged prefix. While both inputs remain, detach the smaller current node and append it; use `<=` so equal values select from the first list. When one list ends, attach the entire remaining sorted chain from the other list. Return `dummy.next`. The dummy is only a sentinel, not an output data node, and the algorithm allocates no replacement node for any input value.

### 10. Visualization
```text
first:  1(a) -> 3(b) -> 3(c) -> 7(d)
second: 1(e) -> 3(f) -> 5(g) -> 8(h)

dummy -> 1(a) -> 1(e) -> 3(b) -> 3(c) -> 3(f)
                                                  tail
remaining first:  7(d)
remaining second: 5(g) -> 8(h)

take 5(g), then attach remaining first/second as comparisons dictate:
dummy -> a -> e -> b -> c -> f -> g -> d -> h -> null
```

### 11. Step-by-Step Dry Run
| Step | First head | Second head | Choice | Merged identities |
|---:|---|---|---|---|
| 1 | `1(a)` | `1(e)` | `a`, tie favors first | `a` |
| 2 | `3(b)` | `1(e)` | `e` | `a, e` |
| 3 | `3(b)` | `3(f)` | `b`, tie favors first | `a, e, b` |
| 4 | `3(c)` | `3(f)` | `c`, tie favors first | `a, e, b, c` |
| 5 | `7(d)` | `3(f)` | `f` | `a, e, b, c, f` |
| 6 | `7(d)` | `5(g)` | `g` | `a, e, b, c, f, g` |
| 7 | `7(d)` | `8(h)` | `d` | `a, e, b, c, f, g, d` |
| finish | `null` | `8(h)` | attach remainder | `a, e, b, c, f, g, d, h` |

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function mergeSortedLists(
  first: ListNode<number> | null,
  second: ListNode<number> | null,
): ListNode<number> | null {
  const dummy = new ListNode<number>(0);
  let tail = dummy;
  let firstCurrent = first;
  let secondCurrent = second;

  while (firstCurrent !== null && secondCurrent !== null) {
    if (firstCurrent.value <= secondCurrent.value) {
      const next = firstCurrent.next;
      tail.next = firstCurrent;
      tail = firstCurrent;
      firstCurrent = next;
    } else {
      const next = secondCurrent.next;
      tail.next = secondCurrent;
      tail = secondCurrent;
      secondCurrent = next;
    }
  }

  tail.next = firstCurrent ?? secondCurrent;
  return dummy.next;
}
```

### 13. Code Walkthrough
`tail` always ends the already merged prefix, while `firstCurrent` and `secondCurrent` head the untouched remainders. Saving the chosen node's old successor before linking it advances the correct input without losing access. The `<=` comparison enforces the tie rule. Once either current pointer is `null`, the other remainder is already sorted and no more pairwise comparisons are necessary.

### 14. Complexity Analysis
**Time: `O(m + n)`** because each original node is selected or attached once. **Space: `O(1)` auxiliary**: the sentinel and pointer variables occupy constant space. The sentinel is not returned as data; all `m + n` output data nodes are the original objects. Precondition validation, if requested, has its own separate cost.

### 15. Edge Cases
- Both heads are `null`, producing `null`.
- One head is `null`, producing the other original head without copying.
- All first-list values precede all second-list values, or vice versa.
- Equal values across lists place every applicable first-list node before second-list nodes under the head-by-head tie rule.
- Duplicate values within either list retain their original within-list identity order.

### 16. Common Mistakes
- Using `<` instead of `<=`, which lets the second list win equal-value ties.
- Allocating replacement nodes and violating the reuse contract.
- Forgetting that callers holding node references observe the rewired `next` links.
- Assuming shared nodes are harmless; overlapping inputs can create cycles or duplicate structural ownership.
- Charging precondition validation to the stated merge complexity without saying so.

### 17. Interview Follow-ups
1. Validate acyclicity, sortedness, safe integers, and disjoint identity before merging. 2. Merge in descending order. 3. Return the number of cross-list comparisons. 4. Merge without even a sentinel node.

### 18. Variations
- Merge immutable lists by allocating copies. - Merge doubly linked lists and repair `previous` links. - Merge circular sorted lists. - Merge by a generic comparator while preserving source stability.

### 19. Pattern Connections
This is the linked-list form of the merge step in merge sort. It also underlies merging sorted logs and the pairwise building block for merging `k` sorted lists; the key invariant is that the result prefix is final while each remaining input stays sorted.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/merge-sorted` with `{ first: number[], second: number[] }`. Validate safe integers and nondecreasing order at the request boundary, construct disjoint request-scoped nodes with stable IDs, and run only the optimized merge for tracing. Return merged values plus bounded events such as `{ firstId, secondId, chosenId, reason }`, where `reason` distinguishes `smaller`, `tie-first`, and `remainder`; never serialize internal node links directly.

### 21. Mini Challenge
Implement the optimized merge without a dummy sentinel while preserving the first-list tie rule, original node identities, `O(m + n)` time, and `O(1)` auxiliary space.

## Problem 108 — Remove Duplicates from Sorted Linked List

### 1. Problem Statement
Given the head of an acyclic singly linked list of numeric safe-integer values sorted in nondecreasing order, remove duplicate nodes in place and return the same head reference. Keep the first node of each equal-value run. Equality uses numeric `===`, so `-0` and `0` are duplicates; `NaN` is excluded by the safe-integer precondition. Every removed node must be detached by setting its `next` to `null`. A `null` list remains `null`.

Sortedness, safe-integer values, and acyclicity are caller preconditions. Here, `n` is the input length.

### 2. Example
```text
Input identities:  1(a) -> 1(b) -> 2(c) -> 3(d) -> 3(e) -> 3(f) -> null
Output identities: 1(a) -> 2(c) -> 3(d) -> null

Detached nodes:
1(b) -> null
3(e) -> null
3(f) -> null

Output head === input head
```

### 3. Understand the Problem
The list itself must be rewired; returning newly allocated values would violate identity preservation. The first node in each run survives, and every later equal node is individually detached so external references to removed nodes no longer lead into the retained chain. The sorted order places all duplicates next to their first occurrence.

#### In Simple Terms
Keep the first node in each block of equal numbers, skip the rest, and cut every skipped node's outgoing arrow.

### 4. Real-World Analogy
A sorted guest list contains repeated tickets next to one another. Keep the first physical ticket for each number, remove later copies, and place each removed ticket by itself rather than leaving it stapled to the list.

### 5. Key Observation
In a sorted list, all nodes equal to a keeper form one contiguous run. Therefore, comparing `keeper.value` with `keeper.next.value` is enough: an equal successor is removable, while a different successor becomes the next keeper.

### 6. Pattern
```text
Pattern: Sorted Run Compression In Place
Why this pattern applies: equal values are contiguous in sorted order
How to recognize it: keep one representative per adjacent run and preserve its identity
Similar problems: unique elements in a sorted array, run-length encoding, compact sorted records
```

### 7. Brute Force
Traverse with a `Set<number>` of values already kept. For each node, keep it if its value has not appeared; otherwise save its successor, bypass it through `previous.next`, set the removed node's `next` to `null`, and continue from the saved successor. This works even without sortedness, so it does not exploit the strongest input guarantee. For safe integers, the set's SameValueZero equality agrees with the required `===` behavior, including treating `-0` and `0` as equal.

#### Pseudocode
```text
seen = empty value set
previous = null
current = head
while current is not null:
    next = current.next
    if seen contains current.value:
        previous.next = next
        current.next = null
    else:
        add current.value to seen
        previous = current
    current = next
return head
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function removeSortedDuplicatesBruteForce(
  head: ListNode<number> | null,
): ListNode<number> | null {
  const seen = new Set<number>();
  let previous: ListNode<number> | null = null;
  let current = head;

  while (current !== null) {
    const next = current.next;

    if (seen.has(current.value)) {
      previous!.next = next;
      current.next = null;
    } else {
      seen.add(current.value);
      previous = current;
    }

    current = next;
  }

  return head;
}
```

#### Visualization
```text
seen {}       keep 1(a)       seen {1}
1(a) -> 1(b) -> 2(c) -> 3(d) -> 3(e) -> 3(f)

seen has 1: bypass 1(b), then set 1(b).next = null
seen lacks 2: keep 2(c)
seen lacks 3: keep 3(d)
seen has 3: detach 3(e), then detach 3(f)

retained: 1(a) -> 2(c) -> 3(d) -> null
```

#### Dry Run
Node `a` is the first `1`, so the baseline records `1` and makes `a` the previous retained node. Node `b` is bypassed and detached. Nodes `c` and `d` introduce `2` and `3`. Nodes `e` and `f` are each bypassed from `d` using their saved successors, then each receives `next = null`.

#### Brute-Force Complexity
**Time: `O(n)` expected** because each node performs expected constant-time set work. **Space: `O(n)` auxiliary** for up to one stored value per distinct run. The original head and first node of every value are preserved.

### 8. Why Brute Force Is Not Ideal
The set solves a more general unsorted problem and stores every distinct value. Sortedness already guarantees that any duplicate of the current keeper must be adjacent, so one local comparison can remove duplicates with constant auxiliary space.

### 9. Optimal Approach
Start `keeper` at `head`. While it has a successor, compare their values with `===`. If equal, save the duplicate's successor, set the duplicate's `next` to `null`, and reconnect `keeper.next` to the saved successor; do not advance `keeper`, because another duplicate may follow. If different, advance `keeper`. Return the original `head`.

### 10. Visualization
```text
keeper
  |
  v
1(a) -> 1(b) -> 2(c) -> 3(d) -> 3(e) -> 3(f) -> null
         duplicate

save next = 2(c)
set 1(b).next = null
set 1(a).next = 2(c)

later at 3(d):
3(d) -> 3(e) -> 3(f) -> null
  keep    cut      cut

result: 1(a) -> 2(c) -> 3(d) -> null
```

### 11. Step-by-Step Dry Run
| Step | Keeper | Successor | Comparison | Action |
|---:|---|---|---|---|
| 1 | `1(a)` | `1(b)` | equal | detach `b`, reconnect to `c` |
| 2 | `1(a)` | `2(c)` | different | advance keeper to `c` |
| 3 | `2(c)` | `3(d)` | different | advance keeper to `d` |
| 4 | `3(d)` | `3(e)` | equal | detach `e`, reconnect to `f` |
| 5 | `3(d)` | `3(f)` | equal | detach `f`, reconnect to `null` |
| finish | `3(d)` | `null` | no successor | return original head `a` |

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function removeSortedDuplicates(
  head: ListNode<number> | null,
): ListNode<number> | null {
  let keeper = head;

  while (keeper !== null && keeper.next !== null) {
    if (keeper.value === keeper.next.value) {
      const removed = keeper.next;
      const next = removed.next;
      removed.next = null;
      keeper.next = next;
    } else {
      keeper = keeper.next;
    }
  }

  return head;
}
```

### 13. Code Walkthrough
`keeper` always references the first retained node of the current run. On equality, `removed` is saved as an object and its successor is saved separately before detachment, so reconnecting cannot lose the rest of the list. Keeping `keeper` stationary removes runs of any length. On inequality, the successor is the first node of a new run and becomes the next keeper.

### 14. Complexity Analysis
**Time: `O(n)`** because each node is examined once and either retained or detached. **Space: `O(1)` auxiliary** for pointer references. No node is allocated, the return value is the original head, and every removed node ends with `next === null`.

### 15. Edge Cases
- `head === null` returns `null`.
- A one-node list returns the same node unchanged.
- A list with no duplicates preserves every link.
- A list containing one repeated value keeps only its original head and detaches every later node.
- `-0` and `0` compare equal with `===` and belong to the same duplicate run.

### 16. Common Mistakes
- Advancing `keeper` immediately after a deletion and leaving later nodes in the same run.
- Bypassing a duplicate without setting the removed node's `next` to `null`.
- Returning a new head or allocating replacement nodes.
- Using `Object.is`, which would incorrectly distinguish `-0` from `0` for this contract.
- Using a set in the optimized solution despite the sorted adjacency guarantee.

### 17. Interview Follow-ups
1. Remove every value that appears more than once instead of keeping one. 2. Return the detached nodes as a separate list. 3. Count duplicates without mutating. 4. Validate sortedness before changing any link.

### 18. Variations
- Compact a sorted array in place. - Keep at most two nodes from each run. - Merge counts into `{ value, frequency }` nodes. - Deduplicate a sorted doubly linked list and repair both directions.

### 19. Pattern Connections
This is run compression on a linked structure. Sortedness turns global duplicate detection into an adjacent comparison, the same idea used by two-pointer array compaction and run-length encoding.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/dedupe-sorted` with `{ values: number[] }`. Validate safe integers and nondecreasing order before constructing request-scoped nodes with stable IDs. Run only the optimized adjacent comparison for tracing, and return `{ values, removedIds, trace, truncated }`; each bounded event can record `{ keeperId, candidateId, equal, action }`. Keep detached node objects internal rather than serializing links.

### 21. Mini Challenge
Change the optimized function to return `{ head, removed }`, where `removed` is a new chain of the detached original nodes in removal order, while the retained chain still keeps first occurrences and the algorithm uses `O(1)` auxiliary pointer storage.

## Problem 109 — Remove Duplicates from Unsorted Linked List

### 1. Problem Statement
Given the head of an acyclic singly linked list of generic values, remove duplicates in place and return the same head reference. Equality uses SameValueZero: values are equal when `left === right` or both are `NaN`. Thus `NaN` equals `NaN`, `-0` equals `0`, primitives compare by value, and objects compare by reference rather than deep structure. Preserve the first occurrence order and node identities. Every removed node must be detached by setting its `next` to `null`. A `null` list remains `null`.

Here, `n` is the input length. Acyclicity is a caller precondition.

### 2. Example
```text
Input identities:
3(a) -> NaN(b) -> 2(c) -> 3(d) -> -0(e) -> NaN(f) -> 0(g) -> null

Output identities:
3(a) -> NaN(b) -> 2(c) -> -0(e) -> null

Detached: 3(d), NaN(f), 0(g), each with next -> null
The first occurrence of each SameValueZero value survives in original order.
```

### 3. Understand the Problem
Unsorted duplicates may be far apart, so adjacency is not enough. The result must preserve the first node encountered for each value and bypass later equal nodes without replacing retained objects. SameValueZero matches JavaScript `Set` semantics: unlike `===`, it equates `NaN` with itself, and unlike `Object.is`, it does not distinguish `-0` from `0`.

#### In Simple Terms
Walk from left to right, keep the first time each value appears, remove later matches, and cut every removed node free from the list.

### 4. Real-World Analogy
An unsorted check-in queue may repeat the same badge anywhere. The first appearance keeps its place; every later matching badge leaves the queue and is fully unhooked from the people behind it.

### 5. Key Observation
The decision for each node depends on whether an equal value appeared anywhere earlier, not just beside it. A nested runner can discover later duplicates without extra storage, while a set can summarize all prior values and make each decision in expected constant time.

### 6. Pattern
```text
Pattern: Stable First-Occurrence Filtering
Why this pattern applies: remove later matches while retaining original encounter order
How to recognize it: unsorted input, global duplicate detection, in-place link mutation
Similar problems: unique stream filtering, duplicate record cleanup, stable array deduplication
```

### 7. Brute Force
For each retained `anchor`, scan the rest of the chain with `previous` and `current`. When `current.value` is SameValueZero-equal to the anchor value, save `current.next`, bypass `current`, detach it, and continue from the saved node without advancing `previous`. Otherwise advance both runner pointers. After the runner finishes, move `anchor` to its next retained node.

#### Pseudocode
```text
anchor = head
while anchor is not null:
    previous = anchor
    current = anchor.next
    while current is not null:
        next = current.next
        if current.value SameValueZero-equals anchor.value:
            previous.next = next
            current.next = null
        else:
            previous = current
        current = next
    anchor = anchor.next
return head
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function sameValueZero<T>(left: T, right: T): boolean {
  return left === right || (left !== left && right !== right);
}

function removeUnsortedDuplicatesBruteForce<T>(
  head: ListNode<T> | null,
): ListNode<T> | null {
  let anchor = head;

  while (anchor !== null) {
    let previous = anchor;
    let current = anchor.next;

    while (current !== null) {
      const next = current.next;

      if (sameValueZero(anchor.value, current.value)) {
        previous.next = next;
        current.next = null;
      } else {
        previous = current;
      }

      current = next;
    }

    anchor = anchor.next;
  }

  return head;
}
```

#### Visualization
```text
anchor 3(a): scan right, remove 3(d)
3(a) -> NaN(b) -> 2(c) -> -0(e) -> NaN(f) -> 0(g)

anchor NaN(b): scan right, remove NaN(f)
3(a) -> NaN(b) -> 2(c) -> -0(e) -> 0(g)

anchor -0(e): SameValueZero matches 0(g), so remove 0(g)
3(a) -> NaN(b) -> 2(c) -> -0(e) -> null
```

#### Dry Run
Anchor `3(a)` removes `3(d)` and leaves its runner predecessor at `2(c)`. Anchor `NaN(b)` uses the explicit SameValueZero helper to remove `NaN(f)`. After earlier removals, anchor `-0(e)` reaches `0(g)`; `-0 === 0`, so node `g` is bypassed and detached. The retained identities remain `a, b, c, e` in their original order.

#### Brute-Force Complexity
**Time: `O(n^2)`** in the worst case because each retained anchor may scan most later nodes. **Space: `O(1)` auxiliary** for pointers. The algorithm preserves first occurrences and detaches every removed node without allocating data nodes.

### 8. Why Brute Force Is Not Ideal
The nested runner repeatedly compares nodes that were already known to differ from earlier anchors. A set remembers every encountered value, allowing one left-to-right pass at the cost of linear auxiliary storage.

### 9. Optimal Approach
Maintain a `Set<T>` named `seen`, plus `previous` and `current` pointers. JavaScript sets use SameValueZero, exactly matching the contract. If `seen` already contains `current.value`, save its successor, bypass and detach the node, then continue from the saved successor. Otherwise add the value and advance `previous`. Return the original `head`.

### 10. Visualization
```text
current: 3(a)    seen: {}              -> keep, add 3
current: NaN(b)  seen: {3}             -> keep, add NaN
current: 2(c)    seen: {3, NaN}        -> keep, add 2
current: 3(d)    seen: {3, NaN, 2}     -> bypass and detach
current: -0(e)   seen: {3, NaN, 2}     -> keep, add -0
current: NaN(f)  seen contains NaN      -> bypass and detach
current: 0(g)    seen contains 0/-0     -> bypass and detach

result: 3(a) -> NaN(b) -> 2(c) -> -0(e) -> null
```

### 11. Step-by-Step Dry Run
| Step | Current | Seen before | Decision | Previous retained node |
|---:|---|---|---|---|
| 1 | `3(a)` | `{}` | keep and add `3` | `a` |
| 2 | `NaN(b)` | `{3}` | keep and add `NaN` | `b` |
| 3 | `2(c)` | `{3, NaN}` | keep and add `2` | `c` |
| 4 | `3(d)` | `{3, NaN, 2}` | detach duplicate | `c` |
| 5 | `-0(e)` | `{3, NaN, 2}` | keep and add `-0` | `e` |
| 6 | `NaN(f)` | `{3, NaN, 2, -0}` | detach duplicate | `e` |
| 7 | `0(g)` | `{3, NaN, 2, -0}` | detach duplicate of `-0` | `e` |

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function removeUnsortedDuplicates<T>(
  head: ListNode<T> | null,
): ListNode<T> | null {
  const seen = new Set<T>();
  let previous: ListNode<T> | null = null;
  let current = head;

  while (current !== null) {
    const next = current.next;

    if (seen.has(current.value)) {
      previous!.next = next;
      current.next = null;
    } else {
      seen.add(current.value);
      previous = current;
    }

    current = next;
  }

  return head;
}
```

### 13. Code Walkthrough
The first node cannot be a duplicate, so `previous` is nonnull whenever the duplicate branch runs; the assertion records that invariant. Saving `next` before mutation keeps traversal alive after `current.next` is cleared. `previous` advances only for retained nodes, which lets consecutive duplicates be bypassed correctly. The native set supplies the contract's `NaN` and signed-zero behavior without a custom helper.

### 14. Complexity Analysis
**Time: `O(n)` expected** from one traversal and expected constant-time set operations. **Space: `O(n)` auxiliary** in the worst case when every value is distinct. The same head and first-occurrence node identities are returned, and each removed node is detached.

### 15. Edge Cases
- `head === null` returns `null`.
- A one-node list returns the same node unchanged.
- Consecutive and widely separated duplicates are both removed safely.
- Multiple `NaN` values keep the first `NaN` node because SameValueZero equates them.
- `-0` and `0` share one set entry, so whichever appears first survives.
- Distinct object instances with equal fields both remain; repeated references to the same object are duplicates.

### 16. Common Mistakes
- Using `Object.is`, which incorrectly treats `-0` and `0` as distinct for this contract.
- Using only `===`, which fails to recognize repeated `NaN` values.
- Advancing `previous` after removing a node, which can break consecutive deletion.
- Clearing `current.next` before saving the successor.
- Deep-comparing objects even though the contract uses reference equality for them.

### 17. Interview Follow-ups
1. Use a caller-provided equality and hashing strategy. 2. Return all detached nodes in removal order. 3. Keep the last occurrence instead of the first. 4. Deduplicate with bounded memory and discuss the tradeoff.

### 18. Variations
- Deduplicate by an object key such as `id`. - Count occurrences without mutation. - Remove all values that occur more than once. - Deduplicate a stream where links cannot be revisited.

### 19. Pattern Connections
This is stable filtering backed by a hash set. It connects linked-list mutation with array deduplication, streaming `distinct` operations, and database uniqueness rules; the equality definition is part of the API contract, not an implementation footnote.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/dedupe-unsorted` with `{ values: unknown[] }` only after defining which JSON values the endpoint accepts and how they map to SameValueZero. Construct request-scoped nodes with stable IDs, run only the optimized set-based pass for tracing, and return `{ values, removedIds, trace, truncated }`. Bound events such as `{ nodeId, seenBefore, action }`; for object payloads, prefer an explicit key contract because JSON cannot express repeated in-memory object identity or `NaN` directly.

### 21. Mini Challenge
Implement `removeUnsortedDuplicatesBy<T, K>(head, keyOf)` so duplicates are determined by SameValueZero equality of `keyOf(node.value)`, while preserving first-occurrence node identity, detaching removals, and retaining `O(n)` expected time.

## Problem 110 — Check if Linked List is Palindrome

### 1. Problem Statement
Given the head of an acyclic singly linked list of generic values, return whether its values form a palindrome. Equality uses SameValueZero: values are equal when `left === right` or both are `NaN`, so `NaN` equals `NaN`, `-0` equals `0`, and objects compare by reference. The optimized algorithm may reverse links temporarily, but it must restore the exact original `next`-link structure before returning. It must not leave any lasting mutation. Empty and single-node lists return `true`.

Here, `n` is the input length. Acyclicity is a caller precondition.

### 2. Example
```text
Input:  NaN(a) -> -0(b) -> 0(c) -> NaN(d) -> null
Output: true

SameValueZero comparisons:
NaN equals NaN
-0 equals 0

After the call, the links are still exactly a -> b -> c -> d -> null.
```

### 3. Understand the Problem
A palindrome reads the same from both ends. A singly linked list supports forward traversal but has no direct backward movement, so the two ends cannot be compared in place without either storing values or changing part of the link direction temporarily. Restoration is part of correctness, including when a mismatch is found.

#### In Simple Terms
Compare matching values from the front and back, answer `true` only if every pair matches, and leave every link exactly as it started.

### 4. Real-World Analogy
Imagine checking whether train-car labels are symmetric. You can write all labels on a clipboard and compare its ends, or temporarily turn the second half of the train around, compare paired cars, and then reconnect every coupling in its original direction before anyone uses the train again.

### 5. Key Observation
Slow and fast pointers locate the end of the first half. With `fast.next !== null && fast.next.next !== null`, the slow pointer stops at the middle node for odd lengths and at the left-middle node for even lengths. Reversing only `firstHalfEnd.next` aligns the front half with the back half. The unmatched center of an odd-length list is intentionally ignored.

### 6. Pattern
```text
Pattern: Fast/Slow Midpoint + Reverse/Restore
Why this pattern applies: symmetric pairs live in opposite halves of a forward-only list
How to recognize it: palindrome check, O(1) working space, strict no-lasting-mutation contract
Similar problems: reorder list, twin sum, split-and-reverse comparisons
```

### 7. Brute Force
Copy every value into an array. Use `left` and `right` indices to compare mirrored entries with SameValueZero, moving inward until they cross. This baseline never changes a link.

#### Pseudocode
```text
values = []
current = head
while current is not null:
    append current.value to values
    current = current.next
left = 0
right = values.length - 1
while left < right:
    if values[left] and values[right] are not SameValueZero-equal:
        return false
    left++
    right--
return true
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function sameValueZero<T>(left: T, right: T): boolean {
  return left === right || (left !== left && right !== right);
}

function isPalindromeBruteForce<T>(head: ListNode<T> | null): boolean {
  const values: T[] = [];
  let current = head;

  while (current !== null) {
    values.push(current.value);
    current = current.next;
  }

  let left = 0;
  let right = values.length - 1;

  while (left < right) {
    if (!sameValueZero(values[left], values[right])) {
      return false;
    }
    left++;
    right--;
  }

  return true;
}
```

#### Visualization
```text
list:   NaN -> -0 -> 0 -> NaN
array: [NaN,   -0,   0,    NaN]
        L                    R   equal
              L     R           equal
```

#### Dry Run
The copied array is `[NaN, -0, 0, NaN]`. The outer values match because SameValueZero equates both `NaN` values. The inner values match because `-0 === 0`. The indices cross, so the function returns `true`; the list was never mutated.

#### Brute-Force Complexity
**Time: `O(n)`** for copying and comparing. **Space: `O(n)` working space** for the value array. The boolean result has `O(1)` output size.

### 8. Why Brute Force Is Not Ideal
The array makes backward comparison convenient, but it duplicates every stored value reference. The same comparison can use constant working space by reversing the second half, provided restoration happens on every return path.

### 9. Optimal Approach
Find the end of the first half with slow and fast pointers. Reverse the second half and retain its reversed head. Compare the first half against that reversed chain using SameValueZero, recording a mismatch in a boolean rather than returning early. Reverse the same chain again and reconnect it through `firstHalfEnd.next` before returning the recorded result. External references can observe transient link changes only during the synchronous call; the postcondition restores every original link.

### 10. Visualization
```text
Odd:  1(a) -> 2(b) -> 3(c) -> 2(d) -> 1(e)
firstHalfEnd = c
reverse d..e: 1(e) -> 2(d) -> null
compare: a/e, b/d; center c is unpaired
restore: c.next = d, d.next = e, e.next = null

Even: 1(a) -> 2(b) -> 2(c) -> 1(d)
firstHalfEnd = b
reverse c..d: 1(d) -> 2(c) -> null
compare: a/d, b/c
restore: b.next = c, c.next = d, d.next = null
```

### 11. Step-by-Step Dry Run
| Phase | Left | Right | State / action |
|---|---|---|---|
| midpoint | `1(a)` | `fast = 1(a)` | slow stops at `3(c)` for the odd list |
| reverse | - | `1(e)` | second half becomes `1(e) -> 2(d)` |
| compare 1 | `1(a)` | `1(e)` | equal; advance both |
| compare 2 | `2(b)` | `2(d)` | equal; reversed half is exhausted |
| restore | - | `1(e)` | reverse again and assign `c.next = d` |
| return | - | - | `true`, with original identities and links restored |

For `1 -> 2 -> 3 -> 4 -> 1`, comparison records `2 !== 4`, stops comparing, restores the entire second half, and only then returns `false`.

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function sameValueZero<T>(left: T, right: T): boolean {
  return left === right || (left !== left && right !== right);
}

function reverseList<T>(head: ListNode<T> | null): ListNode<T> | null {
  let previous: ListNode<T> | null = null;
  let current = head;

  while (current !== null) {
    const next = current.next;
    current.next = previous;
    previous = current;
    current = next;
  }

  return previous;
}

function findFirstHalfEnd<T>(head: ListNode<T>): ListNode<T> {
  let slow = head;
  let fast = head;

  while (fast.next !== null && fast.next.next !== null) {
    slow = slow.next!;
    fast = fast.next.next;
  }

  return slow;
}

function isPalindrome<T>(head: ListNode<T> | null): boolean {
  if (head === null || head.next === null) {
    return true;
  }

  const firstHalfEnd = findFirstHalfEnd(head);
  const reversedSecondHalfHead = reverseList(firstHalfEnd.next);
  let left: ListNode<T> | null = head;
  let right = reversedSecondHalfHead;
  let matches = true;

  while (matches && right !== null) {
    if (left === null || !sameValueZero(left.value, right.value)) {
      matches = false;
    } else {
      left = left.next;
      right = right.next;
    }
  }

  firstHalfEnd.next = reverseList(reversedSecondHalfHead);
  return matches;
}
```

### 13. Code Walkthrough
The midpoint loop leaves `slow` at the first half's final node for both parities. `reverseList` changes only the second-half links. The comparison ends on a mismatch or when every second-half node has matched, but it never returns from inside that loop. Calling `reverseList` again on the saved reversed head reconstructs the original second-half order, and assigning its result to `firstHalfEnd.next` restores the split link before the boolean is returned.

### 14. Complexity Analysis
**Time: `O(n)`**: midpoint search, reversal, comparison, and restoration are each linear over at most the list. **Space: `O(1)` auxiliary** because only pointers and a boolean are stored. The input structure is transiently mutated but has no lasting mutation, and the boolean output uses constant space.

### 15. Edge Cases
- `null` and a single node return `true` without mutation.
- Two equal nodes return `true`; two unequal nodes return `false`, both restored.
- Odd lengths skip the center value; even lengths compare every node in a pair.
- Repeated `NaN` values compare equal, and `-0` compares equal to `0`.
- Distinct object instances with equal fields do not match; the same object reference does.
- A mismatch near the front still triggers complete second-half restoration.

### 16. Common Mistakes
- Returning immediately on mismatch and leaving the second half reversed.
- Finding the wrong midpoint and comparing the odd-length center against another node.
- Using `===` alone, which makes `NaN` fail the required equality.
- Using `Object.is`, which incorrectly separates `-0` and `0`.
- Restoring the reversed nodes but forgetting to reconnect `firstHalfEnd.next`.

### 17. Interview Follow-ups
1. Accept a caller-provided equality predicate. 2. Make the check concurrency-safe when other code can inspect links during the call. 3. Detect cycles rather than requiring acyclicity. 4. Return the first mismatching pair of indices while still restoring links.

### 18. Variations
- Check a doubly linked list from both ends. - Check a list whose values are case-insensitive strings. - Determine whether deleting at most one node can make a palindrome. - Compare node IDs rather than values.

### 19. Pattern Connections
This combines fast/slow midpoint discovery with reversible in-place transformation. It connects to reorder-list algorithms and transactional mutation: temporary state is acceptable only because the original structure is restored before control returns.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/palindrome` with `{ values: unknown[] }`, construct request-scoped nodes with stable IDs, and run only the optimized reverse/compare/restore path. Return `{ palindrome, restoredNodeIds, trace, truncated }`. Trace bounded events such as `{ phase, leftId, rightId, equal }`, plus restoration completion. JSON cannot represent `NaN` or distinguish signed zero reliably, so either restrict the endpoint contract to JSON-safe values or accept tagged values and decode them explicitly.

### 21. Mini Challenge
Implement `isPalindromeBy<T>(head, equals)` in `O(n)` time and `O(1)` auxiliary space, guaranteeing exact link restoration even when `equals` reports a mismatch on the first pair.

## Problem 111 — Intersection of Two Linked Lists

### 1. Problem Statement
Given the heads of two acyclic singly linked lists, return the first node reference shared by both lists, or `null` if they do not intersect. Intersection is determined by node identity, not by value. The lists may have different prefix lengths, and neither list may be mutated. If two singly linked lists share a node, every node after it is necessarily the same shared suffix because each node has only one `next` link.

Here, `m` and `n` are the two input lengths. Acyclicity is a caller precondition.

### 2. Example
```text
first:   4(a) -> 1(b) --+
                         +-> 8(x) -> 4(y) -> 5(z) -> null
second:  5(c) -> 6(d) -> 1(e) --+

Output: node x, the exact shared node reference

Equal values in b and e do not intersect because b !== e by identity.
```

### 3. Understand the Problem
Two nodes containing the same value are not enough; both traversals must reach the same object. Different prefix lengths make lockstep traversal misalign corresponding suffix nodes. Because the lists are acyclic and singly linked, a shared node cannot later diverge: both paths read the same node's single `next` reference.

#### In Simple Terms
Find the first physical node object that belongs to both chains, ignoring whether separate nodes happen to store equal values.

### 4. Real-World Analogy
Two walking trails may have separately named markers with identical text, but they intersect only where both routes use the very same marker post. Once the trails merge into one one-way path, they cannot split again without that shared post having two different next directions.

### 5. Key Observation
If pointer A walks first list then second, and pointer B walks second list then first, both travel exactly `m + n` positions. Switching heads cancels the prefix-length difference. They therefore meet at the first shared node, or both become `null` when no shared suffix exists.

### 6. Pattern
```text
Pattern: Pointer Switching / Path-Length Equalization
Why this pattern applies: two paths have unequal private prefixes but a possible shared suffix
How to recognize it: identity intersection, acyclic lists, O(1) auxiliary-space target
Similar problems: aligned suffix comparison, linked-list merge point, path convergence
```

### 7. Brute Force
Insert every node reference from the first list into a set. Scan the second list from its head; the first node found in the set is the first shared node along the second path and therefore the start of the common suffix. Do not compare values or alter links.

#### Pseudocode
```text
nodesInFirst = empty identity set
current = firstHead
while current is not null:
    add current to nodesInFirst
    current = current.next
current = secondHead
while current is not null:
    if nodesInFirst contains current:
        return current
    current = current.next
return null
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function getIntersectionNodeBruteForce<T>(
  firstHead: ListNode<T> | null,
  secondHead: ListNode<T> | null,
): ListNode<T> | null {
  const nodesInFirst = new Set<ListNode<T>>();
  let current = firstHead;

  while (current !== null) {
    nodesInFirst.add(current);
    current = current.next;
  }

  current = secondHead;
  while (current !== null) {
    if (nodesInFirst.has(current)) {
      return current;
    }
    current = current.next;
  }

  return null;
}
```

#### Visualization
```text
Set after first scan: { a, b, x, y, z }

second scan:
c not in set
d not in set
e not in set
x is in set -> return x
```

#### Dry Run
The first traversal stores identities `a, b, x, y, z`. The second traversal checks `c`, `d`, and `e`; even if `e.value === b.value`, `e` is a different object and is absent. Node `x` is the first identity present in the set, so the function returns that exact reference. No `next` link changes.

#### Brute-Force Complexity
**Time: `O(m + n)` expected** using expected constant-time set operations. **Space: `O(m)` auxiliary** for identities from the first list. Returning a node reference uses `O(1)` output space.

### 8. Why Brute Force Is Not Ideal
The set solves alignment but stores every node from one list. Pointer switching achieves the same path alignment implicitly, preserving linear time while using only two pointers.

### 9. Optimal Approach
Start one pointer at each head. While their references differ, advance each pointer by one node; when a pointer becomes `null`, redirect it to the other list's head. Each pointer traverses both prefixes and the shared suffix in opposite list order. They meet at the first shared reference. If there is no intersection, both reach `null` together. The acyclic precondition guarantees termination.

### 10. Visualization
```text
A path: a -> b -> x -> y -> z -> null -> c -> d -> e -> x
B path: c -> d -> e -> x -> y -> z -> null -> a -> b -> x
                                                        ^ meet

Each route includes both private prefixes, so their length difference vanishes.
```

### 11. Step-by-Step Dry Run
For compactness, use `first = a -> b -> x -> y` and `second = c -> x -> y`.

| Step | pointerA | pointerB | Same identity? |
|---:|---|---|---|
| 0 | `a` | `c` | no |
| 1 | `b` | `x` | no |
| 2 | `x` | `y` | no |
| 3 | `y` | `null` | no |
| 4 | `null` | `a` | no |
| 5 | `c` | `b` | no |
| 6 | `x` | `x` | yes, return `x` |

With disjoint lists, the same switching rule eventually makes both pointers `null`, which is equal and ends the loop.

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function getIntersectionNode<T>(
  firstHead: ListNode<T> | null,
  secondHead: ListNode<T> | null,
): ListNode<T> | null {
  let pointerA = firstHead;
  let pointerB = secondHead;

  while (pointerA !== pointerB) {
    pointerA = pointerA === null ? secondHead : pointerA.next;
    pointerB = pointerB === null ? firstHead : pointerB.next;
  }

  return pointerA;
}
```

### 13. Code Walkthrough
The loop compares references directly. A pointer that falls off its original list continues from the other head, so both pointers cover `m + n` positions in complementary order. Equal heads exit immediately. A null head needs no special branch: switching still aligns the paths, and disjoint or empty inputs converge at `null`. No field is written.

### 14. Complexity Analysis
**Time: `O(m + n)`** because each pointer traverses at most both lists once. **Space: `O(1)` auxiliary** for two pointers. The returned shared reference or `null` uses constant output space, and neither input is mutated.

### 15. Edge Cases
- If either head is `null`, the result is `null` unless both are the same `null` reference, which is also `null`.
- If both heads are the same node, that node is returned immediately.
- Equal values in distinct nodes do not count as intersection.
- Different prefix lengths are neutralized by switching heads.
- Disjoint lists make both pointers meet at `null`.
- One list may begin inside the other, making its head the first shared node.

### 16. Common Mistakes
- Comparing `node.value` instead of node references.
- Resetting both pointers to their own heads, which never corrects the length difference.
- Mutating a tail to connect lists temporarily and violating the no-mutation contract.
- Forgetting that cyclic inputs can prevent the switching proof from terminating.
- Returning a later shared node even though the first shared node starts the common suffix.

### 17. Interview Follow-ups
1. Solve by computing both lengths and advancing the longer prefix. 2. Handle lists that may contain cycles. 3. Find intersection among three acyclic lists. 4. Return the private-prefix lengths along with the shared node.

### 18. Variations
- Determine whether two lists intersect without returning the node. - Find the merge point by tail identity plus length alignment. - Intersect immutable persistent lists. - Report the number of shared suffix nodes.

### 19. Pattern Connections
Pointer switching is implicit length normalization. It connects to two-route synchronization and aligned suffix comparisons: instead of measuring each prefix explicitly, both pointers pay both prefix costs and arrive at the shared suffix in phase.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/intersection` with a graph-shaped payload such as `{ nodes: [{ id, value, nextId }], firstHeadId, secondHeadId }`; two independent value arrays cannot express shared identity. Validate IDs, one outgoing link per node, reachability, and acyclicity. Run only the optimized pointer-switching algorithm and return `{ intersectionId, trace, truncated }`, with bounded events like `{ step, pointerAId, pointerBId, switchedA, switchedB }`. Never clone each path separately after validation, because cloning would erase the shared references the endpoint is meant to test.

### 21. Mini Challenge
Implement the length-alignment alternative in `O(m + n)` time and `O(1)` auxiliary space, then prove that it returns the same first shared node as pointer switching for every acyclic input pair.

## Problem 112 — Add Two Numbers

### 1. Problem Statement
Two acyclic singly linked lists represent nonnegative base-10 integers in reverse digit order. Every node value must be an integer from `0` through `9`; `null` represents zero. Return their arithmetic sum as a reverse-order list made entirely of fresh nodes, without mutating either input. The inputs may contain redundant high-order zeros, and the output is required only to be a valid representation, not a canonical one, so those zeros may remain. When both inputs are `null`, return one fresh node containing `0`.

Both lists are validated completely before output allocation begins. Invalid digits throw `RangeError`; the acyclic precondition remains the caller's responsibility. Here, `m` and `n` are the input lengths and `d = max(m, n)`.

### 2. Example
```text
first:  2 -> 4 -> 3 -> null  represents 342
second: 5 -> 6 -> 4 -> null  represents 465
output: 7 -> 0 -> 8 -> null  represents 807

Every output node is newly allocated. Neither input link changes.
```

### 3. Understand the Problem
Because digits are reversed, addition starts exactly where the heads point: the ones place. At each position, combine up to two digits with the incoming carry, emit the result modulo ten, and carry the quotient into the next position. Validation must finish first so an invalid later digit cannot leave behind a partially constructed output.

#### In Simple Terms
Add corresponding digits from left to right, carry tens forward, and build a completely new result list.

### 4. Real-World Analogy
This is column addition on paper rotated so the ones column comes first. Each clerk receives two digits and a carry, writes the new digit onto a fresh slip, and passes at most one carry to the next clerk without altering either original ledger.

### 5. Key Observation
Reverse order removes the need for stacks or list reversal: the required arithmetic order matches forward traversal. A final nonzero carry becomes one extra node. Redundant high-order input zeros may produce redundant high-order output zeros, which is permitted by the representation contract.

### 6. Pattern
```text
Pattern: Synchronized Traversal with Carry
Why this pattern applies: each output digit depends on aligned input digits and one carry bit
How to recognize it: positional arithmetic, reverse-order digits, fresh output chain
Similar problems: add binary strings, sum digit arrays, multiply linked-list numbers
```

### 7. Brute Force
First validate both lists. Convert each reverse-order digit sequence into a `bigint` using a growing decimal place, add the two values, then repeatedly take a remainder modulo `10n` to build a fresh reverse-order result. A small explicit mapper converts only a known remainder from `0n` through `9n` into its digit value; the full integer is never converted to `number`.

#### Pseudocode
```text
validate every digit in both lists
firstValue = reverseDigitsToBigInt(firstHead)
secondValue = reverseDigitsToBigInt(secondHead)
sum = firstValue + secondValue
if sum is zero: return fresh node 0
while sum > zero:
    append fresh node for sum modulo 10
    sum = sum divided by 10
return output head
```

#### Brute-Force TypeScript
```typescript
class ListNode {
  constructor(
    public value: number,
    public next: ListNode | null = null,
  ) {}
}

function validateDigitList(head: ListNode | null): void {
  let current = head;
  while (current !== null) {
    if (!Number.isInteger(current.value) || current.value < 0 || current.value > 9) {
      throw new RangeError("Every node value must be an integer from 0 through 9");
    }
    current = current.next;
  }
}

function reverseDigitsToBigInt(head: ListNode | null): bigint {
  let value = 0n;
  let place = 1n;
  let current = head;

  while (current !== null) {
    value += BigInt(current.value) * place;
    place *= 10n;
    current = current.next;
  }

  return value;
}

function bigIntRemainderToDigit(remainder: bigint): number {
  if (remainder === 0n) return 0;
  if (remainder === 1n) return 1;
  if (remainder === 2n) return 2;
  if (remainder === 3n) return 3;
  if (remainder === 4n) return 4;
  if (remainder === 5n) return 5;
  if (remainder === 6n) return 6;
  if (remainder === 7n) return 7;
  if (remainder === 8n) return 8;
  if (remainder === 9n) return 9;
  throw new RangeError("Remainder is not a decimal digit");
}

function bigIntToReverseDigits(value: bigint): ListNode {
  if (value === 0n) {
    return new ListNode(0);
  }

  const dummy = new ListNode(0);
  let tail = dummy;
  let remaining = value;

  while (remaining > 0n) {
    tail.next = new ListNode(bigIntRemainderToDigit(remaining % 10n));
    tail = tail.next;
    remaining /= 10n;
  }

  return dummy.next!;
}

function addTwoNumbersBruteForce(
  firstHead: ListNode | null,
  secondHead: ListNode | null,
): ListNode {
  validateDigitList(firstHead);
  validateDigitList(secondHead);

  const sum = reverseDigitsToBigInt(firstHead) + reverseDigitsToBigInt(secondHead);
  return bigIntToReverseDigits(sum);
}
```

#### Visualization
```text
2 -> 4 -> 3  => 2*1 + 4*10 + 3*100 = 342n
5 -> 6 -> 4  => 5*1 + 6*10 + 4*100 = 465n
342n + 465n = 807n

807n % 10n = 7
 80n % 10n = 0
  8n % 10n = 8
result: 7 -> 0 -> 8
```

#### Dry Run
Validation accepts all six digits before any result node exists. The converters produce `342n` and `465n`. Repeated remainder and division steps emit fresh nodes `7`, `0`, and `8`. The dummy node is not returned, and no input node is reused or changed.

#### Brute-Force Complexity
Let `d = max(m, n)`. **Time: `O(d^2)` conservatively** because repeated multiplication, addition, remainder, and division operate on `bigint` values whose digit width grows to `O(d)`; these are not unit-cost operations. **Space: `O(d)`** for arbitrary-precision integer storage plus the required `O(d)` fresh output. Excluding output, working storage is still `O(d)`.

### 8. Why Brute Force Is Not Ideal
Converting entire numbers introduces arbitrary-precision arithmetic whose operations become more expensive as values grow. Grade-school addition needs only constant-size digit arithmetic and never materializes either complete integer.

### 9. Optimal Approach
Validate both complete input lists first. Then walk them together with a carry, treating a missing node as digit zero. For each position, compute `sum = firstDigit + secondDigit + carry`, append a fresh node containing `sum % 10`, and set `carry = Math.floor(sum / 10)`. Continue while either pointer remains or carry is nonzero. If both heads are `null`, return a fresh zero node.

### 10. Visualization
```text
first:   9 -> 9 -> null
second:  1 -> null

position 0: 9 + 1 + carry 0 = 10 -> emit 0, carry 1
position 1: 9 + 0 + carry 1 = 10 -> emit 0, carry 1
position 2: 0 + 0 + carry 1 =  1 -> emit 1, carry 0

output: 0 -> 0 -> 1 -> null
```

### 11. Step-by-Step Dry Run
For `2 -> 4 -> 3` plus `5 -> 6 -> 4`:

| Position | First digit | Second digit | Carry in | Sum | Output digit | Carry out |
|---:|---:|---:|---:|---:|---:|---:|
| 0 | 2 | 5 | 0 | 7 | 7 | 0 |
| 1 | 4 | 6 | 0 | 10 | 0 | 1 |
| 2 | 3 | 4 | 1 | 8 | 8 | 0 |

The fresh output is `7 -> 0 -> 8`. The original six nodes retain their values and `next` links.

### 12. TypeScript Solution
```typescript
class ListNode {
  constructor(
    public value: number,
    public next: ListNode | null = null,
  ) {}
}

function validateDigitList(head: ListNode | null): void {
  let current = head;
  while (current !== null) {
    if (!Number.isInteger(current.value) || current.value < 0 || current.value > 9) {
      throw new RangeError("Every node value must be an integer from 0 through 9");
    }
    current = current.next;
  }
}

function addTwoNumbers(
  firstHead: ListNode | null,
  secondHead: ListNode | null,
): ListNode {
  validateDigitList(firstHead);
  validateDigitList(secondHead);

  const dummy = new ListNode(0);
  let tail = dummy;
  let first = firstHead;
  let second = secondHead;
  let carry = 0;

  while (first !== null || second !== null || carry !== 0) {
    const firstDigit = first?.value ?? 0;
    const secondDigit = second?.value ?? 0;
    const sum = firstDigit + secondDigit + carry;

    tail.next = new ListNode(sum % 10);
    tail = tail.next;
    carry = Math.floor(sum / 10);
    first = first?.next ?? null;
    second = second?.next ?? null;
  }

  return dummy.next ?? new ListNode(0);
}
```

### 13. Code Walkthrough
The validation passes complete before the dummy or any output node is allocated, so a bad digit throws without partial output construction. The addition loop reads at most one node from each input per position. Optional chaining supplies zero after a list ends. Since two digits and carry total at most `19`, the emitted digit and next carry are constant-size arithmetic. The fallback allocates the required zero node only when both inputs are `null`.

### 14. Complexity Analysis
**Time: `O(m + n)`** including validation and addition; each input is traversed once to validate and once at most to add. **Space: `O(1)` auxiliary** beyond the fresh output list. The output contains `O(max(m, n) + 1)` new nodes in the worst case. Neither input is mutated or reused.

### 15. Edge Cases
- `null + null` returns a fresh single node `0`.
- `null` plus a valid list returns an equal-valued but entirely fresh list.
- A final carry appends an extra node, as in `9 + 1 = 0 -> 1`.
- Unequal lengths use zero for the exhausted input.
- Redundant high-order zeros are accepted and may remain in the valid output representation.
- Negative, fractional, `NaN`, infinite, or greater-than-nine digits throw `RangeError` before output allocation.

### 16. Common Mistakes
- Returning an input node when the other input is `null`, violating fresh ownership.
- Forgetting the final carry node.
- Mutating input digits or links to reuse storage.
- Converting the whole value to `number`, which loses precision for long lists.
- Claiming `bigint` conversion is linear under a unit-cost model for growing integers.
- Allocating output before complete validation when the API promises validation first.

### 17. Interview Follow-ups
1. Support an arbitrary base from 2 through 36. 2. Add forward-order digit lists without mutating inputs. 3. Return a canonical output by trimming redundant most-significant zeros. 4. Detect cycles and report a structured validation error.

### 18. Variations
- Subtract two nonnegative numbers. - Add signed digit lists. - Stream digits when lengths are enormous. - Reuse nodes under an explicitly mutable ownership contract.

### 19. Pattern Connections
Carry propagation is a small state machine over synchronized traversals. It connects linked-list arithmetic to elementary addition, binary-string addition, and merge-like loops where exhausted inputs contribute a neutral value.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/add` with `{ first: number[] | null, second: number[] | null }`, validate acyclicity implicitly through array construction and validate every digit before invoking the algorithm. Build separate request-scoped input chains, run only the optimized carry traversal, and return `{ digits, trace, truncated }`. Bound trace events such as `{ position, firstDigit, secondDigit, carryIn, outputDigit, carryOut }`; never serialize or expose the discarded dummy node.

### 21. Mini Challenge
Implement `addTwoNumbersInBase(firstHead, secondHead, base)` for integer bases from `2` through `36`, preserving full prevalidation, fresh-node ownership, and `O(m + n)` time with `O(1)` auxiliary space excluding output.

## Problem 113 — Reverse Linked List in Groups of K

### 1. Problem Statement
Given the head of an acyclic generic singly linked list and a positive integer `k`, reverse every complete consecutive group of exactly `k` nodes. Leave a final group containing fewer than `k` nodes in its original order. Mutate and reuse the existing nodes, return the new head, and return `null` for an empty list. If `k` is not a positive integer, throw `RangeError` before mutating any link. Acyclicity is a caller precondition.

For `k = 1`, the list is unchanged. Node values are irrelevant: the operation changes links while preserving every data-node identity.

### 2. Example
```text
input:  A -> B -> C -> D -> E -> null, k = 2
output: B -> A -> D -> C -> E -> null

The final one-node group is incomplete, so E remains in place.
```

### 3. Understand the Problem
Groups are defined by the original consecutive traversal order. A complete group is reversed by changing `next` links, not by swapping values or allocating replacement data nodes. Before reversing a group in the optimized solution, first prove that its `k`th node exists; otherwise even a temporarily reversed short suffix would violate the contract.

#### In Simple Terms
Flip each full block of `k` physical nodes, but do not touch a shorter block at the end.

### 4. Real-World Analogy
Imagine train cars entering a service bay in fixed-size batches. A full batch leaves in reverse order using the same cars. If the last batch cannot fill the bay, it continues forward untouched. The batch size must be valid before any coupling is changed.

### 5. Key Observation
For each group, the node after the `k`th node is a fixed boundary named `groupNext`. Reversing links until the cursor reaches that boundary prevents the algorithm from consuming the next group. The old group start becomes the new group end and therefore the predecessor for the following iteration.

### 6. Pattern
```text
Pattern: Fixed-Size Window Rewiring with a Dummy Predecessor
Why this pattern applies: each complete node block is transformed in place
How to recognize it: grouped mutation, preserved identities, incomplete suffix unchanged
Similar problems: reverse a sublist, rotate fixed blocks, pairwise node swaps
```

### 7. Brute Force
Validate `k`, collect every node reference in an array, and copy those references into an output-order array. Reverse each complete `k`-sized slice in the output-order array. Finally, rewire every referenced node to its successor in that logical order and terminate the last node with `null`. The array changes order; the data nodes themselves are reused.

#### Pseudocode
```text
if k is not a positive integer: throw RangeError
nodes = all node references in traversal order
ordered = shallow copy of nodes
for each start where start + k <= length:
    reverse ordered[start .. start + k)
for each index in ordered:
    ordered[index].next = ordered[index + 1] or null
return ordered[0] or null
```

#### Brute-Force TypeScript
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function reverseKGroupBruteForce<T>(
  head: ListNode<T> | null,
  k: number,
): ListNode<T> | null {
  if (!Number.isInteger(k) || k <= 0) {
    throw new RangeError("k must be a positive integer");
  }

  const nodes: ListNode<T>[] = [];
  let current = head;

  while (current !== null) {
    nodes.push(current);
    current = current.next;
  }

  const orderedNodes = nodes.slice();
  for (let start = 0; start + k <= orderedNodes.length; start += k) {
    let left = start;
    let right = start + k - 1;
    while (left < right) {
      [orderedNodes[left], orderedNodes[right]] = [
        orderedNodes[right],
        orderedNodes[left],
      ];
      left += 1;
      right -= 1;
    }
  }

  for (let index = 0; index < orderedNodes.length; index += 1) {
    orderedNodes[index].next = orderedNodes[index + 1] ?? null;
  }

  return orderedNodes[0] ?? null;
}
```

#### Visualization
```text
collected identities: [A, B, C, D, E]
k = 2
logical order:         [B, A, D, C, E]
rewired chain:          B -> A -> D -> C -> E -> null
```

#### Dry Run
For `A -> B -> C -> D -> E` with `k = 2`, collection records five identities. The complete slices `[A, B]` and `[C, D]` become `[B, A]` and `[D, C]`; `[E]` is not a complete slice. Rewiring follows `[B, A, D, C, E]`, and every returned node is one of the original five objects.

#### Brute-Force Complexity
**Time: `O(n)`** because collection, slice reversal across disjoint groups, and rewiring each visit `O(n)` references. **Space: `O(n)` auxiliary** for the two reference arrays. No data node is allocated.

### 8. Why Brute Force Is Not Ideal
The arrays make group boundaries easy to see but retain every node reference. Direct pointer rewiring needs only a dummy predecessor and a constant number of cursors while preserving the same linear running time.

### 9. Optimal Approach
Validate `k` before traversal or mutation. Create a link-only dummy whose `next` is `head`, and let `groupPrev` identify the predecessor of the next candidate group. Walk exactly `k` steps from `groupPrev` to locate `kth`. If `kth` does not exist, return immediately without changing the short suffix. Otherwise save `groupNext = kth.next`, reverse links from the group start until reaching `groupNext`, reconnect `groupPrev.next` to `kth`, and move `groupPrev` to the old group start.

### 10. Visualization
```text
before one group:
groupPrev -> A -> B -> C -> groupNext, k = 3

reverse only until groupNext:
groupPrev -> C -> B -> A -> groupNext
             ^              ^
          new start       fixed boundary

next iteration: groupPrev is A
```

### 11. Step-by-Step Dry Run
For `A -> B -> C -> D -> E` and `k = 2`:

| Phase | `groupPrev` | Located `kth` | `groupNext` | Chain after reconnect |
|---|---|---|---|---|
| start | dummy | `B` | `C` | `B -> A -> C -> D -> E` |
| second group | `A` | `D` | `E` | `B -> A -> D -> C -> E` |
| final check | `C` | absent | n/a | unchanged; return |

During the first reversal, `A.next` becomes `C` before `B.next` becomes `A`. Saving each node's original successor prevents loss of the unprocessed path.

### 12. TypeScript Solution
```typescript
class ListNode<T> {
  constructor(
    public value: T,
    public next: ListNode<T> | null = null,
  ) {}
}

function reverseKGroup<T>(
  head: ListNode<T> | null,
  k: number,
): ListNode<T> | null {
  if (!Number.isInteger(k) || k <= 0) {
    throw new RangeError("k must be a positive integer");
  }

  const dummy: { next: ListNode<T> | null } = { next: head };
  let groupPrev: { next: ListNode<T> | null } = dummy;

  while (true) {
    let kth: ListNode<T> | null = groupPrev.next;
    for (let step = 1; step < k && kth !== null; step += 1) {
      kth = kth.next;
    }

    if (kth === null) {
      return dummy.next;
    }

    const groupNext = kth.next;
    const oldGroupStart = groupPrev.next!;
    let previous = groupNext;
    let current: ListNode<T> | null = oldGroupStart;

    while (current !== groupNext) {
      const next: ListNode<T> | null = current!.next;
      current!.next = previous;
      previous = current;
      current = next;
    }

    groupPrev.next = kth;
    groupPrev = oldGroupStart;
  }
}
```

### 13. Code Walkthrough
The guard runs before the dummy is created or any input link is read for rewiring. `kth` starts at the first candidate node, so the loop advances only `k - 1` more times; this also makes `k = 1` work without a special branch. `groupNext` is captured before reversal. The inner loop uses that exact reference as its stopping sentinel, and `oldGroupStart` becomes the tail of the reversed group and the next `groupPrev`.

### 14. Complexity Analysis
**Time: `O(n)`** because each node is examined a constant number of times while locating and reversing complete groups. **Space: `O(1)` auxiliary** because the link-only dummy and pointers do not grow with input size. All data nodes are reused, and the final incomplete group is not mutated.

### 15. Edge Cases
- `null` returns `null` for every valid `k`.
- `k = 1` preserves the exact head and all links.
- `k` greater than the list length leaves the entire list unchanged.
- A final group shorter than `k` retains its original identities and order.
- A length exactly divisible by `k` reverses every node within a full group.
- Zero, negative, fractional, `NaN`, and infinite `k` values throw before mutation.

### 16. Common Mistakes
- Reversing a candidate suffix before proving that it contains `k` nodes.
- Losing `groupNext` or the current node's saved successor during rewiring.
- Swapping node values instead of preserving value-to-identity association.
- Returning the original head after the first group changes it.
- Moving `groupPrev` to the new group start instead of the old group start.
- Detecting an invalid `k` after links have already changed.

### 17. Interview Follow-ups
1. Reverse alternating groups of `k`. 2. Reverse groups whose sizes increase as `1, 2, 3, ...`. 3. Restore the original list after processing. 4. Accept a predicate that decides whether a complete group should be reversed.

### 18. Variations
- Swap adjacent pairs. - Reverse nodes between two positions. - Reverse only even-length groups. - Process a doubly linked list while maintaining both directions.

### 19. Pattern Connections
This combines the ordinary three-pointer reversal pattern with a fixed-size lookahead. The dummy generalizes head replacement, while the boundary sentinel resembles half-open array ranges: mutate `[groupStart, groupNext)` and never cross `groupNext`.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/reverse-k-group` with `{ values: unknown[], k: number }`. Validate `k` before constructing or mutating request-scoped nodes, assign stable node IDs, run only the optimized algorithm, and return `{ values, nodeIds, trace, truncated }`. Bound trace events such as `{ group, groupPrevId, kthId, groupNextId, phase }`, including an `incomplete-group` event that proves the suffix was left untouched. Do not return the link-only dummy or treat it as caller-owned data.

### 21. Mini Challenge
Implement `reverseAlternatingKGroups(head, k)`, reversing the first complete group, skipping the next complete group, and continuing alternately while preserving the same invalid-before-mutation guarantee.

## Problem 114 — Sort a Linked List

### 1. Problem Statement
Given the head of an acyclic singly linked list whose values are numeric safe integers, stably sort the list in nondecreasing order. Numeric safe-integer values and acyclicity are caller preconditions; validating values separately would cost `O(n)`. Mutate and reuse the existing nodes, return the new head, and leave `null` or a single node unchanged. Equal-valued nodes must retain their original relative order.

The optimized solution must allocate no replacement data nodes. A temporary dummy used during merging is allowed. External references to input nodes remain valid, but those observers see their `next` links rewired into sorted order.

### 2. Example
```text
input identities:  4(a) -> 2(b) -> 4(c) -> 1(d) -> null
sorted output:     1(d) -> 2(b) -> 4(a) -> 4(c) -> null

a remains before c because the sort is stable.
```

### 3. Understand the Problem
Sorting values alone is insufficient when identity matters. The algorithm must move existing nodes by changing links and preserve original order among ties. Linked lists do not support efficient random access, but they split with slow/fast pointers and merge by relinking front nodes, making merge sort a natural fit.

#### In Simple Terms
Reorder the same physical nodes from smallest value to largest, keeping equal nodes in their original order.

### 4. Real-World Analogy
Two already sorted lines of tickets can be combined by repeatedly admitting the smaller front ticket. When front values tie, admitting the left line first preserves arrival order. No ticket is copied; only the queue links change.

### 5. Key Observation
Top-down merge sort needs three operations that fit linked lists: find a midpoint, cut one chain into two shorter chains, and stably merge two sorted chains. Choosing the left node on `<=` is the exact rule that preserves stability across the split.

### 6. Pattern
```text
Pattern: Divide and Conquer / Linked-List Merge Sort
Why this pattern applies: sequential access supports linear split and merge operations
How to recognize it: stable O(n log n) sort, node reuse, no random access
Similar problems: merge two sorted lists, external merge sort, sort list fragments
```

### 7. Brute Force
Collect records containing each node and its original position. Sort the records by node value, breaking ties by original position, then rewire all nodes in sorted-record order. The explicit position makes stability independent of the runtime sort implementation.

#### Pseudocode
```text
records = empty array
traverse list, recording { node, originalOrder }
sort records by node.value, then originalOrder
rewire each record.node to the next record.node
return first node or null
```

#### Brute-Force TypeScript
```typescript
class ListNode {
  constructor(
    public value: number,
    public next: ListNode | null = null,
  ) {}
}

interface NodeRecord {
  node: ListNode;
  originalOrder: number;
}

function compareSafeIntegers(first: number, second: number): number {
  if (first < second) return -1;
  if (first > second) return 1;
  return 0;
}

function sortListBruteForce(head: ListNode | null): ListNode | null {
  const records: NodeRecord[] = [];
  let current = head;
  let originalOrder = 0;

  while (current !== null) {
    records.push({ node: current, originalOrder });
    originalOrder += 1;
    current = current.next;
  }

  records.sort((first, second) => {
    const valueOrder = compareSafeIntegers(first.node.value, second.node.value);
    return valueOrder !== 0 ? valueOrder : first.originalOrder - second.originalOrder;
  });

  for (let index = 0; index < records.length; index += 1) {
    records[index].node.next = records[index + 1]?.node ?? null;
  }

  return records[0]?.node ?? null;
}
```

#### Visualization
```text
records: [(4,a,0), (2,b,1), (4,c,2), (1,d,3)]
sorted:  [(1,d,3), (2,b,1), (4,a,0), (4,c,2)]
rewire:   d -> b -> a -> c -> null
```

#### Dry Run
Collection records identities and original positions before any rewiring. Sorting puts `d`, then `b`, then the equal-valued pair `a`, `c`; their position keys keep `a` before `c`. Rewiring produces the sorted chain using all four original nodes and terminates `c.next` with `null`.

#### Brute-Force Complexity
**Time: `O(n log n)`** for comparison sorting after an `O(n)` traversal and rewiring pass. **Space: `O(n)` auxiliary** for records and the sorting implementation's storage. All data-node identities are reused.

### 8. Why Brute Force Is Not Ideal
The record array provides indexing that the list itself does not need. Merge sort can split and merge directly through links, reducing auxiliary storage from linear to the logarithmic recursion stack.

### 9. Optimal Approach
Return immediately for zero or one node. For a longer list, use `slow = head` and `fast = head.next`; advancing `fast` from the second node ensures that a two-node list splits into one node on each side. Save `rightHead = slow.next`, set `slow.next = null`, recursively sort both halves, and stably merge them. During merge, choose the left node when `left.value <= right.value`, detach only through the advancing cursor, and append the remaining sorted suffix.

### 10. Visualization
```text
4(a) -> 2(b) -> 4(c) -> 1(d)
          split
[4(a) -> 2(b)]    [4(c) -> 1(d)]
   sort                 sort
[2(b) -> 4(a)]    [1(d) -> 4(c)]
          stable merge
1(d) -> 2(b) -> 4(a) -> 4(c) -> null
```

### 11. Step-by-Step Dry Run
Final merge of `2(b) -> 4(a)` and `1(d) -> 4(c)`:

| Step | Left front | Right front | Chosen | Reason | Merged prefix |
|---:|---|---|---|---|---|
| 1 | `2(b)` | `1(d)` | `1(d)` | `1 < 2` | `d` |
| 2 | `2(b)` | `4(c)` | `2(b)` | `2 < 4` | `d -> b` |
| 3 | `4(a)` | `4(c)` | `4(a)` | left wins `<=` tie | `d -> b -> a` |
| 4 | exhausted | `4(c)` | suffix `4(c)` | one side remains | `d -> b -> a -> c` |

The tie at step 3 demonstrates stability by identity, not merely equal output values.

### 12. TypeScript Solution
```typescript
class ListNode {
  constructor(
    public value: number,
    public next: ListNode | null = null,
  ) {}
}

function mergeSortedHalves(
  leftHead: ListNode | null,
  rightHead: ListNode | null,
): ListNode | null {
  const dummy = new ListNode(0);
  let tail = dummy;
  let left = leftHead;
  let right = rightHead;

  while (left !== null && right !== null) {
    if (left.value <= right.value) {
      const nextLeft = left.next;
      tail.next = left;
      tail = left;
      left = nextLeft;
    } else {
      const nextRight = right.next;
      tail.next = right;
      tail = right;
      right = nextRight;
    }
  }

  tail.next = left ?? right;
  return dummy.next;
}

function sortList(head: ListNode | null): ListNode | null {
  if (head === null || head.next === null) {
    return head;
  }

  let slow = head;
  let fast: ListNode | null = head.next;

  while (fast !== null && fast.next !== null) {
    slow = slow.next!;
    fast = fast.next.next;
  }

  const rightHead = slow.next;
  slow.next = null;

  const leftSorted = sortList(head);
  const rightSorted = sortList(rightHead);
  return mergeSortedHalves(leftSorted, rightSorted);
}
```

### 13. Code Walkthrough
The base case prevents unnecessary splitting. Starting `fast` at `head.next` leaves `slow` at the first node for a two-node list, so cutting `slow.next` strictly reduces both recursive inputs and guarantees termination. Each merge saves the chosen node's successor before relinking it. The `<=` branch selects the left node on equal values, preserving the relative order established by the original split.

### 14. Complexity Analysis
**Time: `O(n log n)`** because there are `O(log n)` split levels and each level merges all `n` nodes once. **Space: `O(log n)` auxiliary** for recursion; each merge uses `O(1)` local pointer storage and one temporary dummy. No replacement data nodes are created. A separate value-validation pass, if required by an API rather than assumed as a caller precondition, costs `O(n)` time.

### 15. Edge Cases
- `null` returns `null`.
- A single node returns the exact same node with its link unchanged.
- Two nodes split into two terminating one-node lists.
- Already sorted and reverse-sorted lists both remain `O(n log n)` in this implementation.
- Duplicate and negative safe integers sort correctly and stably.
- External node references remain valid but observe changed `next` links.

### 16. Common Mistakes
- Starting both slow and fast at `head` and producing a non-shrinking two-node left half.
- Forgetting `slow.next = null`, causing recursive overlap or nontermination.
- Choosing the right node on equal values and silently making the sort unstable.
- Copying values into new nodes instead of rewiring existing identities.
- Losing a successor before attaching the selected merge node.
- Claiming `O(1)` total auxiliary space while using top-down recursion.

### 17. Interview Follow-ups
1. Implement bottom-up merge sort for `O(1)` auxiliary space. 2. Validate safe-integer values before mutation. 3. Sort by a generic comparator. 4. Detect already ordered runs and reduce merging work.

### 18. Variations
- Sort a doubly linked list. - Partition around a pivot. - Merge natural sorted runs. - Sort while preserving immutable input nodes by allocating a fresh output.

### 19. Pattern Connections
The merge is Problem 107's sorted-list pattern with an explicit stable tie rule. Slow/fast midpoint discovery connects to Problem 102, while cutting the list illustrates why temporary mutation and ownership must be part of the contract.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/sort` with `{ values: number[] }`. Validate every value as a safe integer before constructing or mutating request-scoped nodes, assign stable node IDs, run only top-down merge sort, and return `{ values, nodeIds, trace, truncated }`. Bound trace events such as `{ phase: "split" | "take-left" | "take-right" | "append", depth, leftId, rightId, chosenId }`. The returned node-ID order makes stable ties and identity reuse observable without exposing temporary dummy nodes.

### 21. Mini Challenge
Implement stable bottom-up linked-list merge sort in `O(n log n)` time and `O(1)` auxiliary space, reusing every data node and correctly terminating each run before merging it.

## Problem 115 — Merge K Sorted Linked Lists

### 1. Problem Statement
Given a readonly array of heads for `k` acyclic singly linked lists, merge all nodes into one stably and deterministically sorted list. Every value is a numeric safe integer, every input list is already sorted in nondecreasing order, and all input node sets are pairwise disjoint; these are caller preconditions. Mutate and reuse every data node, do not mutate the heads array, and return the merged head or `null`.

For equal values, every node from a lower input-list index must appear before equal-valued nodes from a higher input-list index. Original order within each input list must remain unchanged. Let `N` be the total number of nodes.

### 2. Example
```text
list 0: 1(a) -> 4(b) -> 4(c) -> null
list 1: 1(d) -> 3(e) -> 4(f) -> null
list 2: 2(g) -> null

output: 1(a) -> 1(d) -> 2(g) -> 3(e) -> 4(b) -> 4(c) -> 4(f)

At value 4, list 0 identities b and c precede list 1 identity f.
```

### 3. Understand the Problem
The output owns no new data nodes: it is one rewired chain containing all original identities exactly once. Pairwise disjoint input node sets make that ownership transfer unambiguous. Stability has two levels: preserve order inside each list and use the original list index as the deterministic priority among lists when values tie.

#### In Simple Terms
Repeatedly take the smallest front node, breaking equal-value ties by the list's original array index.

### 4. Real-World Analogy
Several sorted checkout lines feed one lane. The smallest numbered ticket goes next. If tickets match, the lower-numbered source line has priority, while customers within one source line never overtake each other.

### 5. Key Observation
Only the current front node of each nonempty list can be the next output node, so the candidate set has at most `k` entries. A min heap finds the best candidate in `O(log k)`. After popping a node, reinserting its successor with the same `listIndex` ensures that all equal values from a lower-index list continue to outrank equal values from higher-index lists.

### 6. Pattern
```text
Pattern: K-Way Merge with a Stable Min Heap
Why this pattern applies: each input is sorted and contributes only one current candidate
How to recognize it: many sorted streams, O(N log k) target, deterministic ties
Similar problems: external merge, merge k arrays, smallest range across lists
```

### 7. Brute Force
Collect a record for every node containing its `listIndex` and position within that list. Sort records by value, then list index, then within-list order. Rewire records in sorted order. Read `heads` by index but never assign into it.

#### Pseudocode
```text
records = empty array
for each listIndex and head in readonly heads:
    traverse and collect { node, listIndex, orderWithinList }
sort by node.value, then listIndex, then orderWithinList
rewire every node in sorted record order
return first node or null
```

#### Brute-Force TypeScript
```typescript
class ListNode {
  constructor(
    public value: number,
    public next: ListNode | null = null,
  ) {}
}

interface MergeRecord {
  node: ListNode;
  listIndex: number;
  orderWithinList: number;
}

function compareSafeIntegers(first: number, second: number): number {
  if (first < second) return -1;
  if (first > second) return 1;
  return 0;
}

function mergeKListsBruteForce(
  heads: readonly (ListNode | null)[],
): ListNode | null {
  const records: MergeRecord[] = [];

  for (let listIndex = 0; listIndex < heads.length; listIndex += 1) {
    let current = heads[listIndex];
    let orderWithinList = 0;

    while (current !== null) {
      records.push({ node: current, listIndex, orderWithinList });
      orderWithinList += 1;
      current = current.next;
    }
  }

  records.sort((first, second) => {
    const valueOrder = compareSafeIntegers(first.node.value, second.node.value);
    if (valueOrder !== 0) return valueOrder;
    if (first.listIndex !== second.listIndex) {
      return first.listIndex - second.listIndex;
    }
    return first.orderWithinList - second.orderWithinList;
  });

  for (let index = 0; index < records.length; index += 1) {
    records[index].node.next = records[index + 1]?.node ?? null;
  }

  return records[0]?.node ?? null;
}
```

#### Visualization
```text
records before sort:
(1,a,L0,0) (4,b,L0,1) (4,c,L0,2)
(1,d,L1,0) (3,e,L1,1) (4,f,L1,2)
(2,g,L2,0)

sorted keys (value, listIndex, order):
a, d, g, e, b, c, f
```

#### Dry Run
Both value-1 records sort by list index, so `a` precedes `d`. At value 4, records `b` and `c` from list 0 retain their within-list order and both precede `f` from list 1. Rewiring links those exact identities and sets the final node's `next` to `null`; the readonly array entries are never changed.

#### Brute-Force Complexity
**Time: `O(N log N)`** for sorting all records, plus `O(N + k)` collection and rewiring. **Space: `O(N)` auxiliary** for records and sort storage. The output reuses all data nodes.

### 8. Why Brute Force Is Not Ideal
Global sorting ignores the useful fact that every input is already sorted. Keeping only one candidate from each list shrinks the comparison structure from `N` records to at most `k` heap entries and improves time to `O(N log k)`.

### 9. Optimal Approach
Create a custom min heap of entries `{ node, listIndex }`, ordered first by node value using `<` and `>` and then by list index. Push each nonnull head, so the heap contains at most one node per list. Repeatedly pop the minimum, save its successor before mutation, detach the popped node, append it to the output, and push the saved successor with the same list index. A link-only dummy simplifies output construction without becoming a data node.

### 10. Visualization
```text
initial heap fronts: (1,a,L0), (1,d,L1), (2,g,L2)
pop a; push (4,b,L0)
pop d; push (3,e,L1)
pop g
pop e; push (4,f,L1)

heap tie at value 4:
(4,b,L0) wins, then successor (4,c,L0) is reinserted
(4,c,L0) still wins before (4,f,L1)
```

### 11. Step-by-Step Dry Run
For the example lists:

| Step | Heap minimum | Saved successor | Appended IDs | Pushed entry |
|---:|---|---|---|---|
| 1 | `1(a), L0` | `4(b)` | `a` | `4(b), L0` |
| 2 | `1(d), L1` | `3(e)` | `a, d` | `3(e), L1` |
| 3 | `2(g), L2` | `null` | `a, d, g` | none |
| 4 | `3(e), L1` | `4(f)` | `a, d, g, e` | `4(f), L1` |
| 5 | `4(b), L0` | `4(c)` | `a, d, g, e, b` | `4(c), L0` |
| 6 | `4(c), L0` | `null` | `a, d, g, e, b, c` | none |
| 7 | `4(f), L1` | `null` | `a, d, g, e, b, c, f` | none |

Detaching each popped node makes the partial output terminate cleanly even while its former successor waits in the heap.

### 12. TypeScript Solution
```typescript
class ListNode {
  constructor(
    public value: number,
    public next: ListNode | null = null,
  ) {}
}

interface HeapEntry {
  node: ListNode;
  listIndex: number;
}

function compareEntries(first: HeapEntry, second: HeapEntry): number {
  if (first.node.value < second.node.value) return -1;
  if (first.node.value > second.node.value) return 1;
  return first.listIndex - second.listIndex;
}

class MinHeap {
  private readonly entries: HeapEntry[] = [];

  get size(): number {
    return this.entries.length;
  }

  push(entry: HeapEntry): void {
    this.entries.push(entry);
    let child = this.entries.length - 1;

    while (child > 0) {
      const parent = Math.floor((child - 1) / 2);
      if (compareEntries(this.entries[parent], this.entries[child]) <= 0) break;
      [this.entries[parent], this.entries[child]] = [
        this.entries[child],
        this.entries[parent],
      ];
      child = parent;
    }
  }

  pop(): HeapEntry | null {
    if (this.entries.length === 0) return null;

    const minimum = this.entries[0];
    const last = this.entries.pop()!;
    if (this.entries.length === 0) return minimum;

    this.entries[0] = last;
    let parent = 0;

    while (true) {
      const left = parent * 2 + 1;
      const right = left + 1;
      let smallest = parent;

      if (
        left < this.entries.length &&
        compareEntries(this.entries[left], this.entries[smallest]) < 0
      ) {
        smallest = left;
      }
      if (
        right < this.entries.length &&
        compareEntries(this.entries[right], this.entries[smallest]) < 0
      ) {
        smallest = right;
      }
      if (smallest === parent) break;

      [this.entries[parent], this.entries[smallest]] = [
        this.entries[smallest],
        this.entries[parent],
      ];
      parent = smallest;
    }

    return minimum;
  }
}

function mergeKLists(heads: readonly (ListNode | null)[]): ListNode | null {
  const heap = new MinHeap();

  for (let listIndex = 0; listIndex < heads.length; listIndex += 1) {
    const node = heads[listIndex];
    if (node !== null) {
      heap.push({ node, listIndex });
    }
  }

  const dummy: { next: ListNode | null } = { next: null };
  let tail = dummy;

  while (heap.size > 0) {
    const entry = heap.pop()!;
    const successor = entry.node.next;
    entry.node.next = null;
    tail.next = entry.node;
    tail = entry.node;

    if (successor !== null) {
      heap.push({ node: successor, listIndex: entry.listIndex });
    }
  }

  return dummy.next;
}
```

### 13. Code Walkthrough
The comparator avoids subtracting safe-integer node values, so even extreme values cannot produce an unsafe comparison result. List-index subtraction is safe because both indexes are bounded by the same realizable array length. Initialization reads but never writes `heads`. Saving `successor` before detaching prevents path loss. At most one entry per input list exists because a successor is pushed only after that list's current entry is popped.

### 14. Complexity Analysis
**Time: `O(N log k)`** because each of `N` nodes is pushed and popped once from a heap of size at most `k`; initialization is `O(k log k)` here and is bounded by the stated total when nonempty heads contribute to `N`, while `k` empty entries are scanned in `O(k)`. More precisely, total time is `O(k + N log k)`. **Space: `O(k)` auxiliary** for the heap. Output nodes are all reused, and only the link-only dummy is temporary.

### 15. Edge Cases
- `k = 0`, represented by an empty heads array, returns `null`.
- An array containing only `null` heads returns `null`.
- `k = 1` returns that list's nodes in their existing order while still preserving the readonly array entry.
- Empty lists may appear between nonempty lists without affecting original indexes.
- Equal values within one list preserve their original order.
- Equal values across lists are ordered by lower input-list index.

### 16. Common Mistakes
- Comparing only values and allowing heap shape to decide equal-value order.
- Using value subtraction for safe integers near opposite numeric limits.
- Pushing every node initially and using `O(N)` heap space.
- Overwriting `node.next` before saving its successor.
- Assigning merged heads back into the readonly input array.
- Accepting shared input nodes, which can duplicate identities or create cycles after rewiring.

### 17. Interview Follow-ups
1. Validate sortedness, safe integers, acyclicity, and disjoint ownership before mutation. 2. Merge asynchronous sorted streams. 3. Use pairwise divide-and-conquer merging. 4. Return source-list provenance alongside each output position.

### 18. Variations
- Merge `k` sorted arrays. - Find the smallest range covering every list. - Merge descending lists with a max heap. - Merge immutable lists into fresh nodes.

### 19. Pattern Connections
The heap generalizes Problem 107's two-list merge from two current fronts to `k` fronts. Its tie key turns a merely sorted result into a deterministic stable result, while pairwise-disjoint ownership prevents the intersection behavior from Problem 111 from corrupting a destructive merge.

### 20. NestJS Project Connection
Expose `POST /api/dsa/linked-list/merge-k` with `{ lists: number[][] }`. Validate every value as a safe integer and each array as nondecreasing before building pairwise-disjoint request-scoped lists with stable IDs. Run only the optimized heap merge and return `{ values, nodeIds, sourceListIndexes, trace, truncated }`. Bound trace events such as `{ phase: "pop" | "append" | "push-successor", nodeId, listIndex, heapSize }`; preserving source indexes in the trace makes equal-value stability testable.

### 21. Mini Challenge
Implement the pairwise divide-and-conquer alternative in `O(N log k)` time and `O(log k)` recursion space while preserving the same lower-input-index tie rule across every merge round.

# Topic Summary

## What You Learned
Problems 96–115 build a progression from representation to ownership-aware structural algorithms. You created and traversed lists; inserted, deleted, searched, and reversed nodes; used slow/fast pointers for middles and cycles; located and removed cycle entry links; merged and deduplicated lists; restored a palindrome after temporary reversal; compared identity for intersection; performed carry arithmetic with fresh output; and finished with grouped reversal, stable merge sort, and heap-based k-way merge.

The recurring distinction is value versus identity. Some operations allocate fresh nodes, while others promise to reuse caller-owned nodes and therefore must preserve identities even as links move. Acyclicity is often a caller precondition because ordinary traversal cannot safely validate a cycle without a separate strategy. Mutation contracts also determine when validation must finish, whether a dummy is caller-visible, and whether an algorithm must restore temporary changes.

## Patterns You Should Recognize
- **Linear traversal:** advance one cursor until `null` for construction, search, validation, and counting.
- **Dummy predecessor:** normalize head and middle rewiring while keeping the dummy outside returned data.
- **Save before mutation:** capture `next` before changing a link or the unprocessed suffix may be lost.
- **Slow/fast pointers:** find middles, detect cycles, and identify cycle entry with constant auxiliary space.
- **Length alignment or pointer switching:** synchronize unequal prefixes before identity comparison.
- **In-place reversal:** maintain previous, current, and saved-next references over a bounded region.
- **Sorted merge:** choose one front node at a time; make equality behavior explicit for stability.
- **Temporary mutation with restoration:** optimize space while guaranteeing the caller's original structure returns.
- **Carry state:** synchronize digit lists with a constant-size arithmetic state and fresh output ownership.
- **Fixed-group lookahead:** prove a complete region exists before mutating it.
- **Divide and conquer:** split linked lists and merge stable sorted halves.
- **Heap frontier:** keep one candidate per sorted input and encode deterministic tie rules in the comparator.

## Complexity Cheat Sheet
| Problem | Core operation | Time | Auxiliary space |
|---:|---|---|---|
| 96 | Create a singly linked list | `O(n)` | `O(1)` excluding output |
| 97 | Traverse a linked list | `O(n)` | `O(1)` excluding returned values |
| 98 | Insert a node | `O(n)` | `O(1)` |
| 99 | Delete a node | `O(n)` | `O(1)` |
| 100 | Search in a linked list | `O(n)` | `O(1)` |
| 101 | Reverse a linked list | `O(n)` | `O(1)` |
| 102 | Find middle | `O(n)` | `O(1)` |
| 103 | Nth node from end | `O(n)` | `O(1)` |
| 104 | Detect cycle | `O(n)` | `O(1)` |
| 105 | Find cycle start | `O(n)` | `O(1)` |
| 106 | Remove cycle | `O(n)` | `O(1)` |
| 107 | Merge two sorted lists | `O(n + m)` | `O(1)` |
| 108 | Deduplicate sorted list | `O(n)` | `O(1)` |
| 109 | Deduplicate unsorted list | `O(n)` expected | `O(n)` |
| 110 | Palindrome with restoration | `O(n)` | `O(1)` |
| 111 | Find identity intersection | `O(m + n)` | `O(1)` |
| 112 | Add reverse-order numbers | `O(m + n)` | `O(1)` excluding output |
| 113 | Reverse complete groups of `k` | `O(n)` | `O(1)` |
| 114 | Stable linked-list merge sort | `O(n log n)` | `O(log n)` |
| 115 | Merge `k` sorted lists | `O(k + N log k)` | `O(k)` |

## Common Interview Traps
- Comparing values when the contract asks whether two references are the same node.
- Rewiring `current.next` before saving the original successor.
- Validating an argument only after mutation has begun instead of throwing before mutation.
- Traversing until `null` when cyclic input is possible and no cycle guard exists.
- Returning a dummy node, serializing it, or treating it as caller-owned data.
- Temporarily reversing links for a palindrome check but failing to restore them on every outcome.
- Destructively merging inputs that share nodes even though ownership is not pairwise disjoint.
- Ignoring the digit range, order, base, or fresh-node contract in linked-list arithmetic.
- Reversing an incomplete final `k` group before discovering that it is short.
- Splitting merge sort so a two-node recursive input does not shrink.
- Omitting a heap tie key and accidentally producing unstable equal-value output.
- Claiming constant space while using recursion, an identity set, records, or a heap.

## Practice Checklist
- [ ] 96. Create a Singly Linked List
- [ ] 97. Traverse a Linked List
- [ ] 98. Insert a Node
- [ ] 99. Delete a Node
- [ ] 100. Search in a Linked List
- [ ] 101. Reverse a Linked List
- [ ] 102. Find Middle of Linked List
- [ ] 103. Nth Node from End
- [ ] 104. Detect Cycle
- [ ] 105. Find Starting Point of Cycle
- [ ] 106. Remove Cycle
- [ ] 107. Merge Two Sorted Linked Lists
- [ ] 108. Remove Duplicates from Sorted Linked List
- [ ] 109. Remove Duplicates from Unsorted Linked List
- [ ] 110. Check if Linked List is Palindrome
- [ ] 111. Intersection of Two Linked Lists
- [ ] 112. Add Two Numbers
- [ ] 113. Reverse Linked List in Groups of K
- [ ] 114. Sort a Linked List
- [ ] 115. Merge K Sorted Linked Lists