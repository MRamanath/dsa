# Binary Trees

## Learning Objectives

By the end of this topic, you should understand:

- how a generic `TreeNode<T>` represents one value and up to two child references
- how root, parent, child, leaf, subtree, depth, height, and width describe tree structure
- why node identity is different from stored-value equality
- which invariants define a proper finite binary tree
- how preorder, inorder, postorder, and level-order traversals determine output order
- how recursive DFS and explicit-stack DFS trade call-stack risk for owned auxiliary storage
- how a head-index queue works and why a level-frontier BFS can use `O(w)` auxiliary space
- why balanced and skewed shapes change height-sensitive space costs
- how structural recursion and postorder summaries solve tree problems
- how ownership rules, bounded traces, and serialization previews protect production APIs

## Prerequisites

- objects, references, generics, arrays, loops, recursion, and destructuring
- stacks and queues from `09-stack-queue.md`
- recursion and call frames from `10-recursion-backtracking.md`
- Big-O notation and the distinction between auxiliary and output space
- TypeScript strict mode, `null`, readonly arrays, and `RangeError`

## Core Theory

### Generic node model

A binary-tree node stores one value and references to at most two children:

```text
TreeNode<T> {
  value: T
  left: TreeNode<T> | null
  right: TreeNode<T> | null
}
```

`T` is the value type. The node's shape is independent of whether values are numbers, strings, records, or another type. `null` means that a child position is empty. An empty tree is represented by a `null` root, not by a node with an undefined value.

### Structural vocabulary

- The **root** is the only node with no parent.
- A **parent** directly references a **child** through `left` or `right`.
- A **leaf** has both child references equal to `null`.
- A **subtree** consists of a chosen node and every node reachable below it.
- A node's **depth** measures its distance from the root.
- A tree's **height** measures the longest downward path from its root to a leaf.

This chapter uses the **edge convention** unless stated otherwise: root depth is `0`, a leaf has height `0`, and an empty tree has height `-1`. Under the node-count convention, the same root-to-node path has one more unit: root depth is `1`, a leaf has height `1`, and an empty tree has height `0`. State the convention before comparing formulas.

### Identity versus value

Two different nodes may store equal values. They remain different nodes because their object identities and positions can differ. Comparing only `a.value === b.value` cannot prove that `a` and `b` are the same node or that their subtrees are equal.

Algorithms involving parents, lowest common ancestors, visited sets, or mutation often care about identity. Traversal output in this chapter returns values, so duplicate values are preserved in positional order rather than deduplicated.

### Proper finite binary-tree contract

The traversal functions assume the caller supplies a **proper finite binary tree**:

- following child references eventually stops at `null`
- no child path returns to an ancestor, so the structure is acyclic
- every non-root node has exactly one parent
- no node is shared as the child of multiple parents or through multiple child slots

A cycle would make a traversal nonterminating without cycle detection. A shared child would make the structure a directed acyclic graph, causing the same identity to be visited more than once. Validating arbitrary object graphs requires a visited set and is outside the traversal contracts here.

### Ownership and mutation

A builder owns the nodes it creates but does not own its caller's input array. Problem 146 reads a readonly representation, allocates fresh nodes, and never rewrites input slots. Traversals borrow the tree: they read `value`, `left`, and `right`, return fresh arrays, and do not relink or annotate nodes.

If an algorithm intentionally mutates a tree, its contract must say whether mutation is in place, whether removed subtrees remain reachable elsewhere, and who owns newly allocated nodes. Hidden mutation is especially risky because callers may retain references to any node, not only the root.

### Depth-first traversal orders

Depth-first search follows one branch before returning to a sibling. The location of the root visit defines the order:

- **preorder:** root, left, right
- **inorder:** left, root, right
- **postorder:** left, right, root

For a binary search tree, inorder yields sorted values only when the tree satisfies the search-order invariant. Inorder traversal itself does not sort an arbitrary binary tree.

Recursive DFS naturally uses the runtime call stack. For `n` nodes and edge-height `h`, every DFS visits each node once in `O(n)` time and keeps `O(h + 1)` active frames. This is `O(log n)` for a height-balanced tree and `O(n)` for a skewed tree.

### Breadth-first traversal

Breadth-first search visits nodes level by level. A queue preserves discovery order. A general array queue should advance a head index instead of calling `shift()`, because repeated shifting moves remaining elements. When grouped level output is required, two frontier arrays are another precise option: read `currentLevel`, append its children to `nextLevel`, then replace the frontier.

If `w` is maximum tree width, frontier BFS uses `O(w)` auxiliary node references. A single append-only queue with a head index takes `O(n)` retained array slots unless it clears or compacts consumed storage, even when only `O(w)` entries are logically live.

### Structural recursion

Trees are recursive structures, so many solutions share one skeleton:

```text
solve(node):
    if node is null: return base summary
    leftSummary = solve(node.left)
    rightSummary = solve(node.right)
    return combine(node.value, leftSummary, rightSummary)
```

The base summary must be an identity appropriate to the problem. Height may use `-1` for an empty subtree under the edge convention; subtree size uses `0`; a validity check may use a record containing several fields.

### Postorder summaries

When a parent's answer depends on both children, compute child summaries before the parent. This is postorder even when the function does not emit a traversal array. Height, subtree size, balance checking, diameter, and many dynamic-programming tree problems follow this pattern.

A useful summary should contain all information the parent needs and no whole-subtree rescans. Returning height and balance together, for example, avoids recomputing height separately at every node.

### Bounded traces and serialization previews

Tracing every visit can produce a response proportional to the tree, and snapshotting a growing output at every step can make copying quadratic. Production-facing educational endpoints should trace only the optimized path, cap events such as at 200, and return a `truncated` flag. A compact event can contain `{ phase, value, depth }` or `{ level, frontierSize }`.

Do not blindly serialize caller-provided node graphs. Cycles throw during ordinary JSON serialization, shared references lose identity information, and large trees create oversized payloads. Produce a bounded preview instead: visit at most a fixed number of nodes, encode missing children explicitly when shape matters, convert values through an approved serializer, and report whether nodes were omitted. Validate the proper-tree contract separately when untrusted graphs are accepted.

## Visual Mental Model

Use this tree throughout the traversal examples:

```text
          1
        /   \
       2     3
        \
         4

root: 1
leaves: 4, 3
subtree rooted at 2: nodes 2 and 4
edge-depth of 4: 2
edge-height of the tree: 2
```

The three DFS orders move the root visit:

```text
preorder   root-left-right  -> 1, 2, 4, 3
inorder    left-root-right  -> 2, 4, 1, 3
postorder  left-right-root  -> 4, 2, 3, 1
level order by depth        -> [1], [2, 3], [4]
```

A balanced tree keeps paths short; a skewed tree behaves like a linked list:

```text
balanced-ish             right-skewed

      A                       A
    /   \                      \
   B     C                      B
  / \   / \                      \
 D   E F   G                      C
                                     \
                                      D
```

Both can contain the same number of nodes, but the first has logarithmic height while the second has linear height.

## Complexity Concepts

### Nodes, height, and width

Let `n` be node count, `h` be edge-height, and `w` be maximum number of nodes on one level. For a nonempty proper binary tree, `0 <= h <= n - 1` and `1 <= w <= n`. Balanced shape makes `h = O(log n)`; a one-sided chain makes `h = n - 1`.

### Traversal time

A complete traversal performs constant local work per node, so it takes `O(n)` time. The visitation order changes semantics, not the node count. Returning all values also imposes `O(n)` output space.

### DFS auxiliary space

Recursive DFS uses one active frame per node on the current root-to-node path: `O(h + 1)`, conventionally written `O(h)` for nontrivial trees. An explicit DFS stack has the same asymptotic bound for these binary-tree traversals. It moves storage out of the runtime call stack and avoids recursion-depth failure, but it does not make the traversal constant-space.

### BFS auxiliary space

BFS stores a frontier rather than a root-to-leaf path. A two-frontier level traversal uses `O(w)` auxiliary references, with at most two adjacent levels represented during transition. The returned `T[][]` is separate `O(n)` output space.

### Output and instrumentation space

Traversal results are required output, not auxiliary workspace. A tree built for return contains `k` output nodes. Temporary node-index arrays, stacks, queues, validation sets, traces, and previews are auxiliary or instrumentation space and should be reported separately.

### Repeated-height trap

Computing tree height once is `O(n)`. Computing height again for many subtrees can become `O(n^2)` on a skewed tree. Combine related facts in one postorder summary when a parent needs them repeatedly.

## Common Patterns

### Recursive DFS template

```text
visit(node):
    if node is null: return
    choose the required position for process(node)
    visit(node.left)
    visit(node.right)
```

### Iterative preorder

```text
push root
while stack is not empty:
    pop node and emit it
    push right, then left
```

Right is pushed first because the LIFO stack must expose left first.

### Iterative inorder

```text
current = root
while current exists or stack is not empty:
    push the entire left chain
    pop and emit one node
    move to its right child
```

### Iterative postorder with expanded frames

```text
push { root, expanded: false }
when an unexpanded node is popped:
    push its expanded frame, right child, then left child
when an expanded node is popped:
    emit it
```

### Level frontiers

```text
current = [root]
while current is not empty:
    emit values from current
    build next from each current node's children
    current = next
```

### Postorder summary

```text
left = summarize(node.left)
right = summarize(node.right)
return combine node with left and right
```

# Problems

## Problem 146 — Implement Binary Tree

### 1. Problem Statement
Implement a generic binary tree builder from a readonly complete-slot level-order representation `readonly Array<T | null>`. Index `i` has children at `2 * i + 1` and `2 * i + 2`; `null` reserves a missing slot. Every non-null index after the root must have a non-null parent slot, otherwise throw `RangeError` before allocating any node. An empty array or `[null]` returns `null`; later non-null entries under a null root are invalid. Trailing null slots are allowed. Return fresh nodes and do not mutate the input.

### 2. Example
```text
Input: [1, 2, 3, null, 4]

Indexes:       0  1  2     3  4
Slots:        [1, 2, 3, null, 4]

Output tree:
       1
      / \
     2   3
      \
       4

Input:  [1, null, 2, 7]
Output: RangeError because index 3 has missing parent index 1
```

### 3. Understand the Problem
This is complete-slot indexing, not the compact queue encoding used by some interview platforms. A null slot still occupies its index, so its descendant indexes cannot contain nodes. Validation must scan the representation before allocation to make rejection atomic.

#### In Simple Terms
Treat the array like numbered parking spaces: a child may occupy its space only when its parent's space is occupied.

### 4. Real-World Analogy
Imagine an organization chart stored in fixed positions. Positions `2i + 1` and `2i + 2` report to position `i`. A person cannot appear in a reporting position when the manager position is empty.

### 5. Key Observation
For every non-null index `i > 0`, the only possible parent is `Math.floor((i - 1) / 2)`. Checking that slot is enough because the same rule is applied to every earlier non-null ancestor. Performing this scan first prevents partial tree allocation on invalid input.

### 6. Pattern
```text
Pattern: Validate Flat Representation, Then Materialize Structure
Why this pattern applies: array indexes encode every parent-child relationship
How to recognize it: output references can be derived from deterministic indexes
Similar problems: heap construction, array-encoded tries, parent-index forests
```

### 7. Brute Force
First validate every non-null slot's parent. Then recursively build the node at index `i`, followed by the nodes at its two child indexes. Null and out-of-range indexes return `null`.

#### Pseudocode
```text
for each non-null slot after index 0:
    if its parent slot is null: throw RangeError
build(index):
    if index is outside input or slot is null: return null
    return fresh node(value, build(left index), build(right index))
return build(0)
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function buildBinaryTreeRecursive<T>(
  values: readonly (T | null)[],
): TreeNode<T> | null {
  for (let index = 1; index < values.length; index += 1) {
    if (values[index] !== null) {
      const parentIndex = Math.floor((index - 1) / 2);
      if (values[parentIndex] === null) {
        throw new RangeError(`orphan node at index ${index}`);
      }
    }
  }

  const build = (index: number): TreeNode<T> | null => {
    if (index >= values.length) return null;
    const value = values[index];
    if (value === null) return null;

    return {
      value,
      left: build(2 * index + 1),
      right: build(2 * index + 2),
    };
  };

  return build(0);
}
```

#### Visualization
```text
build(0: 1)
+-- build(1: 2)
|   +-- build(3: null)
|   `-- build(4: 4)
`-- build(2: 3)
```

#### Dry Run
For `[1, 2, 3, null, 4]`, validation accepts indexes `1`, `2`, and `4`; index `4` has parent index `1`. `build(0)` allocates `1`, recursively creates subtrees rooted at `2` and `3`, and attaches `4` as the right child of `2`. The readonly array is unchanged.

#### Brute-Force Complexity
Let `n` be slot count, `k` be non-null node count, and `h` be output-tree edge-height. Validation costs `O(n)`. Construction visits reachable node and null boundaries, so total time remains `O(n)`. Recursive auxiliary stack is `O(h + 1)`, and the `k` fresh returned nodes use `O(k)` output space.

### 8. Why Brute Force Is Not Ideal
Recursion mirrors the structure cleanly, but a deeply indexed skewed shape can exhaust JavaScript's call stack. The iterative version avoids recursive frames. It is not asymptotically faster and uses a full slot array, so the improvement is operational stack safety rather than Big-O time.

### 9. Optimal Approach
Validate the entire representation first. Allocate a same-length array of node-or-null slots, creating one fresh node for each non-null input. Make a second pass over non-null nodes and link `left` and `right` from their computed child indexes. Return slot zero or `null`.

### 10. Visualization
```text
values: [1,    2,    3,    null, 4]
nodes:  [N1,   N2,   N3,   null, N4]
links:   N1 -> left N2, right N3
         N2 -> left null, right N4
```

### 11. Step-by-Step Dry Run
| Pass | Index | Slot | Action |
|---|---:|---:|---|
| validate | 1 | `2` | parent index `0` exists |
| validate | 4 | `4` | parent index `1` exists |
| allocate | 0 | `1` | create `N1` |
| allocate | 3 | `null` | keep null slot |
| link | 0 | `N1` | attach `N2` and `N3` |
| link | 1 | `N2` | attach null and `N4` |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function buildBinaryTree<T>(
  values: readonly (T | null)[],
): TreeNode<T> | null {
  for (let index = 1; index < values.length; index += 1) {
    if (values[index] !== null) {
      const parentIndex = Math.floor((index - 1) / 2);
      if (values[parentIndex] === null) {
        throw new RangeError(`orphan node at index ${index}`);
      }
    }
  }

  const nodes: Array<TreeNode<T> | null> = new Array(values.length).fill(null);
  for (let index = 0; index < values.length; index += 1) {
    const value = values[index];
    if (value !== null) {
      nodes[index] = { value, left: null, right: null };
    }
  }

  for (let index = 0; index < nodes.length; index += 1) {
    const node = nodes[index];
    if (node === null) continue;
    node.left = nodes[2 * index + 1] ?? null;
    node.right = nodes[2 * index + 2] ?? null;
  }

  return nodes[0] ?? null;
}
```

### 13. Code Walkthrough
The first pass rejects any orphan while the function has allocated nothing. The second pass maps occupied slots to fresh node identities. The third pass derives links from complete-slot indexes; `?? null` handles child indexes beyond the array. Empty input and an all-null root representation return `null`, while the readonly source remains untouched.

### 14. Complexity Analysis
Three linear passes take `O(n)` time. The node-slot array uses `O(n)` auxiliary space. The returned tree contains `k` fresh nodes and therefore uses `O(k)` output space. The iterative approach has no recursion stack, but it is not asymptotically faster than the recursive baseline.

### 15. Edge Cases
- `[]` and `[null]` return `null`.
- `[null, null, null]` returns `null`; trailing null slots are harmless.
- `[null, 1]` throws `RangeError` because the non-null child has a null parent.
- `[1, null, 2, 7]` throws because index `3` has null parent index `1`.
- Duplicate values create distinct node identities.
- A value type containing `null` is incompatible because `null` is reserved as the missing-slot marker.

### 16. Common Mistakes
- Silently treating the input as compact queue encoding.
- Allocating nodes before validating every orphan condition.
- Skipping null slots and shifting later indexes.
- Mutating the input array into node objects.
- Returning shared nodes across separate builder calls.
- Claiming the iterative version uses `O(1)` auxiliary space while ignoring its node-slot array.

### 17. Interview Follow-ups
1. Accept compact queue encoding instead. 2. Serialize a proper tree back to complete-slot form. 3. Trim trailing null slots canonically. 4. Validate a caller-provided object graph for cycles and shared children.

### 18. Variations
- Parent-index arrays. - Heap construction. - Sparse maps keyed by index. - Binary search tree insertion. - Immutable persistent trees.

### 19. Pattern Connections
Complete-slot indexing is the same arithmetic used by array heaps, but this problem allows missing slots subject to ancestry validity. The validate-then-build phases also demonstrate atomic validation: no externally relevant work begins until the representation contract is proven.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/build` with body `{ values: Array<unknown | null> }`. Validate array shape and a configured slot limit, then call only `buildBinaryTree`; map `RangeError` to HTTP `400`. Return a bounded complete-slot preview such as `{ slots, nodeCount, truncated }` rather than recursively serializing an unbounded tree. Cap optimized trace events at 200 and report `truncated` explicitly.

### 21. Mini Challenge
Write a serializer that converts a proper finite binary tree to complete-slot indexing, removes only trailing null slots, and rejects an output whose highest required index exceeds a configured limit. Preserve duplicate values and do not mutate the tree.

## Problem 147 — Preorder Traversal

### 1. Problem Statement
Given the root of a caller-owned proper finite generic binary tree, return a fresh `T[]` containing values in preorder: root, left subtree, right subtree. Return `[]` for `null`. Preserve duplicate values and do not mutate any node.

### 2. Example
```text
Input tree:
       1
      / \
     2   3
      \
       4

Output: [1, 2, 4, 3]
```

### 3. Understand the Problem
Preorder emits a node before either child. The proper-tree precondition guarantees termination and one visit per identity. The function returns stored values, not nodes, so equal values may appear multiple times.

#### In Simple Terms
Read the current node first, then fully explore left, then fully explore right.

### 4. Real-World Analogy
Think of outlining folders: write the current folder name before listing the contents of its left and right subfolders.

### 5. Key Observation
An explicit LIFO stack reverses push order. To process left before right, push the right child first and the left child second.

### 6. Pattern
```text
Pattern: Root-First Depth-First Search
Why this pattern applies: each parent must be emitted before descendants
How to recognize it: output begins with the subtree root
Similar problems: tree cloning, prefix expression output, directory listing
```

### 7. Brute Force
Use structural recursion. Emit the current value, recursively traverse the left child, then recursively traverse the right child.

#### Pseudocode
```text
visit(node):
    if node is null: return
    append node.value
    visit(node.left)
    visit(node.right)
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function preorderRecursive<T>(root: TreeNode<T> | null): T[] {
  const result: T[] = [];

  const visit = (node: TreeNode<T> | null): void => {
    if (node === null) return;
    result.push(node.value);
    visit(node.left);
    visit(node.right);
  };

  visit(root);
  return result;
}
```

#### Visualization
```text
visit 1 -> emit 1
  visit 2 -> emit 2
    visit 4 -> emit 4
  visit 3 -> emit 3
```

#### Dry Run
Starting at `1`, append `1`. Traverse the left subtree: append `2`, skip its null left child, then append `4`. Return to `1`, traverse right, and append `3`. The result is `[1, 2, 4, 3]`.

#### Brute-Force Complexity
Every node is visited once, so time is `O(n)`. Recursive frames use `O(h + 1)` auxiliary space, and the returned values use `O(n)` output space.

