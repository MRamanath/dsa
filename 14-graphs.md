# Graphs

## Learning Objectives

By the end of this topic, you should understand:

- how vertices and edges model relationships that are not hierarchical
- how adjacency lists represent directed, undirected, weighted, and disconnected graphs
- how breadth-first and depth-first traversal use visited state to terminate on cycles
- how parent tracking and color states detect undirected and directed cycles
- how grid problems become implicit graph traversals over neighboring cells
- how identity maps clone cyclic object graphs without duplicating shared vertices
- how indegrees support topological ordering and dependency feasibility
- why BFS finds shortest paths in unweighted graphs
- why Dijkstra requires nonnegative edge weights and a stale-entry check
- how Kruskal's algorithm combines sorted edges with disjoint sets
- how path compression and union by size make connectivity operations nearly constant amortized time
- how validation, deterministic order, checked arithmetic, ownership, and bounded traces affect production APIs

## Prerequisites

- arrays, objects, maps, sets, queues, stacks, and heaps
- recursion and iterative traversal
- binary-tree traversal concepts from `11-binary-trees.md`
- priority queues from `13-heaps-priority-queues.md`
- strict TypeScript, readonly input, object identity, and explicit mutation contracts
- Big-O analysis in terms of vertices `V`, edges `E`, rows `R`, and columns `C`
- safe-integer validation and checked addition

## Core Theory

### Graph vocabulary

A graph contains **vertices** and **edges**. An edge connects two vertices:

- A **directed** edge `u -> v` can be followed from `u` to `v`.
- An **undirected** edge `u -- v` can be followed both ways.
- A **weighted** edge associates a cost such as distance or time.
- A **self-loop** connects a vertex to itself.
- **Parallel edges** are multiple edges between the same endpoint pair.

A **path** follows consecutive edges. A **cycle** returns to an already visited vertex through a nonempty edge sequence. A graph may have multiple disconnected components.

### Dense vertex IDs

Array-backed problems in this chapter use dense integer vertex IDs:

```text
0, 1, 2, ..., vertexCount - 1
```

An adjacency list has one array per vertex:

```typescript
type AdjacencyList = readonly (readonly number[])[];
```

The outer index is the source vertex. Each inner value is a destination. Neighbor array order is observable and determines deterministic traversal order unless a problem states another tie-break.

### Adjacency-list validation

An unweighted adjacency list is valid when every neighbor is a safe integer in `[0, graph.length)`. Algorithms validate the complete structure before producing output or relying on a start vertex when their contract promises atomic rejection.

For an undirected graph, callers must additionally supply symmetric edge occurrences: every occurrence of `u -> v` has a matching `v -> u` occurrence. Two occurrences each way represent parallel edges. A self-loop is one adjacency occurrence and is its own reverse. Problems that assume this undirected representation state it explicitly.

### Edge-list construction

Problem 186 builds adjacency lists from readonly endpoint pairs. Directed construction appends only `from -> to`. Undirected construction appends both directions, except that a self-loop is appended once. Input edge order and endpoint order determine neighbor order. Parallel edges are retained rather than deduplicated.

The builder validates vertex count and all endpoints before allocating the returned adjacency arrays, giving invalid input an atomic failure contract.

### Breadth-first search

Breadth-first search, or BFS, explores vertices by increasing edge distance from a start:

```text
mark start visited and enqueue it
while queue has unread entries:
    read one vertex
    for each neighbor in stored order:
        if unseen, mark immediately and enqueue
```

Marking on enqueue prevents the same vertex from entering the queue multiple times. An array queue advances a head index instead of calling `shift()`, avoiding repeated reindexing.

### Depth-first search

Depth-first search, or DFS, follows one branch before returning. Recursive DFS uses `O(V)` call depth in the worst case. Iterative DFS uses an explicit stack. Reverse-pushing neighbors is often sufficient when only a valid DFS order matters, but exact recursive stored-neighbor preorder on arbitrary graphs requires resumable frames because cross edges can otherwise change which branch first claims a vertex.

Visited state is required for both forms. Without it, an undirected edge immediately sends traversal back to the parent, and any reachable cycle can make traversal nonterminating.

### Deterministic traversal scope

Problems 187 and 188 traverse only the component reachable from a required start vertex. They validate the whole adjacency list first, then visit the start and follow each vertex's stored neighbor order. They do not sort, deduplicate, or mutate adjacency arrays.

To traverse every component, run the same traversal from each still-unvisited vertex in ascending vertex-ID order.

### Undirected cycle detection

In undirected DFS, seeing the parent edge is expected; seeing another already visited neighbor proves a cycle. Multigraphs need one extra detail: skip only one occurrence of the parent edge. A second parallel edge back to the parent forms a two-edge cycle. A self-loop also forms a cycle.

### Directed cycle detection

Directed DFS uses three colors:

- `0`: unvisited
- `1`: active on the current DFS path
- `2`: fully processed

An edge to an active vertex is a back edge and proves a directed cycle. An edge to a fully processed vertex does not.

### Grids as implicit graphs

A rectangular grid can be viewed as a graph whose cells are vertices. Four-directional neighbors are up, right, down, and left when in bounds. The grid need not allocate an explicit adjacency list; row and column arithmetic generates neighbors on demand.

This chapter validates every grid row before traversal. Empty outer arrays are valid where stated. Nonempty grids must be rectangular, and cell values must belong to each problem's documented domain.

### Object-graph identity

Problem 194 uses object references rather than dense IDs:

```typescript
interface GraphNode<T> {
  value: T;
  neighbors: GraphNode<T>[];
}
```

Cloning requires a map from original node identity to fresh clone identity. Create and register a clone before exploring its neighbors; that early registration terminates cycles and preserves shared references, duplicate neighbor entries, self-loops, and neighbor order. Values themselves remain the original `T` references.

### Topological ordering

A topological order places every directed edge `u -> v` with `u` before `v`. Such an order exists exactly when the directed graph is acyclic.

Kahn's algorithm counts indegrees, enqueues all zero-indegree vertices in ascending ID order, removes them through a head-index queue, and decrements outgoing neighbors in stored order. If fewer than `V` vertices are emitted, a cycle remains.

### Unweighted shortest paths

In an unweighted graph, every edge contributes one step. BFS discovers each vertex by the fewest possible edges, so assigning `distance[neighbor] = distance[current] + 1` at first discovery is correct. This chapter uses `-1` for unreachable vertices.

### Dijkstra's shortest paths

Dijkstra handles nonnegative weighted directed graphs. It stores tentative distances and repeatedly removes the smallest distance entry from a min heap. Because the same vertex may enter the heap multiple times, an entry whose distance differs from the current best is stale and must be skipped.

Negative weights invalidate the greedy finalization argument. Every weight and source must be validated before traversal, and every evaluated distance addition must remain a safe integer.

### Minimum spanning trees

For a connected undirected weighted graph, a minimum spanning tree connects all vertices with `V - 1` edges and minimum total weight. Kruskal sorts edges by weight, then input index for deterministic ties. It accepts an edge only when its endpoints currently belong to different components.

Self-loops are naturally rejected. Parallel edges are allowed. Negative safe-integer weights are valid. Checked addition protects the accumulated total. This chapter returns `null` when a nonempty graph is disconnected and `{ weight: 0, edges: [] }` for zero vertices.

### Disjoint set union

A disjoint set union structure, also called union-find, maintains a partition of dense IDs. `find` returns a representative, `union` merges two components, and `connected` compares representatives.

Path compression shortens parent chains during `find`. Union by size attaches the smaller root below the larger root. Together, a sequence of operations takes `O(alpha(n))` amortized time per operation, where `alpha` is the inverse Ackermann function and grows extraordinarily slowly.

### Ownership and mutation

Unless stated otherwise, graph and grid algorithms borrow readonly inputs and return fresh arrays or objects. Flood fill returns a fresh rectangular grid and does not mutate its input. Clone Graph returns fresh node objects while preserving original value references. Disjoint Set Union intentionally mutates only its private owned arrays.

### Safe arithmetic

Vertex IDs, counts, dimensions, and array indexes must be valid safe integers within actual array bounds. Dijkstra and MST weights are safe integers. Every evaluated path or total-weight addition uses checked arithmetic:

```typescript
function safeAdd(left: number, right: number): number {
  const sum = left + right;
  if (!Number.isSafeInteger(sum)) throw new RangeError("unsafe graph arithmetic");
  return sum;
}
```

## Visual Mental Model

For this directed adjacency list:

```text
graph = [
  [1, 2],
  [3],
  [3],
  []
]

0 -> 1 -> 3
 \-> 2 --^
```

BFS from `0` visits `0, 1, 2, 3`. DFS from `0`, following stored neighbors, visits `0, 1, 3, 2`.

A disconnected vertex is valid but absent from a start-scoped traversal:

```text
0 -- 1     2 -- 3

start 0 -> visits only 0, 1
```

## Complexity Concepts

### Adjacency-list size

Let `V` be the outer array length and `E` the number of directed adjacency occurrences. A directed edge contributes one occurrence. An ordinary non-self undirected edge contributes two. Storage is `O(V + E)`.

### Traversal cost

BFS and DFS visit each reachable vertex once and inspect each outgoing occurrence once. On a fully reachable graph, time is `O(V + E)` and visited plus queue/stack storage is `O(V)`.

### Grid cost

A rectangular `R x C` grid has `R * C` cells and at most four generated neighbors per cell. Complete grid traversals therefore take `O(R * C)` time and up to `O(R * C)` visited, queue, copied-grid, or output storage depending on the contract.

### Weighted algorithms

With a binary heap, Dijkstra takes `O((V + E) log V)` conventionally, or `O(E log V)` when all relevant vertices have incident edges. Kruskal takes `O(E log E)` for sorting plus near-linear disjoint-set work.

### Output versus auxiliary space

Traversal orders, distance arrays, cloned graphs, filled grids, and MST edge arrays are required output. Visited arrays, color arrays, queues, stacks, indegrees, heaps, maps, and disjoint-set arrays are auxiliary or owned state and should be reported separately.

## Common Patterns

### Validate then traverse

```text
validate every neighbor or cell
initialize visited state
traverse without mutating input
```

### Head-index BFS queue

```text
queue = [start]
head = 0
while head < queue.length:
    current = queue[head]
    head += 1
```

### DFS colors

```text
enter vertex -> active
edge to active -> cycle
finish vertex -> processed
```

### Heap relaxation

```text
pop smallest tentative distance
skip stale entry
for each edge:
    candidate = checked distance + weight
    if candidate improves best:
        update and push
```

### Union before accepting edge

```text
for edges in increasing weight:
    if union(u, v) merged components:
        accept edge
```

## Common Mistakes

- Confusing vertex count with edge count.
- Forgetting to validate neighbor IDs before indexing arrays.
- Marking BFS vertices only when dequeued and enqueuing duplicates.
- Calling `shift()` repeatedly on an array queue.
- Omitting visited state because a tree traversal did not need graph cycle protection.
- Treating every visited undirected neighbor as a cycle without excluding one parent edge.
- Using one visited boolean instead of active/processed states for directed cycles.
- Assuming a grid is rectangular without validation.
- Mutating a readonly grid or adjacency list accidentally.
- Cloning graph values while failing to preserve graph-node sharing and cycles.
- Returning a partial topological order as if it were complete.
- Using BFS for arbitrary weighted shortest paths.
- Running Dijkstra with negative weights or without skipping stale heap entries.
- Adding unsafe path or MST weights without checked arithmetic.
- Returning a spanning tree for a disconnected graph.
- Implementing union-find without validating dense IDs.

## Production Notes

- Bound vertex count, edge occurrences, grid cells, recursion depth, heap entries, output size, and trace events.
- Validate complete transport structures before algorithms that promise atomic rejection.
- Use request-scoped copies for mutating DTO workflows and readonly representations for borrowed data.
- Trace compact events such as vertex IDs, edge indexes, distances, or queue sizes; never snapshot the entire graph per step.
- Use fixed graph direction and weight semantics per endpoint rather than client-supplied executable behavior.
- Prefer iterative traversal for adversarial depth and established graph libraries for specialized flow, matching, spatial, or distributed algorithms.

## Problem 186 — Represent Graph using Adjacency List

### 1. Problem Statement
Implement `buildAdjacencyList(vertexCount: number, edges: readonly (readonly [number, number])[], directed: boolean): number[][]`. `vertexCount` must be a nonnegative safe integer no greater than `2^32 - 1`, the maximum JavaScript array length. Each edge must be an exact two-element array whose endpoints are safe integers in `[0, vertexCount)`. Validate the count and every complete edge before allocating the output; invalid input throws `RangeError` atomically. Production callers must impose a much lower practical vertex bound because the language-level maximum is generally impossible to allocate.

For a directed edge `[from, to]`, append only `to` to `from`'s list. For an undirected edge, also append `from` to `to`'s list, except that a self-loop is stored once. Preserve edge input order, retain parallel edges, support empty input, and do not mutate `edges` or its tuples. Return a fresh outer array containing fresh neighbor arrays.

### 2. Example
```text
vertexCount = 4
edges = [[0, 1], [0, 2], [1, 2], [2, 2], [0, 1]]

directed = true
result = [[1, 2, 1], [2], [2], []]

directed = false
result = [[1, 2, 1], [0, 2, 0], [0, 1, 2], []]
```

The repeated `[0, 1]` remains a parallel edge. The undirected self-loop `[2, 2]` contributes one `2`, not two.

### 3. Understand the Problem
The outer index represents a dense vertex ID, and each nested array records outgoing neighbors in observable order. Validation must finish before creating even the empty neighbor arrays, so malformed input cannot produce a partial graph or trigger allocation first.

#### In Simple Terms
Check every endpoint first, create one empty list per vertex, then record each edge in the order received.

### 4. Real-World Analogy
Think of a transit directory with one departure board per station. A one-way route appears only on its origin board; a two-way route appears on both boards. Repeated services remain repeated entries, and a route from a station back to itself appears once on that station's board.

### 5. Key Observation
Each valid edge can be appended directly to its endpoint list without searching, sorting, or deduplicating. This gives a one-pass construction after validation. A matrix is a poor baseline here because boolean cells lose parallel multiplicity, while count cells group neighbors by vertex ID and violate the required edge input order.

### 6. Pattern
```text
Pattern: Validate, Allocate, Then Bucket by Source
Why this pattern applies: dense vertex IDs map directly to array indexes
How to recognize it: convert an ordered edge stream into per-vertex neighbors
Similar problems: reverse adjacency, indegree tables, grouped event indexes
```

### 7. Brute Force
After full validation, consider each vertex in ascending order and rescan the entire edge list. Append each matching occurrence as it is encountered. The repeated scans are slow, but they preserve edge input order, parallel edges, direction, and the single-copy self-loop rule exactly.

#### Pseudocode
```text
validate vertexCount
validate every exact edge tuple and both endpoints
create an empty result list for every vertex
for each vertex:
  for each edge in input order:
    if edge starts at vertex: append its destination
    if undirected, edge ends at vertex, and it is not a self-loop:
      append its source
return the result
```

#### Brute-Force TypeScript
```typescript
function buildAdjacencyListByScanning(
  vertexCount: number,
  edges: readonly (readonly [number, number])[],
  directed: boolean,
): number[][] {
  const maximumArrayLength = 2 ** 32 - 1;
  if (
    !Number.isSafeInteger(vertexCount) ||
    vertexCount < 0 ||
    vertexCount > maximumArrayLength
  ) {
    throw new RangeError("vertexCount must be a practical array length");
  }

  for (const edge of edges) {
    if (!Array.isArray(edge) || edge.length !== 2) {
      throw new RangeError("every edge must be an exact endpoint pair");
    }
    const from = edge[0];
    const to = edge[1];
    if (
      !Number.isSafeInteger(from) ||
      !Number.isSafeInteger(to) ||
      from < 0 ||
      from >= vertexCount ||
      to < 0 ||
      to >= vertexCount
    ) {
      throw new RangeError("every endpoint must be a valid vertex ID");
    }
  }

  const adjacency = new Array<number[]>(vertexCount);
  for (let vertex = 0; vertex < vertexCount; vertex += 1) {
    adjacency[vertex] = [];
  }

  for (let vertex = 0; vertex < vertexCount; vertex += 1) {
    const neighbors = adjacency[vertex]!;
    for (const edge of edges) {
      const from = edge[0];
      const to = edge[1];
      if (from === vertex) neighbors.push(to);
      if (!directed && to === vertex && from !== to) neighbors.push(from);
    }
  }

  return adjacency;
}
```

#### Visualization
```text
vertex 0 scans all edges -> sees [0,1], [0,2], [0,1] -> [1,2,1]
vertex 1 scans all edges -> undirected reverses plus [1,2] -> [0,2,0]
vertex 2 scans all edges -> reverses plus one self-loop -> [0,1,2]
vertex 3 scans all edges -> []
```

#### Dry Run
For the undirected example, validation first accepts all five tuples. Vertex `0`'s scan appends `1`, `2`, and `1`. Vertex `1`'s independent scan appends the reverse of the first edge, destination `2`, and the reverse of the final edge. Vertex `2` appends two reverse endpoints and stores the self-loop once. No source tuple changes.

#### Brute-Force Complexity
Let `V` be `vertexCount` and `E` the number of input edges. Validation takes `O(E)`, initialization takes `O(V)`, and the nested scans take `O(VE)`, for `O(V + VE)` time. The returned adjacency list uses `O(V + A)` output space, where `A` is the number of stored adjacency occurrences, and the algorithm uses `O(1)` auxiliary space beyond that output.

### 8. Why Brute Force Is Not Ideal
Every edge is reconsidered once per vertex even though its endpoints already identify exactly which list or lists need an entry. The rescans turn a naturally linear conversion into `O(VE)` work.

### 9. Optimal Approach
Complete the same validation first. Allocate and initialize one fresh neighbor array per vertex. Then process edges once in input order: append `to` to `from`, and for an undirected non-self edge append `from` to `to`. Direct appends preserve order and multiplicity naturally.

### 10. Visualization
```text
start: [[], [], [], []]
[0,1] -> [[1], [], [], []]          and add reverse when undirected
[0,2] -> [[1,2], ...]
[1,2] -> vertex 1 gains 2
[2,2] -> vertex 2 gains one 2
[0,1] -> vertex 0 gains another 1
```

### 11. Step-by-Step Dry Run
| Edge | Directed append | Extra undirected append | Undirected lists after edge |
|---|---|---|---|
| `[0, 1]` | `0 -> 1` | `1 -> 0` | `[[1], [0], [], []]` |
| `[0, 2]` | `0 -> 2` | `2 -> 0` | `[[1,2], [0], [0], []]` |
| `[1, 2]` | `1 -> 2` | `2 -> 1` | `[[1,2], [0,2], [0,1], []]` |
| `[2, 2]` | `2 -> 2` | none | `[[1,2], [0,2], [0,1,2], []]` |
| `[0, 1]` | `0 -> 1` | `1 -> 0` | `[[1,2,1], [0,2,0], [0,1,2], []]` |

### 12. TypeScript Solution
```typescript
function buildAdjacencyList(
  vertexCount: number,
  edges: readonly (readonly [number, number])[],
  directed: boolean,
): number[][] {
  const maximumArrayLength = 2 ** 32 - 1;
  if (
    !Number.isSafeInteger(vertexCount) ||
    vertexCount < 0 ||
    vertexCount > maximumArrayLength
  ) {
    throw new RangeError("vertexCount must be a practical array length");
  }

  for (const edge of edges) {
    if (!Array.isArray(edge) || edge.length !== 2) {
      throw new RangeError("every edge must be an exact endpoint pair");
    }
    const from = edge[0];
    const to = edge[1];
    if (
      !Number.isSafeInteger(from) ||
      !Number.isSafeInteger(to) ||
      from < 0 ||
      from >= vertexCount ||
      to < 0 ||
      to >= vertexCount
    ) {
      throw new RangeError("every endpoint must be a valid vertex ID");
    }
  }

  const adjacency = new Array<number[]>(vertexCount);
  for (let vertex = 0; vertex < vertexCount; vertex += 1) {
    adjacency[vertex] = [];
  }

  for (const [from, to] of edges) {
    adjacency[from]!.push(to);
    if (!directed && from !== to) {
      adjacency[to]!.push(from);
    }
  }

  return adjacency;
}
```

### 13. Code Walkthrough
The count check and complete endpoint-validation loop run before `adjacency` is allocated. Initialization creates distinct nested arrays rather than shared references. The construction loop reads each tuple once, always performs the forward append, and conditionally performs exactly one reverse append. Nothing sorts, deduplicates, or writes to caller-owned input.

### 14. Complexity Analysis
Let `V` be the vertex count, `E` the edge count, and `A` the stored adjacency occurrences. Validation takes `O(E)`, array initialization takes `O(V)`, and construction takes `O(E)`, for `O(V + E)` time. The fresh return value occupies `O(V + A)` output space, and only `O(1)` additional auxiliary space is used. For directed input, `A = E`; for undirected input, `A` is at most `2E` because self-loops contribute once.

### 15. Edge Cases
- `vertexCount = 0` with no edges returns `[]`.
- Zero vertices with any edge throws because no endpoint is in range.
- Isolated vertices receive distinct empty arrays.
- Directed edges append only from source to destination.
- An undirected self-loop is stored exactly once.
- Parallel edges remain parallel adjacency occurrences in input order.
- Fractional, negative, non-finite, unsafe, or over-limit counts throw before edge validation or allocation.
- A malformed tuple or endpoint throws before any output array is allocated.

### 16. Common Mistakes
- Allocating adjacency arrays before validating every edge.
- Accepting arrays with fewer or more than two endpoint values.
- Using a boolean matrix and silently discarding parallel edges.
- Using a count matrix and returning neighbors grouped by ID instead of input order.
- Adding an undirected self-loop twice.
- Deduplicating or sorting observable neighbor arrays.
- Filling the outer array with one shared inner array.
- Treating the JavaScript maximum array length as a reasonable production request bound.

### 17. Interview Follow-ups
1. Build weighted adjacency entries. 2. Return both outgoing and incoming lists. 3. Support string vertex labels through an ID map. 4. Produce a compressed sparse row representation. 5. Enforce a deterministic production vertex and edge budget.

### 18. Variations
- Weighted graph builder. - Reverse graph. - Bipartite adjacency. - Compressed sparse row storage. - Deduplicated simple-graph representation.

### 19. Pattern Connections
The output is a bucketed index: each source ID chooses a destination bucket while input order supplies stable order within every bucket. The validate-then-allocate contract connects graph construction to atomic DTO processing. Unlike matrix conversion, direct bucketing preserves both multigraph multiplicity and stream order.

### 20. NestJS Project Connection
For `POST /api/dsa/graphs/adjacency-list`, use a bounded DTO with explicit limits far below `2^32 - 1`, such as a configured maximum vertex count, edge count, and total stored occurrences. Validate exact two-number tuples at the transport boundary and retain the service-level checks for defense in depth. Trace only bounded records such as `{ edgeIndex, from, to, reverseAdded }`; mark trace truncation explicitly and never log the full graph or allocate from an unbounded client count.

### 21. Mini Challenge
Implement `buildWeightedAdjacencyList` for exact triples `[from, to, weight]`. Preserve edge order and parallel edges, store an undirected self-loop once, require safe-integer endpoints and weights, and retain the same validate-before-allocation failure contract.

## Problem 187 — BFS Traversal

### 1. Problem Statement
Implement `bfsTraversal(graph: readonly (readonly number[])[], start: number): number[]`. First validate that `start` is a safe integer in `[0, graph.length)`. This check occurs before graph validation, so an empty graph always throws because it has no valid start. After `start` passes, validate every neighbor in the complete graph as a safe integer in `[0, graph.length)` before allocating traversal output or beginning traversal. Invalid input throws `RangeError`.

Traverse only vertices reachable from `start`, following each adjacency array's stored order. Directed graphs and symmetric undirected graphs are accepted; self-loops and parallel edges are valid. Mark a vertex when it is enqueued. Do not mutate the readonly graph, and return a fresh order array. The optimized solution must use a head index rather than `shift()`.

### 2. Example
```text
graph = [
  [1, 2],
  [3],
  [3, 4],
  [1],
  [],
  [0],
]
start = 0

result = [0, 1, 2, 3, 4]
```

Vertex `5` is valid but unreachable from `0`, so it is not returned. The repeated paths to `3` enqueue it only once.

### 3. Understand the Problem
BFS explores a reachable component in layers. Stored neighbor order breaks ties within a layer. The input contract deliberately validates the start first, then the entire graph, so an invalid neighbor in an unreachable component still rejects the call before output allocation.

#### In Simple Terms
Visit the start, then its unseen neighbors, then their unseen neighbors, continuing level by level.

