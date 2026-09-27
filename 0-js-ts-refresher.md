# JavaScript / TypeScript Refresher

Read this before `00-dsa-fundamentals.md`. Every code snippet in this curriculum compiles under TypeScript's strictest settings (`--strict --noUncheckedIndexedAccess --target ES2022`). This chapter is a refresher for every piece of language syntax those snippets use — nothing more, nothing less. If you can read every example here comfortably, no line of code in Problems 1–200 will surprise you syntactically.

## Learning Objectives

By the end of this refresher, you should be able to:

- declare and type variables, functions, arrays, tuples, objects, maps, sets, and classes
- read and write union types such as `number | null` and narrow them safely
- understand why indexing an array can produce `undefined` under `noUncheckedIndexedAccess`, and the guard pattern used everywhere in this curriculum
- work with safe integers, `Number.isSafeInteger`, `Number.MAX_SAFE_INTEGER`, and `Math` helpers
- write generic functions and classes with `<T>`
- throw and interpret `RangeError` / `TypeError` in validation contracts
- read `readonly` parameter types and understand why inputs are never mutated
- recognize `?.`, `??`, spread `...`, destructuring, getters, and `satisfies`

## Prerequisites

- you have written *some* JavaScript or TypeScript before, in any context
- you can run a `.ts` file (for example with `npx tsx file.ts`) or paste snippets into the TypeScript Playground

---

## 1. Variables: `const` and `let`

The curriculum uses `const` by default and `let` only when a binding must be reassigned (loop counters, accumulators, moving pointers). `var` is never used.

```typescript
const limit = 10;        // cannot be reassigned
let total = 0;           // reassigned in the loop below

for (let index = 0; index < limit; index++) {
  total += index;
}

// limit = 20;           // compile error: cannot assign to a const
```

`const` prevents *reassignment*, not *mutation*: a `const` array can still be pushed into. Immutability of contents is expressed separately with `readonly` (section 10).

## 2. Primitive Types, `null`, and `undefined`

TypeScript's primitives used here: `number`, `string`, `boolean`, plus the two "absence" values.

- `undefined` — a value that was never produced (a missing array index, a `Map.get` miss).
- `null` — a deliberate "no result" chosen by the programmer (a function returning `number | null`).

The curriculum's convention: functions *return* `null` to mean "no answer exists"; `undefined` appears only as a consequence of lookups and is immediately guarded away.

```typescript
function findFirstNegative(values: readonly number[]): number | null {
  for (const value of values) {
    if (value < 0) {
      return value;
    }
  }
  return null; // deliberate "not found"
}

const found = findFirstNegative([3, -1, 4]);
if (found !== null) {
  const doubled = found * 2; // found is narrowed to number here
}
```

Two special numbers appear in prose and occasionally in code:

- `Infinity` — larger than every finite number; sometimes used as an initial "best so far".
- `NaN` — "not a number"; the result of invalid arithmetic. `NaN !== NaN`, so it is detected with `Number.isNaN(x)`, never with `=== NaN`.

## 3. Numbers, Safe Integers, and `Math`

JavaScript has one number type: 64-bit floating point. Integers are exact only up to `Number.MAX_SAFE_INTEGER` (2^53 − 1). Beyond that, additions silently lose precision — which is why the curriculum validates integers and checks additions instead of trusting them.

```typescript
const big = Number.MAX_SAFE_INTEGER; // 9007199254740991

console.log(Number.isInteger(4.0));      // true  (4.0 is the integer 4)
console.log(Number.isInteger(4.5));      // false
console.log(Number.isSafeInteger(big));  // true
console.log(Number.isSafeInteger(big + 1)); // false: precision no longer guaranteed

// The curriculum's checked-addition pattern:
function safeAdd(a: number, b: number): number {
  const sum = a + b;
  if (!Number.isSafeInteger(sum)) {
    throw new RangeError(`Sum ${a} + ${b} leaves the safe integer range`);
  }
  return sum;
}
```

