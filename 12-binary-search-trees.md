# Binary Search Trees

## Learning Objectives

By the end of this topic, you should understand:

- how a binary search tree adds an ordering invariant to an ordinary binary tree
- how a comparator defines key equality and left-versus-right placement
- why this chapter stores one node per comparator-distinct key
- how tree height controls search, insertion, deletion, and neighbor-query costs
- why inorder traversal visits valid BST keys in strictly increasing comparator order
- how mutation, returned roots, detached nodes, and caller-owned references interact
- how bounds validate the whole tree without relying on numeric sentinels
- how successor, predecessor, LCA, and kth-smallest queries exploit ordering
- why sorted insertion can create a linear chain and how midpoint construction avoids it
- how production APIs enforce comparator, size, depth, ownership, and trace contracts

## Prerequisites

- binary-tree structure, identity, ownership, and traversal from `11-binary-trees.md`
- arrays, generics, loops, recursion, and object references
- strict TypeScript, `null`, readonly input, and explicit return types
- Big-O notation, especially the difference between node count `n` and height `h`
- the distinction between comparator equality and JavaScript identity or `===`

## Core Theory

### Node and comparator model

Every problem uses this structural node shape and comparator contract:

```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;
```

Only the comparator result's sign matters:

- `compare(a, b) < 0` means `a` comes before `b`
- `compare(a, b) === 0` means `a` and `b` are the same key
- `compare(a, b) > 0` means `a` comes after `b`

A comparator must be deterministic and define a strict total order: comparison is consistent with itself, reversed arguments have reversed ordering, and transitive ordering does not contradict itself. Algorithms cannot repair an invalid or stateful comparator.

### Strict BST invariant

This chapter uses a **strict, duplicate-free BST**. For every node:

- every key in its left subtree compares less than the node's key
- every key in its right subtree compares greater than the node's key
- no two nodes contain comparator-equal keys
- both subtrees satisfy the same rules recursively

The invariant concerns all descendants, not only immediate children. A value smaller than the root remains bounded by the root even when it appears many levels below.

### Comparator equality versus identity

Two different objects may compare as the same key. For example, records `{ id: 7, name: "A" }` and `{ id: 7, name: "B" }` are equal keys under an `id` comparator even though they are different objects. Search returns the stored node whose value compares equal to the query; it does not replace that value with the query object.

Insertion of a comparator-equal key is a no-op in this chapter. Deletion removes the stored comparator-equal key. These choices make every operation consistent with the strict invariant.

### Proper finite tree precondition

Unless a problem explicitly constructs a new tree, its input must be a caller-owned proper finite binary tree: no cycles, no shared child identities, and every non-root node has exactly one parent. Most BST operations additionally assume the ordering invariant already holds. Problem 172 is the exception because its purpose is to test that invariant.

The algorithms do not repeatedly validate shape or ordering before each operation. Doing so would turn an expected `O(h)` lookup into `O(n)`. Validate untrusted transport data once at the boundary, then operate on the trusted in-memory tree.

### Height controls performance

Let `h` be edge-height. Search follows one root-to-null path, so it takes `O(h + 1)`, conventionally written `O(h)`. The same bound applies to ordinary insertion, deletion, minimum, maximum, successor, predecessor, and BST LCA.

```text
balanced BST                  right-skewed BST

       8                            2
     /   \                           \
    4    12                           4
   / \   / \                           \
  2   6 10 14                           6
                                          \
                                           8

h = O(log n)                    h = n - 1
```

Therefore those operations are `O(log n)` on a height-balanced tree but `O(n)` in the worst case. A plain BST does not rebalance itself.

### Inorder ordering theorem

Inorder traversal visits left subtree, node, then right subtree. Under the strict BST invariant, every emitted key is comparator-greater than the previous key, so inorder output is strictly sorted.

The reverse implication is also useful: if a complete inorder traversal is not strictly increasing, the tree is not a valid strict BST. A bounds-based validator expresses the invariant more directly and can stop early without storing all values.

### Mutation and returned roots

Insertion and deletion in this chapter mutate links in the caller-owned tree and return the current root. Callers must assign the result:

```text
root = insertIntoBst(root, value, compare)
root = deleteFromBst(root, key, compare)
```

Assignment matters because insertion into an empty tree creates a root, and deleting the old root may replace it. Existing nodes retain identity unless deletion removes that exact node. A deleted node is detached by setting both of its child fields to `null`, making the postcondition clear even when the caller kept a reference to it.

### Deletion cases

Deleting a node has three structural cases:

1. **No children:** replace its parent link with `null`.
2. **One child:** replace its parent link with that child.
3. **Two children:** remove the inorder successor, the minimum node in the right subtree, and transplant that existing node into the deleted node's position.

Moving the successor node preserves value identity and avoids silently copying a value into a different node. The removed target is detached; all surviving node identities remain in the tree.

### Bounds instead of numeric sentinels

A generic validator cannot assume `-Infinity` and `Infinity` belong to `T`. Carry optional lower and upper values instead. A node must compare strictly above a present lower bound and strictly below a present upper bound.

```text
validate(node, lower?, upper?):
    require lower < node.value < upper when bounds exist
    validate(left, lower, node.value)
    validate(right, node.value, upper)
```

Bounds also catch deep violations that parent-child-only checks miss.

### Order statistics

The kth-smallest key is the kth value emitted by inorder traversal. Without stored subtree sizes, finding it takes `O(h + k)` time with an explicit inorder stack and `O(h)` auxiliary space. An augmented BST that maintains each subtree's node count can answer the query in `O(h)` time, but every mutation must then update those counts correctly.

### Balanced construction from sorted input

Given strictly sorted distinct input, choosing the middle element as root leaves nearly equal numbers of values on both sides. Repeating that choice recursively creates a height-balanced BST with `O(n)` fresh nodes and `O(log n)` recursive depth.

This chapter uses the lower middle index for even-length ranges. That deterministic tie-break does not change balance or correctness, but it fixes the exact returned shape.

## Visual Mental Model

Use this strict numeric BST for examples:

```text
          8
        /   \
       4    12
      / \   / \
     2   6 10 14

inorder: 2, 4, 6, 8, 10, 12, 14
minimum: 2
maximum: 14
successor of 6: 8
predecessor of 10: 8
LCA of 2 and 6: 4
4th smallest: 8
```

A search compares once per visited node:

```text
search 10:
10 > 8   -> right
10 < 12  -> left
10 = 10  -> found
```

## Complexity Concepts

### Height-sensitive operations

| Operation | Balanced tree | Worst-case skewed tree | Auxiliary space in iterative form |
|---|---:|---:|---:|
| Search | `O(log n)` | `O(n)` | `O(1)` |
| Insert | `O(log n)` | `O(n)` | `O(1)` |
| Delete | `O(log n)` | `O(n)` | `O(1)` |
| Minimum / maximum | `O(log n)` | `O(n)` | `O(1)` |
| Successor / predecessor by key | `O(log n)` | `O(n)` | `O(1)` |
| LCA by keys | `O(log n)` | `O(n)` | `O(1)` |

### Whole-tree operations

Validation must inspect every node when the tree is valid, so it takes `O(n)` time and `O(h)` recursive stack space. Building a balanced tree from sorted input also takes `O(n)` time because it allocates one node per value.

### Output and ownership space

Returning one existing node reference is `O(1)` output. Insertion allocates at most one node. Deletion allocates none. Sorted-array construction returns `O(n)` fresh nodes; those nodes are required output rather than auxiliary workspace.

## Common Patterns

### Comparator-guided descent

```text
current = root
while current exists:
    relation = compare(key, current.value)
    if relation is zero: found
    if relation is negative: move left
    otherwise: move right
```

### Candidate tracking

Successor and predecessor searches remember the best strict ancestor candidate. Moving left makes the current node a possible successor; moving right makes it a possible predecessor.

### Split-point LCA

When both keys are smaller, move left. When both are larger, move right. Otherwise, the current node is where their search paths split or where one key is found.

### Iterative inorder prefix

Push a left chain, pop the next-smallest node, and stop after the kth pop. This avoids traversing nodes that occur after the requested rank.

## Common Mistakes

- Assuming every binary tree is a BST.
- Checking only a node's immediate children during validation.
- Mixing `===` equality with comparator equality.
- Leaving duplicate-key behavior unspecified.
- Forgetting to assign the root returned by insertion or deletion.
- Copying a successor value during deletion without documenting identity effects.
- Calling plain-BST operations `O(log n)` without stating the balance condition.
- Using numeric sentinels for generic keys.
- Returning a neighbor for a query key that is absent when the contract requires presence.
- Building a balanced tree by repeated insertion and accidentally taking `O(n log n)` time.
- Mutating a caller's sorted input array with `shift()`, `splice()`, or sorting.
- Exposing unbounded traces or accepting an arbitrary client-defined comparator in an API.

## Production Notes

- Use a fixed server-side comparator for each endpoint and data model.
- Validate untrusted shape, ordering, node count, and depth once before trusted operations.
- Cap request size, recursion depth, response previews, and trace events.
- Trace only optimized comparisons or relinks; do not serialize entire subtrees per event.
- Serialize keys through an approved mapper and avoid logging sensitive values.
- Treat mutating request operations as request-scoped unless persistence and concurrency are deliberately designed.
- Consider AVL, red-black, B-tree, or database indexes when worst-case balance guarantees matter.

## Problem 166 — Search in BST

### 1. Problem Statement
Given the root of a caller-owned proper finite valid strict BST, a query key, and the tree's comparator, return the stored node whose value is comparator-equal to the key. Return `null` when no such node exists. The search must be read-only: do not change links, replace values, or allocate replacement nodes. Use only the sign of comparator results.

### 2. Example
```text
          8
        /   \
       4    12
      / \   / \
     2   6 10 14

search 10 -> reference to the existing node containing 10
search 5  -> null
```

For object values compared by `id`, the returned value is the object already stored in the tree, not the query object.

### 3. Understand the Problem
The strict BST invariant gives every comparison one useful direction. Comparator equality identifies the key even when the query and stored value are different JavaScript objects. The result must preserve node and value identity because it is a reference into the caller's original tree.

#### In Simple Terms
Compare at the current node, then go only toward the side where the key could still exist.

### 4. Real-World Analogy
A sorted filing cabinet lets you compare a requested label with the current divider. You open only the lower or higher section, and when the labels match, you hand back the original folder rather than making a copy.

### 5. Key Observation
If `compare(key, current.value)` is negative, every node in the right subtree is too large, so only the left subtree can contain the key. A positive result rules out the left subtree. Zero returns the current node immediately.

### 6. Pattern
```text
Pattern: Comparator-Guided Descent
Why this pattern applies: the BST invariant eliminates one subtree per comparison
How to recognize it: a key lookup over strictly ordered nodes
Similar problems: insertion, deletion, successor, predecessor, BST LCA
```

### 7. Brute Force
Ignore the BST ordering and perform a depth-first traversal as if the input were an ordinary binary tree. Compare the key with every visited value until a comparator-equal node is found or all nodes have been examined.

#### Pseudocode
```text
if root is null: return null
stack = [root]
while stack is not empty:
    node = pop stack
    if compare(key, node.value) is zero: return node
    push existing children
return null
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function searchBstByTraversal<T>(
  root: TreeNode<T> | null,
  key: T,
  compare: Comparator<T>,
): TreeNode<T> | null {
  if (root === null) return null;

  const stack: TreeNode<T>[] = [root];
  while (stack.length > 0) {
    const node = stack.pop();
    if (node === undefined) break;

    if (compare(key, node.value) === 0) return node;
    if (node.right !== null) stack.push(node.right);
    if (node.left !== null) stack.push(node.left);
  }

  return null;
}
```

#### Visualization
```text
search 10 without using order:
visit 8 -> 4 -> 2 -> 6 -> 12 -> 10

Both children remain possible after each nonmatching visit.
```

#### Dry Run
Starting at `8`, the stack-based traversal visits the entire left subtree before reaching `12` and then `10`. The comparison at `10` is zero, so the exact existing node is returned. For key `5`, every node is eventually examined and the function returns `null`.

#### Brute-Force Complexity
The traversal takes `O(n)` time in the worst case because it does not exploit ordering. Its DFS stack uses `O(h)` auxiliary space. The returned node reference uses `O(1)` output space, and the tree remains unchanged.

### 8. Why Brute Force Is Not Ideal
The baseline may inspect values that the BST invariant proves irrelevant. In a balanced tree, discarding one whole subtree per comparison reduces a linear scan to a logarithmic-height path.

### 9. Optimal Approach
Keep one `current` reference. Compare the query with `current.value`; return `current` on zero, move left on a negative result, and move right on a positive result. Stop at `null` when the key is absent.