### 4. Real-World Analogy
Imagine news spreading through contacts. The original person hears it first, then all direct contacts in listed order, then contacts two introductions away. Recording someone as soon as they are scheduled prevents duplicate invitations.

### 5. Key Observation
A FIFO queue processes vertices in discovery order, which is nondecreasing distance from the start. Marking on enqueue is essential: if two current vertices share a neighbor, the first one schedules it and the second one sees it as already claimed.

### 6. Pattern
```text
Pattern: Breadth-First Search with a Visited Set and FIFO Queue
Why this pattern applies: traversal must expand one distance layer at a time
How to recognize it: reachable-component order or unweighted edge distance
Similar problems: shortest unweighted paths, level order, multi-source spread
```

### 7. Brute Force
After validation, use an array as a FIFO queue and remove each next vertex with `shift()`. This is logically correct, but each removal can reindex all remaining elements. Mark neighbors before appending them so parallel edges, self-loops, and converging paths do not create duplicate queue entries.

#### Pseudocode
```text
validate start against graph length
validate every neighbor in the complete graph
mark start and enqueue it
while the queue is not empty:
  remove the first queue item with shift
  append it to traversal order
  for each stored neighbor:
    if unseen, mark it and enqueue it
return traversal order
```

#### Brute-Force TypeScript
```typescript
function bfsTraversalWithShift(
  graph: readonly (readonly number[])[],
  start: number,
): number[] {
  if (
    !Number.isSafeInteger(start) ||
    start < 0 ||
    start >= graph.length
  ) {
    throw new RangeError("start must be a valid vertex ID");
  }

  for (const neighbors of graph) {
    for (const neighbor of neighbors) {
      if (
        !Number.isSafeInteger(neighbor) ||
        neighbor < 0 ||
        neighbor >= graph.length
      ) {
        throw new RangeError("every neighbor must be a valid vertex ID");
      }
    }
  }

  const visited = new Array<boolean>(graph.length).fill(false);
  const queue: number[] = [start];
  const order: number[] = [];
  visited[start] = true;

  while (queue.length > 0) {
    const vertex = queue.shift()!;
    order.push(vertex);

    for (const neighbor of graph[vertex]!) {
      if (!visited[neighbor]!) {
        visited[neighbor] = true;
        queue.push(neighbor);
      }
    }
  }

  return order;
}
```

#### Visualization
```text
queue [0]          -> visit 0, enqueue 1,2
queue [1,2]        -> visit 1, enqueue 3
queue [2,3]        -> visit 2, skip 3, enqueue 4
queue [3,4]        -> visit 3, skip 1
queue [4]          -> visit 4
order [0,1,2,3,4]
```

#### Dry Run
Start `0` is validated before every neighbor. Traversal marks `0` immediately. Processing `0` schedules `1` then `2`; processing `1` schedules `3`. When `2` later sees `3`, it is already marked, so only `4` is added. Vertex `5` was validated but never scheduled because it is unreachable.

#### Brute-Force Complexity
Let `V` and `E` describe the complete graph's vertices and adjacency occurrences. Validation takes `O(V + E)`. Traversal inspects reachable occurrences once, but repeated `shift()` calls can move `O(V)` queue entries per removal, giving `O(V^2 + E)` worst-case total time. Visited state and the queue use `O(V)` auxiliary space; the fresh returned order uses `O(V)` output space in the worst case.

### 8. Why Brute Force Is Not Ideal
Removing index `0` from a JavaScript array repeatedly shifts the remaining queue contents. FIFO processing does not require physical removal; an index can identify the next unread entry while prior entries remain in owned storage.

### 9. Optimal Approach
Use the same full prevalidation and mark-on-enqueue rule. Store queued vertices in an array and maintain `head`, the index of the next unread item. Incrementing `head` is constant time, and each reachable vertex enters the queue exactly once.

### 10. Visualization
```text
queue storage: [0, 1, 2, 3, 4]
head:           ^

read queue[head], then increment head
processed entries stay in the array; no values are shifted
```

### 11. Step-by-Step Dry Run
| Read vertex | Newly enqueued | Queue storage after scan | Next head |
|---:|---|---|---:|
| `0` | `1, 2` | `[0,1,2]` | `1` |
| `1` | `3` | `[0,1,2,3]` | `2` |
| `2` | `4` | `[0,1,2,3,4]` | `3` |
| `3` | none | `[0,1,2,3,4]` | `4` |
| `4` | none | `[0,1,2,3,4]` | `5` |

### 12. TypeScript Solution
```typescript
function bfsTraversal(
  graph: readonly (readonly number[])[],
  start: number,
): number[] {
  if (
    !Number.isSafeInteger(start) ||
    start < 0 ||
    start >= graph.length
  ) {
    throw new RangeError("start must be a valid vertex ID");
  }

  for (const neighbors of graph) {
    for (const neighbor of neighbors) {
      if (
        !Number.isSafeInteger(neighbor) ||
        neighbor < 0 ||
        neighbor >= graph.length
      ) {
        throw new RangeError("every neighbor must be a valid vertex ID");
      }
    }
  }

  const visited = new Array<boolean>(graph.length).fill(false);
  const queue: number[] = [start];
  const order: number[] = [];
  let head = 0;
  visited[start] = true;

  while (head < queue.length) {
    const vertex = queue[head]!;
    head += 1;
    order.push(vertex);

    for (const neighbor of graph[vertex]!) {
      if (!visited[neighbor]!) {
        visited[neighbor] = true;
        queue.push(neighbor);
      }
    }
  }

  return order;
}
```

### 13. Code Walkthrough
The first conditional reads only `graph.length` and rejects an invalid start before inspecting adjacency entries. The nested validation loops then check the entire graph before `visited`, `queue`, or `order` is allocated. During traversal, `head` advances through queue storage, and each neighbor is marked in the same operation that schedules it.

### 14. Complexity Analysis
Complete validation takes `O(V + E)` time even when only a small component is reachable. Traversal then takes `O(Vr + Er)`, where `Vr` and `Er` are reachable vertices and inspected outgoing occurrences, so the overall worst case is `O(V + E)`. Visited and queue storage use `O(V)` auxiliary space, and the fresh traversal result uses up to `O(V)` output space.

### 15. Edge Cases
- An empty graph throws for every start before neighbor validation.
- A one-vertex graph with start `0` returns `[0]`, including with self-loops.
- Fractional, non-finite, unsafe, negative, or out-of-range starts throw first.
- An invalid neighbor in an unreachable component still throws before traversal allocation.
- Parallel edges and self-loops do not duplicate output vertices.
- Directed and symmetric undirected adjacency lists are both accepted.
- Disconnected valid vertices outside the start component are omitted.
- Input neighbor arrays remain unchanged.

### 16. Common Mistakes
- Validating graph neighbors before applying the documented start-first check.
- Returning `[]` for an empty graph despite the required start parameter.
- Validating only neighbors reached during traversal.
- Marking on dequeue and enqueuing the same vertex repeatedly.
- Calling `shift()` in the optimized queue.
- Sorting neighbors and changing stored-order traversal.
- Traversing all components instead of only the reachable component.
- Mutating adjacency arrays to record visited state.

### 17. Interview Follow-ups
1. Return unweighted distances. 2. Record a predecessor tree. 3. Traverse every component. 4. Support multiple starts. 5. Stop at a target while retaining full input validation.

### 18. Variations
- Multi-source BFS. - Shortest unweighted path. - Bipartite coloring. - Grid flood fill. - Level-grouped traversal.

### 19. Pattern Connections
BFS is level-order traversal generalized from trees to graphs, with visited state supplying the cycle and sharing protection that trees do not need. The head-index queue also appears in Kahn's topological sort and unweighted shortest paths. Full validation separates transport correctness from reachable execution scope.

### 20. NestJS Project Connection
For `POST /api/dsa/graphs/bfs`, bound vertex count, total adjacency occurrences, and per-vertex degree in the DTO before service execution. Validate `start` against the bounded outer length and retain the algorithm's full neighbor validation. A trace can emit bounded `{ vertex, queueLength, discovered }` events, but should never snapshot the queue or graph per step; include a truncation flag when the event budget is reached.

### 21. Mini Challenge
Implement `bfsDistances(graph, start): number[]` with the same start-first and full-graph validation order. Return `-1` for unreachable vertices, preserve stored neighbor order for discovery ties, and use a head-index queue.

## Problem 188 — DFS Traversal

### 1. Problem Statement
Implement `dfsTraversal(graph: readonly (readonly number[])[], start: number): number[]`. First validate that `start` is a safe integer in `[0, graph.length)`, then validate every neighbor in the complete graph as a safe integer in the same range before allocating output or traversing. An empty graph has no valid start and throws. Invalid input throws `RangeError`.

Traverse only the component reachable from `start` in the exact preorder produced by recursive DFS following stored neighbor order. Directed graphs and symmetric undirected graphs are accepted; parallel edges and self-loops are valid. Do not mutate the readonly graph, and return a fresh order array. The optimized solution must use explicit stack frames to reproduce recursion without call-stack risk.

### 2. Example
```text
graph = [
  [1, 2],
  [3, 4],
  [4],
  [2],
  [],
  [0],
]
start = 0

result = [0, 1, 3, 2, 4]
```

DFS finishes the branch through `1` and `3` before returning to consider `0`'s neighbor `2`. Vertex `5` is valid but unreachable and is omitted.

### 3. Understand the Problem
Recursive DFS pauses a vertex's neighbor loop while it completely explores one unseen neighbor. Exact order therefore depends not only on reversing neighbors but on when a vertex becomes visited. The complete graph is validated first even though traversal remains scoped to one reachable component.

#### In Simple Terms
Follow the first unseen neighbor as deeply as possible, then return to the exact neighbor position where that branch began.

### 4. Real-World Analogy
Imagine exploring a building with a checklist for each room's doors. You enter the first unexplored door, leaving a bookmark at the next door in the current room. When a branch ends, you return to that bookmark instead of restarting the room's checklist.

### 5. Key Observation
A stack frame containing `{ vertex, nextNeighborIndex }` stores precisely the state a recursive call leaves on the language call stack. Marking and emitting a neighbor when its frame is entered, then resuming the parent frame later, reproduces recursive preorder even when directed cross edges connect active branches.

### 6. Pattern
```text
Pattern: Depth-First Search with Explicit Iterator Frames
Why this pattern applies: exact recursive neighbor order must survive deep inputs
How to recognize it: preorder traversal with resumable per-vertex adjacency scans
Similar problems: iterative backtracking, path enumeration, directed cycle search
```

### 7. Brute Force
After full validation, run ordinary recursive DFS. Mark and append a vertex on entry, then recurse into each unseen neighbor in stored order. This is the clearest specification of the required order, but a long path can exceed the JavaScript call-stack limit.

#### Pseudocode
```text
validate start against graph length
validate every neighbor in the complete graph
visit(vertex):
  mark vertex and append it
  for each stored neighbor:
    if unseen: visit(neighbor)
visit(start)
return traversal order
```

#### Brute-Force TypeScript
```typescript
function dfsTraversalRecursively(
  graph: readonly (readonly number[])[],
  start: number,
): number[] {
  if (
    !Number.isSafeInteger(start) ||
    start < 0 ||
    start >= graph.length
  ) {
    throw new RangeError("start must be a valid vertex ID");
  }

  for (const neighbors of graph) {
    for (const neighbor of neighbors) {
      if (
        !Number.isSafeInteger(neighbor) ||
        neighbor < 0 ||
        neighbor >= graph.length
      ) {
        throw new RangeError("every neighbor must be a valid vertex ID");
      }
    }
  }

  const visited = new Array<boolean>(graph.length).fill(false);
  const order: number[] = [];

  const visit = (vertex: number): void => {
    visited[vertex] = true;
    order.push(vertex);

    for (const neighbor of graph[vertex]!) {
      if (!visited[neighbor]!) visit(neighbor);
    }
  };

  visit(start);
  return order;
}
```

#### Visualization
```text
enter 0 -> enter 1 -> enter 3 -> enter 2 -> enter 4
                        return
                  return
             return
       1 next sees 4 already visited -> return
0 next sees 2 already visited -> return
order: [0,1,3,2,4]
```

#### Dry Run
Starting at `0`, recursion enters `1`, then `3`, then follows `3`'s edge to `2`, and then enters `4`. Returning to `1`, its next neighbor `4` is already visited. Returning to `0`, neighbor `2` is also already visited. The call stack itself remembers every suspended next-neighbor position.

#### Brute-Force Complexity
Validation takes `O(V + E)`. Recursive traversal takes `O(Vr + Er)` for the reachable component, so worst-case total time is `O(V + E)`. The visited array and recursion stack use `O(V)` auxiliary space in the worst case; the fresh order uses `O(V)` output space. The recursion depth can exceed the JavaScript runtime's call-stack capacity before the asymptotic bound is exhausted.

### 8. Why Brute Force Is Not Ideal
The recursive implementation delegates traversal state to the runtime call stack. A valid path with many vertices can throw a stack-overflow error, making input depth an operational limit unrelated to the algorithm's theoretical memory bound.

### 9. Optimal Approach
Validate identically, then create a frame for the start vertex with next-neighbor index `0`. Inspect the top frame. If all its neighbors are consumed, pop it. Otherwise, advance its index and inspect that neighbor; when unseen, mark and emit it and push a new frame for it. This explicitly simulates recursive entry, suspension, and return.

A common shortcut pushes all neighbors in reverse and marks them immediately. On arbitrary directed graphs, early marking can let a later sibling claim a cross-edge target before the recursive first sibling reaches it, changing preorder. Frames avoid that mismatch while keeping at most one active frame per visited vertex.

### 10. Visualization
```text
frames [{0,next:0}]
take 0->1, advance 0 bookmark -> [{0,next:1},{1,next:0}]
take 1->3, advance 1 bookmark -> [{0,next:1},{1,next:1},{3,next:0}]
take 3->2                         -> push frame for 2
take 2->4                         -> push frame for 4
pop completed frames and resume their saved indexes
```

### 11. Step-by-Step Dry Run
| Action | Emitted | Active frame vertices / next indexes |
|---|---|---|
| enter start | `0` | `0/0` |
| follow `0 -> 1` | `1` | `0/1, 1/0` |
| follow `1 -> 3` | `3` | `0/1, 1/1, 3/0` |
| follow `3 -> 2` | `2` | `0/1, 1/1, 3/1, 2/0` |
| follow `2 -> 4` | `4` | `0/1, 1/1, 3/1, 2/1, 4/0` |
| finish and resume | none | frames pop back to empty |

### 12. TypeScript Solution
```typescript
function dfsTraversal(
  graph: readonly (readonly number[])[],
  start: number,
): number[] {
  if (
    !Number.isSafeInteger(start) ||
    start < 0 ||
    start >= graph.length
  ) {
    throw new RangeError("start must be a valid vertex ID");
  }

  for (const neighbors of graph) {
    for (const neighbor of neighbors) {
      if (
        !Number.isSafeInteger(neighbor) ||
        neighbor < 0 ||
        neighbor >= graph.length
      ) {
        throw new RangeError("every neighbor must be a valid vertex ID");
      }
    }
  }

  interface Frame {
    vertex: number;
    nextNeighborIndex: number;
  }

  const visited = new Array<boolean>(graph.length).fill(false);
  const order: number[] = [start];
  const stack: Frame[] = [{ vertex: start, nextNeighborIndex: 0 }];
  visited[start] = true;

  while (stack.length > 0) {
    const frame = stack[stack.length - 1]!;
    const neighbors = graph[frame.vertex]!;

    if (frame.nextNeighborIndex >= neighbors.length) {
      stack.pop();
      continue;
    }

    const neighbor = neighbors[frame.nextNeighborIndex]!;
    frame.nextNeighborIndex += 1;
    if (!visited[neighbor]!) {
      visited[neighbor] = true;
      order.push(neighbor);
      stack.push({ vertex: neighbor, nextNeighborIndex: 0 });
    }
  }

  return order;
}
```

### 13. Code Walkthrough
The function applies the start-first check and complete neighbor validation before declaring traversal state. Each frame owns one vertex's resume index. The parent index advances before a child frame is pushed, exactly like a suspended loop around a recursive call. A vertex is marked and emitted at frame entry, while completed frames are simply removed.

### 14. Complexity Analysis
Complete validation takes `O(V + E)`. Each reachable frame is pushed and popped once, and each reachable outgoing adjacency occurrence advances one frame index once, so traversal takes `O(Vr + Er)` and the overall worst case is `O(V + E)`. Visited state and at most `V` active frames use `O(V)` auxiliary space; the fresh order uses up to `O(V)` output space.

### 15. Edge Cases
- An empty graph throws because no start is in range.
- A single isolated start returns `[start]`.
- Self-loops and parallel edges are inspected but never duplicate output vertices.
- An invalid start throws before any malformed neighbor is inspected.
- An invalid neighbor in an unreachable component still throws before traversal state allocation.
- Directed cross edges preserve recursive stored-neighbor preorder.
- Disconnected valid vertices outside the reachable component are omitted.
- Deep paths consume heap-allocated frame storage instead of recursive call depth.

### 16. Common Mistakes
- Validating only the reachable component.
- Allocating output before full graph validation.
- Omitting visited state on cyclic or undirected input.
- Sorting adjacency arrays instead of preserving stored order.
- Pushing neighbors forward and reversing the intended DFS order.
- Marking all reverse-pushed neighbors immediately and assuming arbitrary cross-edge order matches recursion.
- Marking only on frame completion and repeatedly entering cycles.
- Using recursion in the production solution despite adversarial depth.

### 17. Interview Follow-ups
1. Return discovery and finish times. 2. Record parent edges. 3. Traverse every component. 4. Detect directed cycles with color states. 5. Produce postorder using the same frames.

### 18. Variations
- DFS forest. - Iterative postorder. - Directed cycle detection. - Strongly connected components. - Path enumeration with backtracking.

### 19. Pattern Connections
Frame-based DFS is explicit recursion simulation: the stack records local loop state, not merely vertex IDs. That same technique supports postorder, backtracking, and algorithms that need entry and exit events. Full prevalidation keeps traversal logic focused on graph exploration and makes rejection independent of reachability.

### 20. NestJS Project Connection
For `POST /api/dsa/graphs/dfs`, bound vertex count, total adjacency occurrences, and per-vertex degree before invoking the service. The iterative frame implementation avoids request crashes from adversarial recursion depth, but memory and CPU budgets still need enforcement. Emit only bounded `{ event: "enter" | "exit", vertex, depth }` traces, cap depth and event counts independently, mark truncation, and avoid serializing full stack or graph snapshots.

### 21. Mini Challenge
Implement `dfsTraversalWithTimes` using explicit frames and the same validation order. Return preorder plus safe-integer discovery and finish timestamps, preserve stored neighbor order, and emit each vertex's finish event exactly when its frame is popped.

## Problem 189 — Detect Cycle in Undirected Graph

### 1. Problem Statement
Implement `hasUndirectedCycle(graph: readonly (readonly number[])[]): boolean`. The input must be an undirected adjacency occurrence multigraph: every neighbor must be a safe integer vertex ID, every non-self occurrence `u -> v` must have a matching `v -> u` occurrence, and the two occurrence counts must be equal. A self-loop appears once and is its own reverse. Validate every ID and all occurrence multiplicities before cycle detection; invalid input throws `RangeError` without beginning traversal.

Inspect every connected component without mutating the graph. A self-loop is a cycle, and two parallel undirected edge occurrences between the same vertices form a two-edge cycle. Return `true` exactly when any component contains a cycle.

### 2. Example
```text
graph = [
  [1, 1],
  [0, 0],
  [],
]

result = true
```

The two occurrences each way describe two parallel undirected edges. Following one edge from `0` to `1` and the other back to `0` forms a cycle.

### 3. Understand the Problem
In a simple undirected graph, traversal ignores the edge that immediately returns to a vertex's parent. In an occurrence multigraph, it must ignore exactly one such occurrence. A second parent occurrence is a different parallel edge and therefore closes a two-edge cycle.

#### In Simple Terms
Check that every undirected edge is represented equally in both directions, then explore all components while forgiving only one immediate edge back to each parent.

### 4. Real-World Analogy
Imagine corridors recorded once at each endpoint. Before inspecting routes, verify that both rooms report the same number of corridors between them. During a walk, one corridor leads back to the previous room; another corridor to that same room creates a distinct round trip.

### 5. Key Observation
Per-vertex maps from neighbor ID to occurrence count validate reverse multiplicity in average `O(V + E)` time. After validation, parent tracking is sufficient only when paired with a per-vertex `skippedParentOccurrence` flag. Self-loops need no special traversal branch because the current vertex is already visited.

### 6. Pattern
```text
Pattern: Undirected DFS/BFS with Parent-Occurrence Tracking
Why this pattern applies: one reverse occurrence is the tree edge, but another is a cycle
How to recognize it: symmetric adjacency with self-loops or parallel edge occurrences
Similar problems: bridge finding in multigraphs, undirected components, edge validation
```

### 7. Brute Force
Build occurrence-count maps and fully validate symmetry, then run recursive DFS from every unvisited vertex. Each call skips its first occurrence of the parent and treats any other edge to a visited vertex as a cycle. This is already asymptotically optimal, but recursion can overflow on a long valid path.

#### Pseudocode
```text
validate every neighbor ID and count each occurrence
compare every non-self count with its reverse count
for each unvisited vertex:
  recursively visit it with no parent
  skip exactly one occurrence of the parent
  if any other neighbor is visited: return true
return false
```

#### Brute-Force TypeScript
```typescript
function hasUndirectedCycleRecursively(
  graph: readonly (readonly number[])[],
): boolean {
  const counts: Map<number, number>[] = Array.from(
    { length: graph.length },
    () => new Map<number, number>(),
  );

  for (let vertex = 0; vertex < graph.length; vertex += 1) {
    for (const neighbor of graph[vertex]!) {
      if (
        !Number.isSafeInteger(neighbor) ||
        neighbor < 0 ||
        neighbor >= graph.length
      ) {
        throw new RangeError("every neighbor must be a valid vertex ID");
      }
      const vertexCounts = counts[vertex]!;
      vertexCounts.set(neighbor, (vertexCounts.get(neighbor) ?? 0) + 1);
    }
  }

  for (let vertex = 0; vertex < counts.length; vertex += 1) {
    for (const [neighbor, count] of counts[vertex]!) {
      if (neighbor !== vertex && counts[neighbor]!.get(vertex) !== count) {
        throw new RangeError("undirected edge occurrences must be symmetric");
      }
    }
  }

  const visited = new Array<boolean>(graph.length).fill(false);

  const visit = (vertex: number, parent: number): boolean => {
    visited[vertex] = true;
    let skippedParentOccurrence = false;

    for (const neighbor of graph[vertex]!) {
      if (neighbor === parent && !skippedParentOccurrence) {
        skippedParentOccurrence = true;
      } else if (visited[neighbor]!) {
        return true;
      } else if (visit(neighbor, vertex)) {
        return true;
      }
    }

    return false;
  };

  for (let vertex = 0; vertex < graph.length; vertex += 1) {
    if (!visited[vertex]! && visit(vertex, -1)) return true;
  }
  return false;
}
```

#### Visualization
```text
0 == 1     two parallel edges

visit 0, then enter 1 through the first edge
at 1: skip one 1 -> 0 parent occurrence
at 1: second 1 -> 0 reaches visited 0 => cycle
```

#### Dry Run
Validation counts two `0 -> 1` occurrences and two `1 -> 0` occurrences, so the representation is symmetric. DFS marks `0`, enters `1`, and skips the first edge from `1` back to parent `0`. The second edge reaches the already visited parent and returns `true`.

#### Brute-Force Complexity
Validation and traversal each take `O(V + E)` time, where `E` counts adjacency occurrences, so total time is `O(V + E)`. Count maps, visited state, and recursive call frames use `O(V + E)` auxiliary space. Recursion can exceed the JavaScript call-stack limit on a deep component.

### 8. Why Brute Force Is Not Ideal
The recursive version is not slower asymptotically, but it stores one runtime call frame per vertex on a long path. A structurally valid request can therefore fail with stack overflow before available heap memory is exhausted.

### 9. Optimal Approach
Keep the same complete count-map validation, then traverse components with a head-index queue. Queue entries store a vertex and its parent. While scanning one vertex, skip exactly the first parent occurrence. Any remaining edge to a visited vertex proves a cycle; otherwise mark the neighbor before enqueueing it.

Marking on enqueue ensures a vertex has one tree parent. If another processed edge reaches it, that edge is not the single tree-edge reverse and closes a cycle. The method handles disconnected graphs, self-loops, and parallel edges without changing stored neighbor order.