### 8. Why Brute Force Is Not Ideal
The recursive baseline is asymptotically optimal and often the clearest version. Its weakness is JavaScript's finite call stack: a valid but highly skewed user-controlled tree can overflow it. An explicit stack makes that storage visible and heap-backed.

### 9. Optimal Approach
Return early for an empty tree. Otherwise push the root. Repeatedly pop one node, append its value, then push its right child followed by its left child. This preserves root-left-right order without recursive calls.

### 10. Visualization
```text
Tree:       1
           / \
          2   3
           \
            4

pop 1; push 3, 2
pop 2; push 4
pop 4
pop 3
```

### 11. Step-by-Step Dry Run
| Step | Stack before pop, top right | Emitted | Result |
|---:|---|---:|---|
| 1 | `[1]` | `1` | `[1]` |
| 2 | `[3, 2]` | `2` | `[1, 2]` |
| 3 | `[3, 4]` | `4` | `[1, 2, 4]` |
| 4 | `[3]` | `3` | `[1, 2, 4, 3]` |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function preorder<T>(root: TreeNode<T> | null): T[] {
  if (root === null) return [];

  const result: T[] = [];
  const stack: TreeNode<T>[] = [root];

  while (stack.length > 0) {
    const node = stack.pop()!;
    result.push(node.value);
    if (node.right !== null) stack.push(node.right);
    if (node.left !== null) stack.push(node.left);
  }

  return result;
}
```

### 13. Code Walkthrough
The stack begins with the root. Each loop emits exactly one popped node. Right enters before left so left is popped next. No visited set is needed under the proper-tree precondition, and only the fresh result and local stack are mutated.

### 14. Complexity Analysis
Each of `n` nodes is pushed and popped once, giving `O(n)` time. For a proper binary tree, the explicit stack holds `O(h + 1)` nodes, or `O(h)` conventionally. The returned array requires `O(n)` output space.

### 15. Edge Cases
- `null` returns a fresh `[]`.
- A single node returns its one value.
- A left-only or right-only chain preserves root-to-leaf order.
- Duplicate values remain duplicated.
- Cyclic or shared-child graphs violate the caller precondition.

### 16. Common Mistakes
- Pushing left before right and accidentally traversing root-right-left.
- Returning node objects when the contract requires values.
- Using a visited set to hide an invalid-tree contract without documenting graph semantics.
- Mutating child links to mark visits.
- Ignoring stack-overflow risk in the recursive version.

### 17. Interview Follow-ups
1. Return node references instead of values. 2. Implement a lazy preorder generator. 3. Stop when a predicate matches. 4. Validate proper-tree identity while traversing.

### 18. Variations
- N-ary preorder. - Morris preorder with temporary threading. - Root-to-leaf path collection. - Tree cloning. - Prefix expression serialization.

### 19. Pattern Connections
Preorder is root-first structural work. It appears when parent context must be established before descendants, including copying nodes, propagating path state, and serializing a tree with explicit null markers.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/preorder` with a validated bounded tree representation, build it through Problem 146, and call only `preorder`. Return `{ values, trace, truncated }`. Cap trace events at 200 with compact records such as `{ phase: "pop", value, stackSize }`; do not include full stack snapshots or mutate the built tree for tracing.

### 21. Mini Challenge
Return preorder values lazily from a generator so a caller can stop early. Explain which references remain live when iteration pauses and how early termination affects auxiliary space.

## Problem 148 — Inorder Traversal

### 1. Problem Statement
Given the root of a caller-owned proper finite generic binary tree, return a fresh `T[]` containing values in inorder: left subtree, root, right subtree. Return `[]` for `null`, preserve duplicates, and do not mutate the tree. No binary-search-tree ordering is assumed.

### 2. Example
```text
Input tree:
       1
      / \
     2   3
      \
       4

Output: [2, 4, 1, 3]
```

### 3. Understand the Problem
Inorder delays each node until its entire left subtree has been visited, then emits the node before its right subtree. Sorted output is a property of a valid binary search tree, not of inorder alone.

#### In Simple Terms
Go as far left as possible, read the waiting node, then explore its right side.

### 4. Real-World Analogy
Imagine opening nested left folders before reading the label on the folder that contained them, then continuing into that folder's right branch.

### 5. Key Observation
The explicit stack stores ancestors whose left subtrees are being processed. Popping an ancestor means its left side is complete, so its value can be emitted and traversal can move right.

### 6. Pattern
```text
Pattern: Deferred-Root Depth-First Search
Why this pattern applies: root output waits for the left subtree
How to recognize it: process lies between left and right recursion
Similar problems: BST sorted traversal, kth-smallest BST value, expression trees
```

### 7. Brute Force
Recursively visit the left child, append the current value, and recursively visit the right child.

#### Pseudocode
```text
visit(node):
    if node is null: return
    visit(node.left)
    append node.value
    visit(node.right)
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function inorderRecursive<T>(root: TreeNode<T> | null): T[] {
  const result: T[] = [];

  const visit = (node: TreeNode<T> | null): void => {
    if (node === null) return;
    visit(node.left);
    result.push(node.value);
    visit(node.right);
  };

  visit(root);
  return result;
}
```

#### Visualization
```text
descend left from 1 to 2
emit 2
move right to 4; emit 4
return to 1; emit 1
move right to 3; emit 3
```

#### Dry Run
At `1`, recurse left to `2`. Its left is null, so append `2`; then recurse right and append `4`. Return to append `1`, then visit and append `3`. The final result is `[2, 4, 1, 3]`.

#### Brute-Force Complexity
The traversal visits each node once in `O(n)` time. It uses `O(h + 1)` recursive auxiliary space and `O(n)` output space.

### 8. Why Brute Force Is Not Ideal
Recursive inorder is already time-optimal and concise, but runtime recursion depth follows tree height. The iterative version preserves the same deferred-parent logic while avoiding dependence on the JavaScript call-stack limit.

### 9. Optimal Approach
Keep a `current` reference and an ancestor stack. Push `current` and move left until null. Pop one waiting node, append its value, and set `current` to its right child. Continue while either a current node or a waiting ancestor exists.

### 10. Visualization
```text
current 1: push 1, move left
current 2: push 2, move left
current null: pop 2, emit, move right to 4
current 4: push 4, move left
current null: pop 4, emit, move right
then pop 1, emit, and traverse 3
```

### 11. Step-by-Step Dry Run
| Step | Action | Stack, top right | Result |
|---:|---|---|---|
| 1 | push `1` | `[1]` | `[]` |
| 2 | push `2` | `[1, 2]` | `[]` |
| 3 | pop/emit `2` | `[1]` | `[2]` |
| 4 | push then emit `4` | `[1]` | `[2, 4]` |
| 5 | pop/emit `1` | `[]` | `[2, 4, 1]` |
| 6 | push then emit `3` | `[]` | `[2, 4, 1, 3]` |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function inorder<T>(root: TreeNode<T> | null): T[] {
  const result: T[] = [];
  const stack: TreeNode<T>[] = [];
  let current: TreeNode<T> | null = root;

  while (current !== null || stack.length > 0) {
    while (current !== null) {
      stack.push(current);
      current = current.left;
    }

    const node = stack.pop()!;
    result.push(node.value);
    current = node.right;
  }

  return result;
}
```

### 13. Code Walkthrough
The inner loop records a complete left chain. A pop occurs only after that node's left subtree is exhausted. Emitting at the pop places the root between its subtrees, and assigning the right child restarts the same process there. The tree itself is read-only by behavior.

### 14. Complexity Analysis
Each node is pushed once and popped once, so time is `O(n)`. The ancestor stack uses `O(h + 1)` auxiliary space, and the returned array uses `O(n)` output space.

### 15. Edge Cases
- `null` returns `[]`.
- A left-skewed tree returns leaf-to-root values.
- A right-skewed tree returns root-to-leaf values.
- Duplicate values are emitted independently by position.
- An arbitrary binary tree need not produce sorted output.

### 16. Common Mistakes
- Claiming inorder always sorts values.
- Emitting before descending left and producing preorder.
- Forgetting to move to the popped node's right child.
- Looping only while `current` exists and abandoning waiting ancestors.
- Rewiring pointers for constant space without restoring them.

### 17. Interview Follow-ups
1. Find the kth inorder value without producing the full array. 2. Implement a lazy iterator. 3. Traverse a BST in descending order. 4. Use Morris traversal and document temporary mutation.

### 18. Variations
- Reverse inorder. - BST range query. - Kth-smallest BST value. - Expression-tree infix output. - Morris inorder traversal.

### 19. Pattern Connections
The explicit stack simulates suspended recursive frames. This same left-chain pattern powers BST iterators, ordered range scans, and merge operations over two search trees.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/inorder` with a bounded complete-slot representation. Validate and build through Problem 146, then call only `inorder`. Return `{ values, trace, truncated }` with at most 200 events such as `{ phase: "push" | "emit", value, stackSize }`. Never market the result as sorted unless a separate BST validator succeeds.

### 21. Mini Challenge
Implement `kthInorder(root, k)` using the iterative pattern, rejecting nonpositive `k` and returning `undefined` when fewer than `k` nodes exist. Stop without traversing unnecessary right subtrees.

## Problem 149 — Postorder Traversal

### 1. Problem Statement
Given the root of a caller-owned proper finite generic binary tree, return a fresh `T[]` containing values in postorder: left subtree, right subtree, root. Return `[]` for `null`, preserve duplicates, and do not mutate nodes.

### 2. Example
```text
Input tree:
       1
      / \
     2   3
      \
       4

Output: [4, 2, 3, 1]
```

### 3. Understand the Problem
Postorder emits a node only after both child subtrees. This makes it the natural order for summaries and destructive cleanup because a parent is processed after everything it owns below.

#### In Simple Terms
Finish the left side, finish the right side, and read the parent last.

### 4. Real-World Analogy
When removing a folder tree, delete all nested folders first and delete the containing folder only after both branches are empty.

### 5. Key Observation
An iterative stack must remember whether a node is entering or returning. An `{ node, expanded }` frame makes that state explicit: an unexpanded frame schedules children; its later expanded frame emits the parent.

### 6. Pattern
```text
Pattern: Children-First Depth-First Search
Why this pattern applies: parent work depends on completed child work
How to recognize it: process occurs after both recursive calls
Similar problems: subtree height, directory deletion, expression evaluation
```

### 7. Brute Force
Recursively traverse left, recursively traverse right, then append the current value.

#### Pseudocode
```text
visit(node):
    if node is null: return
    visit(node.left)
    visit(node.right)
    append node.value
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function postorderRecursive<T>(root: TreeNode<T> | null): T[] {
  const result: T[] = [];

  const visit = (node: TreeNode<T> | null): void => {
    if (node === null) return;
    visit(node.left);
    visit(node.right);
    result.push(node.value);
  };

  visit(root);
  return result;
}
```

#### Visualization
```text
1 waits
  2 waits
    4 emits
  2 emits
  3 emits
1 emits
```

#### Dry Run
From `1`, recurse into `2`; its right child `4` emits before `2`. Then `3` emits. With both root subtrees complete, `1` emits last, producing `[4, 2, 3, 1]`.

#### Brute-Force Complexity
Each node is visited once, giving `O(n)` time. Active recursive frames use `O(h + 1)` auxiliary space, and the returned array uses `O(n)` output space.

### 8. Why Brute Force Is Not Ideal
The recursive baseline expresses postorder directly and is asymptotically optimal. Its runtime-stack depth can still fail for a highly skewed tree. Explicit frames retain the two-phase logic without recursive calls.

### 9. Optimal Approach
Push an unexpanded root frame. When an unexpanded node is popped, push its expanded frame first, then its right and left children as unexpanded frames. LIFO order processes left, then right, then returns to the expanded parent. Emit only expanded frames.

### 10. Visualization
```text
pop {1, false}
push {1, true}, {3, false}, {2, false}

2 expands before 3
4 emits before expanded 2
3 emits
expanded 1 emits last
```

### 11. Step-by-Step Dry Run
| Step | Popped frame | Action | Result |
|---:|---|---|---|
| 1 | `{1, false}` | schedule `1`, `3`, `2` | `[]` |
| 2 | `{2, false}` | schedule `2`, `4` | `[]` |
| 3 | `{4, false}` | schedule expanded `4` | `[]` |
| 4 | `{4, true}` | emit | `[4]` |
| 5 | `{2, true}` | emit | `[4, 2]` |
| 6 | `{3, true}` | emit | `[4, 2, 3]` |
| 7 | `{1, true}` | emit | `[4, 2, 3, 1]` |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

interface PostorderFrame<T> {
  node: TreeNode<T>;
  expanded: boolean;
}

function postorder<T>(root: TreeNode<T> | null): T[] {
  if (root === null) return [];

  const result: T[] = [];
  const stack: PostorderFrame<T>[] = [{ node: root, expanded: false }];

  while (stack.length > 0) {
    const frame = stack.pop()!;
    if (frame.expanded) {
      result.push(frame.node.value);
      continue;
    }

    stack.push({ node: frame.node, expanded: true });
    if (frame.node.right !== null) {
      stack.push({ node: frame.node.right, expanded: false });
    }
    if (frame.node.left !== null) {
      stack.push({ node: frame.node.left, expanded: false });
    }
  }

  return result;
}
```

### 13. Code Walkthrough
Each node first appears as unexpanded. Its expanded frame acts like the return point of a recursive call and stays below its child frames. Right is scheduled before left so left completes first. Each identity is emitted exactly once, and the algorithm changes only local arrays.

### 14. Complexity Analysis
Every node has one unexpanded and one expanded pop, a constant factor of work, so time is `O(n)`. At most a constant number of frames per active tree level is retained, giving `O(h + 1)` auxiliary space. Output space is `O(n)`.

### 15. Edge Cases
- `null` returns `[]`.
- A leaf emits immediately after its expanded frame returns.
- A skewed tree returns leaf-to-root order.
- Duplicate values remain separate output entries.
- Shared children or cycles violate the proper-tree precondition.

### 16. Common Mistakes
- Emitting an unexpanded frame and producing preorder.
- Pushing children before the expanded parent, causing the parent to emit too early.
- Scheduling left before right on a LIFO stack and reversing subtree order.
- Using reverse-preorder without counting its `O(n)` reversal or result buffer.
- Claiming expanded frames make space `O(n)` for every shape; the bound is `O(h)` here.

### 17. Interview Follow-ups
1. Implement postorder with two stacks. 2. Compute height instead of returning values. 3. Evaluate an arithmetic expression tree. 4. Detect whether a tree is height-balanced in one pass.

### 18. Variations
- Two-stack postorder. - Reverse modified preorder. - Subtree-size summaries. - Tree diameter. - Iterative deletion or cleanup.

### 19. Pattern Connections
Expanded frames are explicit continuation records: they store work that recursion would perform after child calls return. The same technique handles iterative expression evaluation and graph DFS with enter/exit events.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/postorder` with a bounded validated representation, build it through Problem 146, and call only `postorder`. Return `{ values, trace, truncated }` with at most 200 optimized events such as `{ phase: "expand" | "emit", value, stackSize }`. Avoid serializing frame nodes or entire partial outputs.

### 21. Mini Challenge
Change the frame result into a postorder summary that returns both subtree node count and edge-height for every node. Emit one bounded record per completed node without rescanning either subtree.

## Problem 150 — Level Order Traversal

### 1. Problem Statement
Given the root of a caller-owned proper finite generic binary tree, return a fresh `T[][]` containing one value array per depth from top to bottom and left to right. Return `[]` for `null`, preserve duplicate values, and do not mutate the tree.

### 2. Example
```text
Input tree:
       1
      / \
     2   3
      \
       4

Output: [[1], [2, 3], [4]]
```

### 3. Understand the Problem
Unlike DFS output, the result has explicit level boundaries. Nodes on each level follow left-to-right discovery order. The outer array length equals node-height: edge-height plus one for a nonempty tree.

#### In Simple Terms
Read the tree one row at a time, from the root downward.

### 4. Real-World Analogy
Think of calling people by organizational distance from a director: first the director, then all direct reports from left to right, then the next reporting layer.

### 5. Key Observation
A height-first baseline repeatedly searches for one target depth and revisits upper nodes. BFS avoids that repetition by retaining exactly the next frontier discovered from the current level.

### 6. Pattern
```text
Pattern: Breadth-First Search by Frontiers
Why this pattern applies: output is grouped by equal depth
How to recognize it: all nodes at depth d precede every node at depth d + 1
Similar problems: minimum unweighted distance, right-side view, zigzag levels
```

### 7. Brute Force
Recursively compute node-height, then for every target depth run another recursive collector from the root. The collector appends a value when its remaining depth reaches zero.

#### Pseudocode
```text
height(node): 0 for null, otherwise 1 + max(child heights)
collect(node, depth, level):
    if node is null: return
    if depth is 0: append value; return
    collect(left, depth - 1, level)
    collect(right, depth - 1, level)
for depth from 0 through height(root) - 1:
    collect(root, depth, fresh level)
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function levelOrderByHeight<T>(root: TreeNode<T> | null): T[][] {
  const height = (node: TreeNode<T> | null): number => {
    if (node === null) return 0;
    return 1 + Math.max(height(node.left), height(node.right));
  };

  const collect = (
    node: TreeNode<T> | null,
    depth: number,
    level: T[],
  ): void => {
    if (node === null) return;
    if (depth === 0) {
      level.push(node.value);
      return;
    }
    collect(node.left, depth - 1, level);
    collect(node.right, depth - 1, level);
  };

  const result: T[][] = [];
  const levelCount = height(root);
  for (let depth = 0; depth < levelCount; depth += 1) {
    const level: T[] = [];
    collect(root, depth, level);
    result.push(level);
  }
  return result;
}
```

#### Visualization
```text
target depth 0: visit 1                         -> [1]
target depth 1: revisit 1, visit 2 and 3       -> [2, 3]
target depth 2: revisit 1 and 2, visit 4       -> [4]
```

#### Dry Run
The height function returns three node-levels. The first collector call emits `1`; the second descends once and emits `2, 3`; the third descends twice and emits `4`. Upper nodes are revisited for later depths.

#### Brute-Force Complexity
Height costs `O(n)`. Repeated collection costs up to `O(nh)` time in general and `O(n^2)` on a skewed tree. The recursive stack uses `O(h + 1)` auxiliary space. The nested result contains `n` values and uses `O(n)` output space.

### 8. Why Brute Force Is Not Ideal
Repeated target-depth searches revisit ancestors and can become quadratic. BFS discovers every node once and naturally preserves levels. A naive array queue using `shift()` can reintroduce quadratic movement, while an append-only head-index queue retains `O(n)` slots unless compacted; level frontiers give the requested `O(w)` auxiliary bound directly.

### 9. Optimal Approach
For a nonempty tree, start `currentLevel` with the root. For each iteration, append current values to one output row while adding each node's left then right child to `nextLevel`. Replace the frontier and continue until it is empty.

### 10. Visualization
```text
current [1]    -> output [1]    -> next [2, 3]
current [2, 3] -> output [2, 3] -> next [4]
current [4]    -> output [4]    -> next []
```