### 10. Visualization
```text
search 10:

10 > 8   -> right
10 < 12  -> left
10 = 10  -> return that node reference

visited path: 8 -> 12 -> 10
```

### 11. Step-by-Step Dry Run
| Step | Current | Comparator sign for key `10` | Action |
|---:|---:|---:|---|
| 1 | 8 | positive | move right |
| 2 | 12 | negative | move left |
| 3 | 10 | zero | return current node |

An absent search for `5` follows `8 -> 4 -> 6 -> null` and returns `null`.

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function searchBst<T>(
  root: TreeNode<T> | null,
  key: T,
  compare: Comparator<T>,
): TreeNode<T> | null {
  let current = root;

  while (current !== null) {
    const relation = compare(key, current.value);
    if (relation === 0) return current;
    current = relation < 0 ? current.left : current.right;
  }

  return null;
}
```

### 13. Code Walkthrough
`current` is the only changing local reference. The comparator is called once per visited node, and only `relation < 0`, `relation === 0`, or the remaining positive case matters; no algorithm assumes the comparator returns exactly `-1` or `1`. Returning `current` preserves both the stored node identity and its value identity.

### 14. Complexity Analysis
The algorithm visits one root-to-node or root-to-null path, so time is `O(h)` and iterative auxiliary space is `O(1)`. In a height-balanced BST, `h = O(log n)`; in a skewed BST, `h = n - 1`, so worst-case time is `O(n)`. The result is one existing reference or `null`.

### 15. Edge Cases
- An empty tree returns `null` without calling the comparator.
- A comparator-equal root is returned immediately.
- A missing key follows a path to `null`.
- A one-node tree returns either that node or `null`.
- A skewed tree is handled iteratively without recursive stack growth.
- Distinct query and stored objects may compare equal; the stored node is returned.
- Large-magnitude comparator results are valid because only their sign is read.

### 16. Common Mistakes
- Using `===` instead of comparator equality.
- Searching both subtrees and losing the BST advantage.
- Returning the query value or a fresh node instead of the stored node.
- Mutating a node while performing a read-only lookup.
- Claiming unconditional `O(log n)` time for a plain BST.
- Branching on exact comparator outputs such as `-1` and `1`.

### 17. Interview Follow-ups
1. Return the search path. 2. Search recursively. 3. Find the first greater key when the exact key is absent. 4. Search an AVL or red-black tree. 5. Explain behavior under a stateful invalid comparator.

### 18. Variations
- Set membership. - Map lookup by key. - Lower bound. - Upper bound. - Search in a parent-linked BST.

### 19. Pattern Connections
Search is the basic comparator-guided descent reused by insertion and deletion. Candidate tracking extends the same path for successor, predecessor, floor, and ceiling queries, while binary search applies the same elimination idea to indexed arrays.

### 20. NestJS Project Connection
Expose `POST /api/dsa/bst/search` with a bounded tree payload and key. Use a fixed server-side comparator, validate proper finite shape, strict ordering, node count, and depth once at the boundary, then call only `searchBst`. Return an approved serialized stored value or stable node ID, never an in-memory reference. Cap optimized trace events, for example `{ relation, direction }`, and report `truncated` without serializing subtrees or sensitive keys.

### 21. Mini Challenge
Return `{ node, comparisons }`, where `node` has the same identity result as `searchBst` and `comparisons` is the exact number of visited nodes. Keep the operation read-only and `O(h)` time with `O(1)` auxiliary space.

## Problem 167 — Insert into BST

### 1. Problem Statement
Given the root of a caller-owned proper finite valid strict BST, a value, and the tree's comparator, insert the value when no stored value is comparator-equal to it. Mutate links in place, return the current root, and allocate at most one fresh `TreeNode`. Comparator-equal insertion is a complete no-op: return the original root, preserve every node and value identity, and allocate no node. Use only comparator-result signs.

### 2. Example
```text
before inserting 5:          after inserting 5:

       8                            8
      / \                          / \
     4  12                        4  12
    / \                          / \
   2   6                        2   6
                                  /
                                 5  <- one fresh node

insert another key equal to 6 -> no structural or value change
```

The caller must assign the result: `root = insertIntoBst(root, value, compare)`.

### 3. Understand the Problem
Insertion searches for the unique null child position determined by the comparator. Existing nodes are relinked only at that one parent field. The incoming value becomes the value of one fresh node only when its comparator-defined key is absent; an equal stored key is not updated or replaced.

#### In Simple Terms
Follow the key's search path until either it already exists or an empty child slot is found; fill only that empty slot.

### 4. Real-World Analogy
Adding a labeled folder to a sorted cabinet means following dividers to one empty slot. If a folder with the same catalog key already exists, you leave the cabinet and its original folder untouched.

### 5. Key Observation
The failed search path already identifies the only legal insertion parent and side. Delaying allocation until that null link is reached guarantees zero allocation for duplicates and exactly one new node for a successful insertion.

### 6. Pattern
```text
Pattern: Search Path Ending at a Null Link
Why this pattern applies: strict ordering determines one legal location
How to recognize it: lookup and update share the same comparator-guided path
Similar problems: BST search, deletion lookup, trie insertion, ordered-map put-if-absent
```

### 7. Brute Force
First traverse the whole tree without using BST ordering to determine whether any stored value is comparator-equal. If one exists, return the original root unchanged. Otherwise, perform a guided descent to the null child position and allocate one node there.

#### Pseudocode
```text
traverse every node:
    if any value compares equal: remember duplicate
if duplicate exists: return original root
if root is null: return one fresh node
follow comparator-guided path to a null child
attach one fresh node at that child
return original root
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function insertIntoBstAfterTraversal<T>(
  root: TreeNode<T> | null,
  value: T,
  compare: Comparator<T>,
): TreeNode<T> {
  if (root !== null) {
    const stack: TreeNode<T>[] = [root];
    let duplicateFound = false;

    while (stack.length > 0) {
      const node = stack.pop();
      if (node === undefined) break;
      if (compare(value, node.value) === 0) duplicateFound = true;
      if (node.right !== null) stack.push(node.right);
      if (node.left !== null) stack.push(node.left);
    }

    if (duplicateFound) return root;
  }

  const inserted: TreeNode<T> = { value, left: null, right: null };
  if (root === null) return inserted;

  let current = root;
  while (true) {
    if (compare(value, current.value) < 0) {
      if (current.left === null) {
        current.left = inserted;
        return root;
      }
      current = current.left;
    } else {
      if (current.right === null) {
        current.right = inserted;
        return root;
      }
      current = current.right;
    }
  }
}
```

#### Visualization
```text
phase 1: scan 8, 4, 2, 6, 12 to prove key 5 is absent
phase 2: 5 < 8 -> left; 5 > 4 -> right; 5 < 6 -> left
attach: 6.left = fresh node(5)
```

#### Dry Run
The complete traversal compares `5` with every stored value and finds no equality. Guided descent then moves from `8` to `4` to `6`; `6.left` is null, so one previously allocated node is attached there. For an incoming key equal to `6`, the scan finishes, reports a duplicate, returns the original root, and allocates nothing.

#### Brute-Force Complexity
The duplicate scan takes `O(n)` time and `O(h)` DFS-stack space; the later descent takes `O(h)` time, so total time is `O(n)`. A successful call allocates exactly one output node, while a duplicate call allocates no node. Auxiliary storage excludes that required output node.

### 8. Why Brute Force Is Not Ideal
The full duplicate scan examines nodes outside the only possible search path, then repeats comparisons during insertion. One descent can detect equality and locate the empty link at the same time.

### 9. Optimal Approach
Handle an empty root by returning one fresh node. Otherwise, walk from the root. Return the original root immediately on comparator equality. For a negative or positive result, inspect the corresponding child; attach one fresh leaf if it is null, or continue into it if it exists.

### 10. Visualization
```text
insert 5:

5 < 8 -> left
5 > 4 -> right
5 < 6 -> left is null

       4
        \
         6
        /
       5  new leaf
```

### 11. Step-by-Step Dry Run
| Step | Current | Sign for value `5` | Result |
|---:|---:|---:|---|
| 1 | 8 | negative | continue left |
| 2 | 4 | positive | continue right |
| 3 | 6 | negative | `left` is null; attach fresh node |

If the value compares equal at any step, the function returns before constructing a node or changing a link.

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function insertIntoBst<T>(
  root: TreeNode<T> | null,
  value: T,
  compare: Comparator<T>,
): TreeNode<T> {
  if (root === null) {
    return { value, left: null, right: null };
  }

  let current = root;
  while (true) {
    const relation = compare(value, current.value);
    if (relation === 0) return root;

    if (relation < 0) {
      if (current.left === null) {
        current.left = { value, left: null, right: null };
        return root;
      }
      current = current.left;
    } else {
      if (current.right === null) {
        current.right = { value, left: null, right: null };
        return root;
      }
      current = current.right;
    }
  }
}
```

### 13. Code Walkthrough
The empty case creates the new root. In the nonempty case, `relation` is computed once per visited node. Equality returns before allocation, enforcing the no-op contract. A successful path allocates directly into exactly one null child field and returns the unchanged root reference; all preexisting nodes and values keep their identities.

### 14. Complexity Analysis
One path of length at most `h + 1` is visited, so time is `O(h)` and iterative auxiliary space is `O(1)`. A balanced tree gives `O(log n)` time; sorted insertion into a plain BST can create a skewed tree and make a later insertion `O(n)`. Successful insertion allocates one required output node; duplicate insertion allocates none.

### 15. Edge Cases
- Inserting into an empty tree returns one fresh root node.
- A comparator-equal root or descendant makes the entire operation a no-op.
- The incoming object is stored by identity when insertion succeeds.
- The existing stored object is not replaced when another object has an equal key.
- Insertion may attach at a left or right leaf position.
- A skewed valid BST works without recursive stack overflow.
- The caller must assign the returned root, especially for an initially empty tree.

### 16. Common Mistakes
- Allocating a candidate node before duplicate detection.
- Replacing an equal node's value with the incoming value.
- Allowing equal keys consistently on one side and violating the strict invariant.
- Forgetting to return or reassign the root.
- Comparing by `===` instead of the comparator.
- Claiming insertion keeps a plain BST balanced.
- Testing for exact comparator outputs rather than their sign.

### 17. Interview Follow-ups
1. Return both root and inserted node. 2. Report whether insertion occurred. 3. Implement recursive insertion. 4. Add AVL rotations. 5. Make a persistent immutable version with path copying.

### 18. Variations
- Upsert with an explicit replacement policy. - Multiset insertion. - Parent-pointer insertion. - Randomized treap insertion. - Bulk construction from sorted values.

### 19. Pattern Connections
Insertion combines search with a link update at the failed-search boundary. The same put-if-absent idea appears in hash maps and tries, while balanced search trees add rotations after the ordinary BST attachment.

### 20. NestJS Project Connection
Expose `POST /api/dsa/bst/insert` only over a request-scoped or deliberately synchronized tree. Use a fixed server-side comparator and validate the bounded proper finite strict BST before mutation. Call only `insertIntoBst`, assign its returned root, and return a bounded preview plus `{ inserted, trace, truncated }`. Cap optimized comparison and final-link events; never accept executable comparator logic or emit full subtrees in traces.

### 21. Mini Challenge
Return `{ root, node, inserted }` in one descent. On a duplicate, `node` must be the existing stored node and `inserted` must be false; on success, `node` must be the one fresh node and `inserted` must be true.

## Problem 168 — Delete Node from BST

### 1. Problem Statement
Given the root of a caller-owned proper finite valid strict BST, a key, and the tree's comparator, remove the stored comparator-equal node if present. Mutate links in place and return the current root. Allocate no tree nodes, preserve every surviving node identity and value identity, and never copy a successor value. For a two-child target, transplant the existing inorder-successor node. Detach the deleted target by setting both of its links to `null`. An absent key leaves the tree unchanged. Use only comparator-result signs.

### 2. Example
```text
before deleting 8:           after deleting 8:

       8 target                    10 successor identity moved
      / \                          / \
     4  12                        4  12
       /  \                         \
      10  14                         14

target.left === null
target.right === null
```

The returned root is the existing node that contained `10`, not the old root with a copied value.

### 3. Understand the Problem
Search determines the target and its parent. Zero- and one-child deletion bypasses the target with its only child or `null`. Two-child deletion removes the smallest existing node from the target's right subtree and moves that exact node into the target's position. Only links change; surviving values never move between nodes.

#### In Simple Terms
Find the node, reconnect the tree around it, move the existing next-larger node when necessary, and leave the removed node fully disconnected.