### 10. Visualization
```text
component start: enqueue (0, no parent), mark 0
read 0: enqueue (1, parent 0), mark 1
read 1: first 0 is the parent occurrence -> skip
        second 0 is already visited       -> cycle
```

### 11. Step-by-Step Dry Run
| Queue entry | Neighbor | Action | Result |
|---|---:|---|---|
| `(0, -1)` | `1` | mark and enqueue `(1, 0)` | continue |
| `(0, -1)` | `1` | already visited | cycle |

The optimized traversal may detect the parallel cycle while still scanning vertex `0`; that is equivalent to detecting the second parent occurrence later at vertex `1`.

### 12. TypeScript Solution
```typescript
function hasUndirectedCycle(
  graph: readonly (readonly number[])[],
): boolean {
  const counts: Map<number, number>[] = Array.from(
    { length: graph.length },
    () => new Map<number, number>(),
  );

  for (let vertex = 0; vertex < graph.length; vertex += 1) {
    for (const neighbor of graph[vertex]!) {
      if (
        !Number.isSafeInteger(neighbor) ||
        neighbor < 0 ||
        neighbor >= graph.length
      ) {
        throw new RangeError("every neighbor must be a valid vertex ID");
      }
      const vertexCounts = counts[vertex]!;
      vertexCounts.set(neighbor, (vertexCounts.get(neighbor) ?? 0) + 1);
    }
  }

  for (let vertex = 0; vertex < counts.length; vertex += 1) {
    for (const [neighbor, count] of counts[vertex]!) {
      if (neighbor !== vertex && counts[neighbor]!.get(vertex) !== count) {
        throw new RangeError("undirected edge occurrences must be symmetric");
      }
    }
  }

  interface QueueEntry {
    vertex: number;
    parent: number;
  }

  const visited = new Array<boolean>(graph.length).fill(false);

  for (let start = 0; start < graph.length; start += 1) {
    if (visited[start]!) continue;

    const queue: QueueEntry[] = [{ vertex: start, parent: -1 }];
    let head = 0;
    visited[start] = true;

    while (head < queue.length) {
      const { vertex, parent } = queue[head]!;
      head += 1;
      let skippedParentOccurrence = false;

      for (const neighbor of graph[vertex]!) {
        if (neighbor === parent && !skippedParentOccurrence) {
          skippedParentOccurrence = true;
        } else if (visited[neighbor]!) {
          return true;
        } else {
          visited[neighbor] = true;
          queue.push({ vertex: neighbor, parent: vertex });
        }
      }
    }
  }

  return false;
}
```

### 13. Code Walkthrough
The first pass validates IDs while constructing one count map per source vertex. The second pass compares every non-self count with its reverse; only after both passes succeed is `visited` allocated. Each queue entry fixes one tree parent, and the local boolean consumes exactly one reverse occurrence of that tree edge.

### 14. Complexity Analysis
Counting, reverse-count comparison, and traversal each take average `O(V + E)` time, so total average time is `O(V + E)`. The maps use `O(E)` entries in the worst case, while visited and queue storage use `O(V)`, for `O(V + E)` auxiliary space. Hash-map operations provide the stated average-time bound.

### 15. Edge Cases
- An empty graph is valid and acyclic.
- One isolated vertex is acyclic.
- One self-loop occurrence is symmetric by itself and is a cycle.
- Two occurrences between the same distinct endpoints, represented twice each way, form a cycle.
- One edge occurrence represented only one way throws before traversal.
- Unequal reverse multiplicities throw even if a cycle is visible elsewhere.
- Invalid IDs in any component throw before cycle state is allocated.
- Disconnected components are all inspected.

### 16. Common Mistakes
- Checking only whether a reverse neighbor exists instead of comparing occurrence counts.
- Requiring two stored occurrences for a self-loop.
- Skipping every edge to the parent and missing a parallel-edge cycle.
- Treating every edge to a visited vertex as harmless in an undirected graph.
- Traversing only the component containing vertex `0`.
- Beginning cycle detection before complete symmetry validation.
- Calling `shift()` for the optimized queue.

### 17. Interview Follow-ups
1. Return one cycle's vertices. 2. Count connected components. 3. Detect bridges while preserving parallel-edge semantics. 4. Accept an edge list instead. 5. Report the first asymmetric occurrence pair.

### 18. Variations
- Simple-graph cycle detection. - Disjoint-set cycle detection. - Bipartite checking. - Bridge detection. - Dynamic edge insertion.

### 19. Pattern Connections
Parent tracking distinguishes the expected reverse of a DFS or BFS tree edge from a genuine undirected back or cross edge. Occurrence multiplicity extends that familiar pattern from simple graphs to multigraphs. The validation maps also mirror frequency-map techniques used for anagrams and multiset equality.

### 20. NestJS Project Connection
For `POST /api/dsa/graphs/undirected-cycle`, bound vertex count, total adjacency occurrences, and per-vertex degree before service execution. Count-map validation should still run in the service so symmetry is enforced at the algorithm boundary. A bounded trace may emit `{ vertex, parent, neighbor, action }`; cap events and never serialize full maps, queues, or adjacency lists per step.

### 21. Mini Challenge
Implement `findUndirectedCycle` with the same atomic validation and multigraph rules. Return one deterministic cycle as vertex IDs, including `[v, v]` for a self-loop and `[u, v, u]` for a parallel two-edge cycle.

## Problem 190 — Detect Cycle in Directed Graph

### 1. Problem Statement
Implement `hasDirectedCycle(graph: readonly (readonly number[])[]): boolean`. Validate every adjacency entry in the complete graph as a safe integer in `[0, graph.length)` before allocating traversal state. Invalid input throws `RangeError`. Then inspect every component and return whether any directed cycle exists.

The graph may be empty or disconnected. Self-loops and parallel directed edges are valid. Do not mutate or sort the graph; stored neighbor order does not affect the boolean result.

### 2. Example
```text
graph = [
  [1],
  [2],
  [0, 3],
  [],
]

result = true
```

The edges `0 -> 1 -> 2 -> 0` form a directed cycle. The outgoing edge from `2` to `3` does not change that result.

### 3. Understand the Problem
A directed edge to any previously visited vertex does not necessarily form a cycle. It is a cycle only when the destination can lead back to the source, or equivalently during DFS, when an edge points to a vertex still active on the current path.

#### In Simple Terms
Follow arrow directions and detect whether some nonempty directed route can return to where it began.

### 4. Real-World Analogy
Treat vertices as tasks and arrows as dependencies. Reusing a task already completed elsewhere is harmless. A problem occurs when the current unfinished dependency chain points back to one of its own unfinished tasks.

### 5. Key Observation
For any edge `source -> target`, a cycle exists through that edge exactly when `target` can reach `source`. Testing that fact separately is correct but expensive. Three-color DFS shares work: gray means active, black means finished, and an edge to gray proves a return to the current path.

### 6. Pattern
```text
Pattern: Directed DFS with Three Colors and Explicit Frames
Why this pattern applies: active-path membership, not global visited state, identifies a back edge
How to recognize it: arrows, dependencies, or recursion states that can revisit unfinished work
Similar problems: topological sorting, course scheduling, dependency validation
```

### 7. Brute Force
After full validation, consider every directed edge. Starting at that edge's target, run BFS to see whether its source is reachable. A self-loop succeeds immediately. Reinitialize visited state for each edge, which makes the method simple and executable but repeats substantial work.

#### Pseudocode
```text
validate every neighbor in the complete graph
for each edge source -> target:
  if source equals target: return true
  BFS from target
  if BFS reaches source: return true
return false
```

#### Brute-Force TypeScript
```typescript
function hasDirectedCycleByRepeatedReachability(
  graph: readonly (readonly number[])[],
): boolean {
  for (const neighbors of graph) {
    for (const neighbor of neighbors) {
      if (
        !Number.isSafeInteger(neighbor) ||
        neighbor < 0 ||
        neighbor >= graph.length
      ) {
        throw new RangeError("every neighbor must be a valid vertex ID");
      }
    }
  }

  for (let source = 0; source < graph.length; source += 1) {
    for (const target of graph[source]!) {
      if (target === source) return true;

      const visited = new Array<boolean>(graph.length).fill(false);
      const queue: number[] = [target];
      let head = 0;
      visited[target] = true;

      while (head < queue.length) {
        const vertex = queue[head]!;
        head += 1;

        for (const neighbor of graph[vertex]!) {
          if (neighbor === source) return true;
          if (!visited[neighbor]!) {
            visited[neighbor] = true;
            queue.push(neighbor);
          }
        }
      }
    }
  }

  return false;
}
```

#### Visualization
```text
test edge 0 -> 1
start BFS at 1: 1 -> 2 -> 0
source 0 is reachable, so 0 -> 1 closes a cycle
```

#### Dry Run
For edge `0 -> 1`, BFS begins at `1`. It enqueues `2`, then sees `2 -> 0`. Because `0` is the tested edge's source, the route `1 -> 2 -> 0` combines with `0 -> 1` and the function returns `true`.

#### Brute-Force Complexity
Complete validation takes `O(V + E)`. Up to `E` reachability searches each take `O(V + E)`, giving `O(E(V + E))` worst-case time. A visited array and head-index queue use `O(V)` auxiliary space per search.

### 8. Why Brute Force Is Not Ideal
Neighboring edges often launch searches across the same vertices and edges. The method discards all reachability work after each edge, so dense acyclic graphs can trigger many complete traversals before returning `false`.

### 9. Optimal Approach
Use colors `0` for unvisited, `1` for active, and `2` for finished. For each unvisited start, mark it active and push a frame containing its next neighbor index. An edge to color `1` proves a cycle. An edge to color `0` activates a child frame. When a frame exhausts its neighbors, color that vertex `2` and pop it.

Frames avoid recursive call-stack limits while preserving the entry and exit events needed by the color invariant. Starting from every still-unvisited vertex covers disconnected components.

### 10. Visualization
```text
enter 0: colors [1,0,0,0]
enter 1: colors [1,1,0,0]
enter 2: colors [1,1,1,0]
edge 2 -> 0 sees active color 1 => cycle
```

### 11. Step-by-Step Dry Run
| Top frame | Edge | Destination color | Action |
|---|---|---:|---|
| `0/0` | `0 -> 1` | `0` | mark `1` active, push frame |
| `1/0` | `1 -> 2` | `0` | mark `2` active, push frame |
| `2/0` | `2 -> 0` | `1` | return `true` |

### 12. TypeScript Solution
```typescript
function hasDirectedCycle(
  graph: readonly (readonly number[])[],
): boolean {
  for (const neighbors of graph) {
    for (const neighbor of neighbors) {
      if (
        !Number.isSafeInteger(neighbor) ||
        neighbor < 0 ||
        neighbor >= graph.length
      ) {
        throw new RangeError("every neighbor must be a valid vertex ID");
      }
    }
  }

  interface Frame {
    vertex: number;
    nextNeighborIndex: number;
  }

  const colors = new Array<number>(graph.length).fill(0);

  for (let start = 0; start < graph.length; start += 1) {
    if (colors[start] !== 0) continue;

    const stack: Frame[] = [{ vertex: start, nextNeighborIndex: 0 }];
    colors[start] = 1;

    while (stack.length > 0) {
      const frame = stack[stack.length - 1]!;
      const neighbors = graph[frame.vertex]!;

      if (frame.nextNeighborIndex >= neighbors.length) {
        colors[frame.vertex] = 2;
        stack.pop();
        continue;
      }

      const neighbor = neighbors[frame.nextNeighborIndex]!;
      frame.nextNeighborIndex += 1;

      if (colors[neighbor] === 1) return true;
      if (colors[neighbor] === 0) {
        colors[neighbor] = 1;
        stack.push({ vertex: neighbor, nextNeighborIndex: 0 });
      }
    }
  }

  return false;
}
```

### 13. Code Walkthrough
The validation loops finish before `colors` exists. Each frame resumes one adjacency scan. Color changes to `1` occur when a frame is entered, and color changes to `2` occur only when that frame is exhausted, so color `1` exactly represents membership on the active DFS path.

### 14. Complexity Analysis
Validation takes `O(V + E)`. Each vertex enters and exits one frame, and each adjacency occurrence is inspected once during the color traversal, so total time is `O(V + E)`. Colors and at most `V` frames use `O(V)` auxiliary space.

### 15. Edge Cases
- An empty graph is valid and acyclic.
- A self-loop returns `true`.
- Parallel edges alone do not form a directed cycle unless a return path exists.
- An edge to a finished vertex is not mistaken for a cycle.
- A cycle in a component disconnected from vertex `0` is found.
- Invalid IDs anywhere throw before color state is allocated.
- A long directed path is handled without recursive stack growth.

### 16. Common Mistakes
- Treating every edge to a globally visited vertex as a directed cycle.
- Marking a vertex finished before all descendants finish.
- Searching only from vertex `0`.
- Validating only reached neighbors.
- Using recursion for adversarial-depth production input.
- Forgetting that a self-loop points to an active vertex immediately.
- Calling `shift()` in the repeated-reachability BFS.

### 17. Interview Follow-ups
1. Return one directed cycle. 2. Produce a topological order when acyclic. 3. Use Kahn's indegree algorithm. 4. Detect cycles after edge insertion. 5. Return strongly connected components.

### 18. Variations
- Course schedule feasibility. - Dependency deadlock detection. - Topological sorting. - Eventual safe states. - Strongly connected components.

### 19. Pattern Connections
Three-color DFS separates vertices never seen, active on the current path, and completely resolved. That distinction is the directed analogue of parent tracking in undirected graphs. Frame exit events also support postorder and topological ordering.

### 20. NestJS Project Connection
For `POST /api/dsa/graphs/directed-cycle`, bound vertices, total adjacency occurrences, and per-vertex degree in the DTO. Keep full ID validation at the service boundary and use iterative frames to tolerate deep paths. Trace only bounded `{ event: "enter" | "edge" | "exit", vertex, neighbor? }` records, with a truncation flag and no full color or stack snapshots.

### 21. Mini Challenge
Implement `findDirectedCycle(graph): number[] | null` with the same validation order and iterative frames. Track parents so an edge to an active vertex can reconstruct one deterministic closed cycle.

## Problem 191 — Number of Islands

### 1. Problem Statement
Implement `countIslands(grid: readonly (readonly (0 | 1)[])[]): number`, where `0` is water and `1` is land. Validate the complete grid before returning a result: every row must have the same length and every cell must be exactly `0` or `1`. Invalid input throws `RangeError`.

An island is a maximal group of land cells connected up, right, down, or left; diagonal contact does not connect islands. Empty grids and grids with zero columns are valid and contain zero islands. Do not mutate the input.

### 2. Example
```text
grid = [
  [1, 1, 0, 0],
  [0, 1, 0, 1],
  [1, 0, 0, 1],
]

result = 3
```

The top-left land group is one island, the two rightmost vertical cells form another, and the bottom-left cell is the third. Diagonal contact does not merge them.

### 3. Understand the Problem
Each land cell is a vertex in an implicit graph. Four-direction adjacency supplies edges without building a graph object. Scanning every cell starts one traversal exactly when an unvisited land component is discovered.

#### In Simple Terms
Count how many times a row-by-row scan finds land that has not already been reached from earlier land.

### 4. Real-World Analogy
Think of an aerial map divided into square plots. Surveyors may cross shared sides but not corners. Each time the scan reaches an unsurveyed land plot, one team covers that entire connected landmass before the scan continues.

### 5. Key Observation
The first unvisited cell encountered in each island is unique, so increment once and flood all four-directionally connected land. Complete validation must precede that scan so a malformed later row cannot be hidden by an early result.

### 6. Pattern
```text
Pattern: Connected Components in an Implicit Grid Graph
Why this pattern applies: cells are vertices and four-direction neighbors are generated on demand
How to recognize it: count regions under a local adjacency rule
Similar problems: flood fill, largest island, surrounded regions, maze traversal
```

### 7. Brute Force
Validate the grid, make a mutable deep copy, and scan it row by row. When a `1` is found, increment the count and recursively replace that entire island with `0`. The copy preserves input ownership, but recursion risks stack overflow on a large island.

#### Pseudocode
```text
validate rectangular shape and every cell domain
copy every row
for each cell in row-major order:
  when cell is 1:
    increment islands
    recursively erase it and neighbors up, right, down, left
return islands
```

#### Brute-Force TypeScript
```typescript
function countIslandsRecursively(
  grid: readonly (readonly (0 | 1)[])[],
): number {
  const columnCount = grid[0]?.length ?? 0;
  for (const row of grid) {
    if (row.length !== columnCount) {
      throw new RangeError("grid must be rectangular");
    }
    for (const cell of row) {
      if (cell !== 0 && cell !== 1) {
        throw new RangeError("every cell must be 0 or 1");
      }
    }
  }

  const working: (0 | 1)[][] = grid.map((row) => [...row]);
  const directions: readonly (readonly [number, number])[] = [
    [-1, 0],
    [0, 1],
    [1, 0],
    [0, -1],
  ];

  const erase = (row: number, column: number): void => {
    if (
      row < 0 ||
      row >= working.length ||
      column < 0 ||
      column >= columnCount ||
      working[row]![column] !== 1
    ) {
      return;
    }

    working[row]![column] = 0;
    for (const [rowDelta, columnDelta] of directions) {
      erase(row + rowDelta, column + columnDelta);
    }
  };

  let islands = 0;
  for (let row = 0; row < working.length; row += 1) {
    for (let column = 0; column < columnCount; column += 1) {
      if (working[row]![column] === 1) {
        islands += 1;
        erase(row, column);
      }
    }
  }
  return islands;
}
```

#### Visualization
```text
1 1 0 0      x x 0 0
0 1 0 1  ->  0 x 0 1   first recursive flood erases one island
1 0 0 1      1 0 0 1
```

#### Dry Run
The scan reaches `(0, 0)`, counts island one, and erases `(0, 0)`, `(0, 1)`, and `(1, 1)`. It later reaches `(1, 3)`, erases it with `(2, 3)`, and finally finds `(2, 0)`. No diagonal move joins `(1, 1)` to `(2, 0)`.

#### Brute-Force Complexity
Validation, copying, scanning, and flooding each touch at most `R * C` cells, so time is `O(RC)`. The working copy uses `O(RC)` auxiliary space, and recursive call depth can also reach `O(RC)` in the worst case.

### 8. Why Brute Force Is Not Ideal
The recursive calls depend on the JavaScript runtime stack. A large all-land grid can produce a valid traversal deeper than the runtime supports, even though enough heap memory is available for iterative state.

### 9. Optimal Approach
After the same full validation, allocate a rectangular boolean visited grid. Scan cells in row-major order. On each unvisited `1`, increment the count, mark it immediately, and run BFS using an array plus a head index. Generate neighbors strictly up, right, down, then left.

Marking on enqueue prevents duplicate queue entries when multiple land cells reach the same neighbor. The input remains untouched because land values are only read.

### 10. Visualization
```text
discover (0,0), island count 1
queue: (0,0) -> add right (0,1) -> add down (1,1)
discover (1,3), island count 2 -> add down (2,3)
discover (2,0), island count 3
```

### 11. Step-by-Step Dry Run
| Scan cell | State | Action | Islands |
|---|---|---|---:|
| `(0, 0)` | unvisited land | BFS first component | 1 |
| `(1, 3)` | unvisited land | BFS second component | 2 |
| `(2, 0)` | unvisited land | BFS third component | 3 |

### 12. TypeScript Solution
```typescript
function countIslands(
  grid: readonly (readonly (0 | 1)[])[],
): number {
  const columnCount = grid[0]?.length ?? 0;
  for (const row of grid) {
    if (row.length !== columnCount) {
      throw new RangeError("grid must be rectangular");
    }
    for (const cell of row) {
      if (cell !== 0 && cell !== 1) {
        throw new RangeError("every cell must be 0 or 1");
      }
    }
  }

  interface Cell {
    row: number;
    column: number;
  }

  const directions: readonly (readonly [number, number])[] = [
    [-1, 0],
    [0, 1],
    [1, 0],
    [0, -1],
  ];
  const visited: boolean[][] = Array.from(
    { length: grid.length },
    () => new Array<boolean>(columnCount).fill(false),
  );
  let islands = 0;

  for (let row = 0; row < grid.length; row += 1) {
    for (let column = 0; column < columnCount; column += 1) {
      if (grid[row]![column] !== 1 || visited[row]![column]!) continue;

      islands += 1;
      visited[row]![column] = true;
      const queue: Cell[] = [{ row, column }];
      let head = 0;

      while (head < queue.length) {
        const current = queue[head]!;
        head += 1;

        for (const [rowDelta, columnDelta] of directions) {
          const nextRow = current.row + rowDelta;
          const nextColumn = current.column + columnDelta;
          if (
            nextRow >= 0 &&
            nextRow < grid.length &&
            nextColumn >= 0 &&
            nextColumn < columnCount &&
            grid[nextRow]![nextColumn] === 1 &&
            !visited[nextRow]![nextColumn]!
          ) {
            visited[nextRow]![nextColumn] = true;
            queue.push({ row: nextRow, column: nextColumn });
          }
        }
      }
    }
  }

  return islands;
}
```

### 13. Code Walkthrough
`columnCount` is zero for `[]` and for a first empty row. Every row length and cell is checked before visited state exists. The outer scan identifies component starts, while BFS marks each land cell at enqueue time and generates neighbors in the promised deterministic order.

### 14. Complexity Analysis
Validation and traversal each take `O(RC)` time. Every land cell is enqueued at most once, and each checks four directions. The visited grid and queue use `O(RC)` auxiliary space in the worst case. The numeric count is safe because JavaScript array dimensions cannot represent more cells than the safe-integer range.

### 15. Edge Cases
- `[]` returns `0`.
- `[[], []]` is rectangular and returns `0`.
- A jagged grid throws before returning or traversing.
- Runtime values outside exactly `0` and `1` throw.
- An all-water grid returns `0`.
- An all-land grid returns `1`.
- Diagonally touching land cells remain separate islands.
- The readonly input and all row arrays remain unchanged.

### 16. Common Mistakes
- Connecting diagonal cells.
- Validating rows only as traversal reaches them.
- Returning early for zero columns before checking later row lengths.
- Mutating the caller's grid to mark visited cells.
- Marking on dequeue and adding duplicate cells.
- Calling `shift()` in the optimized queue.
- Using recursion without acknowledging stack depth.

### 17. Interview Follow-ups
1. Return every island's area. 2. Find the largest island. 3. Support eight directions. 4. Return island boundaries. 5. Count shapes up to translation.

### 18. Variations
- Max area of island. - Number of enclaves. - Surrounded regions. - Distinct islands. - Dynamic island insertion.

### 19. Pattern Connections
Island counting is connected-component traversal where the graph is implicit in grid coordinates. It reuses visited state and full-component scanning from graph traversal, while its neighbor generator is the same foundation used by flood fill and multi-source grid BFS.

### 20. NestJS Project Connection
For `POST /api/dsa/grids/island-count`, bound rows, columns, and total cells before the service runs. Retain rectangular and domain validation inside the algorithm contract. A trace may emit bounded `{ event: "island" | "visit", row, column, island }` records; cap total events and avoid serializing the grid, queue, or visited matrix per step.

### 21. Mini Challenge
Implement `islandAreas(grid): number[]` with the same validation, ownership, and up/right/down/left order. Return areas in the row-major order in which their islands are first discovered.

## Problem 192 — Flood Fill

### 1. Problem Statement
Implement `floodFill(image: readonly (readonly number[])[], startRow: number, startColumn: number, newColor: number): number[][]`. Validate `newColor` first as a safe integer. Next validate the complete image as rectangular with every cell a safe integer. Finally validate both coordinates as safe integers in bounds. Any failure throws `RangeError` in that order.

Replace the start cell and every up/right/down/left-connected cell having its original color. Return a fresh deep copy and never mutate the input, even when `newColor` equals the source color. An empty or zero-column image has no valid start, so it throws only after image validation succeeds.

### 2. Example
```text
image = [
  [1, 1, 0],
  [1, 0, 0],
  [1, 1, 0],
]
start = (0, 0), newColor = 7

result = [
  [7, 7, 0],
  [7, 0, 0],
  [7, 7, 0],
]
```

Only cells connected to `(0, 0)` through side-sharing cells of color `1` are changed.

### 3. Understand the Problem
The selected region is one connected component in an implicit graph whose vertices are cells of the source color. Ownership is part of the contract: output rows must be fresh even when no color value changes.

#### In Simple Terms
Copy the picture, then recolor the entire same-colored region touching the chosen cell without changing the original picture.

### 4. Real-World Analogy
A paint-bucket tool pours paint into one pixel and lets it flow through side-connected pixels of the same original color. A diagonal touch is only a corner, so paint does not cross it.