### 11. Step-by-Step Dry Run
| Level | Current frontier | Output row | Next frontier |
|---:|---|---|---|
| 0 | `[1]` | `[1]` | `[2, 3]` |
| 1 | `[2, 3]` | `[2, 3]` | `[4]` |
| 2 | `[4]` | `[4]` | `[]` |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function levelOrder<T>(root: TreeNode<T> | null): T[][] {
  if (root === null) return [];

  const result: T[][] = [];
  let currentLevel: TreeNode<T>[] = [root];

  while (currentLevel.length > 0) {
    const values: T[] = [];
    const nextLevel: TreeNode<T>[] = [];

    for (const node of currentLevel) {
      values.push(node.value);
      if (node.left !== null) nextLevel.push(node.left);
      if (node.right !== null) nextLevel.push(node.right);
    }

    result.push(values);
    currentLevel = nextLevel;
  }

  return result;
}
```

### 13. Code Walkthrough
Each iteration owns one complete frontier and creates the next. Reading current nodes left to right and enqueuing each left child before its right child establishes stable level order. Once the assignment occurs, the previous frontier becomes unreachable locally. No tree links are changed.

### 14. Complexity Analysis
Every node appears in exactly one current frontier and is inspected once, so time is `O(n)`. Current and next frontiers coexist during transition but each is bounded by maximum width, giving `O(w)` auxiliary space. The returned nested arrays contain `n` values and use `O(n)` output space.

### 15. Edge Cases
- `null` returns a fresh `[]`.
- A single node returns `[[value]]`.
- A skewed tree returns one value per level and uses `O(1)` frontier width.
- A broad balanced tree may require `Theta(n)` frontier space at its widest level.
- Missing children do not create placeholders in traversal output.
- Duplicate values remain in left-to-right positional order.

### 16. Common Mistakes
- Flattening all values into `T[]` and losing level boundaries.
- Enqueuing right before left and reversing each level's discovery order.
- Calling `shift()` for every dequeue and claiming guaranteed linear time.
- Claiming an append-only head-index array uses `O(w)` physical space without clearing or compaction.
- Counting the required `T[][]` as auxiliary queue space.

### 17. Interview Follow-ups
1. Return a flat BFS sequence. 2. Produce zigzag level order. 3. Return the right-side view. 4. Stream one level at a time. 5. Compute maximum width.

### 18. Variations
- Zigzag traversal. - Bottom-up levels. - Average value per level. - Minimum depth. - Multi-source BFS on an implicit graph.

### 19. Pattern Connections
Level frontiers are a grouped form of queue-based BFS. The same boundary technique processes graph distances, infection waves, shortest unweighted paths, and all states reachable in exactly `d` steps.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/level-order` with a bounded complete-slot representation. Validate and build it through Problem 146, then call only `levelOrder`. Return `{ levels, trace, truncated }` and cap optimized events at 200 using summaries such as `{ level, frontierSize, nextSize }`. Avoid duplicating full level arrays in both trace and result.

### 21. Mini Challenge
Implement an async generator that yields one fresh level array at a time. Describe cancellation behavior, retained frontier references, and how you would enforce node and response limits in a streaming NestJS endpoint.

## Problem 151 — Maximum Depth

### 1. Problem Statement
Given the root of a caller-owned proper finite binary tree, return its maximum root-to-leaf depth measured in **number of nodes**. An empty tree has depth `0`, a root alone has depth `1`, and the function must not mutate the tree.

### 2. Example
```text
Input tree:
       1
      / \
     2   3
    /
   4

Output: 3
Explanation: the longest root-to-leaf path is 1 -> 2 -> 4, containing 3 nodes.
```

### 3. Understand the Problem
Only paths ending at leaves compete for the answer. This problem deliberately uses the node-count convention, not the chapter's default edge convention: `null` contributes `0`, and every nonempty path includes its root.

#### In Simple Terms
Find how many nodes are on the longest path from the root down to any leaf.

### 4. Real-World Analogy
Imagine an organization chart and count the most people encountered from the director through the deepest individual contributor, including both endpoints.

### 5. Key Observation
The deepest path through a node must continue through whichever child subtree is deeper. Therefore a subtree needs to return only one scalar depth rather than every complete path.

### 6. Pattern
```text
Pattern: Postorder Structural Recursion
Why this pattern applies: a node's answer depends on both child answers
How to recognize it: combine independent subtree summaries at their parent
Similar problems: tree height, diameter, balance checking, subtree size
```

### 7. Brute Force
Walk every root-to-leaf path recursively and append each leaf's current node-count depth to an array. Scan the collected depths afterward to find the maximum.

#### Pseudocode
```text
collect(node, currentDepth):
    if node is null: return
    nextDepth = currentDepth + 1
    if node is a leaf: append nextDepth; return
    collect(left, nextDepth)
    collect(right, nextDepth)
scan all collected leaf depths and return the largest, or 0 if none
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function maximumDepthByLeaves<T>(root: TreeNode<T> | null): number {
  const leafDepths: number[] = [];

  const collect = (node: TreeNode<T> | null, currentDepth: number): void => {
    if (node === null) return;

    const nextDepth = currentDepth + 1;
    if (node.left === null && node.right === null) {
      leafDepths.push(nextDepth);
      return;
    }

    collect(node.left, nextDepth);
    collect(node.right, nextDepth);
  };

  collect(root, 0);

  let maximum = 0;
  for (const depth of leafDepths) {
    maximum = Math.max(maximum, depth);
  }
  return maximum;
}
```

#### Visualization
```text
leaf path 1 -> 2 -> 4: collect 3
leaf path 1 -> 3:      collect 2
leaf depths: [3, 2] -> maximum 3
```

#### Dry Run
The walk reaches node `4` with `nextDepth = 3` and records `3`. It later reaches node `3` and records `2`. Scanning `[3, 2]` returns `3`.

#### Brute-Force Complexity
Every node is visited once, so time is `O(n)`. The recursion uses `O(h)` auxiliary stack space and the leaf-depth array uses `O(L)` output-like temporary space, where `L` is the number of leaves and `h` is node-height.

### 8. Why Brute Force Is Not Ideal
Collecting one number per leaf is unnecessary when only the maximum is returned. Structural recursion can combine child depths directly, retain no leaf array or BFS queue, and improve auxiliary storage even though both approaches remain `O(n)` time.

### 9. Optimal Approach
Return `0` for `null`. Recursively obtain the left and right subtree depths, keep the larger one, and add `1` for the current node.

### 10. Visualization
```text
depth(4) = 1 + max(0, 0) = 1
depth(2) = 1 + max(1, 0) = 2
depth(3) = 1 + max(0, 0) = 1
depth(1) = 1 + max(2, 1) = 3
```

### 11. Step-by-Step Dry Run
| Node | Left depth | Right depth | Returned depth |
|---:|---:|---:|---:|
| 4 | 0 | 0 | 1 |
| 2 | 1 | 0 | 2 |
| 3 | 0 | 0 | 1 |
| 1 | 2 | 1 | 3 |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function maximumDepth<T>(root: TreeNode<T> | null): number {
  if (root === null) return 0;
  return 1 + Math.max(maximumDepth(root.left), maximumDepth(root.right));
}
```

### 13. Code Walkthrough
The `null` base case establishes empty depth `0`. Each non-null call waits for both child depths, adds itself to the longer path, and returns one number. The function only reads child references and never mutates a node.

### 14. Complexity Analysis
Each node is evaluated once, giving `O(n)` time. Active recursive calls follow one root-to-leaf path, so auxiliary space is `O(h)`: `O(log n)` for a height-balanced tree and `O(n)` for a skewed tree.

### 15. Edge Cases
- `null` returns `0`.
- A root with no children returns `1`.
- A skewed tree with `n` nodes returns `n`.
- Missing children contribute `0` and do not terminate a path through the other child.
- Duplicate or arbitrary node values do not affect structural depth.
- The caller must provide a proper finite tree; cycles and shared children are outside the contract.

### 16. Common Mistakes
- Returning edge-height when the contract requires number of nodes.
- Returning `-1` for `null` by applying the chapter's default convention here.
- Adding both child depths instead of choosing the maximum.
- Treating a node with one missing child as a leaf.
- Claiming constant space while ignoring recursive call frames.

### 17. Interview Follow-ups
1. Return the deepest leaf. 2. Return one longest root-to-leaf path. 3. Compute depth iteratively. 4. Reject depths above a configured limit. 5. Compute diameter in the same postorder pass.

### 18. Variations
- Maximum edge-depth. - N-ary tree depth. - Weighted maximum root-to-leaf cost. - Maximum depth with parent pointers. - Deepest-level node sum.

### 19. Pattern Connections
This is the smallest postorder summary pattern: each subtree returns one scalar and its parent combines two independent results. Diameter and balance checks extend the summary with additional fields.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/maximum-depth`. Validate a bounded complete-slot representation and build a proper finite tree through Problem 146, then call only `maximumDepth`. Return `{ depth, unit: "nodes", trace, truncated }`, cap optimized postorder events at 200, and reject inputs whose configured node or depth limits would risk the service call stack.

### 21. Mini Challenge
Return both the maximum node-depth and one corresponding root-to-leaf value path. Preserve left-before-right tie-breaking without mutating the tree.

## Problem 152 — Minimum Depth

### 1. Problem Statement
Given the root of a caller-owned proper finite binary tree, return the minimum root-to-leaf depth measured in **number of nodes**. An empty tree has depth `0`, a root alone has depth `1`, and the function must not mutate the tree.

### 2. Example
```text
Input tree:
  1
   \
    2
     \
      3

Output: 3
Explanation: node 3 is the only leaf; a missing child of node 1 is not a leaf.
```

### 3. Understand the Problem
The answer must end at an actual leaf, whose left and right children are both `null`. Under this problem's node-count convention, `null` has depth `0`, a root leaf has depth `1`, and a one-child parent cannot use its missing side as a zero-length route.

#### In Simple Terms
Find the shortest path from the root to a real leaf, counting every node on that path.

### 4. Real-World Analogy
In an organization chart, find the fewest reporting layers needed to reach someone with no direct reports; an empty position beside a manager is not an employee.

### 5. Key Observation
BFS processes depths in increasing order. Therefore the first frontier containing a leaf proves the minimum without exploring any deeper frontier.

### 6. Pattern
```text
Pattern: Breadth-First Search with Early Exit
Why this pattern applies: the first goal reached at the shallowest level is optimal
How to recognize it: edges have equal cost and the objective is minimum depth
Similar problems: shortest unweighted path, nearest target, minimum transformations
```

### 7. Brute Force
Traverse every root-to-leaf path, collect each leaf's node-count depth, and scan the resulting array for its minimum.

#### Pseudocode
```text
collect(node, currentDepth):
    if node is null: return
    nextDepth = currentDepth + 1
    if both children are null: append nextDepth; return
    collect(left, nextDepth)
    collect(right, nextDepth)
return minimum collected depth, or 0 when the root is null
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function minimumDepthByLeaves<T>(root: TreeNode<T> | null): number {
  if (root === null) return 0;

  const leafDepths: number[] = [];
  const collect = (node: TreeNode<T>, depth: number): void => {
    const nextDepth = depth + 1;
    if (node.left === null && node.right === null) {
      leafDepths.push(nextDepth);
      return;
    }
    if (node.left !== null) collect(node.left, nextDepth);
    if (node.right !== null) collect(node.right, nextDepth);
  };

  collect(root, 0);

  let minimum = Number.POSITIVE_INFINITY;
  for (const depth of leafDepths) {
    minimum = Math.min(minimum, depth);
  }
  return minimum;
}
```

#### Visualization
```text
skewed path 1 -> 2 -> 3
node 1: right child exists, so node 1 is not a leaf
node 2: right child exists, so node 2 is not a leaf
node 3: both children null -> collect depth 3
```

#### Dry Run
The collector ignores each missing left child because its parent still has a right child. It reaches node `3`, records depth `3`, and the minimum scan returns `3`.

#### Brute-Force Complexity
The traversal and minimum scan take `O(n + L) = O(n)` time. The leaf-depth array uses `O(L)` temporary space and recursion uses `O(h)` auxiliary stack space, for `O(L + h)` total auxiliary space.

### 8. Why Brute Force Is Not Ideal
The baseline visits the whole tree and stores every leaf depth even when a shallow leaf already determines the answer. Level-order BFS can stop at the first leaf and needs only the current frontier.

### 9. Optimal Approach
Start the root at depth `1`. Inspect one complete frontier at a time; return the current depth immediately when any frontier node is a leaf. Otherwise place its existing children into the next frontier.

### 10. Visualization
```text
depth 1 frontier: [1] -> no leaf -> next [2]
depth 2 frontier: [2] -> no leaf -> next [3]
depth 3 frontier: [3] -> leaf -> return 3
```

### 11. Step-by-Step Dry Run
| Depth in nodes | Frontier | Leaf found? | Action |
|---:|---|---|---|
| 1 | `[1]` | No | enqueue `2` |
| 2 | `[2]` | No | enqueue `3` |
| 3 | `[3]` | Yes | return `3` |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function minimumDepth<T>(root: TreeNode<T> | null): number {
  if (root === null) return 0;

  let depth = 1;
  let currentLevel: TreeNode<T>[] = [root];

  while (currentLevel.length > 0) {
    const nextLevel: TreeNode<T>[] = [];
    for (const node of currentLevel) {
      if (node.left === null && node.right === null) return depth;
      if (node.left !== null) nextLevel.push(node.left);
      if (node.right !== null) nextLevel.push(node.right);
    }
    currentLevel = nextLevel;
    depth += 1;
  }

  throw new Error("A proper nonempty finite tree must contain a leaf");
}
```

### 13. Code Walkthrough
The frontier invariant is that every node in `currentLevel` has the same node-count depth. Both children of a processed node belong to the next depth. A return occurs only when both child references are `null`, so a node with one child is handled correctly. No links or values are changed.

### 14. Complexity Analysis
In the worst case BFS visits all `n` nodes, so time is `O(n)`. It can finish earlier after inspecting nodes through the first leaf level. Current and next frontiers use `O(w)` auxiliary space, where `w` is maximum width.

### 15. Edge Cases
- `null` returns `0`.
- A root alone returns `1`.
- A one-child chain returns its full node count, not `1`.
- A shallow leaf causes early return before deeper levels are built.
- A balanced tree's first leaf level determines the result.
- The caller must provide a proper finite tree; cycles and shared children are outside the contract.

### 16. Common Mistakes
- Computing `1 + min(leftDepth, rightDepth)` when one child is missing.
- Treating any node with one `null` child as a leaf.
- Measuring edges while examples count nodes.
- Using DFS while claiming guaranteed shallow-level early exit.
- Using repeated `shift()` operations and overlooking array movement costs.

### 17. Interview Follow-ups
1. Return the nearest leaf value. 2. Return its path. 3. Break equal-depth ties left to right. 4. Use an explicit compacting queue. 5. Generalize to an N-ary tree.

### 18. Variations
- Minimum edge-depth. - Nearest node satisfying a predicate. - Minimum weighted leaf cost. - Minimum depth in an N-ary tree. - Closest leaf from an arbitrary node.

### 19. Pattern Connections
Minimum depth is shortest-path search on an implicit unweighted graph directed from parent to children. The BFS early-exit proof is the same one used for maze distance and minimum transformation counts.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/minimum-depth`. Build a bounded proper tree through Problem 146, then call only `minimumDepth`. Return `{ depth, unit: "nodes", trace, truncated }` with at most 200 events such as `{ depth, frontierSize, leafFound }`; enforce node, width, and response limits before or during construction.

### 21. Mini Challenge
Return one shortest root-to-leaf path while retaining only `O(w)` frontier state plus the returned path. Explain the additional parent information needed and deterministic tie-breaking.

## Problem 153 — Count Nodes

### 1. Problem Statement
Given the root of a caller-owned proper finite binary tree, return its number of nodes as a JavaScript safe integer. Return `0` for `null`, throw `RangeError` before producing an unsafe count, and do not mutate the tree.

### 2. Example
```text
Input tree:
       8
      / \
     4   9
      \
       6

Output: 4
```

### 3. Understand the Problem
Every reachable node identity contributes exactly one, regardless of its value. The proper-tree precondition excludes cycles and shared nodes, so recursive subtree counts are disjoint and can be added safely after overflow checks.

#### In Simple Terms
Visit the whole tree and count each node once.

### 4. Real-World Analogy
Count every position in an organization chart: each person counts once, whether or not they manage anyone or share the same name.

### 5. Key Observation
For any non-null node, total size is one for the node plus the sizes of its disjoint left and right subtrees.

### 6. Pattern
```text
Pattern: Postorder Subtree Aggregation
Why this pattern applies: subtree size combines two child sizes and the current node
How to recognize it: the desired total is additive over disjoint subtrees
Similar problems: leaf count, subtree sums, tree serialization size
```

### 7. Brute Force
Use level-order frontiers and increment a running count for every visited node. Check before each increment so the function never returns an unsafe integer.

#### Pseudocode
```text
count = 0
frontier = [root] when root exists
while frontier is not empty:
    for each node in frontier:
        if count equals MAX_SAFE_INTEGER: throw RangeError
        increment count
        append existing children to next frontier
return count
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function countNodesByLevel<T>(root: TreeNode<T> | null): number {
  if (root === null) return 0;

  let count = 0;
  let currentLevel: TreeNode<T>[] = [root];

  while (currentLevel.length > 0) {
    const nextLevel: TreeNode<T>[] = [];
    for (const node of currentLevel) {
      if (count === Number.MAX_SAFE_INTEGER) {
        throw new RangeError("Node count exceeds Number.MAX_SAFE_INTEGER");
      }
      count += 1;
      if (node.left !== null) nextLevel.push(node.left);
      if (node.right !== null) nextLevel.push(node.right);
    }
    currentLevel = nextLevel;
  }

  return count;
}
```

#### Visualization
```text
frontier [8]:    count 0 -> 1, next [4, 9]
frontier [4, 9]: count 1 -> 3, next [6]
frontier [6]:    count 3 -> 4, next []
```

#### Dry Run
The root increments the count to `1`. Nodes `4` and `9` raise it to `3`, and node `6` raises it to `4`. Every increment is guarded before it occurs.

#### Brute-Force Complexity
The traversal takes `O(n)` time and level frontiers use `O(w)` auxiliary space. The numeric result itself uses `O(1)` space. A realizable JavaScript object tree cannot approach the safe-integer limit, but the guard states the arithmetic contract explicitly.

### 8. Why Brute Force Is Not Ideal
BFS stores a potentially broad frontier even though counting does not require level order. Postorder recursion keeps only the active root-to-node chain and directly combines subtree totals.

### 9. Optimal Approach
Return `0` for `null`. Recursively count both children, safely add their counts, then safely add one for the current node.

### 10. Visualization
```text
count(6) = safeAdd(safeAdd(0, 0), 1) = 1
count(4) = safeAdd(safeAdd(0, 1), 1) = 2
count(9) = 1
count(8) = safeAdd(safeAdd(2, 1), 1) = 4
```