### 4. Real-World Analogy
Removing a manager from an organization chart requires reconnecting reporting lines. If both divisions exist, the next manager in order takes the position as the same person, rather than merely copying that person's name onto the departing manager's badge.

### 5. Key Observation
The inorder successor has no left child. It can therefore be removed from its old location by replacing it with its right child, then safely receive the target's left and right subtrees. Special handling is required when the successor is already the target's immediate right child.

### 6. Pattern
```text
Pattern: Guided Search plus Structural Transplant
Why this pattern applies: lookup follows order, while deletion rewires one local region
How to recognize it: node identity must survive a two-child replacement
Similar problems: linked-list removal, heap extraction, tree rotations, parent-pointer updates
```

### 7. Brute Force
Ignore BST ordering while locating the comparator-equal target: perform DFS and carry each node's parent. Once found, apply the same identity-preserving structural cases. This baseline may inspect the whole tree even though ordering could identify one path.

#### Pseudocode
```text
DFS over (node, parent) pairs until target is found
if absent: return original root
if target has at most one child: replacement = existing child or null
otherwise:
    find existing minimum node in target.right
    unlink successor from its old parent
    attach target's children to successor
    replacement = successor
replace parent's target link, or replace root
set target.left and target.right to null
return root
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function deleteFromBstAfterTraversal<T>(
  root: TreeNode<T> | null,
  key: T,
  compare: Comparator<T>,
): TreeNode<T> | null {
  if (root === null) return null;

  const nodes: TreeNode<T>[] = [root];
  const parents: Array<TreeNode<T> | null> = [null];
  let target: TreeNode<T> | null = null;
  let parent: TreeNode<T> | null = null;

  while (nodes.length > 0) {
    const node = nodes.pop();
    const nodeParent = parents.pop();
    if (node === undefined || nodeParent === undefined) break;

    if (compare(key, node.value) === 0) {
      target = node;
      parent = nodeParent;
      break;
    }

    if (node.right !== null) {
      nodes.push(node.right);
      parents.push(node);
    }
    if (node.left !== null) {
      nodes.push(node.left);
      parents.push(node);
    }
  }

  if (target === null) return root;

  let replacement: TreeNode<T> | null;
  if (target.left === null) {
    replacement = target.right;
  } else if (target.right === null) {
    replacement = target.left;
  } else {
    let successorParent = target;
    let successor = target.right;

    while (successor.left !== null) {
      successorParent = successor;
      successor = successor.left;
    }

    if (successorParent !== target) {
      successorParent.left = successor.right;
      successor.right = target.right;
    }
    successor.left = target.left;
    replacement = successor;
  }

  if (parent === null) {
    root = replacement;
  } else if (parent.left === target) {
    parent.left = replacement;
  } else {
    parent.right = replacement;
  }

  target.left = null;
  target.right = null;
  return root;
}
```

#### Visualization
```text
target 8 has two children; successor is 10

1. unlink successor: 12.left = 10.right
2. successor.right = target.right
3. successor.left = target.left
4. root = successor
5. target.left = target.right = null
```

#### Dry Run
DFS finds target `8` with no parent. Its successor is `10`, whose parent is `12`. The algorithm replaces `12.left` with `10.right`, links `10.right` to `12` and `10.left` to `4`, and makes `10` the returned root. Finally, both links on the old `8` node are nulled. No value assignment or node construction occurs.

#### Brute-Force Complexity
Unordered target search takes `O(n)` time in the worst case and `O(h)` DFS-stack space. Successor descent adds at most `O(h)` time, so total time remains `O(n)`. No tree nodes are allocated; the arrays are auxiliary traversal storage. The removed target is detached, and surviving nodes are reused.

### 8. Why Brute Force Is Not Ideal
The baseline searches branches that comparator ordering rules out. Guided descent can find both the target and its parent in `O(h)` time while retaining the same transplant logic and identity guarantees.

### 9. Optimal Approach
Iteratively descend by comparator sign while tracking `parent`. If the key is absent, return the original root. Choose a replacement for the found target: its existing child for zero or one child, or the existing inorder successor for two children. Relink the parent or root to that replacement, then null both target links.

### 10. Visualization
```text
two-child case when successor is deeper:

       target                   successor
       /    \                   /       \
      L      R       ->        L         R
            /                            /
       successor                 successor's old right child
          \
           S

The same successor node moves; no value is copied.
```

### 11. Step-by-Step Dry Run
| Step | Identity operation |
|---:|---|
| 1 | Guided search finds target node `8` and `parent = null` |
| 2 | Left descent in the right subtree finds successor node `10` |
| 3 | `12.left` receives successor's old right child |
| 4 | Successor node `10` receives old subtrees rooted at `4` and `12` |
| 5 | Returned root becomes node `10`; old node `8` gets two null links |

When the successor is the immediate right child, its existing right link stays in place and only its left link is set to the target's left subtree.

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function deleteFromBst<T>(
  root: TreeNode<T> | null,
  key: T,
  compare: Comparator<T>,
): TreeNode<T> | null {
  let parent: TreeNode<T> | null = null;
  let target = root;

  while (target !== null) {
    const relation = compare(key, target.value);
    if (relation === 0) break;
    parent = target;
    target = relation < 0 ? target.left : target.right;
  }

  if (target === null) return root;

  let replacement: TreeNode<T> | null;
  if (target.left === null) {
    replacement = target.right;
  } else if (target.right === null) {
    replacement = target.left;
  } else {
    let successorParent = target;
    let successor = target.right;

    while (successor.left !== null) {
      successorParent = successor;
      successor = successor.left;
    }

    if (successorParent !== target) {
      successorParent.left = successor.right;
      successor.right = target.right;
    }
    successor.left = target.left;
    replacement = successor;
  }

  if (parent === null) {
    root = replacement;
  } else if (parent.left === target) {
    parent.left = replacement;
  } else {
    parent.right = replacement;
  }

  target.left = null;
  target.right = null;
  return root;
}
```

### 13. Code Walkthrough
The first loop uses only comparator signs and keeps the exact parent reference needed for relinking. The two-child branch finds the leftmost node in the right subtree. If that successor is deeper, its parent adopts the successor's right child before the successor adopts the target's right subtree; this order avoids losing links. No statement assigns to `.value`, and the final two assignments make the removed target observably detached.

### 14. Complexity Analysis
Target search and successor search each follow a downward path, so total time is `O(h)` and iterative auxiliary space is `O(1)`. This is `O(log n)` for a height-balanced tree and `O(n)` for a skewed tree. The algorithm allocates no tree nodes and preserves all surviving node and value identities.

### 15. Edge Cases
- An empty tree or absent key returns the original root unchanged.
- Deleting the sole node returns `null` and detaches that node.
- A leaf is replaced by `null`.
- A one-child target is replaced by its exact existing child.
- The root may change, so the caller must assign the return value.
- The successor may be the immediate right child or a deeper descendant.
- The successor may have a right child, which must be adopted by its old parent.
- Distinct query and stored objects may compare equal; the stored node is deleted.

### 16. Common Mistakes
- Copying `successor.value` into the target and breaking the identity contract.
- Allocating a replacement node.
- Forgetting to reconnect the successor's right child at its old location.
- Creating a self-link when the successor is the immediate right child.
- Failing to null both links on the removed target.
- Forgetting to assign the returned root.
- Using `===` or exact `-1` and `1` comparator checks.

### 17. Interview Follow-ups
1. Return the detached node with the root. 2. Delete recursively with the same transplant contract. 3. Delete by exact node identity. 4. Maintain parent pointers. 5. Rebalance an AVL tree after deletion.

### 18. Variations
- Delete minimum. - Delete maximum. - Persistent deletion with path copying. - Multiset deletion. - Lazy tombstone deletion.

### 19. Pattern Connections
Deletion is ordered search followed by local pointer surgery. Its successor extraction is a specialized minimum query, and its relinking discipline mirrors linked-list deletion and tree rotations where object identity matters independently of stored values.

### 20. NestJS Project Connection
Expose `POST /api/dsa/bst/delete` only with request-scoped state or an explicit persistence and concurrency design. Use a fixed comparator; validate bounded proper finite shape and strict ordering before mutation; call only `deleteFromBst`; and assign its returned root. Return stable IDs or a bounded preview with `{ deleted, trace, truncated }`. Cap optimized comparison and relink events, and never serialize detached object graphs or accept client code as a comparator.

### 21. Mini Challenge
Change the result to `{ root, deleted }`, where `deleted` is the exact detached target reference or `null`. Preserve `O(h)` time, `O(1)` auxiliary space, zero node allocation, successor transplantation, and the two-null-links postcondition.

## Problem 169 — Minimum and Maximum in BST

### 1. Problem Statement
Given the root of a caller-owned proper finite valid strict BST, return both extreme stored node references as `{ minimum, maximum }`. For an empty tree, return `{ minimum: null, maximum: null }`. For a nonempty tree, both fields must reference existing nodes. The operation is read-only: do not mutate links or values and do not clone nodes. The BST's strict ordering is defined by its comparator.

### 2. Example
```text
          8
        /   \
       4    12
      / \   / \
     2   6 10 14

minimum -> reference to existing node 2
maximum -> reference to existing node 14
```

A one-node tree returns that same node identity in both fields.

### 3. Understand the Problem
The minimum and maximum are nodes, not copied values. In a valid BST, every left step decreases the key and every right step increases it. The empty result must still have both named fields so callers do not need separate result shapes.

#### In Simple Terms
Walk left as far as possible for the minimum and right as far as possible for the maximum.

### 4. Real-World Analogy
In an alphabetically arranged directory tree, the first entry is reached by repeatedly opening the earliest section, while the last entry is reached by repeatedly opening the latest section. You return the original records at both endpoints.

### 5. Key Observation
No node below a missing left link can be smaller than that endpoint, and no node below a missing right link can be larger than its endpoint. Therefore two boundary paths replace a whole-tree comparison scan.

### 6. Pattern
```text
Pattern: Extreme Boundary Descent
Why this pattern applies: BST structure places extremes on outermost paths
How to recognize it: request for minimum or maximum stored key
Similar problems: successor, predecessor, floor, ceiling, delete-minimum
```

### 7. Brute Force
Traverse every node and use the comparator to track the smallest and largest nodes seen so far. This works for any proper finite binary tree whose values are comparable, but it ignores the BST's structural ordering.

#### Pseudocode
```text
if root is null: return { minimum: null, maximum: null }
minimum = root
maximum = root
traverse every node:
    if node.value compares below minimum.value: minimum = node
    if node.value compares above maximum.value: maximum = node
return both existing references
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

interface BstExtremes<T> {
  minimum: TreeNode<T> | null;
  maximum: TreeNode<T> | null;
}

function findBstExtremesByTraversal<T>(
  root: TreeNode<T> | null,
  compare: Comparator<T>,
): BstExtremes<T> {
  if (root === null) return { minimum: null, maximum: null };

  let minimum = root;
  let maximum = root;
  const stack: TreeNode<T>[] = [root];

  while (stack.length > 0) {
    const node = stack.pop();
    if (node === undefined) break;

    if (compare(node.value, minimum.value) < 0) minimum = node;
    if (compare(node.value, maximum.value) > 0) maximum = node;
    if (node.right !== null) stack.push(node.right);
    if (node.left !== null) stack.push(node.left);
  }

  return { minimum, maximum };
}
```

#### Visualization
```text
full traversal candidates:

visit 8  -> min 8, max 8
visit 4  -> min 4, max 8
visit 2  -> min 2, max 8
...
visit 14 -> min 2, max 14
```

#### Dry Run
Both candidates begin at root `8`. Visiting `4` and `2` moves only the minimum candidate; visiting `12` and `14` moves only the maximum candidate. Nodes `6` and `10` change neither. The returned references are the original nodes `2` and `14`.

#### Brute-Force Complexity
Every node is visited, so time is `O(n)` regardless of balance. Iterative DFS uses `O(h)` auxiliary space. The result object contains two existing node references and uses `O(1)` output space; no tree node is allocated or mutated.

### 8. Why Brute Force Is Not Ideal
The baseline compares interior nodes even though the BST invariant places the minimum on the all-left boundary and the maximum on the all-right boundary. The structure makes most of the tree irrelevant.

### 9. Optimal Approach
Return two nulls for an empty root. Otherwise, start both candidates at the root. Move the minimum candidate left until no left child exists, and independently move the maximum candidate right until no right child exists. No comparator call is needed after the boundary has validated that the input is a strict BST under its fixed comparator.

### 10. Visualization
```text
minimum path: 8 -> 4 -> 2 -> null left
maximum path: 8 -> 12 -> 14 -> null right