### 5. Key Observation
Writing `newColor` when a cell is scheduled can double as the visited mark when the colors differ. When they are equal, traversal must stop immediately after copying because writing the same value cannot distinguish scheduled cells.

### 6. Pattern
```text
Pattern: Flood Fill over an Implicit Grid Component
Why this pattern applies: one start selects all equal-valued cells reachable in four directions
How to recognize it: replace or collect one connected region from a seed coordinate
Similar problems: image bucket fill, island area, region labeling, maze reachability
```

### 7. Brute Force
Apply the promised validation order, deep-copy every row, and return that copy immediately if the source already equals `newColor`. Otherwise recursively recolor the current cell before visiting neighbors up, right, down, and left. The copy acts as both output and visited state.

#### Pseudocode
```text
validate newColor
validate every image row and cell
validate start coordinates
deep-copy image
if source color equals newColor: return copy
recursively recolor matching cells up, right, down, left
return copy
```

#### Brute-Force TypeScript
```typescript
function floodFillRecursively(
  image: readonly (readonly number[])[],
  startRow: number,
  startColumn: number,
  newColor: number,
): number[][] {
  if (!Number.isSafeInteger(newColor)) {
    throw new RangeError("newColor must be a safe integer");
  }

  const columnCount = image[0]?.length ?? 0;
  for (const row of image) {
    if (row.length !== columnCount) {
      throw new RangeError("image must be rectangular");
    }
    for (const color of row) {
      if (!Number.isSafeInteger(color)) {
        throw new RangeError("every image color must be a safe integer");
      }
    }
  }

  if (
    !Number.isSafeInteger(startRow) ||
    !Number.isSafeInteger(startColumn) ||
    startRow < 0 ||
    startRow >= image.length ||
    startColumn < 0 ||
    startColumn >= columnCount
  ) {
    throw new RangeError("start coordinates must identify an image cell");
  }

  const result: number[][] = image.map((row) => [...row]);
  const sourceColor = result[startRow]![startColumn]!;
  if (sourceColor === newColor) return result;

  const directions: readonly (readonly [number, number])[] = [
    [-1, 0],
    [0, 1],
    [1, 0],
    [0, -1],
  ];

  const fill = (row: number, column: number): void => {
    if (
      row < 0 ||
      row >= result.length ||
      column < 0 ||
      column >= columnCount ||
      result[row]![column] !== sourceColor
    ) {
      return;
    }

    result[row]![column] = newColor;
    for (const [rowDelta, columnDelta] of directions) {
      fill(row + rowDelta, column + columnDelta);
    }
  };

  fill(startRow, startColumn);
  return result;
}
```

#### Visualization
```text
1 1 0      7 7 0
1 0 0  ->  7 0 0
1 1 0      7 7 0

visit order uses: up, right, down, left
```

#### Dry Run
The source color at `(0, 0)` is `1`. Recursion writes `7` there, checks up out of bounds, fills `(0, 1)`, then eventually reaches `(1, 0)`, `(2, 0)`, and `(2, 1)`. Every `0` blocks the fill, and the original rows are never written.

#### Brute-Force Complexity
Validation and copying take `O(RC)`. Flooding visits at most `RC` cells and checks four directions, so total time is `O(RC)`. The returned copy uses `O(RC)` output space; recursion uses up to `O(RC)` auxiliary call-stack space.

### 8. Why Brute Force Is Not Ideal
A large uniform image can create one recursive call per cell along a deep traversal path. That can overflow the runtime stack even though the image and output fit in memory.

### 9. Optimal Approach
After identical validation and copying, return the fresh copy for an unchanged color. Otherwise mark the start by recoloring it, enqueue it, and process the queue with a head index. For each cell, inspect up, right, down, and left; recolor matching neighbors before enqueueing them.

Recoloring on enqueue ensures each cell enters the queue at most once. No separate visited matrix is needed because `newColor` differs from the source color on the traversal path.

### 10. Visualization
```text
copy image, source = 1
paint and enqueue (0,0)
dequeue (0,0): enqueue right (0,1), then down (1,0)
continue until no adjacent source-colored cell remains
```

### 11. Step-by-Step Dry Run
| Dequeued | Newly recolored in direction order | Queue tail |
|---|---|---|
| `(0, 0)` | `(0, 1)`, `(1, 0)` | `(0, 1), (1, 0)` |
| `(0, 1)` | none | `(1, 0)` |
| `(1, 0)` | `(2, 0)` | `(2, 0)` |
| `(2, 0)` | `(2, 1)` | `(2, 1)` |

### 12. TypeScript Solution
```typescript
function floodFill(
  image: readonly (readonly number[])[],
  startRow: number,
  startColumn: number,
  newColor: number,
): number[][] {
  if (!Number.isSafeInteger(newColor)) {
    throw new RangeError("newColor must be a safe integer");
  }

  const columnCount = image[0]?.length ?? 0;
  for (const row of image) {
    if (row.length !== columnCount) {
      throw new RangeError("image must be rectangular");
    }
    for (const color of row) {
      if (!Number.isSafeInteger(color)) {
        throw new RangeError("every image color must be a safe integer");
      }
    }
  }

  if (
    !Number.isSafeInteger(startRow) ||
    !Number.isSafeInteger(startColumn) ||
    startRow < 0 ||
    startRow >= image.length ||
    startColumn < 0 ||
    startColumn >= columnCount
  ) {
    throw new RangeError("start coordinates must identify an image cell");
  }

  interface Cell {
    row: number;
    column: number;
  }

  const result: number[][] = image.map((row) => [...row]);
  const sourceColor = result[startRow]![startColumn]!;
  if (sourceColor === newColor) return result;

  const directions: readonly (readonly [number, number])[] = [
    [-1, 0],
    [0, 1],
    [1, 0],
    [0, -1],
  ];
  const queue: Cell[] = [{ row: startRow, column: startColumn }];
  let head = 0;
  result[startRow]![startColumn] = newColor;

  while (head < queue.length) {
    const current = queue[head]!;
    head += 1;

    for (const [rowDelta, columnDelta] of directions) {
      const nextRow = current.row + rowDelta;
      const nextColumn = current.column + columnDelta;
      if (
        nextRow >= 0 &&
        nextRow < result.length &&
        nextColumn >= 0 &&
        nextColumn < columnCount &&
        result[nextRow]![nextColumn] === sourceColor
      ) {
        result[nextRow]![nextColumn] = newColor;
        queue.push({ row: nextRow, column: nextColumn });
      }
    }
  }

  return result;
}
```

### 13. Code Walkthrough
The code performs all `newColor`, image, and coordinate checks before reading the source cell or allocating output. `result` deep-copies every row. In the queue version, writing before enqueue makes the copied image its own visited state while preserving the caller's image.

### 14. Complexity Analysis
Validation, deep copying, and traversal each take `O(RC)` time, so total time is `O(RC)`. The required result uses `O(RC)` output space. The optimized queue holds at most `O(RC)` cells as auxiliary space; no additional visited matrix is needed.

### 15. Edge Cases
- Invalid `newColor` throws before a malformed image or coordinates are inspected.
- A jagged row or unsafe cell throws before coordinate validation.
- Empty and zero-column images throw for every start after image validation.
- Fractional, unsafe, negative, or out-of-bounds coordinates throw.
- If `newColor` equals the source, a fresh deep copy is still returned.
- Diagonally matching cells are not filled.
- Negative safe-integer colors are valid.
- Neither the input outer array nor its rows are mutated.

### 16. Common Mistakes
- Validating coordinates before the complete image despite the promised order.
- Returning the original image when the color is unchanged.
- Shallow-copying only the outer array and mutating shared rows.
- Comparing neighbors with `newColor` instead of the original source color.
- Marking after dequeue and adding duplicate cells.
- Including diagonal directions.
- Calling `shift()` in the optimized queue.

### 17. Interview Follow-ups
1. Fill within a color tolerance. 2. Support eight directions. 3. Return the changed-cell count. 4. Fill multiple seeds. 5. Stream a bounded change trace.

### 18. Variations
- Boundary fill. - Region labeling. - Replace surrounded regions. - Image segmentation. - Multi-source recoloring.

### 19. Pattern Connections
Flood fill is a single-source connected-component traversal with the output copy doubling as visited state. It shares four-direction generation with island counting and the mark-on-enqueue discipline with graph BFS.

### 20. NestJS Project Connection
For `POST /api/dsa/grids/flood-fill`, bound rows, columns, total cells, and serialized payload size before service execution. Preserve the documented validation order in the service. A trace can emit bounded `{ row, column, oldColor, newColor }` changes; cap cells and trace events, and avoid returning intermediate image snapshots.

### 21. Mini Challenge
Implement `floodFillWithCount` with the same validation and ownership rules. Return `{ image, changedCells }`, where an unchanged-color request returns a fresh deep copy and `changedCells: 0`.

## Problem 193 — Rotting Oranges

### 1. Problem Statement
Implement `minutesUntilAllRotten(grid: readonly (readonly (0 | 1 | 2)[])[]): number`. Cell `0` is empty, `1` is a fresh orange, and `2` is a rotten orange. Validate the complete grid as rectangular with every cell exactly `0`, `1`, or `2` before returning a result. Invalid input throws `RangeError`.

Each minute, every rotten orange simultaneously rots adjacent fresh oranges up, right, down, and left. Return the minimum minutes until no fresh orange remains, or `-1` when some fresh orange is unreachable. Empty and zero-column grids return `0`. Do not mutate the input.

### 2. Example
```text
grid = [
  [2, 1, 1],
  [1, 1, 0],
  [0, 1, 1],
]

result = 4
```

Rot spreads outward from `(0, 0)`. The farthest fresh orange at `(2, 2)` becomes rotten after four minutes.

### 3. Understand the Problem
All initially rotten oranges spread at the same time, so this is shortest-distance traversal from multiple sources. Empty cells block movement, and each fresh orange's rotting time is its minimum four-direction distance from any initial rotten orange.

#### In Simple Terms
Start from every rotten orange together and count how many one-cell-per-minute waves are needed to reach all fresh oranges.

### 4. Real-World Analogy
Imagine several alarms spreading through adjacent occupied rooms at once. A room records the first minute any alarm reaches it. Empty rooms are walls, and an occupied room sealed away from every alarm is unreachable.

### 5. Key Observation
Enqueueing all initial rotten cells at minute `0` makes ordinary BFS process cells by minimum distance from the nearest source. Marking a fresh cell rotten when enqueued prevents later sources from scheduling it again.

### 6. Pattern
```text
Pattern: Multi-Source BFS on a Grid
Why this pattern applies: simultaneous spread asks for distance from the nearest of many starts
How to recognize it: waves expand one edge per time unit from all initial sources
Similar problems: nearest zero, fire spread, infection time, distance to facilities
```

### 7. Brute Force
Validate and deep-copy the grid. During each minute, scan the entire copy and collect every fresh cell adjacent to a currently rotten cell. Only after the full scan, rot the collected cells as one batch. If a scan finds none while fresh oranges remain, return `-1`.

Batching is essential: changing cells during the scan would let rot cascade multiple steps in one minute and make the answer depend on scan order.

#### Pseudocode
```text
validate rectangular shape and cell domain
copy grid and count fresh cells
while fresh remains:
  scan all cells and collect fresh cells touching rotten cells
  if batch is empty: return -1
  rot the whole batch and decrement fresh count
  increment minutes
return minutes
```

#### Brute-Force TypeScript
```typescript
function minutesUntilAllRottenByScanning(
  grid: readonly (readonly (0 | 1 | 2)[])[],
): number {
  const columnCount = grid[0]?.length ?? 0;
  for (const row of grid) {
    if (row.length !== columnCount) {
      throw new RangeError("grid must be rectangular");
    }
    for (const cell of row) {
      if (cell !== 0 && cell !== 1 && cell !== 2) {
        throw new RangeError("every cell must be 0, 1, or 2");
      }
    }
  }

  interface Cell {
    row: number;
    column: number;
  }

  const directions: readonly (readonly [number, number])[] = [
    [-1, 0],
    [0, 1],
    [1, 0],
    [0, -1],
  ];
  const working: (0 | 1 | 2)[][] = grid.map((row) => [...row]);
  let fresh = 0;
  for (const row of working) {
    for (const cell of row) {
      if (cell === 1) fresh += 1;
    }
  }

  let minutes = 0;
  while (fresh > 0) {
    const newlyRotten: Cell[] = [];

    for (let row = 0; row < working.length; row += 1) {
      for (let column = 0; column < columnCount; column += 1) {
        if (working[row]![column] !== 1) continue;

        let touchesRotten = false;
        for (const [rowDelta, columnDelta] of directions) {
          const nextRow = row + rowDelta;
          const nextColumn = column + columnDelta;
          if (
            nextRow >= 0 &&
            nextRow < working.length &&
            nextColumn >= 0 &&
            nextColumn < columnCount &&
            working[nextRow]![nextColumn] === 2
          ) {
            touchesRotten = true;
            break;
          }
        }

        if (touchesRotten) newlyRotten.push({ row, column });
      }
    }

    if (newlyRotten.length === 0) return -1;
    for (const cell of newlyRotten) {
      working[cell.row]![cell.column] = 2;
      fresh -= 1;
    }
    minutes += 1;
  }

  return minutes;
}
```

#### Visualization
```text
minute 0       minute 1       minute 2
2 1 1          2 2 1          2 2 2
1 1 0    ->    2 1 0    ->    2 2 0
0 1 1          0 1 1          0 1 1

each displayed step applies one complete batch
```

#### Dry Run
The first scan batches `(0, 1)` and `(1, 0)`. Only after scanning does it mark both rotten. Later batches rot cells at increasing distance, with `(2, 2)` in minute four. Because newly rotten cells cannot affect their own scan, each loop represents exactly one minute.

#### Brute-Force Complexity
Validation and copying take `O(RC)`. At most `RC` productive minute scans each inspect `RC` cells and up to four neighbors, so worst-case time is `O((RC)^2)`. The working copy and one batch use `O(RC)` auxiliary space.

### 8. Why Brute Force Is Not Ideal
The simulation repeatedly scans empty cells and already rotten cells that can no longer change. A long narrow route may require `O(RC)` minutes, causing almost the entire grid to be reread for every single wave.

### 9. Optimal Approach
Validate first, then scan once to count fresh oranges and enqueue every initially rotten cell at minute `0`. Maintain a boolean matrix marking cells already rotten or scheduled to rot. Process the queue with a head index. When a fresh unmarked neighbor is found, mark it immediately, decrement `fresh`, and enqueue it with `minute + 1`.

This is multi-source BFS: all sources share one queue, so the first time a fresh orange is reached is its minimum rotting minute. Track the largest enqueued minute and return it if `fresh` reaches zero; otherwise return `-1` after the queue empties.

### 10. Visualization
```text
initial queue: every cell containing 2, each at minute 0
minute 1 entries: all fresh neighbors of any initial source
minute 2 entries: all newly reached fresh neighbors
...
one shared queue merges overlapping waves without duplicates
```

### 11. Step-by-Step Dry Run
| Minute | Newly rotten cells | Fresh remaining |
|---:|---|---:|
| 0 | initial `(0, 0)` | 6 |
| 1 | `(0, 1)`, `(1, 0)` | 4 |
| 2 | `(0, 2)`, `(1, 1)` | 2 |
| 3 | `(2, 1)` | 1 |
| 4 | `(2, 2)` | 0 |

### 12. TypeScript Solution
```typescript
function minutesUntilAllRotten(
  grid: readonly (readonly (0 | 1 | 2)[])[],
): number {
  const columnCount = grid[0]?.length ?? 0;
  for (const row of grid) {
    if (row.length !== columnCount) {
      throw new RangeError("grid must be rectangular");
    }
    for (const cell of row) {
      if (cell !== 0 && cell !== 1 && cell !== 2) {
        throw new RangeError("every cell must be 0, 1, or 2");
      }
    }
  }

  interface QueueEntry {
    row: number;
    column: number;
    minute: number;
  }

  const directions: readonly (readonly [number, number])[] = [
    [-1, 0],
    [0, 1],
    [1, 0],
    [0, -1],
  ];
  const rotten: boolean[][] = Array.from(
    { length: grid.length },
    () => new Array<boolean>(columnCount).fill(false),
  );
  const queue: QueueEntry[] = [];
  let fresh = 0;

  for (let row = 0; row < grid.length; row += 1) {
    for (let column = 0; column < columnCount; column += 1) {
      if (grid[row]![column] === 2) {
        rotten[row]![column] = true;
        queue.push({ row, column, minute: 0 });
      } else if (grid[row]![column] === 1) {
        fresh += 1;
      }
    }
  }

  let head = 0;
  let lastMinute = 0;
  while (head < queue.length) {
    const current = queue[head]!;
    head += 1;

    for (const [rowDelta, columnDelta] of directions) {
      const nextRow = current.row + rowDelta;
      const nextColumn = current.column + columnDelta;
      if (
        nextRow >= 0 &&
        nextRow < grid.length &&
        nextColumn >= 0 &&
        nextColumn < columnCount &&
        grid[nextRow]![nextColumn] === 1 &&
        !rotten[nextRow]![nextColumn]!
      ) {
        const nextMinute = current.minute + 1;
        rotten[nextRow]![nextColumn] = true;
        fresh -= 1;
        lastMinute = nextMinute;
        queue.push({ row: nextRow, column: nextColumn, minute: nextMinute });
      }
    }
  }

  return fresh === 0 ? lastMinute : -1;
}
```

### 13. Code Walkthrough
Shape and domain validation finish before queue or result state is allocated. The initialization scan adds every source at minute zero and counts fresh cells. `rotten` marks both original and scheduled rotten cells, while each queue entry carries the exact shortest minute at which its cell is processed.

### 14. Complexity Analysis
Validation, initialization, and BFS each take `O(RC)` time. Every cell is enqueued at most once and checks four directions. The boolean matrix and queue use `O(RC)` auxiliary space. The input grid is read only and no output grid is required.

### 15. Edge Cases
- `[]` and `[[], []]` return `0` after complete validation.
- A jagged grid throws, including a later nonempty row after an empty first row.
- Runtime values outside exactly `0`, `1`, and `2` throw.
- A grid with no fresh oranges returns `0`.
- Fresh oranges with no initial rotten source return `-1`.
- Fresh oranges separated by empty cells may remain unreachable and return `-1`.
- Multiple initial rotten cells spread simultaneously.
- The readonly input and all rows remain unchanged.

### 16. Common Mistakes
- Starting separate BFS runs from each rotten orange instead of one shared queue.
- Mutating during a brute-force scan and allowing same-minute cascading.
- Returning elapsed queue layers even when fresh oranges remain.
- Validating only cells reached by the spread.
- Returning early for a zero-column first row before checking rectangularity.
- Marking on dequeue and scheduling a fresh cell multiple times.
- Calling `shift()` in the optimized queue.

### 17. Interview Follow-ups
1. Return each cell's rotting minute. 2. Support blocked and weighted cells. 3. Identify unreachable fresh coordinates. 4. Add diagonal spread. 5. Handle newly inserted rotten sources.

### 18. Variations
- Walls and gates. - Distance to nearest zero. - Fire escape. - Infection spread. - Multi-source shortest paths.

### 19. Pattern Connections
Multi-source BFS is ordinary BFS initialized with every source rather than one source. It computes minimum unweighted distance to the nearest source and models simultaneous waves correctly. The same mark-on-enqueue rule prevents duplicate work in graph traversal, island search, and flood fill.

### 20. NestJS Project Connection
For `POST /api/dsa/grids/rotting-oranges`, bound rows, columns, total cells, and request size before invoking the service. Retain complete rectangular/domain validation. A bounded trace can emit `{ row, column, minute }` only when an orange is scheduled to rot; cap events, expose truncation, and do not serialize per-minute grid snapshots.

### 21. Mini Challenge
Implement `rottingTimeline(grid): number[][]` with the same validation and no input mutation. Return `-1` for empty or unreachable fresh cells, `0` for initially rotten cells, and each reachable fresh cell's minimum rotting minute.

## Problem 194 — Clone Graph

### 1. Problem Statement
Given this node shape, implement `cloneGraph<T>(start: GraphNode<T> | null): GraphNode<T> | null`:

```text
GraphNode<T>
  value: T
  neighbors: GraphNode<T>[]
```

The component reachable from `start` is finite, and every neighbor array is valid. Cycles, self-loops, duplicate neighbor entries, and shared nodes are allowed. Return `null` when `start` is `null`.

Otherwise, return an entirely separate graph containing one fresh `GraphNode` object for every reachable original identity. Preserve the exact topology, neighbor order, and duplicate-neighbor multiplicity. Preserve each `value` reference rather than deep-cloning it, and do not mutate any original node or neighbor array.

### 2. Example
```text
original:
  A.neighbors = [B, B, A]
  B.neighbors = [A]

clone:
  A'.neighbors = [B', B', A']
  B'.neighbors = [A']

A' !== A
B' !== B
A'.neighbors[0] === A'.neighbors[1]
A'.neighbors[2] === A'
A'.value === A.value
```

### 3. Understand the Problem
This is not a tree copy. The same node may be reached through many paths, and a path may lead back to a node already being copied. The clone must preserve identity relationships: one original object maps to exactly one fresh clone object.

#### In Simple Terms
Make one new shell for each reachable node, then reproduce every neighbor reference using the matching new shells.

### 4. Real-World Analogy
Imagine duplicating a transit map. A station can appear on several routes, but the copy must still contain one copied station shared by those routes. Route order and repeated stops stay unchanged, while every copied station is physically separate from the original map.

### 5. Key Observation
A `Map<GraphNode<T>, GraphNode<T>>` records which fresh object belongs to each original identity. Registering a clone before following its neighbors ensures that cycles and self-loops reuse that clone instead of creating more objects or traversing forever.

### 6. Pattern
```text
Pattern: Graph Traversal with an Identity Map
Why this pattern applies: references, not values, distinguish graph vertices
How to recognize it: cyclic objects must be copied while preserving sharing
Similar problems: deep-copy random pointers, copy dependency graphs, object deserialization
```

### 7. Brute Force
Use a pedagogical two-pass BFS. The first pass visits the reachable component and stores every original node once. The second phase creates one empty clone for each collected original, then a final wiring pass appends the mapped clones in each original neighbor array's exact order.

This is already asymptotically optimal, but it separates discovery, allocation, and wiring into distinct passes and stores both the collected-node array and the identity map.

#### Pseudocode
```text
if start is null: return null
collect every reachable original with BFS and a seen set
for each original: create and map an empty clone
for each original in collected order:
  for each neighbor in stored order:
    append that neighbor's mapped clone
return the clone mapped from start
```

#### Brute-Force TypeScript
```typescript
interface GraphNode<T> {
  value: T;
  neighbors: GraphNode<T>[];
}

function cloneGraphWithTwoPasses<T>(
  start: GraphNode<T> | null,
): GraphNode<T> | null {
  if (start === null) return null;

  const originals: GraphNode<T>[] = [];
  const queue: GraphNode<T>[] = [start];
  const seen = new Set<GraphNode<T>>([start]);
  let head = 0;

  while (head < queue.length) {
    const current = queue[head]!;
    head += 1;
    originals.push(current);

    for (const neighbor of current.neighbors) {
      if (!seen.has(neighbor)) {
        seen.add(neighbor);
        queue.push(neighbor);
      }
    }
  }

  const clones = new Map<GraphNode<T>, GraphNode<T>>();
  for (const original of originals) {
    clones.set(original, { value: original.value, neighbors: [] });
  }

  for (const original of originals) {
    const clone = clones.get(original)!;
    for (const neighbor of original.neighbors) {
      clone.neighbors.push(clones.get(neighbor)!);
    }
  }

  return clones.get(start)!;
}
```

#### Visualization
```text
original identity map after allocation:

A  ->  A' { neighbors: [] }
B  ->  B' { neighbors: [] }

wiring A.neighbors = [B, B, A]:

A'.neighbors = [map(B), map(B), map(A)]
             = [B',     B',     A']
```

#### Dry Run
Start with `A`. The collection BFS records `A`, discovers `B`, and ignores later references to already-seen `A` and `B`. Allocation creates exactly `A'` and `B'`. Wiring reads `[B, B, A]` without deduplication, so `A'` receives `[B', B', A']`. Wiring `B` then adds `A'` to `B'`.

#### Brute-Force Complexity
Let `V` be the number of reachable node identities and `E` the total number of entries in their neighbor arrays, including duplicates and self-loops. Collection and wiring each inspect every reachable node and edge occurrence, so time is `O(V + E)`. The set, map, arrays, queue, and returned graph use `O(V + E)` total new space; auxiliary bookkeeping excluding the output is `O(V)`.