### 11. Step-by-Step Dry Run
| Node | Left count | Right count | Returned count |
|---:|---:|---:|---:|
| 6 | 0 | 0 | 1 |
| 4 | 0 | 1 | 2 |
| 9 | 0 | 0 | 1 |
| 8 | 2 | 1 | 4 |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function countNodes<T>(root: TreeNode<T> | null): number {
  const safeAdd = (left: number, right: number): number => {
    if (left > Number.MAX_SAFE_INTEGER - right) {
      throw new RangeError("Node count exceeds Number.MAX_SAFE_INTEGER");
    }
    return left + right;
  };

  if (root === null) return 0;

  const leftCount = countNodes(root.left);
  const rightCount = countNodes(root.right);
  return safeAdd(safeAdd(leftCount, rightCount), 1);
}
```

### 13. Code Walkthrough
The recursive calls finish before arithmetic at their parent. All subtree counts are nonnegative safe integers. `safeAdd` checks the remaining safe-integer capacity before each addition, first for the two children and then for the current node. The traversal reads but never changes the tree.

### 14. Complexity Analysis
Each node contributes to one constant-time combination, so time is `O(n)`. The recursive stack uses `O(h)` auxiliary space: `O(log n)` when height-balanced and `O(n)` when skewed.

### 15. Edge Cases
- `null` returns `0`.
- A single node returns `1`.
- A skewed tree still counts every node exactly once.
- Duplicate values count as distinct node identities.
- A safe-integer overflow attempt throws `RangeError` before unsafe arithmetic.
- The caller must provide a proper finite tree; cycles or shared children violate the precondition.

### 16. Common Mistakes
- Counting only leaves or only non-null child links.
- Deduplicating equal values instead of counting identities.
- Traversing a shared graph as though it were a proper tree.
- Adding counts without stating JavaScript number safety.
- Claiming recursive auxiliary space is always `O(log n)`.

### 17. Interview Follow-ups
1. Count nodes iteratively with `bigint`. 2. Count only nodes at depth `k`. 3. Count nodes satisfying a predicate. 4. Exploit complete-tree structure. 5. Maintain subtree sizes under updates.

### 18. Variations
- Count internal nodes. - Count full nodes with two children. - Count nodes in a complete tree faster than linear time. - Count distinct values. - Count nodes in an N-ary tree.

### 19. Pattern Connections
Subtree size is an additive postorder summary and often becomes metadata for order-statistic trees, random node selection, rank queries, and balanced-tree maintenance.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/count-nodes`. Build only bounded proper trees through Problem 146, then call `countNodes`. Return `{ count, trace, truncated }`, cap optimized events at 200 with `{ phase: "combine", leftCount, rightCount }`, and configure a request node limit far below JavaScript's theoretical safe-integer boundary.

### 21. Mini Challenge
For a complete binary tree, implement a count that compares leftmost and rightmost subtree heights and skips perfect subtrees. State the time bound and safe-integer checks.

## Problem 154 — Count Leaf Nodes

### 1. Problem Statement
Given the root of a caller-owned proper finite binary tree, return the number of leaf nodes. A leaf has both `left` and `right` equal to `null`; return `0` for an empty tree, return `1` for a root alone, and do not mutate the tree.

### 2. Example
```text
Input tree:
       1
      / \
     2   3
      \
       4

Output: 2
Explanation: nodes 4 and 3 are leaves; node 2 has one child and is not a leaf.
```

### 3. Understand the Problem
Leaf status belongs to a node, not to a missing child slot. A one-child parent is internal because at least one path continues below it. Node identities count separately even when leaf values are equal.

#### In Simple Terms
Count nodes that have no children at all.

### 4. Real-World Analogy
In an organization chart, count people with no direct reports. A manager with one report is still a manager, not an endpoint.

### 5. Key Observation
A node contributes exactly one when both children are absent; otherwise its leaf count is the sum of the leaf counts in its existing subtrees.

### 6. Pattern
```text
Pattern: Recursive Classification and Aggregation
Why this pattern applies: classify the current node, then combine child totals
How to recognize it: local structure determines contribution to a global count
Similar problems: count internal nodes, count full nodes, predicate-based tree counts
```

### 7. Brute Force
Traverse the whole tree and collect references to every leaf in an array. Return the array length after traversal.

#### Pseudocode
```text
collect(node):
    if node is null: return
    if left and right are null: append node reference; return
    collect(left)
    collect(right)
return number of collected references
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function countLeafNodesByCollection<T>(root: TreeNode<T> | null): number {
  const leaves: TreeNode<T>[] = [];

  const collect = (node: TreeNode<T> | null): void => {
    if (node === null) return;
    if (node.left === null && node.right === null) {
      leaves.push(node);
      return;
    }
    collect(node.left);
    collect(node.right);
  };

  collect(root);
  return leaves.length;
}
```

#### Visualization
```text
node 1: has children -> continue
node 2: has right child -> continue
node 4: no children -> collect
node 3: no children -> collect
collected references: [node 4, node 3]
```

#### Dry Run
Node `2` is not collected because its right child exists. Node `4` is collected, then node `3` is collected. The fresh array has length `2`.

#### Brute-Force Complexity
Every node is visited once, giving `O(n)` time. Recursion uses `O(h)` auxiliary stack space and the reference array uses `O(L)` temporary space, for `O(h + L)` total auxiliary space.

### 8. Why Brute Force Is Not Ideal
Retaining every leaf reference is unnecessary when the caller needs only a count. A scalar postorder result eliminates the `O(L)` temporary array.

### 9. Optimal Approach
Return `0` for `null`, return `1` when the current node has no children, and otherwise safely add the recursive leaf counts of both subtrees.

### 10. Visualization
```text
leafCount(4) = 1
leafCount(2) = 0 + 1 = 1
leafCount(3) = 1
leafCount(1) = 1 + 1 = 2
```

### 11. Step-by-Step Dry Run
| Node | Classification | Left leaves | Right leaves | Return |
|---:|---|---:|---:|---:|
| 4 | leaf | - | - | 1 |
| 2 | internal | 0 | 1 | 1 |
| 3 | leaf | - | - | 1 |
| 1 | internal | 1 | 1 | 2 |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function countLeafNodes<T>(root: TreeNode<T> | null): number {
  if (root === null) return 0;
  if (root.left === null && root.right === null) return 1;

  const leftCount = countLeafNodes(root.left);
  const rightCount = countLeafNodes(root.right);
  if (leftCount > Number.MAX_SAFE_INTEGER - rightCount) {
    throw new RangeError("Leaf count exceeds Number.MAX_SAFE_INTEGER");
  }
  return leftCount + rightCount;
}
```

### 13. Code Walkthrough
The leaf base case stops before child calls and contributes one. Every internal node combines disjoint subtree counts. The safe-add guard keeps the result representable as an exact JavaScript integer, and no node field is changed.

### 14. Complexity Analysis
Every node is classified once, so time is `O(n)`. The only auxiliary storage is the recursive stack, which is `O(h)`: `O(log n)` for a height-balanced tree and `O(n)` for a skewed tree.

### 15. Edge Cases
- `null` returns `0`.
- A root alone returns `1`.
- A one-child parent is not a leaf.
- A skewed nonempty tree has exactly one leaf.
- Equal leaf values still represent separate leaves.
- The caller must supply a proper finite tree; cycles and shared children are outside the contract.

### 16. Common Mistakes
- Counting `null` child slots rather than leaf nodes.
- Treating a node with either child missing as a leaf.
- Returning `0` for a root with no children.
- Collecting values into a `Set` and losing duplicate-valued leaves.
- Ignoring recursive stack growth on skewed input.

### 17. Interview Follow-ups
1. Return all leaf values left to right. 2. Count leaves at a given depth. 3. Count only leaves matching a predicate. 4. Compare leaf sequences of two trees. 5. Count leaves iteratively.

### 18. Variations
- Count internal nodes. - Count full nodes. - Sum leaf values. - Remove leaves in rounds. - Find the nearest leaf.

### 19. Pattern Connections
The leaf test is a local predicate embedded in structural recursion. Replacing the predicate yields a general tree fold for counting nodes with any structural or value-based property.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/count-leaves`. Build a bounded proper tree through Problem 146, then call only `countLeafNodes`. Return `{ count, trace, truncated }`; cap optimized events at 200 using `{ phase: "classify", isLeaf }` without serializing arbitrary values or node references.

### 21. Mini Challenge
Return leaf counts grouped by node-count depth in one traversal. Keep output ordering deterministic and distinguish output space from recursive auxiliary space.

## Problem 155 — Sum of Nodes

### 1. Problem Statement
Given the root of a caller-owned proper finite binary tree whose values are JavaScript safe integers, return a safe-integer sum. Return `0` for `null`, validate each value during traversal, throw `RangeError` when a value or any algorithmic intermediate sum is unsafe, and do not mutate the tree.

### 2. Example
```text
Input tree:
       5
      / \
    -2   8
      \
       3

Output: 14
```

### 3. Understand the Problem
Negative values are allowed, but `NaN`, infinities, fractions, and integers outside the safe range are rejected. Addition order matters because cancellation can make a final mathematical total safe even though an earlier intermediate is unsafe. The baseline reduces values in BFS order; the optimized solution combines each left subtree, then right subtree, then its parent value, and each implementation throws on an unsafe intermediate in its stated order.

#### In Simple Terms
Add every node value, but stop with `RangeError` if any value or running partial sum cannot be represented exactly as a safe JavaScript integer.

### 4. Real-World Analogy
Add account balances only while every recorded balance and every subtotal remains within the ledger's exact numeric capacity; later credits cannot retroactively make an overflowed subtotal valid.

### 5. Key Observation
A subtree's sum is the checked sum of its left total, right total, and current value. A helper that validates both operands and the result makes every arithmetic boundary explicit.

### 6. Pattern
```text
Pattern: Postorder Numeric Aggregation with Checked Arithmetic
Why this pattern applies: each parent combines complete child sums with its own value
How to recognize it: totals are additive but numeric representation has a contract
Similar problems: subtree sums, path sums, weighted tree folds
```

### 7. Brute Force
Traverse the tree by BFS, validating each value and collecting it in level order. Then reduce the array from left to right with checked addition, so intermediate sums follow BFS order.

#### Pseudocode
```text
values = []
for each BFS frontier from root downward:
    validate every node value as a safe integer
    append values left to right
sum = 0
for value in values:
    sum = safeAdd(sum, value)
return sum
```

#### Brute-Force TypeScript
```typescript
interface TreeNode {
  value: number;
  left: TreeNode | null;
  right: TreeNode | null;
}

function sumNodesByLevel(root: TreeNode | null): number {
  const safeAdd = (left: number, right: number): number => {
    if (!Number.isSafeInteger(left) || !Number.isSafeInteger(right)) {
      throw new RangeError("Node values and sums must be safe integers");
    }
    const result = left + right;
    if (!Number.isSafeInteger(result)) {
      throw new RangeError("Node sum exceeds the safe-integer range");
    }
    return result;
  };

  const values: number[] = [];
  let currentLevel: TreeNode[] = root === null ? [] : [root];

  while (currentLevel.length > 0) {
    const nextLevel: TreeNode[] = [];
    for (const node of currentLevel) {
      if (!Number.isSafeInteger(node.value)) {
        throw new RangeError("Every node value must be a safe integer");
      }
      values.push(node.value);
      if (node.left !== null) nextLevel.push(node.left);
      if (node.right !== null) nextLevel.push(node.right);
    }
    currentLevel = nextLevel;
  }

  let sum = 0;
  for (const value of values) {
    sum = safeAdd(sum, value);
  }
  return sum;
}
```

#### Visualization
```text
BFS values: [5, -2, 8, 3]
0 + 5 = 5
5 + -2 = 3
3 + 8 = 11
11 + 3 = 14
```

#### Dry Run
The BFS validates and stores `5, -2, 8, 3`. The checked left-to-right reduction produces partial sums `5, 3, 11, 14`, all safe, so the result is `14`.

#### Brute-Force Complexity
Traversal and reduction each take `O(n)` time. The values array uses `O(n)` auxiliary space and BFS frontiers use `O(w)`, which is subsumed by `O(n)`.

### 8. Why Brute Force Is Not Ideal
The full values array duplicates information needed only once during addition. Postorder recursion combines scalar subtree sums directly and reduces auxiliary storage to the active tree height.

### 9. Optimal Approach
Validate a node's value on entry, recursively compute its left sum and then its right sum, call `safeAdd(leftSum, rightSum)`, and then call `safeAdd(thatResult, node.value)`. This fixed postorder evaluation throws even when the parent value could have canceled an unsafe child subtotal.

### 10. Visualization
```text
sum(3)  = safeAdd(safeAdd(0, 0), 3) = 3
sum(-2) = safeAdd(safeAdd(0, 3), -2) = 1
sum(8)  = safeAdd(safeAdd(0, 0), 8) = 8
sum(5)  = safeAdd(safeAdd(1, 8), 5) = 14
```

### 11. Step-by-Step Dry Run
| Node | Left sum | Right sum | Children subtotal | Return |
|---:|---:|---:|---:|---:|
| 3 | 0 | 0 | 0 | 3 |
| -2 | 0 | 3 | 3 | 1 |
| 8 | 0 | 0 | 0 | 8 |
| 5 | 1 | 8 | 9 | 14 |

### 12. TypeScript Solution
```typescript
interface TreeNode {
  value: number;
  left: TreeNode | null;
  right: TreeNode | null;
}

function sumNodes(root: TreeNode | null): number {
  const safeAdd = (left: number, right: number): number => {
    if (!Number.isSafeInteger(left) || !Number.isSafeInteger(right)) {
      throw new RangeError("Node values and sums must be safe integers");
    }
    const result = left + right;
    if (!Number.isSafeInteger(result)) {
      throw new RangeError("Node sum exceeds the safe-integer range");
    }
    return result;
  };

  if (root === null) return 0;
  if (!Number.isSafeInteger(root.value)) {
    throw new RangeError("Every node value must be a safe integer");
  }

  const leftSum = sumNodes(root.left);
  const rightSum = sumNodes(root.right);
  return safeAdd(safeAdd(leftSum, rightSum), root.value);
}
```

### 13. Code Walkthrough
Each call validates its current value before descending left and then right. Arithmetic occurs postorder: child totals are checked first, their subtotal is checked second, and adding the current value is checked last. An exception can therefore occur after part of the read-only traversal, but the tree remains unchanged.

### 14. Complexity Analysis
Every node is validated and combined once, giving `O(n)` time. Recursive call frames occupy `O(h)` auxiliary space: `O(log n)` for a height-balanced tree and `O(n)` for a skewed tree.

### 15. Edge Cases
- `null` returns `0`.
- A single safe-integer value returns that value.
- Zero and negative safe integers are valid.
- `NaN`, infinities, fractions, and unsafe integers throw `RangeError` when visited.
- A mathematically safe final total still throws if this algorithm's fixed intermediate subtotal is unsafe.
- The caller must provide a proper finite tree; cycles and shared children violate the precondition.

### 16. Common Mistakes
- Checking only the final total after unsafe arithmetic has already occurred.
- Accepting finite fractions even though the contract requires safe integers.
- Assuming cancellation makes every addition order equivalent.
- Mutating values into cached subtree sums.
- Claiming recursion is constant auxiliary space.

### 17. Interview Follow-ups
1. Return an exact `bigint` sum. 2. Sum only leaves. 3. Compute every subtree sum. 4. Detect a target subtree sum. 5. Replace recursion with checked iterative postorder.

### 18. Variations
- Sum leaf values. - Sum values at one depth. - Maximum root-to-leaf sum. - Path-sum existence. - Average value per level.

### 19. Pattern Connections
Checked tree summation is a numeric tree fold. Changing the identity and combine operation yields products, extrema, logical predicates, hashes, and richer postorder summaries.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/sum-nodes`. Validate the bounded representation structurally, then let `sumNodes` validate every numeric value during optimized traversal. Map `RangeError` to a documented `400` response, return `{ sum, trace, truncated }`, and cap postorder trace events at 200 without including untrusted node objects.

### 21. Mini Challenge
Implement an exact `bigint` version whose transport accepts and returns decimal strings. Define validation, serialization, and a node limit suitable for a NestJS endpoint.

## Problem 156 — Same Tree

### 1. Problem Statement
Given the roots of two caller-owned proper finite binary trees, return whether they have exactly the same shape and pairwise SameValueZero-equal values. Under SameValueZero, `NaN` equals `NaN`, `0` equals `-0`, and objects are equal only when they are the same reference. Two empty trees are equal, one empty and one nonempty tree are unequal, and neither tree may be mutated.

### 2. Example
```text
Tree A:       1          Tree B:       1
             / \                       / \
          NaN   3                   NaN   3

Output: true

Tree C:       1          Tree D:       1
             /                           \
            2                             2

Output: false because the shapes differ.
```

### 3. Understand the Problem
Equality includes every missing-child position, not just the values encountered by a traversal. SameValueZero differs from `Object.is` only for signed zero here: `Object.is(0, -0)` is false, while SameValueZero treats them as equal. Distinct objects with identical properties remain unequal because this contract uses identity rather than deep object comparison.

#### In Simple Terms
The trees are the same only when every corresponding position is missing in both trees or contains equal values and equal descendants.

### 4. Real-World Analogy
Compare two organization charts position by position. Matching employee identifiers are insufficient if one employee reports on the left in one chart and on the right in the other.

### 5. Key Observation
At a pair of positions, two `null` nodes match, exactly one `null` node fails, and two real nodes match only if their values match and both corresponding child pairs match.

### 6. Pattern
```text
Pattern: Paired Structural Recursion
Why this pattern applies: equality is decided by corresponding positions in two trees
How to recognize it: each recursive state contains one node from each input
Similar problems: symmetric tree, subtree matching, tree diffing
```

### 7. Brute Force
Serialize each tree independently in preorder, preserving every missing child with an explicit tagged null marker. Compare the resulting arrays position by position, using SameValueZero only for tagged values.

#### Pseudocode
```text
serialize(node):
    if node is null: append NULL marker; return
    append VALUE marker containing node.value
    serialize(node.left)
    serialize(node.right)
serialize both roots
if lengths differ: return false
compare marker kinds and SameValueZero values at every index
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type SerializedEntry<T> =
  | { kind: "null" }
  | { kind: "value"; value: T };

function sameValueZero<T>(left: T, right: T): boolean {
  return left === right || (left !== left && right !== right);
}

function sameTreeBySerialization<T>(
  first: TreeNode<T> | null,
  second: TreeNode<T> | null,
): boolean {
  const serialize = (root: TreeNode<T> | null): SerializedEntry<T>[] => {
    const result: SerializedEntry<T>[] = [];

    const visit = (node: TreeNode<T> | null): void => {
      if (node === null) {
        result.push({ kind: "null" });
        return;
      }
      result.push({ kind: "value", value: node.value });
      visit(node.left);
      visit(node.right);
    };

    visit(root);
    return result;
  };

  const firstSerialized = serialize(first);
  const secondSerialized = serialize(second);
  if (firstSerialized.length !== secondSerialized.length) return false;

  return firstSerialized.every((entry, index) => {
    const other = secondSerialized[index];
    if (entry.kind !== other.kind) return false;
    return entry.kind === "null" ||
      (other.kind === "value" && sameValueZero(entry.value, other.value));
  });
}
```

#### Visualization
```text
Tree A serialization: VALUE(1), VALUE(NaN), NULL, NULL, VALUE(3), NULL, NULL
Tree B serialization: VALUE(1), VALUE(NaN), NULL, NULL, VALUE(3), NULL, NULL
Comparison:           equal at every tagged position
```

#### Dry Run
Both roots emit `VALUE(1)`. Their left children emit `VALUE(NaN)`, which compare equal under SameValueZero, followed by two null markers. Their right subtrees also match, so the arrays compare equal.

#### Brute-Force Complexity
If the trees contain `n` and `m` nodes, serialization and comparison take `O(n + m)` time. The two arrays and recursive serialization stacks use `O(n + m)` auxiliary space overall.

### 8. Why Brute Force Is Not Ideal
Both complete serializations are retained even when the roots already differ. Direct paired recursion can stop at the first shape or value mismatch and stores only active call frames.

### 9. Optimal Approach
Compare one pair of positions at a time. Handle nullability first, compare the two values with SameValueZero, then recursively compare left with left and right with right. Boolean short-circuiting avoids visiting later pairs after a mismatch.

### 10. Visualization
```text
same(1, 1)
├─ values match
├─ same(NaN, NaN) -> true
└─ same(3, 3)     -> true
result: true
```

### 11. Step-by-Step Dry Run
| Pair | Shape check | Value check | Result so far |
|---|---|---|---|
| `1`, `1` | both nodes | equal | inspect children |
| `NaN`, `NaN` | both nodes | SameValueZero equal | both child pairs null |
| `3`, `3` | both nodes | equal | both child pairs null |
| all null pairs | both null | not needed | `true` |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function sameValueZero<T>(left: T, right: T): boolean {
  return left === right || (left !== left && right !== right);
}

function isSameTree<T>(
  first: TreeNode<T> | null,
  second: TreeNode<T> | null,
): boolean {
  if (first === null || second === null) return first === second;
  if (!sameValueZero(first.value, second.value)) return false;
  return isSameTree(first.left, second.left) &&
    isSameTree(first.right, second.right);
}
```