Only the two outer paths are visited.
```

### 11. Step-by-Step Dry Run
| Query | Start | Move 1 | Move 2 | Returned identity |
|---|---:|---:|---:|---:|
| minimum | 8 | 4 | 2 | existing node `2` |
| maximum | 8 | 12 | 14 | existing node `14` |

For a single root with no children, neither loop runs and both fields receive the same root reference.

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

interface BstExtremes<T> {
  minimum: TreeNode<T> | null;
  maximum: TreeNode<T> | null;
}

function findBstExtremes<T>(root: TreeNode<T> | null): BstExtremes<T> {
  if (root === null) return { minimum: null, maximum: null };

  let minimum = root;
  while (minimum.left !== null) minimum = minimum.left;

  let maximum = root;
  while (maximum.right !== null) maximum = maximum.right;

  return { minimum, maximum };
}
```

### 13. Code Walkthrough
The empty branch produces the required stable object shape. For a nonempty tree, `minimum` and `maximum` begin as non-null node references, so each loop remains type-safe under strict TypeScript. The function only reads child links and returns the endpoint nodes themselves. Comparator calls are unnecessary because the already-established BST invariant gives child links their ordering meaning.

### 14. Complexity Analysis
The two walks visit at most the left and right boundary heights, whose sum is `O(h)`. Auxiliary space is `O(1)`, and the two-reference result is `O(1)` output. A balanced tree gives `O(log n)` time; a skewed tree gives `O(n)` worst-case time. The operation allocates no tree nodes and changes nothing in the tree.

### 15. Edge Cases
- An empty tree returns exactly two null fields.
- A one-node tree returns the same existing node in both fields.
- A left-skewed tree makes the root the maximum.
- A right-skewed tree makes the root the minimum.
- Object values remain untouched and preserve identity.
- The function assumes strict ordering and proper finite shape were already validated.
- No numeric sentinel is needed for generic values.

### 16. Common Mistakes
- Returning values instead of node references.
- Returning `undefined` or a differently shaped object for an empty tree.
- Traversing the entire tree despite the valid BST precondition.
- Using numeric `Infinity` sentinels for generic `T`.
- Mutating child links while walking.
- Claiming unconditional `O(log n)` time.
- Calling a comparator during boundary descent as if links were not already ordered.

### 17. Interview Follow-ups
1. Return only minimum or only maximum. 2. Delete and return the minimum node. 3. Find floor and ceiling for a query. 4. Support cached extreme references. 5. Explain updates to extremes during insertion and deletion.

### 18. Variations
- Minimum in a rotated array. - Heap minimum or maximum. - Successor and predecessor. - Range endpoints. - Extreme nodes in an augmented BST.

### 19. Pattern Connections
Minimum and maximum are the simplest boundary descents. Successor and predecessor combine these descents with ancestor candidates, while deletion reuses right-subtree minimum to select an inorder successor.

### 20. NestJS Project Connection
Expose `POST /api/dsa/bst/extremes` with a bounded tree payload. Apply a fixed server-side comparator and validate proper finite shape, strict ordering, node count, and depth once before calling `findBstExtremes`. Return approved serialized keys or stable IDs as `{ minimum, maximum, trace, truncated }`. Trace only capped left/right steps, do not log arbitrary values, and do not rerun whole-tree validation inside the optimized function.

### 21. Mini Challenge
Return the minimum, maximum, and their root-relative depths in one result. Keep empty fields null, preserve node identities, use `O(h)` time and `O(1)` auxiliary space, and do not call the comparator after validation.

## Problem 170 — Inorder Successor

### 1. Problem Statement
Given the root of a caller-owned proper finite valid strict BST, a query key, and the tree's comparator, return the existing node reference for the query's strict inorder successor. Return `null` if no comparator-equal key is present or if the present key has no successor. The operation is read-only. Values may be distinct objects that compare equal, so use only comparator-result signs and never `===` for key equality.

### 2. Example
```text
          8
        /   \
       4    12
      / \   / \
     2   6 10 14

successor of 6  -> existing node 8
successor of 8  -> existing node 10
successor of 14 -> null
successor of 5  -> null because key 5 is absent
```

With records compared by `id`, a separate query object with the same `id` still finds the stored key and returns the stored successor node.

### 3. Understand the Problem
The successor is the first node after the stored comparator-equal key in inorder order, so it must be strictly greater. A candidate seen during descent is not enough to prove the query exists. The function may return that candidate only after finding a comparator-equal stored node.

#### In Simple Terms
Find the key first. If it has a right subtree, return that subtree's leftmost node; otherwise, return the closest larger ancestor remembered on the way down.

### 4. Real-World Analogy
In an ordered filing tree, the next record after a requested ID is either the first record in its later subsection or the nearest earlier folder whose label is larger. If the requested ID is not filed at all, do not return a nearby record.

### 5. Key Observation
For a found node, any key in its right subtree is greater, and the smallest such key is the right subtree's leftmost node. Without a right subtree, the successor is the lowest ancestor for which the search entered the ancestor's left subtree. Candidate tracking remembers exactly that ancestor.

### 6. Pattern
```text
Pattern: Comparator-Guided Search with Successor Candidate
Why this pattern applies: the BST narrows search while left turns reveal larger ancestors
How to recognize it: request for the next strict stored key after a present query key
Similar problems: predecessor, ceiling, range iteration, BST deletion
```

### 7. Brute Force
Perform a complete inorder traversal into an array of existing node references. Because the input is a valid strict BST, that array is strictly increasing. Scan it for a comparator-equal value and return the following node when one exists.

#### Pseudocode
```text
nodes = full inorder traversal
for each index:
    if key compares equal to nodes[index].value:
        return nodes[index + 1] when present, otherwise null
return null
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function findInorderSuccessorByTraversal<T>(
  root: TreeNode<T> | null,
  key: T,
  compare: Comparator<T>,
): TreeNode<T> | null {
  const nodes: TreeNode<T>[] = [];
  const stack: TreeNode<T>[] = [];
  let current = root;

  while (current !== null || stack.length > 0) {
    while (current !== null) {
      stack.push(current);
      current = current.left;
    }

    const node = stack.pop();
    if (node === undefined) break;
    nodes.push(node);
    current = node.right;
  }

  for (let index = 0; index < nodes.length; index += 1) {
    if (compare(key, nodes[index].value) === 0) {
      return index + 1 < nodes.length ? nodes[index + 1] : null;
    }
  }

  return null;
}
```

#### Visualization
```text
full inorder nodes: [2, 4, 6, 8, 10, 12, 14]
query 6: found at index 2 -> return node at index 3
query 5: no comparator-equal entry -> null
```

#### Dry Run
The traversal stores all seven existing node references. For query `6`, the scan finds equality at index `2` and returns node `8`. For query `14`, equality occurs at the last index, so the result is `null`. For query `5`, no equality occurs, so a nearby larger node is never mistaken for a successor.

#### Brute-Force Complexity
The full inorder traversal and scan take `O(n)` time. The node array uses `O(n)` output-temporary space and the traversal stack uses `O(h)`, for `O(n + h)` auxiliary space overall. Returned nodes preserve identity, and the tree is not changed.

### 8. Why Brute Force Is Not Ideal
The baseline visits and stores every node even though comparator-guided descent needs only one root-to-key path and, at most, one left-boundary path inside the right subtree.

### 9. Optimal Approach
Search iteratively for the comparator-equal key. When the key is smaller than the current node, remember the current node as the best larger ancestor and move left. When it is larger, move right. On equality, return the leftmost node of the right subtree if that subtree exists; otherwise return the remembered candidate. If descent reaches `null` before equality, return `null`, not the candidate.

### 10. Visualization
```text
successor of 6:
8 is larger candidate -> left to 4 -> right to 6 -> found
6 has no right subtree -> return candidate 8

successor of 8:
8 is found and has right subtree rooted at 12
leftmost path 12 -> 10 -> return 10
```

### 11. Step-by-Step Dry Run
| Query | Search path | Candidate before equality | Subtree work | Result |
|---|---|---|---|---|
| `6` | `8 -> 4 -> 6` | `8` | none | node `8` |
| `8` | `8` | `null` | `12 -> 10` | node `10` |
| `14` | `8 -> 12 -> 14` | `null` | none | `null` |
| `5` | `8 -> 4 -> 6 -> null` | `6` | key never found | `null` |

The absent-key row is essential: node `6` is a ceiling candidate for `5`, but it is not the successor of a stored key `5` because that key is absent.

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function findInorderSuccessor<T>(
  root: TreeNode<T> | null,
  key: T,
  compare: Comparator<T>,
): TreeNode<T> | null {
  let current = root;
  let candidate: TreeNode<T> | null = null;

  while (current !== null) {
    const relation = compare(key, current.value);

    if (relation < 0) {
      candidate = current;
      current = current.left;
    } else if (relation > 0) {
      current = current.right;
    } else {
      if (current.right === null) return candidate;

      let successor = current.right;
      while (successor.left !== null) successor = successor.left;
      return successor;
    }
  }

  return null;
}
```

### 13. Code Walkthrough
`candidate` changes only on a left move, when the current node is known to be larger than the query. Equality opens the only return path for that candidate. A right subtree takes priority because all of its nodes are below any larger ancestor candidate, and its leftmost node is the smallest of them. Falling out of the search loop proves absence and returns `null`.

### 14. Complexity Analysis
Search plus right-subtree minimum follows at most `O(h)` downward edges, so time is `O(h)` and iterative auxiliary space is `O(1)`. This is `O(log n)` on a height-balanced tree and `O(n)` on a skewed tree. The result is one existing node reference or `null`.

### 15. Edge Cases
- An empty tree returns `null`.
- An absent key returns `null` even when a larger candidate was tracked.
- The maximum stored key has no successor.
- A found node's immediate right child may be its successor.
- A deeper left descendant of the right child may be its successor.
- Distinct objects may compare equal; identity equality is irrelevant.
- The caller-owned tree and all stored values remain unchanged.

### 16. Common Mistakes
- Returning the candidate after an unsuccessful search.
- Returning the right child without finding its leftmost descendant.
- Forgetting the ancestor candidate when the found node has no right subtree.
- Using `===` to match object keys.
- Checking for exact comparator outputs such as `-1` or `1`.
- Returning a copied value instead of the existing node reference.
- Claiming unconditional `O(log n)` time.

### 17. Interview Follow-ups
1. Return the ceiling even when the key is absent. 2. Find the successor from a node with parent pointers. 3. Find both predecessor and successor in one search. 4. Iterate through a key range. 5. Support duplicate counts while preserving strict structural keys.

### 18. Variations
- Inorder predecessor. - Ceiling by query key. - Successor with parent pointers. - Next larger key in an augmented tree. - Range successor iterator.

### 19. Pattern Connections
Successor combines comparator-guided search, ancestor candidate tracking, and subtree minimum. The same right-subtree minimum also supplies the replacement position used by BST deletion, while ceiling differs by allowing a result when the query is absent.

### 20. NestJS Project Connection
Expose `POST /api/dsa/bst/successor` with a bounded tree and query DTO. Use a fixed server-side comparator; validate proper finite shape, strict ordering, node count, and depth at the boundary; then call only `findInorderSuccessor`. Return an approved key or stable ID plus a capped comparison trace. Do not accept comparator code, serialize arbitrary stored objects, or add whole-tree validation inside the optimized function.

### 21. Mini Challenge
Return `{ found, successor }` so callers can distinguish an absent key from a present maximum without another search. Preserve existing node identities, read-only behavior, `O(h)` time, and `O(1)` auxiliary space.

## Problem 171 — Inorder Predecessor

### 1. Problem Statement
Given the root of a caller-owned proper finite valid strict BST, a query key, and the tree's comparator, return the existing node reference for the query's strict inorder predecessor. Return `null` if no comparator-equal key is present or if the present key has no predecessor. The operation is read-only. Values may be distinct objects that compare equal, so use only comparator-result signs and never `===` for key equality.

### 2. Example
```text
          8
        /   \
       4    12
      / \   / \
     2   6 10 14