### 8. Why Brute Force Is Not Ideal
The two-pass version delays all wiring until discovery and allocation are complete. Its extra collected-node array and separate phases are useful for teaching, but clone allocation can happen safely at first discovery, allowing each original's outgoing references to be wired as that original leaves the queue.

### 9. Optimal Approach
Return early for `null`. Otherwise, create the start clone and register it in the identity map before traversal. Run iterative BFS with a head-index queue. For each neighbor, create and register its clone only if absent, enqueue that newly discovered original, then append the mapped clone to the current clone's neighbor array.

Appending once per original neighbor occurrence preserves order and multiplicity. Looking up every repeated reference preserves shared identity. Early registration handles cycles and self-loops. Iterative BFS avoids recursive call-stack overflow on a deep component.

### 10. Visualization
```text
discover A: map A -> A', queue [A]

process A neighbor B:
  discover B, map B -> B', queue [A, B]
  append B' to A'

process A neighbor B again:
  reuse map(B)
  append B' to A' again

process A neighbor A:
  reuse map(A)
  append A' to A'
```

### 11. Step-by-Step Dry Run
| Step | Original processed | Newly registered | Cloned neighbors after step |
|---:|---|---|---|
| 1 | `A` | `B -> B'` | `A' -> [B', B', A']` |
| 2 | `B` | none | `B' -> [A']` |

The map contains two entries even though four neighbor occurrences were copied. Node identity is unique; edge occurrences are not.

### 12. TypeScript Solution
```typescript
interface GraphNode<T> {
  value: T;
  neighbors: GraphNode<T>[];
}

function cloneGraph<T>(
  start: GraphNode<T> | null,
): GraphNode<T> | null {
  if (start === null) return null;

  const startClone: GraphNode<T> = {
    value: start.value,
    neighbors: [],
  };
  const clones = new Map<GraphNode<T>, GraphNode<T>>([
    [start, startClone],
  ]);
  const queue: GraphNode<T>[] = [start];
  let head = 0;

  while (head < queue.length) {
    const original = queue[head]!;
    head += 1;
    const clone = clones.get(original)!;

    for (const neighbor of original.neighbors) {
      let neighborClone = clones.get(neighbor);
      if (neighborClone === undefined) {
        neighborClone = { value: neighbor.value, neighbors: [] };
        clones.set(neighbor, neighborClone);
        queue.push(neighbor);
      }
      clone.neighbors.push(neighborClone);
    }
  }

  return startClone;
}
```

### 13. Code Walkthrough
`startClone` is fresh but keeps the exact `start.value` reference. `clones` is both the visited structure and the original-to-copy lookup. Each newly seen original is registered before it is enqueued. When its turn arrives, its clone already exists, so every outgoing occurrence can be appended immediately in stored order.

### 14. Complexity Analysis
Each reachable original enters the queue once, and each neighbor-array entry is read and appended once. Time is therefore `O(V + E)`. The map and queue use `O(V)` auxiliary space. The returned nodes and neighbor arrays use `O(V + E)` output space. This matches the two-pass asymptotic bounds with fewer phases and no separate collection array.

### 15. Edge Cases
- `null` returns `null`.
- One node with no neighbors produces one fresh isolated node.
- A self-loop points from the clone back to that same clone.
- Duplicate neighbor entries remain duplicated and point to the same clone identity.
- Several paths to one original share one cloned node rather than producing copies.
- Cycles terminate because nodes are registered before their neighbors are explored.
- Object-valued `value` fields preserve the original value reference.
- No original node or neighbor array is mutated.

### 16. Common Mistakes
- Keying clones by `value` instead of original object identity.
- Creating a new clone every time a neighbor reference appears.
- Registering a clone only after recursively processing its neighbors.
- Using a `Set` to wire neighbors and losing duplicates or order.
- Deep-cloning `value` even though the contract preserves its reference.
- Returning any original node inside the cloned component.
- Using recursive DFS for an arbitrarily deep reachable component.

### 17. Interview Follow-ups
1. Clone only nodes satisfying a predicate. 2. Clone edge metadata as well as nodes. 3. Return the original-to-clone map. 4. Clone several entry nodes while preserving cross-entry sharing. 5. Serialize and deserialize the component.

### 18. Variations
- Copy list with random pointer. - Deep-copy a directed object graph. - Clone an undirected graph. - Snapshot a workflow graph. - Copy a graph with immutable values.

### 19. Pattern Connections
The identity map combines visited-state and output construction. It is the object-reference counterpart of mapping dense vertex IDs to result slots. The same early-registration rule used here also prevents infinite recursion when copying cyclic linked structures and preserves shared subgraphs that a tree-style copy would duplicate.

### 20. NestJS Project Connection
For `POST /api/dsa/graphs/clone`, JSON cannot directly carry object identity or cycles, so accept a bounded node table with explicit IDs and ordered neighbor-ID arrays. Validate IDs and limits, build the in-memory component, clone it in the service, then serialize with a new ID table. Keep duplicate neighbor IDs and order observable, cap nodes and edge occurrences, and never place request-scoped identity maps in singleton service state.

### 21. Mini Challenge
Implement `cloneGraphWithMap(start)` so it returns both the cloned start and a readonly original-to-clone map. Preserve the same identity, topology, order, multiplicity, value-reference, and no-mutation contracts.

## Problem 195 — Topological Sort

### 1. Problem Statement
Implement `topologicalSort(graph: readonly (readonly number[])[]): number[] | null` for a directed adjacency list whose vertices are dense IDs from `0` through `graph.length - 1`.

Validate every neighbor ID before allocating algorithm state or producing output. Each ID must be a safe integer in range; invalid input throws `RangeError`. Parallel directed edges are allowed, and each occurrence contributes separately to indegree. A self-loop is a cycle. Do not mutate the graph or its neighbor arrays.

Return `[]` for an empty graph. Return `null` when no topological order exists. The optimized Kahn implementation has an exact deterministic order: scan initial zero-indegree IDs in ascending order, process a head-index queue, inspect outgoing neighbors in stored order, and append a vertex when its indegree first becomes zero.

### 2. Example
```text
graph = [
  [1, 2],
  [3],
  [3],
  [],
]

optimized result = [0, 1, 2, 3]
```

Both `1` and `2` depend on `0`. Vertex `3` becomes available only after both incoming edge occurrences have been removed.

### 3. Understand the Problem
A topological order places every source before each destination of its outgoing edges. More than one valid order may exist. The exact tie-breaking above applies to the optimized Kahn function. The brute-force DFS function may return a different valid order because reverse postorder uses a different traversal rule.

#### In Simple Terms
Arrange all vertices so every prerequisite-like vertex appears before the vertices that point away from it, or report that a cycle makes this impossible.

### 4. Real-World Analogy
Think of tasks connected by one-way dependency arrows. A task can be placed on the schedule only after every incoming dependency is satisfied. If a group of tasks waits in a circle, none of them can be scheduled first.

### 5. Key Observation
In a directed acyclic graph, at least one remaining vertex has indegree zero. Removing such a vertex and its outgoing edge occurrences exposes the next available vertices. If vertices remain but none has indegree zero, those remaining vertices contain a directed cycle.

### 6. Pattern
```text
Pattern: Topological Ordering with Kahn's Algorithm
Why this pattern applies: dependencies are directed and every predecessor must come first
How to recognize it: emit currently dependency-free vertices and remove outgoing edges
Similar problems: course planning, build systems, workflow scheduling, dependency resolution
```

### 7. Brute Force
After validating the complete adjacency list, run recursive DFS with three colors. Mark a vertex active before exploring its outgoing neighbors. An edge to an active vertex proves a cycle. After all descendants finish, mark the vertex complete and append it to a postorder array. Reverse postorder to obtain a valid topological order.

The DFS output is deterministic for its own ascending-start and stored-neighbor traversal, but it is not required to match Kahn's specified order. For the example, DFS returns `[0, 2, 1, 3]`, which is also valid.

#### Pseudocode
```text
validate every neighbor ID
colors = unvisited for every vertex
for each vertex in ascending order:
  if unvisited, recursively visit it
  if an edge reaches an active vertex: return null
  append each vertex after all outgoing neighbors finish
reverse postorder and return it
```

#### Brute-Force TypeScript
```typescript
function topologicalSortWithDfs(
  graph: readonly (readonly number[])[],
): number[] | null {
  const vertexCount = graph.length;
  for (const neighbors of graph) {
    for (const neighbor of neighbors) {
      if (
        !Number.isSafeInteger(neighbor) ||
        neighbor < 0 ||
        neighbor >= vertexCount
      ) {
        throw new RangeError("neighbor must be a valid vertex ID");
      }
    }
  }

  const colors = new Uint8Array(vertexCount);
  const postorder: number[] = [];

  const visit = (vertex: number): boolean => {
    colors[vertex] = 1;

    for (const neighbor of graph[vertex]!) {
      const neighborColor = colors[neighbor]!;
      if (neighborColor === 1) return false;
      if (neighborColor === 0 && !visit(neighbor)) return false;
    }

    colors[vertex] = 2;
    postorder.push(vertex);
    return true;
  };

  for (let vertex = 0; vertex < vertexCount; vertex += 1) {
    if (colors[vertex] === 0 && !visit(vertex)) return null;
  }

  postorder.reverse();
  return postorder;
}
```

#### Visualization
```text
DFS from 0:

0 -> 1 -> 3
|         finish 3, then 1
v
2         finish 2, then 0

postorder:         [3, 1, 2, 0]
reversed result:   [0, 2, 1, 3]
```

#### Dry Run
Vertex `0` becomes active and visits `1`, which visits and completes `3`. DFS then completes `1`, visits and completes `2`, and finally completes `0`. Reversing `[3, 1, 2, 0]` gives `[0, 2, 1, 3]`. If any traversal edge had reached a color-`1` vertex, the function would have returned `null`.

#### Brute-Force Complexity
Validation and DFS each inspect all `V` vertices and `E` directed edge occurrences, so time is `O(V + E)`. Colors, postorder, and recursion use `O(V)` auxiliary space. A path of length `V` can create `O(V)` call depth and overflow the JavaScript stack even though the graph is valid.

### 8. Why Brute Force Is Not Ideal
DFS computes a valid order efficiently, but recursive call depth is controlled by graph shape rather than a safe application limit. It also does not produce the required Kahn tie-breaking. An iterative indegree process gives the specified output and keeps traversal state in heap-allocated arrays.

### 9. Optimal Approach
First validate every neighbor ID without allocating indegrees, a queue, or output. Then allocate an indegree array and count every adjacency occurrence, including parallel edges. Scan vertex IDs from `0` upward to seed the queue with all zero-indegree vertices.

Read the queue with a head index. Emit each dequeued vertex, inspect its outgoing neighbors in stored order, and decrement once per occurrence. Append a neighbor exactly when its indegree reaches zero. At the end, return the order only if it contains all `V` vertices; otherwise return `null` because a cycle prevented the remainder from becoming available.

### 10. Visualization
```text
graph: 0 -> 1, 0 -> 2, 1 -> 3, 2 -> 3

initial indegrees: [0, 1, 1, 2]
initial queue:     [0]

emit 0: indegrees [0, 0, 0, 2], queue [0, 1, 2]
emit 1: indegrees [0, 0, 0, 1]
emit 2: indegrees [0, 0, 0, 0], queue [..., 3]
emit 3: result     [0, 1, 2, 3]
```

### 11. Step-by-Step Dry Run
| Head vertex | Outgoing updates | Newly appended | Output |
|---:|---|---|---|
| `0` | `indegree[1]: 1 -> 0`, `indegree[2]: 1 -> 0` | `1`, then `2` | `[0]` |
| `1` | `indegree[3]: 2 -> 1` | none | `[0, 1]` |
| `2` | `indegree[3]: 1 -> 0` | `3` | `[0, 1, 2]` |
| `3` | none | none | `[0, 1, 2, 3]` |

### 12. TypeScript Solution
```typescript
function topologicalSort(
  graph: readonly (readonly number[])[],
): number[] | null {
  const vertexCount = graph.length;
  for (const neighbors of graph) {
    for (const neighbor of neighbors) {
      if (
        !Number.isSafeInteger(neighbor) ||
        neighbor < 0 ||
        neighbor >= vertexCount
      ) {
        throw new RangeError("neighbor must be a valid vertex ID");
      }
    }
  }

  const indegree = new Array<number>(vertexCount).fill(0);
  for (const neighbors of graph) {
    for (const neighbor of neighbors) {
      indegree[neighbor] = indegree[neighbor]! + 1;
    }
  }

  const queue: number[] = [];
  for (let vertex = 0; vertex < vertexCount; vertex += 1) {
    if (indegree[vertex] === 0) queue.push(vertex);
  }

  const order: number[] = [];
  let head = 0;
  while (head < queue.length) {
    const vertex = queue[head]!;
    head += 1;
    order.push(vertex);

    for (const neighbor of graph[vertex]!) {
      const nextIndegree = indegree[neighbor]! - 1;
      indegree[neighbor] = nextIndegree;
      if (nextIndegree === 0) queue.push(neighbor);
    }
  }

  return order.length === vertexCount ? order : null;
}
```

### 13. Code Walkthrough
The first nested loop is the complete atomic validation phase. Only after it succeeds does the function allocate state. The next pass counts edge occurrences. The ascending initialization scan establishes the first queue order, while stored outgoing order and immediate zero-indegree appends establish every later tie. `order.length` detects whether a cycle withheld any vertex.

### 14. Complexity Analysis
Validation reads `E` edge occurrences. Indegree construction reads them again, and Kahn's loop reads each once more while emitting every available vertex once. These constant passes total `O(V + E)` time. Indegrees, queue, and output each use `O(V)` space. Parallel edges do not change the bound because `E` counts every occurrence.

### 15. Edge Cases
- `[]` returns `[]`.
- An edgeless graph returns all IDs in ascending order.
- A self-loop returns `null`.
- Any directed cycle leaves fewer than `V` emitted vertices and returns `null`.
- Disconnected acyclic components share the deterministic queue according to availability.
- Parallel edges are counted and decremented once per occurrence.
- Unsafe, negative, fractional, or out-of-range IDs throw before algorithm state is allocated.
- The graph and all outgoing arrays remain unchanged.

### 16. Common Mistakes
- Treating the directed graph as undirected.
- Deduplicating parallel edges while constructing only one side of the indegree logic.
- Enqueueing a vertex when indegree is less than or equal to zero instead of exactly zero.
- Seeding the initial queue from adjacency encounter order rather than ascending IDs.
- Sorting newly available vertices and changing the specified queue behavior.
- Returning a partial order when a cycle exists.
- Validating only vertices reachable from the first zero-indegree vertex.
- Calling `shift()` repeatedly instead of using a head index.

### 17. Interview Follow-ups
1. Return one directed cycle when sorting fails. 2. Produce the lexicographically smallest topological order with a min heap. 3. Count topological orders. 4. Process levels of concurrently available vertices. 5. Update an order after adding an edge.

### 18. Variations
- Alien dictionary. - Build-order resolution. - Eventual safe states. - Parallel job scheduling. - DAG dynamic programming.

### 19. Pattern Connections
Kahn's algorithm is directed cycle detection and ordering in one process. Emitting all vertices proves acyclicity; getting stuck proves a cycle remains. DFS reaches the same feasibility conclusion through active-path colors, but its reverse-postorder tie-breaking and recursion behavior differ.

### 20. NestJS Project Connection
For `POST /api/dsa/graphs/topological-sort`, cap vertices, edge occurrences, row lengths, and payload bytes before service execution. Preserve outgoing order when mapping DTOs, validate every ID before allocating proportional algorithm state, and return an explicit cycle status rather than confusing `null` with transport failure. A bounded trace can record queue appends, emissions, and indegree changes with truncation metadata.

### 21. Mini Challenge
Implement `topologicalLayers(graph): number[][] | null` with the same validation. Return each simultaneous zero-indegree wave as one ascending-discovery layer, preserve stored outgoing order within processing, and return `null` on a cycle.

## Problem 196 — Course Schedule

### 1. Problem Statement
Implement `canFinishCourses(courseCount: number, prerequisites: readonly (readonly [number, number])[]): boolean`. Each exact pair `[course, prerequisite]` means the directed dependency `prerequisite -> course`.

`courseCount` must be a nonnegative safe integer no greater than `2^32 - 1`. Every prerequisite entry must contain exactly two values, and both endpoints must be safe integers in `[0, courseCount)`. Validate the count and every complete pair before allocating adjacency or indegree arrays; invalid input throws `RangeError` atomically.

Retain parallel duplicate prerequisite occurrences. They do not change feasibility, but each occurrence is represented and processed. A self-dependency makes the schedule impossible. Zero courses with no pairs returns `true`; any pair with zero courses is invalid. Do not mutate the pair list or any pair.

### 2. Example
```text
courseCount = 4
prerequisites = [
  [1, 0],
  [2, 0],
  [3, 1],
  [3, 2],
]

edges:
0 -> 1 -> 3
 \-> 2 ->/

result = true
```

Course `0` unlocks courses `1` and `2`; both must be processed before course `3` becomes available.

### 3. Understand the Problem
Courses are vertices and prerequisite relationships are directed edges. Finishing every course is possible exactly when this dependency graph has no directed cycle. The pair order is easy to reverse accidentally: `[course, prerequisite]` creates an arrow from the second value to the first.

#### In Simple Terms
Return whether the prerequisite arrows let every course eventually become available, rejecting any circular dependency.

### 4. Real-World Analogy
Imagine training modules that unlock later modules. Repeating the same unlock rule does not make the plan impossible, but a module requiring itself, or a chain that circles back to its start, means no module in that circle can be completed first.

### 5. Key Observation
Course feasibility is topological-sort feasibility. DFS proves feasibility by finding no edge back to an active prerequisite path. Kahn's algorithm proves it by repeatedly processing courses with zero remaining prerequisites and checking whether the processed count reaches `courseCount`.

### 6. Pattern
```text
Pattern: Directed Cycle Detection / Topological Feasibility
Why this pattern applies: prerequisites impose one-way ordering constraints
How to recognize it: determine whether every dependency can be satisfied
Similar problems: build completion, curriculum planning, package dependencies, workflows
```

### 7. Brute Force
Validate `courseCount` and all exact pairs first. Then build `prerequisitesByCourse`, storing the second endpoint under the first endpoint for every occurrence. Run recursive three-color DFS from each course. While exploring one course's prerequisite chain, reaching an active course proves a cycle. Completed prerequisite chains can be reused safely.

This orientation follows “what must this course take first?” and is convenient for recursive reasoning. Reversing every edge would preserve whether a cycle exists, but the optimized version uses the contract's `prerequisite -> course` orientation so indegrees directly count unmet prerequisites.

#### Pseudocode
```text
validate courseCount and every exact endpoint pair
build prerequisitesByCourse, retaining duplicate occurrences
for each course:
  recursively explore its prerequisites with three colors
  if an edge reaches an active course: return false
return true
```

#### Brute-Force TypeScript
```typescript
function canFinishCoursesWithDfs(
  courseCount: number,
  prerequisites: readonly (readonly [number, number])[],
): boolean {
  const maximumCourseCount = 2 ** 32 - 1;
  if (
    !Number.isSafeInteger(courseCount) ||
    courseCount < 0 ||
    courseCount > maximumCourseCount
  ) {
    throw new RangeError("courseCount is outside the supported range");
  }

  for (const pair of prerequisites) {
    if (pair.length !== 2) {
      throw new RangeError("each prerequisite must contain exactly two IDs");
    }
    const course = pair[0];
    const prerequisite = pair[1];
    if (
      !Number.isSafeInteger(course) ||
      !Number.isSafeInteger(prerequisite) ||
      course < 0 ||
      course >= courseCount ||
      prerequisite < 0 ||
      prerequisite >= courseCount
    ) {
      throw new RangeError("prerequisite endpoint is not a valid course ID");
    }
  }

  const prerequisitesByCourse: number[][] = Array.from(
    { length: courseCount },
    () => [],
  );
  for (const [course, prerequisite] of prerequisites) {
    prerequisitesByCourse[course]!.push(prerequisite);
  }

  const colors = new Uint8Array(courseCount);
  const visit = (course: number): boolean => {
    colors[course] = 1;

    for (const prerequisite of prerequisitesByCourse[course]!) {
      const prerequisiteColor = colors[prerequisite]!;
      if (prerequisiteColor === 1) return false;
      if (prerequisiteColor === 0 && !visit(prerequisite)) return false;
    }

    colors[course] = 2;
    return true;
  };

  for (let course = 0; course < courseCount; course += 1) {
    if (colors[course] === 0 && !visit(course)) return false;
  }
  return true;
}
```

#### Visualization
```text
pair [3, 1] means course 3 requires course 1

prerequisitesByCourse[3] = [1, 2]

DFS request for 3:
3 -> prerequisite 1 -> prerequisite 0 -> complete
  -> prerequisite 2 -> prerequisite 0 already complete
complete 3
```

#### Dry Run
For the example, DFS from course `0` completes immediately. Course `1` sees completed prerequisite `0`, then completes. Course `2` does the same. Course `3` sees completed prerequisites `1` and `2`, then completes. No edge reaches an active course, so the function returns `true`.

#### Brute-Force Complexity
Validation, adjacency construction, and DFS each take `O(V + E)` total time, where `V = courseCount` and `E = prerequisites.length`, including duplicates. Adjacency and colors use `O(V + E)` space, and recursion can use `O(V)` call depth. A long valid prerequisite chain may overflow the JavaScript stack.

### 8. Why Brute Force Is Not Ideal
Three-color DFS has optimal asymptotic cost, but its recursive depth can fail on a large linear curriculum. It also answers only through path-state reasoning. Iterative Kahn processing directly models courses becoming available and avoids recursion depth as an additional failure mode.

### 9. Optimal Approach
Complete all validation before allocating graph state. Build outgoing adjacency in the contract direction: for `[course, prerequisite]`, append `course` to `outgoing[prerequisite]` and increment `indegree[course]`. Keep duplicate pairs as duplicate edge occurrences.

Seed a queue with every zero-indegree course in ascending ID order. Process it with a head index. Each outgoing occurrence decrements the destination indegree once, and a course is appended when that count reaches zero. Return whether the number processed equals `courseCount`. Duplicate edges balance because they are counted and removed equally; self-dependencies and longer cycles prevent full processing.

### 10. Visualization
```text
pairs: [1, 0], [2, 0], [3, 1], [3, 2]

outgoing:
0: [1, 2]
1: [3]
2: [3]
3: []

indegree: [0, 1, 1, 2]
queue:    [0] -> [1, 2] -> [3]
processed: 4 of 4, so true
```

### 11. Step-by-Step Dry Run
| Processed course | Remaining-prerequisite changes | Queue additions | Processed count |
|---:|---|---|---:|
| `0` | course `1: 1 -> 0`, course `2: 1 -> 0` | `1`, `2` | 1 |
| `1` | course `3: 2 -> 1` | none | 2 |
| `2` | course `3: 1 -> 0` | `3` | 3 |
| `3` | none | none | 4 |

Since the final count equals `courseCount`, every course can finish.

### 12. TypeScript Solution
```typescript
function canFinishCourses(
  courseCount: number,
  prerequisites: readonly (readonly [number, number])[],
): boolean {
  const maximumCourseCount = 2 ** 32 - 1;
  if (
    !Number.isSafeInteger(courseCount) ||
    courseCount < 0 ||
    courseCount > maximumCourseCount
  ) {
    throw new RangeError("courseCount is outside the supported range");
  }

  for (const pair of prerequisites) {
    if (pair.length !== 2) {
      throw new RangeError("each prerequisite must contain exactly two IDs");
    }
    const course = pair[0];
    const prerequisite = pair[1];
    if (
      !Number.isSafeInteger(course) ||
      !Number.isSafeInteger(prerequisite) ||
      course < 0 ||
      course >= courseCount ||
      prerequisite < 0 ||
      prerequisite >= courseCount
    ) {
      throw new RangeError("prerequisite endpoint is not a valid course ID");
    }
  }

  const outgoing: number[][] = Array.from(
    { length: courseCount },
    () => [],
  );
  const indegree = new Array<number>(courseCount).fill(0);

  for (const [course, prerequisite] of prerequisites) {
    outgoing[prerequisite]!.push(course);
    indegree[course] = indegree[course]! + 1;
  }

  const queue: number[] = [];
  for (let course = 0; course < courseCount; course += 1) {
    if (indegree[course] === 0) queue.push(course);
  }

  let head = 0;
  let processed = 0;
  while (head < queue.length) {
    const prerequisite = queue[head]!;
    head += 1;
    processed += 1;

    for (const course of outgoing[prerequisite]!) {
      const nextIndegree = indegree[course]! - 1;
      indegree[course] = nextIndegree;
      if (nextIndegree === 0) queue.push(course);
    }
  }

  return processed === courseCount;
}
```