### 13. Code Walkthrough
The shared null check makes two null positions `true` and a mixed pair `false`. Real nodes fail immediately when their values differ. The left comparison short-circuits before the right comparison, and all operations are read-only.

### 14. Complexity Analysis
On equal trees with `n` nodes, every corresponding node pair is visited, so time is `O(n)`. On unequal trees, work is proportional to the compared prefix up to the first mismatch, bounded by the smaller explored input. The recursive stacks use `O(h1 + h2)` space in the general two-tree statement, with at most one active paired path.

### 15. Edge Cases
- Two `null` roots return `true`; exactly one `null` root returns `false`.
- `NaN` values compare equal, and `0` equals `-0`.
- Two object values compare equal only when they reference the same object.
- Duplicate values do not hide a shape mismatch.
- A deep mismatch short-circuits without mutation.
- Both inputs must be proper finite trees; cycles and shared children violate the precondition.

### 16. Common Mistakes
- Comparing traversal values without null markers and losing shape information.
- Using `===` alone, which makes `NaN` unequal to itself.
- Using `Object.is`, which incorrectly distinguishes `0` from `-0` for this contract.
- Deep-comparing object properties instead of using object identity.
- Continuing through both trees after a mismatch.

### 17. Interview Follow-ups
1. Compare trees iteratively. 2. Return the first mismatching path. 3. Support a caller-provided value comparator. 4. Test whether one tree is a subtree of another. 5. Compare DAGs with cycle detection.

### 18. Variations
- Flip-equivalent trees. - Approximately equal numeric trees. - Tree edit distance. - Merkle-hash comparison. - Equality ignoring selected metadata.

### 19. Pattern Connections
Paired structural recursion is a product traversal: each state advances through two recursive structures together. The same shape appears in mirror checks, synchronized tree transforms, and structural diffs.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/same`. Validate and build two bounded proper trees through Problem 146, then call only `isSameTree`. Because JSON cannot represent `NaN`, define a tagged transport value if that case is supported. Return `{ same, trace, truncated }`, cap pair-comparison events at 200, and never echo arbitrary object values in traces.

### 21. Mini Challenge
Return the first mismatch as a deterministic path such as `"LRR"`, distinguishing a shape mismatch from a SameValueZero value mismatch without mutating either tree.

## Problem 157 — Symmetric Tree

### 1. Problem Statement
Given the root of a caller-owned proper finite binary tree, return whether it is symmetric around its root. Mirrored positions must have SameValueZero-equal values: `NaN` equals `NaN`, `0` equals `-0`, and objects compare by identity. An empty tree is symmetric, duplicate values do not excuse a null-shape mismatch, and the tree must not be mutated.

### 2. Example
```text
          1
        /   \
       2     2
      / \   / \
     3   4 4   3

Output: true

          1
        /   \
       2     2
        \     \
         3     3

Output: false because the missing-child positions are not mirrors.
```

### 3. Understand the Problem
Symmetry compares the root's left subtree with the root's right subtree in reverse orientation. The outer children must pair together, as must the inner children. Equal level-order values alone are insufficient because null placement determines the mirror shape.

#### In Simple Terms
Fold the tree down its center: every node, value, and missing child on one side must land on a matching position on the other side.

### 4. Real-World Analogy
Two wings of a building are symmetric when rooms at equal distances from the central lobby match in opposite directions, including where no room exists.

### 5. Key Observation
Two subtrees mirror each other when their roots match, the first root's left subtree mirrors the second root's right subtree, and the first root's right subtree mirrors the second root's left subtree.

### 6. Pattern
```text
Pattern: Mirrored Paired Recursion
Why this pattern applies: corresponding positions advance in opposite directions
How to recognize it: compare outer children together and inner children together
Similar problems: same tree, foldable tree, mirror transformation
```

### 7. Brute Force
Serialize the left subtree in normal preorder and the right subtree in mirrored preorder. Preserve every missing child with a tagged null marker, then compare the arrays using SameValueZero for tagged values.

#### Pseudocode
```text
serialize(left subtree, normal child order)
serialize(right subtree, right-before-left mirrored order)
include an explicit NULL marker for every missing position
compare lengths, marker kinds, and SameValueZero values
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type SerializedEntry<T> =
  | { kind: "null" }
  | { kind: "value"; value: T };

function sameValueZero<T>(left: T, right: T): boolean {
  return left === right || (left !== left && right !== right);
}

function symmetricBySerialization<T>(root: TreeNode<T> | null): boolean {
  if (root === null) return true;

  const serialize = (
    subtree: TreeNode<T> | null,
    mirrored: boolean,
  ): SerializedEntry<T>[] => {
    const result: SerializedEntry<T>[] = [];

    const visit = (node: TreeNode<T> | null): void => {
      if (node === null) {
        result.push({ kind: "null" });
        return;
      }
      result.push({ kind: "value", value: node.value });
      visit(mirrored ? node.right : node.left);
      visit(mirrored ? node.left : node.right);
    };

    visit(subtree);
    return result;
  };

  const leftSerialized = serialize(root.left, false);
  const rightSerialized = serialize(root.right, true);
  if (leftSerialized.length !== rightSerialized.length) return false;

  return leftSerialized.every((entry, index) => {
    const other = rightSerialized[index];
    if (entry.kind !== other.kind) return false;
    return entry.kind === "null" ||
      (other.kind === "value" && sameValueZero(entry.value, other.value));
  });
}
```

#### Visualization
```text
left normal:    VALUE(2), VALUE(3), NULL, NULL, VALUE(4), NULL, NULL
right mirrored: VALUE(2), VALUE(3), NULL, NULL, VALUE(4), NULL, NULL
result: equal serializations -> symmetric
```

#### Dry Run
The left serialization visits `2, 3, 4`. Mirrored serialization visits the right subtree as `2, 3, 4` by choosing right before left. Their tagged null positions also align, so comparison returns `true`.

#### Brute-Force Complexity
Every node and missing-child marker is processed a constant number of times, so time is `O(n)`. The two serializations and traversal stacks require `O(n)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
Materializing both subtree serializations retains the entire tree shape. Direct mirror recursion compares positions as they are reached, stops on the first mismatch, and needs only stack space.

### 9. Optimal Approach
For a nonempty root, recursively compare its left and right children as a mirror pair. Two null positions match, one null position fails, values use SameValueZero, and recursive calls cross outer and inner children.

### 10. Visualization
```text
mirror(2L, 2R)
├─ mirror(3 outer-left, 3 outer-right)
└─ mirror(4 inner-right, 4 inner-left)
result: true
```

### 11. Step-by-Step Dry Run
| Mirror pair | Value result | Next pairs |
|---|---|---|
| left `2`, right `2` | equal | outer `3/3`, inner `4/4` |
| outer `3`, `3` | equal | null/null twice |
| inner `4`, `4` | equal | null/null twice |
| all missing pairs | match | `true` |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function sameValueZero<T>(left: T, right: T): boolean {
  return left === right || (left !== left && right !== right);
}

function isSymmetric<T>(root: TreeNode<T> | null): boolean {
  const areMirrors = (
    left: TreeNode<T> | null,
    right: TreeNode<T> | null,
  ): boolean => {
    if (left === null || right === null) return left === right;
    if (!sameValueZero(left.value, right.value)) return false;
    return areMirrors(left.left, right.right) &&
      areMirrors(left.right, right.left);
  };

  return root === null || areMirrors(root.left, root.right);
}
```

### 13. Code Walkthrough
The empty-root expression returns `true`. For real mirror pairs, nullability and values are checked before recursion. The first recursive call compares outer positions and short-circuits before the inner pair on failure; no field is written.

### 14. Complexity Analysis
Each node participates in at most one mirror comparison, giving `O(n)` time in the full traversal and earlier termination on a mismatch. The recursive stack is `O(h)`, which becomes `O(n)` for a skewed tree.

### 15. Edge Cases
- `null` and a root alone are symmetric.
- `NaN` mirrors `NaN`, and `0` mirrors `-0`.
- Distinct object references do not match even if their properties look equal.
- Duplicate values still require mirrored null placement.
- A tree with only a left child at the root is not symmetric.
- The input must be a proper finite tree; cycles and shared children are outside the contract.

### 16. Common Mistakes
- Comparing left with left instead of crossing child directions.
- Comparing only values at each level and ignoring null positions.
- Using `===` without the `NaN` case.
- Treating duplicate values as evidence of symmetry.
- Building a mirrored copy and mutating the caller's tree.

### 17. Interview Follow-ups
1. Solve iteratively with a queue of node pairs. 2. Return the first asymmetric path pair. 3. Check structural symmetry while ignoring values. 4. Make an immutable mirrored copy. 5. Generalize symmetry to N-ary trees.

### 18. Variations
- Foldable trees. - Mirror equality of two roots. - Vertical-axis symmetry by coordinates. - Symmetric serialization. - Palindromic level views with null markers.

### 19. Pattern Connections
Symmetry is Same Tree with one traversal direction reversed. That small change from corresponding children to crossed children is the defining signal for mirrored paired recursion.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/symmetric`. Validate one bounded proper-tree payload, then call only `isSymmetric`. Use tagged transport values if `NaN` must be represented, return `{ symmetric, trace, truncated }`, and cap mirror-pair trace events at 200 without serializing arbitrary values.

### 21. Mini Challenge
Return the first asymmetric pair of root-relative paths, such as `{ left: "LR", right: "RL" }`, while preserving outer-before-inner comparison order.

## Problem 158 — Balanced Binary Tree

### 1. Problem Statement
Given the root of a caller-owned proper finite binary tree, return whether it is height-balanced. Every node must have `abs(leftHeight - rightHeight) <= 1`, where height is measured in nodes and an empty subtree has height `0`. An empty tree is balanced, values are irrelevant, and the tree must not be mutated.

### 2. Example
```text
Balanced:             Unbalanced:
       1                    1
      / \                  /
     2   3                2
    /                      /
   4                      3
                         /
                        4

Output: true          Output: false
```

### 3. Understand the Problem
Balance is a condition at every node, not only at the root. A tree can have root subtree heights differing by at most one while a deeper subtree is already unbalanced. Node-count height makes a leaf's height `1` and a missing child height `0`.

#### In Simple Terms
At every node, the left and right sides may differ in height by no more than one level.

### 4. Real-World Analogy
A mobile sculpture remains stable only when every crossbar is locally balanced. A balanced top crossbar cannot repair an unstable crossbar lower down.

### 5. Key Observation
A parent needs both child heights and both child balance results. Postorder traversal obtains those facts from the children before deciding the parent.

### 6. Pattern
```text
Pattern: Postorder Summary with Failure Sentinel
Why this pattern applies: a parent depends on completed child heights
How to recognize it: compute a metric upward and stop when a subtree is invalid
Similar problems: diameter, AVL validation, subtree metadata
```

### 7. Brute Force
At every node, independently recompute the heights of its left and right subtrees, check their difference, then recursively test both children. Repeated height traversals make this the baseline approach.

#### Pseudocode
```text
height(null) = 0
height(node) = 1 + max(height(left), height(right))
balanced(null) = true
balanced(node):
    if abs(height(left) - height(right)) > 1: return false
    return balanced(left) and balanced(right)
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function isBalancedByRecomputation<T>(root: TreeNode<T> | null): boolean {
  const height = (node: TreeNode<T> | null): number => {
    if (node === null) return 0;
    return 1 + Math.max(height(node.left), height(node.right));
  };

  if (root === null) return true;
  if (Math.abs(height(root.left) - height(root.right)) > 1) return false;
  return isBalancedByRecomputation(root.left) &&
    isBalancedByRecomputation(root.right);
}
```

#### Visualization
```text
At node 1: recompute left height 2 and right height 1
At node 2: recompute left height 1 and right height 0
At leaves: recompute two empty heights
The same lower nodes participate in several height calls.
```

#### Dry Run
For the balanced example, node `2` has child heights `1` and `0`, while root `1` has subtree heights `2` and `1`. Every difference is at most one, so recursive checks return `true`.

#### Brute-Force Complexity
For each node, height work is proportional to its subtree height and size. The total is `O(nh)`: `O(n log n)` on a height-balanced tree and `O(n^2)` on a skewed tree. Active recursive frames use `O(h)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
The height of the same subtree is recomputed for multiple ancestors. A single postorder traversal can return each height once and propagate imbalance immediately.

### 9. Optimal Approach
Use a postorder helper that returns a nonnegative node-count height for a balanced subtree or `-1` for an unbalanced subtree. Check the left result first, then the right, then the local difference. The sentinel is unambiguous because every valid height is at least `0`.

### 10. Visualization
```text
null -> 0
leaf -> 1
node 2: left 1, right 0 -> height 2
node 1: left 2, right 1 -> height 3
no call returns -1 -> balanced
```

### 11. Step-by-Step Dry Run
| Node | Left result | Right result | Decision | Return |
|---:|---:|---:|---|---:|
| 4 | 0 | 0 | difference 0 | 1 |
| 2 | 1 | 0 | difference 1 | 2 |
| 3 | 0 | 0 | difference 0 | 1 |
| 1 | 2 | 1 | difference 1 | 3 |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function isBalanced<T>(root: TreeNode<T> | null): boolean {
  const balancedHeight = (node: TreeNode<T> | null): number => {
    if (node === null) return 0;

    const leftHeight = balancedHeight(node.left);
    if (leftHeight === -1) return -1;

    const rightHeight = balancedHeight(node.right);
    if (rightHeight === -1) return -1;
    if (Math.abs(leftHeight - rightHeight) > 1) return -1;

    return 1 + Math.max(leftHeight, rightHeight);
  };

  return balancedHeight(root) !== -1;
}
```

### 13. Code Walkthrough
`null` contributes height zero. A `-1` from either child is returned immediately, so an invalid subtree cannot be mistaken for a height. Only nonnegative child heights reach the difference check and the parent-height calculation. The traversal reads the tree without changing it.

### 14. Complexity Analysis
Each node is processed once, so time is `O(n)`. The recursive stack is `O(h)`: `O(log n)` for a height-balanced tree and `O(n)` for a skewed tree. No per-node collection is retained.

### 15. Edge Cases
- `null` and a single node return `true`.
- A two-node tree is balanced.
- A chain of three nodes is unbalanced at its top node.
- A root-level height difference of zero does not guarantee deeper balance.
- `-1` is safe as a sentinel because legitimate node-count heights are nonnegative.
- Values, including duplicates and `NaN`, do not affect structural balance.

### 16. Common Mistakes
- Checking only the root's height difference.
- Measuring one subtree in edges and the other in nodes.
- Recomputing heights inside the optimized traversal.
- Continuing into the right subtree after the left returned `-1`.
- Using `0` as the failure sentinel even though empty height is `0`.

### 17. Interview Follow-ups
1. Return the first unbalanced node. 2. Compute balance iteratively. 3. Report each node's balance factor. 4. Validate AVL ordering and balance together. 5. Find the minimum nodes for a given balanced height.

### 18. Variations
- Weight-balanced trees. - Perfect-tree validation. - Complete-tree validation. - AVL validation. - Maximum imbalance factor.

### 19. Pattern Connections
The height-or-failure return type fuses computation and validation. This postorder sentinel pattern also detects invalid BST bounds, failed parses, impossible dynamic-programming states, and malformed recursive structures.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/balanced`. Enforce a bounded proper-tree representation before recursion, then call only `isBalanced`. Return `{ balanced, trace, truncated }`; cap postorder events at 200 with child heights or an imbalance phase, and omit arbitrary node values because they are irrelevant.

### 21. Mini Challenge
Return `{ balanced, height, firstUnbalancedPath }` in one traversal, choosing the first path in left-before-right postorder and keeping valid heights distinct from failure state.

## Problem 159 — Diameter of Binary Tree

### 1. Problem Statement
Given the root of a caller-owned proper finite binary tree, return its diameter: the number of edges on the longest path between any two nodes. The path need not pass through the root. An empty tree and a single-node tree both have diameter `0`, node values are irrelevant, and the tree must not be mutated.

### 2. Example
```text
        1
       /
      2
     / \
    3   4
       / \
      5   6

Output: 3
Explanation: 3 -> 2 -> 4 -> 5 is one longest path with 3 edges.
The longest path lies inside the root's left subtree.
```

### 3. Understand the Problem
Diameter counts edges, while the helper height convention counts nodes: `height(null) = 0` and `height(leaf) = 1`. With that convention, a path passing through a node contains `leftHeight + rightHeight` edges. The best path can be entirely inside either child subtree.

#### In Simple Terms
Find the greatest number of links on any path between two nodes anywhere in the tree.

### 4. Real-World Analogy
In a branching cable network, find the most cable segments on a route between two devices. The longest route may be contained within one branch below the main hub.

### 5. Key Observation
At each node, the longest path through that node joins the deepest left descendant to the deepest right descendant. A postorder traversal can compute child heights and update one global maximum.

### 6. Pattern
```text
Pattern: Postorder Height with Global Optimum
Why this pattern applies: each candidate uses two completed child summaries
How to recognize it: return one branch upward but score two branches locally
Similar problems: maximum path sum, tree cameras, longest univalue path
```

### 7. Brute Force
For every node, recompute its left and right heights to obtain the diameter passing through it, recursively compute the best diameter in each child, and return the maximum of all three candidates.

#### Pseudocode
```text
height(null) = 0
height(node) = 1 + max(height(left), height(right))
diameter(null) = 0
diameter(node):
    through = height(left) + height(right)
    return max(through, diameter(left), diameter(right))
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function diameterByRecomputation<T>(root: TreeNode<T> | null): number {
  const height = (node: TreeNode<T> | null): number => {
    if (node === null) return 0;
    return 1 + Math.max(height(node.left), height(node.right));
  };

  if (root === null) return 0;
  const throughRoot = height(root.left) + height(root.right);
  const leftDiameter = diameterByRecomputation(root.left);
  const rightDiameter = diameterByRecomputation(root.right);
  return Math.max(throughRoot, leftDiameter, rightDiameter);
}
```

#### Visualization
```text
At node 4: left height 1 + right height 1 = 2 edges
At node 2: left height 1 + right height 2 = 3 edges
At node 1: left height 3 + right height 0 = 3 edges
maximum candidate: 3
```

#### Dry Run
Leaves return diameter `0`. Node `4` finds a two-edge path from `5` to `6`. Node `2` combines height `1` through node `3` and height `2` through node `4`, producing a three-edge candidate. Root `1` also produces `3`, so the result remains `3`.

#### Brute-Force Complexity
Repeated height traversals give `O(nh)` time: `O(n log n)` on a height-balanced tree and `O(n^2)` on a skewed tree. The nested recursive work still has at most `O(h)` active stack frames at once.

### 8. Why Brute Force Is Not Ideal
Every ancestor recomputes heights already discovered below it. One postorder traversal can calculate each height once while evaluating every through-node diameter candidate.

### 9. Optimal Approach
Maintain a maximum edge count. A postorder helper returns node-count height; after receiving both child heights, compute `leftHeight + rightHeight`, update the maximum, and return one plus the larger child height.

### 10. Visualization
```text
node 3 -> heights 0,0; through 0; return 1
node 5 -> heights 0,0; through 0; return 1
node 6 -> heights 0,0; through 0; return 1
node 4 -> heights 1,1; through 2; return 2
node 2 -> heights 1,2; through 3; return 3
node 1 -> heights 3,0; through 3; return 4
maximum diameter = 3 edges
```

### 11. Step-by-Step Dry Run
| Node | Left height | Right height | Through edges | Maximum | Return height |
|---:|---:|---:|---:|---:|---:|
| 3 | 0 | 0 | 0 | 0 | 1 |
| 5 | 0 | 0 | 0 | 0 | 1 |
| 6 | 0 | 0 | 0 | 0 | 1 |
| 4 | 1 | 1 | 2 | 2 | 2 |
| 2 | 1 | 2 | 3 | 3 | 3 |
| 1 | 3 | 0 | 3 | 3 | 4 |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function diameterOfBinaryTree<T>(root: TreeNode<T> | null): number {
  let maximumDiameter = 0;

  const height = (node: TreeNode<T> | null): number => {
    if (node === null) return 0;

    const leftHeight = height(node.left);
    const rightHeight = height(node.right);
    maximumDiameter = Math.max(maximumDiameter, leftHeight + rightHeight);
    return 1 + Math.max(leftHeight, rightHeight);
  };

  height(root);
  return maximumDiameter;
}
```