predecessor of 10 -> existing node 8
predecessor of 8  -> existing node 6
predecessor of 2  -> null
predecessor of 5  -> null because key 5 is absent
```

With records compared by `id`, a separate query object with the same `id` still finds the stored key and returns the stored predecessor node.

### 3. Understand the Problem
The predecessor is the node immediately before the stored comparator-equal key in inorder order, so it must be strictly smaller. A smaller candidate encountered during descent is not a valid result unless the query key is later found.

#### In Simple Terms
Find the key first. If it has a left subtree, return that subtree's rightmost node; otherwise, return the closest smaller ancestor remembered on the way down.

### 4. Real-World Analogy
In an ordered filing tree, the record immediately before a requested ID is either the last record in its earlier subsection or the nearest earlier folder whose label is smaller. If the requested ID is missing, a nearby record is not returned.

### 5. Key Observation
For a found node, any key in its left subtree is smaller, and the largest such key is the left subtree's rightmost node. Without a left subtree, the predecessor is the lowest ancestor for which the search entered the ancestor's right subtree. Candidate tracking remembers that ancestor.

### 6. Pattern
```text
Pattern: Comparator-Guided Search with Predecessor Candidate
Why this pattern applies: the BST narrows search while right turns reveal smaller ancestors
How to recognize it: request for the previous strict stored key before a present query key
Similar problems: successor, floor, range iteration, BST deletion
```

### 7. Brute Force
Perform a complete inorder traversal into an array of existing node references. Scan the strictly increasing array for a comparator-equal value and return the preceding node when one exists.

#### Pseudocode
```text
nodes = full inorder traversal
for each index:
    if key compares equal to nodes[index].value:
        return nodes[index - 1] when present, otherwise null
return null
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function findInorderPredecessorByTraversal<T>(
  root: TreeNode<T> | null,
  key: T,
  compare: Comparator<T>,
): TreeNode<T> | null {
  const nodes: TreeNode<T>[] = [];
  const stack: TreeNode<T>[] = [];
  let current = root;

  while (current !== null || stack.length > 0) {
    while (current !== null) {
      stack.push(current);
      current = current.left;
    }

    const node = stack.pop();
    if (node === undefined) break;
    nodes.push(node);
    current = node.right;
  }

  for (let index = 0; index < nodes.length; index += 1) {
    if (compare(key, nodes[index].value) === 0) {
      return index > 0 ? nodes[index - 1] : null;
    }
  }

  return null;
}
```

#### Visualization
```text
full inorder nodes: [2, 4, 6, 8, 10, 12, 14]
query 10: found at index 4 -> return node at index 3
query 5: no comparator-equal entry -> null
```

#### Dry Run
The traversal stores all seven existing node references. For query `10`, the scan finds equality at index `4` and returns node `8`. For query `2`, equality occurs at index `0`, so the result is `null`. For query `5`, the scan finds no equality and returns `null` rather than node `4`.

#### Brute-Force Complexity
The full inorder traversal and scan take `O(n)` time. The node array uses `O(n)` output-temporary space and the traversal stack uses `O(h)`, for `O(n + h)` auxiliary space overall. No node or value is copied or mutated.

### 8. Why Brute Force Is Not Ideal
The baseline materializes the complete sorted order even though the BST can locate the query and its predecessor using only downward paths and one stored ancestor candidate.

### 9. Optimal Approach
Search iteratively for the comparator-equal key. When the key is larger than the current node, remember the current node as the best smaller ancestor and move right. When it is smaller, move left. On equality, return the rightmost node of the left subtree if that subtree exists; otherwise return the remembered candidate. If descent reaches `null` before equality, return `null`.

### 10. Visualization
```text
predecessor of 10:
8 is smaller candidate -> right to 12 -> left to 10 -> found
10 has no left subtree -> return candidate 8

predecessor of 8:
8 is found and has left subtree rooted at 4
rightmost path 4 -> 6 -> return 6
```

### 11. Step-by-Step Dry Run
| Query | Search path | Candidate before equality | Subtree work | Result |
|---|---|---|---|---|
| `10` | `8 -> 12 -> 10` | `8` | none | node `8` |
| `8` | `8` | `null` | `4 -> 6` | node `6` |
| `2` | `8 -> 4 -> 2` | `null` | none | `null` |
| `5` | `8 -> 4 -> 6 -> null` | `4` | key never found | `null` |

The absent-key row shows why the final return cannot be `candidate`: node `4` is the floor of `5`, but no stored key `5` exists whose predecessor can be requested.

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function findInorderPredecessor<T>(
  root: TreeNode<T> | null,
  key: T,
  compare: Comparator<T>,
): TreeNode<T> | null {
  let current = root;
  let candidate: TreeNode<T> | null = null;

  while (current !== null) {
    const relation = compare(key, current.value);

    if (relation < 0) {
      current = current.left;
    } else if (relation > 0) {
      candidate = current;
      current = current.right;
    } else {
      if (current.left === null) return candidate;

      let predecessor = current.left;
      while (predecessor.right !== null) predecessor = predecessor.right;
      return predecessor;
    }
  }

  return null;
}
```

### 13. Code Walkthrough
`candidate` changes only on a right move, when the current node is strictly smaller than the query. The candidate can be returned only after comparator equality confirms presence. A left subtree takes priority over any ancestor candidate, and its rightmost node is that subtree's greatest key. Reaching `null` without equality discards any floor candidate.

### 14. Complexity Analysis
Search plus left-subtree maximum traverses `O(h)` downward edges in total, so time is `O(h)` and iterative auxiliary space is `O(1)`. This becomes `O(log n)` on a height-balanced tree and `O(n)` on a skewed tree. The function returns an existing node or `null` and allocates no tree nodes.

### 15. Edge Cases
- An empty tree returns `null`.
- An absent key returns `null` even when a smaller candidate was tracked.
- The minimum stored key has no predecessor.
- A found node's immediate left child may be its predecessor.
- A deeper right descendant of the left child may be its predecessor.
- Comparator-equal objects need not be identical references.
- The operation leaves all caller-owned links and values unchanged.

### 16. Common Mistakes
- Returning a floor candidate when the query is absent.
- Returning the left child without walking to its rightmost descendant.
- Losing the smaller ancestor candidate during right moves.
- Comparing generic values with `===`.
- Assuming comparator results are exactly `-1`, `0`, or `1`.
- Returning a newly allocated node instead of the stored reference.
- Ignoring the skewed-tree `O(n)` worst case.

### 17. Interview Follow-ups
1. Return the floor even for an absent key. 2. Use parent pointers. 3. Return predecessor and successor together. 4. Implement reverse inorder iteration. 5. Maintain predecessor links in a threaded BST.

### 18. Variations
- Inorder successor. - Floor by query key. - Predecessor with parent pointers. - Previous smaller key in an augmented tree. - Reverse range iterator.

### 19. Pattern Connections
Predecessor mirrors successor: reverse the comparison directions, ancestor candidate, and subtree extreme. It also reuses the maximum-boundary walk from Problem 169 and forms the reverse step used by descending BST iterators.

### 20. NestJS Project Connection
Expose `POST /api/dsa/bst/predecessor` with bounded tree and query DTOs. Select a fixed server-side comparator, validate proper finite shape and strict ordering once at the boundary, and cap node count, depth, and trace events. Return only an approved serialized key or stable ID. Do not accept executable comparator input or perform hidden whole-tree work inside `findInorderPredecessor`.

### 21. Mini Challenge
Write one function that returns `{ found, predecessor, successor }` in a single `O(h)` search with `O(1)` auxiliary space. Preserve existing references and return both neighbors as `null` when the key is absent.

## Problem 172 — Validate BST

### 1. Problem Statement
Given the root of any caller-owned proper finite binary tree and a comparator, return whether the tree is a valid strict BST. The input is not assumed to be a BST, an empty tree is valid, and comparator-equal duplicates must be rejected anywhere in the tree. The operation is read-only and must use only comparator-result signs.

### 2. Example
```text
valid:                    invalid deep placement:
      8                           8
    /   \                       /   \
   4    12                     4    12
       /                            /
      10                           6

inorder: 4, 8, 10, 12      node 6 is in 8's right subtree
result: true               result: false

empty tree -> true
```

A duplicate key represented by a distinct object is still invalid when the comparator returns zero.

### 3. Understand the Problem
Every node must satisfy restrictions inherited from all of its ancestors, not merely compare correctly with its parent. A left descendant receives an upper bound, a right descendant receives a lower bound, and strict comparisons reject equality at either boundary.

#### In Simple Terms
As you recurse, carry the allowed open range for each node. Reject any value that is not strictly inside that range.

### 4. Real-World Analogy
Imagine nested shelves labeled with exclusive lower and upper catalog limits. An item can fit its immediate shelf divider yet still violate a limit inherited from a shelf several levels above. Validation carries every relevant outer limit downward.

### 5. Key Observation
A valid strict BST's complete inorder sequence is strictly increasing, but storing that sequence is unnecessary. Bounds encode the same global rule locally: left recursion tightens the upper bound to the current value, and right recursion tightens the lower bound to it. Equality fails because both bounds are exclusive.

### 6. Pattern
```text
Pattern: Recursive Validation with Exclusive Bounds
Why this pattern applies: ancestor constraints define each subtree's legal key range
How to recognize it: a local parent check cannot catch deep ordering violations
Similar problems: validate search trees, range-constrained recursion, serialization checks
```

### 7. Brute Force
Traverse the entire tree inorder and store every existing node reference. Then require each adjacent pair to compare in strictly increasing order. A zero or positive adjacent comparison rejects duplicate or descending keys.

#### Pseudocode
```text
nodes = full inorder traversal
for index from 1 to nodes.length - 1:
    if previous.value does not compare strictly below current.value:
        return false
return true
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function isValidBstByInorder<T>(
  root: TreeNode<T> | null,
  compare: Comparator<T>,
): boolean {
  const nodes: TreeNode<T>[] = [];
  const stack: TreeNode<T>[] = [];
  let current = root;

  while (current !== null || stack.length > 0) {
    while (current !== null) {
      stack.push(current);
      current = current.left;
    }

    const node = stack.pop();
    if (node === undefined) break;
    nodes.push(node);
    current = node.right;
  }

  for (let index = 1; index < nodes.length; index += 1) {
    if (compare(nodes[index - 1].value, nodes[index].value) >= 0) {
      return false;
    }
  }

  return true;
}
```

#### Visualization
```text
valid inorder:   [4, 8, 10, 12] -> 4 < 8 < 10 < 12
duplicate:       [4, 8, 8, 12]  -> 8 !< 8
deep violation:  [4, 8, 6, 12]  -> 8 !< 6
```

#### Dry Run
For the valid tree, all three adjacent comparisons are negative, so validation returns `true`. In the deep-violation tree, inorder produces `[4, 8, 6, 12]`; comparing `8` with `6` is nonnegative, so it returns `false`. A duplicate produces an adjacent comparator result of zero and is also rejected.

#### Brute-Force Complexity
The traversal and adjacent scan take `O(n)` time. The full inorder list is an `O(n)` output-temporary, and the iterative traversal stack uses `O(h)` auxiliary space, for `O(n + h)` total auxiliary space. The empty list is accepted.

### 8. Why Brute Force Is Not Ideal
The baseline always stores all node references before checking order. Bounds can reject a violation during traversal and need only the recursive call stack plus the active lower and upper constraints.

### 9. Optimal Approach
Recursively validate each node against optional exclusive lower and upper bounds. A present lower bound requires `node.value` to compare greater than it; a present upper bound requires `node.value` to compare less than it. Recurse left with the current value as the upper bound and right with it as the lower bound. Use explicit bound objects rather than numeric sentinels so the algorithm remains generic.

### 10. Visualization
```text
validate node 8 in (-infinity, +infinity)
left node 4 receives upper bound 8
right node 12 receives lower bound 8
right-left node 6 receives open bounds (8, 12)
compare(6, 8) <= 0 -> reject deep violation

The words infinity describe absent bounds; no numeric sentinel is stored.
```

### 11. Step-by-Step Dry Run
| Node | Lower bound | Upper bound | Check | Outcome |
|---:|---:|---:|---|---|
| `8` | absent | absent | no bound comparison | continue |
| `12` | `8` | absent | `12 > 8` | continue |
| `6` | `8` | `12` | `6 <= 8` | return `false` |

For a comparator-equal duplicate `8` anywhere in the original root's right subtree, the lower-bound comparison returns zero and validation rejects it even if its immediate parent is larger.

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

type Bound<T> =
  | { present: false }
  | { present: true; value: T };