### 13. Code Walkthrough
The scalar count check and complete pair-validation loop happen before `outgoing` or `indegree` exists. Construction uses `prerequisite -> course`, matching the contract. Indegree represents how many prerequisite occurrences remain. The queue contains available courses, and `processed` replaces the need to materialize a full topological order.

### 14. Complexity Analysis
Validation and graph construction each read all `E` pairs. Queue initialization reads `V` courses. Kahn processing dequeues each course at most once and reads each stored edge occurrence once. Total time is `O(V + E)`. Outgoing adjacency, indegrees, and the queue use `O(V + E)` auxiliary space. The iterative approach uses no graph-dependent call stack.

### 15. Edge Cases
- `courseCount = 0` with no pairs returns `true`.
- Any pair when `courseCount = 0` throws because no endpoint is in range.
- A self-dependency such as `[0, 0]` returns `false`.
- A directed cycle of any length returns `false`.
- Disconnected acyclic course groups return `true`.
- Duplicate pairs are retained, counted, decremented, and do not change feasibility.
- Reversing pair interpretation can produce the same cycle answer but breaks the documented dependency model and traces.
- Invalid count, malformed pair length, or invalid endpoint throws before adjacency allocation.
- Input pairs and the outer prerequisite array remain unchanged.

### 16. Common Mistakes
- Building `course -> prerequisite` while claiming indegree counts unmet prerequisites.
- Validating endpoints only while constructing adjacency, after allocation has begun.
- Accepting fractional, unsafe, negative, or out-of-range IDs.
- Deduplicating only adjacency or only indegree and making counts inconsistent.
- Treating duplicate prerequisite occurrences as separate courses.
- Returning `true` merely because the initial queue is nonempty.
- Using recursive DFS in the optimized implementation.
- Calling `shift()` for every available course.

### 17. Interview Follow-ups
1. Return one valid course order. 2. Return a cycle blocking completion. 3. Compute semester layers. 4. Limit courses per semester. 5. Support adding prerequisite pairs incrementally.

### 18. Variations
- Course Schedule II. - Build-order feasibility. - Package installation. - Curriculum semesters. - Dependency cycle reporting.

### 19. Pattern Connections
This problem asks only whether topological sorting can emit every vertex. Kahn's processed count is therefore enough; storing the order is optional. DFS colors prove the same property from active paths. Retaining parallel edges demonstrates the general rule that indegree construction and decrement logic must represent the same edge multiset.

### 20. NestJS Project Connection
For `POST /api/dsa/courses/can-finish`, enforce practical limits far below the language maximum for course count, pair count, and payload bytes before calling the service. DTO validation should reject non-integers and malformed runtime arrays, while the domain function retains complete atomic validation. Keep pair orientation explicit in API examples, and cap any trace of queue events or indegree changes with truncation metadata.

### 21. Mini Challenge
Implement `courseOrder(courseCount, prerequisites): number[] | null` with the same validation and `prerequisite -> course` orientation. Use ascending initial zero-indegree IDs, stored pair order for outgoing edges, a head-index queue, and return `null` unless all courses are emitted.

## Problem 197 — Shortest Path in Unweighted Graph

### 1. Problem Statement
Implement `shortestPathDistances(adjacency: readonly (readonly number[])[], start: number): number[]` for an unweighted adjacency list whose vertices are dense IDs from `0` through `adjacency.length - 1`. Each inner array lists out-neighbors, so the input may describe a directed graph or an undirected graph supplied as symmetric occurrences.

Validate `start` first: it must be a safe integer in `[0, adjacency.length)`, otherwise throw `RangeError`. Then validate every neighbor ID as a safe integer in range before allocating any traversal state; any violation throws `RangeError`. Parallel edges and self-loops are allowed and harmless. Do not mutate the adjacency list or any neighbor array.

Return a fresh array where index `v` holds the minimum number of edges from `start` to `v`, `-1` marks an unreachable vertex, and `distances[start]` is `0`.

### 2. Example
```text
adjacency = [
  [1, 2],
  [3],
  [3],
  [],
  [0],
]
start = 0

result = [0, 1, 1, 2, -1]
```

Vertices `1` and `2` are one edge away, vertex `3` needs two edges through either middle vertex, and vertex `4` only points into the component, so it is unreachable.

### 3. Understand the Problem
Every edge costs exactly one step, so the shortest path to a vertex is the smallest count of edges along any route from `start`. The answer covers every vertex, including those in other components, which report `-1`. Self-loops never shorten a path, and parallel edges only repeat a discovery that has already happened.

#### In Simple Terms
Starting from one vertex, count the fewest hops needed to reach every other vertex, and report `-1` for vertices no sequence of hops can reach.

### 4. Real-World Analogy
Think of a rumor spreading through coworkers. On day one the starter knows it, on day two everyone the starter talks to knows it, and each later day adds the people one conversation further out. A person's "day number" is their hop distance, and someone with no conversational path never hears the rumor.

### 5. Key Observation
BFS visits vertices in nondecreasing edge distance from the start. The first time a vertex is discovered, it is discovered through a shortest route, so assigning `distances[neighbor] = distances[current] + 1` exactly once at first discovery is final and never needs revision.

### 6. Pattern
```text
Pattern: Unweighted Shortest Path with BFS
Why this pattern applies: every edge contributes exactly one unit of distance
How to recognize it: fewest edges, minimum moves, or minimum steps in an unweighted graph
Similar problems: word ladder, minimum knight moves, maze escape steps, degrees of separation
```

### 7. Brute Force
After validating `start` and every neighbor ID, run iterative relaxation with unit weights, the Bellman-Ford idea specialized to this problem. Set `distances[start] = 0` and all other entries to `-1`. Then repeat up to `V - 1` passes: in each pass, scan every vertex with a known distance and try to improve each out-neighbor to `distances[vertex] + 1`. Stop early when a full pass changes nothing.

This converges because any shortest path uses at most `V - 1` edges, so after pass `k` every vertex whose shortest path has `k` edges holds its final distance.

#### Pseudocode
```text
validate start, then every neighbor ID
distances = [-1] * V, distances[start] = 0
repeat up to V - 1 times:
  changed = false
  for each vertex with distances[vertex] != -1:
    candidate = distances[vertex] + 1
    for each out-neighbor:
      if unreachable so far or candidate is smaller:
        update neighbor, changed = true
  if not changed: stop
return distances
```

#### Brute-Force TypeScript
```typescript
function shortestPathDistancesWithRelaxation(
  adjacency: readonly (readonly number[])[],
  start: number,
): number[] {
  const vertexCount = adjacency.length;
  if (!Number.isSafeInteger(start) || start < 0 || start >= vertexCount) {
    throw new RangeError("start must be a valid vertex ID");
  }
  for (const neighbors of adjacency) {
    for (const neighbor of neighbors) {
      if (
        !Number.isSafeInteger(neighbor) ||
        neighbor < 0 ||
        neighbor >= vertexCount
      ) {
        throw new RangeError("neighbor must be a valid vertex ID");
      }
    }
  }

  const distances = new Array<number>(vertexCount).fill(-1);
  distances[start] = 0;

  for (let pass = 1; pass < vertexCount; pass += 1) {
    let changed = false;

    for (let vertex = 0; vertex < vertexCount; vertex += 1) {
      const sourceDistance = distances[vertex]!;
      if (sourceDistance === -1) continue;
      const candidate = sourceDistance + 1;

      for (const neighbor of adjacency[vertex]!) {
        const best = distances[neighbor]!;
        if (best === -1 || candidate < best) {
          distances[neighbor] = candidate;
          changed = true;
        }
      }
    }

    if (!changed) break;
  }

  return distances;
}
```

#### Visualization
```text
distances after each relaxation pass, start = 0:

initial: [0, -1, -1, -1, -1]
pass 1:  [0,  1,  1,  2, -1]   (vertex 1 improves 3 within the same pass)
pass 2:  no change, stop

vertex 4 never gains a distance: nothing reaches it
```

#### Dry Run
Pass 1 scans vertex `0` and sets `distances[1] = 1` and `distances[2] = 1`. Later in the same pass, vertex `1` already holds `1`, so `distances[3]` becomes `2`; vertex `2` offers `2` again and changes nothing. Vertex `4` still holds `-1` and is skipped as a source. Pass 2 finds no improvement and stops, returning `[0, 1, 1, 2, -1]`.

#### Brute-Force Complexity
Each pass scans all `V` vertices and all `E` adjacency occurrences, and up to `V - 1` passes may be needed when improvements arrive one edge per pass, so worst-case time is `O(V * E)`. The distances array is required output, and auxiliary space is `O(1)` beyond it, or `O(V)` counting the output. The result is correct but repeats work that a level-order traversal performs once.

### 8. Why Brute Force Is Not Ideal
Relaxation re-reads every edge in every pass because it does not know which distances are already final. With unit weights, BFS makes that knowledge structural: vertices leave the queue in nondecreasing distance order, so each first discovery is final and each edge needs exactly one inspection. BFS therefore supersedes repeated relaxation whenever all edges cost the same.

### 9. Optimal Approach
Validate `start`, then every neighbor ID, before allocating anything. Fill a fresh distances array with `-1`, set `distances[start] = 0`, and seed an array queue with `start`.

Read the queue with a head index instead of `shift()`. For each dequeued vertex, compute `distances[vertex] + 1` once, then inspect stored neighbors in order. A neighbor still holding `-1` is discovered now: assign its distance and enqueue it immediately. Assigning at enqueue time prevents duplicate queue entries, and self-loops or parallel edges are skipped naturally because their targets already hold a distance.

### 10. Visualization
```text
adjacency: 0 -> 1, 0 -> 2, 1 -> 3, 2 -> 3, 4 -> 0

queue [0]          head=0   distances [0, -1, -1, -1, -1]
read 0: discover 1, 2       distances [0, 1, 1, -1, -1]
read 1: discover 3          distances [0, 1, 1, 2, -1]
read 2: 3 already seen
read 3: no neighbors

result: [0, 1, 1, 2, -1]
```

### 11. Step-by-Step Dry Run
| Head vertex | Neighbor checks | Newly discovered | Queue |
|---:|---|---|---|
| `0` | `1: -1 -> 1`, `2: -1 -> 1` | `1`, `2` | `[0, 1, 2]` |
| `1` | `3: -1 -> 2` | `3` | `[0, 1, 2, 3]` |
| `2` | `3` already `2` | none | `[0, 1, 2, 3]` |
| `3` | none | none | `[0, 1, 2, 3]` |

Vertex `4` never enters the queue and keeps `-1`.

### 12. TypeScript Solution
```typescript
function shortestPathDistances(
  adjacency: readonly (readonly number[])[],
  start: number,
): number[] {
  const vertexCount = adjacency.length;
  if (!Number.isSafeInteger(start) || start < 0 || start >= vertexCount) {
    throw new RangeError("start must be a valid vertex ID");
  }
  for (const neighbors of adjacency) {
    for (const neighbor of neighbors) {
      if (
        !Number.isSafeInteger(neighbor) ||
        neighbor < 0 ||
        neighbor >= vertexCount
      ) {
        throw new RangeError("neighbor must be a valid vertex ID");
      }
    }
  }

  const distances = new Array<number>(vertexCount).fill(-1);
  distances[start] = 0;

  const queue: number[] = [start];
  let head = 0;
  while (head < queue.length) {
    const vertex = queue[head]!;
    head += 1;
    const nextDistance = distances[vertex]! + 1;

    for (const neighbor of adjacency[vertex]!) {
      if (distances[neighbor] === -1) {
        distances[neighbor] = nextDistance;
        queue.push(neighbor);
      }
    }
  }

  return distances;
}
```

### 13. Code Walkthrough
The start check runs before the neighbor sweep, and both run before the distances array exists, giving invalid input an atomic failure. `-1` doubles as the unreachable marker and the not-yet-discovered test, which is safe because `distances[start]` is `0` before the loop begins. The head index advances through the queue without reindexing, and each vertex computes its outgoing distance once. Distances stay bounded by `V - 1`, so no checked addition is needed for unit steps.

### 14. Complexity Analysis
Validation reads all `E` adjacency occurrences. BFS enqueues each reachable vertex once and inspects each outgoing occurrence once, so total time is `O(V + E)`. The distances array is required output; the queue holds at most `V` entries, so auxiliary space is `O(V)`. Parallel edges and self-loops only add constant-time rejected checks.

### 15. Edge Cases
- A single-vertex graph with `start = 0` returns `[0]`.
- `[]` throws because no `start` is in range.
- Unreachable vertices, including those with edges pointing into the reachable component, return `-1`.
- A self-loop at any vertex changes nothing: its target is already discovered.
- Parallel edges trigger extra constant-time checks but never a second discovery.
- An out-of-range, negative, fractional, or unsafe `start` throws before neighbor validation.
- Any invalid neighbor ID anywhere throws before traversal state is allocated.
- The adjacency list and every neighbor array remain unchanged, and the result is a fresh array.

### 16. Common Mistakes
- Validating only vertices reachable from `start` instead of the whole structure.
- Assigning distances at dequeue time and enqueueing the same vertex repeatedly.
- Calling `shift()` on the queue and degrading each dequeue to linear time.
- Using DFS and reporting a path that is valid but not shortest.
- Reusing `0` as the unreachable marker and colliding with `distances[start]`.
- Treating a self-loop or parallel edge as an error rather than a harmless occurrence.
- Returning the input-sized array without setting the start distance in the empty-neighbors case.
- Reaching for Dijkstra when unit weights make BFS strictly simpler and faster.

### 17. Interview Follow-ups
1. Return one actual shortest path, not just distances. 2. Support multiple sources with a multi-seed queue. 3. Compute shortest paths on an implicit grid graph. 4. Count the number of distinct shortest paths per vertex. 5. Handle 0/1 edge weights with a deque.

### 18. Variations
- Word ladder transformations. - Minimum knight moves. - Shortest bridge between islands. - Nearest exit in a maze. - Open-the-lock state search.

### 19. Pattern Connections
This problem upgrades Problem 187's BFS from an order to a distance labeling: the queue's nondecreasing-distance property is what both rely on. It is also the unit-weight special case of Problem 198, where varying weights force a priority queue and stale-entry handling that plain BFS never needs.

### 20. NestJS Project Connection
For `POST /api/dsa/graphs/shortest-path-distances`, cap vertex count, adjacency occurrences, row lengths, and payload bytes before service execution. Validate the start vertex and every neighbor in the DTO layer while keeping the domain function's atomic `RangeError` contract. Return distances alongside the start ID for auditability, and bound any trace of discoveries or queue growth with truncation metadata.

### 21. Mini Challenge
Implement `shortestPath(adjacency, start, target): number[] | null` with the same validation plus a `target` range check. Track each discovered vertex's parent during the same BFS, rebuild the path by walking parents backward from `target`, return it in start-to-target order, and return `null` when `target` is unreachable.

## Problem 198 — Dijkstra's Shortest Path

### 1. Problem Statement
Implement `dijkstraShortestPaths(adjacency: readonly (readonly { target: number; weight: number }[])[], start: number): number[]` for a weighted directed adjacency list whose vertices are dense IDs from `0` through `adjacency.length - 1`.

Validate `start` first: it must be a safe integer in `[0, adjacency.length)`, otherwise throw `RangeError`. Then validate every edge before allocating traversal state: each `target` must be a safe integer in range, and each `weight` must be a nonnegative safe integer. A negative weight is a contract violation and throws `RangeError`, because Dijkstra's greedy finalization is unsound with negative weights. Zero weights, parallel edges, and self-loops are valid.

Return a fresh array of minimum path weights from `start`, with `-1` for unreachable vertices and `distances[start] = 0`. Every evaluated distance addition must go through checked `safeAdd`; a sum outside the safe-integer range throws `RangeError`. Do not mutate the adjacency list or any edge object.

### 2. Example
```text
adjacency = [
  [{ target: 1, weight: 4 }, { target: 2, weight: 1 }],
  [{ target: 3, weight: 1 }],
  [{ target: 1, weight: 2 }, { target: 3, weight: 5 }],
  [],
  [{ target: 0, weight: 7 }],
]
start = 0

result = [0, 3, 1, 4, -1]
```

The direct edge to vertex `1` costs `4`, but the detour through vertex `2` costs `1 + 2 = 3`. Vertex `3` is cheapest through that detour plus one more edge: `3 + 1 = 4`. Vertex `4` only points into the component and is unreachable.

### 3. Understand the Problem
Each edge now carries a nonnegative integer cost, and a path's cost is the sum of its edge weights, so a route with more edges can be cheaper than a direct one. BFS order no longer matches cost order. The algorithm must always extend the cheapest known frontier vertex, and it must prove that once a vertex is taken from that frontier, no future route can undercut it — a proof that holds only because weights cannot be negative.

#### In Simple Terms
Find the cheapest total cost from the start to every vertex when each road has its own nonnegative price, reporting `-1` for vertices no route reaches.

### 4. Real-World Analogy
Picture toll roads between cities. The cheapest way to a far city may pass through several cheap tolls rather than one expensive highway. Planning outward from the origin, you always commit next to the city with the lowest total toll so far, because every remaining route into it would have to pay at least that much before adding more nonnegative tolls.

### 5. Key Observation
Among all vertices not yet finalized, the one with the smallest tentative distance can be finalized immediately: any alternative path to it must leave the finalized set through some tentative vertex whose distance is already no smaller, and nonnegative weights can only add cost after that. A single negative weight destroys this argument, which is why the contract rejects negative weights outright.

### 6. Pattern
```text
Pattern: Greedy Frontier Expansion with a Min Heap (Dijkstra)
Why this pattern applies: nonnegative weights let the cheapest tentative vertex be finalized greedily
How to recognize it: minimum-cost path where edges carry individual nonnegative costs
Similar problems: network delay time, cheapest flights, path with minimum effort, swim in rising water
```

### 7. Brute Force
After full validation, run Dijkstra with an array scan instead of a heap. Keep tentative distances and a `finalized` flag per vertex. Each round, linearly scan all vertices for the unfinalized one with the smallest known distance, finalize it, and relax its outgoing edges with `safeAdd`. Stop when no reachable unfinalized vertex remains.

This is the same greedy algorithm with the same validation and checked-arithmetic contract; only the minimum-selection data structure differs.

#### Pseudocode
```text
validate start, then every edge target and weight
distances = [-1] * V, distances[start] = 0
finalized = [false] * V
repeat up to V times:
  pick unfinalized vertex with smallest known distance (linear scan)
  if none reachable remains: stop
  finalize it
  for each outgoing edge:
    candidate = safeAdd(distance, weight)
    if target unreached or candidate smaller: update target
return distances
```

#### Brute-Force TypeScript
```typescript
function safeAdd(left: number, right: number): number {
  const sum = left + right;
  if (!Number.isSafeInteger(sum)) {
    throw new RangeError("unsafe graph arithmetic");
  }
  return sum;
}

function dijkstraShortestPathsWithArrayScan(
  adjacency: readonly (readonly { target: number; weight: number }[])[],
  start: number,
): number[] {
  const vertexCount = adjacency.length;
  if (!Number.isSafeInteger(start) || start < 0 || start >= vertexCount) {
    throw new RangeError("start must be a valid vertex ID");
  }
  for (const edges of adjacency) {
    for (const edge of edges) {
      if (
        !Number.isSafeInteger(edge.target) ||
        edge.target < 0 ||
        edge.target >= vertexCount
      ) {
        throw new RangeError("edge target must be a valid vertex ID");
      }
      if (!Number.isSafeInteger(edge.weight) || edge.weight < 0) {
        throw new RangeError("edge weight must be a nonnegative safe integer");
      }
    }
  }

  const distances = new Array<number>(vertexCount).fill(-1);
  distances[start] = 0;
  const finalized = new Array<boolean>(vertexCount).fill(false);

  for (let round = 0; round < vertexCount; round += 1) {
    let current = -1;
    let currentDistance = -1;
    for (let vertex = 0; vertex < vertexCount; vertex += 1) {
      const distance = distances[vertex]!;
      if (finalized[vertex] === true || distance === -1) continue;
      if (current === -1 || distance < currentDistance) {
        current = vertex;
        currentDistance = distance;
      }
    }
    if (current === -1) break;
    finalized[current] = true;

    for (const edge of adjacency[current]!) {
      const candidate = safeAdd(currentDistance, edge.weight);
      const best = distances[edge.target]!;
      if (best === -1 || candidate < best) {
        distances[edge.target] = candidate;
      }
    }
  }

  return distances;
}
```

#### Visualization
```text
round scans, start = 0:

round 1: pick 0 (0)   relax -> [0, 4, 1, -1, -1]
round 2: pick 2 (1)   relax -> [0, 3, 1, 6, -1]
round 3: pick 1 (3)   relax -> [0, 3, 1, 4, -1]
round 4: pick 3 (4)   no outgoing edges
round 5: only vertex 4 remains at -1 -> stop
```

#### Dry Run
Round 1 finalizes `0` and sets `distances[1] = 4`, `distances[2] = 1`. Round 2 scans and picks vertex `2`, improving vertex `1` to `3` and setting vertex `3` to `6`. Round 3 picks vertex `1` and improves vertex `3` to `4`. Round 4 finalizes vertex `3`, which has no edges. Round 5 finds only vertex `4` with `-1`, so the loop stops and returns `[0, 3, 1, 4, -1]`.

#### Brute-Force Complexity
Each of up to `V` rounds performs an `O(V)` scan, and every edge is relaxed at most once after its source is finalized, so time is `O(V^2 + E)`. Distances are required output; the finalized array uses `O(V)` auxiliary space. On dense graphs this is acceptable, but on sparse graphs the repeated full scans dominate the useful work.

### 8. Why Brute Force Is Not Ideal
The linear scan pays `O(V)` to find a minimum the algorithm already saw being created. A binary min heap remembers tentative distances in order, reducing each selection to `O(log V)`. For sparse graphs — the common production case — `O((V + E) log V)` is far below `O(V^2)`, and the heap version needs no new correctness reasoning, only stale-entry handling.

### 9. Optimal Approach
Validate `start`, then every edge, before allocating anything. Fill distances with `-1`, set `distances[start] = 0`, and push `{ vertex: start, distance: 0 }` into a binary min heap ordered by distance.

Repeatedly pop the smallest entry. Because this implementation pushes a fresh entry on every improvement instead of decreasing keys, the same vertex may sit in the heap several times; an entry whose distance exceeds the recorded best is stale and is skipped without work (lazy deletion). A fresh entry relaxes each outgoing edge: compute `safeAdd(entry.distance, edge.weight)` and, when it beats the target's best (or the target is unreached), record it and push a new heap entry. Equal-distance entries may pop in unspecified order, but every relaxation is a strict improvement, so the final distances are deterministic.

### 10. Visualization
```text
heap entries as (vertex, distance), start = 0:

push (0,0)
pop  (0,0)  relax: push (1,4), push (2,1)   distances [0, 4, 1, -1, -1]
pop  (2,1)  relax: push (1,3), push (3,6)   distances [0, 3, 1, 6, -1]
pop  (1,3)  relax: push (3,4)               distances [0, 3, 1, 4, -1]
pop  (1,4)  stale: 4 > best 3, skip
pop  (3,4)  no outgoing edges
pop  (3,6)  stale: 6 > best 4, skip

result: [0, 3, 1, 4, -1]
```

### 11. Step-by-Step Dry Run
| Popped entry | Stale? | Relaxations | Distances |
|---|---|---|---|
| `(0, 0)` | no | `1: -1 -> 4`, `2: -1 -> 1` | `[0, 4, 1, -1, -1]` |
| `(2, 1)` | no | `1: 4 -> 3`, `3: -1 -> 6` | `[0, 3, 1, 6, -1]` |
| `(1, 3)` | no | `3: 6 -> 4` | `[0, 3, 1, 4, -1]` |
| `(1, 4)` | yes | skipped | `[0, 3, 1, 4, -1]` |
| `(3, 4)` | no | none | `[0, 3, 1, 4, -1]` |
| `(3, 6)` | yes | skipped | `[0, 3, 1, 4, -1]` |