### 13. Code Walkthrough
Each call first obtains both child heights. Their sum is exactly the through-node edge count under the node-count height convention. The closure retains the best candidate seen anywhere, while the helper returns only the taller downward branch needed by its parent. No node is modified.

### 14. Complexity Analysis
Every node contributes one constant-time candidate, so time is `O(n)`. The recursive stack uses `O(h)` auxiliary space: `O(log n)` for a height-balanced tree and `O(n)` for a skewed tree.

### 15. Edge Cases
- `null` returns `0`, and a single node returns `0`.
- Two connected nodes have diameter `1`.
- A chain of `n` nodes has diameter `n - 1`.
- The longest path may lie entirely below the root.
- Duplicate or unusual values do not affect the structural answer.
- JavaScript `number` is exact for practical in-memory tree heights and diameters; the API convention rejects theoretical representations requiring a result above `Number.MAX_SAFE_INTEGER` before building the tree.

### 16. Common Mistakes
- Counting nodes on the path instead of edges.
- Returning only `leftHeight + rightHeight` at the root.
- Returning the through-node diameter as the parent height.
- Recomputing child heights and losing linear time.
- Mutating nodes to cache heights without permission.

### 17. Interview Follow-ups
1. Return the diameter's endpoint nodes. 2. Return one longest path. 3. Handle weighted edges. 4. Compute diameter iteratively. 5. Find the diameter of an N-ary tree.

### 18. Variations
- Maximum path sum. - Longest univalue path. - Weighted tree diameter. - Forest diameter. - Diameter after each insertion.

### 19. Pattern Connections
Diameter is the classic distinction between what a node returns upward and what it scores locally. The parent can extend only one branch, while the global candidate may join two branches.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/diameter`. Reject payloads above a configured node limit before construction, then call only `diameterOfBinaryTree`. Return `{ diameterEdges, trace, truncated }`, cap postorder events at 200 with child heights and through-edge candidates, and document the practical safe-integer result convention.

### 21. Mini Challenge
Return one longest path as an array of root-relative directions or stable node IDs. Define deterministic tie-breaking and account separately for recursion, reconstruction, and output space.

## Problem 160 — Maximum Path Sum

### 1. Problem Statement
Given the root of a caller-owned proper finite binary tree whose values are safe integers, return the maximum safe-integer sum of any nonempty simple path. A path follows parent-child edges, may start and end at any nodes, and may change from upward to downward at most once. Throw `RangeError` for an empty tree, an unsafe node value, or any unsafe intermediate sum evaluated by the algorithm. Do not mutate the tree.

### 2. Example
```text
       -10
       /  \
      9    20
          /  \
         15   7

Output: 42
Explanation: 15 -> 20 -> 7 has sum 42.
```

### 3. Understand the Problem
The path does not need to include the root or a leaf. At its highest node, it can use at most one downward branch from the left and at most one from the right. Every input value and every addition performed must stay within JavaScript's safe-integer range; a later smaller result does not excuse an earlier unsafe intermediate sum.

#### In Simple Terms
Find the most valuable connected route anywhere in the tree, using each parent-child link at most once.

### 4. Real-World Analogy
Imagine choosing one continuous route through a branching network where each station adds profit or loss. The best route can remain inside one branch and should discard extensions whose total contribution is negative.

### 5. Key Observation
A parent may extend only one child branch upward, but a path scored at the parent may join one nonnegative gain from each child. This is the same return-one-branch, score-two-branches distinction used by tree diameter, now with checked arithmetic and weighted nodes.

### 6. Pattern
```text
Pattern: Postorder Gain with Global Optimum
Why this pattern applies: each node returns one extendable branch but scores two
How to recognize it: a path may join left and right only at one highest node
Similar problems: tree diameter, longest univalue path, weighted tree diameter
```

### 7. Brute Force
Treat every node as a possible highest node. Recompute all sums that start at each child and descend to any node, then use the greatest left and right sums to score the best one-sided and two-sided paths through the chosen node. Checked addition makes the arithmetic contract explicit.

#### Pseudocode
```text
downwardSums(node):
    validate node.value
    sums = [node.value]
    append node.value + each sum from downwardSums(left)
    append node.value + each sum from downwardSums(right)
    return sums

for each node as highest:
    recompute downward sums from each child
    score node alone, either best child sum + node, and both sides + node
return greatest score
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function maximumPathSumByRecomputation(root: TreeNode<number> | null): number {
  if (root === null) throw new RangeError("The tree must be nonempty");

  const assertSafe = (value: number): void => {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("Path values and sums must be safe integers");
    }
  };
  const safeAdd = (left: number, right: number): number => {
    const sum = left + right;
    assertSafe(sum);
    return sum;
  };
  const downwardSums = (node: TreeNode<number>): number[] => {
    assertSafe(node.value);
    const sums = [node.value];

    if (node.left !== null) {
      for (const childSum of downwardSums(node.left)) {
        sums.push(safeAdd(node.value, childSum));
      }
    }
    if (node.right !== null) {
      for (const childSum of downwardSums(node.right)) {
        sums.push(safeAdd(node.value, childSum));
      }
    }
    return sums;
  };

  let maximum = -Infinity;
  const scoreEveryHighestNode = (node: TreeNode<number> | null): void => {
    if (node === null) return;
    assertSafe(node.value);

    const leftSums = node.left === null ? [] : downwardSums(node.left);
    const rightSums = node.right === null ? [] : downwardSums(node.right);
    maximum = Math.max(maximum, node.value);

    const bestLeft = leftSums.length === 0 ? null : Math.max(...leftSums);
    const bestRight = rightSums.length === 0 ? null : Math.max(...rightSums);
    if (bestLeft !== null) maximum = Math.max(maximum, safeAdd(bestLeft, node.value));
    if (bestRight !== null) maximum = Math.max(maximum, safeAdd(node.value, bestRight));
    if (bestLeft !== null && bestRight !== null) {
      maximum = Math.max(
        maximum,
        safeAdd(safeAdd(bestLeft, node.value), bestRight),
      );
    }

    scoreEveryHighestNode(node.left);
    scoreEveryHighestNode(node.right);
  };

  scoreEveryHighestNode(root);
  return maximum;
}
```

#### Visualization
```text
highest 20:
left downward sums  = [15]
right downward sums = [7]
candidates          = 20, 35, 27, 42

Other highest nodes cannot beat 42.
```

#### Dry Run
At node `20`, the best downward sum from the left is `15` and from the right is `7`. Checked additions evaluate `15 + 20 = 35`, `20 + 7 = 27`, and then `(15 + 20) + 7 = 42`. Nodes `-10`, `9`, `15`, and `7` produce no larger candidate.

#### Brute-Force Complexity
Each highest node rebuilds arrays for its descendants, so time is `O(nh)` and therefore `O(n^2)` on a skewed tree. A recomputation can retain `O(n)` sums plus `O(h)` recursive frames; these are released before the next highest node is scored.

### 8. Why Brute Force Is Not Ideal
The same downward sums are rebuilt for many ancestors. A single postorder traversal can reduce each subtree to one extendable gain while still evaluating the best complete path at every node.

### 9. Optimal Approach
Postorder traversal returns the greatest nonnegative gain that a parent may extend through one child. At each node, checked additions first form `node + leftGain`, then add `rightGain` for the complete through-node path. Update the global maximum and return `node + max(leftGain, rightGain)`. Zero discards a negative child extension but never permits an empty final path.

### 10. Visualization
```text
node 15 -> through 15, return gain 15
node 7  -> through 7,  return gain 7
node 20 -> through 20 + 15 + 7 = 42, return gain 35
node 9  -> through 9, return gain 9
node -10 -> through -10 + 9 + 35 = 34, return gain 25
global maximum remains 42
```

### 11. Step-by-Step Dry Run
| Node | Left gain | Right gain | Through sum | Returned gain | Best |
|---:|---:|---:|---:|---:|---:|
| 9 | 0 | 0 | 9 | 9 | 9 |
| 15 | 0 | 0 | 15 | 15 | 15 |
| 7 | 0 | 0 | 7 | 7 | 15 |
| 20 | 15 | 7 | 42 | 35 | 42 |
| -10 | 9 | 35 | 34 | 25 | 42 |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function maximumPathSum(root: TreeNode<number> | null): number {
  if (root === null) throw new RangeError("The tree must be nonempty");

  const assertSafe = (value: number): void => {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("Path values and sums must be safe integers");
    }
  };
  const safeAdd = (left: number, right: number): number => {
    const sum = left + right;
    assertSafe(sum);
    return sum;
  };

  let maximum = -Infinity;
  const gain = (node: TreeNode<number> | null): number => {
    if (node === null) return 0;
    assertSafe(node.value);

    const leftGain = Math.max(0, gain(node.left));
    const rightGain = Math.max(0, gain(node.right));
    const throughLeft = safeAdd(node.value, leftGain);
    const throughNode = safeAdd(throughLeft, rightGain);
    maximum = Math.max(maximum, throughNode);

    return safeAdd(node.value, Math.max(leftGain, rightGain));
  };

  gain(root);
  return maximum;
}
```

### 13. Code Walkthrough
The nonempty guard keeps `-Infinity` from becoming an answer. Every visited value is validated before use. Child gains are completed first, negative gains are replaced by zero, and each performed addition passes through `safeAdd`. The global value scores a path ending on both sides, while the return value exposes only one branch to the parent.

### 14. Complexity Analysis
Each node is processed once, so time is `O(n)`. The recursion uses `O(h)` auxiliary space: `O(log n)` for a height-balanced tree and `O(n)` for a skewed tree. No path arrays are retained.

### 15. Edge Cases
- An empty tree throws `RangeError`; a one-node tree returns that safe value.
- An all-negative tree returns its greatest node value, not `0`.
- The optimal path may be wholly inside one subtree.
- Any unsafe node value throws even when its branch would contribute negatively.
- Any evaluated unsafe intermediate sum throws even if another path would win.
- Safe-integer addition is associated exactly as shown in the implementation.

### 16. Common Mistakes
- Returning zero for an all-negative tree and thereby allowing an empty path.
- Forcing both child branches into the gain returned to the parent.
- Checking only the final answer instead of every evaluated addition.
- Initializing the maximum to zero.
- Mutating nodes to cache gains without permission.

### 17. Interview Follow-ups
1. Return the endpoint node identities. 2. Reconstruct one maximizing path. 3. Add deterministic tie-breaking. 4. Handle safe-integer edge weights instead of node weights. 5. Implement an iterative postorder traversal.

### 18. Variations
- Maximum root-to-leaf sum. - Maximum leaf-to-leaf sum. - Tree diameter. - Longest positive-gain path. - Maximum path in an N-ary tree.

### 19. Pattern Connections
This problem is a weighted form of diameter. Both return one branch upward and score two branches locally; the differences are discarding negative gains, requiring a nonempty path, and enforcing checked arithmetic.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/maximum-path-sum`. Validate a bounded proper-tree payload, reject non-safe-integer values before construction, then call only `maximumPathSum`. Map `RangeError` to a documented client error, return `{ maximumSum, trace, truncated }`, and cap postorder trace events at 200 without evaluating extra arithmetic solely for tracing.

### 21. Mini Challenge
Return `{ sum, start, end }` using stable node IDs. Preserve the checked-addition contract and define which endpoints win when several nonempty paths have the same maximum sum.

## Problem 161 — Lowest Common Ancestor

### 1. Problem Statement
Given the root of a caller-owned proper finite binary tree and two non-null target node references, return their lowest common ancestor by identity. Both targets must occur in the tree; if either is absent, return `null`. If both references are the same present node, return that node. Values are irrelevant, duplicate values are allowed, and the tree must not be mutated or require parent pointers.

### 2. Example
```text
          3
         / \
        5   1
       / \ / \
      6  2 0  8
        / \
       7   4

first = node 5, second = node 4
Output: the node 5 identity
```

### 3. Understand the Problem
An ancestor may be one of the targets. “Lowest” means deepest among nodes whose subtree contains both target identities. Equal stored values do not identify a target, and a plausible split point is not an answer unless both references are confirmed present.

#### In Simple Terms
Find the deepest node that contains both requested node objects in its subtree, counting a target as its own ancestor.

### 4. Real-World Analogy
In an organization chart, find the closest shared manager of two exact employee records. Two employees may share a display name, so record identity rather than name determines whom the request means.

### 5. Key Observation
Each subtree can report which targets it contains as a two-bit mask. The first postorder subtree whose combined mask is `3` contains both targets; a descendant's completed ancestor must be preserved as the result moves upward.

### 6. Pattern
```text
Pattern: Postorder Presence Summary
Why this pattern applies: an ancestor is valid only after both targets are found
How to recognize it: identity targets may be absent or may be the same object
Similar problems: distance between nodes, directions between nodes, subtree search
```

### 7. Brute Force
Find a root-to-target path for each reference using identity comparisons. If either search fails, return `null`. Otherwise, compare the two path arrays from the root and retain their last shared node.

#### Pseudocode
```text
findPath(node, target, path):
    if node is null: return false
    append node
    if node is target: return true
    if left or right finds target: return true
    remove node and return false

build both paths
if either is missing: return null
scan while path entries are identical; return last match
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function lowestCommonAncestorByPaths<T>(
  root: TreeNode<T> | null,
  first: TreeNode<T>,
  second: TreeNode<T>,
): TreeNode<T> | null {
  const findPath = (
    node: TreeNode<T> | null,
    target: TreeNode<T>,
    path: TreeNode<T>[],
  ): boolean => {
    if (node === null) return false;
    path.push(node);
    if (node === target) return true;
    if (findPath(node.left, target, path) || findPath(node.right, target, path)) {
      return true;
    }
    path.pop();
    return false;
  };

  const firstPath: TreeNode<T>[] = [];
  const secondPath: TreeNode<T>[] = [];
  if (!findPath(root, first, firstPath) || !findPath(root, second, secondPath)) {
    return null;
  }

  let ancestor: TreeNode<T> | null = null;
  const sharedLength = Math.min(firstPath.length, secondPath.length);
  for (let index = 0; index < sharedLength; index += 1) {
    if (firstPath[index] !== secondPath[index]) break;
    ancestor = firstPath[index];
  }
  return ancestor;
}
```

#### Visualization
```text
path to 5: 3 -> 5
path to 4: 3 -> 5 -> 2 -> 4
shared identity prefix: 3 -> 5
last shared node: 5
```

#### Dry Run
The first DFS records root `3` and target `5`. The second records `3`, `5`, `2`, and `4`. Comparison succeeds at the identities for `3` and `5`, then the shorter path ends, so node `5` is returned.

#### Brute-Force Complexity
Two depth-first searches and one path comparison take `O(n)` time. The two retained paths and recursive search stack use `O(h)` auxiliary space. Nodes outside a target path are visited but not retained.

### 8. Why Brute Force Is Not Ideal
The tree is searched twice and two complete root paths are retained. One postorder traversal can combine target-presence information and determine the ancestor while also proving that both identities occur.

### 9. Optimal Approach
Assign bit `1` to `first` and bit `2` to `second`. Postorder recursion ORs the child masks and sets bits for identity matches at the current node; if both targets are the same identity, that node sets both bits. Preserve a child ancestor when present, otherwise mark the current node when its combined mask first becomes `3`. Return the ancestor only when the root mask is `3`.

### 10. Visualization
```text
node 4 matches second -> mask 2
node 2 combines mask 2
node 5 matches first and receives mask 2 -> mask 3, ancestor 5
node 3 preserves descendant ancestor 5
root mask 3 confirms both targets are present
```

### 11. Step-by-Step Dry Run
| Node | Left mask | Right mask | Own match | Combined | Ancestor |
|---:|---:|---:|---:|---:|---:|
| 4 | 0 | 0 | 2 | 2 | null |
| 2 | 0 | 2 | 0 | 2 | null |
| 5 | 0 | 2 | 1 | 3 | 5 |
| 1 | 0 | 0 | 0 | 0 | null |
| 3 | 3 | 0 | 0 | 3 | 5 |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function lowestCommonAncestor<T>(
  root: TreeNode<T> | null,
  first: TreeNode<T>,
  second: TreeNode<T>,
): TreeNode<T> | null {
  interface SearchResult {
    mask: number;
    ancestor: TreeNode<T> | null;
  }

  const search = (node: TreeNode<T> | null): SearchResult => {
    if (node === null) return { mask: 0, ancestor: null };

    const left = search(node.left);
    const right = search(node.right);
    let mask = left.mask | right.mask;
    if (node === first) mask |= 1;
    if (node === second) mask |= 2;

    return {
      mask,
      ancestor: left.ancestor ?? right.ancestor ?? (mask === 3 ? node : null),
    };
  };

  const result = search(root);
  return result.mask === 3 ? result.ancestor : null;
}
```

### 13. Code Walkthrough
Null subtrees contain no targets. Each real node combines child masks before adding its own identity matches. Two separate `if` statements intentionally let one node set both bits when `first === second`. Nullish selection preserves a lower completed ancestor instead of replacing it at every ancestor above it.

### 14. Complexity Analysis
The optimized traversal visits every node once to guarantee the missing-target contract, so time is `O(n)`. The recursive stack uses `O(h)` auxiliary space, and each frame stores only a constant-size summary.

### 15. Edge Cases
- A null root returns `null`.
- If either or both target references are absent, the result is `null`.
- If both targets are the same present node, that exact node is returned.
- One target may be an ancestor of the other.
- Different nodes with equal values remain distinct targets.
- The input must be a proper finite tree, so a target identity cannot occupy two positions.

### 16. Common Mistakes
- Comparing target values instead of node identities.
- Returning a found target before proving the other target is present.
- Treating the same-target case as only one match.
- Replacing a lower ancestor with every higher node whose mask is `3`.
- Adding parent fields or annotations to caller-owned nodes.

### 17. Interview Follow-ups
1. Support many LCA queries with preprocessing. 2. Return the distance between targets. 3. Find directions from one target to the other. 4. Implement iterative traversal with a parent map. 5. Generalize to an N-ary tree.

### 18. Variations
- LCA in a binary search tree. - LCA with parent pointers. - Deepest leaves' LCA. - Smallest subtree containing a node set. - Distance between node identities.