function isValidBst<T>(
  root: TreeNode<T> | null,
  compare: Comparator<T>,
): boolean {
  const absent: Bound<T> = { present: false };

  function validate(
    node: TreeNode<T> | null,
    lower: Bound<T>,
    upper: Bound<T>,
  ): boolean {
    if (node === null) return true;
    if (lower.present && compare(node.value, lower.value) <= 0) return false;
    if (upper.present && compare(node.value, upper.value) >= 0) return false;

    const currentBound: Bound<T> = { present: true, value: node.value };
    return (
      validate(node.left, lower, currentBound) &&
      validate(node.right, currentBound, upper)
    );
  }

  return validate(root, absent, absent);
}
```

### 13. Code Walkthrough
The discriminated `Bound<T>` type separates absence from every possible `T`, so values such as numbers, strings, objects, `null`, or `undefined` need no sentinel. Each node first checks inherited exclusive bounds. The current value then becomes the left subtree's upper bound and the right subtree's lower bound. Short-circuiting stops after the first violation, and no tree field is assigned.

### 14. Complexity Analysis
Each node is checked at most once, so time is `O(n)`. Recursive call depth and live bound objects are `O(h)` auxiliary space: `O(log n)` for a height-balanced tree and `O(n)` for a skewed tree. There is no traversal-array temporary. The empty tree returns `true` in `O(1)` time.

### 15. Edge Cases
- An empty tree is valid.
- A one-node tree is valid.
- Comparator-equal duplicates are invalid regardless of object identity or depth.
- A deep node may satisfy its parent relation yet violate an ancestor bound.
- Generic values require no numeric minimum or maximum sentinel.
- A highly skewed finite tree may reach the runtime's recursion limit.
- The caller-owned tree remains unchanged.
- Proper finite shape is a precondition; this function does not detect cycles or shared children.
- A malformed or stateful comparator is outside the contract and can make any ordering result unreliable.

### 16. Common Mistakes
- Checking only immediate parent-child relationships.
- Allowing equality on one bound and accidentally accepting duplicates.
- Using `-Infinity` and `Infinity` for generic `T`.
- Comparing values with `<`, `>`, or `===` instead of the comparator.
- Expecting comparator outputs to be exactly `-1`, `0`, and `1`.
- Sorting the inorder list, which destroys evidence of invalid order.
- Claiming the validator also checks cycles or shared-node shape.

### 17. Interview Follow-ups
1. Implement an iterative bounds validator. 2. Return the first invalid node and violated bound. 3. Validate a duplicate policy such as duplicates-only-on-right. 4. Guard against excessive depth. 5. Combine shape and ordering validation for untrusted input.

### 18. Variations
- Validate an inclusive-bound tree. - Validate an AVL tree. - Recover two swapped BST nodes. - Validate serialized BST input. - Validate with an inorder previous-value state.

### 19. Pattern Connections
Exclusive bounds express global ancestor constraints without materializing sorted output. The approach generalizes to range-constrained recursion, while the brute-force inorder theorem connects validation to sorted-sequence checks and the chapter's kth-smallest and neighbor operations.

### 20. NestJS Project Connection
Expose `POST /api/dsa/bst/validate` with a bounded binary-tree DTO and a fixed server-side comparator. Parse and separately validate proper finite shape, node count, and depth before calling `isValidBst`, because the algorithm checks ordering only. Return a boolean and, if needed, a capped sanitized trace of bound failures. Reject client-provided comparator code and avoid logging arbitrary values.

### 21. Mini Challenge
Return a discriminated result containing either `{ valid: true }` or `{ valid: false, node, violated: "lower" | "upper" }`. Keep the tree read-only, reject duplicates, preserve `O(n)` time and `O(h)` auxiliary space, and do not add numeric sentinels.

## Problem 173 — Lowest Common Ancestor in BST

### 1. Problem Statement
Given the root of a caller-owned proper finite valid strict BST, two query keys, and the tree's comparator, return the existing node reference for their lowest common ancestor. Return `null` unless comparator-equal stored nodes exist for both keys. If the two query keys compare equal and that key is present, return that stored node. The operation is read-only, different query objects may comparator-match stored values, and only comparator-result signs may determine ordering and key equality.

### 2. Example
```text
          8
        /   \
       4    12
      / \   / \
     2   6 10 14

keys 2 and 6  -> existing node 4
keys 2 and 12 -> existing node 8
keys 8 and 14 -> existing node 8
keys 6 and 6  -> existing node 6
keys 6 and 7  -> null because key 7 is absent
```

For records compared by `id`, query objects `{ id: 2, label: "query" }` and `{ id: 6, label: "other" }` can find stored values with those IDs even when none of the objects are identical.

### 3. Understand the Problem
The LCA is the deepest existing node whose subtree contains both stored target nodes. The query supplies keys rather than node references, so both keys must first be proven present under comparator equality. A node where the two search directions split is only a candidate until that presence requirement has been established.

#### In Simple Terms
First find both keys in the BST. If both exist, walk down again until the keys stop belonging strictly to the same side; that existing node is their LCA.

### 4. Real-World Analogy
In an ordered filing hierarchy, two requested record IDs share a deepest folder. The folder where their routes diverge is meaningful only if both records are actually filed; a plausible route for a missing record does not create a common folder result.

### 5. Key Observation
At a current node, if both stored target values compare smaller, their LCA must be left. If both compare larger, it must be right. Otherwise, the current node equals one target or lies at the split between their search paths, making it the LCA. Separate comparator-guided searches prove that both targets exist before this split logic can return anything.

### 6. Pattern
```text
Pattern: Presence-Proven Comparator-Guided Split Descent
Why this pattern applies: BST ordering puts both targets on one side until their paths split
How to recognize it: lowest shared ancestor of two keys in an ordered tree
Similar problems: BST search, range queries, binary-tree LCA, successor and predecessor
```

### 7. Brute Force
Build a root-to-key path of existing node references for each query using comparator-guided search. If either search reaches `null`, return `null`. Otherwise, scan the two paths from the root while their node identities are equal; the last shared node reference is the LCA.

#### Pseudocode
```text
firstPath = root-to-first-key path, or absent
secondPath = root-to-second-key path, or absent
if either path is absent: return null
lca = null
while nodes at the same path index are identical:
    lca = that existing node
return lca
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function lowestCommonAncestorByPaths<T>(
  root: TreeNode<T> | null,
  firstKey: T,
  secondKey: T,
  compare: Comparator<T>,
): TreeNode<T> | null {
  function pathToKey(key: T): TreeNode<T>[] | null {
    const path: TreeNode<T>[] = [];
    let current = root;

    while (current !== null) {
      path.push(current);
      const relation = compare(key, current.value);

      if (relation < 0) {
        current = current.left;
      } else if (relation > 0) {
        current = current.right;
      } else {
        return path;
      }
    }

    return null;
  }

  const firstPath = pathToKey(firstKey);
  if (firstPath === null) return null;

  const secondPath = pathToKey(secondKey);
  if (secondPath === null) return null;

  let lca: TreeNode<T> | null = null;
  const sharedLength = Math.min(firstPath.length, secondPath.length);

  for (let index = 0; index < sharedLength; index += 1) {
    const firstNode = firstPath[index];
    const secondNode = secondPath[index];
    if (firstNode === undefined || secondNode === undefined) break;
    if (firstNode !== secondNode) break;
    lca = firstNode;
  }

  return lca;
}
```

#### Visualization
```text
path to 2:  [8, 4, 2]
path to 6:  [8, 4, 6]
same nodes:  8, 4
first identity mismatch at final position -> return existing node 4

path to absent 7: no path -> return null
```

#### Dry Run
For keys `2` and `6`, comparator-guided searches produce paths `[8, 4, 2]` and `[8, 4, 6]`. Identity comparison keeps node `8`, then node `4`, and stops when nodes `2` and `6` differ, so the result is the existing node `4`. For two queries comparator-equal to stored key `6`, both paths end at the same stored node and the result is node `6`. If key `7` is queried, its path search reaches `null`, so the function returns `null` without treating node `6` or `8` as an LCA.

#### Brute-Force Complexity
Each path search takes `O(h)` time and comparing the paths takes `O(h)`, so total time is `O(h)`. The two retained path arrays use `O(h)` auxiliary space. No node or stored value is copied or mutated, and the result is an existing node reference.

### 8. Why Brute Force Is Not Ideal
Although path construction already has optimal `O(h)` asymptotic time on a valid BST, it retains two `O(h)` arrays solely to rediscover their common prefix. Repeated comparator-guided descents can prove presence and find the split while retaining only a few node references.

### 9. Optimal Approach
Search independently for each comparator-equal stored node. If either search fails, return `null`; this is necessary because a naive split candidate can exist even when one or both targets are absent. Once both stored nodes are known, start at the root: move left while both values compare smaller, move right while both compare larger, and otherwise return the current existing node. If both query keys compare equal, both searches find the same unique stored node and the split descent returns it.

### 10. Visualization
```text
present targets 2 and 6:
find 2 -> node 2
find 6 -> node 6
at 8: both smaller -> left
at 4: 2 is smaller, 6 is larger -> split -> return node 4

targets 6 and absent 7:
find 6 -> node 6
find 7 -> null
return null before considering a split candidate
```

### 11. Step-by-Step Dry Run
| Queries | Presence searches | Split descent | Result |
|---|---|---|---|
| `2`, `6` | both found | `8: left`, `4: split` | node `4` |
| `8`, `14` | both found | `8: first target equal` | node `8` |
| `6`, `6` | both resolve to stored node `6` | `8: left`, `4: right`, `6: equal` | node `6` |
| `6`, `7` | `6` found, `7` absent | not started | `null` |

The final row shows why descending only to where the query directions split is insufficient: absent keys still define comparison directions, but they do not satisfy the contract's stored-target requirement.

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function lowestCommonAncestor<T>(
  root: TreeNode<T> | null,
  firstKey: T,
  secondKey: T,
  compare: Comparator<T>,
): TreeNode<T> | null {
  function findNode(key: T): TreeNode<T> | null {
    let current = root;

    while (current !== null) {
      const relation = compare(key, current.value);
      if (relation < 0) {
        current = current.left;
      } else if (relation > 0) {
        current = current.right;
      } else {
        return current;
      }
    }

    return null;
  }

  const firstNode = findNode(firstKey);
  if (firstNode === null) return null;

  const secondNode = findNode(secondKey);
  if (secondNode === null) return null;

  let current = root;
  while (current !== null) {
    const firstRelation = compare(firstNode.value, current.value);
    const secondRelation = compare(secondNode.value, current.value);

    if (firstRelation < 0 && secondRelation < 0) {
      current = current.left;
    } else if (firstRelation > 0 && secondRelation > 0) {
      current = current.right;
    } else {
      return current;
    }
  }

  return null;
}
```

### 13. Code Walkthrough
`findNode` follows one BST path and returns the stored comparator-equal node, never a query object or copied value. The two null checks establish the precondition for returning an LCA. The final loop compares the found stored values with each current node and moves only when both belong strictly to the same child subtree. Equality or opposite signs identify the deepest split. No tree field is assigned.

### 14. Complexity Analysis
Each presence search descends at most `h + 1` nodes, and the split search does the same. At most three bounded descents still take `O(h)` time because constant factors do not change asymptotic complexity. Iteration retains only fixed node references and comparator results, so auxiliary space is `O(1)`. Time is `O(log n)` for a height-balanced tree and `O(n)` for a skewed tree.

### 15. Edge Cases
- An empty tree returns `null`.
- If either or both keys are absent, return `null`.
- Comparator-equal query keys return their one stored node when present.
- Distinct query objects may comparator-match stored values.
- One stored target may be an ancestor of the other and is then the LCA.
- Targets in different root subtrees have the root as their LCA.
- The caller-owned tree and stored values remain unchanged.
- Proper finite strict BST shape and ordering are preconditions, not revalidated by the function.
- A malformed or stateful comparator is outside the contract.

### 16. Common Mistakes
- Returning the first split node without proving that both targets exist.
- Accepting node references as queries when the API contract requires keys.
- Using `===` for key equality instead of comparator equality.
- Returning a query object, copied node, or copied value instead of an existing node reference.
- Expecting comparator outputs to be exactly `-1`, `0`, and `1`.
- Building paths and then claiming `O(1)` auxiliary space.
- Calling the running time unconditionally `O(log n)`.

### 17. Interview Follow-ups
1. Return which query key was absent. 2. Solve binary-tree LCA without the BST ordering invariant. 3. Support parent pointers. 4. Find the LCA of more than two present keys. 5. Add a trusted membership index while preserving node identity.

### 18. Variations
- LCA from node references. - LCA in a general binary tree. - LCA with parent pointers. - LCA of minimum and maximum query keys. - Deepest common prefix of search paths.

### 19. Pattern Connections
BST LCA combines ordinary comparator-guided membership search with split-point descent. The presence checks mirror successor and predecessor's requirement not to return a nearby candidate for an absent key, while path-prefix comparison connects the problem to general-tree LCA.

### 20. NestJS Project Connection
Expose `POST /api/dsa/bst/lca` with a bounded tree DTO and two key DTOs. Use a fixed server-side comparator and validate proper finite shape, strict ordering, node count, and depth once at the boundary. Trace only the optimized phases (`find-first`, `find-second`, and `split`) with direction and a stable approved node ID; cap events using the accepted depth bound, mark a truncated trace, and never serialize arbitrary values or accept comparator code. The algorithm itself remains read-only and performs no unbounded whole-tree trace.