### 12. TypeScript Solution
```typescript
interface HeapEntry {
  vertex: number;
  distance: number;
}

class MinDistanceHeap {
  private readonly entries: HeapEntry[] = [];

  get size(): number {
    return this.entries.length;
  }

  push(entry: HeapEntry): void {
    this.entries.push(entry);
    let index = this.entries.length - 1;
    while (index > 0) {
      const parentIndex = (index - 1) >> 1;
      if (this.entries[parentIndex]!.distance <= this.entries[index]!.distance) {
        break;
      }
      this.swap(parentIndex, index);
      index = parentIndex;
    }
  }

  pop(): HeapEntry {
    const top = this.entries[0];
    if (top === undefined) throw new RangeError("heap is empty");
    const last = this.entries.pop()!;
    if (this.entries.length > 0) {
      this.entries[0] = last;
      let index = 0;
      for (;;) {
        const left = 2 * index + 1;
        const right = left + 1;
        let smallest = index;
        if (
          left < this.entries.length &&
          this.entries[left]!.distance < this.entries[smallest]!.distance
        ) {
          smallest = left;
        }
        if (
          right < this.entries.length &&
          this.entries[right]!.distance < this.entries[smallest]!.distance
        ) {
          smallest = right;
        }
        if (smallest === index) break;
        this.swap(smallest, index);
        index = smallest;
      }
    }
    return top;
  }

  private swap(first: number, second: number): void {
    const held = this.entries[first]!;
    this.entries[first] = this.entries[second]!;
    this.entries[second] = held;
  }
}

function safeAdd(left: number, right: number): number {
  const sum = left + right;
  if (!Number.isSafeInteger(sum)) {
    throw new RangeError("unsafe graph arithmetic");
  }
  return sum;
}

function dijkstraShortestPaths(
  adjacency: readonly (readonly { target: number; weight: number }[])[],
  start: number,
): number[] {
  const vertexCount = adjacency.length;
  if (!Number.isSafeInteger(start) || start < 0 || start >= vertexCount) {
    throw new RangeError("start must be a valid vertex ID");
  }
  for (const edges of adjacency) {
    for (const edge of edges) {
      if (
        !Number.isSafeInteger(edge.target) ||
        edge.target < 0 ||
        edge.target >= vertexCount
      ) {
        throw new RangeError("edge target must be a valid vertex ID");
      }
      if (!Number.isSafeInteger(edge.weight) || edge.weight < 0) {
        throw new RangeError("edge weight must be a nonnegative safe integer");
      }
    }
  }

  const distances = new Array<number>(vertexCount).fill(-1);
  distances[start] = 0;

  const heap = new MinDistanceHeap();
  heap.push({ vertex: start, distance: 0 });

  while (heap.size > 0) {
    const entry = heap.pop();
    if (entry.distance > distances[entry.vertex]!) continue;

    for (const edge of adjacency[entry.vertex]!) {
      const candidate = safeAdd(entry.distance, edge.weight);
      const best = distances[edge.target]!;
      if (best === -1 || candidate < best) {
        distances[edge.target] = candidate;
        heap.push({ vertex: edge.target, distance: candidate });
      }
    }
  }

  return distances;
}
```

### 13. Code Walkthrough
Validation of the start vertex and every edge completes before distances or the heap exist, keeping the failure atomic. The heap is self-contained: `push` sifts up, `pop` moves the last entry to the root and sifts down, and both compare only `distance`. Every push follows a recorded improvement, so a vertex's best distance is never `-1` when one of its entries pops; an entry whose distance exceeds that best is stale and skipped. Each relaxation routes its addition through `safeAdd`, so an overflowing candidate throws instead of corrupting comparisons.

### 14. Complexity Analysis
Every improvement pushes one heap entry, and each adjacency occurrence causes at most one improvement attempt, so the heap holds `O(V + E)` entries in the worst case. Pushes and pops cost `O(log V)` each (the heap size stays polynomial in `V`, keeping logarithms equivalent), giving `O((V + E) log V)` total time. Auxiliary space is `O(V + E)` worst case for the heap plus `O(V)` for distances, which are required output.

### 15. Edge Cases
- A single-vertex graph returns `[0]`.
- `[]` throws because no `start` is in range.
- Unreachable vertices return `-1` even when they have outgoing edges into the component.
- Zero-weight edges are valid and can chain without increasing cost.
- Parallel edges are all relaxed; only the cheaper effective one survives in distances.
- A self-loop never improves its own vertex because weights are nonnegative.
- A negative weight anywhere throws `RangeError` before traversal state is allocated.
- A `MAX_SAFE_INTEGER` weight on a reachable relaxation makes `safeAdd` throw instead of finishing.
- Equal-distance heap entries may pop in unspecified order, but final distances are deterministic.
- The adjacency list and every edge object remain unchanged, and the result is a fresh array.

### 16. Common Mistakes
- Accepting negative weights and silently returning wrong greedy answers.
- Skipping the stale-entry check and re-relaxing a vertex from an outdated distance.
- Implementing decrease-key with linear heap searches instead of lazy deletion.
- Adding distances with raw `+` and letting overflow scramble heap ordering.
- Using `-1` sentinel comparisons where an unreached target accidentally wins `candidate < best`.
- Validating edges lazily during relaxation after state has been allocated.
- Marking vertices finalized at push time rather than at first fresh pop.
- Using BFS layer logic on weighted edges and finalizing vertices in hop order.

### 17. Interview Follow-ups
1. Return one actual cheapest path with parent tracking. 2. Stop early once a given target pops fresh. 3. Handle negative weights with Bellman-Ford and detect negative cycles. 4. Compare against 0/1 BFS with a deque when weights are only 0 and 1. 5. Run A* with an admissible heuristic on a spatial graph.

### 18. Variations
- Network delay time. - Cheapest flights within K stops. - Path with minimum effort. - Swim in rising water. - Probability-maximizing paths with a max heap.

### 19. Pattern Connections
Dijkstra generalizes Problem 197: with all weights equal to one, the heap degenerates into BFS's queue and no entry ever goes stale. The heap itself is the structure from `13-heaps-priority-queues.md`, and lazy deletion revisits that chapter's stale-entry idea. Checked `safeAdd` mirrors the accumulation contract Kruskal-style algorithms use for total weights.

### 20. NestJS Project Connection
For `POST /api/dsa/graphs/dijkstra`, cap vertex count, edge occurrences, weight magnitude, and payload bytes well below language limits before service execution. DTO validation should reject negative or fractional weights early while the domain function keeps its atomic `RangeError` contract. Report unreachable vertices explicitly rather than encoding them as huge numbers, and bound any trace of pops, stale skips, and relaxations with truncation metadata.

### 21. Mini Challenge
Implement `dijkstraShortestPathTo(adjacency, start, target): { distance: number; path: number[] } | null` with the same validation plus a `target` range check. Track parents on every recorded improvement, stop when `target` pops fresh, rebuild the path from parents in start-to-target order, and return `null` when `target` is unreachable.

## Problem 199 — Minimum Spanning Tree

### 1. Problem Statement
Implement `minimumSpanningTree(vertexCount: number, edges: readonly (readonly [number, number, number])[]): { totalWeight: number; edges: [number, number, number][] } | null` for an undirected weighted graph given as an edge list over dense vertex IDs `0` through `vertexCount - 1`, where each entry is `[u, v, weight]`.

Validate `vertexCount` first: it must be a positive safe integer, otherwise throw `RangeError`. Then validate every edge before allocating any algorithm state: both endpoints must be safe integers in `[0, vertexCount)`, and each weight must be a nonnegative safe integer, otherwise throw `RangeError`. Self-loops are valid input but must never be selected, because an edge from a vertex to itself cannot connect two components. Parallel edges are valid, and the cheapest one wins naturally.

Return the total weight and the selected edges of a minimum spanning tree — a subset of `vertexCount - 1` edges connecting all vertices with minimum possible total weight. Emit selected edges in selection order, each oriented as `[treeVertex, newVertex, weight]` where `treeVertex` was already in the tree and `newVertex` is the vertex the edge brought in. Return `null` when the graph is disconnected, detected as fewer than `vertexCount - 1` edges selected. `vertexCount = 1` returns `{ totalWeight: 0, edges: [] }`. Every addition into `totalWeight` must go through checked `safeAdd`; a sum outside the safe-integer range throws `RangeError`. Do not mutate the edge list or any edge tuple.

### 2. Example
```text
vertexCount = 4
edges = [
  [0, 1, 4],
  [0, 2, 3],
  [0, 2, 1],
  [1, 2, 2],
  [1, 3, 1],
  [2, 3, 5],
  [0, 0, 9],
]

result = { totalWeight: 4, edges: [[0, 2, 1], [2, 1, 2], [1, 3, 1]] }
```

The self-loop `[0, 0, 9]` is never selected. Of the parallel edges `[0, 2, 3]` and `[0, 2, 1]`, only the cheaper one is chosen. The direct edge `[0, 1, 4]` loses to the two-step route through vertex `2` costing `1 + 2 = 3`, and vertex `3` attaches through its cheap edge to vertex `1`. Three edges span four vertices at total weight `4`.

### 3. Understand the Problem
A spanning tree keeps every vertex connected using exactly `vertexCount - 1` edges and no cycles; among all spanning trees, the problem asks for one with minimum total edge weight. Unlike shortest paths, the objective is global: no per-vertex distance matters, only the summed cost of the chosen edge set. The output must also witness the tree itself, not just its weight, and it must recognize when no spanning tree exists at all because the graph is disconnected.

#### In Simple Terms
Pick the cheapest set of edges that still lets you travel between every pair of vertices, report their total cost and the picked edges, and report `null` if some vertex can never be reached.

### 4. Real-World Analogy
Picture wiring offices together with fiber cable. Every office must end up on the network, cable segments have different prices, and loops are wasted money. Starting from one wired office, you always buy the cheapest cable that crosses from the wired cluster to an unwired office. No cheaper plan exists for that crossing: any network must eventually cross from wired to unwired somewhere, and this is the cheapest place to do it.

### 5. Key Observation
The cut property: for any split of the vertices into the current tree set and the rest, a minimum-weight edge crossing that split belongs to some minimum spanning tree. Swapping it into any spanning tree that omits it removes another crossing edge of equal or greater weight without disconnecting anything, so greedily committing to the cheapest crossing edge is always safe. This single exchange argument justifies the entire greedy algorithm.

### 6. Pattern
```text
Pattern: Greedy Edge Selection over a Growing Cut (Prim's MST)
Why this pattern applies: the cut property proves the cheapest edge leaving the tree is always safe to take
How to recognize it: connect every vertex of an undirected weighted graph at minimum total edge cost
Similar problems: min cost to connect all points, connecting cities with minimum cost, optimize water distribution
```

### 7. Brute Force
After full validation, run Prim's algorithm with array scans instead of a heap. Build an adjacency list from the edge list, skipping self-loops. Keep per-vertex arrays: `cheapest[v]` holds the lightest known weight of any edge from the tree to `v` (`-1` when none is known), and `bestFrom[v]` holds the tree endpoint of that edge. Seed `cheapest[0] = 0`. Each round, linearly scan for the untreed vertex with the smallest known candidate, pull it into the tree, record its stored edge through `safeAdd`, and relax its incident edges into the candidate arrays.

This is the same greedy algorithm with the same validation, orientation, and checked-arithmetic contract; only the minimum-selection data structure differs.

#### Pseudocode
```text
validate vertexCount, then every edge endpoint and weight
build adjacency from edges, skipping self-loops
cheapest = [-1] * V, cheapest[0] = 0
bestFrom = [-1] * V, inTree = [false] * V
repeat up to V times:
  pick untreed vertex with smallest cheapest != -1 (linear scan)
  if none remains: stop
  move it into the tree; if it has a stored edge, add weight via safeAdd and record the edge
  for each incident edge to an untreed neighbor:
    if weight beats the neighbor's cheapest: store weight and this vertex
if selected edge count < V - 1: return null
return { totalWeight, edges: selected }
```

#### Brute-Force TypeScript
```typescript
function safeAdd(left: number, right: number): number {
  const sum = left + right;
  if (!Number.isSafeInteger(sum)) {
    throw new RangeError("unsafe graph arithmetic");
  }
  return sum;
}

function minimumSpanningTreeWithArrayScan(
  vertexCount: number,
  edges: readonly (readonly [number, number, number])[],
): { totalWeight: number; edges: [number, number, number][] } | null {
  if (!Number.isSafeInteger(vertexCount) || vertexCount <= 0) {
    throw new RangeError("vertexCount must be a positive safe integer");
  }
  for (const [u, v, weight] of edges) {
    if (
      !Number.isSafeInteger(u) || u < 0 || u >= vertexCount ||
      !Number.isSafeInteger(v) || v < 0 || v >= vertexCount
    ) {
      throw new RangeError("edge endpoint must be a valid vertex ID");
    }
    if (!Number.isSafeInteger(weight) || weight < 0) {
      throw new RangeError("edge weight must be a nonnegative safe integer");
    }
  }

  const adjacency: { target: number; weight: number }[][] = Array.from(
    { length: vertexCount },
    () => [],
  );
  for (const [u, v, weight] of edges) {
    if (u === v) continue;
    adjacency[u]!.push({ target: v, weight });
    adjacency[v]!.push({ target: u, weight });
  }

  const inTree = new Array<boolean>(vertexCount).fill(false);
  const cheapest = new Array<number>(vertexCount).fill(-1);
  const bestFrom = new Array<number>(vertexCount).fill(-1);
  cheapest[0] = 0;

  const selected: [number, number, number][] = [];
  let totalWeight = 0;

  for (let round = 0; round < vertexCount; round += 1) {
    let current = -1;
    let currentWeight = -1;
    for (let vertex = 0; vertex < vertexCount; vertex += 1) {
      const candidate = cheapest[vertex]!;
      if (inTree[vertex] === true || candidate === -1) continue;
      if (current === -1 || candidate < currentWeight) {
        current = vertex;
        currentWeight = candidate;
      }
    }
    if (current === -1) break;
    inTree[current] = true;

    const from = bestFrom[current]!;
    if (from !== -1) {
      totalWeight = safeAdd(totalWeight, currentWeight);
      selected.push([from, current, currentWeight]);
    }

    for (const edge of adjacency[current]!) {
      if (inTree[edge.target] === true) continue;
      const known = cheapest[edge.target]!;
      if (known === -1 || edge.weight < known) {
        cheapest[edge.target] = edge.weight;
        bestFrom[edge.target] = current;
      }
    }
  }

  if (selected.length < vertexCount - 1) {
    return null;
  }
  return { totalWeight, edges: selected };
}
```

#### Visualization
```text
round scans on the example (self-loop dropped at adjacency build):

start:   cheapest [0, -1, -1, -1]   bestFrom [-1, -1, -1, -1]
round 1: pick 0 (seed, no edge)     relax -> cheapest [0, 4, 1, -1]  (parallel 3 then improved to 1)
round 2: pick 2 (1), edge [0,2,1]   relax -> cheapest [0, 2, 1, 5]
round 3: pick 1 (2), edge [2,1,2]   relax -> cheapest [0, 2, 1, 1]
round 4: pick 3 (1), edge [1,3,1]

selected = [[0,2,1], [2,1,2], [1,3,1]], totalWeight = 4
```

#### Dry Run
Round 1 pulls the seed vertex `0` into the tree with no recorded edge and relaxes its incident edges: vertex `1` gets candidate `4`, and vertex `2` first gets the parallel weight `3`, immediately improved to `1`. Round 2 scans, picks vertex `2` at weight `1`, records `[0, 2, 1]`, and improves vertex `1` to `2` and vertex `3` to `5`. Round 3 picks vertex `1`, records `[2, 1, 2]`, and improves vertex `3` to `1`. Round 4 picks vertex `3` and records `[1, 3, 1]`. Three edges equal `vertexCount - 1`, so the result is `{ totalWeight: 4, edges: [[0, 2, 1], [2, 1, 2], [1, 3, 1]] }`.

#### Brute-Force Complexity
Each of up to `V` rounds performs an `O(V)` scan, and every adjacency occurrence is relaxed at most once after its source joins the tree, so time is `O(V^2 + E)`. Auxiliary space is `O(V + E)` for the adjacency list and the candidate arrays. On dense graphs the scans are proportionate, but on sparse graphs they dominate the useful work.

### 8. Why Brute Force Is Not Ideal
The linear scan spends `O(V)` per round rediscovering a minimum the relaxations already produced. A binary min heap keeps candidate edges ordered, reducing each selection to `O(log E)`. For sparse graphs `O(E log E)` beats `O(V^2)` decisively, and the heap version needs no new correctness reasoning — only the same lazy stale-entry skipping Problem 198 already established.

### 9. Optimal Approach
Validate `vertexCount`, then every edge, before allocating anything. Build the adjacency list, skipping self-loops since they can never join two components. Push a seed entry `{ vertex: 0, weight: 0, from: -1 }` into a binary min heap ordered by weight; each heap entry stores the full candidate edge — the outside vertex, the connecting weight, and the tree endpoint it would attach through.

Repeatedly pop the lightest entry. If its vertex is already in the tree the entry is stale and is skipped without work (lazy deletion). A fresh entry commits its vertex: for non-seed entries, add the weight into `totalWeight` through `safeAdd` and record `[from, vertex, weight]` in selection order. Then push one candidate entry per incident edge to a vertex still outside the tree. Parallel edges each produce entries, and the cheapest pops first, making the others stale — the cheapest wins naturally. When the heap drains with fewer than `vertexCount - 1` selections, the graph is disconnected and the result is `null`.

Kruskal's algorithm is the classic alternative: sort all edges by weight and take each edge whose endpoints lie in different components, tracked with the Disjoint Set Union structure that Problem 200 builds. Both greedy strategies are correct by the same cut property. On determinism: every minimum spanning tree of a graph has the same total weight, and the edge set is unique exactly when it makes greedy choices unambiguous — with distinct weights the MST is unique. When weights tie, alternative valid MSTs exist; this implementation is nonetheless fully deterministic given the input order, always producing the same edge list for the same input.

### 10. Visualization
```text
heap entries as (vertex, weight, from), seed at vertex 0:

push (0,0,-1)
pop  (0,0,-1)  fresh seed          push (1,4,0), (2,3,0), (2,1,0)
pop  (2,1,0)   select [0,2,1]      push (1,2,2), (3,5,2)         total 1
pop  (1,2,2)   select [2,1,2]      push (3,1,1)                  total 3
pop  (3,1,1)   select [1,3,1]      no untreed neighbors          total 4
pop  (2,3,0)   stale: 2 in tree, skip
pop  (1,4,0)   stale: 1 in tree, skip
pop  (3,5,2)   stale: 3 in tree, skip

result: { totalWeight: 4, edges: [[0,2,1], [2,1,2], [1,3,1]] }
```

### 11. Step-by-Step Dry Run
| Popped entry | Stale? | Action | Total weight | Selected edges |
|---|---|---|---|---|
| `(0, 0, -1)` | no | seed, push 3 candidates | `0` | `[]` |
| `(2, 1, 0)` | no | select `[0,2,1]`, push 2 | `1` | `[[0,2,1]]` |
| `(1, 2, 2)` | no | select `[2,1,2]`, push 1 | `3` | `[[0,2,1],[2,1,2]]` |
| `(3, 1, 1)` | no | select `[1,3,1]` | `4` | `[[0,2,1],[2,1,2],[1,3,1]]` |
| `(2, 3, 0)` | yes | skipped | `4` | unchanged |
| `(1, 4, 0)` | yes | skipped | `4` | unchanged |
| `(3, 5, 2)` | yes | skipped | `4` | unchanged |

### 12. TypeScript Solution
```typescript
interface CandidateEdge {
  vertex: number;
  weight: number;
  from: number;
}

class MinCandidateHeap {
  private readonly entries: CandidateEdge[] = [];

  get size(): number {
    return this.entries.length;
  }

  push(entry: CandidateEdge): void {
    this.entries.push(entry);
    let index = this.entries.length - 1;
    while (index > 0) {
      const parentIndex = (index - 1) >> 1;
      if (this.entries[parentIndex]!.weight <= this.entries[index]!.weight) {
        break;
      }
      this.swap(parentIndex, index);
      index = parentIndex;
    }
  }

  pop(): CandidateEdge {
    const top = this.entries[0];
    if (top === undefined) throw new RangeError("heap is empty");
    const last = this.entries.pop()!;
    if (this.entries.length > 0) {
      this.entries[0] = last;
      let index = 0;
      for (;;) {
        const left = 2 * index + 1;
        const right = left + 1;
        let smallest = index;
        if (
          left < this.entries.length &&
          this.entries[left]!.weight < this.entries[smallest]!.weight
        ) {
          smallest = left;
        }
        if (
          right < this.entries.length &&
          this.entries[right]!.weight < this.entries[smallest]!.weight
        ) {
          smallest = right;
        }
        if (smallest === index) break;
        this.swap(smallest, index);
        index = smallest;
      }
    }
    return top;
  }

  private swap(first: number, second: number): void {
    const held = this.entries[first]!;
    this.entries[first] = this.entries[second]!;
    this.entries[second] = held;
  }
}

function safeAdd(left: number, right: number): number {
  const sum = left + right;
  if (!Number.isSafeInteger(sum)) {
    throw new RangeError("unsafe graph arithmetic");
  }
  return sum;
}

function minimumSpanningTree(
  vertexCount: number,
  edges: readonly (readonly [number, number, number])[],
): { totalWeight: number; edges: [number, number, number][] } | null {
  if (!Number.isSafeInteger(vertexCount) || vertexCount <= 0) {
    throw new RangeError("vertexCount must be a positive safe integer");
  }
  for (const [u, v, weight] of edges) {
    if (
      !Number.isSafeInteger(u) || u < 0 || u >= vertexCount ||
      !Number.isSafeInteger(v) || v < 0 || v >= vertexCount
    ) {
      throw new RangeError("edge endpoint must be a valid vertex ID");
    }
    if (!Number.isSafeInteger(weight) || weight < 0) {
      throw new RangeError("edge weight must be a nonnegative safe integer");
    }
  }

  const adjacency: { target: number; weight: number }[][] = Array.from(
    { length: vertexCount },
    () => [],
  );
  for (const [u, v, weight] of edges) {
    if (u === v) continue;
    adjacency[u]!.push({ target: v, weight });
    adjacency[v]!.push({ target: u, weight });
  }

  const inTree = new Array<boolean>(vertexCount).fill(false);
  const selected: [number, number, number][] = [];
  let totalWeight = 0;

  const heap = new MinCandidateHeap();
  heap.push({ vertex: 0, weight: 0, from: -1 });

  while (heap.size > 0) {
    const entry = heap.pop();
    if (inTree[entry.vertex] === true) continue;
    inTree[entry.vertex] = true;

    if (entry.from !== -1) {
      totalWeight = safeAdd(totalWeight, entry.weight);
      selected.push([entry.from, entry.vertex, entry.weight]);
    }

    for (const edge of adjacency[entry.vertex]!) {
      if (inTree[edge.target] === true) continue;
      heap.push({ vertex: edge.target, weight: edge.weight, from: entry.vertex });
    }
  }

  if (selected.length < vertexCount - 1) {
    return null;
  }
  return { totalWeight, edges: selected };
}
```

### 13. Code Walkthrough
Validation of `vertexCount` and every edge completes before the adjacency list, tree flags, or heap exist, keeping the failure atomic. The adjacency build drops self-loops, so no heap entry can ever propose one. Each heap entry carries a complete candidate edge, which lets a fresh pop both accumulate `totalWeight` through `safeAdd` and emit `[from, vertex, weight]` in the discovered orientation without any lookup. The `inTree` check at pop time implements lazy deletion: parallel and superseded candidates simply skip. The final length check converts an undrained frontier into the `null` disconnected answer, and `vertexCount = 1` falls through with zero selections satisfying `0 >= 0`.

### 14. Complexity Analysis
Each undirected edge appears twice in the adjacency list, and each occurrence pushes at most one heap entry when its source joins the tree, so the heap holds `O(E)` entries. Pushes and pops cost `O(log E)` each, giving `O(E log E)` time — equivalently `O(E log V)`, since `E <= V^2` makes the logarithms proportional — plus `O(V + E)` for validation and the adjacency build. Auxiliary space is `O(V + E)` for the adjacency list, tree flags, and heap; the selected edge list is required output.

### 15. Edge Cases
- `vertexCount = 1` returns `{ totalWeight: 0, edges: [] }`, even when self-loops on vertex `0` exist.
- A zero or negative or fractional `vertexCount` throws `RangeError` before edges are examined.
- A disconnected graph returns `null`, including `vertexCount = 2` with no edges.
- Self-loops pass validation but are never selected.
- Parallel edges are all pushed; the cheapest pops first and the rest go stale.
- Zero-weight edges are valid and contribute nothing to `totalWeight`.
- An out-of-range endpoint or negative, fractional, or unsafe weight throws `RangeError` atomically.
- Two `MAX_SAFE_INTEGER` weights on selected edges make `safeAdd` throw instead of corrupting the total.
- Tied weights admit alternative valid MSTs; this implementation still returns one deterministic edge list for a given input.
- The edge list and every tuple remain unchanged, and the returned object and arrays are fresh.