`Math` helpers used throughout:

```typescript
console.log(Math.floor(7 / 2));  // 3   round down — array midpoints, halving
console.log(Math.trunc(-7 / 2)); // -3  drop the fraction toward zero
console.log(Math.min(3, 9));     // 3
console.log(Math.max(3, 9));     // 9
console.log(Math.abs(-5));       // 5
console.log(10 % 3);             // 1   remainder (modulo)
console.log(-1 % 3);             // -1  careful: JS remainder keeps the sign of the left operand
```

The negative-remainder behavior matters in circular-index math: `(index - 1 + length) % length` is the standard fix for stepping backwards around a ring.

## 4. Strings

Strings are immutable — every "modification" produces a new string. Character access by index is common in the string chapters.

```typescript
const word = "graph";

console.log(word.length);          // 5
console.log(word.charCodeAt(0));   // 103 — numeric code of "g", used for counting arrays
console.log(word.slice(1, 3));     // "ra" — substring, end exclusive
console.log(word.split(""));       // ["g","r","a","p","h"]
console.log(["g", "r"].join(""));  // "gr" — inverse of split
console.log(word.includes("rap")); // true
console.log(word.startsWith("gr"));// true
console.log(word.endsWith("ph"));  // true
console.log("ab".repeat(3));       // "ababab"
console.log(`word is ${word}`);    // template literal: interpolation with backticks

const letter = word[2]; // type: string | undefined (see section 12)
if (letter !== undefined) {
  console.log(letter.toUpperCase()); // "A"
}
```

`charCodeAt(i)` returns the UTF-16 code unit as a `number` — the curriculum uses it to map `"a"…"z"` onto array slots via `code - "a".charCodeAt(0)`.

## 5. Arrays

The workhorse structure. Key operations and their costs — the curriculum cares deeply about the difference:

```typescript
const values: number[] = [];

values.push(10);          // O(1) amortized — add to the end
values.push(20, 30);
const last = values.pop(); // O(1) — remove from the end; type: number | undefined

// values.shift() removes from the FRONT in O(n) — every element slides left.
// The curriculum bans shift() in queues and uses a head index instead:
const queue = [1, 2, 3];
let head = 0;
while (head < queue.length) {
  const current = queue[head];
  head++;
  if (current !== undefined) {
    console.log(current);
  }
}
```

Creation patterns:

```typescript
const zeros = new Array<number>(5).fill(0);          // [0, 0, 0, 0, 0]
const squares = Array.from({ length: 5 }, (_, i) => i * i); // [0, 1, 4, 9, 16]
const lists: number[][] = Array.from({ length: 3 }, () => []); // 3 INDEPENDENT empty arrays
// Never new Array(3).fill([]) — that would share ONE array across all slots.
```

Non-mutating helpers:

```typescript
const nums = [5, 3, 8, 1];

const copy = nums.slice();          // shallow copy
const firstTwo = nums.slice(0, 2);  // [5, 3]
const doubledAll = nums.map((n) => n * 2);   // [10, 6, 16, 2]
const evens = nums.filter((n) => n % 2 === 0); // [8]
const hasBig = nums.some((n) => n > 6);      // true
const allPositive = nums.every((n) => n > 0); // true
const sum = nums.reduce((acc, n) => acc + n, 0); // 17
```

Sorting — the default sort compares *strings*, so numbers always need a comparator:

```typescript
const data = [10, 2, 33];
data.sort((a, b) => a - b); // [2, 10, 33] ascending
data.sort((a, b) => b - a); // [33, 10, 2] descending
// Without a comparator, sort compares STRING forms: [10, 9].sort() gives [10, 9]
// because "10" < "9" character by character. Always pass a comparator for numbers.
```

Destructuring and spread:

```typescript
const pair: [number, number] = [3, 7];
const [low, high] = pair;            // low = 3, high = 7

const front = [1, 2];
const merged = [...front, 3, 4];     // [1, 2, 3, 4] — fresh array, inputs untouched
const maxOf = Math.max(...merged);   // spread into arguments: 4

const point = { x: 1, y: 2 };
const { x, y } = point;              // object destructuring
const shifted = { ...point, x: 10 }; // { x: 10, y: 2 } — shallow copy with override
```

## 6. Tuples and Type Aliases

A tuple is an array with a fixed length and per-position types. The graph chapters pass edges as tuples.

```typescript
type WeightedEdge = readonly [number, number, number]; // [from, to, weight]

const edge: WeightedEdge = [0, 2, 5];
const from = edge[0];  // number — tuple positions are always defined
const weight = edge[2];

type Result = { totalWeight: number; edges: [number, number, number][] };
```

`type` creates a name for any type shape. The curriculum uses `type` for data shapes and `interface` for object contracts that classes implement — both appear.

```typescript
interface Comparable {
  compareTo(other: Comparable): number;
}

type Maybe = number | null; // union alias
```

## 7. Objects and `Record`

Plain objects are used as named bundles of values. When keys are dynamic, the curriculum prefers `Map` (next section), but `Record` types appear in DTO-style shapes.

```typescript
type Step = {
  action: string;
  index: number;
  detail?: string; // optional property: string | undefined
};

const step: Step = { action: "visit", index: 3 };
console.log(step.detail ?? "no detail"); // ?? fallback (section 13)

const counts: Record<string, number> = {};
counts["apple"] = 2;

const keys = Object.keys(counts);     // ["apple"]
const values = Object.values(counts); // [2]
const pairs = Object.entries(counts); // [["apple", 2]]
```

## 8. `Map` and `Set`

Preferred over plain objects for dynamic keys: no prototype surprises, any key type, guaranteed insertion-order iteration, and an O(1) average `size`.

```typescript
const frequency = new Map<string, number>();
frequency.set("a", 1);
frequency.set("a", (frequency.get("a") ?? 0) + 1); // get returns number | undefined
console.log(frequency.get("a")); // 2
console.log(frequency.has("b")); // false
frequency.delete("a");
console.log(frequency.size);     // 0

const seen = new Set<number>();
seen.add(5);
seen.add(5);                     // duplicates are ignored
console.log(seen.has(5));        // true
console.log(seen.size);          // 1

for (const [key, value] of frequency.entries()) {
  console.log(key, value);
}
for (const item of seen.values()) {
  console.log(item);
}
```

The `get(...) ?? fallback` idiom is everywhere: `Map.get` returns `V | undefined`, and `??` supplies the default only when the value is `null` or `undefined`.

## 9. Functions, Arrow Functions, and Closures

```typescript
// Function declaration — used for the main exported algorithm of each problem
function clamp(value: number, low: number, high: number): number {
  return Math.min(high, Math.max(low, value));
}

// Arrow function — used for comparators, callbacks, and short helpers
const byAscending = (a: number, b: number): number => a - b;

// Closures: an inner function keeps access to outer variables
function makeCounter(): () => number {
  let count = 0;
  return () => {
    count++;
    return count;
  };
}
const next = makeCounter();
console.log(next(), next()); // 1 2
```

Recursion — a function calling itself — appears from the recursion chapter onward. Every recursive function in the curriculum states its base case first:

```typescript
function factorial(n: number): number {
  if (!Number.isInteger(n) || n < 0) {
    throw new RangeError(`n must be a nonnegative integer, received ${n}`);
  }
  if (n <= 1) {
    return 1; // base case
  }
  return n * factorial(n - 1);
}
```

## 10. `readonly` Parameters — the No-Mutation Contract

Every algorithm in this curriculum promises not to modify its input. TypeScript enforces this promise at compile time with `readonly` array and tuple types:

```typescript
function total(values: readonly number[]): number {
  // values.push(1);       // compile error: push does not exist on readonly number[]
  // values[0] = 9;        // compile error: index signature is readonly
  let sum = 0;
  for (const value of values) {
    sum += value;
  }
  return sum;
}

// Nested readonly — adjacency lists in the graph chapters:
function degreeOf(adjacency: readonly (readonly number[])[], vertex: number): number {
  const neighbors = adjacency[vertex];
  if (neighbors === undefined) {
    throw new RangeError(`Vertex ${vertex} is out of range`);
  }
  return neighbors.length;
}
```

A regular `number[]` can be passed where `readonly number[]` is expected — the annotation restricts only what the *function* may do. When an algorithm needs to modify data, it copies first (`values.slice()`, `row.slice()`, spread) and mutates the copy it owns.

## 11. Control Flow

```typescript
const n: number = 7; // annotated as number, not the literal type 7

// if / else if / else
if (n < 0) {
  console.log("negative");
} else if (n === 0) {
  console.log("zero");
} else {
  console.log("positive");
}

// ternary — only for simple value selection
const parity = n % 2 === 0 ? "even" : "odd";

// classic for — when the index itself matters
for (let i = 0; i < n; i++) {
  if (i === 2) continue; // skip to next iteration
  if (i === 5) break;    // leave the loop entirely
}

// for...of — when only the values matter
for (const ch of "abc") {
  console.log(ch);
}

// while — when the number of iterations is not known upfront
let remaining = n;
while (remaining > 1) {
  remaining = Math.floor(remaining / 2);
}
console.log(parity, remaining);
```

Early `return` is preferred over deep nesting: validate, return or throw on bad input, then write the main logic unindented.

## 12. Strict Indexing: `noUncheckedIndexedAccess`

The single most visible strictness setting in the curriculum. With it enabled, `array[i]` has type `T | undefined` — because at compile time TypeScript cannot prove `i` is in bounds. Every snippet therefore guards lookups:

```typescript
function firstElement(values: readonly number[]): number {
  const first = values[0]; // type: number | undefined
  if (first === undefined) {
    throw new RangeError("Expected a non-empty array");
  }
  return first; // narrowed to number
}

// Loop bodies use the same guard even when the index is provably valid,
// because the compiler does not track the proof:
function sumAll(values: readonly number[]): number {
  let sum = 0;
  for (let i = 0; i < values.length; i++) {
    const value = values[i];
    if (value === undefined) {
      continue; // unreachable for dense arrays, but satisfies the type system
    }
    sum += value;
  }
  return sum;
}
```

When you see `if (x === undefined) { throw ... }` or `?? fallback` immediately after an index or `Map.get`, that is this setting at work — not paranoia.

## 13. Union Types, Narrowing, `?.`, and `??`

A union type lists alternatives; *narrowing* is how TypeScript learns which alternative you hold.

```typescript
type TreeNode = { value: number; left: TreeNode | null; right: TreeNode | null };

function depth(node: TreeNode | null): number {
  if (node === null) {
    return 0; // narrowed: below this line node is TreeNode
  }
  return 1 + Math.max(depth(node.left), depth(node.right));
}

const root: TreeNode | null = { value: 1, left: null, right: null };

// Optional chaining: evaluate the right side only when the left is not null/undefined
const leftValue = root?.left?.value; // type: number | undefined

// Nullish coalescing: fallback only for null/undefined (NOT for 0 or "")
const shown = leftValue ?? -1;
console.log(shown); // -1

// typeof narrowing for primitives
function describe(input: number | string): string {
  if (typeof input === "string") {
    return input.toUpperCase();
  }
  return input.toFixed(2);
}
console.log(describe(3), describe("ok"));
```

Three special types you will meet in signatures:

- `void` — the function returns nothing useful (`function log(msg: string): void`).
- `never` — the code point is unreachable (a function that always throws, or an exhausted union).
- `unknown` — "some value, type not known yet"; must be narrowed before use. Appears at validation boundaries.