### 21. Mini Challenge
Return `{ lca, first, second }`, where all non-null fields are existing stored node references and `lca` is non-null only when both targets exist. Preserve comparator equality, `O(h)` time, `O(1)` auxiliary space, and read-only behavior.

## Problem 174 — Kth Smallest Element in BST

### 1. Problem Statement
Given the root of a caller-owned proper finite valid strict BST and a one-based rank `k`, return the existing node reference for the kth-smallest key. `k` must be a positive safe integer; throw `RangeError` otherwise, even when the tree is empty. Return `null` when the tree contains fewer than `k` nodes. The operation is read-only, preserves object identity, and relies only on the comparator-defined strict BST ordering rather than comparing values during traversal.

### 2. Example
```text
          8
        /   \
       4    12
      / \   / \
     2   6 10 14

inorder nodes: [2, 4, 6, 8, 10, 12, 14]
k = 1 -> existing node 2
k = 4 -> existing node 8
k = 7 -> existing node 14
k = 8 -> null
k = 0 -> throw RangeError
empty tree, k = 1 -> null
empty tree, k = NaN -> throw RangeError
```

If values are objects, the returned result is still the exact stored node at rank `k`, not a new node or copied value.

### 3. Understand the Problem
Inorder traversal of a valid strict BST visits nodes from smallest to largest. Because `k` is one-based, the answer is the node removed from the traversal stack for the kth time. Invalid rank input is a contract error, while a valid rank larger than the tree is an ordinary missing result.

#### In Simple Terms
Walk the BST in sorted order and count visited nodes. Return the exact node at count `k`, or `null` if traversal ends first.

### 4. Real-World Analogy
Imagine opening an ordered archive from the earliest label onward. The kth file you physically reach is the answer. An invalid ticket number such as zero is rejected, while a valid ticket beyond the archive's size simply has no matching file.

### 5. Key Observation
An iterative inorder traversal can pause after the kth pop: push a left chain, pop the next-smallest node, increment the rank, and then enter its right subtree. Nodes after rank `k` are irrelevant. No subtree-size augmentation is assumed, so navigation cannot skip entire subtrees by rank.

### 6. Pattern
```text
Pattern: Iterative Inorder Prefix with Early Stop
Why this pattern applies: valid BST inorder order is strict sorted order
How to recognize it: request for a small rank without subtree-size metadata
Similar problems: inorder iterator, range prefix, successor sequence, order statistics
```

### 7. Brute Force
Complete an iterative inorder traversal and store every existing node reference in an array. After the traversal, return the node at zero-based index `k - 1` when that index exists. Validate `k` before inspecting or traversing the root.

#### Pseudocode
```text
if k is not a positive safe integer: throw RangeError
nodes = complete inorder traversal
if k exceeds nodes.length: return null
return nodes[k - 1]
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function kthSmallestByTraversal<T>(
  root: TreeNode<T> | null,
  k: number,
  _compare: Comparator<T>,
): TreeNode<T> | null {
  if (!Number.isSafeInteger(k) || k <= 0) {
    throw new RangeError("k must be a positive safe integer");
  }

  const nodes: TreeNode<T>[] = [];
  const stack: TreeNode<T>[] = [];
  let current = root;

  while (current !== null || stack.length > 0) {
    while (current !== null) {
      stack.push(current);
      current = current.left;
    }

    const node = stack.pop();
    if (node === undefined) break;
    nodes.push(node);
    current = node.right;
  }

  return nodes[k - 1] ?? null;
}
```

#### Visualization
```text
complete inorder array: [node 2, node 4, node 6, node 8, node 10, node 12, node 14]
k = 4 -> array index 3 -> existing node 8
k = 8 -> array index 7 is absent -> null
```

#### Dry Run
For `k = 4`, the traversal still visits and stores all seven nodes in sorted order, then returns the existing node at index `3`, node `8`. For `k = 8`, it builds the same complete array and returns `null`. For invalid `k = 0`, validation throws before allocating the arrays or examining an empty or nonempty root.

#### Brute-Force Complexity
For valid `k`, complete traversal takes `O(n)` time. The retained node array uses `O(n)` temporary space and the traversal stack uses `O(h)`, for `O(n + h)` auxiliary space. Invalid `k` throws in `O(1)` time and space. The tree and all stored values remain unchanged.

### 8. Why Brute Force Is Not Ideal
The baseline traverses every node and retains the complete inorder array even when a small rank is requested. An inorder stack already exposes nodes in rank order, so the optimized traversal can return immediately at the kth pop and avoid the `O(n)` array.

### 9. Optimal Approach
Validate `k` first. Iteratively push the current node and every left descendant. Pop the next-smallest node, increment a visit count, and return that exact node when the count equals `k`; otherwise continue from its right child. If both the current pointer and stack become empty before count `k`, return `null`. This uses no subtree-size augmentation.

### 10. Visualization
```text
k = 4
push: 8, 4, 2
pop 2 -> rank 1
pop 4 -> rank 2; enter right subtree
push 6
pop 6 -> rank 3
pop 8 -> rank 4 -> return existing node 8

Nodes 10, 12, and 14 are never visited.
```

### 11. Step-by-Step Dry Run
| Action | Stack bottom to top | Popped rank | Outcome |
|---|---|---:|---|
| Push left chain from `8` | `[8, 4, 2]` | - | continue |
| Pop `2` | `[8, 4]` | `1` | move to `2.right` |
| Pop `4` | `[8]` | `2` | move to node `6` |
| Push and pop `6` | `[8]` | `3` | move to `6.right` |
| Pop `8` | `[]` | `4` | return node `8` |

For `k = 8`, all seven nodes are popped, traversal ends with count `7`, and the function returns `null`. For `k = 0`, it throws before any stack action, including when `root` is `null`.

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function kthSmallest<T>(
  root: TreeNode<T> | null,
  k: number,
  _compare: Comparator<T>,
): TreeNode<T> | null {
  if (!Number.isSafeInteger(k) || k <= 0) {
    throw new RangeError("k must be a positive safe integer");
  }

  const stack: TreeNode<T>[] = [];
  let current = root;
  let visited = 0;

  while (current !== null || stack.length > 0) {
    while (current !== null) {
      stack.push(current);
      current = current.left;
    }

    const node = stack.pop();
    if (node === undefined) break;

    visited += 1;
    if (visited === k) return node;
    current = node.right;
  }

  return null;
}
```

### 13. Code Walkthrough
Validation precedes every root-dependent branch, so malformed ranks always throw. The stack represents suspended ancestors whose left subtrees are complete. Each pop exposes the next node in strict inorder order; the kth pop returns that existing reference immediately. The comparator parameter documents the generic BST contract but is intentionally unused because valid-BST inorder structure already determines rank. No node field or value is assigned.

### 14. Complexity Analysis
Before the answer, traversal pushes at most an initial left path plus paths exposed by the first `k` pops, giving `O(h + k)` time; it never visits more than all `n` nodes, so the bound is also `O(n)` and can be written `O(min(n, h + k))`. The explicit stack contains at most `O(h)` node references. A height-balanced tree uses `O(log n)` stack space, while a skewed tree can use `O(n)`. Invalid `k` throws in `O(1)` time and space.

### 15. Edge Cases
- `k = 1` returns the minimum stored node.
- `k = n` returns the maximum stored node.
- A valid `k > n` returns `null` after traversal ends.
- An empty tree with valid `k` returns `null`.
- Zero, negative numbers, fractions, `NaN`, infinities, and unsafe integers throw `RangeError`, even for an empty tree.
- Object-valued nodes preserve the exact stored node and value identities.
- The caller-owned tree remains unchanged.
- Proper finite valid strict BST shape and ordering are preconditions.
- No subtree-size field or other augmentation is assumed.

### 16. Common Mistakes
- Treating `k` as zero-based.
- Returning `null` for an empty tree before validating `k`.
- Accepting positive fractions or unsafe integers.
- Traversing the entire tree after the kth node has been popped.
- Returning a copied value rather than the existing node reference.
- Claiming `O(1)` auxiliary space while using an explicit or recursive traversal stack.
- Claiming `O(h)` rank selection without subtree-size augmentation.
- Mutating links or values during a read-only traversal.

### 17. Interview Follow-ups
1. Return the kth largest node. 2. Implement a reusable lazy inorder iterator. 3. Handle many rank queries. 4. Add subtree sizes and maintain them during insertion and deletion. 5. Return a bounded prefix of ranked nodes.

### 18. Variations
- Kth largest node. - Rank of a present key. - Nodes in a rank interval. - Median of a BST. - Selection in an augmented balanced BST.

### 19. Pattern Connections
Early-stop inorder traversal is the same sorted stream used by validation, successor reasoning, and range iteration, but consumes only the needed prefix. Without metadata, structural traversal is necessary. With correctly maintained subtree-size augmentation, each rank query can choose a side and skip an entire subtree in `O(h)` time.

### 20. NestJS Project Connection
Expose `POST /api/dsa/bst/kth-smallest` with a bounded tree DTO and numeric `k`. Validate `k` as a positive safe integer before handling an empty decoded tree, and validate proper finite shape, strict ordering, node count, and depth once at the boundary with a fixed server-side comparator. Trace only optimized `push` and `pop` events using stable approved node IDs; cap events by the accepted node/depth limits, stop tracing or mark it truncated at the response budget, and never serialize arbitrary stored values. Return an approved representation of the selected existing node or `null`.

### 21. Mini Challenge
Implement kth largest with a reverse-inorder stack and early stop. Keep one-based positive-safe-integer validation, existing-node identity, read-only behavior, `O(h + k)` time bounded by `O(n)`, and `O(h)` auxiliary space without subtree-size augmentation.

## Problem 175 — Sorted Array to Balanced BST

### 1. Problem Statement
Given a `readonly T[]` and a `Comparator<T>`, build a fresh proper finite strict binary search tree containing the array's values. The input must be strictly increasing in comparator order: if any adjacent pair is comparator-equal, inverted, or otherwise does not compare as strictly increasing, throw `RangeError`. Validate the entire input before allocating any tree node, so invalid input cannot leave a partially constructed tree. An empty array returns `null`.

Do not mutate the input or clone its values. Every output node must be a fresh `TreeNode`, while each node's `value` preserves the exact generic value reference from the array. For every inclusive range, choose its deterministic lower-middle index as `low + Math.floor((high - low) / 2)`. The resulting strict BST must be height-balanced: at every node, its child subtree heights measured in edges differ by at most one.

### 2. Example
```text
values: [2, 4, 6, 8]

lower-middle construction:
        4
       / \
      2   6
           \
            8

The root is 4, not 6, because range [0, 3] chooses index 1.
```

For object values `records = [recordA, recordB, recordC, recordD]` sorted by a fixed `id` comparator, the same four-node shape is created. Its inorder node values satisfy `nodeA.value === recordA`, `nodeB.value === recordB`, and so on. The nodes themselves are new objects; the records are not cloned.

### 3. Understand the Problem
The input already provides the required inorder sequence. Choosing a middle value as the root leaves smaller values for the left subtree and larger values for the right subtree. Choosing the lower middle makes even-length ranges deterministic without implying that every valid height-balanced BST shape is unique.

The validation contract is atomic with respect to node construction. Every adjacent comparison must finish successfully before the first output node is allocated. Checking `!(compare(previous, current) < 0)` rejects comparator equality, positive inversions, and non-ordering numeric results such as `NaN`.

#### In Simple Terms
Check that the whole array is strictly sorted first, then repeatedly make the lower middle value the root of each remaining range.

### 4. Real-World Analogy
Imagine arranging an already sorted catalog into a decision tree. Place the lower middle card at the desk, then do the same with the cards before it and the cards after it. Before placing any card, inspect the entire catalog to ensure no label is duplicated or out of order.

### 5. Key Observation
For an inclusive range of length `m`, the lower-middle split creates child ranges whose sizes differ by at most one. Applying the same rule recursively makes both children height-balanced and keeps their edge heights within one. The original array can remain shared and read-only because the optimized recursion needs only range boundaries.

### 6. Pattern
```text
Pattern: Divide and Conquer over a Sorted Range
Why this pattern applies: inorder order is fixed, so a middle element can become the root
How to recognize it: sorted input must become a shallow search tree without reordering values
Similar problems: balanced tree construction, binary-search recursion, sorted-list conversion
```

### 7. Brute Force
First validate every adjacent pair in the original array. Then recursively process an array slice: choose its lower-middle element, allocate one node, and recursively pass fresh left and right slices. Slicing makes each recursive input convenient but repeatedly copies references that index boundaries could avoid.

#### Pseudocode
```text
for every adjacent pair in the complete input:
    if previous is not strictly less than current: throw RangeError