### 16. Common Mistakes
- Skipping the stale-entry check and selecting the same vertex twice.
- Detecting disconnection by heap exhaustion alone instead of counting selected edges.
- Allowing self-loops into the adjacency list and accidentally selecting one on a single-vertex component.
- Treating the edge list as directed and only adding one adjacency direction.
- Accumulating `totalWeight` with raw `+` and letting overflow pass silently.
- Validating edges lazily during the adjacency build after state has been allocated.
- Emitting edges in input order instead of selection order with the discovered orientation.
- Assuming the MST edge set is unique when weights tie.

### 17. Interview Follow-ups
1. Implement Kruskal's algorithm with sorting and the DSU from Problem 200, and compare constant factors. 2. Produce a maximum spanning tree by negating the comparison. 3. Decide whether the MST is unique. 4. Find the second-best spanning tree by exchanging one edge. 5. Maintain an MST as edges are added online.

### 18. Variations
- Min cost to connect all points. - Connecting cities with minimum cost. - Optimize water distribution in a village. - Find critical and pseudo-critical MST edges. - Maximum spanning tree.

### 19. Pattern Connections
The heap with lazy stale-entry skipping is lifted directly from Problem 198 — Prim is Dijkstra with the priority changed from path distance to single-edge weight, which is why no `safeAdd` appears in relaxation, only in the total. The heap structure itself comes from `13-heaps-priority-queues.md`. Kruskal's alternative formulation is the flagship consumer of Problem 200's Disjoint Set Union, where a failed union identifies a cycle-forming edge exactly as Problem 189 detects cycles.

### 20. NestJS Project Connection
For `POST /api/dsa/graphs/minimum-spanning-tree`, cap vertex count, edge count, weight magnitude, and payload bytes well below language limits before service execution. DTO validation should reject malformed tuples, fractional weights, and out-of-range endpoints early while the domain function keeps its atomic `RangeError` contract. Return the disconnected case as an explicit `null`-bodied or flagged response rather than an error, and bound any trace of pops, stale skips, and selections with truncation metadata.

### 21. Mini Challenge
Implement `minimumSpanningForest(vertexCount, edges): { totalWeight: number; edges: [number, number, number][]; componentCount: number }` with the same validation. Instead of returning `null` on disconnection, seed the heap once per unreached vertex, build the minimum spanning forest across all components, and report how many components it contains.

## Problem 200 — Disjoint Set Union / Union Find

### 1. Problem Statement
Implement a class `DisjointSetUnion` maintaining a partition of dense vertex IDs `0` through `size - 1` into disjoint components:

- `constructor(size: number)` — `size` must be a positive safe integer, otherwise throw `RangeError`. Initially every vertex is its own component.
- `find(x: number): number` — return the representative (root) of `x`'s component, applying full path compression so every vertex on the walked path points directly at the root afterward.
- `union(a: number, b: number): boolean` — merge the components of `a` and `b` using union by rank; return `true` only when two distinct roots actually merge, `false` when `a` and `b` were already in the same component.
- `connected(a: number, b: number): boolean` — whether `a` and `b` share a component.
- `get componentCount(): number` — the current number of disjoint components.

Every method taking vertex arguments must validate each one as a safe integer in `[0, size)` and throw `RangeError` otherwise, before touching any state. The structure never shrinks or grows after construction.

### 2. Example
```text
dsu = new DisjointSetUnion(6)          componentCount = 6
dsu.union(0, 1)      -> true           componentCount = 5
dsu.union(2, 3)      -> true           componentCount = 4
dsu.union(1, 3)      -> true           componentCount = 3   {0,1,2,3} merged
dsu.union(0, 2)      -> false          componentCount = 3   already connected
dsu.connected(1, 2)  -> true
dsu.connected(4, 5)  -> false
dsu.find(3)          -> 0              (root of {0,1,2,3} in this run)
```

Each successful `union` reduces the component count by exactly one; the failed `union(0, 2)` changes nothing and reports `false`.

### 3. Understand the Problem
The structure answers two online questions about a shifting partition: "merge these two groups" and "are these two items in the same group". Components only ever merge — nothing splits — so each component can be represented by a single canonical vertex, its root, and both questions reduce to comparing roots. The challenge is keeping root lookup fast even after long chains of merges, which is what path compression and union by rank exist to guarantee.

#### In Simple Terms
Keep track of which items belong to the same group while groups keep merging, and answer "same group?" almost instantly no matter how many merges have happened.

### 4. Real-World Analogy
Think of companies merging. Each company has one chief executive; when two companies merge, one chief becomes the boss of both and the other now reports to them. Asking whether two employees work for the same company means walking up the reporting chain to the top — and once you have walked it, you pin a shortcut so everyone you passed reports directly to the top chief, making every future question about them instant.

### 5. Key Observation
Only root identity matters: the internal tree shape carries no information beyond which root each vertex reaches, so the trees can be reshaped freely as long as membership is preserved. Path compression flattens every walked path to depth one, and union by rank attaches the shallower tree under the deeper root so no walk gets long in the first place. Either optimization alone gives `O(log n)`; together they give amortized inverse-Ackermann time, which is constant for every practical input.

### 6. Pattern
```text
Pattern: Disjoint Set Union (Union-Find) with Path Compression and Union by Rank
Why this pattern applies: online merges and connectivity queries need near-constant amortized cost
How to recognize it: a stream of "merge these" and "are these connected" operations over dense IDs
Similar problems: number of provinces, redundant connection, accounts merge, Kruskal's minimum spanning tree
```

### 7. Brute Force
Represent each component by an explicit label per vertex: `labels[x]` is the identity of `x`'s component. `find` is a direct `O(1)` array read. `union` compares the two labels, and when they differ, sweeps the entire array relabeling every vertex of one component to the other's label and decrements the count. `connected` compares labels, and `componentCount` returns the maintained counter.

The API, validation contract, and observable behavior — including `union`'s boolean and the count progression — are identical; only the merge cost differs.

#### Pseudocode
```text
constructor(size): validate size; labels = [0..size-1]; components = size
find(x): validate x; return labels[x]
union(a, b):
  labelA = find(a); labelB = find(b)
  if labelA == labelB: return false
  for every vertex: if its label is labelB, relabel to labelA
  components -= 1; return true
connected(a, b): find(a) == find(b)
componentCount: return components
```

#### Brute-Force TypeScript
```typescript
class DisjointSetUnionWithLabels {
  private readonly labels: number[];
  private components: number;

  constructor(size: number) {
    if (!Number.isSafeInteger(size) || size <= 0) {
      throw new RangeError("size must be a positive safe integer");
    }
    this.labels = Array.from({ length: size }, (_, vertex) => vertex);
    this.components = size;
  }

  get componentCount(): number {
    return this.components;
  }

  find(x: number): number {
    this.assertVertex(x);
    return this.labels[x]!;
  }

  union(a: number, b: number): boolean {
    const labelA = this.find(a);
    const labelB = this.find(b);
    if (labelA === labelB) {
      return false;
    }
    for (let vertex = 0; vertex < this.labels.length; vertex += 1) {
      if (this.labels[vertex]! === labelB) {
        this.labels[vertex] = labelA;
      }
    }
    this.components -= 1;
    return true;
  }

  connected(a: number, b: number): boolean {
    return this.find(a) === this.find(b);
  }

  private assertVertex(x: number): void {
    if (!Number.isSafeInteger(x) || x < 0 || x >= this.labels.length) {
      throw new RangeError("vertex must be a valid ID");
    }
  }
}
```

#### Visualization
```text
size 6, labels start [0, 1, 2, 3, 4, 5], components 6

union(0,1): labels differ (0,1)  relabel 1s -> [0, 0, 2, 3, 4, 5]  components 5
union(2,3): labels differ (2,3)  relabel 3s -> [0, 0, 2, 2, 4, 5]  components 4
union(1,3): labels differ (0,2)  relabel 2s -> [0, 0, 0, 0, 4, 5]  components 3
union(0,2): labels equal  (0,0)  no sweep                          components 3, false
```

#### Dry Run
`union(0, 1)` reads labels `0` and `1`, sweeps all six slots relabeling the single `1` to `0`, and drops the count to `5`. `union(2, 3)` similarly merges to `[0, 0, 2, 2, 4, 5]` with count `4`. `union(1, 3)` reads labels `0` and `2` and relabels both `2`s to `0`, giving `[0, 0, 0, 0, 4, 5]` and count `3`. `union(0, 2)` reads equal labels `0` and returns `false` without sweeping. `connected(1, 2)` compares `0` with `0` and returns `true`; `connected(4, 5)` compares `4` with `5` and returns `false`.

#### Brute-Force Complexity
`find` and `connected` are `O(1)`, but every successful `union` sweeps all `n` slots, so a sequence of `m` operations costs `O(m * n)` worst case — a build-up of `n - 1` merges alone is `O(n^2)`. Space is `O(n)` for the label array. The relabeling sweep does work proportional to the whole universe for what is conceptually a constant-size bookkeeping change.

### 8. Why Brute Force Is Not Ideal
Relabeling an entire component on every merge repeats work that a pointer redirect can do in one step: pointing one root at another merges arbitrarily large components in `O(1)` writes. The cost moves into `find`, and the two classic optimizations — path compression and union by rank — drive that cost down to amortized inverse-Ackermann, effectively constant, without changing any observable behavior of the API.

### 9. Optimal Approach
Store a parent forest: `parent[x]` points toward `x`'s root, with roots pointing at themselves, plus a `rank` array of tree-depth upper bounds and a live component counter. `find` walks parent pointers iteratively to the root, then makes a second pass rewriting every visited vertex's parent directly to the root — full path compression without recursion, so no stack depth limits apply. `union` finds both roots; equal roots return `false` untouched. Distinct roots attach the lower-rank root under the higher-rank one; on a rank tie either may win, the winner's rank increments, and the counter decrements.

The amortized cost per operation is `O(alpha(n))`, where `alpha` is the inverse Ackermann function — stated honestly, this is not `O(1)`, but `alpha(n) <= 4` for any `n` that fits in a physical machine, so it is constant for all practical purposes. Rank stays an upper bound rather than the exact height after compression shortens paths, which is exactly why the analysis uses rank and not height. This structure is the engine behind Kruskal's algorithm for Problem 199 — each accepted edge is a successful `union`, each rejected edge a failed one — and the failed-`union` signal is precisely the cycle detection of Problem 189.

### 10. Visualization
```text
size 6, parent index 0..5, rank all 0

start:      parent [0, 1, 2, 3, 4, 5]                    components 6
union(0,1): roots 0,1 tie -> parent[1]=0, rank[0]=1
            parent [0, 0, 2, 3, 4, 5]                    components 5
union(2,3): roots 2,3 tie -> parent[3]=2, rank[2]=1
            parent [0, 0, 2, 2, 4, 5]                    components 4
union(1,3): find(1)=0, find(3)=2, ranks tie at 1
            -> parent[2]=0, rank[0]=2
            parent [0, 0, 0, 2, 4, 5]                    components 3
find(3):    walk 3 -> 2 -> 0, compress parent[3]=0
            parent [0, 0, 0, 0, 4, 5]
```

### 11. Step-by-Step Dry Run
| Operation | Roots found | Returns | Parent array after | componentCount |
|---|---|---|---|---|
| `new DisjointSetUnion(6)` | — | — | `[0, 1, 2, 3, 4, 5]` | `6` |
| `union(0, 1)` | `0`, `1` | `true` | `[0, 0, 2, 3, 4, 5]` | `5` |
| `union(2, 3)` | `2`, `3` | `true` | `[0, 0, 2, 2, 4, 5]` | `4` |
| `union(1, 3)` | `0`, `2` | `true` | `[0, 0, 0, 2, 4, 5]` | `3` |
| `union(0, 2)` | `0`, `0` | `false` | unchanged | `3` |
| `find(3)` | `0` | `0` | `[0, 0, 0, 0, 4, 5]` (compressed) | `3` |
| `connected(4, 5)` | `4`, `5` | `false` | unchanged | `3` |

### 12. TypeScript Solution
```typescript
class DisjointSetUnion {
  private readonly parent: number[];
  private readonly rank: number[];
  private components: number;

  constructor(size: number) {
    if (!Number.isSafeInteger(size) || size <= 0) {
      throw new RangeError("size must be a positive safe integer");
    }
    this.parent = Array.from({ length: size }, (_, vertex) => vertex);
    this.rank = new Array<number>(size).fill(0);
    this.components = size;
  }

  get componentCount(): number {
    return this.components;
  }

  find(x: number): number {
    this.assertVertex(x);
    let root = x;
    while (this.parent[root]! !== root) {
      root = this.parent[root]!;
    }
    let current = x;
    while (current !== root) {
      const next = this.parent[current]!;
      this.parent[current] = root;
      current = next;
    }
    return root;
  }

  union(a: number, b: number): boolean {
    const rootA = this.find(a);
    const rootB = this.find(b);
    if (rootA === rootB) {
      return false;
    }
    const rankA = this.rank[rootA]!;
    const rankB = this.rank[rootB]!;
    if (rankA < rankB) {
      this.parent[rootA] = rootB;
    } else if (rankA > rankB) {
      this.parent[rootB] = rootA;
    } else {
      this.parent[rootB] = rootA;
      this.rank[rootA] = rankA + 1;
    }
    this.components -= 1;
    return true;
  }

  connected(a: number, b: number): boolean {
    return this.find(a) === this.find(b);
  }

  private assertVertex(x: number): void {
    if (!Number.isSafeInteger(x) || x < 0 || x >= this.parent.length) {
      throw new RangeError("vertex must be a valid ID");
    }
  }
}
```

### 13. Code Walkthrough
The constructor validates `size` before allocating, then builds the identity forest where each vertex roots itself at rank zero. `find` validates its argument, walks to the root in a first pass, then rewrites every visited parent pointer in a second pass — full compression with plain iteration, immune to stack overflow on adversarially long chains. `union` delegates validation to its two `find` calls, so invalid arguments throw before any state changes; equal roots return `false` with nothing touched, and distinct roots merge by rank, incrementing rank only on ties and decrementing the counter exactly once per real merge. `connected` is two compressed finds and a comparison, and `componentCount` reads the counter maintained solely by successful unions.

### 14. Complexity Analysis
With both path compression and union by rank, any sequence of `m` operations on `n` vertices costs `O(m * alpha(n))` — amortized `O(alpha(n))` per operation, where the inverse Ackermann function `alpha(n)` is at most `4` for any physically representable `n`, so practically constant while honestly not `O(1)`. A single uncompressed-path `find` can still cost `O(log n)` thanks to rank alone. Space is `O(n)` for the parent and rank arrays, fixed at construction.

### 15. Edge Cases
- `new DisjointSetUnion(1)` has `componentCount` `1`, `find(0)` returns `0`, and `union(0, 0)` returns `false`.
- A zero, negative, fractional, or unsafe `size` throws `RangeError` from the constructor.
- Every method throws `RangeError` on negative, out-of-range, fractional, or unsafe vertex arguments before mutating anything.
- `union(a, a)` always returns `false` and never changes `componentCount`.
- Repeating a successful `union` returns `false` the second time; `componentCount` drops exactly once per distinct merge.
- `connected(x, x)` is always `true`.
- `componentCount` never falls below `1`, since a single component has no distinct roots left to merge.
- After a deep chain merges, one `find` on the deepest vertex flattens the whole walked path; a second identical `find` performs no writes.
- `rank` is a depth upper bound, not the exact height, once compression has shortened paths — this is expected and harmless.
- The structure holds exactly `size` vertices forever; there is no add or remove.

### 16. Common Mistakes
- Implementing `find` recursively and overflowing the stack on long parent chains.
- Decrementing `componentCount` before checking whether the roots were already equal.
- Compressing only the queried vertex instead of every vertex on the walked path.
- Incrementing rank on unequal-rank merges instead of only on ties.
- Validating arguments in the constructor only and letting methods index out of range silently.
- Comparing `parent[a]` with `parent[b]` in `connected` instead of comparing roots.
- Treating rank as the exact tree height after compression and reasoning incorrectly from it.
- Returning `true` from `union` unconditionally, breaking cycle detection built on the boolean.

### 17. Interview Follow-ups
1. Track component sizes at the roots and expose `sizeOf(x)`. 2. Implement union by size instead of rank and compare guarantees. 3. Build a rollback DSU — no compression, a stack of parent/rank changes — for offline dynamic connectivity. 4. Extend to a weighted DSU storing parity or offsets along parent pointers. 5. Support arbitrary keys with a map from key to dense ID.

### 18. Variations
- Number of provinces. - Redundant connection. - Accounts merge. - Most stones removed with same row or column. - Satisfiability of equality equations.

### 19. Pattern Connections
This structure is the engine of Kruskal's algorithm, the classic alternative to Problem 199's Prim: scan edges by ascending weight and keep exactly those whose `union` returns `true`. That same boolean is Problem 189's undirected cycle detector in disguise — an edge whose endpoints are already connected closes a cycle. The dense-ID validation contract matches every problem in this chapter, and the amortized-analysis mindset echoes the stale-entry accounting of Problems 198 and 199.

### 20. NestJS Project Connection
For `POST /api/dsa/graphs/union-find`, accept a `size` and a bounded list of operations, capping size, operation count, and payload bytes well below language limits before service execution. DTO validation should whitelist operation names and reject fractional or out-of-range indices early while the domain class keeps its per-method `RangeError` contract. Return per-operation results alongside the final `componentCount` for auditability, and bound any trace of parent-array snapshots with truncation metadata.

### 21. Mini Challenge
Add per-component size tracking: maintain a `sizes` array meaningful at roots, seed every entry with `1`, and on each successful `union` add the loser root's size into the winner's. Expose `sizeOf(x: number): number` with the same argument validation, returning the size of `x`'s component in amortized `O(alpha(n))`. As a harder alternative, build a rollback variant: drop path compression, keep union by rank, record each union's overwritten parent and rank in a stack, and expose `rollback()` undoing the most recent successful union.

# Topic Summary

## What You Learned
Graphs generalize trees: vertices connect through arbitrary edges, cycles and disconnected components are normal, and dense IDs `0..V-1` let adjacency lists index directly into arrays. Full validation before any traversal keeps failure atomic, and readonly inputs with fresh outputs keep ownership explicit. Directed and undirected graphs need different cycle logic: undirected traversal must consume one parent-edge occurrence, while directed detection needs a recursion-stack or three-color state.

Breadth-first search explores by distance layers with a head-index queue and answers unweighted shortest-path questions exactly. Depth-first search with explicit resumable frames reproduces recursive order without stack-overflow risk. Grid problems are graphs in disguise: rectangular validation, four-directional movement, and visited state per cell. Kahn's algorithm converts indegree bookkeeping into topological order and proves cycles by counting emitted vertices.

Weighted graphs add arithmetic contracts: Dijkstra requires nonnegative weights and checked distance additions, Prim's cut property justifies greedy MST growth, and both use lazy stale-entry skipping instead of decrease-key. Disjoint Set Union tracks components across incremental unions with path compression and union by rank, complementing traversal-based connectivity when edges arrive over time.

## Patterns You Should Recognize
- **Validate-then-traverse:** reject the whole structure atomically before any work.
- **Head-index queue BFS:** layer-order exploration without `shift()` costs.
- **Resumable-frame DFS:** exact recursive preorder with owned stack storage.
- **Parent-edge accounting:** consume one occurrence to tell cycles from backtracking in undirected graphs.
- **Recursion-stack coloring:** distinguish back edges from cross edges in directed graphs.
- **Grid-as-graph:** rectangular validation plus four-directional neighbor generation.
- **Multi-source BFS:** seed every source at distance zero and expand one frontier.
- **Identity-map cloning:** map originals to copies before expanding cyclic neighbors.
- **Indegree draining:** emit zero-indegree vertices and prove acyclicity by count.
- **Lazy-deletion heaps:** push duplicates and skip stale entries instead of decrease-key.
- **Cut-property greedy growth:** the cheapest crossing edge is always MST-safe.
- **Compressed forests:** near-constant amortized connectivity through path compression and rank.

## Complexity Cheat Sheet
| Problem | Main approach | Time | Auxiliary space | Required output space |
|---:|---|---|---|---|
| 186 | Validated adjacency-list construction | `O(V + E)` | `O(1)` beyond output | `O(V + E)` fresh lists |
| 187 | Head-index queue BFS | `O(V + E)` | `O(V)` | `O(reached)` order array |
| 188 | Resumable-frame DFS | `O(V + E)` | `O(V)` | `O(reached)` order array |
| 189 | Parent-edge BFS per component | `O(V + E)` | `O(V)` | `O(1)` boolean |
| 190 | Three-color directed DFS | `O(V + E)` | `O(V)` | `O(1)` boolean |
| 191 | Component-count grid BFS | `O(R * C)` | `O(R * C)` worst case | `O(1)` scalar |
| 192 | Fresh-copy flood fill | `O(R * C)` | `O(R * C)` worst case | `O(R * C)` fresh image |
| 193 | Multi-source rotting BFS | `O(R * C)` | `O(R * C)` worst case | `O(1)` scalar |
| 194 | Identity-map graph clone | `O(V + E)` | `O(V)` map and queue | `O(V + E)` fresh graph |
| 195 | Kahn indegree draining | `O(V + E)` | `O(V)` | `O(V)` order or `null` |
| 196 | Prerequisite-edge Kahn feasibility | `O(V + E)` | `O(V + E)` built adjacency | `O(1)` boolean |
| 197 | Unweighted BFS distances | `O(V + E)` | `O(V)` | `O(V)` distances |
| 198 | Lazy-deletion heap Dijkstra | `O((V + E) log V)` | `O(V + E)` worst case | `O(V)` distances |
| 199 | Prim with stale-entry skipping | `O(E log E)` | `O(V + E)` | `O(V)` selected edges or `null` |
| 200 | Path compression and union by rank | `O(alpha(n))` amortized per operation | `O(n)` owned forest | `O(1)` per query |

## Common Interview Traps
- Traversing before validating neighbor IDs and letting a bad edge corrupt partial results.
- Calling `shift()` on a BFS queue and silently making traversal quadratic.
- Reverse-pushing all neighbors onto a DFS stack and claiming exact recursive preorder on graphs with cross edges.
- Marking an undirected cycle at the parent edge instead of consuming one parent occurrence, breaking on parallel edges.
- Using a plain visited set for directed cycle detection and misreading cross edges as cycles.
- Forgetting that a self-loop is always a cycle in both orientations.
- Skipping rectangular-grid validation and indexing into jagged rows.
- Flood-filling when the start color equals the new color and looping forever, or mutating the caller's image.
- Counting rotten minutes off by one or ignoring unreachable fresh oranges.
- Cloning a cyclic graph without registering the copy before expanding neighbors.
- Emitting a partial topological order instead of `null` when Kahn drains fewer than `V` vertices.
- Reversing prerequisite orientation so courses unlock their own prerequisites.
- Running Dijkstra with negative weights or skipping checked distance additions near safe-integer bounds.
- Returning a spanning forest when the contract requires `null` for disconnected MST input.
- Implementing DSU find without compression and quietly degrading to linear chains.
- Claiming a unique MST edge set when tied weights permit alternatives, or nondeterministic totals when total weight is invariant.

## Practice Checklist
- [ ] 186. Build validated adjacency lists atomically with preserved edge order.
- [ ] 187. Traverse breadth-first with a head-index queue from a validated start.
- [ ] 188. Reproduce recursive DFS order with explicit resumable frames.
- [ ] 189. Detect undirected cycles by consuming exactly one parent-edge occurrence.
- [ ] 190. Detect directed cycles with recursion-stack coloring, not just visited sets.
- [ ] 191. Count islands with component-scoped grid BFS and full rectangular validation.
- [ ] 192. Flood fill a fresh image copy with an early same-color exit.
- [ ] 193. Rot oranges with multi-source BFS and report unreachable fresh cells.
- [ ] 194. Clone cyclic graphs through an original-to-copy identity map.
- [ ] 195. Produce a topological order or `null` by draining indegrees deterministically.
- [ ] 196. Decide course feasibility with prerequisite-to-course edges and emitted counts.
- [ ] 197. Compute unit-edge shortest distances with `-1` for unreachable vertices.
- [ ] 198. Run Dijkstra with nonnegative validated weights, safeAdd, and stale-entry skipping.
- [ ] 199. Grow a minimum spanning tree with Prim's cut property or return `null`.
- [ ] 200. Maintain components with path compression, union by rank, and exact union results.