## 14. Errors and Validation Contracts

Every problem validates its inputs *fully and atomically* before doing any work, throwing on the first violation. Two error classes carry the meaning:

- `RangeError` — the value is the right kind but out of the allowed range (negative size, vertex ID ≥ V, unsafe integer).
- `TypeError` — the value is the wrong kind entirely (fractional where an integer is required, in some chapters).

```typescript
function makeBuffer(size: number): number[] {
  if (!Number.isSafeInteger(size) || size <= 0) {
    throw new RangeError(`size must be a positive safe integer, received ${size}`);
  }
  return new Array<number>(size).fill(0);
}

try {
  makeBuffer(-1);
} catch (error) {
  if (error instanceof RangeError) {
    console.log(`Rejected: ${error.message}`);
  } else {
    throw error; // never swallow unknown errors
  }
}
```

`throw` unwinds immediately — nothing after it in the function runs, which is what makes "validate everything first" atomic: either all inputs pass and work begins, or nothing was touched.

## 15. Classes

Used for owned data structures: linked-list nodes, tree nodes, heaps, and DSU.

```typescript
class MinHeap {
  private readonly items: number[] = []; // private: invisible outside the class

  get size(): number {          // getter: read as heap.size, no parentheses
    return this.items.length;
  }

  push(value: number): void {
    if (!Number.isSafeInteger(value)) {
      throw new RangeError(`value must be a safe integer, received ${value}`);
    }
    this.items.push(value);
    this.bubbleUp(this.items.length - 1);
  }

  pop(): number | null {
    const top = this.items[0];
    if (top === undefined) {
      return null;
    }
    const last = this.items.pop();
    if (last !== undefined && this.items.length > 0) {
      this.items[0] = last;
      this.sinkDown(0);
    }
    return top;
  }

  private bubbleUp(index: number): void {
    let child = index;
    while (child > 0) {
      const parent = Math.floor((child - 1) / 2);
      const childValue = this.items[child];
      const parentValue = this.items[parent];
      if (childValue === undefined || parentValue === undefined || parentValue <= childValue) {
        return;
      }
      this.items[child] = parentValue;
      this.items[parent] = childValue;
      child = parent;
    }
  }

  private sinkDown(index: number): void {
    let parent = index;
    while (true) {
      let smallest = parent;
      for (const child of [parent * 2 + 1, parent * 2 + 2]) {
        const childValue = this.items[child];
        const smallestValue = this.items[smallest];
        if (childValue !== undefined && smallestValue !== undefined && childValue < smallestValue) {
          smallest = child;
        }
      }
      if (smallest === parent) {
        return;
      }
      const a = this.items[parent];
      const b = this.items[smallest];
      if (a === undefined || b === undefined) {
        return;
      }
      this.items[parent] = b;
      this.items[smallest] = a;
      parent = smallest;
    }
  }
}

const heap = new MinHeap();
heap.push(5);
heap.push(2);
console.log(heap.size);  // 2
console.log(heap.pop()); // 2
```

Syntax to recognize:

- `constructor(size: number) { ... }` — runs on `new`.
- `private` / `readonly` on fields — visibility and reassignment control.
- `get name(): T` — a computed property read without parentheses.
- `this` — the current instance; every field access inside methods goes through it.
- `extends` (inherit a class) and `implements` (promise an interface's shape) appear occasionally.

## 16. Generics

A generic parameter `<T>` lets one implementation work for every element type while staying fully type-checked:

```typescript
function lastOf<T>(items: readonly T[]): T | null {
  const last = items[items.length - 1];
  return last === undefined ? null : last;
}

console.log(lastOf([1, 2, 3]));       // 3        (T = number)
console.log(lastOf(["a", "b"]));      // "b"      (T = string)
console.log(lastOf<number>([]));      // null

// Generic class with a comparator — the heap chapters use exactly this shape:
class Stack<T> {
  private readonly items: T[] = [];
  push(item: T): void {
    this.items.push(item);
  }
  pop(): T | null {
    const item = this.items.pop();
    return item === undefined ? null : item;
  }
  get size(): number {
    return this.items.length;
  }
}

const frameStack = new Stack<[number, number]>();
frameStack.push([0, 1]);
```

## 17. `satisfies` and Structural Typing

`satisfies` checks that a value conforms to a type *without* widening the value's own inferred type. The curriculum uses it in configuration-shaped literals:

```typescript
type Limits = Record<string, number>;

const limits = {
  maxVertices: 10_000,
  maxEdges: 50_000,
} satisfies Limits;

// limits.maxVertices keeps the precise type number (and autocomplete keeps both keys),
// while the compiler verified every value is a number.
const cap: number = limits.maxEdges;
console.log(cap);
```

TypeScript typing is *structural*: any object with the right shape satisfies a type, no declaration ceremony needed.

## 18. Equality and Copy Semantics

Two rules prevent most beginner bugs in this curriculum:

```typescript
// 1. Always === / !==, never == / !=. Loose equality coerces types silently.
console.log(0 === -0);        // true (fine)
console.log("5" === String(5)); // true — compare after explicit conversion, never via ==

// 2. Objects and arrays compare by REFERENCE, not by contents.
const a = [1, 2];
const b = [1, 2];
console.log(a === b);          // false — different arrays
const c = a;
console.log(a === c);          // true — same array
c.push(3);
console.log(a.length);         // 3 — mutating c mutated a; they are one object

// Fresh copies break the sharing:
const independent = a.slice();
independent.push(4);
console.log(a.length);         // still 3
```

This is exactly why `readonly` parameters plus explicit copies (`slice`, spread, `Array.from`) form the ownership discipline used in every problem — and why the clone-graph problem needs an identity `Map` from originals to copies.

---

# Quick Reference

| Construct | Meaning | Where you will see it |
|---|---|---|
| `const` / `let` | binding without / with reassignment | everywhere |
| `number \| null` | union: a number or deliberate absence | search results, tree pointers, topological sort |
| `x === undefined` guard | strict-indexing check | every array/`Map` lookup |
| `?? fallback` | default only for `null`/`undefined` | frequency maps, optional fields |
| `?.` | stop early when left side is nullish | tree/list pointer walks |
| `readonly number[]` | function promises not to mutate | every algorithm input |
| `readonly [number, number, number]` | fixed-shape tuple | weighted edges |
| `new Array<number>(n).fill(0)` | preallocated dense array | distances, visited flags, counts |
| `Array.from({ length: n }, () => [])` | n independent arrays | adjacency lists |
| head-index loop instead of `shift()` | O(1) queue removal | every BFS |
| `(a, b) => a - b` | numeric ascending comparator | every numeric `sort` |
| `Map` / `Set` | dynamic keys, O(1) average operations | hashing chapter onward |
| `Number.isSafeInteger` / `safeAdd` | overflow-checked arithmetic | prefix sums, Dijkstra, MST |
| `Math.floor((low + high) / 2)` | integer midpoint | binary search, heaps |
| `throw new RangeError(...)` | atomic input validation | every problem's first lines |
| `class` + `private` + `get` | owned structure with controlled access | linked lists, heaps, DSU |
| `<T>` | generic over element type | stacks, queues, heaps |
| `satisfies` | check a literal against a type without widening | bounded configs |
| `...spread` / destructuring | fresh copies and unpacking | copy-then-mutate discipline |

## How to Practice

1. Retype each snippet in this file and run it — do not copy-paste.
2. Break each snippet on purpose (remove a guard, drop a comparator, mutate a `readonly` input) and read the compiler error until it makes sense.
3. When any later chapter shows syntax you hesitate on, return to the matching section here before continuing.

Next: `00-dsa-fundamentals.md`.