### 19. Pattern Connections
The bitmask is a compact postorder summary: each bit proves one requirement. The same technique combines multiple validations, tracks required descendants, and distinguishes “candidate found” from “all prerequisites confirmed.”

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/lowest-common-ancestor` using stable transport IDs for nodes, never serialized object identity supplied directly by a client. Build and validate one bounded proper tree, resolve both IDs to exact node references, call only `lowestCommonAncestor`, and return the ancestor ID or `null` with a capped postorder trace.

### 21. Mini Challenge
Return the root-relative direction strings from the ancestor to both targets in the same traversal, while preserving the absent-target and same-target behavior.

## Problem 162 — Path Sum

### 1. Problem Statement
Given the root of a caller-owned proper finite binary tree with numeric values and a numeric target, return whether any root-to-leaf path sums exactly to the target. The target and every node value must be safe integers. Validate the target before traversing, validate every node value even if an earlier path matches, and throw `RangeError` for any unsafe running sum. An empty tree returns `false` after target validation, and the tree must not be mutated.

### 2. Example
```text
          5
         / \
        4   8
       /   / \
      11  13  4
     /  \      \
    7    2      1

Target: 22
Output: true
Explanation: 5 -> 4 -> 11 -> 2 sums to 22.
```

### 3. Understand the Problem
Only paths beginning at the root and ending at a leaf qualify. Stopping at an internal node whose running sum equals the target is not enough. The validation contract covers the whole proper tree, so an invalid value in an unvisited branch must still throw even when another branch already matches.

#### In Simple Terms
Check whether one complete top-to-bottom route ends at a leaf with exactly the requested total.

### 4. Real-World Analogy
Think of every leaf as a checkout point on a branching route. A route qualifies only after reaching checkout, and its accumulated charges must equal the budget exactly.

### 5. Key Observation
The optimized search needs only one checked running sum, but short-circuiting that search could skip invalid values or overflowing prefixes. A separate full validation pass checks every value and every root-to-node sum; the second pass may then stop safely at the first matching leaf.

### 6. Pattern
```text
Pattern: Root-to-Leaf DFS with Running State
Why this pattern applies: each child inherits one accumulated prefix sum
How to recognize it: the endpoint must be a leaf, not any matching prefix
Similar problems: all path sums, path number construction, root-to-leaf maximum
```

### 7. Brute Force
Enumerate every root-to-leaf value array, copying the current path at each descent. Validate values while building paths. Then safely sum each completed path from zero and compare it with the already validated target.

#### Pseudocode
```text
validate target
collect(node, path):
    validate node.value
    nextPath = copy(path) plus node.value
    if node is leaf: store nextPath
    otherwise collect each child

for each stored path:
    sum values with checked addition
    if sum equals target: return true
return false
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function hasPathSumByEnumeration(
  root: TreeNode<number> | null,
  target: number,
): boolean {
  const assertSafe = (value: number): void => {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("Target, values, and sums must be safe integers");
    }
  };
  const safeAdd = (left: number, right: number): number => {
    const sum = left + right;
    assertSafe(sum);
    return sum;
  };

  assertSafe(target);
  if (root === null) return false;

  const paths: number[][] = [];
  const collect = (node: TreeNode<number>, path: readonly number[]): void => {
    assertSafe(node.value);
    const nextPath = [...path, node.value];
    if (node.left === null && node.right === null) {
      paths.push(nextPath);
      return;
    }
    if (node.left !== null) collect(node.left, nextPath);
    if (node.right !== null) collect(node.right, nextPath);
  };
  collect(root, []);

  for (const path of paths) {
    let sum = 0;
    for (const value of path) sum = safeAdd(sum, value);
    if (sum === target) return true;
  }
  return false;
}
```

#### Visualization
```text
completed paths:
[5, 4, 11, 7]  -> 27
[5, 4, 11, 2]  -> 22  match
[5, 8, 13]     -> 26
[5, 8, 4, 1]   -> 18
```

#### Dry Run
The collection pass visits the entire example and stores four fresh path arrays. Checked summation produces `27`, then `22`, which matches the target. All values were already validated during collection, so returning at that point cannot hide an unsafe unvisited node.

#### Brute-Force Complexity
Copying prefixes and later summing completed paths takes `O(nh)` time in the worst case. The stored paths can occupy `O(nh)` output-temporary space, and recursive traversal uses `O(h)` additional stack space.

### 8. Why Brute Force Is Not Ideal
Only the current route is needed to answer a boolean question, yet the baseline stores every route and repeatedly copies prefixes. After one validation pass, a running-sum DFS can search with only stack state.

### 9. Optimal Approach
First validate the target, then traverse the entire tree once with a checked running sum to validate every value and every root-to-node prefix. A second DFS carries the sum from the root, compares only at leaves, and may short-circuit on a match because all contract failures have already been ruled out. This remains linear because two constant-work passes are `O(n)`.

### 10. Visualization
```text
validation pass: all node values and root-prefix sums are safe

search:
5 -> 9 -> 20 -> 27 at leaf 7, no match
             -> 22 at leaf 2, match
return true
```

### 11. Step-by-Step Dry Run
| Node | Incoming sum | Checked sum | Leaf? | Decision |
|---:|---:|---:|---|---|
| 5 | 0 | 5 | no | descend left |
| 4 | 5 | 9 | no | descend left |
| 11 | 9 | 20 | no | try left, then right |
| 7 | 20 | 27 | yes | not equal |
| 2 | 20 | 22 | yes | return true |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function hasPathSum(root: TreeNode<number> | null, target: number): boolean {
  const assertSafe = (value: number): void => {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError("Target, values, and sums must be safe integers");
    }
  };
  const safeAdd = (left: number, right: number): number => {
    const sum = left + right;
    assertSafe(sum);
    return sum;
  };

  assertSafe(target);
  const validateTree = (
    node: TreeNode<number> | null,
    runningSum: number,
  ): void => {
    if (node === null) return;
    assertSafe(node.value);
    const nextSum = safeAdd(runningSum, node.value);
    validateTree(node.left, nextSum);
    validateTree(node.right, nextSum);
  };
  validateTree(root, 0);

  const search = (node: TreeNode<number> | null, runningSum: number): boolean => {
    if (node === null) return false;
    const nextSum = safeAdd(runningSum, node.value);
    if (node.left === null && node.right === null) return nextSum === target;
    return search(node.left, nextSum) || search(node.right, nextSum);
  };

  return search(root, 0);
}
```

### 13. Code Walkthrough
Target validation happens before the null-root result. `validateTree` deliberately checks both branches, including every root-prefix addition, before the search can short-circuit. The search repeats checked additions, recognizes a leaf only when both children are null, and preserves left-before-right evaluation through `||`.

### 14. Complexity Analysis
Validation visits all `n` nodes and search visits at most `n`, so total time is `O(n)`. They run sequentially and each uses `O(h)` recursive stack space. No root-to-leaf arrays are allocated.

### 15. Edge Cases
- A safe target with an empty tree returns `false`; an unsafe target throws first.
- A one-node tree matches only when its value equals the target.
- A matching internal-node prefix does not qualify.
- Negative values and a negative target are supported.
- A one-child node continues through its existing child; it is not a leaf.
- An unsafe value or prefix sum in a branch the search would skip still throws during prevalidation.
- A checked running sum can throw even when all individual values are safe.

### 16. Common Mistakes
- Accepting a target match at an internal node.
- Subtracting from the target without checking subtraction overflow.
- Returning early before validating values and prefix sums in the other branch.
- Treating a node with one missing child as a leaf.
- Storing every path in the optimized boolean solution.

### 17. Interview Follow-ups
1. Return one matching path. 2. Return all matching root-to-leaf paths. 3. Count matching paths. 4. Handle `bigint` values. 5. Implement the search iteratively with explicit stack entries.

### 18. Variations
- Path Sum II. - Any-downward-path sum. - Root-to-leaf number sum. - Maximum root-to-leaf sum. - Prefix-sum path counting.

### 19. Pattern Connections
Root-to-leaf DFS carries immutable scalar state, while backtracking problems often carry a mutable path buffer. The separate validation pass shows how API-wide input guarantees can coexist with an otherwise short-circuiting search.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/has-path-sum`. Validate the target and every bounded payload value as safe integers before construction, validate proper-tree shape, then call only `hasPathSum`. Return `{ exists, trace, truncated }`, cap search events at 200, and map arithmetic `RangeError` consistently even if transport validation normally catches unsafe inputs first.

### 21. Mini Challenge
Return one matching root-to-leaf path as a fresh value array while still validating every node before any successful early return and preserving checked running sums.

## Problem 163 — All Root-to-Leaf Paths

### 1. Problem Statement
Given the root of a caller-owned proper finite binary tree, return every root-to-leaf value path as a fresh `T[]`, ordered by left-before-right depth-first traversal. Return `[]` for an empty tree. The outer array and every path array must be fresh and non-aliased; object values themselves retain their original references. Do not mutate the tree or its values.

### 2. Example
```text
       A
      / \
     B   C
    / \   \
   D   E   F

Output: [[A, B, D], [A, B, E], [A, C, F]]
```

### 3. Understand the Problem
Each result begins at the root and ends only at a leaf. Traversing the left child before the right fixes deterministic output order. Fresh path arrays prevent editing one result from changing another, while generic values are copied as references rather than cloned.

#### In Simple Terms
List every complete route from the top to an endpoint, visiting left routes before right routes.

### 4. Real-World Analogy
A route planner lists all terminal itineraries through a branching network. Each itinerary gets its own sheet, but the location records written on those sheets still refer to the same locations.

### 5. Key Observation
All active recursive calls share one root-to-current prefix. Push before descending and pop after both children return; copy the buffer only when a leaf produces an output path.

### 6. Pattern
```text
Pattern: DFS Backtracking with Output Snapshots
Why this pattern applies: sibling routes share a prefix but need independent results
How to recognize it: mutate one owned path buffer and copy only completed answers
Similar problems: path-sum listing, maze routes, combination generation
```

### 7. Brute Force
Pass a newly copied prefix into every recursive call. At a leaf, wrap that already fresh prefix in an outer array. Combine left results before right results with array spreads.

#### Pseudocode
```text
collect(null, path) = []
collect(node, path):
    nextPath = copy(path) plus node.value
    if node is leaf: return [nextPath]
    return collect(left, nextPath) followed by collect(right, nextPath)
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function allRootToLeafPathsByCopying<T>(root: TreeNode<T> | null): T[][] {
  const collect = (node: TreeNode<T> | null, path: readonly T[]): T[][] => {
    if (node === null) return [];
    const nextPath = [...path, node.value];
    if (node.left === null && node.right === null) return [nextPath];
    return [
      ...collect(node.left, nextPath),
      ...collect(node.right, nextPath),
    ];
  };

  return collect(root, []);
}
```

#### Visualization
```text
A -> copy [A]
  B -> copy [A, B]
    D -> copy [A, B, D], emit
    E -> copy [A, B, E], emit
  C -> copy [A, C]
    F -> copy [A, C, F], emit
```

#### Dry Run
The call at `A` creates `[A]`. Its left call creates `[A, B]`; leaves `D` and `E` each create and return separate arrays. The right call independently creates `[A, C]` and then `[A, C, F]`. Concatenation preserves left-before-right order.

#### Brute-Force Complexity
Copying the prefix at every descent takes `O(nh)` time in the worst case, in addition to unavoidable output construction. Temporary prefixes and returned output can occupy `O(nh)` space, with `O(h)` recursive frames.

### 8. Why Brute Force Is Not Ideal
Internal prefixes are copied even though they are never returned. One owned backtracking buffer can represent the active route, reducing copying to the leaf snapshots that the caller actually receives.

### 9. Optimal Approach
Maintain one mutable local path array. On entry, push the node's value. At a leaf, append `[...path]` to the result. Otherwise recurse left and then right. Pop exactly once before returning so each sibling sees the correct prefix.

### 10. Visualization
```text
push A: [A]
push B: [A, B]
push D: [A, B, D] -> copy, pop D
push E: [A, B, E] -> copy, pop E
pop B
push C: [A, C]
push F: [A, C, F] -> copy, pop F
pop C, pop A
```

### 11. Step-by-Step Dry Run
| Event | Shared buffer | Output count |
|---|---|---:|
| enter D | `[A, B, D]` | 0 |
| snapshot D | `[A, B, D]` | 1 |
| enter E after D pop | `[A, B, E]` | 1 |
| snapshot E | `[A, B, E]` | 2 |
| enter F after left subtree unwinds | `[A, C, F]` | 2 |
| snapshot F | `[A, C, F]` | 3 |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function allRootToLeafPaths<T>(root: TreeNode<T> | null): T[][] {
  const paths: T[][] = [];
  const path: T[] = [];

  const visit = (node: TreeNode<T> | null): void => {
    if (node === null) return;
    path.push(node.value);

    if (node.left === null && node.right === null) {
      paths.push([...path]);
    } else {
      visit(node.left);
      visit(node.right);
    }

    path.pop();
  };

  visit(root);
  return paths;
}
```

### 13. Code Walkthrough
Both arrays are owned by the function. `path` is temporary mutable state, while each spread at a leaf creates a distinct result array. Left recursion occurs first, and the final pop restores the exact prefix held by the caller's frame. Stored object values are assigned normally, so their references are preserved.

### 14. Complexity Analysis
Traversal work is `O(n)`, and copying leaf paths adds time proportional to total output size, giving `O(n + output size)`. Excluding returned arrays, the recursive stack and shared buffer use `O(h)` auxiliary space. The output itself contains one array slot per value occurrence across all root-to-leaf paths.

### 15. Edge Cases
- An empty tree returns a fresh empty outer array.
- A single node returns `[[value]]`.
- A chain produces one path.
- A node with exactly one child is not a leaf.
- Left subtree paths always precede right subtree paths.
- Every returned path array is distinct, even when values are duplicates.
- Object values preserve identity and are not deep-cloned.

### 16. Common Mistakes
- Pushing the shared `path` itself into the result instead of copying it.
- Forgetting to pop after a leaf or subtree.
- Emitting paths at null children rather than at real leaves.
- Traversing right before left and changing the specified order.
- Deep-cloning generic values and breaking their reference identities.

### 17. Interview Follow-ups
1. Return direction strings instead of values. 2. Yield paths lazily with a generator. 3. Filter paths by a predicate. 4. Include node identities alongside values. 5. Implement iterative DFS while preserving order.

### 18. Variations
- Binary tree paths as strings. - Path Sum II. - Root-to-leaf number generation. - All paths in an N-ary tree. - Lazy path iteration.

### 19. Pattern Connections
This is tree-shaped backtracking: push, explore, and pop. Copying only accepted states is the same ownership pattern used for combinations, permutations, maze paths, and other output-sensitive searches.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/root-to-leaf-paths` only for bounded trees because output can be much larger than the input. Build and validate a proper tree, call only `allRootToLeafPaths`, serialize values through an approved mapper, and return `{ paths, truncated }` with explicit response limits rather than partially aliasing or mutating results.

### 21. Mini Challenge
Implement a generator that yields fresh root-to-leaf path arrays in the same left-before-right order, then explain which storage remains `O(h)` when the consumer processes one yielded path at a time.

## Problem 164 — Zigzag Level Order

### 1. Problem Statement
Given the root of a caller-owned proper finite binary tree, return its values grouped by depth. Read the root level from left to right, the next level from right to left, and alternate direction after every level. Return `[]` for an empty tree. The outer array and every level array must be fresh, generic values retain their original references, and the tree must not be mutated.

### 2. Example
```text
        1
       / \
      2   3
     / \ / \
    4  5 6  7

Output: [[1], [3, 2], [4, 5, 6, 7]]
```

### 3. Understand the Problem
Nodes still need to be discovered level by level. Only the order written into each returned level alternates. Child discovery must remain left before right so the next frontier retains normal structural order, independent of how the current values are displayed.

#### In Simple Terms
List one row at a time, reading the first row forward, the next backward, and continuing in alternating directions.

### 4. Real-World Analogy
Imagine inspecting shelves in a warehouse with a serpentine route. You scan one shelf left to right, move down, then scan the next shelf right to left without rearranging where any item is stored.

### 5. Key Observation
Breadth-first traversal already isolates one depth at a time. For a level of size `k`, the `i`th discovered value belongs at index `i` during a forward level and at index `k - 1 - i` during a backward level. Direct placement avoids reversing or front-inserting values.

### 6. Pattern
```text
Pattern: Level-Order BFS with Directional Placement
Why this pattern applies: output direction changes by depth, not by child discovery
How to recognize it: values are grouped by level with alternating order
Similar problems: level order, right-side view, vertical traversal, spiral matrix
```

### 7. Brute Force
Build every level in ordinary left-to-right BFS order. After finishing a level, reverse its array when the depth is odd. Two frontier arrays avoid repeated `shift()` calls and retain only adjacent levels.

#### Pseudocode
```text
frontier = [root]
leftToRight = true
while frontier is not empty:
    values = values of frontier in discovery order
    if not leftToRight: reverse values
    append values to result
    build next frontier from left child, then right child
    toggle direction
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function zigzagLevelOrderWithReversal<T>(
  root: TreeNode<T> | null,
): T[][] {
  if (root === null) return [];

  const levels: T[][] = [];
  let frontier: TreeNode<T>[] = [root];
  let leftToRight = true;

  while (frontier.length > 0) {
    const values: T[] = [];
    const nextFrontier: TreeNode<T>[] = [];

    for (const node of frontier) {
      values.push(node.value);
      if (node.left !== null) nextFrontier.push(node.left);
      if (node.right !== null) nextFrontier.push(node.right);
    }

    if (!leftToRight) values.reverse();
    levels.push(values);
    frontier = nextFrontier;
    leftToRight = !leftToRight;
  }

  return levels;
}
```

#### Visualization
```text
discover normally: [1] -> [2, 3] -> [4, 5, 6, 7]
reverse odd levels: [1] -> [3, 2] -> [4, 5, 6, 7]
```

#### Dry Run
The first frontier contains node `1`, so `[1]` is emitted unchanged. The second frontier is discovered as `[2, 3]` and reversed to `[3, 2]`. Their children are still discovered as `4, 5, 6, 7`, so the third level is emitted in that order.

#### Brute-Force Complexity
Every node is discovered once, and reversing all backward levels touches at most `n` values in total, so time is `O(n)`. The current and next frontiers use `O(w)` auxiliary references, excluding the `O(n)` returned output.

### 8. Why Brute Force Is Not Ideal
The baseline is already asymptotically optimal, but every backward level receives a second pass. Directly writing each discovered value to its final index expresses the alternating order without a reversal phase.

### 9. Optimal Approach
Process one normal left-to-right frontier at a time. Allocate a level array with the exact frontier length. Write each value either at its discovery index or at the mirrored index, append children left before right to the next frontier, emit the completed level, and toggle direction.

### 10. Visualization
```text
frontier [2, 3], size 2, backward:
discover 2 at i=0 -> write index 1: [_, 2]
discover 3 at i=1 -> write index 0: [3, 2]

children still append left before right for the next frontier
```

### 11. Step-by-Step Dry Run
| Depth | Frontier values | Direction | Write indexes | Output level |
|---:|---|---|---|---|
| 0 | `1` | left to right | `0` | `[1]` |
| 1 | `2, 3` | right to left | `1, 0` | `[3, 2]` |
| 2 | `4, 5, 6, 7` | left to right | `0, 1, 2, 3` | `[4, 5, 6, 7]` |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

function zigzagLevelOrder<T>(root: TreeNode<T> | null): T[][] {
  if (root === null) return [];

  const levels: T[][] = [];
  let frontier: TreeNode<T>[] = [root];
  let leftToRight = true;

  while (frontier.length > 0) {
    const levelSize = frontier.length;
    const values = new Array<T>(levelSize);
    const nextFrontier: TreeNode<T>[] = [];

    for (let index = 0; index < levelSize; index += 1) {
      const node = frontier[index];
      const writeIndex = leftToRight ? index : levelSize - 1 - index;
      values[writeIndex] = node.value;

      if (node.left !== null) nextFrontier.push(node.left);
      if (node.right !== null) nextFrontier.push(node.right);
    }

    levels.push(values);
    frontier = nextFrontier;
    leftToRight = !leftToRight;
  }

  return levels;
}
```