if input is empty: return null

build(sorted slice):
    if slice is empty: return null
    low = 0
    high = slice.length - 1
    middle = low + floor((high - low) / 2)
    node = fresh node containing sorted[middle]
    node.left = build(copy before middle)
    node.right = build(copy after middle)
    return node
```

#### Brute-Force TypeScript
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function sortedArrayToBalancedBstBySlicing<T>(
  values: readonly T[],
  compare: Comparator<T>,
): TreeNode<T> | null {
  for (let index = 1; index < values.length; index += 1) {
    if (!(compare(values[index - 1], values[index]) < 0)) {
      throw new RangeError("Values must be strictly increasing");
    }
  }

  if (values.length === 0) return null;

  const build = (sorted: readonly T[]): TreeNode<T> | null => {
    if (sorted.length === 0) return null;

    const low = 0;
    const high = sorted.length - 1;
    const middle = low + Math.floor((high - low) / 2);
    const node: TreeNode<T> = {
      value: sorted[middle],
      left: null,
      right: null,
    };

    node.left = build(sorted.slice(0, middle));
    node.right = build(sorted.slice(middle + 1));
    return node;
  };

  return build(values);
}
```

#### Visualization
```text
[2, 4, 6, 8]
 middle 4
 left slice [2]       -> node 2
 right slice [6, 8]  -> node 6 with right child 8

Every recursive edge receives a newly copied array slice.
```

#### Dry Run
Validation compares `2 < 4`, `4 < 6`, and `6 < 8` before any node exists. The first slice has inclusive indexes `0..3`, so the formula chooses index `1` and allocates node `4`. Its left slice builds node `2`. Its right slice `[6, 8]` chooses local index `0`, builds node `6`, and places node `8` on the right.

If the input is `[2, 4, 4, 8]`, the comparison between the two `4` values is not negative. The function throws before defining the recursive builder or allocating a node, so no partial tree is constructed.

#### Brute-Force Complexity
Validation takes `O(n)` time. Across balanced recursion levels, slicing copies `O(n)` element references per level for `O(n log n)` aggregate copying time and `O(n log n)` total temporary copied element slots allocated over the run. Because slices are created one child call at a time and active slice lengths form geometric chains, at most `O(n)` copied element slots need be live simultaneously, plus `O(log n)` recursive frames. The required output contains `O(n)` fresh nodes.

### 8. Why Brute Force Is Not Ideal
The recursive subproblems are contiguous ranges of the original array, but slicing materializes those ranges repeatedly. Passing inclusive indexes describes the same subproblems without copying any element-reference arrays.

### 9. Optimal Approach
Validate all adjacent pairs before handling the empty result or invoking construction. Then recurse over inclusive indexes `low` and `high`. Return `null` when `low > high`; otherwise compute the lower middle, allocate one node with the original value reference, and build the two disjoint child ranges. No array operation writes to or copies the input.

### 10. Visualization
```text
indexes:  0  1  2  3
values:  [2, 4, 6, 8]

build(0, 3): middle 1 -> 4
build(0, 0): middle 0 -> 2
build(2, 3): middle 2 -> 6
build(3, 3): middle 3 -> 8

Only index pairs move through recursion; no slices are created.
```

### 11. Step-by-Step Dry Run
| Call | Inclusive range | Middle index | Node value | Left range | Right range |
|---|---|---:|---:|---|---|
| `build(0, 3)` | `[0, 3]` | `1` | `4` | `[0, 0]` | `[2, 3]` |
| `build(0, 0)` | `[0, 0]` | `0` | `2` | empty | empty |
| `build(2, 3)` | `[2, 3]` | `2` | `6` | empty | `[3, 3]` |
| `build(3, 3)` | `[3, 3]` | `3` | `8` | empty | empty |

The inorder traversal is `[2, 4, 6, 8]`, and each node's left and right edge heights differ by at most one.

### 12. TypeScript Solution
```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
}

type Comparator<T> = (left: T, right: T) => number;

function sortedArrayToBalancedBst<T>(
  values: readonly T[],
  compare: Comparator<T>,
): TreeNode<T> | null {
  for (let index = 1; index < values.length; index += 1) {
    if (!(compare(values[index - 1], values[index]) < 0)) {
      throw new RangeError("Values must be strictly increasing");
    }
  }

  if (values.length === 0) return null;

  const build = (low: number, high: number): TreeNode<T> | null => {
    if (low > high) return null;

    const middle = low + Math.floor((high - low) / 2);
    return {
      value: values[middle],
      left: build(low, middle - 1),
      right: build(middle + 1, high),
    };
  };

  return build(0, values.length - 1);
}
```

### 13. Code Walkthrough
The validation loop runs to the end before the empty return and before `build` can allocate a node. Its negated strict comparison rejects zero, positive, and `NaN` comparator results. Construction uses the required inclusive-range formula, and each recursive call receives a strictly smaller range. Object literals create fresh nodes, while direct assignment from `values[middle]` preserves the stored value reference. No input index or value is assigned.

### 14. Complexity Analysis
Validation examines `n - 1` adjacent pairs, and construction allocates and initializes each of the `n` output nodes once, so total time is `O(n)`. Lower-middle splits keep recursion depth `O(log n)`, which is the auxiliary frame cost. Excluding the required `O(n)` output tree, the optimized algorithm allocates no arrays or other storage proportional to `n`.

### 15. Edge Cases
- An empty array performs no comparisons and returns `null` without allocating a node.
- A one-value array returns one fresh node containing the exact input value reference.
- An even-length range always chooses its lower middle.
- Comparator-equal adjacent values throw `RangeError`, including distinct objects with equal keys.
- An inversion throws `RangeError` before any node allocation.
- A `NaN` comparator result fails the strict-negative check and throws `RangeError`.
- Object values are not cloned; each output node preserves the corresponding input reference.
- The readonly input is never sorted, sliced, or otherwise mutated by the optimized solution.
- Multiple height-balanced shapes may be valid, but this contract selects one deterministic shape.

### 16. Common Mistakes
- Building part of the tree before discovering an invalid pair later in the array.
- Checking only for equality and accidentally accepting inversions.
- Using `compare(previous, current) >= 0`, which does not reject `NaN`.
- Sorting the caller's array and violating the input and validation contracts.
- Choosing the upper middle for even-length ranges.
- Copying slices in the optimized solution while claiming `O(log n)` auxiliary space.
- Cloning generic values instead of preserving their references.
- Claiming that a balanced BST shape is universally unique.

### 17. Interview Follow-ups
1. Build a height-balanced BST from a sorted singly linked list. 2. Return parent pointers without changing asymptotic complexity. 3. Construct an immutable persistent node type. 4. Stream values when random access is unavailable. 5. Verify height balance and inorder identity correspondence in tests.

### 18. Variations
- Sorted array to AVL tree. - Sorted list to balanced BST. - Upper-middle deterministic construction. - Minimal-height tree from distinct keys. - Balanced map construction from sorted key-value entries.

### 19. Pattern Connections
This construction is the inverse of inorder traversal: the sorted sequence fixes inorder positions, while middle selection supplies a balanced recursive shape. Its range recursion mirrors binary search, merge sort, and segment-tree construction, but every input value becomes exactly one output node.

### 20. NestJS Project Connection
Expose `POST /api/dsa/bst/from-sorted` with a bounded readonly array DTO. Enforce fixed server-side item semantics and comparator selection; never accept executable comparator code. Apply explicit bounds for array length, payload depth, serialized item size, and construction or trace depth. Call only `sortedArrayToBalancedBst` with the fixed comparator, map `RangeError` to a stable validation response, cap lower-middle range trace events, and serialize only approved value fields. The algorithm's full prevalidation ensures an invalid request creates no partial tree.

### 21. Mini Challenge
Write a test helper that proves the output is a proper finite strict BST, is height-balanced by edge height, contains exactly `values.length` fresh nodes, follows lower-middle choices, preserves every value reference in inorder order, and leaves the readonly input unchanged.

# Topic Summary

## What You Learned
A binary search tree combines tree structure with comparator-defined ordering. In a valid strict BST, every left-subtree value is below its node and every right-subtree value is above it under the same comparator. Comparator equality defines duplicate keys even when values are different object references, while node identity still matters for mutation and returned-node contracts.

Search, insertion, deletion, extrema, successor, predecessor, and BST LCA follow one or a constant number of downward paths. Their `O(h)` time is `O(log n)` only when height is controlled; a plain skewed BST can make it `O(n)`. Inorder traversal exposes strict sorted order, recursive exclusive bounds validate global ordering, and lower-middle divide and conquer turns a strictly sorted array into a deterministic height-balanced tree.

Production-quality implementations make ownership and validation explicit. They distinguish read-only operations from link mutation, preserve stored object identities, validate before allocation when atomic failure is required, avoid assuming exact comparator magnitudes, and bound input size, depth, serialization, and traces at service boundaries.

## Patterns You Should Recognize
- **Comparator-guided descent:** use only the sign of one comparison to discard a subtree.
- **Failed-search insertion point:** attach one fresh node at the null link where search ends.
- **Node transplantation:** delete by relinking existing nodes rather than copying values.
- **Boundary descent:** follow only left links for minimum or right links for maximum.
- **Ancestor candidate tracking:** remember the best larger or smaller node while descending.
- **Exclusive recursive bounds:** validate every node against all relevant ancestors.
- **Presence before split:** prove both BST keys exist before returning their LCA.
- **Early-stop inorder:** consume only the sorted prefix needed for a rank query.
- **Validate before construction:** reject the complete input before allocating output nodes.
- **Sorted-range divide and conquer:** choose deterministic middle indexes without slicing.

## Complexity Cheat Sheet
| Problem | Main approach | Time | Auxiliary space | Required output space |
|---:|---|---|---|---|
| 166 | Comparator-guided search | `O(h)` | `O(1)` | one existing reference or `null` |
| 167 | Iterative descent and attachment | `O(h)` | `O(1)` | at most one fresh node |
| 168 | Search and successor transplantation | `O(h)` | `O(1)` | `O(1)` returned root reference |
| 169 | Left and right boundary walks | `O(h)` | `O(1)` | two existing references |
| 170 | Successor candidate descent | `O(h)` | `O(1)` | one existing reference or `null` |
| 171 | Predecessor candidate descent | `O(h)` | `O(1)` | one existing reference or `null` |
| 172 | Recursive exclusive bounds | `O(n)` | `O(h)` | `O(1)` boolean |
| 173 | Two presence searches plus split descent | `O(h)` | `O(1)` | one existing reference or `null` |
| 174 | Early-stop iterative inorder | `O(min(n, h + k))` | `O(h)` | one existing reference or `null` |
| 175 | Indexed lower-middle construction | `O(n)` | `O(log n)` | `O(n)` fresh tree |

## Common Interview Traps
- Claiming unconditional `O(log n)` time for operations on a plain BST.
- Comparing generic keys with language operators or object identity instead of the comparator.
- Assuming a comparator returns exactly `-1`, `0`, or `1` rather than reading its sign.
- Allowing comparator-equal duplicates in a strict BST.
- Returning query objects or copied values instead of existing stored node references.
- Allocating before duplicate detection during insert or before full validation during construction.
- Deleting a two-child node by copying its successor value when node identity must be preserved.
- Forgetting to assign a root returned by insertion or deletion.
- Returning a successor, predecessor, or LCA candidate before proving the queried key exists.
- Checking only parent-child relations during BST validation and missing ancestor violations.
- Treating one-based rank as a zero-based index or skipping rank validation for an empty tree.
- Claiming rank selection is `O(h)` without subtree-size augmentation.
- Sorting or slicing a readonly input while claiming indexed `O(n)` construction.
- Using an upper middle when the contract requires the lower middle.
- Claiming every height-balanced BST shape for a sorted sequence is unique.

## Practice Checklist
- [ ] 166. Search by comparator sign and return the existing stored node.
- [ ] 167. Insert at one null link while making duplicates a complete no-op.
- [ ] 168. Delete by transplanting nodes and detach the removed target.
- [ ] 169. Find minimum and maximum through boundary links only.
- [ ] 170. Track a larger ancestor candidate for inorder successor.
- [ ] 171. Track a smaller ancestor candidate for inorder predecessor.
- [ ] 172. Validate strict BST ordering with inherited exclusive bounds.
- [ ] 173. Prove both keys are present before returning their BST LCA.
- [ ] 174. Stop inorder traversal at a validated one-based rank.
- [ ] 175. Build a fresh deterministic balanced BST from validated sorted values.