### 13. Code Walkthrough
`frontier` always stores the current depth in normal left-to-right structural order. `writeIndex` changes only the output position, so child discovery remains consistent. Every slot in `values` is assigned exactly once before the fresh level array is appended to the fresh outer result.

### 14. Complexity Analysis
Each node is read and written once, so time is `O(n)`. The current and next frontiers hold at most two adjacent levels and therefore use `O(w)` auxiliary references. The returned level arrays use `O(n)` required output space.

### 15. Edge Cases
- An empty tree returns a fresh `[]`.
- A one-node tree returns `[[value]]`.
- A one-sided chain produces one value per level, so direction changes are not visually apparent.
- Duplicate values are preserved in their positional order.
- Object values preserve identity and are not deep-cloned.
- A very wide level is allocated once and filled without front insertion.

### 16. Common Mistakes
- Reversing child-enqueue order and corrupting the next level's structural order.
- Calling `unshift()` for every value on backward levels, which can make a wide level quadratic.
- Calling `shift()` repeatedly on a general array queue.
- Toggling direction after each node instead of after each level.
- Returning one flat array when the contract requires grouping by depth.

### 17. Interview Follow-ups
1. Return one flat zigzag sequence. 2. Implement the traversal with a deque. 3. Yield one level at a time. 4. Return node identities instead of values. 5. Generalize to an N-ary tree.

### 18. Variations
- Ordinary level order. - Reverse level order. - Bottom-up level order. - Right-side view. - Spiral traversal with two stacks.

### 19. Pattern Connections
Zigzag traversal separates structural discovery from presentation order. That distinction also appears when BFS computes a right-side view, groups nodes by column, or emits a transformed view without changing the underlying graph traversal.

### 20. NestJS Project Connection
Expose `POST /api/dsa/trees/zigzag-level-order`. Validate and build a bounded proper tree through Problem 146, then call only `zigzagLevelOrder`. Serialize generic values through an approved mapper, return `{ levels, trace, truncated }`, and cap optimized events such as `{ depth, frontierSize, leftToRight }` without copying whole frontier snapshots.

### 21. Mini Challenge
Implement a lazy generator that yields one fresh zigzag level at a time. Explain why pausing after a yield retains the next frontier and how that affects auxiliary-space accounting.

## Problem 165 — Serialize and Deserialize Binary Tree

### 1. Problem Statement
Implement `serializeBinaryTree` and `deserializeBinaryTree` for caller-owned proper finite binary trees. Serialization must preserve exact shape and use a caller-supplied `ValueCodec<T>` to convert each generic value to and from a string payload. The wire format is preorder: `N;` represents null, and `V<length>:<payload>` represents a node followed by its left and right subtrees. Length is the payload's number of JavaScript UTF-16 code units. Deserialization returns fresh nodes, consumes exactly one complete tree, and throws `RangeError` for malformed grammar or trailing data. Codec errors propagate unchanged. Neither operation mutates its input.

### 2. Example
```text
Tree:          Wire form with decimal-number codec:
      8        V1:8 V1:3 N; N; V2:10 N; N;
     / \
    3  10      (spaces shown only for readability)

Actual string: "V1:8V1:3N;N;V2:10N;N;"
```

### 3. Understand the Problem
Values alone cannot recover missing-child positions, so nulls must be encoded. Payload delimiters cannot be trusted because a codec may produce any string, including `:`, `N;`, or Unicode text. A canonical decimal length says exactly how many UTF-16 code units belong to the payload; parsing then resumes at the next subtree marker.

#### In Simple Terms
Turn the whole tree into one precise string, including empty child positions, and rebuild an equivalent tree from that string without guessing where values end.

### 4. Real-World Analogy
A shipping manifest records each container's payload length before the payload itself. Even when the cargo label contains punctuation that looks like manifest syntax, the receiver knows exactly how many characters to consume before reading the next container.

### 5. Key Observation
Preorder determines the root before its two subtrees, and explicit null markers terminate every missing branch. Therefore one recursive parser can consume exactly one tree from left to right. Length-prefixing makes arbitrary codec strings self-delimiting.

### 6. Pattern
```text
Pattern: Preorder Encoding with Explicit Nulls
Why this pattern applies: root-first order plus null markers preserves full shape
How to recognize it: a recursive structure must round-trip through a linear format
Similar problems: AST persistence, trie encoding, expression parsing, protocol framing
```

### 7. Brute Force
First mirror the tree as nested JSON-compatible records whose values are codec payload strings and whose absent children are `null`. Use `JSON.stringify` for serialization. For deserialization, call `JSON.parse`, validate every parsed record and exact key set, decode values, and allocate fresh tree nodes.

#### Pseudocode
```text
encode(node):
    if null: return null
    return record(value payload, encode(left), encode(right))
serialize = JSON.stringify(encode(root))

deserialize:
    JSON.parse input
    recursively validate null or exact node record
    decode payload and allocate fresh node
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

interface ValueCodec<T> {
  encode(value: T): string;
  decode(payload: string): T;
}

type JsonTree = {
  value: string;
  left: JsonTree | null;
  right: JsonTree | null;
};

function serializeBinaryTreeAsJson<T>(
  root: TreeNode<T> | null,
  codec: ValueCodec<T>,
): string {
  const encode = (node: TreeNode<T> | null): JsonTree | null => {
    if (node === null) return null;
    return {
      value: codec.encode(node.value),
      left: encode(node.left),
      right: encode(node.right),
    };
  };

  return JSON.stringify(encode(root));
}

function deserializeBinaryTreeFromJson<T>(
  data: string,
  codec: ValueCodec<T>,
): TreeNode<T> | null {
  let parsed: unknown;
  try {
    parsed = JSON.parse(data) as unknown;
  } catch {
    throw new RangeError("Malformed serialized tree");
  }

  const decode = (input: unknown): TreeNode<T> | null => {
    if (input === null) return null;
    if (typeof input !== "object" || Array.isArray(input)) {
      throw new RangeError("Malformed serialized tree");
    }

    const record = input as Record<string, unknown>;
    const keys = Object.keys(record);
    const hasExactKeys =
      keys.length === 3 &&
      keys.every(
        (key) => key === "value" || key === "left" || key === "right",
      );
    if (!hasExactKeys || typeof record.value !== "string") {
      throw new RangeError("Malformed serialized tree");
    }

    return {
      value: codec.decode(record.value),
      left: decode(record.left),
      right: decode(record.right),
    };
  };

  return decode(parsed);
}
```

#### Visualization
```text
tree record allocation:
{ value: "8",
  left:  { value: "3", left: null, right: null },
  right: { value: "10", left: null, right: null } }

JSON.stringify then escapes and frames the nested record.
```

#### Dry Run
The encoder creates the record for `8`, recursively creates leaf records for `3` and `10`, and gives `JSON.stringify` the completed mirror. The decoder parses that JSON, validates each exact three-key node record, decodes each payload, and creates three new node objects with the same shape.

#### Brute-Force Complexity
Let `s` be the serialized string length. Mirroring, stringifying, parsing, validating, and rebuilding take `O(n + s)` time. The nested mirror or parsed JSON tree uses `O(n + s)` temporary space in addition to recursive `O(h)` frames, the returned nodes, and the serialized string.

### 8. Why Brute Force Is Not Ideal
The JSON approach creates a full intermediate object tree before serialization and another parsed object tree before reconstruction. A purpose-built grammar can append tokens directly and parse them with one cursor, while remaining unambiguous for arbitrary codec payload strings.

### 9. Optimal Approach
Serialize with preorder DFS. Append `N;` for null; otherwise append `V`, the canonical decimal payload length, `:`, the payload, then both child encodings. Deserialize with one shared cursor. Validate the marker and length, slice exactly the declared payload, decode it, recursively parse left and right, and reject any bytes left after the root tree.

### 10. Visualization
```text
        8
       / \
      3  10

preorder tokens:
8, 3, null, null, 10, null, null

framed string:
V1:8 V1:3 N; N; V2:10 N; N;
```

### 11. Step-by-Step Dry Run
| Cursor token | Action | Partial result |
|---|---|---|
| `V1:8` | decode root `8` | node 8 allocated after children parse |
| `V1:3` | decode left value `3` | parse two `N;` children |
| `N; N;` | finish node `3` | left subtree complete |
| `V2:10` | decode right value `10` | parse two `N;` children |
| end | cursor equals string length | return reconstructed root |

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

interface ValueCodec<T> {
  encode(value: T): string;
  decode(payload: string): T;
}

function serializeBinaryTree<T>(
  root: TreeNode<T> | null,
  codec: ValueCodec<T>,
): string {
  const chunks: string[] = [];

  const visit = (node: TreeNode<T> | null): void => {
    if (node === null) {
      chunks.push("N;");
      return;
    }

    const payload = codec.encode(node.value);
    chunks.push(`V${payload.length}:${payload}`);
    visit(node.left);
    visit(node.right);
  };

  visit(root);
  return chunks.join("");
}

function deserializeBinaryTree<T>(
  data: string,
  codec: ValueCodec<T>,
): TreeNode<T> | null {
  let cursor = 0;

  const malformed = (): never => {
    throw new RangeError("Malformed serialized tree");
  };

  const parse = (): TreeNode<T> | null => {
    if (data.startsWith("N;", cursor)) {
      cursor += 2;
      return null;
    }
    if (data[cursor] !== "V") malformed();

    const lengthStart = cursor + 1;
    const colon = data.indexOf(":", lengthStart);
    if (colon === -1) malformed();

    const lengthText = data.slice(lengthStart, colon);
    if (!/^(0|[1-9]\d*)$/.test(lengthText)) malformed();
    const payloadLength = Number(lengthText);
    if (!Number.isSafeInteger(payloadLength)) malformed();

    const payloadStart = colon + 1;
    if (payloadLength > data.length - payloadStart) malformed();
    const payloadEnd = payloadStart + payloadLength;
    const payload = data.slice(payloadStart, payloadEnd);
    cursor = payloadEnd;

    const value = codec.decode(payload);
    const left = parse();
    const right = parse();
    return { value, left, right };
  };

  const root = parse();
  if (cursor !== data.length) malformed();
  return root;
}
```

### 13. Code Walkthrough
Serialization accumulates chunks so repeated string concatenation cannot copy an ever-growing prefix. `payload.length` and `slice` both use UTF-16 code units, including for surrogate pairs and lone surrogates. During parsing, only canonical nonnegative decimal lengths are accepted. A node is returned only after both required subtree encodings are consumed, and the final cursor check rejects concatenated trees or other trailing data.

### 14. Complexity Analysis
Let `s` be the wire-string length. Every node marker and payload code unit is emitted and consumed once, so serialization and deserialization each take `O(n + s)` time. Serialization uses `O(n + s)` chunk/output storage plus `O(h)` frames. Deserialization uses `O(h)` parsing frames plus `O(n)` returned nodes; payload slices contribute total `O(s)` transient string data across the parse.

### 15. Edge Cases
- An empty tree serializes as `N;` and deserializes to `null`.
- Empty payloads are valid and use `V0:` before the two subtree encodings.
- Payloads may contain markers, colons, digits, newlines, Unicode, or lone surrogates.
- `V01:xN;N;`, negative lengths, missing children, truncated payloads, unknown markers, and trailing data throw `RangeError`.
- A syntactically valid payload rejected by `codec.decode` produces the codec's original error.
- Every reconstructed node is fresh; reconstructed value identity and equality follow the codec's contract.
- Recursive implementations require an agreed depth bound for untrusted or highly skewed input.

### 16. Common Mistakes
- Omitting null markers and losing whether a single child was left or right.
- Splitting on a delimiter that may occur inside an encoded value.
- Measuring payloads in code points while slicing JavaScript strings by UTF-16 code units.
- Accepting one valid prefix while ignoring trailing data.
- Using `JSON.stringify` directly on arbitrary generic values and pretending every value round-trips.
- Claiming that deserialization preserves original node or object-value identities.

### 17. Interview Follow-ups
1. Add an iterative parser for very deep trees. 2. Include a version header. 3. Add a checksum. 4. Stream tokens without holding the whole wire string. 5. Encode binary payloads with byte lengths instead of UTF-16 lengths.

### 18. Variations
- Level-order serialization. - BST serialization without null markers. - N-ary tree encoding. - Binary protocol framing. - DAG serialization with identity tables.

### 19. Pattern Connections
This is recursive-descent parsing applied to a recursive data structure. Explicit nulls play the role of grammar terminals, while payload lengths provide protocol framing. The same ideas appear in AST storage, network messages, and any format where content may contain delimiter-like text.

### 20. NestJS Project Connection
Expose separate bounded endpoints for serialization and deserialization only with a fixed, reviewed transport codec rather than accepting executable codec behavior from clients. Enforce request-byte, payload-length, node-count, and depth limits before or during parsing; call only the optimized functions; map grammar `RangeError` to HTTP `400`; and return compact metadata such as `{ data, nodeCount }` or a bounded tree representation without logging full payloads.

### 21. Mini Challenge
Replace recursive deserialization with an explicit stack of unfinished node frames. Preserve the exact grammar, left-before-right attachment order, fresh-node semantics, codec error behavior, and rejection of incomplete or trailing input.

# Topic Summary

## What You Learned
Binary trees combine recursive shape with reference identity. A proper finite tree has no cycles or shared child identities, and algorithms must distinguish node identity from stored-value equality. Height-sensitive DFS storage depends on tree shape, while BFS storage depends on maximum width.

Traversal order determines when a node is processed: preorder handles the root before its children, inorder places it between them, and postorder combines child summaries before the parent. Level-order traversal instead follows breadth. Efficient implementations use explicit stacks, frontier arrays, or head-index queues without hidden `shift()` or front-insertion costs.

Many advanced tree problems return one small summary upward while scoring a richer candidate locally. Balance, diameter, and maximum path sum use this postorder pattern. Production contracts also need explicit empty-tree behavior, ownership, checked numeric arithmetic, bounded traces, deterministic output order, and unambiguous serialization with strict malformed-input handling.

## Patterns You Should Recognize
- **Recursive structural traversal:** solve left and right subtrees around one root operation.
- **Explicit-stack DFS:** preserve recursive order while moving frames into owned storage.
- **Frontier BFS:** process one depth at a time with `O(w)` auxiliary references.
- **Leaf-aware search:** distinguish a real endpoint from a node with only one missing child.
- **Postorder summaries:** return height, count, gain, mask, or another constant-size result to the parent.
- **Sentinel propagation:** combine an ordinary answer with failure, as in balance checking.
- **Return one branch, score two:** extend one child upward while joining both children locally.
- **Identity-based target search:** locate exact node objects rather than matching duplicate values.
- **DFS backtracking:** reuse one path buffer and snapshot only completed root-to-leaf paths.
- **Recursive-descent serialization:** pair structural markers with framed value payloads.

## Complexity Cheat Sheet
| Problem | Main approach | Time | Auxiliary space | Required output space |
|---:|---|---|---|---|
| 146 | Complete-slot-index construction | `O(s)` for `s` slots | `O(n)` node table | `O(n)` fresh tree |
| 147 | Iterative preorder stack | `O(n)` | `O(h)` | `O(n)` |
| 148 | Iterative inorder stack | `O(n)` | `O(h)` | `O(n)` |
| 149 | Framed iterative postorder | `O(n)` | `O(h)` | `O(n)` |
| 150 | Two-frontier level order | `O(n)` | `O(w)` | `O(n)` |
| 151 | Recursive maximum depth | `O(n)` | `O(h)` | `O(1)` scalar |
| 152 | Leaf-aware breadth-first search | `O(n)` worst case | `O(w)` | `O(1)` scalar |
| 153 | Postorder node count | `O(n)` | `O(h)` | `O(1)` scalar |
| 154 | Recursive leaf count | `O(n)` | `O(h)` | `O(1)` scalar |
| 155 | Checked postorder sum | `O(n)` | `O(h)` | `O(1)` scalar |
| 156 | Paired structural comparison | `O(n)` worst case | `O(h)` | `O(1)` boolean |
| 157 | Mirrored pair comparison | `O(n)` worst case | `O(h)` | `O(1)` boolean |
| 158 | Height-or-failure sentinel | `O(n)` | `O(h)` | `O(1)` boolean |
| 159 | Postorder height and diameter | `O(n)` | `O(h)` | `O(1)` scalar |
| 160 | Postorder gain and global maximum | `O(n)` | `O(h)` | `O(1)` scalar |
| 161 | Postorder target-presence mask | `O(n)` | `O(h)` | `O(1)` node reference |
| 162 | Prefix validation plus path search | `O(n)` | `O(h)` | `O(1)` boolean |
| 163 | Backtracking path snapshots | `O(n + output size)` | `O(h)` | proportional to returned paths |
| 164 | BFS with directional placement | `O(n)` | `O(w)` | `O(n)` |
| 165 | Framed preorder codec | `O(n + s)` for wire length `s` | `O(h)` plus string storage | `O(n)` decoded tree or `O(s)` wire string |

## Common Interview Traps
- Traversing a cyclic or shared-node graph while claiming the input is a proper tree.
- Comparing values when the problem identifies exact node references.
- Calling recursive DFS constant-space or ignoring skewed-tree stack depth.
- Using `shift()` or repeated `unshift()` and hiding array-reindexing costs.
- Treating a node with one child as a leaf.
- Mixing edge-count and node-count conventions for depth, height, or diameter.
- Recomputing subtree heights and turning a linear postorder problem quadratic.
- Returning both child branches to a parent when only one can be extended.
- Allowing an empty maximum path in an all-negative tree.
- Short-circuiting before whole-tree validation required by the contract.
- Comparing LCA targets by value or returning a candidate before proving presence.
- Storing one mutable path reference for every root-to-leaf result.
- Reversing child discovery instead of only the displayed zigzag level.
- Omitting serialization null markers or using ambiguous value delimiters.
- Ignoring trailing serialized data, malformed lengths, codec semantics, or depth limits.

## Practice Checklist
- [ ] 146. Build a fresh binary tree from validated complete-slot indexes.
- [ ] 147. Produce root-left-right preorder with an explicit stack.
- [ ] 148. Simulate inorder's left descent without mutating nodes.
- [ ] 149. Emit postorder values only after both child frames complete.
- [ ] 150. Group values by depth with bounded frontier storage.
- [ ] 151. Derive maximum node-depth from child depths.
- [ ] 152. Stop breadth-first search at the first real leaf.
- [ ] 153. Count every node with checked safe-integer arithmetic.
- [ ] 154. Count only nodes whose two child references are null.
- [ ] 155. Validate and sum all numeric node values safely.
- [ ] 156. Compare two trees by shape and SameValueZero values.
- [ ] 157. Compare mirrored child pairs to test symmetry.
- [ ] 158. Propagate an unbalanced sentinel in one postorder traversal.
- [ ] 159. Return one height upward while scoring diameter in edges.
- [ ] 160. Discard negative gains and score a nonempty maximum path.
- [ ] 161. Find identity-based LCA while proving both targets are present.
- [ ] 162. Validate every root-prefix sum before a short-circuiting path search.
- [ ] 163. Snapshot fresh root-to-leaf paths from one backtracking buffer.
- [ ] 164. Place each BFS value directly into its zigzag output index.
- [ ] 165. Round-trip generic trees with explicit nulls and framed payloads.